# Instancing

A variable font holds a whole design space. An application that only ever uses the bold weight carries every other weight with it, and a renderer that can’t read variations can’t use the font at all. An **instance** is a font cut from a variable font at one position, or over a smaller part of its design space. Fixing an axis at one value is **pinning** it: the axis disappears, and its effect is baked into the outlines and metrics. Narrowing an axis’s range, and perhaps moving its default, is **limiting** it: the axis stays, with new limits, and every delta that refers to it has to be rewritten. A **full instance** pins every axis and is an ordinary static font; anything else is a **partial instance**, still variable.

HarfBuzz instances inside its subsetter. Axis settings are part of the subset input, and each table’s `subset` method both drops the glyphs the [plan](subsetting.html#the-plan) leaves out and rewrites the table’s variation data, in one pass. Pinning arrived in 6.0.0, for TrueType fonts with every axis pinned; partial instancing, CFF2, COLR v1 and the other tables followed, and the API was declared stable in 8.5.0 (NEWS). The algorithms are ports of fontTools’ instancer, `fontTools.varLib.instancer`, and the source says so file by file.

The examples are cut from the test fonts of [Design space](design-space.html), and from one more built on them, by HarfBuzz’s `hb-subset` in `fonts/instancing/build.sh`. Every block on this page reads those fonts with `hb-shape`, `hb-info` and `hb-vector` from HarfBuzz {{hb}}; the TTX views are fontTools’ dumps of the same files.

## Asking for an instance

Five functions in `hb-subset.h` set axes on an `hb_subset_input_t`. Each looks the axis up in the face’s `fvar` and stores a triple of user coordinates (minimum, default, maximum) under its tag (`hb-subset-input.cc:436`–`562`):

| Function | Stores | Since |
|---|---|---|
| `hb_subset_input_pin_axis_location` | the value, clamped to the axis, three times | 6.0.0 |
| `hb_subset_input_pin_axis_to_default` | the axis default, three times | 6.0.0 |
| `hb_subset_input_pin_all_axes_to_default` | the same, for every axis | 8.3.1 |
| `hb_subset_input_set_axis_range` | a new minimum, default and maximum; NaN keeps the old value | 8.5.0 |
| `hb_subset_input_get_axis_range` | reads back what was set | 8.5.0 |

`set_axis_range` clamps the new minimum and maximum to the old ones, and then the default to the new range, so a range that leaves out the old default moves the default to whichever end is nearer. A range that is exactly the axis’s own is forgotten before anything else happens (`hb-subset-plan-var.cc:471`).

The `hb-subset` tool takes the same settings as `--variations` (called `--instance` before 8.3.0): `wght=700` pins, `wght=400:800` limits and keeps the default, `wght=500:700:900` also moves it, an empty part such as `wght=:500:` keeps that value, and `wdth=drop` or `*=drop` pins at the default (`util/helper-subset.hh:38`). A pin given this way goes through `set_axis_range` with three equal values. `hb_subset_axis_range_from_string` (10.2.0) parses the range syntax for other programs.

fontTools’ documentation names four levels of instancing: dropping an axis at its default (L1), pinning it elsewhere (L2), restricting its range (L3), and moving its default (L4). HarfBuzz does all four, and they can be mixed in one call.

## Pinning an axis

`var-wght700.ttf` is `var.ttf` with `--variations=wght=700`. The weight axis has gone from `fvar`, the width axis remains, and of the four named instances only Bold, the one at weight 700, survives: an instance is kept only if it sits at the pinned value of every pinned axis and inside the range of every limited one (`hb-ot-var-fvar-table.hh:89`). `var-static700.ttf` pins width too, at its default, and has no variation tables left:

```info
variations/var.ttf  --list-variations
var-wght700.ttf  --list-variations
var-static700.ttf  --list-tables
```

What should come out is the original font as it looks at that position. Here is the original at weight 700, at two widths, and then the pinned font, which has no weight axis to set:

```draw
font: variations/var.ttf
text: IHV
variations: wght=700
at: wdth=75; wdth=100
```

```draw
font: var-wght700.ttf
text: IHV
at: wdth=75; wdth=100
```

The drawings and the advances under them agree, and so do the extents:

```shape
variations/var.ttf  IHV  --variations=wght=700,wdth=75  --show-extents
var-wght700.ttf  IHV  --variations=wdth=75  --show-extents
variations/var.ttf  IHV  --variations=wght=700  --show-extents
var-static700.ttf  IHV  --show-extents
```

They agree up to rounding. `hb-vector` draws the original’s `V` at weight 700 with points at fractional positions, such as x = 1386.41, because the [inferred](varying-outlines.html#points-that-arent-there-iup) and interpolated deltas are fractional; `glyf` stores integers, so the instance has 1386. A pinned instance is the original at that position rounded to the font’s grid, and drawings at any ordinary size can’t tell them apart.

When every axis is pinned, `_should_drop_table` (`hb-subset.cc:219`) drops `fvar`, `avar`, `gvar`, `HVAR`, `VVAR`, `MVAR` and `cvar` outright. Any other table loses its variation data as it is subset. A table that still has deltas after a partial pin, but none that vary, is dropped too: `var-wght700.ttf` has no `MVAR`, because only weight moved the x-height. In `OS/2`, `usWeightClass` and `usWidthClass` are set from the new default (`hb-ot-os2-table.hh:273`).

## Limiting an axis

`var-wght400-800.ttf` limits weight to 400–800, keeping the default at 400. `fvar` gets the new limits:

```table
font: var-wght400-800.ttf
table: fvar
```

The coordinates every variation table uses are normalized ([Design space](design-space.html#from-user-coordinates-to-normalized-ones)): −1 at the axis minimum, 0 at the default, +1 at the maximum. In the new font, +1 means weight 800, not 900. So the instancer must **renormalize** every coordinate that refers to the axis: positions in the old normalized space, such as 0.8 for weight 800, become positions in the new one, 1. A region that ran past the new maximum has to be rewritten, and a region that lies wholly outside the new range is dropped. That is the solver’s job, described two sections down.

The instance still varies like the original inside its range. At weights 600 and 775:

```shape
variations/var.ttf  IHV  --variations=wght=600
var-wght400-800.ttf  IHV  --variations=wght=600
variations/var.ttf  IHV  --variations=wght=775,wdth=80
var-wght400-800.ttf  IHV  --variations=wght=775,wdth=80
```

Outside it, the instance clamps, as any variable font does: asked for weight 900, it gives weight 800.

## What the plan works out

The plan ([Subsetting](subsetting.html#the-plan)) turns the user’s triples into everything the tables will need before any table is written. `normalize_axes_location` (`hb-subset-plan-var.cc:465`) runs first, and fills these members of the plan:

* `user_axes_location`: the user’s triples, in user coordinates. `fvar`, `STAT` and `OS/2` read these.
* `axes_location`: the same triples normalized, through `fvar` and then `avar`. Every table that rebases deltas reads these.
* `normalized_coords`: the new default, as a full set of old normalized coordinates. The deltas that are folded into the default values are evaluated here.
* `axes_index_map` and `axis_tags`: which axes survive, and their new order.
* `axes_triple_distances`: each limited axis’s old distances from default to minimum and to maximum, in user units, for renormalizing (below).
* `all_axes_pinned` and `pinned_at_default`, two flags that let most tables skip work.

The normalization isn’t quite the one shaping uses. Shaping normalizes in 16.16 fixed point, applies `avar`, and rounds to 2.14 last ([Design space](design-space.html#from-user-coordinates-to-normalized-ones)). The plan rounds to 2.14 *before* `avar`, and a comment explains why (`hb-subset-plan-var.cc:529`):

```c++
// TODO(behdad): Spec says axis normalization should be done in 16.16;
// We used to do it in 2.14, but that's not correct.  I fixed this in
// the fvar/avar code, but keeping 2.14 here for now to keep tests
// happy. We might need to adjust fonttools as well.
```

The tests are comparisons with fontTools’ output, and fontTools rounds the same way. The outlines don’t use this path: `glyf` makes its new default outlines with an `hb_font_t` given the user’s values through `hb_font_set_variations`, the same call `hb-shape --variations` makes, so a pinned outline is computed exactly as shaping would compute it (`OT/glyf/glyf.hh:722`). The research for this chapter pinned `var.ttf` and `var-avar.ttf` at sixty-two weights and compared advances, extents and x-heights with the originals’, and found no case where the two normalizations gave different results; with a steep `avar` segment and an unlucky value, a difference of one 2.14 unit is possible (inferred).

Some work is done per glyph in the plan as well. For a partial instance, `get_instance_glyphs_contour_points` (`:802`) reads every kept glyph’s default points, which `gvar` will need; for CFF2, `update_instance_metrics_map_from_cff2` (`:699`) works out each glyph’s new advance and side bearing from its instanced outline. The rest happens table by table, in the loop that [Subsetting](subsetting.html#writing-the-tables) describes, where `hmtx`, `vmtx`, `maxp` and `OS/2` wait for `glyf`, whose outlines give their numbers.

## How the solver rewrites a region

A delta applies over a **tent** on each axis: a start, a peak and an end, in normalized coordinates ([Deltas](deltas.html#regions-and-scalars)). When the axis range shrinks or the default moves, each tent must be re-expressed in the new coordinates so that, at every position the new font can reach, the deltas add up to what they did before. `rebase_tent` (`hb-subset-instancer-solver.cc:419`) does this for one tent on one axis. Its file begins:

```c++
/* This file is a straight port of the following:
 *
 * https://github.com/fonttools/fonttools/blob/f73220816264fc383b8a75f2146e8d69e455d398/Lib/fontTools/varLib/instancer/solver.py
```

It returns a list of pairs: a scalar to multiply the deltas by, and a new tent. A pair with no tent is the **gain**: the part of the delta that applies everywhere in the new font, which is the tent’s value at the new default. The gain is folded into the default values (into the outlines, the advances, the `OS/2` metrics) and the other pairs are kept as variations.

The core, `_solve`, first mirrors the problem so that the new default lies at or below the peak, then works through cases. Here is the part that matters for the example, trimmed:

```c++
float gain = supportScalar (axisDef, tent);
out.push(hb_pair (gain, Triple{}));

// outGain is the scalar of axisMax at the tent.
float outGain = supportScalar (axisMax, tent);
if (gain >= outGain)
  { /* Case 3a: the down-slope crosses the new maximum: split in two or three */ }
else
{
  /* Case 4: New limit doesn't fit; we need to chop into two tents,
   * because the shape of a triangle with part of one side cut off
   * cannot be represented as a triangle itself. */
  Triple loc1 {hb_max (axisDef, lower), peak, axisMax};
  float scalar1 = 1.f;
  Triple loc2 {peak, axisMax, axisMax};
  float scalar2 = outGain;
  out.push (hb_pair (scalar1 - gain, loc1));
  // Don't add a dirac delta!
  if (peak < axisMax)
    out.push (hb_pair (scalar2 - gain, loc2));
}
```

`var.ttf` has three tents on the weight axis: (−1, −1, 0) for the light master, (0, 0.5, 1) for the intermediate master at 650, and (0.5, 1, 1) for the heavy one. Limiting weight to 400–800 makes the axis limit (0, 0, 0.8) in old coordinates. Take the intermediate tent:

| Step | Arithmetic |
|---|---|
| value at the new default, 0 | the tent starts there: gain = 0 |
| value at the new maximum, 0.8 | on the down-slope: outGain = (1 − 0.8) / (1 − 0.5) = 0.4 |
| gain < outGain: case 4 | (0, 0.5, 0.8) × 1, and (0.5, 0.8, 0.8) × 0.4 |
| the negative side | below the default there is nothing left: a pair with scalar 0, which is dropped |
| renormalize, dividing by 0.8 | (0, 0.625, 1) × 1, and (0.625, 1, 1) × 0.4 |

The old tent, cut off at 0.8, is no longer a triangle: it rises to 1 at 0.5 and falls only to 0.4 at the edge. The sum of a full tent that falls to 0 at the edge and a second one that rises from 0 to 0.4 follows the same line. The heavy master’s tent, (0.5, 1, 1), peaks beyond the new maximum, so it takes case 2: it is scaled by its value at 0.8, (0.8 − 0.5) / 0.5 = 0.6, and becomes (0.5, 0.8, 0.8), which renormalizes to (0.625, 1, 1). The light master’s tent lies wholly below the new default and is dropped (case 1).

Two of the new tents are the same, (0.625, 1, 1), and `merge_tuple_variations` (`hb-ot-var-common.hh:1179`) adds them: 0.4 of the intermediate master’s deltas plus 0.6 of the heavy one’s. `V`’s top-right point moves 160 units at the intermediate master and 300 at the heavy one, so the merged delta is 0.4 × 160 + 0.6 × 300 = 244:

```table
font: var-wght400-800.ttf
table: gvar
glyph: V
```

The same regions appear in `HVAR`, whose advances are rewritten the same way: `V`’s is item 4, 160 and 244 again, since its advance grows as its corner does. At the new maximum the instance gives 0.4 × 160 + 0.6 × 300, which is what the original gave at 0.8.

Two details of the port. The case the code calls “case 3”, which would keep a cut tent as one triangle by stretching its far end beyond +1, is disabled with `if (false && …)` (`:261`), because the OpenType Sanitizer rejected the result (“ots doesn’t like us”, citing fontTools issue 3350); the solver always splits instead. And when the new default moves across the old one, renormalizing has to measure in user units, because the two sides of the old default had different scales: in `var.ttf`, normalized −1 is 300 weight units from the default and +1 is 500. `renormalizeValue` (`:369`) uses `axes_triple_distances` for that case. The solver works in `float`, where fontTools uses Python’s double precision; in the larger fonts compared below, a few deltas came out one unit apart, probably for that reason (inferred).

`change_tuple_variations_axis_limits` (`hb-ot-var-common.hh:1131`) runs the solver axis by axis, in order of tag, over every tuple; a tuple whose tent crosses zero or is out of order, which shaping would treat as constant, is dropped instead (`:390`).

## Outlines and gvar

The new default outlines are made by drawing each kept glyph at the new default, with its variations applied, and writing the points back into `glyf` (`OT/glyf/Glyph.hh:239`). The four phantom points come along, and `update_mtx` (`:139`) turns them into the glyph’s new advance and side bearings for `hmtx` and `vmtx`. The glyph’s bounding box, and the font-wide maxima in `head` and `maxp`, are recomputed from the same points. This drawing is done for every kept glyph, and composite components are drawn again for each glyph that uses them, so the whole table shares one budget of work, `HB_BUDGET_SUBSET_GLYF`, 2²⁸ units, across retries too (`hb-limits.hh:250`, `hb-subset.hh:64`); running out fails the table. Its comment says the largest variable font in HarfBuzz’s tests uses about 1.2 million.

For a partial instance, `gvar` is then rebuilt (`hb-ot-var-gvar-table.hh:142`). Each glyph’s tuples are decompiled, rebased as above, and merged. Before merging, `calc_inferred_deltas` (`hb-ot-var-common.hh:662`) fills in every point a tuple left out, by the same [IUP](varying-outlines.html#points-that-arent-there-iup) interpolation a renderer would use: two tuples with different sets of points can only be added point by point. The result is a `gvar` with every delta written out, even in tuples the instancing didn’t change. The width tuple of `V` above had three explicit points in `var.ttf`; it now has eleven.

`HB_SUBSET_FLAGS_OPTIMIZE_IUP_DELTAS` (8.5.0; `--optimize` in `hb-subset`) runs the reverse: the IUP optimizer, a port of fontTools’ `iup.py` (`hb-subset-instancer-iup.cc:31`), finds the fewest points whose deltas, interpolated, reproduce all the others within half a unit. `var-wght400-800-iup.ttf` was made with it:

```table
font: var-wght400-800-iup.ttf
table: gvar
glyph: V
```

The flag is off by default, where fontTools’ instancer has it on. It costs time, and it works one contour at a time: a contour of more than 512 points keeps all its deltas (`hb-subset-instancer-iup.cc:364`).

> For programmers: instancing a font without `OPTIMIZE_IUP_DELTAS` can make its `gvar` bigger than the original’s, though the font has less variation in it. If size matters, set the flag.

## The other tables

Most other variation data lives in [item variation stores](deltas.html#item-variation-stores), and `item_variations_t::instantiate` (`hb-ot-var-common.hh:1851`) handles them all the same way: each `VarData` row becomes a set of tuples, one per region, which are rebased and merged like `gvar`’s, and then packed back into rows, with a port of fontTools’ row-encoding optimizer choosing how. Unused regions are pruned. What becomes of the gain depends on the table:

* **HVAR and VVAR.** The new advances are already in `hmtx` and `vmtx`, from the outlines, so the gain is dropped.
* **MVAR.** `OS/2`, `hhea`, `vhea` and `post` add each metric’s delta at the new default to the stored value.
* **GDEF, GPOS and BASE.** The plan evaluates every variation index used by a kept value record or anchor, and makes a map from old index to (new index, delta) (`remap_variation_indices`, `hb-subset-plan-var.cc:384`). Each value is serialized with its delta added, and its device table pointed at the new index, or dropped when nothing varies anymore.
* **COLR.** COLR v1’s variable paints get the same treatment through a map of their own.
* **cvar.** `cvar` is rebased like `gvar`, and its gain is added to `cvt ` whenever the default moves (`hb-subset.cc:347`), for partial instances as well as full ones.

`STAT` keeps its axis records and drops axis values outside the new ranges (`hb-ot-stat-table.hh:60`).

## avar

An `avar` segment map is rewritten by the same renormalization (`hb-ot-var-avar-table.hh:263`). Its “from” side is in the new font’s default normalization and its “to” side in the new mapped space, so each is renormalized against its own range, and the three fixed mappings (−1, 0, +1) are added back. `var-avar.ttf` maps weight 600 (normalized 0.4) to the design at 750 (0.7). Limited to 400–800, weight 600 is 0.5 in the new font, and the new maximum, which `avar` maps to 0.9, becomes 1, so 0.7 becomes 0.7 / 0.9 = 0.7778:

```table
font: var-avar-wght400-800.ttf
table: avar
```

The `gvar` tents are renormalized in the mapped space, so the intermediate master’s peak is now 0.5 / 0.9 = 0.5556. Rounding in this chain can move a coordinate by one 2.14 unit and an advance by one unit. At weight 570:

```shape
variations/var-avar.ttf  I  --variations=wght=570
var-avar-wght400-800.ttf  I  --variations=wght=570
```

fontTools’ instance of the same font gives 369 there: it stores the mapping as from 0.50006, not 0.5, because it renormalizes the 2.14 value of 0.4 against the 2.14 value of weight 800, 0.79999, where HarfBuzz unmaps the mapped limit through `avar` and gets 0.8. The function’s own comment calls this approach temporary: “unmap_axis_range needs to be killed.”

### avar 2

[avar 2](design-space.html#avar-version-2) is supported, not refused. Full instancing has worked since 12.3.0 and partial instancing since 14.3.0 (NEWS), with fixes in 14.4.0. It is the hardest case, because an `avar` 2 table can make one axis’s final coordinate depend on others: in `var-avar2.ttf`, the weight that is drawn depends on the width. Pinning weight can’t simply bake its contribution in, since what weight 700 means still changes with width.

So for a partial instance of an `avar` 2 font, HarfBuzz follows a fontTools design called offset compensation (described in `AVAR2-INSTANCING.md` at the top of the source tree). The plan normalizes through the `avar` 1 maps only (`hb-subset-plan-var.cc:572`) and rebases the `avar` 2 variation store, adding deltas so that the final coordinates stay in the *old* coordinate space. A pinned axis stays in `fvar`, hidden, with minimum, default and maximum all at the pinned value, and an identity segment map; the other tables keep their variations unchanged, apart from regions the instance can never reach, which are dropped. Only a **self-contained** pinned axis, one whose final coordinate the plan can prove is constant, is removed and baked in like an ordinary pin (`:322`). Weight in `var-avar2.ttf` isn’t one:

```info
var-avar2-wght700.ttf  --list-variations
```

```shape
variations/var-avar2.ttf  IHV  --variations=wght=700,wdth=80
var-avar2-wght700.ttf  IHV  --variations=wdth=80
```

fontTools 4.68 makes the same choice, a hidden axis.

## Feature variations

GSUB and GPOS can swap a feature’s lookups in part of the design space ([From features to lookups](from-features-to-lookups.html)). `var-rvrn.ttf`, built by `fonts/instancing/build-instancing-fonts.py`, is `var.ttf` with one more glyph, `I.serif`, and an `rvrn` feature variation that substitutes it for `I` from weight 600 up, as a font might change a dollar sign’s design at heavy weights. HarfBuzz turns `rvrn` on by itself. `var-rvrn-wght700.ttf` pins weight at 700, and `var-rvrn-500-700-900.ttf` limits it to 500–900 and moves the default to 700, inside the condition:

```shape
var-rvrn.ttf  I  --variations=wght=550
var-rvrn-500-700-900.ttf  I  --variations=wght=550
var-rvrn.ttf  I  --variations=wght=650
var-rvrn-500-700-900.ttf  I  --variations=wght=650
var-rvrn-wght700.ttf  I
```

No HarfBuzz tool prints feature variations, so `build.sh` writes a summary with fontTools (`fonts/instancing/featvars-summary.txt`):

```
var-rvrn.ttf
  rvrn: no lookups
  record 0, when wght 0.400024 to 1: rvrn -> lookup 0
var-rvrn-wght700.ttf
  rvrn: lookup 0
  record 0, when always: rvrn -> lookup 0
var-rvrn-500-700-900.ttf
  rvrn: lookup 0
  record 0, when wght -0.499939 to 1: rvrn -> lookup 0
  record 1, when always: rvrn -> no lookups
```

Each condition is checked against the new limits (`hb-ot-layout-common.hh:4374`): one that can never be met drops its record; one that is always met on a pinned axis is dropped from the record; any other is renormalized. Then the record that applies at the new default, if any, has its substitute features copied into the feature list itself. That changes what applies where no record does, so a **catch-all record**, with no conditions, is appended to restore the original features there. In the limited font, record 0 still swaps in the serif from 600 up, and below it the catch-all gives `rvrn` back its empty lookup list. In the pinned font, width is still variable, so the record survives with its condition gone: redundant, but harmless.

At weight 600 exactly, the original and the limited font disagree:

```shape
var-rvrn.ttf  I  --variations=wght=600
var-rvrn-500-700-900.ttf  I  --variations=wght=600
```

The condition’s minimum, 0.4, is stored in 2.14 as 0.400024, which is weight 600.012. In the original, weight 600 normalizes to 0.4, which rounds to the same 2.14 value, and the condition holds. In the instance, 600 is exactly −0.5 and the renormalized minimum is −0.499939, so it doesn’t. fontTools’ instance has the same GSUB, and does the same. Thresholds that sit exactly on a value users ask for, as this one does, are worth testing after instancing.

## CFF2 and VARC

CFF2 outlines keep their deltas inside the charstrings, as `blend` operators. Instancing always desubroutinizes them (`hb-subset-cff2.cc:1185`). A full instance evaluates every blend; `HB_SUBSET_FLAGS_DOWNGRADE_CFF2` (13.0.0) then writes a CFF table instead, for renderers that can’t read CFF2. Partial instancing, added in 14.3.0, is a port of fontTools’ `instantiateCFF2` with one simplification its comment describes (`:44`): it rebases each `VarData`’s regions once, as a linear map, and rewrites every blend through it. Since 15.1.0, a CFF2 font with nested blends makes partial instancing fail, where before it changed the outlines.

A [VARC](variable-components.html) table can’t be instanced on any axis its components use: the component coordinates and their variation data would need rewriting (`OT/Var/VARC/VARC-subset.hh:101`). Since 15.0.0, axes VARC doesn’t use can be pinned or limited. Pinning weight in `varc-ac00-ac01.ttf` makes `hb-subset` fail, unless VARC is dropped.

## Compared with fontTools

The research for this chapter instanced the same fonts with `hb-subset` and with fontTools 4.68’s instancer (unoptimized deltas on both sides unless stated): the fonts on this page, and from HarfBuzz’s tests a Roboto and a Roboto Flex subset, a Muli subset with `cvar`, two CFF2 fonts, and a COLR v1 font, with pins and limits. Apart from the subsetter’s own defaults (HarfBuzz empties `.notdef` and drops Macintosh `name` records unless asked), most tables came out identical, and where they didn’t, the outlines and metrics agreed. These were the differences:

| Where | HarfBuzz | fontTools |
|---|---|---|
| IUP optimization | off unless `OPTIMIZE_IUP_DELTAS`; with it, the same `gvar` | on by default |
| full instances | `glyf` overlap flags left alone (`SET_OVERLAPS_FLAG` sets them) | sets `OVERLAP_SIMPLE` on every simple glyph |
| a variation table left with no variation | dropped (`HVAR`, `MVAR`) | kept, empty |
| `avar` maps after limiting | can differ by one 2.14 unit (above) | |
| `gvar` deltas | a few one unit apart in Roboto Flex (`float` arithmetic, inferred) | double precision |
| CFF2 partial instances | `hmtx` side bearings recomputed from the new outlines | the old default’s kept |
| `avar` 2, weight pinned in `var-avar2.ttf` | one more, unreachable region kept in `MVAR` | |
| GPOS value formats | device-table bits with nothing left dropped | kept |

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-subset-input.cc` | pinning and range functions, `hb_subset_axis_range_from_string` |
| `util/helper-subset.hh` | `hb-subset --variations` |
| `src/hb-subset-plan-var.cc` | `normalize_axes_location`, avar 2 reachable ranges and self-contained axes, variation-index maps, CFF2 metrics |
| `src/hb-subset-instancer-solver.{hh,cc}` | `Triple`, `rebase_tent`, `renormalizeValue` |
| `src/hb-ot-var-common.hh` | `tuple_delta_t`, tuple variations (rebasing, inferred deltas, merging), `item_variations_t` |
| `src/hb-subset-instancer-iup.{hh,cc}` | the IUP optimizer |
| `src/hb-ot-var-gvar-table.hh`, `src/OT/glyf/glyf.hh`, `src/OT/glyf/Glyph.hh` | `gvar` rebuilding; new default outlines and metrics |
| `src/hb-ot-var-fvar-table.hh`, `src/hb-ot-var-avar-table.hh`, `src/hb-ot-stat-table.hh` | axes, named instances, segment maps, avar 2, STAT |
| `src/hb-ot-layout-common.hh` | conditions and feature variations under new limits |
| `src/hb-subset-cff2.cc`, `src/OT/Var/VARC/VARC-subset.hh` | CFF2 instancing; VARC’s refusal |
| `src/hb-subset.cc`, `src/hb-limits.hh` | dropped tables, table order, `cvt `; `HB_BUDGET_SUBSET_GLYF` |
| `AVAR2-INSTANCING.md` | the design of avar 2 partial instancing |

## In short

* Instancing is part of subsetting: axis settings go in the subset input, and each table instances itself as it is subset.
* Pinning removes an axis and bakes its position into the default; limiting keeps it with a new range, and perhaps a new default, so every coordinate that refers to it is renormalized.
* The solver, ported from fontTools, rewrites each tent for the new limits: a gain folded into the default, plus new tents. A tent cut off by the new range becomes two.
* New default outlines are drawn with shaping’s own normalization, so a pinned outline matches the original up to rounding to integers, and `hmtx`, `head` and `maxp` follow from them. Other tables normalize in 2.14 first, as fontTools does.
* Rebuilt `gvar` deltas are written out in full unless `OPTIMIZE_IUP_DELTAS` is set.
* avar 2 is supported, fully and partially; a pinned axis usually stays as a hidden axis.
* Feature variations are rewritten, with the default’s substitutions copied into the feature list and a catch-all record to undo them elsewhere.
* VARC axes can’t be instanced, and CFF2 with nested blends can’t be partially instanced.
* Against fontTools, the results are the same or within a unit, with different defaults for IUP optimization and overlap flags.
