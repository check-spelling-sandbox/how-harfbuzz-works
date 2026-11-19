# Fallbacks

A font doesn’t have to have GPOS. Plenty of fonts in use have no layout tables at all, or a GSUB and no GPOS, or a `kern` table from the 1990s and nothing else. HarfBuzz still has to put the accent somewhere, give an em space its width and, where it can, kern. It does each of these itself, from Unicode properties and the font’s metrics, when the font doesn’t.

This chapter is about those **fallbacks** inside the `ot` backend: what HarfBuzz does in place of a table the font lacks. There are four. Marks are placed over and under their bases from glyph extents (steps 10 and 20 in [The pipeline, step by step](the-pipeline-step-by-step.html)). Marks lose their advance, and are moved back to make up for it (step 18). Space characters the font has no glyph for get suitable widths (step 15). And kerning can come from the font funcs (step 17). They are not the `fallback` *backend*, which maps characters to glyphs and does nothing else ([hb_shape and its backends](hb_shape-and-its-backends.html#fallback)). Two other fallbacks belong to script shapers: Arabic shaped from the presentation forms in the `cmap` when GSUB lacks the joining features is in [Joining scripts](joining-scripts.html), and Thai shaped with the private-use glyphs of old Thai fonts is in [Hebrew, Thai, Lao and Hangul](hebrew-thai-lao-and-hangul.html).

The examples use a test font, `fonts/fallbacks/fallback.ttf`, with no GDEF, no GPOS and no `kern`, and variants of it with one table added. Unlike the other chapters’ fonts, its marks have outlines, drawn where a designer might draw them, because fallback positioning works from the boxes around glyphs’ ink. Its bases are boxes of different heights: `o` is 500 units tall, `l` and `f` 750. Every result on this page was produced by HarfBuzz {{hb}} when the page was built.

## Who decides

Which fallbacks run is decided once, when the shape plan is made, in `hb_ot_shape_planner_t::compile` (`hb-ot-shape.cc:106`). Here are its decisions about positioning, trimmed of the `#ifdef`s that leave out AAT and `kern` in smaller builds:

```c++
bool disable_gpos = plan.shaper->gpos_tag &&
                    plan.shaper->gpos_tag != plan.map.chosen_script[1];
bool has_gpos = !disable_gpos && hb_ot_layout_has_positioning (face);

if (has_kerx && !(has_gsub && has_gpos)) plan.apply_kerx = true;
else if (has_gpos)                        plan.apply_gpos = true;

if (!plan.apply_kerx && (!has_gpos_kern || !plan.apply_gpos))
{
  if (has_kerx) plan.apply_kerx = true;
  else if (hb_ot_layout_has_kerning (face))
    plan.apply_kern = script_fallback_position; // Not all shapers apply legacy `kern`
}

plan.apply_fallback_kern = script_fallback_position &&
                           !(plan.apply_gpos || plan.apply_kerx || plan.apply_kern);

plan.zero_marks = script_zero_marks && !plan.apply_kerx &&
                  (!plan.apply_kern || !hb_ot_layout_has_machine_kerning (face));

plan.adjust_mark_positioning_when_zeroing = !plan.apply_gpos && !plan.apply_kerx &&
                  (!plan.apply_kern || !hb_ot_layout_has_cross_kerning (face));

plan.fallback_mark_positioning = plan.adjust_mark_positioning_when_zeroing &&
                                 script_fallback_position;

if (plan.apply_morx)
  plan.adjust_mark_positioning_when_zeroing = false;
```

Four plan flags come out of it, besides the choice of table:

| Flag | Means | True when |
|---|---|---|
| `zero_marks` | marks lose their advance (step 18, or before GPOS) | the shaper zeroes marks, and there is no `kerx` and no state-machine `kern` |
| `adjust_mark_positioning_when_zeroing` | a zeroed mark is moved back by its old advance | no GPOS, no `kerx`, no cross-stream `kern`, no `morx` |
| `fallback_mark_positioning` | marks are recategorized (step 10) and placed from extents (step 20) | the same, `morx` aside, and the shaper allows it |
| `apply_fallback_kern` | the font funcs are asked for kerning (step 17) | the shaper allows it, and there is no GPOS, `kerx` or `kern` to apply |

Two settings of the script shaper feed these: `zero_width_marks` and `fallback_position` ([Choosing a script shaper](choosing-a-script-shaper.html#what-the-choice-decides)). They are copied into `script_zero_marks` and `script_fallback_position` when the planner is made (`hb-ot-shape.cc:95`), before an AAT font’s shaper is swapped for `dumber`. Read from the shapers’ source:

| Shaper | Zeroes marks | `fallback_position` |
|---|---|---|
| default | late | yes |
| Arabic, Hebrew | late | yes |
| Hangul | no | yes |
| Thai (and Lao) | late | no |
| USE, Myanmar | early | no |
| Indic, Khmer, Zawgyi | no | no |
| `dumber` | no | no |

So a font gets fallback mark positioning only if it has no GPOS at all. A GPOS that holds nothing but kerning turns it off as surely as one full of anchors: what is tested is whether the table has data (`hb_ot_layout_has_positioning`), not whether it has a `mark` feature. A `kern` table leaves it on, unless the table has cross-stream kerning, which moves glyphs up and down and so might be placing marks itself. Here is the same text in `fallback.ttf` and in three variants: one with a GPOS holding a single kerning pair, `o o`, and no `mark` feature; one with that pair in a `kern` table; and one whose `kern` subtable is marked cross-stream:

```shape
fallback.ttf  oo\u0301  --language=en
with-gpos.ttf  oo\u0301  --language=en
kern-table.ttf  oo\u0301  --language=en
cross-kern.ttf  oo\u0301  --language=en
```

Only `fallback.ttf` and `kern-table.ttf` place the acute. In the other two it keeps the position its outline gives it, with no advance.

A trace shows the fallback running, as “start fallback mark”, after GSUB and after the advances are in place:

```trace
font: fallback.ttf
text: o\u0301
options: --language=en
from: end table GSUB
```

**Hebrew’s GPOS tag.** The Hebrew shaper is the only one with a `gpos_tag`. If GPOS chose a script other than `hebr`, as a font with only `DFLT` does, `has_gpos` is false and the whole GPOS is ignored, kerning and all. Fallback mark positioning takes over, and a `kern` table, if there is one, or fallback kerning takes the GPOS kerning’s place. [Choosing a script shaper](choosing-a-script-shaper.html#hebrew-and-its-gpos-tag) shows a hiriq losing its anchor this way.

**AAT fonts.** A font shaped with `morx` gets the `dumber` shaper in place of any but the default one, and `dumber` neither zeroes marks nor allows fallback positioning. But the plan never reads `dumber`’s `fallback_position`: it uses the original shaper’s, copied before the swap. So an Arabic run in a `morx` font with no GPOS or `kerx` still has its marks placed from extents. `with-morx.ttf` is `fallback.ttf` with a `morx` table, whose one subtable changes `period` into `zero` and doesn’t touch this text:

```shape
fallback.ttf  \u0628\u064E  --language=ar
with-morx.ttf  \u0628\u064E  --language=ar
with-morx.ttf  \u0E01\u0E48\u0E01  --language=th
```

The fatha is placed the same with `morx` as without. Thai, whose shaper doesn’t allow fallback positioning, gets `dumber`’s settings: the mark isn’t zeroed and keeps its 200-unit advance. What a `morx` font loses is the adjustment of zeroed marks, “because Apple Color Emoji assumes this will NOT be done when forming emoji sequences”, as the comment says (issue #2967).

## Zeroing marks

A **mark** here is a glyph whose GDEF class is mark. If the font has no GDEF glyph classes, HarfBuzz makes them up from the general categories: a non-spacing mark (Mn) that isn’t a default ignorable is a mark, and everything else is a base ([GSUB](gsub.html#glyph-properties)). A spacing mark (Mc) is a base, so it keeps its advance.

Shapers that zero marks do it either “early”, before GPOS, or “late”, after it (`hb_ot_position_plan`, `hb-ot-shape.cc:1029`). Either way it is `zero_mark_widths_by_gdef` (`:988`), which sets each mark’s advance to zero, and first, if asked, moves its offset back by the advance it had:

```c++
if (_hb_glyph_info_is_mark (&info[i]))
{
  if (adjust_offsets)
    adjust_mark_offsets (&buffer->pos[i]);   /* offset -= advance */
  zero_mark_width (&buffer->pos[i]);        /* advance = 0 */
}
```

The adjustment is made only when `adjust_mark_positioning_when_zeroing` is set and the run is forward, left to right or top to bottom (`:1040`). The comment there explains why:

> If the font has no GPOS and direction is forward, then when zeroing mark widths, we shift the mark with it, such that the mark is positioned hanging over the previous glyph. When direction is backward we don’t shift and it will end up hanging over the next glyph after the final reordering.

A mark with an advance was probably drawn over that advance, so taking the advance away without moving it back would leave it over the next letter. With GPOS, no adjustment is made: GPOS is trusted to have placed the mark. In a right-to-left run the mark comes before its base once the buffer is reversed, and its zero advance puts it at the base’s origin, so it needs no adjustment.

The Thai shaper zeroes marks but doesn’t allow fallback positioning, so it shows the adjustment on its own. The Thai MAI EK in the test fonts has an advance of 200 units and its outline over that advance:

```shape
fallback.ttf  \u0E01\u0E48\u0E01  --language=th
with-gpos.ttf  \u0E01\u0E48\u0E01  --language=th
```

Without GPOS the tone mark is moved back 200 units, over its KO KAI. With a GPOS that has no anchors for it, it loses its advance and stays where it was drawn, over the start of the next letter.

When fallback positioning runs, the offsets this step sets are replaced, as the comment at `:1037` says: “If fallback positioning happens, we don’t care about this as it will be overridden.”

## Recategorizing marks

Unicode’s canonical combining classes from 200 to 240 are named after positions: 220 is Below, 230 Above, 232 Above_Right, and so on. The classes from 10 to 199 are not. Each is a “fixed position” class for one or a few marks of one script, Hebrew’s from 10 to 26, Arabic’s from 27 to 35, Thai’s 103 and 107, chosen so that canonical ordering sorts them, and they say nothing about where the mark goes. The fallback positioner understands only positions. So before it can use them, `_hb_ot_shape_fallback_mark_position_recategorize_marks` (`hb-ot-shape-fallback.cc:169`) rewrites the class of every non-spacing mark into one of the positional ones, through `recategorize_combining_class` (`:35`). This is **recategorizing**. Read from the code, with Unicode’s classes in place of HarfBuzz’s [modified](unicode-inside-harfbuzz.html#modified-combining-classes) ones, which are what it actually switches on:

| Script | Unicode class | Marks | Becomes |
|---|---|---|---|
| Hebrew | 10–18, 20, 22 | sheva to qamats, qubuts, meteg | Below (220) |
| | 23 | rafe | Attached_Above (214) |
| | 24 | shin dot | Above_Right (232) |
| | 19, 25 | holam, sin dot | Above_Left (228) |
| | 26 | point varika | Above (230) |
| | 21 | dagesh | unchanged |
| Arabic, Syriac | 27, 28, 30, 31, 33–36 | fathatan, dammatan, fatha, damma, shadda, sukun, superscript alef and alaph | Above |
| | 29, 32 | kasratan, kasra | Below |
| Thai | 103 | sara u, sara uu | Below_Right (222) |
| | 107 | the tone marks | Above_Right |
| | 9 | phinthu (U+0E3A) | Below_Right |
| | 0 | U+0E31, U+0E34–0E37, U+0E47, U+0E4C–0E4E | Above_Right |
| Lao | 118 | sign u, sign uu | Below |
| | 122 | the tone marks | Above |
| | 0 | U+0EB1, U+0EB4–0EB7, U+0EBB, U+0ECC, U+0ECD; U+0EBC | Above; Below |
| Tibetan | 129, 132 | sign aa, sign u | Below |
| | 130 | sign i | Above |

Any class of 200 or more is kept, and so is any other class, such as 1 (overlays), 7 (nuktas) and 9 (viramas, apart from Thai’s). Only marks with general category Mn are touched (`:180`).

It runs in step 10 (`hb-ot-shape.cc:895`), in an odd place, as its own comment admits: “This is unfortunate to go here, but necessary...”. It has to come after normalization, which sorts marks by their classes and decides from them which marks may compose with a base. Positional classes would sort differently, and two marks given the same class would block each other’s composition ([Normalization](normalization.html#round-two-reorder)). And it has to come before step 11, which replaces each character in the buffer with its glyph, because the Thai and Lao cases look at the character itself.

Most rows of that table are rarely used. Thai and Lao always get the Thai shaper, which doesn’t allow fallback positioning: `NEWS` for 0.9.7 says “Disable fallback positioning for Indic and Thai shapers”, while 0.9.4’s had announced “Better fallback glyph positioning, specially with Thai / Lao marks”. Tibetan goes to the USE, which doesn’t allow it either, unless the font has a `DFLT` or `latn` script and no `tibt`, which sends it to the default shaper. So the Thai rows run only for Thai text given another script, as here:

```shape
fallback.ttf  \u0E01\u0E48  --language=th
fallback.ttf  \u0E01\u0E48  --language=th  --script=Latn
```

Under `Latn`, the default shaper places the tone mark at the right of the KO KAI, above it.

## Placing marks

The positioner itself is `_hb_ot_shape_fallback_mark_position` (`hb-ot-shape-fallback.cc:470`). It runs in step 20, after GPOS would have, after attachments have been resolved and default ignorables zeroed.

### Bases and their marks

It divides the buffer into runs. A run starts at each glyph that is not a mark (by general category, Mn, Mc or Me), not hidden and not a default ignorable (`:487`). The buffer’s cluster numbers aren’t looked at. Within a run, the first glyph that isn’t a mark is the base, and the marks after it are its marks, passing over any hidden or ignorable glyphs on the way (`:442`). A ZWJ or a CGJ between a letter and its accent doesn’t separate them:

```shape
fallback.ttf  o\u0301  --language=en
fallback.ttf  o\u200D\u0301  --language=en
fallback.ttf  o\u034F\u0301  --language=en
```

The ZWJ has become the space glyph with no advance ([Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#default-ignorables)), and the acute is still over the `o`. A run with no marks is left alone. One with marks is marked unsafe to break from the base to its last mark (`:334`).

Here the mark’s *category* decides what is a mark, not its GDEF class, and its *combining class* decides how it is placed. A mark whose class is 0 isn’t placed at all: it keeps its offset, and its advance counts toward the distance back to the base for the marks after it (`:414`).

### The base’s box

The positioner asks the font funcs for the base’s extents ([Where metrics come from](where-metrics-come-from.html#glyph-extents)), and then changes them (`:344`):

```c++
base_extents.y_bearing += buffer->pos[base].y_offset;
/* Use horizontal advance for horizontal positioning.
 * Generally a better idea.  Also works for zero-ink glyphs.  See:
 * https://github.com/harfbuzz/harfbuzz/issues/1532 */
base_extents.x_bearing = 0;
base_extents.width = font->get_glyph_h_advance (buffer->info[base].codepoint);
```

(trimmed). So the box a mark is placed against is the base’s ink vertically and its advance horizontally. So a mark goes in the middle of the letter’s space even when its ink is off-center, and a base with no ink still has a width to center on. If the base has no extents, its marks are given no advance and left where they are (`:340`).

### Each mark, by its class

For each mark, `position_mark` (`:211`) asks for the mark’s own extents, and works out an offset that moves its ink to the right place against the box. Horizontally:

| Classes | The mark’s ink is |
|---|---|
| 233 Double_Below, 234 Double_Above | centered on the base’s right edge, in a left-to-right run, or on its left edge, in a right-to-left one, so that it spans two letters; otherwise centered |
| 200 Attached_Below_Left, 218 Below_Left, 228 Above_Left | aligned with the left edge |
| 216 Attached_Above_Right, 222 Below_Right, 232 Above_Right | aligned with the right edge |
| every other class | centered |

Vertically, with a **gap** of a sixteenth of the font’s y scale, 62 units in a 1000-unit font at its natural size:

| Classes | The mark’s ink is |
|---|---|
| 218, 220 Below, 222, 233 | a gap below the base |
| 200, 202 Attached_Below | just below the base |
| 228, 230 Above, 232, 234 | a gap above the base |
| 214 Attached_Above, 216 | just above the base |
| every other class | where its outline puts it |

Two corrections follow. A below mark is never moved up: if its outline is already low enough, it stays (`:288`). An above mark drawn higher than it needs to be is moved down only halfway (`:310`): “Don’t shift down "above" marks too much.” `fallback.ttf` has its acute drawn from 550 to 700 units, its grave from 800 to 950, and its dot below from −100 to −200:

```shape
fallback.ttf  o\u0301  --language=en
fallback.ttf  l\u0301  --language=en
fallback.ttf  o\u0300  --language=en
fallback.ttf  o\u0323  --language=en
fallback.ttf  o\u0327  --language=en
fallback.ttf  o\u031B  --language=en
fallback.ttf  o\u0361o  --language=en
```

The acute moves up 12 units over the `o`, to 562, a gap above its top, and 262 over the taller `l`. The grave belongs at 562 too, 238 units below where it is drawn, and is moved down 119. The dot below is already lower than a gap below the `o`, so it isn’t moved. The cedilla, Attached_Below, goes just below the base, with no gap, which is where it is drawn already. The horn, Attached_Above_Right, goes on top of the `o` at its right edge, which is not where a horn belongs. The double inverted breve is centered on the join between the two `o`s.

Every placed mark loses its advance, and gets an offset back to its base: in a forward run, the base’s advance plus those of any unplaced glyphs between them (`:411`). A right-to-left run needs no such offset, since after reversal the mark comes before its base.

### Stacking

When a mark has been placed, the box is grown to include it, so that the next mark of the same class goes beyond it (`:294` and `:318`). The box is reset to the base’s whenever the class changes (`:401`):

```shape
fallback.ttf  o\u0301\u0301  --language=en
fallback.ttf  o\u0301\u0323\u0301  --language=en
```

The second acute goes a gap above the first, at 224. In the second line the dot below sits between the two acutes as typed, but normalization has already sorted the marks by class, dot below (220) before the acutes (230), so the acutes are together and stack. That is a reason for running after normalization too: marks of one class are next to each other by then. Recategorizing can still separate them, since it gives several of Hebrew’s classes the same position while dagesh, between them in the sorted order, keeps a class of its own (inferred from the code). Drawn by hb-vector, which shapes before it draws:

```draw
font: fallback.ttf
text: o\u0301l\u0301o\u0300o\u0323\u0301\u0301o\u0327o\u0361o
```

The same text in `with-gpos.ttf`, where every mark keeps its outline’s position:

```draw
font: with-gpos.ttf
text: o\u0301l\u0301o\u0300o\u0323\u0301\u0301o\u0327o\u0361o
```

### Ligatures

A ligature carries the number of its components, and each mark that GSUB passed over while forming it carries the component it followed ([GSUB](gsub.html#ligature-substitution)). The fallback positioner divides the ligature’s advance into that many equal parts, and places each mark against its own part (`:371`). A mark with no component, or with a different ligature ID, goes on the last part. In a right-to-left script the parts are counted from the right.

```shape
fallback.ttf  f\u0301i  --language=en
fallback.ttf  fi\u0301  --language=en
fallback.ttf  f\u0301f\u0301i\u0301  --language=en
```

The acute over the first half of `f_i` is 325 units, half of 650, to the left of the one over the second half. The three on `f_f_i` are a third of 1050 apart. GPOS, with an anchor for each component, can do better, as [GPOS: positioning glyphs](positioning-glyphs-gpos.html#mark-to-ligature) shows.

### Hebrew and Arabic

The same rules place Hebrew and Arabic points, after recategorizing:

```shape
fallback.ttf  \u05D1\u05BC\u05B4  --language=he
fallback.ttf  \u05E9\u05C1  --language=he
fallback.ttf  \u05E9\u05C2  --language=he
fallback.ttf  \u0628\u0651\u0650  --language=ar
```

The dagesh keeps its class, so it is centered and left at its drawn height, inside the letter. The hiriq is centered below. The shin dot goes to the right, the sin dot to the left. The shadda goes above and the kasra below the beh, each against the beh’s own box, since their classes differ. Both the shadda and the fatha are drawn higher than they need to be, and are moved down only halfway.

### What it doesn’t take into account

**The base’s horizontal offset.** The base’s vertical offset is added to its box, but its horizontal one isn’t. `kern` and fallback kerning split a pair’s kerning between the two glyphs’ advances and move the second glyph back by half ([Legacy kern](legacy-kern.html#how-the-kerning-is-applied)). Its mark is then placed as if it hadn’t moved:

```shape
fallback.ttf  oo\u0301  --language=en
kern-table.ttf  oo\u0301  --language=en
```

The second `o` has an advance of 450 and is drawn 50 units left of its pen position, so its ink is centered 200 units from that position. The acute is centered 250 units from it, as if the `o` hadn’t moved. To be centered over the ink it would need an offset of −25, not 25. This looks like a bug.

> Since {{hb}}: fixed on HarfBuzz main by [`641971d`](https://github.com/harfbuzz/harfbuzz/commit/641971dd14283a97f5e7cf8a7f12ac6337db74f7) (17 November 2026), so main includes the base’s x offset, and the acute over the kerned `o` is at −25.

**Left and Right.** The comment in `position_mark` says “We don’t position LEFT and RIGHT marks”, but classes 224 and 226 fall into the `default` case of the horizontal switch, which centers the mark, and every placed mark loses its advance. The few characters in those classes are spacing marks, Mc, such as the Hangul tone marks and U+1D16D MUSICAL SYMBOL COMBINING AUGMENTATION DOT, which should follow its note:

```shape
fallback.ttf  o\u{1D16D}  --language=en
with-gpos.ttf  o\u{1D16D}  --language=en
```

Without GPOS the dot is centered on the `o` and loses its 200-unit advance; with GPOS it keeps both. That too looks like a bug, or at least a comment that no longer matches the code.

**Synthetic slant.** Fallback positioning runs after `position_finish_offsets`, which is where a synthetically slanted font has its vertically offset glyphs moved sideways to follow the slant ([Fonts and font functions](fonts-and-font-functions.html#slant)). So a fallback-placed mark doesn’t follow the slant. Its extents are measured from the slanted outline, as [Fonts and font functions](fonts-and-font-functions.html#extents-of-a-synthetic-font) shows, and centring them on the upright advance puts the mark over the middle of the upright letter, not over its slanted top. Synthetic bold, which changes the extents, is taken into account:

```shape
fallback.ttf  o\u0301  --language=en  --show-extents
fallback.ttf  o\u0301  --language=en  --show-extents  --font-bold=0.1
```

## Spaces

[Normalization](normalization.html#spaces-and-one-hyphen) maps a space character the font has no glyph for to the glyph for U+0020, records which kind of space it was in the byte that holds a mark’s combining class, and sets the buffer flag `HAS_SPACE_FALLBACK`. In step 15, after the default advances are read and only if that flag is set (`hb-ot-shape.cc:1024`), `_hb_ot_shape_fallback_spaces` (`hb-ot-shape-fallback.cc:559`) gives each such space its width. The **space fallback types** come from `space_fallback_type` (`hb-unicode.hh:219`):

| Type | Characters | Width |
|---|---|---|
| `SPACE` | U+0020 SPACE, U+00A0 NO-BREAK SPACE | the space glyph’s own |
| `SPACE_EM` | U+2001 EM QUAD, U+2003 EM SPACE, U+3000 IDEOGRAPHIC SPACE | an em |
| `SPACE_EM_2` | U+2000 EN QUAD, U+2002 EN SPACE | half an em |
| `SPACE_EM_3`, `_4`, `_6` | U+2004, U+2005, U+2006, the three-, four- and six-per-em spaces | a third, a quarter, a sixth |
| `SPACE_EM_5` | U+2009 THIN SPACE | a fifth |
| `SPACE_EM_16` | U+200A HAIR SPACE | a sixteenth |
| `SPACE_4_EM_18` | U+205F MEDIUM MATHEMATICAL SPACE | four eighteenths |
| `SPACE_FIGURE` | U+2007 FIGURE SPACE | the advance of the first of the digits 0 to 9 the font has |
| `SPACE_PUNCTUATION` | U+2008 PUNCTUATION SPACE | the advance of the full stop, or failing that the comma |
| `SPACE_NARROW` | U+202F NARROW NO-BREAK SPACE | half the space glyph’s |

An em is the font’s scale, `x_scale` (`y_scale` in vertical text), not its units per em, and the fractions of an em are rounded to the nearest unit. Four eighteenths isn’t rounded, but cut short. U+1680 OGHAM SPACE MARK is the one Zs character left out. The figure and punctuation widths are asked of the font funcs: `get_nominal_glyph` for the characters, then the glyph’s advance. `fallback.ttf` has glyphs for U+0020, the digit zero, 560 units wide, and the full stop, 200:

```shape
fallback.ttf  \u2003  --language=en
fallback.ttf  \u2002  --language=en
fallback.ttf  \u2004  --language=en
fallback.ttf  \u2006  --language=en
fallback.ttf  \u2009  --language=en
fallback.ttf  \u200A  --language=en
fallback.ttf  \u205F  --language=en
fallback.ttf  \u2007  --language=en
fallback.ttf  \u2008  --language=en
fallback.ttf  \u202F  --language=en
fallback.ttf  \u2003  --language=en  --font-size=2000
```

The comment on the narrow space explains its choice: Unicode’s chart says about a quarter or a fifth of an em, “However, in my testing, many fonts have their regular space being about that size. To me, a percentage of the space width makes more sense. Half is as good as any.”

If the font has no U+0020 either, normalization uses the buffer’s **invisible glyph**, if the application has set one with `hb_buffer_set_invisible_glyph`; otherwise the character gets the not-found glyph and no fallback at all. A space that became the invisible glyph is given a quarter of an em first (`:571`), and then its type’s width. `no-space.ttf` is `fallback.ttf` without U+0020 in its `cmap`; glyph 4 is `o`, 500 units wide, used here only because its width is easy to tell from a quarter of an em:

```shape
no-space.ttf  \u0020  --language=en
no-space.ttf  \u0020  --language=en  --invisible-glyph=4
no-space.ttf  \u2003  --language=en  --invisible-glyph=4
no-space.ttf  \u202F  --language=en  --invisible-glyph=4
```

The narrow space is half of the quarter em it was given first. A space that GSUB has made part of a ligature is skipped (`:568`). One that GSUB merely replaced keeps its type, since the type travels with the character’s properties, so the replacement glyph gets the width. Since all this happens before GPOS, GPOS can still kern against the space.

## Kerning through the font funcs

The last fallback asks the font funcs for kerning. `_hb_ot_shape_fallback_kern` (`hb-ot-shape-fallback.cc:523`) runs in step 17 when the plan has `apply_fallback_kern`: the shaper allows fallback positioning, and the font has no GPOS, `kerx` or `kern` that HarfBuzz will apply itself. Then it checks that the font funcs implement `glyph_h_kerning` (`glyph_v_kerning` in vertical text), in this font or any of its parents (`hb-font.cc:1090`), and does nothing if they don’t. If they do, it sends “start fallback kern” to the trace, and runs the same kerning machine as the `kern` table, `hb_kern_machine_t` (`hb-kern.hh:39`), with a driver that asks the funcs for each pair:

```c++
hb_position_t get_kerning (hb_codepoint_t first, hb_codepoint_t second) const
{
  hb_position_t kern = 0;
  font->get_glyph_kerning_for_direction (first, second, direction, &kern, &kern);
  return kern;
}
```

The machine skips marks, kerns only glyphs whose mask has the `kern` feature’s bit, and splits each value between the two glyphs ([Legacy kern](legacy-kern.html#how-the-kerning-is-applied) has the details). `kern` is one of the features added with `F_HAS_FALLBACK`, so its bit exists even when no table has the feature ([From features to lookups](from-features-to-lookups.html#finding-the-features-in-the-font)), and `--features=-kern` turns the fallback off. A right-to-left buffer is reversed around the call and back, so that pairs are asked for in visual order (`:540`).

Only one font funcs implementation in HarfBuzz sets `glyph_h_kerning`: hb-ft, which asks FreeType’s `FT_Get_Kerning` (`hb-ft.cc:719`) ([Fonts and font functions](fonts-and-font-functions.html#the-other-funcs)). hb-ot doesn’t, and doesn’t need to. Since 2.0.0, whose `NEWS` says “Apply TrueType kern table internally, instead of relying on hb_font_t callbacks”, the shaper reads the font’s `kern` table itself, and when there is one, the plan applies it and never gets as far as the fallback. The kerning callbacks were deprecated in that release. 2.5.2 undeprecated the horizontal one “and re-enable[d it] in hb-ft, such that Type1 fonts will continue kerning”. That is the case the fallback is for: a font whose kerning HarfBuzz can’t read as a table, but whose font funcs can supply.

Our build has no FreeType, so `hb-shape` has only the `ot` funcs, and this fallback can’t be shown here. The trace above has no “start fallback kern”: the plan asked for it, and it returned at once, since the `ot` funcs have no kerning callback.

> For programmers: to give HarfBuzz kerning from your own source, for a font format it can’t read, implement `glyph_h_kerning` in your font funcs (`hb_font_funcs_set_glyph_h_kerning_func`). It is used only when the face has no GPOS, `kerx` or `kern` and the script’s shaper allows fallback positioning, which the Indic, Khmer, Myanmar, Thai and USE shapers don’t. In a build with `HB_DISABLE_DEPRECATED`, which `HB_LEAN` implies, `get_glyph_h_kerning` returns 0 and the fallback is compiled out, though the horizontal callback is no longer deprecated.

> For type designers: a font with no GPOS still gets its marks placed, but only roughly: centered on the advance, a sixteenth of an em from the ink, all ligature components equal. Any GPOS at all, even a lone `kern` feature, switches that off, and marks then sit exactly where their outlines put them. If a font has GPOS, it needs mark attachment too. And draw zero-width marks to the left of the origin, over the glyph before, because that is where they stay when nothing moves them.

## Configuration

`HB_NO_OT_SHAPE_FALLBACK` empties the recategorizing, mark positioning and kerning functions (`hb-ot-shape-fallback.cc:173`, `:475`, `:527`), and removes hb-ft’s kerning callback. The plan’s flags are still computed, so marks are still zeroed and adjusted. The space fallback has no macro of its own. `HB_NO_OT_SHAPE_FALLBACK` also turns off the Arabic, Hebrew and Thai shapers’ fallbacks, vowel constraints and the Zawgyi shaper (`hb-config.hh:174`), and `HB_NO_LEGACY` implies it (`:156`). `HB_NO_OT_KERN` leaves out the `kern` table, so a font with one gets fallback kerning instead.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-ot-shape-fallback.cc` | recategorizing, fallback mark positioning, fallback kerning, space widths |
| `src/hb-ot-shape.cc` | the plan’s decisions (`compile`), step 10’s call, `zero_mark_widths_by_gdef`, `hb_ot_position_default` and `hb_ot_position_plan` |
| `src/hb-ot-shape.hh` | the plan’s flags |
| `src/hb-ot-shaper-*.cc` | each shaper’s `zero_width_marks` and `fallback_position` |
| `src/hb-kern.hh` | `hb_kern_machine_t`, shared with the `kern` table |
| `src/hb-unicode.hh` | `space_fallback_type`, the modified combining classes |
| `src/hb-ot-shape-normalize.cc` | where spaces are mapped to the space glyph |
| `src/hb-ft.cc` | `hb_ft_get_glyph_h_kerning` |

## In short

* The plan decides which fallbacks run, once: fallback mark positioning only with no GPOS and no `kerx` (and no cross-stream `kern`), and only if the script’s shaper allows it; fallback kerning only with no GPOS, `kerx` or `kern`.
* A GPOS with only kerning turns fallback mark positioning off. Hebrew’s GPOS is ignored unless its script is `hebr`. A `morx` font keeps the original shaper’s permission.
* Shapers zero marks by GDEF class, or by classes made up from Mn. Without GPOS, a zeroed mark in a forward run is moved back by its old advance.
* Recategorizing turns script-specific combining classes into positional ones, after normalization and before glyph mapping. Its Thai, Lao and Tibetan rows are rarely reached.
* The positioner places each mark from extents: against the base’s ink vertically and its advance horizontally, by class, a sixteenth of an em away, stacking marks of the same class, dividing ligatures equally.
* It ignores the base’s horizontal offset, centers Left and Right marks, and doesn’t follow synthetic slant.
* Missing space characters get widths from their kind: fractions of an em, a digit’s or full stop’s advance, or half a space.
* Fallback kerning asks the font funcs; only hb-ft implements it, for fonts like Type 1 whose kerning isn’t in a table HarfBuzz reads.
