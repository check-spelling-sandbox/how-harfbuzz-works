# Deltas

A position in the design space, as [the previous chapter](design-space.html) showed, is a list of normalized coordinates. What a variable font stores is not the font at every position but the font at its default, plus **deltas**: numbers to add, each tied to a part of the design space where it applies, and weighted by how far into that part the position is. This chapter is about how HarfBuzz finds those weights and adds up the deltas. The machinery is the same everywhere in a variable font, with one exception that gets its own chapter, the glyph outlines in `gvar`.

## Regions and scalars

A delta belongs to a **region**: for each axis, a start, a peak and an end, in normalized coordinates. At the peak, the delta counts in full. Between the start and the peak, and between the peak and the end, it counts in proportion. Outside the start and end, it doesn’t count at all. On one axis, that weight, the **scalar**, is a tent. On several, it’s the product of the tents, so a region that peaks at (+1, +1) counts in full only at that corner.

Here is the tent, `VarRegionAxis::evaluate` (`hb-ot-layout-common.hh:2549`):

```c++
float evaluate (int coord) const
{
  int peak = peakCoord.to_int ();
  if (peak == 0 || coord == peak)
    return 1.f;
  else if (coord == 0) // Faster
    return 0.f;

  int start = startCoord.to_int (), end = endCoord.to_int ();

  /* TODO Move these to sanitize(). */
  if (unlikely (start > peak || peak > end))
    return 1.f;
  if (unlikely (start < 0 && end > 0))
    return 1.f;

  if (coord <= start || end <= coord)
    return 0.f;

  /* Interpolate */
  if (coord < peak)
    return float (coord - start) / (peak - start);
  else
    return float (end - coord) / (end - peak);
}
```

The comparisons are on 2.14 integers, and there is one division, in floating point. A peak of 0 means the region doesn’t depend on this axis, so the factor is 1. The checks for a region whose start, peak and end are out of order, or that crosses 0, look like robustness code, but they are exactly what the specification says to do: such an axis is ignored, which means a factor of 1, not 0. The `TODO` is still there: those checks are made every time a region is evaluated, not once when the table is loaded.

The test font `var.ttf` has masters at weights 100, 400, 650 and 900, and at width 75. Its HVAR table, which holds the deltas for advances, has four regions: one for the light master, peaking at −1; one for each of the two heavier masters; and one for the narrow master.

```table
font: variations/var.ttf
table: HVAR
```

The 650 master is an **intermediate** one: its region on the weight axis starts at 0, peaks at 0.5 and ends at 1, rather than running all the way from 0 to the end. Without it, the advances would change in one straight line from 400 to 900. With it, there are two:

```shape
variations/var.ttf  I  --variations=wght=400
variations/var.ttf  I  --variations=wght=525
variations/var.ttf  I  --variations=wght=650
variations/var.ttf  I  --variations=wght=775
variations/var.ttf  I  --variations=wght=900
```

`I` is glyph 3, so its deltas are item 3 in the store’s only `VarData`: 100 for the (0, 0.5, 1) region and 150 for the (0.5, 1, 1) one, among others. At weight 775, normalized 0.75, the first region’s tent has come halfway down and the second’s has gone halfway up, so the delta is 0.5 × 100 + 0.5 × 150 = 125, and the advance is 260 + 125 = 385.

## Item variation stores

Regions and deltas are kept in an **item variation store**, a structure shared by every variation table except gvar. It has a list of regions, and one or more `VarData` tables of rows. Each row is one **item**, such as one glyph’s advance or one metric, with a delta for each region the `VarData` uses. To save space, the first columns of a row can be 16-bit and the rest 8-bit, or, with the `LONG_WORDS` bit, 32-bit and 16-bit. An item is addressed by two numbers, *outer* (which `VarData`) and *inner* (which row).

`VarData::_get_delta` (`hb-ot-layout-common.hh:3073`) walks a row in its runs of long, short and byte values, multiplying each by its region’s scalar and skipping regions whose scalar is 0. A `VarData` with no regions returns at once, with a comment that “This is quite common.”

### No rounding in the store

The store returns a `float`, and never rounds. Rounding is left to whoever asked, and they don’t all do it the same way:

* Advances, from HVAR or VVAR (`hb-ot-hmtx-table.hh:353`) and from phantom points (`OT/glyf/glyf.hh:492`), are rounded **in font units**, before the font’s size is applied.
* MVAR metrics, device tables and VORG are scaled first and rounded after.

So an advance is always a whole number of font units, even when the text is enormous. At a weight where `I`’s exact advance is 345.2 units, a font size 100 times its units per em gives 34500, not 34520:

```shape
variations/var.ttf  I  --variations=wght=613  --font-size=100000
```

The specification only recommends rounding as late as possible. Most of the time, the difference is too small to see. But it means the spacing of very large text moves in steps of whole font units as a slider moves, and it means that HarfBuzz’s advances and positioning deltas aren’t rounded at the same stage.

The addition itself is in floating point, too: the advance from `hmtx` plus the rounded delta. A float has 24 bits of precision, so for huge values the sum is inexact. A HarfBuzz test font with deltas of two billion shows it: an advance that should be 500,000,500 comes out as 500,000,512, and one that would overflow comes out as −1, which an API test now asserts.

### The scalar cache

A glyph’s advance usually depends on only a few regions, but a run of text asks for many glyphs, at the same position. So HarfBuzz keeps a cache of region scalars, `hb_scalar_cache_t` (`hb-ot-layout-common.hh:2637`): one float per region, with NaN meaning “not worked out yet”. The values are stored as relaxed atomics. A scalar depends only on the region and the position, so two threads that fill in the same entry at once write the same number, and no lock is needed.

That matters for GPOS. The cache used for GDEF’s store, which holds the deltas for GPOS device tables and anchors, is the OpenType shaper’s per-font data (`hb-ot-shape.cc:433`), shared by every shaping call on that font at once. The caches in HarfBuzz’s font functions, for advances and outlines, are handed to one caller at a time and thrown away when the position changes, as the previous chapter showed.

If memory runs out, `create` returns the shared empty cache, and an empty cache answers 0 for every region. Variations quietly disappear: the font behaves as if it were at its default.

### Index maps

A table that needs many items can address them through a **delta-set index map**, `DeltaSetIndexMap` (`hb-ot-layout-common.hh:4015`), which maps item numbers (glyph IDs, for HVAR) to outer and inner indices. HarfBuzz follows the specification where it says what to do: an index past the end of the map uses the last entry (`if (v >= mapCount) v = mapCount - 1;`). Where it doesn’t say, HarfBuzz chooses to keep going: a map with no entries, and a map in a format HarfBuzz doesn’t know, pass the index through unchanged, and an unknown format even passes the sanitizer.

Without a map, HVAR’s glyph *n* is outer 0, inner *n*. `get_advance_delta_unscaled` (`hb-ot-var-hvar-table.hh:436`) has a guard for fonts with more than 65,536 glyphs: “An oversized glyph id must not select another table.” Without it, glyph 65,536 would become outer 1, inner 0.

## Advances: HVAR or the outlines

A variable glyph’s advance can come from two places: from HVAR, or from the outlines, since gvar varies a glyph’s **phantom points**, the points that mark its advance, along with its real ones (see [Varying outlines](varying-outlines.html)). The specification says HVAR is optional for TrueType fonts but makes them faster, and is required for CFF2 fonts, which have no phantom points.

HarfBuzz’s choice is in `hb_ot_get_glyph_h_advances` (`hb-ot-font.cc:382`):

1. At the default position, `hmtx` alone, without looking at any variation table.
2. Otherwise, if there is HVAR, `hmtx` plus HVAR’s delta.
3. Otherwise, if there is gvar, the distance between the glyph’s varied phantom points.
4. Otherwise, `hmtx` alone. A CFF2 font without HVAR doesn’t vary its advances at all.

Vertical advances follow the same steps with VVAR, in a copy of the function whose comment says “Duplicated from h_advances. Ugly. Keep in sync'ish.”

`var-nohvar.ttf` is `var.ttf` with HVAR removed, so its advances come from the phantom points. They agree, as they should, since varLib made both from the same masters:

```shape
variations/var.ttf  IHV  --variations=wght=900
variations/var-nohvar.ttf  IHV  --variations=wght=900
```

But nothing makes a font’s HVAR and gvar agree, and HarfBuzz doesn’t check. `var-badhvar.ttf` has every HVAR delta doubled. Its outlines are the same as `var.ttf`’s, from gvar, but its advances come from HVAR:

```shape
variations/var.ttf  IHV  --variations=wght=900
variations/var-badhvar.ttf  IHV  --variations=wght=900
```

```draw
font: variations/var-badhvar.ttf
text: IHV
slider: wght 100 900 50
start: 400
```

At the default the font is fine, because no deltas are applied. Away from it the glyphs drift apart, or into each other. HarfBuzz trusts HVAR, which is what the specification intends: HVAR exists so that a shaper doesn’t have to touch the outlines. The research for this chapter found the same thing in a real font, one of HarfBuzz’s test fonts, `SourceSerifVariable-Roman-VVAR.abc.ttf`, whose VVAR and vertical phantom points disagree by up to 54 units at its heaviest weight. HarfBuzz uses VVAR.

HVAR can also hold maps for left and right side bearings, and VVAR for top and bottom ones. HarfBuzz never reads them when shaping, only when subsetting. It doesn’t need them, since it gets glyph extents from the varied outlines themselves.

The cache for advances has one more limit. `hb_ot_font_advance_cache_t` is `hb_cache_t<24, 16>` (`hb-ot-font.cc:65`): 256 entries, each holding a glyph ID of up to 24 bits and an advance of up to 16 bits, in font units. An advance of 65,536 units or more is simply never cached.

## Metrics: MVAR

Font-wide values vary too: the ascender and descender, the x-height and cap height, the underline’s position and thickness, the caret slope. MVAR lists them by four-letter tag, each with an item in its store. The test fonts vary only the x-height, from 480 at the lightest weight to 530 at the heaviest:

```table
font: variations/var.ttf
table: MVAR
```

```info
variations/var.ttf  --get-metric=xhgt  --variations=wght=100
variations/var.ttf  --get-metric=xhgt  --variations=wght=400
variations/var.ttf  --get-metric=xhgt  --variations=wght=900
```

The tags of `hb_ot_metrics_tag_t`, the public API for these values, *are* the MVAR tags, so `hb-ot-metrics.cc` can look the variation up directly. There is one difference from the specification. MVAR’s `hasc`, `hdsc` and `hlgp` tags vary the typographic ascender, descender and line gap in the OS/2 table. HarfBuzz reports the `hhea` values instead when the font doesn’t set OS/2’s `USE_TYPO_METRICS` flag, and it applies the same MVAR deltas to them (`hb-ot-metrics.cc:85`). The research for this chapter checked it on a patched test font: an `hhea` ascender of 1000 with a `hasc` delta of 765 gave 1765.

## Everyone else

The same store, read the same way, serves almost every other variable table:

| Table | What varies |
|---|---|
| GDEF | GPOS value records and anchors, ligature carets, through device tables; also MATH’s values, since MATH has no store of its own |
| GSUB and GPOS FeatureVariations, version 1.1 | the values that conditions are compared with, through GDEF’s store |
| BASE | baseline values |
| COLR version 1 | colors, positions, and transforms of paints |
| avar version 2 | the axes themselves (see [Design space](design-space.html)) |
| CFF2 | blends in outlines, which use only the store’s region scalars, not its deltas |

VARC uses a variant, the **multiple item variation store**, whose items are lists of deltas rather than single numbers, and whose regions name only the axes they use. It is in the 2026 ISO standard, not in Microsoft’s 1.9.1, and it comes up again in [Variable components](variable-components.html). One detail belongs here. An ordinary store applies no deltas at the default (`ItemVarStoreInstancer`: “treat explicit all-zero coordinates as the default instance, which uses unadjusted values (OFF 7.1)”, `hb-ot-layout-common.hh:4219`). The multiple store does, because a region that names no axes has a scalar of 1 everywhere, even at the default.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-ot-layout-common.hh` | the stores: `VarRegionAxis`, 2549; `hb_scalar_cache_t`, 2637; `VarRegionList`, 2728; `VarData`, 3054; `ItemVariationStore`, 3553; `MultiItemVariationStore`, 3858; `DeltaSetIndexMap`, 4015 |
| `src/hb-ot-var-hvar-table.hh` | HVAR and VVAR |
| `src/hb-ot-var-mvar-table.hh`, `src/hb-ot-metrics.cc` | MVAR, and the metrics API |
| `src/hb-ot-font.cc` | where advances come from, 382 and 506; the caches |
| `src/hb-ot-hmtx-table.hh` | `hmtx` plus HVAR, and where the advance is rounded |

## In short

* A delta’s weight is the product of a tent on each axis. Regions that are malformed count as 1, which is what the specification asks for.
* The store returns unrounded floats. Advances are rounded in font units, before scaling; metrics and device deltas after.
* Advances come from HVAR when there is one, and from gvar’s phantom points otherwise. HarfBuzz trusts HVAR even when it disagrees with the outlines.
* MVAR’s ascender, descender and line gap deltas are also applied to `hhea`’s values.
* Region scalars are cached per font and position, in atomics, so shaping calls can share them without locks.
