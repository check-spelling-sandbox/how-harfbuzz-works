# Drawing outlines

Shaping ends with glyph IDs and positions. Something still has to turn each glyph ID into a shape, and HarfBuzz can do that too. Its draw API hands a glyph’s outline to the client one segment at a time, as moves, lines and curves, with variations, synthetic bold and slant, and the font’s scale already applied. It doesn’t rasterize anything, and it doesn’t hint: as the manual says, the outlines are “as they are defined in the font, without applying hinting to fit the curves to the pixel grid” (`docs/usermanual-fonts-and-faces.xml`).

This chapter starts with the API as a client sees it, then takes the outline formats in the order HarfBuzz tries them: `glyf`, with its quadratic curves, composites and, in the newer `GLYF` table, cubic curves; then the CFF and CFF2 charstring interpreter. It ends with the limits that stop a hostile font from making one glyph take forever, and the budget shared between them. How gvar moves the points is in [Varying outlines](varying-outlines.html), VARC is in [Variable components](variable-components.html), and color glyphs, which are painted rather than drawn, are in [Color](color.html). The figures are drawn by `hb-vector`, itself a client of the draw API, from test fonts in `fonts/drawing-outlines/`.

## The draw API

A client asks for an outline with `hb_font_draw_glyph_or_fail (font, glyph, dfuncs, draw_data)`. The **draw funcs**, an `hb_draw_funcs_t`, are a table of five callbacks: `move_to`, `line_to`, `quadratic_to`, `cubic_to` and `close_path`. `draw_data` is the client’s own pointer, passed back to each of them. The function returns false if the font has no outline for the glyph. It has had that name since 11.2.0; the older `hb_font_draw_glyph` does the same and returns nothing (`hb-font.cc:1637`).

Every callback also gets an `hb_draw_state_t`, the **draw state**: whether a path is open, where it started, and the current point. A callback can read the current point from it, since a segment doesn’t repeat its start:

```c
typedef struct hb_draw_state_t {
  hb_bool_t path_open;

  float path_start_x;
  float path_start_y;

  float current_x;
  float current_y;

  /*< private >*/
  hb_var_num_t   reserved1;
  /* … reserved2 to reserved7 */
} hb_draw_state_t;
```

(trimmed). Coordinates are floats, in the font’s scale, with y pointing up. They aren’t rounded.

### What the client sees

Every outline source draws through `hb_draw_funcs_t`’s inline methods (`hb-draw.hh:113–189`), which keep the draw state and tidy the path on the way:

* `move_to` sends nothing. It only records the point. The client’s `move_to` is sent by the first line or curve after it (`start_path`, `hb-draw.hh:182`). So a contour that is only a move never reaches the client, and two moves in a row reach it as one.
* A move while a path is open closes that path first.
* `close_path` sends a `line_to` back to the start of the contour if the last segment ended anywhere else, and then the client’s `close_path`. Every contour the client sees ends where it began, and is explicitly closed.

Each outline source works through an `hb_draw_session_t` (`hb-draw.hh:193`), an internal helper that holds the funcs, the client’s data and a fresh draw state for one glyph. Its destructor closes whatever path is still open, so even a glyph whose data ends mid-contour reaches the client closed.

These rules don’t remove degenerate contours. A `glyf` contour of one point, which old fonts used as anchors, arrives as a `move_to`, a `line_to` to the same point, and a `close_path`: the path builder draws a line back to the contour’s first point (`OT/glyf/path-builder.hh:168`), and that line opens the path. A client that fills will draw nothing there; a client that strokes, or counts contours, will see it.

### Callbacks a client leaves out

A new `hb_draw_funcs_t` starts with every callback set to a “nil” function, and setting one to `NULL` puts the nil one back. All but one of the nil functions do nothing. The exception is `quadratic_to`, whose nil version converts the curve to a cubic and calls `cubic_to` (`hb-draw.cc:62`):

```c++
#define HB_TWO_THIRD 0.66666666666666666666666667
dfuncs->emit_cubic_to (draw_data, *st,
                       (float) (current_x + (cx - current_x) * HB_TWO_THIRD),
                       (float) (current_y + (cy - current_y) * HB_TWO_THIRD),
                       (float) (tx + (cx - tx) * HB_TWO_THIRD),
                       (float) (ty + (cy - ty) * HB_TWO_THIRD),
                       to_x, to_y);
```

(trimmed). The two cubic control points lie two thirds of the way from each end toward the quadratic one, which gives exactly the same curve. It is computed in doubles, so that, says the comment, “the result does not depend on how the compiler evaluates float expressions”.

| The client leaves out | What happens |
|---|---|
| `quadratic_to` | quadratic curves arrive as cubics |
| `cubic_to` | cubic curves are lost, and so are quadratics if `quadratic_to` is missing too |
| `move_to`, `line_to`, `close_path` | those calls are lost |

Nothing converts the other way: a client that can only draw quadratics has to approximate cubics itself. `hb-vector`’s PDF output relies on the conversion (“No quadratic_to: null fallback auto-promotes to cubic”, `hb-vector-draw.cc:184`).

> For programmers: a client that sets `move_to`, `line_to`, `cubic_to` and `close_path` gets every outline HarfBuzz can draw. Set `quadratic_to` as well only if quadratic segments are worth keeping, for instance to write them out unchanged.

Since 14.2.0 the API also has `hb_draw_line`, `hb_draw_rectangle` and `hb_draw_circle`, which draw those shapes through any draw funcs. The outline code doesn’t use them.

### Above the outline sources

Between the client and the outline is the font layer (`hb-font.hh:690`), which passes the request to the font funcs, rescales it through any parent fonts, and applies synthetic slant and bold, as [Fonts and font functions](fonts-and-font-functions.html#synthetic-bold-and-slant) explains. The rest of this chapter is hb-ot’s side of that callback.

## Where the outline comes from

hb-ot tries four sources, in a fixed order, and the first that says it has the glyph wins (`hb-ot-font.cc:900`):

```c++
hb_draw_session_t draw_session {draw_funcs, draw_data};
…
if (font->face->table.VARC->get_path (font, glyph, draw_session)) return true;
// Keep the following in synch with VARC::get_path_at()
if (font->face->table.glyf->get_path (font, glyph, draw_session, gvar_cache,
                                      &draw_session.get_budget ())) return true;
if (font->face->table.cff2->get_path (font, glyph, draw_session,
                                      &draw_session.get_budget ())) return true;
if (font->face->table.cff1->get_path (font, glyph, draw_session,
                                      &draw_session.get_budget ())) return true;
return false;
```

(trimmed of `#ifdef`s). [VARC](variable-components.html) comes first, so a glyph it covers is drawn from it whatever `glyf` or CFF holds. Then `glyf`, then CFF2, then CFF. A font with both `glyf` and CFF outlines is drawn from `glyf`.

“Has the glyph” means a glyph ID within the table’s range. A `glyf` glyph with no contours, like a space, counts: `glyf` draws nothing and returns true, and CFF isn’t asked. So does a glyph whose `loca` entry is broken, which `glyf` treats as empty (`OT/glyf/glyf.hh:574`). Only a glyph past the end of `loca` falls through to the next table.

Bitmaps and color glyphs aren’t on this list. They are painted, through a different callback; when a color glyph has no paint data, painting falls back to the outline. [Color](color.html) covers that side. [Glyph extents](where-metrics-come-from.html#glyph-extents) are found in yet another order, with bitmaps and color first.

## glyf: from points to a path

A `glyf` glyph is either simple, a list of contours made of points, or composite, a list of other glyphs placed with transforms. For both, `glyf_accelerator_t::get_points` (`OT/glyf/glyf.hh:288`) first gathers every point of the whole glyph into one array, with gvar’s deltas applied and the four phantom points at the end, as [Varying outlines](varying-outlines.html#a-glyphs-points-and-four-more) describes. Only then are the points turned into draw calls, by `path_builder_t` (`OT/glyf/path-builder.hh`), which scales each point to the font’s size as it reads it. So a broken glyph fails before anything is drawn: a `glyf` glyph is drawn whole or not at all.

### Simple glyphs

A simple glyph stores the last point number of each contour, its hinting instructions, then a flag byte per point, then the x coordinates and the y coordinates, as deltas from the previous point. The flags say how each coordinate is stored (a byte or two, with a sign, or the same as the last), whether the flag repeats, and whether the point is **on-curve**, a point the outline passes through, or **off-curve**, a control point of a quadratic curve. `SimpleGlyph::get_contour_points` reads them (`OT/glyf/SimpleGlyph.hh:186`). It skips the instructions without looking at them: HarfBuzz has no TrueType interpreter. The `OVERLAP_SIMPLE` flag is read and kept but means nothing to drawing.

Two off-curve points in a row have an **implied on-curve point** halfway between them. That is what lets a TrueType outline chain quadratic curves without storing the points where they join, and it is most of what the path builder does. Its comment credits ttf-parser’s version of the same loop (`path-builder.hh:34`). For each point:

* after an on-curve point, an on-curve point is a `line_to`, and an off-curve one is remembered;
* after an off-curve point, an on-curve point ends a `quadratic_to`, and another off-curve point ends one at the midpoint of the two, which becomes the start of the next.

A contour can be entirely off-curve. Then every point is a control point and every midpoint is on the outline. The test font has three glyphs made of the same four points, the corners of a square: in `oncurve` they are on-curve, in `offcurve` off-curve, and `b` is a stem and a bowl whose bowl has two off-curve points in a row, with the implied point between them at the bowl’s right-hand extreme:

```draw
font: outlines.ttf
text: b12
```

The `offcurve` glyph is a rounded square through the midpoints of the square’s sides, four quadratic curves with the corners as their control points.

One detail decides where a contour starts. If its first point is off-curve, HarfBuzz starts with the last point, and if that is off-curve too, at the midpoint of the two (`glyf.hh:334–385`). The simpler loop that would just take points in order is still in the source, under `if (false)`, with the comment “FreeType and CoreText do it differently, so we match those”. It makes no difference to the shape, but it does to a client that looks at where each contour begins.

### Composite glyphs

A composite glyph is a list of components, each another glyph with flags, two arguments and, optionally, a scale, a pair of scales, or a 2×2 matrix (`OT/glyf/CompositeGlyph.hh`). Usually the arguments are an x and y offset. HarfBuzz loads each component’s points recursively, transforms them, and adds them to the composite’s array. As far as drawing is concerned, a composite is the simple glyph its components add up to.

The test font’s `d`, `p` and `q` are all `b`, transformed: `d` is mirrored with the matrix (−1, 0, 0, 1), `p` with (1, 0, 0, −1), and `q` turned half round with (−1, 0, 0, −1), each with an offset that puts it back in place:

```draw
font: outlines.ttf
text: bdpq
```

Most of the component flags change nothing in HarfBuzz’s drawing:

| Flag | In HarfBuzz’s drawing |
|---|---|
| `ARGS_ARE_XY_VALUES` | set: the arguments are an offset; clear: they are point numbers to match (below) |
| `WE_HAVE_A_SCALE`, `WE_HAVE_AN_X_AND_Y_SCALE`, `WE_HAVE_A_TWO_BY_TWO` | the transform, in F2Dot14 |
| `SCALED_COMPONENT_OFFSET`, `UNSCALED_COMPONENT_OFFSET` | whether the offset is transformed too (below) |
| `USE_MY_METRICS` | the composite takes this component’s phantom points, so its metrics, and so the left-side-bearing shift ([Varying outlines](varying-outlines.html#composites)) |
| `ROUND_XY_TO_GRID` | ignored: nothing is rounded |
| `WE_HAVE_INSTRUCTIONS`, `OVERLAP_COMPOUND` | ignored |

Two flags decide the order of the transform and the offset. With `SCALED_COMPONENT_OFFSET` alone, the offset is added first and transformed with the points; otherwise, with neither flag or both, the points are transformed and then moved by the offset (`CompositeGlyph.hh:129`). `outlines-scaled.ttf` is `outlines.ttf` with `SCALED_COMPONENT_OFFSET` set on the components of `d`, `p` and `q`:

```draw
font: outlines-scaled.ttf
text: bdpq
```

`p` and `q` drop a long way below the baseline: their offset of 450 has been flipped to −450 along with the outline. The flipped x offsets of `d` and `q` don’t show, because of the left-side-bearing shift: HarfBuzz moves every outline so that its left edge is where `hmtx` says it should be, and the font’s `hmtx` was made for the other flag.

### Matching points

A component whose `ARGS_ARE_XY_VALUES` flag is clear is placed by **point matching**: its two arguments are point numbers, one in the composite so far and one in the component, and the component is moved so that the two points meet (`OT/glyf/Glyph.hh:539`). This is how TrueType attached accents before GPOS existed, and because it works on points after gvar has moved them, an accent placed this way follows its base through a variable font.

The test font’s `i` is a `dotlessi` and a `dotaccent`. The dot glyph has a second contour of a single point 150 units below the dot, and the composite matches that point, number 4, with point 1 of `dotlessi`, the top of its stem. The test font’s `x` is a composite of `b` and that `i`, side by side:

```draw
font: outlines.ttf
text: bix
```

In `i` the dot is 150 units above the stem, as intended. In `x` it is 250 units higher. The point numbers are meant to count the points of the composite being built, here `i`, but HarfBuzz counts from the start of the whole glyph’s array, which is the same thing only at the top level. Inside `x`, point 1 is the top of `b`’s stem, at 750, not the top of `dotlessi`’s, at 500. fontTools, asked for the same glyph’s coordinates, puts the dot where it is in `i`. This is a bug in 15.1.0, found while writing this chapter. It affects only composites that match points and are themselves used as components, which are rare.

## Cubic curves in glyf

TrueType outlines are quadratic, and CFF outlines cubic. A proposal for HarfBuzz’s “boring expansion” of OpenType adds cubic curves to `glyf`, and HarfBuzz supports it: experimentally since 7.0.0, whose `NEWS` points to the specification (`glyf1-cubicOutlines.md` in the `boring-expansion-spec` repository), and by default since 15.0.0, as part of the ISO standard’s new tables.

The extension uses bit 7 of a point’s flag byte, which was reserved: `FLAG_CUBIC`, 0x80 (`SimpleGlyph.hh:23`). An off-curve point with that bit set is a control point of a cubic curve, and two in a row are the two controls of one cubic. Between two such pairs there is an implied on-curve point, at the midpoint of the second control of one pair and the first of the next, just as between two quadratic control points. The path builder handles both kinds in one pass: it holds up to two control points, draws a `cubic_to` when it has two cubic ones, and a `quadratic_to` when it has one. A cubic control point on its own, or the third of three in a row, is drawn as a quadratic one (inferred from `path-builder.hh:41–127`). The contour-start rotation above is skipped for a contour that starts and ends with cubic control points, since moving one point to the front would pair the controls differently (`glyf.hh:362–371`).

HarfBuzz reads the bit only in the `GLYF` table, the one the ISO standard introduces for fonts with more than 65,535 glyphs ([Fonts as bytes](fonts-as-bytes.html#smalltypes-and-mediumtypes)). In `glyf` it masks it off (`SimpleGlyph.hh:138`):

```c++
uint8_t flag = *p++;
if (!extended)
  flag &= ~FLAG_CUBIC;
```

`extended` is true only when the face has both `GLYF` and `LOCA`, which are then used instead of `glyf` and `loca` (`glyf.hh:224–240`). The test font’s `cubic` glyph is the same four corners again, with the cubic bit set. `outlines.ttf` stores it in `glyf`, and `outlines-GLYF.ttf` stores exactly the same bytes in `GLYF`:

```draw
font: outlines.ttf
text: 123
```

```draw
font: outlines-GLYF.ttf
text: 123
```

In `glyf`, `cubic` is drawn like `offcurve`, with quadratics. In `GLYF`, its four corners are two pairs of cubic controls, the implied points are the middles of the top and bottom edges, and it is drawn as two cubic curves: narrower, since a cubic doesn’t get as close to its controls as a quadratic does, and touching only the top and bottom of the square. A font that wants cubic outlines in HarfBuzz {{hb}} has to use `GLYF`, whatever its glyph count.

## CFF and CFF2: running charstrings

A CFF glyph is a **charstring**, a small program in Adobe’s Type 2 format: numbers pushed on a stack, and operators that use them to move, draw lines and draw cubic curves. HarfBuzz runs it with an interpreter built from templates, shared between drawing, extents and subsetting. The CFF and CFF2 code was contributed by Adobe, and merged in 2.3.0, mostly for the subsetter but also for glyph extents (`NEWS`):

| Layer | Where | What it adds |
|---|---|---|
| `interp_env_t` | `hb-cff-interp-common.hh:574` | the byte cursor, and the argument stack: 513 numbers, as doubles |
| `cs_interp_env_t` | `hb-cff-interp-cs-common.hh:113` | the current point, the subroutines, a call stack 10 deep, hint counts, the budget |
| `cs_opset_t` | `hb-cff-interp-cs-common.hh:247` | what each charstring operator does |
| `path_procs_t` | `hb-cff-interp-cs-common.hh:459` | the path operators, turned into `moveto`, `line` and `curve` |
| `cff1_…`, `cff2_…` | `hb-cff1-interp-cs.hh`, `hb-cff2-interp-cs.hh` | the differences between the two formats |
| `cff1_path_param_t`, `cff2_path_param_t` | `hb-ot-cff1-table.cc:453`, `hb-ot-cff2-table.cc:164` | the draw session, and scaling to the font’s size |

With other parameters, the same templates collect a bounding box for [extents](where-metrics-come-from.html#glyph-extents).

The path operators, from `rlineto` to `vhcurveto`, differ only in how they pack their coordinates, and `path_procs_t` unpacks each into lines and cubic curves. The four flex operators, which a hinting rasterizer may flatten at small sizes, are always drawn as their two curves. A charstring never produces a quadratic: CFF outlines reach the client entirely as `move_to`, `line_to`, `cubic_to` and `close_path`.

Much of a charstring is for hinting, and HarfBuzz reads it only to get past it. The stem operators, `hstem`, `vstem` and their `hm` forms, only count their stems, because the count says how many bytes of mask follow each `hintmask` and `cntrmask`, which are then skipped. `dotsection` does nothing. The width, which can come before the first stack-clearing operator, is noticed and dropped: advances come from `hmtx`. An operator HarfBuzz doesn’t know, including the arithmetic operators that Type 2 inherited from PostScript and few fonts use, clears the stack and is otherwise ignored (`hb-cff-interp-common.hh:646`; “Ignore unknown CFF operators”, `NEWS` 10.4.0).

### Subroutines

Shared pieces of outline are stored once, as subroutines, global or local to a font dict, and called with `callgsubr` or `callsubr`. The number on the stack is biased, so that small numbers, which cost fewer bytes, can reach the whole list: by 107 if there are fewer than 1,240 subroutines, 1,131 if fewer than 33,900, and otherwise 32,768 (`hb-cff-interp-cs-common.hh:63`). A call pushes the current position on the call stack; `return`, or the end of a CFF2 subroutine, pops it. A call that would nest more than 10 deep, or a subroutine number out of range, is an error.

### seac

A CFF (version 1) charstring can end with `endchar` and four extra numbers. That is the **seac** form, inherited from Type 1’s “standard encoding accented character”: draw the glyph for one StandardEncoding code, then the glyph for another, moved by an offset. The codes are looked up through the font’s charset, so it works only for glyphs with standard names. `process_seac` (`hb-ot-cff1-table.cc:528`) draws both glyphs into the same session, and refuses a seac inside a seac. In `cff.otf`, `odieresis` is the charstring `600 150 600 111 200 endchar`: a width of 600, then an offset of (150, 600), `o` (code 111) and `dieresis` (code 200):

```draw
font: cff.otf
text: oö
```

### CFF2: blend and vsindex

CFF2 charstrings vary. Each varying number is given with a delta for each region of the variation store, followed by the `blend` operator, which replaces them with the value at the font’s position. Which of the store’s lists of regions applies is set by the Private DICT, or, for one charstring, by a `vsindex` operator at its start. HarfBuzz computes the region scalars at the first `blend` and adds the deltas without rounding (`hb-cff2-interp-cs.hh:142–162, 314`), as [Varying outlines](varying-outlines.html#cff2) says. A `vsindex` after a `blend`, or a second one, is an error, and the glyph fails (`:168`).

HarfBuzz’s test font `AdobeVFPrototype_vsindex.otf` has an `A` and a `B` whose charstrings are the same except that `B`’s starts with `1 vsindex`. The second list of regions swaps two of the first list’s regions, which peak at the heaviest weight, for ones that peak halfway there, in normalized coordinates:

```draw
font: hb:test/api/fonts/AdobeVFPrototype_vsindex.otf
text: AB
slider: wght 200 900 50
start: 400
print: 200 400 650 900
```

Up to the default weight, about 389, the two are identical. Above it they part: `A` keeps getting heavier, while `B` is heaviest at about 640 (halfway, once `avar` has been applied) and is back to its default outline at 900. (Their advances differ throughout: they come from `hmtx` and `HVAR`, which this test font didn’t change. And `hb-shape` calls them `gid1` and `gid2`, since the font has no glyph names.)

CFF2 also drops `endchar`: a charstring simply ends. HarfBuzz’s CFF2 interpreter makes one up when it runs out of bytes, and a `return` at the end of a subroutine (`hb-cff2-interp-cs.hh:125`).

### When a charstring fails

Unlike a `glyf` glyph, a charstring is drawn as it is run. If it fails halfway, by overflowing the stack, calling a subroutine that doesn’t exist, or running out of budget, the part already drawn has reached the client by the time `hb_font_draw_glyph_or_fail` returns false.

A CFF (version 1) charstring that ends without `endchar` shows another side of this. Running off the end isn’t an error in CFF: the interpreter reads an invalid operator, ignores it, and carries on until it has counted 200,000 operators. In a test font made while writing this chapter, such a glyph was drawn, and then reported as a failure. Each one also used 200,000 units of the budget described next, so in `hb-vector` a line of about 80 of them used all of it, and the glyphs after them weren’t drawn. It is probably a bug, though only a malformed font can trigger it.

## Limits, and the shared budget

[The sanitizer](trust-nothing-the-sanitizer.html#checks-that-wait) leaves `glyf` and charstrings to be checked as they are used. Composites nest, charstrings call subroutines, and VARC components can be composites, so a small font could ask for an enormous amount of work. Each format has its own limits (`hb-limits.hh` unless noted):

| Limit | Value | What it bounds |
|---|---|---|
| `HB_MAX_NESTING_LEVEL` | 64 | how deep composites nest |
| `HB_MAX_GRAPH_EDGE_COUNT` | 16,384 | how many glyphs one composite loads, counting repeats |
| `HB_GLYF_MAX_POINTS` | 200,000 | points in one composite |
| `HB_CFF_MAX_OPS` | 200,000 | operators in one charstring, subroutines included |
| `kMaxCallLimit` | 10 | subroutine nesting (`hb-cff-interp-cs-common.hh:59`) |
| argument stack | 513 | numbers on a charstring’s stack (`hb-cff-interp-common.hh:443`) |
| `HB_BUDGET_GLYPH` | 2²⁴ | the work budget, below |

A composite that contains itself, directly or further down, is caught by a cycle detector, and the component that closes the cycle is skipped while the rest of the glyph is drawn (`Glyph.hh:470`). The other limits fail the glyph. In `limits.ttf`, `spiral64` is a square and `spiral63`, turned 15° and shrunk; `spiral63` is a square and `spiral62`, and so on down to `spiral0`, a plain square. So `spiralN` nests N composites deep:

```draw
font: limits.ttf
text: 12
```

`spiral64`, on the `1`, is drawn: 64 levels, 65 squares. `spiral65`, on the `2`, is one level too deep, and nothing is drawn. Its extents still come from its header, so to `hb-shape` and to layout it looks like any other glyph.

### The budget

Each limit bounds one table, but tables drive each other: VARC loads `glyf` and CFF outlines, COLR paints outlines, and a renderer draws thousands of glyphs. Limits that are safe one at a time multiply. So since 14.5.0 every drawing session also carries one **work budget**, a count that every piece of work subtracts from and nothing adds back to (`NEWS`; `hb-limits.hh:197`):

```c++
/* The common finite default for one top-level glyph rendering session.
 * Nested outline and paint work shares the same live counter. */
#define HB_BUDGET_GLYPH ((int64_t) 1 << 24)

static HB_ALWAYS_INLINE bool
hb_budget_spend (int64_t &budget, unsigned int cost, unsigned int mult = 1)
{
  budget -= (int64_t) cost * mult;
  return budget >= 0;
}
```

The units are rough weights. `glyf` charges 16 for each glyph it loads and each component it prepares, 1 for each point it reads, 1 for each point each time a composite level transforms it, and 1 for each point at the end (`Glyph.hh:350, 378, 523`; `SimpleGlyph.hh:198`; `glyf.hh:309`). gvar charges 1 per point for each tuple that applies (`hb-ot-var-gvar-table.hh:825`). The charstring interpreter charges 1 per operator, and `blend` 1 per delta. It caps its own operator count at whatever budget is left, so it can’t overshoot (`hb-cff-interp-cs-common.hh:892`). VARC charges for its components and axis values. When the budget runs out, the work in hand fails, and so does everything after it in the session.

Where the budget lives depends on the client. The draw funcs have three more callbacks, added with it: `set_budget`, `get_budget`, and `get_budget_remaining`, which returns a pointer to a live count. If the client’s funcs provide that pointer, the draw session uses it, and the count carries over from one glyph to the next until the client resets it. If not, each call to `hb_font_draw_glyph_or_fail` gets its own budget of 2²⁴ (`hb-draw.hh:195–201`). `hb_draw_set_budget` sets a client’s policy: a number, `HB_BUDGET_DEFAULT` for 2²⁴, or `HB_BUDGET_UNLIMITED`, which leaves only the structural limits. For funcs that don’t track a budget, `hb_draw_get_budget_remaining` reports “unlimited”, though each call is still limited to 2²⁴.

HarfBuzz’s own renderers track it, and `hb-vector` keeps one budget for its whole output. In `limits.ttf`, `heavy` is a square and 120 components, each of them 120 components of an empty glyph. That is 14,521 glyphs to load, inside the edge limit, and about half a million units of budget, for one square. Here are 34 of them, then `s`, a plain square:

```draw
font: limits.ttf
text: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhs
```

Thirty-two are drawn. The thirty-third runs the budget out, and from there on nothing is drawn, not even the plain square, which on its own costs almost nothing. The shaping beneath is unaffected: the budget belongs to drawing.

> For programmers: a client with its own draw funcs gets a fresh budget for every glyph, which is safe and usually what it wants. A renderer that draws whole pages from untrusted fonts can provide `get_budget_remaining` to bound the page as a whole, and then has to recharge it, with `set_budget`, at whatever unit of work it thinks right.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-draw.h`, `src/hb-draw.cc` | the public API; the nil callbacks, and the quadratic-to-cubic conversion, 62; the line, rectangle and circle helpers |
| `src/hb-draw.hh` | `hb_draw_funcs_t`’s lazy `move_to` and closing `close_path`, 113–189; `hb_draw_session_t`, 193 |
| `src/hb-font.hh`, `src/hb-font.cc` | `draw_glyph_or_fail`, with synthetic slant and bold, 690; the sub-font adaptor, `hb-font.cc:770` |
| `src/hb-ot-font.cc` | `hb_ot_draw_glyph_or_fail`, 900: VARC, `glyf`, CFF2, CFF |
| `src/OT/glyf/glyf.hh` | `get_points`, 288, and the contour-start rotation; `GLYF` and `LOCA`, 224 |
| `src/OT/glyf/Glyph.hh` | loading composites, 326: limits, cycles, transforms, point matching |
| `src/OT/glyf/SimpleGlyph.hh`, `src/OT/glyf/CompositeGlyph.hh` | flags and points; component flags and transforms |
| `src/OT/glyf/path-builder.hh` | points to draw calls: implied points, cubic pairs |
| `src/hb-cff-interp-common.hh`, `src/hb-cff-interp-cs-common.hh` | the charstring interpreter: stacks, operators, subroutines, `interpret`, 887 |
| `src/hb-cff1-interp-cs.hh`, `src/hb-ot-cff1-table.cc` | CFF: width, `seac`, 528; drawing, 453 |
| `src/hb-cff2-interp-cs.hh`, `src/hb-ot-cff2-table.cc` | CFF2: `blend`, `vsindex`, the made-up `endchar`; drawing, 164 |
| `src/hb-limits.hh` | the limits, and the budget, 197 |

## In short

* A client gets an outline as `move_to`, `line_to`, `quadratic_to`, `cubic_to` and `close_path` calls, scaled and unrounded, unhinted. Moves are sent lazily, every contour is closed back to its start, and a missing `quadratic_to` becomes `cubic_to`.
* hb-ot draws from VARC, then `glyf`, then CFF2, then CFF. An empty `glyf` glyph still counts as drawn.
* `glyf` points become curves through implied on-curve points between off-curve ones. Composites are transformed and flattened before anything is drawn; `ROUND_XY_TO_GRID` and the overlap flags are ignored. Point matching inside a nested composite counts points from the wrong place in 15.1.0.
* Cubic `glyf` curves use flag bit 0x80, in pairs, and HarfBuzz reads the bit only in `GLYF`, not `glyf`.
* CFF and CFF2 charstrings run in one templated interpreter that ignores hints, follows subroutines 10 deep, draws `seac` accents, and applies CFF2 blends unrounded. A charstring that fails halfway has already drawn part of its glyph.
* Nesting, components, points and operators each have a limit, and since 14.5.0 a shared budget of 2²⁴ units bounds a whole drawing session, per glyph for most clients, and per output for HarfBuzz’s own renderers, like `hb-vector`. Those renderers are the subjects of [hb-raster](hb-raster.html), [hb-vector](hb-vector.html) and [hb-gpu](hb-gpu.html).
