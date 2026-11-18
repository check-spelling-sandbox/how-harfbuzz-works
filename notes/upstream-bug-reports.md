# Draft bug reports for HarfBuzz

Eighty-six possible bugs found while writing the book, drafted for the HarfBuzz issue tracker. None has been filed. Each report carries a tag naming the batch it was sent in, to go with it when it’s filed: 1–65 are `hhw-20261117`, the batch shown to Behdad Esfahbod on 17 November 2026. 66–82, found later, are `hhw-20261118`, the second batch, of 18 November 2026. 83 to 86 are `hhw-20261120`, of 20 November 2026; 84 to 86 were found on 15.2.0, during the book’s move to it. Each was reproduced on HarfBuzz 15.1.0 (commit `3df40d9b`), built by `tools/build-harfbuzz.sh` (meson, with FreeType, ICU and cairo off), on macOS. Repros use `test/subset/data/fonts/Roboto-Regular.ttf` from HarfBuzz’s own repository where they can, so they need nothing from this book.

**The numbers are permanent.** HarfBuzz pull requests and commits already cite these reports by number (“How HarfBuzz Works bug 75”), so a number, once given, always means the same report. Cite a report as **HHW-75**, as the book does: “HHW” for How HarfBuzz Works, then its number.

* Never renumber, reorder or reuse a number, and never close up a gap.
* A new report takes the next number after the highest ever given.
* A report found to be wrong, a duplicate or not a bug keeps its number and heading, marked **Withdrawn.** with the reason. It is never deleted.
* Fixed reports stay, with how they were fixed.
* The tables in the appendix Bugs this book found (`book/appendices/bugs-this-book-found.md`) use the same numbers, and each row’s anchor is its ID in lower case (`bugs-this-book-found.html#hhw-75`).

Since 17 November many of them have been fixed on HarfBuzz’s `main` branch, and others have open pull requests. On 18 November 2026 every report was rechecked against `main` at commit `454f9c49c` (17 November 2026), built with the same options, by rerunning its repro or, for those read from the code, reading the code again. Report 83, written on 20 November, was checked against `main` at `94715f603` (19 November 2026). Each report ends with the result, under **On main.**: fixed (with the commit and pull request), changed, in progress (with the open pull request and its branch), or still present. On 20 November 2026 every report was checked again, against HarfBuzz 15.2.0, released that day with every fix merged up to 19 November, and against `main` at `8846dbbf6` (20 November 2026), by rerunning its repro where it had changed; each report ends with the result, under **In 15.2.0 and on main**. Fifty-three are fixed in 15.2.0 and fifteen more on `main`; seven are partly fixed, one is changed but not fixed, two were declined (13 and 32), and five still apply.

---

## 1. A null or unknown-format Coverage in SingleSubst/SinglePos format 1 makes the sanitizer reject the whole GSUB/GPOS

**ID:** `HHW-1` · **Tag:** `hhw-20261117`

**Summary.** `Coverage::get_population ()` returns `NOT_COVERED` (`0xFFFFFFFF`) for a format it doesn’t know (`src/OT/Layout/Common/Coverage.hh:138`), and so for the Null Coverage, whose format is 0. `SingleSubstFormat1_3::sanitize` passes half the population to `check_ops` (`src/OT/Layout/GSUB/SingleSubstFormat1.hh:125`), and so does `SinglePosFormat1::sanitize` (`src/OT/Layout/GPOS/SinglePosFormat1.hh:39`). With a null coverage offset, or a coverage of unknown format, that charges about 2 billion ops, `max_ops` runs out, and the whole table fails to sanitize.

Elsewhere a null offset, or an unknown format, makes that one subtable inert: the same change to a SingleSubst *format 2* subtable loses only that lookup. Since 12.3.0 a failed sanitize drops the whole table, so here one inert subtable takes every other lookup in GSUB (or GPOS) with it.

**Repro.** This script (fontTools) builds `good.ttf`, whose `liga` has two lookups, a format 2 SingleSubst (`a c` → `b a`) and a format 1 SingleSubst (`x` → `y`), and `null-coverage-f1.ttf`, the same font with the format 1 subtable’s `coverageOffset` set to 0:

```python
import struct
from fontTools.fontBuilder import FontBuilder
from fontTools.pens.ttGlyphPen import TTGlyphPen
from fontTools.feaLib.builder import addOpenTypeFeaturesFromString
from fontTools.ttLib import TTFont

names = [".notdef", "a", "b", "c", "x", "y"]
fb = FontBuilder(1000, isTTF=True)
fb.setupGlyphOrder(names)
fb.setupCharacterMap({ord(n): n for n in "abcxy"})
pen = TTGlyphPen(None)
fb.setupGlyf({n: pen.glyph() for n in names})
fb.setupHorizontalMetrics({n: (500, 0) for n in names})
fb.setupHorizontalHeader(ascent=800, descent=-200)
fb.setupNameTable({"familyName": "NullCov", "styleName": "Regular"})
fb.setupOS2(); fb.setupPost()
addOpenTypeFeaturesFromString(fb.font, """
lookup L0 { sub [a c] by [b a]; } L0;
lookup L1 { sub x by y; } L1;
feature liga { lookup L0; lookup L1; } liga;
""")
fb.save("good.ttf")

data = bytearray(open("good.ttf", "rb").read())
g = TTFont("good.ttf").reader.tables["GSUB"].offset
u16 = lambda o: struct.unpack_from(">H", data, o)[0]
lookup_list = g + u16(g + 8)
lookup1 = lookup_list + u16(lookup_list + 4)
sub = lookup1 + u16(lookup1 + 6)
assert u16(sub) == 1                      # SingleSubst format 1
struct.pack_into(">H", data, sub + 2, 0)  # coverageOffset = 0
open("null-coverage-f1.ttf", "wb").write(data)
```

```
$ hb-shape good.ttf acx
[b=0+500|a=1+500|y=2+500]
$ hb-shape null-coverage-f1.ttf acx
[a=0+500|c=1+500|x=2+500]
```

**Expected.** Lookup 1 does nothing (its coverage is empty) and lookup 0 still applies: `[b=0+500|a=1+500|x=2+500]`.

**Actual.** Neither lookup applies: GSUB was rejected.

**Possible fix.** Have `Coverage::get_population ()` return 0 in its `default` case, which is what an empty or unknown coverage contains. The only other caller, `single_subst_closure` (`SingleSubstFormat1.hh:61`), uses the value as a size hint (`if (pop >= mask) return;`) and would just go on to intersect an empty coverage.

Real fonts are unlikely to have null coverage offsets, so the impact is small. But a coverage format from a future OpenType version would have the same effect, which seems contrary to the sanitizer’s policy of letting unknown formats pass.

**On main.** Fixed on 17 November 2026 by `6426ee4ca` (pull request #6696), which makes `get_population ()` return 0 for an unknown format. `null-coverage-f1.ttf` now gives `[b=0+500|a=1+500|x=2+500]`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 2. The “start/end table GPOS” trace messages print GSUB’s chosen script

**ID:** `HHW-2` · **Tag:** `hhw-20261117`

**Summary.** `hb_ot_map_t::position` (`src/hb-ot-layout.cc:2149`) formats its trace messages with `chosen_script[0]`, the script chosen for GSUB. GPOS’s is `chosen_script[1]` (`hb-ot-map.cc:80` fills them per table index). The code looks copied from `hb_ot_map_t::substitute` just above it.

**Repro.** Any font whose GSUB and GPOS choose different scripts. The plainest case is a font with no GSUB, where GSUB’s chosen script is empty. Roboto with its GSUB removed (`del font['GSUB']` in fontTools) shows it: the trace says GPOS has no script tag either, though GPOS chose `latn` and kerned:

```
$ hb-shape --trace Roboto-noGSUB.ttf AV | grep 'table G\|kern'
trace: start table GSUB script tag ''   buffer: [gid37=0|gid58=1]
trace: end table GSUB script tag ''     buffer: [gid37=0|gid58=1]
trace: start table GPOS script tag ''   buffer: [gid37=0+1336|gid58=1+1303]
trace: start lookup 1 feature 'kern'    buffer: [gid37=0+1336|gid58=1+1303]
trace: end lookup 1 feature 'kern'      buffer: [gid37=0+1249|gid58=1+1303]
trace: end table GPOS script tag ''     buffer: [gid37=0+1249|gid58=1+1303]
```

**Expected.** The GPOS lines name GPOS’s chosen script.

**Possible fix.** `hb_tag_to_string (chosen_script[1], tag);` in `hb_ot_map_t::position`. Only the debug messages are affected.

**On main.** Fixed on 17 November 2026 by `24795efd3` (#6686). The GPOS lines of the trace now say `script tag 'latn'`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 3. A synthetic font reports extents `<0,-1,-1,1>` for an empty glyph

**ID:** `HHW-3` · **Tag:** `hhw-20261117`

**Summary.** With any synthetic setting (bold, grade or slant), `hb_font_t::get_glyph_extents` measures the glyph by painting or drawing it (`src/hb-font.hh:586`–`630`). For a glyph with no outline, such as space, nothing is drawn, so the `hb_extents_t` stays in its initial, *void* state: `xmin = 0, ymin = 0, xmax = -1, ymax = -1` (`src/hb-geometry.hh:121`–`124`). `draw_glyph_or_fail` still succeeds, and `hb_extents_t::to_glyph_extents` (`hb-geometry.hh:100`) converts the void box as if it were real, giving x_bearing 0, y_bearing −1, width −1 and height 1.

**Repro.**

```
$ hb-shape test/subset/data/fonts/Roboto-Regular.ttf "a b" --show-extents
[gid69=0+1114<109,1102,893,-1122>|gid4=1+507<0,0,0,0>|gid70=2+1149<140,1536,916,-1556>]
$ hb-shape test/subset/data/fonts/Roboto-Regular.ttf "a b" --show-extents --font-bold=0.02
[gid69=0+1155<109,1143,934,-1163>|gid4=1+548<0,-1,-1,1>|gid70=2+1190<140,1577,957,-1597>]
$ hb-shape test/subset/data/fonts/Roboto-Regular.ttf "a b" --show-extents --font-slant=0.2
[gid69=0+1114<141,1102,1002,-1122>|gid4=1+507<0,-1,-1,1>|gid70=2+1149<140,1536,1077,-1556>]
```

**Expected.** `<0,0,0,0>` for the space, as without synthetic settings.

**Actual.** `<0,-1,-1,1>`: a box with negative width and positive height, one unit below the origin.

**Possible fix.** Return zero extents for a void box in `to_glyph_extents`:

```c++
if (unlikely (is_void ()))
  return hb_glyph_extents_t {0, 0, 0, 0};
```

or check `is_void ()` in the two callers in `get_glyph_extents` (`hb-font.hh:617`, `:627`). The COLR caller (`OT/Color/COLR/COLR.hh:3168`) may want the same. `src/test-geometry.cc` would be the place for a test.

**On main.** Fixed on 17 November 2026 by `46ab0d095` (#6685), which returns zero extents for a void box. With `--font-bold=0.02` or `--font-slant=0.2` the space now has `<0,0,0,0>`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 4. Scaled glyph extents are rounded inwards vertically

**ID:** `HHW-4` · **Tag:** `hhw-20261117`

**Summary.** `hb_font_t::scale_glyph_extents` (`src/hb-font.hh:201`) takes the floor of `x1` and `y1` and the ceiling of `x2` and `y2`. Across, `x1` is the left edge and `x2` the right, so the box grows outwards. Up and down it’s the other way round: `y1` is `y_bearing`, the *top* edge, and `y2 = y1 + height` (height being negative) is the bottom. Taking the floor of the top moves it down, and the ceiling of the bottom moves it up, so the box shrinks vertically. NEWS for 14.2.1 says extents are now scaled “so the reported box always covers the glyph”. It does across, but not up and down.

**Repro.** Roboto’s `p` (2048 units per em) at a scale of 13:

```
$ hb-shape test/subset/data/fonts/Roboto-Regular.ttf p --show-extents
[gid84=0+1149<140,1102,914,-1518>]
$ hb-shape test/subset/data/fonts/Roboto-Regular.ttf p --show-extents --font-size=13
[gid84=0+7<0,6,7,-8>]
```

Scaled exactly, the ink runs from y = 6.995 (1102 × 13/2048) down to y = −2.641. Reported, it runs from 6 down to −2: almost a whole unit short at the top, and 0.64 short at the bottom. Across, 0.889 to 6.691 becomes 0 to 7, which does cover it.

**Expected.** y_bearing 7, height −10 (top 7, bottom −3).

**Possible fix.** Round each edge away from the box’s center, whichever way the axes point. For example, round `hb_min (y1, y2)` down and `hb_max (y1, y2)` up, then put them back as top and bottom, and the same for x, which would also keep it right for negative scales. A sketch, not tested:

```c++
float xlo = floorf (hb_min (x1, x2)), xhi = ceilf (hb_max (x1, x2));
float ylo = floorf (hb_min (y1, y2)), yhi = ceilf (hb_max (y1, y2));
extents->x_bearing = hb_clamp_to<hb_position_t> (x1 <= x2 ? xlo : xhi);
extents->width     = hb_clamp_to<hb_position_t> (x1 <= x2 ? (double) xhi - xlo : (double) xlo - xhi);
extents->y_bearing = hb_clamp_to<hb_position_t> (y1 >= y2 ? yhi : ylo);
extents->height    = hb_clamp_to<hb_position_t> (y1 >= y2 ? (double) ylo - yhi : (double) yhi - ylo);
```

Expected outputs in `test/shape/` and `test/api/` that print scaled extents would change.

**On main.** Fixed on 17 November 2026 by `f5e925cd2` (#6707). `p` at a scale of 13 is now `<0,7,7,-10>`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 5. Glyph-flag ranges treat cluster level 3 (GRAPHEMES) as monotone

**ID:** `HHW-5` · **Tag:** `hhw-20261117`

*Found by reading the code; not yet reproduced in shaping output.*

**Summary.** `_infos_set_glyph_flags` and `_infos_find_min_cluster` (`src/hb-buffer.hh:664` and `:694`) take a shortcut that is only valid when clusters are in order: they look at the first and last glyphs of a range rather than scanning it. They skip the shortcut with

```c++
if (cluster_level == HB_BUFFER_CLUSTER_LEVEL_CHARACTERS)
```

but level 3, `HB_BUFFER_CLUSTER_LEVEL_GRAPHEMES` (11.0.0), is not monotone either. Everywhere else the code tests `HB_BUFFER_CLUSTER_LEVEL_IS_MONOTONE` (`hb-buffer.hh:400`, `:423`, `hb-buffer-verify.cc:66`, and the CoreText and Graphite backends), and these two tests look as though they were missed when level 3 was added.

**Effect, as far as the code shows.** At level 3, after reordering puts clusters out of order, a range whose smallest cluster is neither its first nor its last glyph gets the wrong minimum from `_infos_find_min_cluster`, and `_infos_set_glyph_flags` then marks the wrong glyphs unsafe to break or unsafe to concat (it leaves out the glyphs it believes belong to the minimum cluster). The verifier doesn’t catch it, because it skips non-monotone levels.

**Possible fix.** Use the predicate in both places:

```c++
if (!HB_BUFFER_CLUSTER_LEVEL_IS_MONOTONE (cluster_level) || …)
```

**Before filing.** Try to construct a level-3 example whose flags differ from level 2’s in a way the scan would correct, for instance an Indic syllable whose reordering leaves its smallest cluster in the middle of a flagged range, and include it.

**On main.** Fixed on 17 November 2026 by `7f0fe3561` (#6706): both functions now test `HB_BUFFER_CLUSTER_LEVEL_IS_MONOTONE` (`hb-buffer.hh:677`, `:707` on `main`). Checked in the code; there was no example to rerun.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 6. Small slips in `hb-shape-plan.cc`

**ID:** `HHW-6` · **Tag:** `hhw-20261117`

*Found by reading the code. None changes shaping results today.* Three separate one-line issues, which could go in one report.

* **`features[0]` for `features[i]`.** `hb_shape_plan_key_t::init` copies the user features and, to “make start/end uniform to easier catch bugs”, sets every non-global `start` to 1 and `end` to 2. The loop indexes `features[0]` on every pass (`src/hb-shape-plan.cc:95`–`99`), so only the first feature is normalized. Nothing reads the copied ranges, so it’s harmless.
* **A null check after the dereference.** `hb_shape_plan_create2` tests `HB_DIRECTION_IS_VALID (props->direction)` (`:255`) before `if (unlikely (!props)) goto bail;` (`:260`), so the null check can never fire: a null `props` crashes first.
* **No null-face guard in the cached path.** `hb_shape_plan_create_cached2` reads `face->shape_plans` (`:553`) before anything checks `face`, so a null face crashes. `hb_shape_plan_create2` handles one by using the empty face.

**Possible fixes.** `features[i]` in the loop; move the `!props` test above the direction test; give `create_cached2` the same null-face handling as `create2`.

**On main.** All three fixed on 17 November 2026: `caf12a809` (#6693) indexes `features[i]`, and `93d26d851` (#6698) tests `props` before using it and gives `create_cached2` the empty face for a null one. Checked in the code.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 7. FeatureVariations minor version 1 means different things to HarfBuzz and to the ISO draft

**ID:** `HHW-7` · **Tag:** `hhw-20261117`

*A question about the standard as much as a bug. Raise it as a discussion before filing as a bug.*

**Summary.** Since 14.6.0 HarfBuzz reads a FeatureVariations table of version 1.1 or later as having an array of `LookupVariationRecord`s after its `FeatureVariationRecord`s (`src/hb-ot-layout-common.hh:5957`), and its sanitizer requires that array to be there (`:6187`). The ISO/IEC 14496-22 working draft (WD9) defines minorVersion 1 differently: it allows Condition format 2, with nothing after the records.

A FeatureVariations table that fails to sanitize makes the whole GSUB (or GPOS) fail (`src/hb-ot-layout-gsubgpos.hh:6369`), and since 12.3.0 a failed table is dropped. So a font that follows the ISO draft’s minorVersion 1 loses every substitution, not only its variations.

**Repro.** Take a font with an ordinary FeatureVariations table (fontTools’ `addFeatureVariations` writes version 1.0) and set its minorVersion to 1, with nothing else changed. HarfBuzz then rejects the font’s GSUB: in the book’s test font, the `rvrn` substitutions stop at every weight.

**Expected.** Either that HarfBuzz and the ISO draft agree on what follows the records in a 1.1 table, or that HarfBuzz tells the two apart (for example by checking that the bytes after the records can be a lookup variation array) rather than dropping the table.

**Before raising it.** Check the current state of the ISO draft and of HarfBuzz’s “beyond 64k” / lookup-variations proposal, which may already have been reconciled, and quote the draft’s exact wording.

**On main.** Still present. The FeatureVariations code is unchanged, and `varliga-minor1.ttf` (the book’s font with minorVersion 1) still loses its GSUB: `I` at weight 800 stays `I`, where `varliga.ttf` gives `V`.


**In 15.2.0 and on main (20 November 2026).** Still present in 15.2.0 and on main at `8846dbbf6`.

---

## 8. A ranged request followed by a global one for the same feature clears the global bit

**ID:** `HHW-8` · **Tag:** `hhw-20261117`

**Summary.** When the same tag is requested twice, `hb_ot_map_builder_t::compile` merges the requests (`src/hb-ot-map.cc:238`), and a later global request makes the feature global. With value 1 it then shares the global bit (bit 31). But the earlier ranged request is still in the user features passed to the run, and `hb_ot_shape_setup_masks` (`src/hb-ot-shape.cc:760`) applies its range with the feature’s mask, which is now the global bit. The range clears bit 31 on those clusters, which turns off *every* global feature there, not just the one named.

**Repro.** `gsub/ligatures.ttf` from the book’s test fonts, whose `calt` turns `e` into `e.init` after a space or an `e.init`:

```
$ hb-shape ligatures.ttf ' eee' --language=en
[space=0+250|e.init=1+500|e.init=2+500|e.init=3+500]
$ hb-shape ligatures.ttf ' eee' --language=en --features=-liga[2:3],liga
[space=0+250|e.init=1+500|e=2+500|e=3+500]
$ hb-shape ligatures.ttf ' eee' --language=en --features=liga,-liga[2:3]
[space=0+250|e.init=1+500|e.init=2+500|e.init=3+500]
```

**Expected.** `calt` unaffected: only `liga` is named.

**Actual.** `calt` stops at cluster 2 (and, since it looks back at its own output, cluster 3).

**Possible fix.** In `hb_ot_shape_setup_masks`, skip a ranged feature whose mask is the global mask, or have the merge record that a later global request superseded earlier ranged ones so `setup_masks` can ignore them.

**On main.** Fixed on 17 November 2026 by `5d9dfde25` (#6704). `--features=-liga[2:3],liga` now gives `e.init` on all three `e`s.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 9. Mask bit 30 is never handed out

**ID:** `HHW-9` · **Tag:** `hhw-20261117`

**Summary.** `compile` gives a feature bits only if `next_bit + bits_needed >= global_bit_shift` is false (`src/hb-ot-map.cc:292`). With `global_bit_shift` 31, a one-bit feature starting at bit 30 fails the test, so the usable bits are 4–29: 26, not the 27 between the glyph flags and the global bit. Off by one.

**Repro.** A font with 27 features of one substitution each (`bits.ttf` in the book’s test fonts), all requested over a range so that each needs its own bit: the 27th by tag order is silently dropped.

**Possible fix.** `>` for `>=`. Low impact, but features dropped for lack of bits are dropped silently, so each bit counts.

**On main.** Fixed on 17 November 2026 by `ec9476415` (#6697). All 27 ranged features of `bits.ttf` now apply: the last glyph is `zero.alt`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 10. In `COMPOSED_DIACRITICS_NO_SHORT_CIRCUIT` mode, text without marks is decomposed but never recomposed

**ID:** `HHW-10` · **Tag:** `hhw-20261117`

**Summary.** The Indic, Khmer, Myanmar and USE shapers normalize in `HB_OT_SHAPE_NORMALIZATION_MODE_COMPOSED_DIACRITICS_NO_SHORT_CIRCUIT`, documented as “Always fully decomposes and then recompose back” (`src/hb-ot-shape-normalize.hh:43`). Round 1 does fully decompose every character in this mode. But rounds 2 and 3 are skipped when `all_simple` is true, that is, when the input had no marks (`src/hb-ot-shape-normalize.cc:324`, `:356`, `:374`, `:422`). So whether a character is recomposed depends on whether *some other* character in the buffer is a mark.

**Repro.** A font with glyphs for U+0929 (NNNA), U+0928 + U+093C, U+093E, and for é and e + U+0301 (`norm.ttf` in the book’s test fonts):

```
$ hb-shape norm.ttf $'\u0929'
[na=0+600|nukta=0+0]
$ hb-shape norm.ttf $'\u0929\u093E'
[nnna=0+600|aaMatra=0+250]
$ hb-shape norm.ttf $'\u00E9' --script=deva
[e=0+500|acutecomb=0+0]
$ hb-shape norm.ttf $'\u00E9a\u0301' --script=deva
[eacute=0+500|aacute=1+500]
```

**Expected.** The same glyphs for NNNA, and for é, whatever else is in the buffer: recomposed, as the mode’s comment says.

**Possible fix.** In this mode, don’t let `all_simple` skip round 3 when round 1 decomposed anything (or track “decomposed something” separately from “had marks”).

**On main.** Fixed on 17 November 2026 by `bc7dff9a5` (#6705). U+0929 alone now gives `[nnna=0+600]`, and é under `--script=deva` gives `[eacute=0+500]`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 11. The not-found glyph is ignored in variation-selector clusters

**ID:** `HHW-11` · **Tag:** `hhw-20261117`

**Summary.** `handle_variation_selector_cluster` maps the base with `set_glyph` (`src/hb-ot-shape-normalize.cc:79`), which calls `get_nominal_glyph` without falling back to `buffer->not_found`. Everywhere else, a character the font lacks gets the not-found glyph set with `hb_buffer_set_not_found_glyph`; here it gets glyph 0.

**Repro.** With `norm.ttf`, which has an unencoded glyph 27 and no `x`:

```
$ hb-shape norm.ttf q --not-found-glyph=27
[missing=0+500]
$ hb-shape norm.ttf $'x\uFE00' --not-found-glyph=27
[.notdef=0+500|space=0+0]
```

**Expected.** `missing` for the `x` in the second run too.

**Possible fix.** Fall back to `buffer->not_found` in `set_glyph`’s callers in the variation-selector path, as `decompose_current_character` does.

**On main.** Fixed on 17 November 2026 by `97d879446` (#6699): the second run now gives `[missing=0+500|space=0+0]`. An open pull request, #6719 (`codex/upstream-report-11-restore-backends`), only restores a test.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 12. Mark-to-base and mark-to-ligature pass over a base that lacks the lookup’s mask bit

**ID:** `HHW-12` · **Tag:** `hhw-20261117`

**Summary.** When a mark-to-base or mark-to-ligature lookup looks back for the mark’s base, it treats a glyph without the lookup’s mask bit as if it were something to skip (`src/OT/Layout/GPOS/MarkBasePosFormat1.hh:113`–`127`, `MarkLigPosFormat1.hh:102`–`116`). So turning `mark` off on a base doesn’t stop marks attaching to it: it makes them attach to an *earlier* base instead.

**Repro.** `gpos.ttf` from the book’s test fonts: `a`, `e` and a combining acute with mark anchors on both bases.

```
$ hb-shape gpos.ttf $'ae\u0301' --cluster-level=1
[a=0+500|e=1+500|acutecomb=2@-250,500+0]
$ hb-shape gpos.ttf $'ae\u0301' --cluster-level=1 --features='mark[1:2]=0'
[a=0+500|e=1+500|acutecomb=2@-750,500+0]
```

**Expected.** With `mark` off on the `e`, the acute either attaches to nothing or is unaffected (the acute itself still has the bit). It shouldn’t move 750 units back onto the `a`.

**Possible fix.** In the base search, stop at (rather than skip) a non-mark glyph that lacks the mask bit, and decline to attach.

**On main.** In progress: pull request #6736 (branch `codex/upstream-report-12`, “Stop mark base searches at non-matching glyphs”), not merged. On `main` the acute still moves to −750.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6736.

---

## 13. The nesting limit still breaks long cursive chains that run against the buffer’s direction

**ID:** `HHW-13` · **Tag:** `hhw-20261117`

**Summary.** `propagate_attachment_offsets` resolves attachment chains with a depth limit of `HB_MAX_NESTING_LEVEL`, 64 (`src/OT/Layout/GPOS/GPOS.hh:93`). Issue #5514 changed the loop order (`:177`) so that chains in the buffer’s direction resolve in one pass, but chains that run against it still recurse, and past 64 links the offsets stop accumulating. That covers LTR text with the RightToLeft lookup flag, and RTL text without it, which is the usual case for Arabic cursive joining.

**Repro.** `gpos.ttf`, whose `ss03` gives `n` a cursive attachment that steps each glyph 100 units, against the direction:

```
$ hb-shape gpos.ttf nnnn…n (67 n) --features=-curs,ss03
```

The offsets step by 100 until glyph 65, where two consecutive glyphs have the same offset, and every glyph before it is 100 units off. The book’s writer saw the same with 70 Arabic behs in RTL.

**Expected.** Chains of any length resolve correctly, or a limit high enough that real text never reaches it. Long cursive chains do occur in Arabic, Nastaliq especially.

**On main.** Still present. The output is unchanged: glyphs 64 and 65 are both at −100.


**In 15.2.0 and on main (20 November 2026).** Declined upstream: #6753 closed, the 64-link limit is kept on purpose.

---

## 14. With `HB_NO_HINTING`, y Device tables fall through to the variation branch

**ID:** `HHW-14` · **Tag:** `hhw-20261117`

*Found by reading the code; not testable in the book’s build.*

**Summary.** In `Device::get_y_delta` (`src/hb-ot-layout-common.hh:6430`), the labels `case 1: case 2: case 3:` sit outside `#ifndef HB_NO_HINTING`. In a build with `HB_NO_HINTING` but not `HB_NO_VAR`, a hinting Device table (formats 1–3) therefore falls through into `case 0x8000` and is read as a VariationIndex. `get_x_delta` (`:6410`) has the case labels inside the guard, as both should.

**Possible fix.** Move the case labels inside the `#ifndef HB_NO_HINTING` block, as in `get_x_delta`.

**On main.** Fixed on 17 November 2026 by `489157f82` (#6694). Checked in the code.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 15. A cross-stream `kern` value of 0x8000 isn’t treated as a reset

**ID:** `HHW-15` · **Tag:** `hhw-20261117`

**Summary.** The `kern` specification says “The value 0x8000 in the kerning data resets the cross-stream kerning back to 0.” Only the format 1 state machine checks for it (`src/hb-aat-layout-kerx-table.hh:360`). The kern machine, which runs formats 0, 2 and 3 (`src/hb-kern.hh`), takes 0x8000 as −32,768.

**Repro.** `cross.ttf` in the book’s test fonts: a cross-stream format 0 subtable with A V +100 and T o 0x8000.

```
$ hb-shape cross.ttf AVTo --language=en
[A=0+560|V=1@-40,100+560|T=2@0,100+550|o=3@-50,-32668+450]
```

**Expected.** The `o` back on the line (y offset 0).

**On main.** In progress: pull request #6732 (branch `codex/upstream-report-15`, “Honor cross-stream pair reset values”), not merged. On `main` the `o` is still at −32,668.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6732.

---

## 16. Fallback mark positioning ignores the base’s x offset

**ID:** `HHW-16` · **Tag:** `hhw-20261117`

**Summary.** The fallback positioner (`src/hb-ot-shape-fallback.cc`) builds the base’s box from its advance and ink and adds the base’s y offset, but not its x offset. After kerning that splits a value between two glyphs (the `kern` table, or fallback kerning), the second glyph has an x offset, and a mark on it is placed off-center by that amount.

**Repro.** `kern-table.ttf` (no GPOS, a `kern` table with o o −50) against `fallback.ttf` (the same without `kern`):

```
$ hb-shape fallback.ttf $'o\u00F3' --language=en
[o=0+500|o=1+500|acutecomb=1@-25,12+0]
$ hb-shape kern-table.ttf $'o\u00F3' --language=en
[o=0+450|o=1@-50,0+450|acutecomb=1@25,12+0]
```

**Expected.** The acute centered over the kerned `o`, as it is over the unkerned one.

**On main.** Fixed on 17 November 2026 by `641971dd1` (#6701). Over the kerned `o` the acute is now at −25, as over the unkerned one.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 17. Fallback positioning centers LEFT and RIGHT marks, contrary to its comment

**ID:** `HHW-17` · **Tag:** `hhw-20261117`

**Summary.** `position_mark` in `src/hb-ot-shape-fallback.cc` says “We don’t position LEFT and RIGHT marks” (`:228`), but the switch has no cases for `HB_UNICODE_COMBINING_CLASS_LEFT` (224) and `RIGHT` (226), so they fall into `default` and are centered on the base, and lose their advance like every placed mark.

**Repro.** U+1D16D MUSICAL SYMBOL COMBINING AUGMENTATION DOT (class 226) after `o`, in a font without GPOS:

```
$ hb-shape fallback.ttf $'o\U0001D16D' --language=en
[o=0+500|augdot=0@-350,0+0]
```

**Expected.** Either leave such marks where they are (as the comment says), or place them to the left or right of the base.

**On main.** In progress, as documentation: pull request #6742 (branch `codex/upstream-report-17-fallback-docs`) rewrites the comment as “Classes without dedicated placement are centered horizontally”, keeping the centering, which it says commit `e1ba62811a` added on purpose. Not merged; `main` is unchanged.


**In 15.2.0 and on main (20 November 2026).** Changed in 15.2.0: #6742 rewrites the comment and keeps the centering, which upstream says is deliberate.

---

## 18. Vertical `kern` subtables can never apply

**ID:** `HHW-18` · **Tag:** `hhw-20261117`

**Summary.** In vertical text the plan’s kerning mask is `vkrn`’s (`src/hb-ot-shape.cc:127`). `vkrn` isn’t on by default, and unlike `kern` it has no `F_HAS_FALLBACK`, so when GPOS lacks it the feature isn’t kept and its mask is 0. Subtables of the `kern` table marked for vertical text therefore never run, even with `--features=vkrn`.

**Repro.** A `kern` table with a vertical (non-horizontal) format 0 subtable, shaped with `--direction=ttb`, with and without `--features=vkrn`: no kerning either way.

**Possible fix.** Give `vkrn` `F_HAS_FALLBACK` when it is requested, or consult the vertical `kern` subtables under `vert`’s default kerning, whichever matches the intended policy.

**On main.** In progress: pull request #6734 (branch `codex/upstream-report-18`, “Keep fallback masks for requested vertical kerning”), not merged. The code on `main` is unchanged (not rerun: the book has no font for it).


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6734.

---

## 19. The Hangul shaper’s dotted circle copies the tone mark’s properties

**ID:** `HHW-19` · **Tag:** `hhw-20261117`

**Summary.** For a tone mark that doesn’t follow a syllable, `preprocess_text_hangul` inserts a dotted circle with `replace_glyphs` (`src/hb-ot-shaper-hangul.cc:245`), which copies the tone mark’s buffer entry, Unicode properties included. The circle is then a spacing mark (Mc, class 224), so after a base the fallback positioner zeroes its advance and places it over the base. The generic dotted circle gets U+25CC’s own properties (`src/hb-ot-shape.cc:567`).

**Repro.** `hangul.ttf` in the book’s test fonts (no GPOS):

```
$ hb-shape hangul.ttf $'\u1100\u302E'
[kiyeok=0+500|singledot=0@-350,0+0|dottedcircle=0@-500,0+0]
```

**Expected.** The dotted circle as a base with its own advance, carrying the tone mark.

**On main.** Fixed on 17 November 2026 by `2831a8a45` (#6711). The circle now keeps its own advance: `[kiyeok=0+500|singledot=0@-350,0+0|dottedcircle=0+500]`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 20. Two dotted circles before a Hangul tone mark at the start of text

**ID:** `HHW-20` · **Tag:** `hhw-20261117`

**Summary.** With `HB_BUFFER_FLAG_BOT`, the generic dotted circle is inserted before a leading mark (`src/hb-ot-shape.cc:567`). The Hangul shaper doesn’t recognize that circle as a syllable, so it inserts a second one.

**Repro.**

```
$ hb-shape hangul.ttf $'\u302E' --bot
[dottedcircle=0+500|singledot=0@-350,0+0|dottedcircle=0@-500,0+0]
```

**Expected.** One dotted circle.

**On main.** Fixed on 17 November 2026 by `59fa8f0c2` (#6712), which accepts a dotted circle as the tone mark’s base. There is now one circle: `[singledot=0+200|dottedcircle=0+500]`, the spacing tone mark moved before its base as the Hangul shaper does with any spacing tone mark.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 21. Hebrew: `has_gpos_mark` and `gpos_tag` disagree about a `mark` feature under `DFLT`

**ID:** `HHW-21` · **Tag:** `hhw-20261117`

**Summary.** The Hebrew shaper composes presentation forms only when the plan has no `mark` feature (`has_gpos_mark`, `src/hb-ot-shape.cc:196`; `hb-ot-shaper-hebrew.cc:81`). Its `gpos_tag` makes GPOS apply only when GPOS chose `hebr`. In a font whose `mark` feature is under `DFLT` only, `has_gpos_mark` is true, so presentation forms aren’t composed, but GPOS is ignored because of `gpos_tag`. The points fall to fallback positioning though the font has presentation-form glyphs that would serve better.

**Repro.** `hebrew-mark-DFLT.ttf` in the book’s test fonts: shin with dagesh and shin dot gives separate glyphs placed by the fallback positioner, where `hebrew-old.ttf` (no GPOS) gives U+FB2C’s glyph.

**Possible fix.** Compute `has_gpos_mark` from the GPOS the plan will actually apply (after the `gpos_tag` test).

**On main.** In progress: pull request #6738 (branch `codex/upstream-report-21`, “Ignore unused GPOS mark features for fallback”), not merged. On `main`, `hebrew-mark-DFLT.ttf` still gives three glyphs placed by the fallback positioner.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6738.

---

## 22. Indic `init` ignores pre-context

**ID:** `HHW-22` · **Tag:** `hhw-20261117`

**Summary.** The Indic shaper applies `init` to a pre-base matra at the start of a word, testing for a word start only inside the buffer (`src/hb-ot-shaper-indic.cc:1431`). With the previous consonant supplied as pre-context, the syllable still gets the word-initial form.

**Repro.** `full.ttf` in the book’s test fonts (an `init` lookup for the I matra):

```
$ hb-shape full.ttf $'\u0915\u093F'
[iMatra.init=0+250|ka=0+600]
$ hb-shape full.ttf $'\u0915\u093F' --text-before=$'\u0915'
[iMatra.init=0+250|ka=0+600]
$ hb-shape full.ttf $'\u0915\u0915\u093F'
[ka=0+600|iMatra=1+250|ka=1+600]
```

**Expected.** With a letter before it in the pre-context, the matra isn’t word-initial, as when the text is shaped whole.

**On main.** Fixed on 17 November 2026 by `5795f8699` (#6713). With KA as pre-context the matra is now `iMatra`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 23. Syllables longer than 64 glyphs are cut into pieces

**ID:** `HHW-23` · **Tag:** `hhw-20261117`

*Seen by the book’s writer; not yet in the book.*

**Summary.** `foreach_syllable` caps a syllable at `HB_MAX_SYLLABLE_LENGTH`, 64 (`src/hb-limits.hh:62`). A longer syllable is processed in pieces, so reordering can go wrong at the cut: with 32 TA + virama pairs followed by KA + I, the matra lands mid-syllable and the 32nd TA keeps a visible virama. The cap also makes the `end - start > 127` branch in `hb-ot-shaper-indic.cc:810` unreachable.

**Before filing.** Reproduce with a real Devanagari font and confirm the cut is the cause; a limit is reasonable, but splitting silently mid-syllable might be worse than treating the whole run as broken.

**On main.** Changed, not fixed. Commit `fd5702517` (#6558, 10 November 2026) cuts the syllable into the same pieces in every pass and makes the run one cluster, but the reordering still goes wrong at the cut. With `test/subset/data/fonts/NotoSansDevanagari-Regular.ttf`, 32 TA + virama followed by KA + I gives the same glyphs on `main` as on 15.1.0, all now in cluster 0: the last TA keeps a visible virama, and the I matra stands just before KA rather than at the start of the syllable.


**In 15.2.0 and on main (20 November 2026).** Unchanged: Changed, not fixed: #6558 keeps the pieces the same in every pass and makes the whole syllable one cluster, but this draft’s example is still reordered wrongly at the cut; the pull requests named are in 15.2.0.

---

## 24. `stch` tiles are shifted when they fit the word exactly

**ID:** `HHW-24` · **Tag:** `hhw-20261117`

**Summary.** In `apply_stch` (`src/hb-ot-shaper-arabic.cc:625`), the first tile starts at `x_offset = w_remaining / 2`. `w_remaining` is set to zero only on the path that adds an extra, overlapping repeat (`:598`). When the repeating tiles fill the word exactly, or the word is narrower than the fixed tiles plus one repeat, the whole line is shifted by half of what remains.

**Repro.** `stch.ttf` in the book’s test fonts: U+070F SYRIAC ABBREVIATION MARK followed by two beths (a word 1,000 units wide).

```
$ hb-shape stch.ttf $'\u070F\u0712\u0712' --script=Syrc
[beth=2+500|beth=1+500|sam.end=0@-600,0+0|sam.rep=0@-500,0+0|sam.rep=0@-300,0+0|sam.rep=0@-100,0+0|sam.rep=0@100,0+0|sam.end=0@300,0+0]
```

The tiles run from −600 to 400, extending 400 units past the word, where −1000 to 0 would cover it. With one or three beths the overlap path is taken and the fit is right. HarfBuzz’s own test (`test/shape/data/in-house/tests/arabic-stch.tests`) happens to go through the overlap path.

**Possible fix.** Zero `w_remaining` whenever the tiles fill the word, not only on the overlap path.

**On main.** In progress: pull request #6733 (branch `codex/upstream-report-24`, “Align exact-fit stretch tiles”), not merged. On `main` the tiles still run from −600.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6733 and #6763.

---

## 25. CBDT format 19 images are never painted

**ID:** `HHW-25` · **Tag:** `hhw-20261117`

*Found by reading the code; not shown live.*

**Summary.** `CBDT::paint_glyph` needs the image’s pixel extents, but for format 19 (PNG with metrics in the CBLC index) they come from `IndexSubtable::get_extents`, which is a TODO that always returns false (`src/OT/Color/CBDT/CBDT.hh:334`). So a format 19 glyph can be fetched with `hb_ot_color_glyph_reference_png` but is never painted. Only index subtable formats 1 and 3 are read (`:345`).

**On main.** Still present: `IndexSubtable::get_extents` still handles only formats 1 and 3, with the TODO for the rest (`CBDT.hh:334` on `main`). Checked in the code.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6754 (reading and painting; the subset writer still handles only index formats 1 and 3).

---

## 26. The bitmap tables are tried in different orders for different purposes

**ID:** `HHW-26` · **Tag:** `hhw-20261117`

**Summary.** `hb_ot_color_glyph_reference_png` tries `sbix` before CBDT (`src/hb-ot-color.cc:417`), and so do glyph extents, but painting tries CBDT first. For a glyph in both tables, a client can get one table’s image from one call and the other’s from another. Low impact (fonts rarely have both), but the order should be one.

**On main.** Fixed on 17 November 2026 by `7e97cd8a1` (#6723): painting now tries `sbix` before CBDT, like the other two. Checked in the code.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 27. Point matching in nested composites counts from the wrong start

**ID:** `HHW-27` · **Tag:** `hhw-20261117`

*Shown in a drawing in the book; the expected position not yet worked out independently.*

**Summary.** When a `glyf` component is placed by point matching (`ARGS_ARE_XY_VALUES` clear), the anchor point number of the parent is counted from the start of the whole `all_points` array (`src/OT/glyf/Glyph.hh:539`–`551`), not from the start of the inner composite’s own points. At nesting depth above 0, the outer glyph’s earlier points are counted too, so the component is attached to the wrong point.

**Repro.** `outlines.ttf` in the book’s test fonts: `x` is a composite of `b` and `i`, with `i` placed by point matching inside a nested composite. HarfBuzz draws the dot of the `i` 250 units too high; fontTools places it correctly.

**Before filing.** Confirm the expected position against FreeType or fontTools’ pen output, and reduce the font to the smallest case.

**On main.** Still present. `Glyph.hh` is unchanged, and `x` has the same extents.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6749.

---

## 28. A CFF charstring with no `endchar` runs to the operation limit

**ID:** `HHW-28` · **Tag:** `hhw-20261117`

**Summary.** In CFF1, running off the end of a charstring isn’t treated as an error: `fetch_op` returns `OpCode_Invalid`, which only clears the arguments, and the interpreter loops until `HB_CFF_MAX_OPS` (200,000). The outline so far is emitted to the client, then `hb_font_draw_glyph_or_fail` returns false, and the glyph has no extents. Each such glyph uses 200,000 of the shared drawing budget, so a few of them stop later glyphs from being drawn at all.

**Repro.** A CFF font with 90 charstrings lacking `endchar` and one good glyph: `hb-vector` draws 84 paths and the good glyph vanishes (checked by the book’s writer with `hb-vector` and a small C client).

**Expected.** Running off the end ends the charstring (as an implied `endchar`, or as an error) at once.

**On main.** In progress: pull request #6737 (branch `codex/upstream-report-28`, “Fail promptly when opcode bytes are missing”), which makes `fetch_op` set an error when the charstring runs out. Not merged; not rerun.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6737.

---

## 29. Khmer moves a second COENG + RO in front of the first

**ID:** `HHW-29` · **Tag:** `hhw-20261117`

**Summary.** In Khmer initial reordering (`src/hb-ot-shaper-khmer.cc:239`), the test `num_coengs <= 2` is checked before the counter is incremented, so a third COENG still passes; and `num_coengs = 2; /* Done. */` after moving a COENG + RO doesn’t stop the loop. So a second COENG + RO in the same syllable is also moved before the base, in front of the first. Microsoft’s Khmer specification says the move applies while the “subscript count is less than two”.

**Repro.** `khmer.ttf` in the book’s test fonts:

```
$ hb-shape khmer.ttf $'\u1780\u17D2\u179A\u17D2\u179A'
[ro.pref=0+300|ro.pref=0+300|ka=0+600]
```

**Expected.** Only the first COENG + RO moved. Low impact: the text is already invalid.

**Possible fix.** `break` after moving the COENG + RO, and test `num_coengs < 2`.

**On main.** In progress: pull request #6735 (branch `codex/upstream-report-29`, “Stop reordering coengs at the subscript limit”), not merged. On `main` both ROs still move.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6735.

---

## 30. USE: a mark after a space attaches to the space

**ID:** `HHW-30` · **Tag:** `hhw-20261117`

**Summary.** In the USE grammar, `symbol_cluster = (O | GB | SB) tail?` (`src/hb-ot-shaper-use-machine.rl`) lets a space (category `O`) take any marks after it. A vowel sign typed after a space is then reordered before the space, with no dotted circle. Microsoft’s specification lets an independent cluster of category `O` take only a variation selector.

**Repro.** `deva3.ttf` in the book’s test fonts (Devanagari under `dev3`), against `deva2.ttf` (`dev2`, the Indic shaper):

```
$ hb-shape deva3.ttf $'\u0915 \u093F'
[ka=0+600|iMatra=1+250|space=1+250]
$ hb-shape deva2.ttf $'\u0915 \u093F'
[ka=0+600|space=1+250|iMatra.init=1+250|dottedcircle=1+500]
```

**On main.** Still present: the output is unchanged.

**Background** (added 18 November 2026). The permissive tail is deliberate: “allow any non-numeric tail in a symbol cluster” (`NEWS` for 4.0.1), and Microsoft’s engine lets symbol clusters take consonant modifiers too, which the specification was changed to allow in 2024 (MicrosoftDocs/typography-issues#1179). This report is about the space only. The specification puts white space (UGC=Zs) in `O`, an independent cluster, which takes nothing but a variation selector; HarfBuzz merges `O` with `IND` and `S` (`gen-use-table.py:271`), and so a space takes marks.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6765.

---

## 31. USE: a space doesn’t end a word for topographical forms

**ID:** `HHW-31` · **Tag:** `hhw-20261117`

**Summary.** HarfBuzz gives non-joining USE scripts its own topographical forms (`isol`/`init`/`medi`/`fina`), which Microsoft’s specification says aren’t implemented for them at all. A space is a symbol cluster, and symbol clusters join, so a syllable after a space isn’t word-initial and one before it isn’t final. ZWNJ doesn’t break the chain either; only a non-cluster such as WORD JOINER does.

**Repro.**

```
$ hb-shape deva3.ttf $'\u0915\u093F \u0915\u093F'
[iMatra.init=0+250|ka=0+600|space=2+250|iMatra=3+250|ka=3+600]
```

The second कि, after a space, doesn’t get `init`.

**On main.** Still present: the output is unchanged.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6764 (the report’s own repro now gives plain iMatra for both: each one-syllable word is now isol, and deva3.ttf has no isol; two-syllable words separated by a space each get init).

---

## 32. USE: ZWNJ after a virama doesn’t stop a half form

**ID:** `HHW-32` · **Tag:** `hhw-20261117`

**Summary.** Under the Indic shaper, ZWNJ after a virama stops a half form (`F_MANUAL_ZWNJ` on the basic features). USE’s features lack `F_MANUAL_ZWNJ`, so the lookup passes over the ZWNJ and the half form is made.

**Repro.**

```
$ hb-shape deva3.ttf $'\u0915\u094D\u200C\u0930'
[ka.half=0+400|space=2+0|ra=3+500]
$ hb-shape deva2.ttf $'\u0915\u094D\u200C\u0930'
[ka=0+600|virama=0+0|space=2+0|ra=3+500]
```

**Expected.** As under `dev2`: a visible virama. (Also seen: RA + virama + ZWJ still makes a reph under USE, where the Indic shaper gives none.)

**Also noticed, not a bug report:** `hb-ot-shaper-use-table.hh` records `DerivedCoreProperties-17.0.0` while every other input is 18.0.0, apparently because `update-unicode-tables.make` leaves that file out of `SOURCES`.

**On main.** Still present: the output is unchanged. For the note above: `update-unicode-tables.make` now lists `DerivedCoreProperties.txt` (report 53), but the USE table hasn’t been regenerated, and its header still records `DerivedCoreProperties-17.0.0`.


**In 15.2.0 and on main (20 November 2026).** Declined upstream: #6768 closed unmerged, “Indic3 tags are a novelty only”.

---

## 33. Indic: a broken cluster starting with RA + virama loses its reph

**ID:** `HHW-33` · **Tag:** `hhw-20261117`

**Summary.** When the Indic syllable machine finds a broken cluster, `hb_syllabic_insert_dotted_circles` (`src/hb-ot-shaper-syllabic.cc`) inserts the circle after any leading *repha character*, but the Indic shaper passes only the category `Repha`, which in the Indic table is just U+0D4E. So RA + virama + a vowel sign gets its circle *before* the RA: the reph never forms, and a visible virama follows the circle.

**Repro.** `circle.ttf` in the book’s test fonts (Devanagari with `rphf` and a dotted circle):

```
$ hb-shape circle.ttf $'\u0930\u094D\u093F'
[iMatra=0+250|dottedcircle=0+500|ra=0+500|virama=0+0]
$ hb-shape circle.ttf $'\u0930\u094D\u25CC\u093F'
[iMatra=0+250|dottedcircle=0+500|reph=0+0]
```

**Expected.** The circle after RA + virama, as when it is typed, so that a reph forms on it.

**On main.** Still present: the output is unchanged.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only (after 15.2.0) by #6830.

---

## 34. A ZWJ defeats the vowel constraints

**ID:** `HHW-34` · **Tag:** `hhw-20261117`

**Summary.** The vowel constraints (`src/hb-ot-shaper-vowel-constraints.cc`, generated from Microsoft’s `IndicShapingInvalidCluster.txt`) match only adjacent code points. Putting a ZWJ between an independent vowel and a vowel sign avoids the dotted circle, which defeats the anti-spoofing purpose (issue #1019).

**Repro.**

```
$ hb-shape circle.ttf $'\u0905\u093E'
[a.deva=0+600|dottedcircle=0+500|aaMatra=0+250]
$ hb-shape circle.ttf $'\u0905\u200D\u093E'
[a.deva=0+600|space=0+0|aaMatra=0+250]
```

**Expected.** A circle in both, or a documented reason why ZWJ is allowed through.

**On main.** Still present: the output is unchanged.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6767.

---

## 35. The vowel-constraint dotted circle copies the next character’s properties

**ID:** `HHW-35` · **Tag:** `hhw-20261117`

**Summary.** The circle is inserted with `output_glyph`, which copies `cur()` (`src/hb-buffer.hh:313`), so it takes the following vowel sign’s Unicode properties: general category Mn. In a font without GDEF, synthesized classes then make it a mark, and USE zeroes its advance and moves it back over the letter. Compare report 19 (the Hangul circle copying the tone mark). The circles from the other paths are inserted with U+25CC’s own properties (generic) or with zero properties (syllabic).

**Repro.** `bare.ttf` in the book’s test fonts (no GDEF or GPOS): Brahmi A + AA.

**Possible fix.** Give every inserted dotted circle U+25CC’s own Unicode properties, in one helper used by all four paths.

**On main.** Fixed on 17 November 2026 by `74fc8c86c` (#6710). With `--unicodes=U+11005,U+11038`, `bare.ttf` now gives `[a.brah=0+600|dottedcircle=0+500|aa.brah=0+0]`: the circle keeps its advance.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 36. hb-vector’s PDF output drops PNG alpha

**ID:** `HHW-36` · **Tag:** `hhw-20261117`

**Summary.** In `src/hb-vector-paint-pdf.cc:305`, `(void) has_alpha;`: RGBA and gray+alpha PNGs are embedded as `/DeviceRGB` or `/DeviceGray` images with `/Colors 4` or `2` and no `/SMask`. The alpha channel is read as color, and the image renders as stripes.

**Repro.** A font with an RGBA PNG bitmap glyph (`images.ttf` in the book’s test fonts): `hb-vector images.ttf P -O pdf`, rendered with `pdftoppm`.

**Expected.** The alpha channel split into an `/SMask` image, as PDF requires.

**On main.** Still present: `(void) has_alpha;` is still at `hb-vector-paint-pdf.cc:305`. Checked in the code.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6758.

---

## 37. Smaller hb-vector and hb-gpu inconsistencies

**ID:** `HHW-37` · **Tag:** `hhw-20261117`

* PDF: linear and radial gradients with REPEAT or REFLECT extend modes only pad (`hb-vector-paint-pdf.cc:1099`, `:1200`), so they lose their repeats. Possibly a deliberate limitation; worth documenting if so.
* Stale comments and docs: `hb-vector-paint-pdf.cc:49` says sweeps are “approximated via solid fallback” (they are Coons-patch meshes); the `hb-gpu.cc` docs example calls `hb_gpu_draw_encode (draw)` without the `extents` argument added in 14.2.0; `README.md` says the vector library is “currently SVG”; the `HB_VECTOR_MAX_DOCUMENT_SIZE` comment in `hb-limits.hh` says “one glyph per context”.
* API: `hb_vector_paint_set_palette` takes `int` where the raster and GPU versions take `unsigned`, and `hb_vector_paint_set_custom_palette_color` returns `void` where theirs return `hb_bool_t`.

**On main.** Partly fixed. `24fe9f0e2` (#6691, 17 November 2026) corrects the sweep comment, the `hb-gpu.cc` example, `README.md` and the `hb-limits.hh` comment. `aa3cd40bc` (#6730) documents that PDF linear and radial gradients support only PAD, in `hb-vector.cc` and the PDF backend’s list, so the first item is now a documented limitation. The API differences are still present (`hb-vector.h:281`, `:288` on `main`).


**In 15.2.0 and on main (20 November 2026).** Unchanged: Partly fixed: #6691 corrects the stale comments and documentation, and #6730 documents that PDF gradients only pad; the API differences remain; the pull requests named are in 15.2.0.

---

## 38. A font with GSUB, GPOS and `kerx` never applies `kerx`, and stops zeroing marks

**ID:** `HHW-38` · **Tag:** `hhw-20261117`

**Summary.** When a font has GSUB and GPOS, and GPOS has no `kern` feature in the language system it chose, and the font also has `kerx`, the planner sets both `apply_gpos` and `apply_kerx` (`src/hb-ot-shape.cc:167`–`195`). `hb_ot_shape_plan_t::position` runs only GPOS, so `kerx` never kerns. And because the plan thinks `kerx` is in use, marks are no longer zeroed.

**Repro.** `mark-gpos.ttf` (GSUB, GPOS without `kern`) and `mark-gpos-kerx.ttf` (the same plus a `kerx` A V −40) in the book’s test fonts:

```
$ hb-shape mark-gpos.ttf $'a\u0301AV' --language=en
[.notdef=0+500|acutecomb=0+0|A=2+600|V=3+600]
$ hb-shape mark-gpos-kerx.ttf $'a\u0301AV' --language=en
[.notdef=0+500|acutecomb=0+200|A=2+600|V=3+600]
```

**Expected.** Either `kerx` applies after GPOS (A V −40) or the plan doesn’t claim it; and marks zeroed as before.

**On main.** Still present: the output is unchanged.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6761 (cross-stream kerx now skipped when GPOS is active; kerx pair/class kerning applies with GPOS when GPOS has no kerning feature).

---

## 39. `trak` vertical tracking is added with the wrong sign

**ID:** `HHW-39` · **Tag:** `hhw-20261117`

**Summary.** `trak` adds its vertical value to `y_advance` without negating it (`src/hb-aat-layout-trak-table.hh:242`). Vertical advances are negative in HarfBuzz, so positive tracking, which should loosen a column, tightens it. GPOS negates in the same situation.

**Repro.** `trak.ttf` in the book’s test fonts (vertical track 0 = +100 at 12 pt):

```
$ hb-shape trak.ttf AV --direction=ttb --font-ptem=12
[A=0@-300,-750+0,-900|V=1@-300,-750+0,-900]
```

Without `trak` the advance is −1000; with +100 tracking it should be −1100.

**On main.** Fixed on 17 November 2026 by `45c7759a0` (#6700), with a follow-up for the Core Text font functions, `1e2cf5daf` (#6716). The advances are now −1,100.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 40. `kerx` format 4 control-point attachment does nothing with the default font functions

**ID:** `HHW-40` · **Tag:** `hhw-20261117`

**Summary.** Format 4’s control-point actions (type 0) ask the font functions for contour points, which hb-ot doesn’t implement (`glyph_contour_point` is commented out in `hb-ot-font.cc`). Unlike GPOS anchor format 2, there is no fallback to coordinates, so nothing is attached. Also `HB_DIRECTION_LTR /*XXX*/` at `hb-aat-layout-kerx-table.hh:628`.

**Repro.** `points.ttf` in the book’s test fonts: the marked glyph stays unattached with the default `ot` funcs.

**On main.** Still present: the output is unchanged, and `HB_DIRECTION_LTR /*XXX*/` is still there (`hb-aat-layout-kerx-table.hh:661`, `:665` on `main`).


**In 15.2.0 and on main (20 November 2026).** Still present in 15.2.0 and on main at `8846dbbf6`.

---

## 41. In AAT, the first of two requests for one feature wins

**ID:** `HHW-41` · **Tag:** `hhw-20261117`

**Summary.** For `morx` fonts, the AAT map sorts requested features by type, selector pair and then sequence, and keeps the first (`src/hb-aat-map.cc:139`–`155`). The OpenType map lets the later request win (`hb-ot-map.cc:238`). So the same feature list does opposite things in the two kinds of font.

**Repro.** `latin.ttf` (a `morx` font with an f i ligature) against `ligatures.ttf` (GSUB), both from the book’s test fonts:

```
$ hb-shape latin.ttf fi --features=-liga,liga
[f=0+500|i=1+500]
$ hb-shape ligatures.ttf fi --features=-liga,liga
[f_i=0+900]
```

**Expected.** The later request wins in both, as `hb_shape`’s documentation of feature lists implies.

**On main.** Fixed on 17 November 2026 by `f346ea17a` (#6714). `latin.ttf` with `--features=-liga,liga` now gives `[f_i=0+900]`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 42. hb-depend: documentation errors and loose ends

**ID:** `HHW-42` · **Tag:** `hhw-20261117`

* `src/hb-subset-depend.h:90` documents `layout_tag` as 0 for edges not from GSUB; it is `HB_CODEPOINT_INVALID` (0xFFFFFFFF), as a program built against the API shows.
* `docs/depend-implementation.md`: says an edge takes only the first feature using its lookup (the code makes one edge per feature, `hb-depend-data.hh:559`); names the header `hb-depend.h`; calls the utility “TODO” though `util/hb-subset-depend.cc` exists; shows `lookup_features[256]`; gives the closure of {f} under `f i → f_i` as {f, fi}, where `Ligature.hh:35` gives {f}.
* `hb_depend_data_builder_t::get_gsub_dependencies` is declared (`hb-depend-data.hh:650`) but never defined; the builder fields `unicodes` and `nominal_glyphs` are unused.
* `test/fuzzing/README.md:18` asks for `-Ddepend_api=true`; the option is `experimental_api`.
* The closure parity test (`hb-depend-closure-parity.cc:27`, `:326`) says it mirrors the subsetter’s order but runs GSUB before MATH; the subsetter runs MATH first (`hb-subset-plan.cc:467`). A GSUB output with math variants could make the two disagree (inferred, not tried).

**On main.** Partly fixed. `88bb79a40` (#6703, 17 November 2026) corrects the `layout_tag` documentation and `docs/depend-implementation.md`, and `202b3e2c2` (#6692) the fuzzing README’s option (adding the GPU fuzzer to its list too, the note at the end of this file). Removing the undefined `get_gsub_dependencies` and the unused fields is in progress: pull request #6744 (branch `codex/upstream-report-42-unused-builder`), not merged. The closure parity test’s order is still present: its comment (`hb-depend-closure-parity.cc:27`) still says GSUB before MATH.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6744 and #6745.

---

## 43. The subsetter’s GSUB closure cuts long nested chains

**ID:** `HHW-43` · **Tag:** `hhw-20261117`

**Summary.** `hb_ot_layout_lookups_substitute_closure` runs at most `HB_CLOSURE_MAX_STAGES` + 2 = 14 passes, and glyphs produced by nested lookups aren’t flushed until the next pass (`SubstLookup.hh:209`). A context rule that applies many lookups in a row at one position loses the glyphs after the fourteenth.

**Repro.** `nested.ttf` in the book’s test fonts (one context rule applying 20 single substitutions in turn, a → s1 → … → s20):

```
$ hb-subset nested.ttf --text=a -o nested-a.ttf
$ hb-shape nested.ttf a
[s20=0+500]
$ hb-shape nested-a.ttf a
[s14=0+500]
```

**Expected.** The subset shapes as the original does.

**On main.** Changed, not fixed. `0209ec286` (#6725, 17 November 2026, for report 44) raises `HB_CLOSURE_MAX_STAGES` to 16, so the closure runs 18 passes: `nested-a.ttf` cut by `main` now shapes `a` as `s18`, still short of `s20`.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6757 (closure now reaches s20; the stage limit is still 16).

---

## 44. `HB_CLOSURE_MAX_STAGES` is smaller than its own comment requires

**ID:** `HHW-44` · **Tag:** `hhw-20261117`

**Summary.** `hb-limits.hh:68` says the limit “must be larger than the number of times add_gsub_pause() is called in a collect_features call of any shaper”. It is 12; `collect_features_indic` makes 14 such calls (three named pauses plus one after each of the eleven basic features, `hb-ot-shaper-indic.cc:249`–`265`).

**On main.** Fixed on 17 November 2026 by `0209ec286` (#6725), which raises `HB_CLOSURE_MAX_STAGES` to 16.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 45. MATH data is lost for glyphs that only GSUB produces

**ID:** `HHW-45` · **Tag:** `hhw-20261117`

**Summary.** The closure runs MATH before GSUB, and MathVariants is subset against `_glyphset_cmaped`, MathGlyphInfo against `_glyphset_mathed` (`hb-ot-math-table.hh:232`, `:1064`). Glyphs reached only through GSUB (`rtlm`, `ssty`, stylistic sets) keep their outlines but lose their size variants and italics corrections.

**Repro.** `math.ttf` in the book’s test fonts, subset with its `rtlm` forms; and HarfBuzz’s own `STIXTwoMath-Regular.ttf` with `ss08` kept: `uni222B.up` loses its variants and italics correction. Requesting the glyphs explicitly (`math-rtlm.ttf`) works around it.

**On main.** In progress: pull request #6745 (branch `codex/upstream-report-45-math-closure`, “Preserve MATH data on GSUB outputs”), not merged.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6745.

---

## 46. `hb_set_invert`’s “Since” is wrong

**ID:** `HHW-46` · **Tag:** `hhw-20261117`

**Summary.** `src/hb-set.cc:505` documents `hb_set_invert` as “Since: 3.0.0”. `NEWS` has it deprecated in 1.6.1 and undeprecated in 2.9.1, so it has existed since before 1.6.1. Documentation only.

**On main.** Fixed on 17 November 2026 by `72e4cf3a3` (#6690): now “Since: 0.9.10”, with a note of the deprecation in 1.6.1 and the restoration in 2.9.1.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 47. The sanitizer drops a whole valid GSUB whose SingleSubst lookups cover many glyphs

**ID:** `HHW-47` · **Tag:** `hhw-20261117`

**Summary.** `SingleSubstFormat1::sanitize` charges the operation budget half its Coverage population (`src/OT/Layout/GSUB/SingleSubstFormat1.hh:125`), and the budget is max(`HB_SANITIZE_MAX_OPS_MIN` 16384, `HB_SANITIZE_MAX_OPS_FACTOR` 64 × table length). Range-based Coverage makes such lookups tiny, so a small table with a few of them exceeds the budget, and the whole GSUB is rejected.

**Repro.** Two fonts of 20,001 glyphs, each lookup a SingleSubst format 1 over a range Coverage of about 20,000 glyphs. One lookup (72-byte GSUB): `hb-shape C1.ttf --unicodes=U+4E01 --features=ss01` gives `[g2=0+500]`. Three lookups (152-byte GSUB): `hb-shape C3.ttf --unicodes=U+4E01 --features=ss01` gives `[g1=0+500]`, and `ss02`, `ss03` likewise do nothing. `hb-subset` reports “sanitize failed on source table”.

**Expected.** A valid table accepted; a budget charge proportional to work the sanitizer actually does, or at least only the offending lookup dropped. Large CJK fonts with whole-repertoire stylistic sets could hit this.

**On main.** Still present: the sanitizer’s charge is unchanged (`SingleSubstFormat1.hh:126` on `main`). Checked in the code; the fonts weren’t rebuilt.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6751.

---

## 48. The subsetter’s 256× buffer cap can make a valid subset fail

**ID:** `HHW-48` · **Tag:** `hhw-20261117`

**Summary.** The subsetter retries a table with a doubled buffer up to 256 × the source table’s size (`src/hb-subset-table.hh:61`). A tiny source table whose subset legitimately grows larger (a SingleSubst format 1 over a range becoming format 2 once glyphs are dropped) hits the cap, and `hb-subset` fails with “Operation failed. Probably a bug.”

**Repro.** A 72-byte GSUB, one SingleSubst format 1 over 20,000 glyphs, subset with `--no-layout-closure` dropping glyphs whose IDs are multiples of 5 or 7 (checked by script in the chapter 36 work, not in the book).

**On main.** Still present: the cap is unchanged (`hb-subset-table.hh:62` on `main`). Checked in the code.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6752 (cap is now max(256x, 1 MiB); HB_SUBSET_MAX_TABLE_SIZE_FACTOR and _MIN).

---

## 49. `docs/serializer.md` and `docs/repacker.md` are out of date

**ID:** `HHW-49` · **Tag:** `hhw-20261117`

* `docs/serializer.md`’s example pushes the root, `pop_pack`s it, then calls `end_serialize`, which returns early because nothing is current (`hb-serialize.hh:310`), so its links are never resolved.
* `docs/repacker.md` says plain Kahn sorting is tried first (there is only the shortest-distance sort, itself Kahn’s algorithm with a priority queue); that only GPOS subtables can be split (GSUB LigatureSubst can be, `graph/gsubgpos-graph.hh:383`); and that spaces add n × 2^32 to a link’s weight (the code multiplies it by space + 1, `graph/graph.hh:1911`).

**On main.** Fixed on 17 November 2026 by `4d41c69d4` (#6702), which corrects both documents.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 50. A reordering custom glyph map breaks PairPos and ClassDef in the subset

**ID:** `HHW-50` · **Tag:** `hhw-20261117`

**Summary.** The API allows any old→new glyph map, but several layout subsetters write records in old-glyph order and assume the new IDs keep it (`hb_map_retains_sorting`): PairPos format 1’s PairSet (`src/OT/Layout/GPOS/PairSet.hh:255`) comes out unsorted by second glyph, so its binary search fails; `ClassDef_remap_and_serialize` (`hb-ot-layout-common.hh:1500`) pushes unsorted pairs into an `hb_sorted_vector_t`, and the ClassDef loses glyphs. SingleSubst survives in HarfBuzz but writes Coverage indices out of order (fontTools warns “Coverage is not sorted by glyph ids”).

**Repro.** `kern.ttf` in the book’s test fonts (PairPos format 1, A against eight glyphs), subset with a reversing `--gid-map` into `kern-map.ttf`. Each pair `A C`, `A G`, … `A Y` kerns −10 to −80 in the original; in the subset only `A Q` still kerns.

**Expected.** Records sorted by new glyph ID, so the subset kerns as the original does.

**On main.** Still present: `kern-map.ttf` cut by `main` still kerns only `A Q`.


**In 15.2.0 and on main (20 November 2026).** Documented on main only (after 15.2.0) by #6831, docs and help only; behavior unchanged.

---

## 51. Subsetting a symbol font drops its only `cmap` subtable

**ID:** `HHW-51` · **Tag:** `hhw-20261117`

**Summary.** `cmap` subsetting keeps only (0,3), (0,4), (3,1), (3,10) and variation-selector subtables (`src/hb-ot-cmap-table.hh:2502`). A font whose only subtable is (3,0), as symbol fonts have, comes out with no `cmap`, though its glyphs are kept. fontTools drops (3,0) by default too but has `--symbol-cmap`; HarfBuzz has no option. Related, from the code only: a (3,1) subtable in a format other than 4 or 12 is skipped silently.

**Repro.** `symbol.ttf` (a (3,0)-only `cmap`) in the book’s test fonts, subset to `symbol-a.ttf`: `hb-info --list-tables` shows no `cmap`.

**On main.** Still present: `symbol-a.ttf` cut by `main` still has no `cmap`.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6746.

---

## 52. `HB_SUBSET_FLAGS_NOTDEF_OUTLINE` is ignored for CFF

**ID:** `HHW-52` · **Tag:** `hhw-20261117`

**Summary.** Only the glyf and gvar subsetters read the flag (`Glyph.hh:265`), so a CFF font’s `.notdef` keeps its outline whether or not the flag is set. fontTools empties it in both.

**Repro.** `cff-o.otf` in the book’s test fonts, subset without `--notdef-outline`: `hb-shape --show-extents` shows `.notdef` with its original extents.

**On main.** Still present: `cff-o.otf` cut by `main` still has `.notdef` with its outline.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6748.

---

## 53. Small faults in the data generators

**ID:** `HHW-53` · **Tag:** `hhw-20261117`

None affects the generated tables.

* `src/gen-hb-version.py:20`: the regex `(\d.\d.\d)` allows one digit per part, so since 10.0.0 the “already current” early exit never fires and the header is always rewritten.
* `src/gen-ragel-artifacts.py`: checks `len(sys.argv) < 4` but reads `sys.argv[4]`, so a missing argument gives an `IndexError` instead of the usage message.
* `src/update-unicode-tables.make:16`: `SOURCES` omits `DerivedCoreProperties.txt` and `emoji-test.txt`, so `clean-sources` leaves stale copies; possibly why the USE table header records DerivedCoreProperties 17.0.0 beside 18.0.0 for the rest (inferred).
* `src/gen-arabic-table.py`: reads `Blocks.txt` into a dictionary nothing uses; U+F2EE appears twice in `LIGATURES`.
* `src/gen-arabic-joining-list.py`: its usage string names `gen-arabic-joining-table.py`.
* `src/gen-emoji-table.py`: its header comment names only `emoji-data.txt`, not `emoji-test.txt`.
* `src/gen-indic-table.py`: the `Consonant_Killer: 'M'` entry, commented “U+17CD only”, decides nothing, since U+17CD is overridden to `Xgroup`.
* `gen-tag-table.py` and `gen-os2-unicode-ranges.py` cite `docs.microsoft.com` URLs, and the ISO 639 table cites `www-01.sil.org`.

**On main.** Partly fixed, on 17 November 2026: `ebfcf19c7` (#6688) fixes the version regex, `c66b1a505` (#6689) the Ragel argument check, and `b873113f4` (#6715) the `SOURCES` list (the USE table hasn’t been regenerated since, and still records DerivedCoreProperties 17.0.0). The rest are in progress, in two open pull requests: #6739 (branch `codex/upstream-report-53-arabic-cleanup`: the unused `Blocks.txt` dictionary and the duplicate U+F2EE) and #6740 (branch `codex/upstream-report-53-generator-docs`: the usage string, the emoji header comment, the `Consonant_Killer` comment and the Microsoft URLs). Neither changes the `www-01.sil.org` citation.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6739 and #6740, except the www-01.sil.org citation, which upstream keeps.

---

## 54. Texts of 524,289 to 1,048,575 characters go almost unshaped: the operation budget starts negative

**ID:** `HHW-54` · **Tag:** `hhw-20261117`

**Summary.** `hb_buffer_t::enter` computes the operation budget as `int limit = (int) hb_max (mul, (unsigned) HB_BUFFER_MAX_OPS_MIN)` with `mul = len × HB_BUFFER_MAX_OPS_FACTOR` (4096) unsigned (`src/hb-buffer.cc:345`). For lengths from 524,289 to 1,048,575 the cast gives a negative `max_ops`, so the budget is spent before shaping starts and most lookups don’t run. At exactly 524,288 it is `INT_MIN`, and the next decrement is signed overflow, undefined behavior. From 1,048,576 the unsigned multiply overflows, the `if` is skipped, and the default budget applies again. Nothing reports it: `hb-shape` doesn’t check `hb_buffer_allocation_successful`.

**Repro.** `ligatures.ttf` in the book’s test fonts, with `fi` repeated:

```
 524,286 characters: 262,143 f_i ligatures
 524,290 characters: 1 ligature, then plain f and i
1,000,000 characters: 1 ligature
1,048,576 characters: 524,288 ligatures
```

**Expected.** The budget clamped to `INT_MAX` (or computed in a wider type), so long texts shape fully.

**On main.** Changed, still present. On `main` the budget still starts negative, and texts from 524,300 to 1,048,575 characters get no ligature at all, not even the first. The report’s 524,290 characters now shape fully, probably because the first charge takes the budget past `INT_MIN` and it wraps around to a large positive value: the undefined behavior the summary mentions. Bisecting 15.1.0..main puts the change at `339aa68`, “Charge each GSUB/GPOS lookup pass to max_ops” (11 November 2026).


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6664 (`HB_BUFFER_MAX_OPS_FACTOR` 65536, `HB_BUFFER_MAX_OPS_MIN` 524288, skipped glyphs charged 1/32 unit); #6664 doesn’t cite the book.

---

## 55. `HB_LEAN` removes drawing and painting in every Meson build

**ID:** `HHW-55` · **Tag:** `hhw-20261117`

**Summary.** `hb-config.hh:85` keeps drawing and painting under `HB_LEAN` only if `HB_HAS_CAIRO`, `HB_HAS_RASTER`, `HB_HAS_VECTOR` or `HB_HAS_GPU` is defined. Those come only from `hb-features.h`, which no library source includes (only `harfbuzz-world.cc` does, when `HB_FEATURES_H` is set); Meson’s `config.h` has `HAVE_RASTER` and so on instead. Checked with the preprocessor against a build with raster and vector on: `-DHB_LEAN` still defines `HB_NO_DRAW`.

**On main.** Fixed on 17 November 2026 by `c90000dd8` (#6724), which makes `hb-config.hh` accept the `HAVE_*` renderer macros too. With the same preprocessor check against the `main` build, `-DHB_LEAN` now leaves drawing on.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 56. `hb-features.h` never defines `HB_HAS_GPU` or `HB_HAS_GRAPHITE`

**ID:** `HHW-56` · **Tag:** `hhw-20261117`

**Summary.** `src/meson.build:587` maps `HAVE_GPU` and `HAVE_GRAPHITE` to `HB_HAS_*`, but Meson sets `HAVE_GRAPHITE2`, and the `gpu` option sets no `HAVE_` macro. The book’s build has `#undef HB_HAS_GPU` in `hb-features.h` though `libharfbuzz-gpu` was built. (The graphite half is from reading the Meson files.)

**On main.** Fixed on 17 November 2026 by `2b873b805` (#6721) and `506cc943b` (#6722). The `main` build’s `hb-features.h` now has `#define HB_HAS_GPU 1`, and `meson.build:253` sets `HAVE_GRAPHITE`, the name the list looks up, so a build with graphite2 defines `HB_HAS_GRAPHITE` (read from the Meson files; the book’s build has no graphite2). The chapter on integrations found the graphite half independently on 15.1.0: a graphite2-enabled build lists `graphite2` in `hb-shape --list-shapers` but has `#undef HB_HAS_GRAPHITE`. It is the same bug, so it is recorded here rather than as a new report.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 57. Unused and undocumented configuration and limits

**ID:** `HHW-57` · **Tag:** `hhw-20261117`

* `HB_MINIMIZE_MEMORY_USAGE_VAL` (`hb-config.hh:205`), `HB_SVG_MAX_PATH_SEGMENTS` and `HB_SANITIZE_MAX_EDITS` are defined but never used.
* `CONFIG.md` describes `HB_MINI` as dropping AAT and legacy shaping, but it also sets `HB_NO_BEYOND_64K` and `HB_NO_CUBIC_GLYF`; it doesn’t mention `HB_CONFIG_OVERRIDE_LAST_H`, nor about 70 other `HB_NO_*` macros.
* A composite component dropped by the subsetter’s nesting or operation limit keeps its old glyph ID when written (`SubsetGlyph.hh:55`), so it may point at the wrong glyph in the subset (inferred).

**On main.** Partly fixed. `91722141b` (#6728, 17 November 2026) removes the three unused definitions, and `9e6b1a989` (#6727) removes `HB_NO_CUBIC_GLYF` and adds `HB_MINI`’s exclusions and `HB_CONFIG_OVERRIDE_LAST_H` to `CONFIG.md`. Still present: `CONFIG.md` still names only seven `HB_NO_*` macros, and the composite component in `SubsetGlyph.hh:55` is unchanged.


**In 15.2.0 and on main (20 November 2026).** Partly fixed: one more item, the composite-component item, by #6759 in 15.2.0; `CONFIG.md`’s `HB_NO_*` list remains.

**New in 15.2.0.** `hb-limits.hh:258`–`:268` defines fixed budget weights from `HB_BUDGET_1` to `HB_BUDGET_1024`; four of them, `HB_BUDGET_64`, `HB_BUDGET_128`, `HB_BUDGET_512`, and `HB_BUDGET_1024`, are used nowhere in `src/` or `util/`. The same kind of slip as the unused macros above.

---

## 58. Cross-stream kerning breaks the unsafe-to-break promise

**ID:** `HHW-58` · **Tag:** `hhw-20261117`

**Summary.** In legacy `kern` and `kerx`, a cross-stream value raises every later glyph, but only the pair is marked unsafe to break. A line broken later in the run shapes differently.

**Repro.** `cross.ttf` in the book’s `legacy-kern` test fonts (and `aat-positioning/cross.ttf`, `cross-first.ttf` with `גבגב`):

```
$ hb-shape cross.ttf AVAT --verify
harfbuzz buffer verify error: unsafe-to-break test failed.
```

**On main.** Still present: `--verify` still fails.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6747.

---

## 59. Hangul at cluster level 1 breaks the unsafe-to-break promise

**ID:** `HHW-59` · **Tag:** `hhw-20261117`

**Summary.** `hb-ot-shaper-hangul.cc:385` calls `merge_out_grapheme_clusters`, which at levels other than 0 neither merges nor flags, so a trailing jamo keeps its own cluster, unflagged, though it shapes with the syllable before it.

**Repro.** `hangul-koh.ttf` in the book’s test fonts: `hb-shape hangul-koh.ttf $'\uAC00\u11EB' --cluster-level=1 --verify` fails the unsafe-to-break test.

**On main.** Fixed on 17 November 2026 by `8e6c16eef` (#6709). The repro now passes `--verify`: `[kiyeok=0+500|a=0+500|pansiosT=1+500]`.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 60. `hb-shape --text-before` echoes its value to stderr

**ID:** `HHW-60` · **Tag:** `hhw-20261117`

**Summary.** A leftover debugging line, `fprintf(stderr, "%s\n", opts->text_before);`, at `util/text-options.hh:224`.

**On main.** Fixed on 17 November 2026 by `55ecd750d` (#6687).


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 61. Small instancing differences from fontTools

**ID:** `HHW-61` · **Tag:** `hhw-20261117`

* After limiting an axis, `SegmentMaps::subset` maps the new range back through `avar` instead of using the quantized limit (its comment: “unmap_axis_range needs to be killed”), and the `avar` map can be one 2.14 unit off. In `var-avar.ttf` limited to wght 400–800, `I` at wght 570 has advance 370 where the original has 369 (fontTools matches the original).
* Pinning an axis that VARC uses is refused by design (`VARC-subset.hh:101`), but `hb-subset` reports “Operation failed. Probably a bug. File github issue.”
* With `avar` 2 and weight pinned, one unreachable region stays in `MVAR`.

**On main.** Still present: `var-avar.ttf` limited to wght 400–800 by `main` still gives `I` at 570 an advance of 370. The other two items weren’t rerun.


**In 15.2.0 and on main (20 November 2026).** Partly fixed in 15.2.0: avar by #6770, MVAR by #6772; the VARC message item declined (#6766 closed).

---

## 62. `hb_style_get_value (opsz)` returns OS/2 optical sizes in twentieths of a point

**ID:** `HHW-62` · **Tag:** `hhw-20261117`

**Summary.** OS/2 version 5’s `usLowerOpticalPointSize` and `usUpperOpticalPointSize` are in TWIPs (1/20 pt). `OS2V5Tail::get_optical_size` (`src/hb-ot-os2-table.hh:86`) returns them unconverted, and `hb_style_get_value` (`hb-style.cc:101`) averages them as points.

**Repro.** `style-size.ttf` in the book’s test fonts, with an optical range of 8–24 pt (160–480 TWIPs): `hb_style_get_value (font, HB_STYLE_TAG_OPTICAL_SIZE)` returns 320, not 16.

**On main.** Fixed on 17 November 2026 by `b14a94e52` (#6695). `hb-info --get-style=opsz` on the book’s font with the 8–24 pt range (`style.ttf`) now gives 16.


**In 15.2.0 and on main (20 November 2026).** The fix is in 15.2.0.

---

## 63. BASE min/max: missing values read as 0

**ID:** `HHW-63` · **Tag:** `hhw-20261117`

**Summary.** For a script with language-system MinMax records but no `DefaultMinMax`, `get_min_max` returns true with ascender and descender 0 for every other language (`src/hb-ot-layout-base-table.hh:482`). A null Min or Max coord is also read as 0 rather than reported missing. And a script record with MinMax but no BaseValues hides `DFLT`’s baselines from that script.

**Repro.** `base.ttf` in the book’s test fonts, queried with the chapter’s `query.c`.

**On main.** Still present: `hb-ot-layout-base-table.hh` is unchanged. Checked in the code.


**In 15.2.0 and on main (20 November 2026).** Partly fixed in 15.2.0 by #6750: MinMax missing values; “MinMax without BaseValues hides `DFLT`” remains.

---

## 64. `STAT::get_value` takes the first axis value, and STAT suppresses synthetic slant

**ID:** `HHW-64` · **Tag:** `hhw-20261117`

**Summary.** `STAT::get_value` returns the first axis value record for the axis, which need not be the font’s own (weight 300 in a bold font); format 4 records are never matched. Because STAT (or `fvar`) is consulted before `post`, a STAT `slnt` value also stops `hb_style_get_value` adding synthetic slant (`hb-style.cc:114` is reached only in the `post` fallback).

**Repro.** `style-stat.ttf` in the book’s test fonts.

**On main.** The synthetic slant is in progress: pull request #6743 (branch `codex/upstream-report-64-synthetic-slant`, “Apply synthetic slant to fvar and STAT values”), not merged. The first-axis-value half is still present: `style-stat.ttf` still reports weight 300.


**In 15.2.0 and on main (20 November 2026).** Partly fixed in 15.2.0: synthetic slant by #6743; #6755 adds sibling flag and format 4; style-stat.ttf still weight 300, deliberately not guessed.

---

## 65. Smaller things in `name`, BASE and MATH

**ID:** `HHW-65` · **Tag:** `hhw-20261117`

* `hb-info --list-names` prints each entry in the command-line language, not the entry’s own, so the same string repeats.
* `minConnectorOverlap` (uint16) goes through `em_scale_dir (int16_t)` (`hb-font.hh:169`), so values ≥ 32768 come back negative (read from the code).
* For Unicode-platform name records, `name` calls `ltag::get_language`, which interns strings with `hb_language_from_string`: the quadratic risk `hb-aat-ltag-table.hh:77` says `morx` avoids (read from the code).
* `choose_base_tags` keeps the last script tag (`deva`), so a BASE keyed `dev2` is missed and `DFLT` used.

**On main.** Partly fixed. The `minConnectorOverlap` sign is fixed on 17 November 2026 by `eaf004a2c` (#6708). `hb-info --list-names` is in progress: pull request #6741 (branch `codex/upstream-report-65-name-list`), not merged. The other two are still present: `name` still calls `ltag::get_language` (`OT/name/name.hh:111` on `main`; the 9 November change to `ltag`, `488e98dd8`, avoids interning only for `morx`), and the BASE tags are still the last script tag (the function is now `choose_layout_tags`).

---

## Found while writing the chapters on building, the tools and the renderers, November 2026

These were found after the batch tagged `hhw-20261117`: 66–78 while writing the chapters on hb-raster, hb-vector, hb-gpu, building HarfBuzz and the command-line tools, 79–81 while writing the chapter on integrations, and 82 while revising the chapter on the Universal Shaping Engine. Each was reproduced on HarfBuzz 15.1.0 (commit `3df40d9b`), or is marked as read from the code, and checked against `main` at commit `454f9c49c` (17 November 2026), built with the same options. On 18 November `main` was still at that commit, and no open pull request addresses any of them. File and line references are to 15.1.0. They are tagged `hhw-20261118`.


**In 15.2.0 and on main (20 November 2026).** Partly fixed in 15.2.0, 3 of 4: list-names by #6741, BASE dev2 by #6760; ltag interning remains.

---

## 66. `hb_color_lerp` reverses a color’s bytes, so every PDF sweep gradient has the wrong colors

**ID:** `HHW-66` · **Tag:** `hhw-20261118`

**Summary.** `hb_color_lerp` (`src/hb-paint.hh:292`) interpolates each byte of two `hb_color_t`s and returns `HB_COLOR (lerp (0), lerp (8), lerp (16), lerp (24))`. But `HB_COLOR` takes its arguments as blue, green, red, alpha (`hb-common.h:497`), and `lerp (0)` is alpha, so the result has its four bytes in reverse order: red becomes green, green becomes red, alpha becomes blue and blue becomes alpha.

The PDF sweep writer calls it for both ends of every Coons patch, even when `t` is 0 (`hb-vector-paint-pdf.cc:1348`), so every PDF sweep is painted in the wrong colors. `hb_paint_sweep_gradient_tiles` (`hb-paint.cc:1113`, `:1181`, `:1200`, `:1285`, `:1291`) calls it where a sweep crosses 0° or 360°, so SVG sweeps that do are wrong at those tiles too, and so are cairo’s, which use the same tiles (`hb-cairo-utils.cc:581`; cairo is read from the code, not run).

**Repro.** `colr1.ttf` in the book’s test fonts (`fonts/color/`), whose `S` is a sweep through red, amber, green and blue, all opaque:

```
$ hb-vector colr1.ttf S -O pdf -o s.pdf && pdftoppm -png s.pdf s
```

poppler and Ghostscript both draw it in cyan, pale blue and purple. The first patch’s colors in the mesh stream are `26 D7 FF`, where the first stop is red, `D7 26 3D`. `hb-raster colr1.ttf S` draws the right colors.

For SVG, a font with a pad sweep from −90° to 270° between red and blue: `hb-vector` writes a fan whose colors go from `#42A8FE` to blue, with no red at all, where at 0° the color should be three quarters red, about `#A84266`.

**Expected.** `HB_COLOR (lerp (24), lerp (16), lerp (8), lerp (0))`.

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6814.

---

## 67. hb-vector’s PDF sweeps leave a square hole at their center

**ID:** `HHW-67` · **Tag:** `hhw-20261118`

**Summary.** A PDF sweep is a mesh of Coons patches, each running from 0.5 to 32,767 units from the center (`hb-vector-paint-pdf.cc:1335–1336`). Its points are written as 16-bit numbers over a `Decode` range 65,536 units wide (`:1291`, `:1513`), so they land on a grid of one unit. In `hb-vector`’s output, the mesh is painted under a 64× scale (`64 0 0 64 cm` before `sh`), so a grid step is 64 font units. The inner ends of the patches, half a unit from the center, are rounded onto the corners of one grid square, and that square is never painted.

**Repro.** `colr1.ttf` in the book’s test fonts:

```
$ hb-vector colr1.ttf S -O pdf -o s.pdf && pdftoppm -png s.pdf s
```

poppler and Ghostscript both leave a white square 64 units on a side beside the center (500, 400). Decoded, the four inner corners of the patches are (8.3, 6.8), (7.3, 6.8), (7.3, 5.7) and (8.3, 5.7) in the mesh’s units.

**Expected.** No hole: a `Decode` range sized to the glyph, or the inner ends brought to the center, or more bits per coordinate.

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6824.

---

## 68. hb-vector’s PDF output ignores custom palette colors

**ID:** `HHW-68` · **Tag:** `hhw-20261118`

**Summary.** The PDF paint funcs don’t set `custom_palette_color` (`hb-vector-paint-pdf.cc:1650`), so `hb_vector_paint_set_custom_palette_color` has no effect on PDF. The SVG funcs do (`hb-vector-paint-svg.cc:292`).

**Repro.** `colr0.ttf` in the book’s test fonts, whose `A` has a disc in palette entry 1:

```
$ hb-vector colr0.ttf A --custom-palette=1=#2E8B57 -o a.svg
$ hb-vector colr0.ttf A --custom-palette=1=#2E8B57 -O pdf -o a.pdf
```

The SVG fills the disc with `#2E8B57`; the PDF with `0.1647 0.4353 0.8588 rg`, the palette’s own blue.

**Expected.** The same colors in both.

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed in 15.2.0 by #6804 (doesn’t cite the book).

---

## 69. hb-vector’s PDF compositing isn’t isolated

**ID:** `HHW-69` · **Tag:** `hhw-20261118`

**Summary.** `hb_pdf_paint_push_group_for` (`hb-vector-paint-pdf.cc:1607`) writes `q` and an ExtGState with the blend mode, and the group’s content follows in the page’s own content stream. There is no transparency group, so the source is blended with everything already on the page, the white background included, not only with the backdrop the COLR graph composites it onto.

**Repro.** `modes.ttf` in the book’s test fonts (`fonts/hb-vector/`): each letter is a blue disc and a red disc joined by one PaintComposite.

```
$ hb-vector modes.ttf S -O pdf -o s.pdf && pdftoppm s.pdf s
$ hb-vector modes.ttf D -O pdf -o d.pdf && pdftoppm d.pdf d
```

In `S` (SCREEN), the part of the red disc outside the blue one disappears: red screened on white is white. In `D` (DIFFERENCE), it turns cyan (`#28D9C2`). poppler and Ghostscript agree. `hb-raster` draws both parts red, as COLR requires: outside the backdrop, the source is drawn as it is.

**Expected.** Each `push_group` written as a transparency group (a form XObject with `/Group << /S /Transparency /I true >>`), painted with the blend mode on `pop_group`.

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6825.

---

## 70. hb-gpu drops a fill whose only clip is a rectangle

**ID:** `HHW-70` · **Tag:** `hhw-20261118`

**Summary.** The paint encoder ignores rectangle clips, and its comment says the result is merely bigger: “the encoder simply ignores the rectangle (output is bigger than it should be, but renders)” (`hb-gpu-paint.cc:1074`). But a layer is emitted only inside a glyph or path clip (`if (unlikely (c->clip_depth == 0)) return;`, `:673`, and the same in the gradient emitters), so a fill clipped only by a ClipBox or PaintClipBox is dropped, and a glyph made of one encodes as the empty blob.

**Repro.** `colr1.ttf` in the book’s test fonts, whose `C` is a PaintSolid inside its ClipBox. With `fonts/hb-gpu/dump-gpu-blob.c` (a few lines around `hb_gpu_paint_glyph` and `hb_gpu_paint_encode`):

```
$ dump-gpu-blob colr1.ttf C paint
C: 0 bytes, 0 texels
$ dump-gpu-blob colr1.ttf L paint
L: 488 bytes, 61 texels
```

`hb-raster colr1.ttf C` paints a green square.

**Expected.** The rectangle encoded as a clip (or, as the comment says, ignored, with the fill painted unclipped).

**On main.** Still present (`:665`, `:1066`).


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6823 (hb-gpu paint-blob header changed by #6832 too).

---

## 71. hb-gpu’s stem darkening is documented with pixels per unit for its ppem (inferred, from reading the code)

**ID:** `HHW-71` · **Tag:** `hhw-20261118`

**Summary.** `hb_gpu_stem_darken (coverage, brightness, ppem)` fades out between 8 and 48 pixels per em (`hb-gpu-fragment.glsl:344`). Its documentation computes `ppem` as `1.0 / max (fwidth (v_texcoord).x, fwidth (v_texcoord).y)` (`hb-gpu.cc:349`, and the examples at `:357`, `:374` and `:511`), and so does the `hb-gpu` viewer (`util/gpu/demo-fragment.glsl:35`). The documentation sets texture coordinates up in font units, so that is pixels per font unit, smaller than pixels per em by the font’s scale: for a 1,000-unit font the darkening never fades out. `hb_gpu_ppem (renderCoord, glyphLoc)` (`:167`) divides by the scale stored in the blob and gives the right figure.

Not run: the book’s build has no GPU.

**Expected.** The documentation and the viewer calling `hb_gpu_ppem`.

**On main.** Still present (`hb-gpu.cc:350`).


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6832.

---

## 72. `HB_FEATURES_H` is both `harfbuzz-world.cc`’s header name and `hb-features.h`’s include guard

**ID:** `HHW-72` · **Tag:** `hhw-20261118`

**Summary.** `harfbuzz-world.cc` starts with `#ifdef HB_FEATURES_H` / `#include HB_FEATURES_H` (`src/harfbuzz-world.cc.in:1–3`), so that a project can name its own features header. `hb-features.h` uses the same macro as its include guard (`hb-features.h.in:25`). Naming the `hb-features.h` that Meson writes, or a copy of it, therefore includes a file whose whole body is skipped, and none of its `HB_HAS_*` macros is defined.

**Repro.** With the preprocessor, against a Meson build with raster on (its `hb-features.h` has `#define HB_HAS_RASTER 1`):

```
$ printf '#include HB_FEATURES_H\n#ifdef HB_HAS_RASTER\nON\n#else\nOFF\n#endif\n' > t.cc
$ c++ -E -P t.cc -I src -I build/src -DHB_FEATURES_H='"hb-features.h"' | tail -1
OFF
```

**Expected.** Different names for the two, for example `HB_FEATURES_H_IN` for the guard, so that Meson’s own header can be passed to `harfbuzz-world.cc`.

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6815.

---

## 73. `hb-shape -v` turns on `--ned` as well as `--verbose`

**ID:** `HHW-73` · **Tag:** `hhw-20261118`

**Summary.** `--verbose` and `--ned` both have the short name `v` (`util/shape-format.hh:187`, `:199`). GLib runs both callbacks for `-v`, so the clusters and advances disappear. The help lists `-v` only for `--verbose`.

**Repro.**

```
$ hb-shape Roboto-Regular.ttf ab --verbose
1: (ab)
1: <U+0061=0|U+0062=1>
1: [gid69=0+1114|gid70=1+1149]
$ hb-shape Roboto-Regular.ttf ab -v
1: (ab)
1: <U+0061=0|U+0062=1>
1: [gid69|gid70@1114,0]
```

**Expected.** `--ned` with no short name (`0`), or a different letter.

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6813.

---

## 74. `--features` and `--variations` split at spaces, so the CSS forms their help offers don’t work

**ID:** `HHW-74` · **Tag:** `hhw-20261118`

**Summary.** The help for `--features` says “All valid CSS font-feature-settings values other than 'normal'” are accepted, and `--variations` says the same of `font-variation-settings`. `hb_feature_from_string` and `hb_variation_from_string` do accept `"liga" 0` and `wght 900`. But the tools split the argument at spaces as well as commas before parsing (`strpbrk (p, ", ")`, `util/shape-options.hh:339`, `:352`; `util/font-options.hh:179`, `:192`), so `"liga" 0` becomes `"liga"`, which turns the feature on, and `0`, which doesn’t parse and is dropped silently.

**Repro.** With Roboto and `var.ttf` from the book’s test fonts (`fonts/variations/`):

```
$ hb-shape Roboto-Regular.ttf a
[gid69=0+1114]
$ hb-shape Roboto-Regular.ttf a --features='"smcp" 0'
[gid563=0+1157]
$ hb-shape var.ttf I --variations='wght=900'
[I=0+410]
$ hb-shape var.ttf I --variations='wght 900'
[I=0+260]
```

Asking for small caps off turns them on, and the weight is ignored with no message.

**Expected.** The CSS forms parsed as the help says (splitting at commas only, as CSS does), or the help changed and a value that doesn’t parse reported.

**On main.** Still present (`font-options.hh:181`, `:194`).


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6822.

---

## 75. `hb-vector --show-extents` and `hb-raster --show-extents` draw each box after the first at twice its pen position

**ID:** `HHW-75` · **Tag:** `hhw-20261118`

**Summary.** `hb-vector` sets the glyph’s transform to the pen position (`util/vector-output.hh:265`, `:288`), then draws the extents overlay through it with the pen position added again (`:273`, `:296`; `util/helper-extents-overlay.hh:66`). The first glyph’s pen position is 0, so only the boxes after it are misplaced. `hb-raster` does the same in all three of its paths: outlines (`util/raster-output.hh:251`, `:257`), color glyphs (`:330`, `:341`) and per-glyph foreground colors (`:426`, `:430`). (The first draft of this report said `hb-raster` was correct; a picture in the book showed otherwise.)

**Repro.** `var.ttf` in the book’s test fonts, whose `I` has an advance of 260 and a stem from 80 to 180:

```
$ hb-vector var.ttf III --show-extents
```

The second `I` is drawn at x 340–440, but its box at 599–701 and its origin dot at 520; the third `I` at 600–700, its box at 1119–1221. The boxes also fall outside the picture’s `viewBox`.

```
$ hb-raster test/subset/data/fonts/Roboto-Regular.ttf Wave --font-size=32 --show-extents -o wave.png
```

`hb-shape --show-extents` puts the *a*’s ink at x 28–43 (pen 27, bearing 1, width 15), but its box is drawn at about 55–70, around the *v* and *e*; the boxes of *v* and *e* fall outside the image, which is sized to the glyphs.

**Expected.** The overlay drawn at the glyph’s origin within the transform, or without the transform.

**On main.** Still present (no change to these files on main since 17 November, checked 18 November 2026).


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6818.

---

## 76. Smaller things in hb-raster, hb-vector and hb-gpu

**ID:** `HHW-76` · **Tag:** `hhw-20261118`

* hb-raster computes luminosity for the HSL composite modes with the weights 0.299, 0.587 and 0.114 (`hb-raster-image.cc:297`). COLR takes its modes from the W3C Compositing and Blending specification, whose `Lum` uses 0.3, 0.59 and 0.11. Red over blue in HSL_LUMINOSITY comes out as `(0, 122, 192)` where the specification gives `(0, 123, 193)`.
* The documentation of `hb_vector_paint_set_custom_palette_color` (`hb-vector-paint.cc:453`) says the overrides are consulted by “SVG glyph content using `var(--colorN)`”; nothing in HarfBuzz reads `var(--colorN)` (read from the code).
* The documentation of `hb_gpu_draw_recycle_blob` (`hb-gpu-draw.cc:1160`) says that it “simply destroys the blob”; it keeps it and the next encode reuses its memory (`hb-blob.hh:220`).
* The documentation of `hb_gpu_draw_encode` (`hb-gpu-draw.cc:436`) says the context is cleared “On success”; a scope guard clears it on every return (`:457`).
* The design notes in `hb-gpu-paint.hh:37–238` describe an earlier blob layout: one-texel operations, 16.16 transforms, palette indices in color stops, and a sentinel `custom_palette_color` callback. The encoder writes three-texel layer operations and resolved colors (read from the code).
* The paint shader’s comment (`hb-gpu-paint-fragment.glsl:28`) and the documentation of `hb_gpu_paint_shader_source` (`hb-gpu-paint.cc:1750`) say the draw shader source must be prepended, because the interpreter calls `hb_gpu_draw ()`. It doesn’t: it calls `_hb_gpu_slug` from the shared helpers, and the draw source defines only `hb_gpu_draw` (read from the code).

**On main.** All still present. (Three stale comments in report 37, in `hb-vector-paint-pdf.cc:49`, `hb-limits.hh` and `README.md`, are fixed on main by `24fe9f0e2`.)


**In 15.2.0 and on main (20 November 2026).** Partly fixed in 15.2.0 by #6806; fixed on main by #6816, #6817, and #6826.

---

## 77. Smaller things in the build files and build documentation

**ID:** `HHW-77` · **Tag:** `hhw-20261118`

* `harfbuzz-world.cc` maps `HB_HAS_GRAPHITE` to `HAVE_GRAPHITE` (`src/harfbuzz-world.cc.in:20–22`), but the Graphite code tests `HAVE_GRAPHITE2` (`hb-graphite2.cc:31`, `hb-shaper-list.hh:41`), so the single-file build can’t include the Graphite backend. Preprocessing `harfbuzz-world.cc` with a features header that defines `HB_HAS_GRAPHITE` gives no Graphite code. **Fixed on main** by `506cc943b`, which renames `HAVE_GRAPHITE2` to `HAVE_GRAPHITE` throughout (the follow-up to report 56).
* The `libs` alias target (`meson.build:449–459`) lists the core, subset, raster and vector libraries but not `libharfbuzz-gpu`: `ninja -C build -t query libs` doesn’t list it, though it is built by default. Still present on main (`:463`).
* `BUILD.md:38` and `:42` and `TESTING.md:4` and `:36` use `meson build`, without `setup`; Meson 1.12 warns that this form is “ambiguous and deprecated”. Still present on main.
* The single-file `hb-shape` build in `BUILD.md:57` warns `'HAVE_ISATTY' is not defined, evaluates to 0 [-Wundef]` at `util/output-options.hh:102`, because only Meson’s `config.h` defines it. Still present on main.


**In 15.2.0 and on main (20 November 2026).** Partly fixed in 15.2.0 (earlier items); fixed on main by #6820, #6821, and #6819.

---

## 78. Smaller things in the command-line tools

**ID:** `HHW-78` · **Tag:** `hhw-20261118`

* `--font-size` is described as “1/2 integers or 'upem'” (`util/font-options.hh:319`), but is parsed with `%lf` (`:214`) and used as a fraction. Its error message says “one or two space-separated numbers” (`:219`), but a comma separates them too: `--font-size=500,250` works.
* `output_options_t::refuse_tty` (`util/output-options.hh:124`) is tested (`:103`) but never set, so no tool refuses to write binary output to a terminal through it (read from the code).
* `hb-fc-list`’s usage text without long options says `-V` displays the “HarfBuzz version” (`util/hb-fc-list.c:93`), but it prints fontconfig’s (`:131`); the long-options text (`:87`) is right. No Meson file builds `hb-fc-list.c` or `hb-fc.cc` (read from the code).

**On main.** All still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6819.

---

## 79. `hb_directwrite_font_set_funcs` leaks a reference to the DirectWrite font face

**ID:** `HHW-79` · **Tag:** `hhw-20261118`

*Found by reading the code; Windows only, not run.*

**Summary.** `hb_directwrite_font_set_funcs` (`src/hb-directwrite-font.cc:392`) takes the `IDWriteFontFace` from the font’s DirectWrite data, calls `dw_face->AddRef ()` (`:403`), and installs the DirectWrite font funcs with `nullptr` for both the font data and its destroy callback (`:404`–`:406`). Nothing releases that reference. The funcs don’t use it: each callback reads the face from `font->data.directwrite` again (`:67`, `:105`, …), which holds its own reference, taken by `hb_directwrite_font_create` (`hb-directwrite.cc:433`) or made by `_hb_directwrite_shaper_font_data_create` (`hb-directwrite-shape.cc:66`), and released by `_hb_directwrite_shaper_font_data_destroy` (`hb-directwrite-shape.cc:113`) when the font is destroyed. So each call leaves the font face with one reference too many, and it is never freed.

`hb_coretext_font_set_funcs` (`hb-coretext-font.cc:525`), its Core Text counterpart, takes no extra reference.

**Expected.** No `AddRef`, as in the Core Text version; or the face passed as the font data with a destroy callback that releases it.

**On main.** Still present (`hb-directwrite-font.cc:403`).


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6812.

---

## 80. `_hb_gdi_reference_table` leaks the screen DC when `SelectObject` fails

**ID:** `HHW-80` · **Tag:** `hhw-20261118`

*Found by reading the code; Windows only, not run.*

**Summary.** `_hb_gdi_reference_table` (`src/hb-gdi.cc:42`), the table loader behind `hb_gdi_face_create`, gets the screen’s device context with `GetDC (nullptr)` and selects the font into it. If `SelectObject` fails, it returns at once (`:48`–`:49`), before the `HB_SCOPE_GUARD (ReleaseDC (nullptr, hdc))` on the next line (`:50`), so the DC is never released. The loader runs once for each table HarfBuzz asks for, so a font whose `HFONT` can’t be selected leaks a DC each time.

**Expected.** The guard set up before the `SelectObject` test, as it is for the later early return when `GetFontData` fails (`:53`).

**On main.** Still present.


**In 15.2.0 and on main (20 November 2026).** Fixed on main only by #6811.

---

## 81. Smaller things in the integration documentation

**ID:** `HHW-81` · **Tag:** `hhw-20261118`

*Found by reading the code and the documentation.*

* `hb_ft_font_set_funcs`’s documentation (`src/hb-ft.cc:1908`–`:1910`) still says that after changing the font “you need to call hb_ft_hb_font_changed() to update the underlying FT_Face”. The documentation of that function (`:1566`) says that since 11.0.0 calling it isn’t necessary, as HarfBuzz notices the change itself.
* `hb_cairo_glyphs_from_buffer`’s documentation has the descriptions of its two scale factors swapped: “@x_scale_factor: scale factor to divide #hb_position_t Y values by” and “@y_scale_factor: scale factor to divide #hb_position_t X values by” (`src/hb-cairo.cc:1056`–`:1057`). The code divides x positions by `x_scale_factor` (`:1159`, `:1166`).
* `hb_icu_script_to_script` and `hb_icu_script_from_script` (`src/hb-icu.cc:68`, `:88`) and `hb_uniscribe_font_get_logfontw` and `hb_uniscribe_font_get_hfont` (`src/hb-uniscribe.cc:550`, `:567`) have no `Since:` line. `hb_uniscribe_font_get_hfont`’s return value is “the HFONT retrieved”.
* `NEWS` for 11.0.0 presents Core Text font functions and `hb_ft_face_create_from_file_or_fail` and `hb_coretext_face_create_from_file_or_fail` as new in that release (“`hb-coretext` has gained one”, and the list of new face loaders). All three came in 10.1.0, whose `NEWS` lists them, and whose version their doc comments give (`hb-coretext-font.cc:522`, `hb-ft.cc:1772`, `hb-coretext.cc:431`). The 11.0.0 “New API” list itself is right.
* `docs/usermanual-integration.xml`: the FreeType example calls `hb_ft_font_create(face)` (`:216`) with one argument, where the function takes a destroy callback as well (`hb-ft.h:109`), as the paragraph after the example says; it names `hb_ft_font_get_face()` (`:259`), deprecated since 10.4.0 for `hb_ft_font_get_ft_face`; it gives the autotools option `--with-icu=builtin` (`:609`), where the Meson build has `-Dicu_builtin=true` (`meson.options:61`); and it says the Python bindings “support Python 2 and Python 3” (`:630`).

**On main.** All still present (`hb-ft.cc:1918`, `hb-cairo.cc:936`).



**In 15.2.0 and on main (20 November 2026).** Partly fixed on main only by #6817 and #6821: everything except the NEWS 11.0.0 item.

---

## 82. USE: the generator’s list of generic bases is dead code

**ID:** `HHW-82` · **Tag:** `hhw-20261118`

**Summary.** `is_BASE_OTHER` (`src/gen-use-table.py:224`) makes U+2015 HORIZONTAL BAR, U+2022 BULLET and U+25FB to U+25FE (medium squares) generic bases (`GB`), as Microsoft’s USE specification does. They come out `O` in the generated table. The generator makes a record only for a code point that appears in one of the first four files it reads, `IndicSyllabicCategory.txt`, `IndicPositionalCategory.txt`, `ArabicShaping.txt` and the default ignorables of `DerivedCoreProperties.txt` (`:101`–`:108`, `if i >= 4: continue`), and these six are in none of them, so the predicates never see them.

**Repro.** Build `src/test-use-table.cc` against the source tree and print the categories:

```
$ ./test-use-table | grep -E '^U\+(00A0|2010|2015|2022|25FB) '
U+00A0 5
U+2010 5
U+2015 0
U+2022 0
U+25FB 0
```

5 is `GB`, 0 is `O`. U+00A0 and U+2010 are `GB` because Unicode gives them `Consonant_Placeholder`.

**Effect.** Small. An `O` starts a symbol cluster, which in HarfBuzz takes the same tail as a standard cluster, so most text shapes alike. What differs is what can come before the base: a standard cluster may start with a repha or a consonant with stacker, and a symbol cluster can’t.

**Expected.** Either the six code points are given records so that `is_BASE_OTHER` applies, or the list is removed and the difference from the specification noted.

**On main.** Still present (`test-use-table` built from `main` at `454f9c49c` gives the same categories).


**In 15.2.0 and on main (20 November 2026).** Still present in 15.2.0 and on main at `8846dbbf6`.

---

## 83. The user manual’s Configuration options leave out HarfBuzz’s own libraries and backends, and give Graphite’s option its deprecated name

**ID:** `HHW-83` · **Tag:** `hhw-20261120`

**Summary.** The user manual’s “Configuration options” section (`docs/usermanual-install-harfbuzz.xml:178`, published at <https://harfbuzz.github.io/building.html#configuration>) describes ten of the thirty-four options in `meson_options.txt`: `-Dcairo`, `-Dcoretext`, `-Ddirectwrite`, `-Ddocs`, `-Dfreetype`, `-Dgdi`, `-Dglib`, `-Dgobject`, `-Dgraphite`, and `-Dicu`, mostly the outside libraries HarfBuzz can use.

Of the twenty-four it leaves out, eight are for HarfBuzz’s own development and packaging, and are reasonably left out of a user manual: `tests`, `doc_tests`, `benchmark`, `gpu_demo`, `ragel_subproject`, `fuzzer_ldflags`, `cmakepackagedir`, and `with_libstdcxx`. The other sixteen matter to anyone building HarfBuzz:

* HarfBuzz’s own optional libraries and tools: `subset`, `raster`, `vector`, `gpu`, and `utilities`, all enabled by default, so a packager needs to know they can be turned off.
* The other backends: `harfrust`, `fontations`, `kbts`, `wasm`, and `graphite2`. Two of these change what `hb_shape` returns: `wasm` and `graphite2` come before `ot` in the default shaper list (`src/hb-shaper-list.hh:36`–`:47`), so a build with either shapes fonts with a `Wasm` or `Silf` table through that backend.
* Other dependencies and outputs: `png`, `zlib`, `chafa`, `introspection`, and `icu_builtin`.
* `experimental_api`.

And the Graphite entry gives the option as `-Dgraphite`, which `meson_options.txt` describes as “Deprecated use graphite2 option instead”.

**Repro.**

```
$ comm -23 <(grep -o "^option('[a-z0-9_]*'" meson_options.txt | sed "s/option('//;s/'//" | sort) \
           <(grep -o '<command>-D[a-z0-9_]*' docs/usermanual-install-harfbuzz.xml | sed 's/<command>-D//' | sort -u) | wc -l
24
```

**Expected.** The sixteen described, or the section pointing to `meson_options.txt` and `meson configure` for the rest; and `-Dgraphite2` in place of `-Dgraphite`.

**On main.** Still present (`main` at `94715f603`, 19 November 2026).

**In 15.2.0 and on main (20 November 2026).** Still present in 15.2.0 and on main at `8846dbbf6`.

---

## 84. The Core Text face loader fails for a font with named instances or several faces and no PostScript name

**ID:** `HHW-84` · **Tag:** `hhw-20261120`

**Summary.** Since #6683 (“[coretext] Resolve collection faces by PostScript name”, 16 November 2026, in 15.2.0), when Core Text returns more than one font descriptor for a file or blob, which it does for a collection and for a font with named instances, `create_cg_font` picks the descriptor whose name matches the face’s PostScript name, name ID 6 (`src/hb-coretext.cc:199`–`:240`). A font with no name ID 6 has nothing to match: `postscript_name` stays null and the function returns `nullptr` (`:224`). So `hb_coretext_face_create_from_file_or_fail` fails for such a font at every face index, and so does `create_cg_font (hb_blob_t *, unsigned)` for a nonzero index (`:244`), which the `coretext` shaper uses. In 15.1.0 the same calls worked.

Fonts that have a PostScript name, such as the system’s Skia and SFNS, work, so this mostly affects test fonts and fonts built without name ID 6.

**Repro.** With a build that has Core Text (`-Dcoretext=enabled`), on macOS, and `fonts/blobs-faces-and-lazy-tables/instances.ttf` from the book’s repository (named instances, and no name ID 6: `hb-info --list-names` shows IDs 1, 2, and 256–260):

```
$ hb-shape fonts/blobs-faces-and-lazy-tables/instances.ttf IHV --face-loader=coretext
$ hb-shape fonts/blobs-faces-and-lazy-tables/instances.ttf IHV --shapers=coretext --face-index=65536
```

Both fail in 15.2.0, and both shape the text in 15.1.0.

**Expected.** A fallback when the face has no PostScript name: for example, the face’s data opened directly, as for a single-face file, or the descriptor chosen another way.

**On main.** Still present (`main` at `8846dbbf6`, 20 November 2026). Found during the book’s move to 15.2.0: the repro was run with a Core Text build, and the code path read again for this report.

---

## 85. Vertical `kern` (and `kerx`) values are applied with the horizontal sign, so negative kerning moves glyphs apart

**ID:** `HHW-85` · **Tag:** `hhw-20261120`

**Summary.** Since 15.2.0 a vertical legacy `kern` subtable applies when `vkrn` is requested (#6734, the fix for HHW-18). The kern machine then adds the value to the vertical advances as it is, half to each glyph of the pair, and the second half to the second glyph’s `y_offset` too (`src/hb-kern.hh:141`–`:147`). Vertical advances are negative, growing downwards, so a negative value makes them longer and moves the glyphs apart. The OpenType `kern` specification says that a value “less than zero” means “the characters will be moved closer together”. GPOS handles this by negating vertical values. It is the mistake `trak` had in vertical text (HHW-39). `kerx` formats 0, 2, and 6 in vertical text run on the same code (read from the code, not run).

HarfBuzz’s own `test/shape/data/in-house/tests/kern-vertical.tests:3` expects these advances, so the behavior may be deliberate; but that test came with #6734, so it records what the fix did rather than a decision about the sign.

**Repro.** HarfBuzz’s own test font, whose vertical subtable has the pairs a b −100 and b c −200:

```
$ hb-shape test/shape/data/in-house/fonts/e671efb00d233d04699b796e0495015fa4e73335.ttf abcd --direction=ttb --features=vkrn
[a=0@-250,-850+0,-1050|b=1@-250,-900+0,-1150|c=2@-250,-950+0,-1100|d=3@-250,-850+0,-1000]
```

Without `vkrn` every advance is −1000. With it, a’s advance is −1050 and b is offset 50 further down, so b sits 100 units further from a: farther apart, not closer.

**Expected.** The values negated in vertical text, so that the advances shorten (a’s to −950) and the glyphs move together, as the specification says and as GPOS does; or, if HarfBuzz means to keep this, a note saying so.

**On main.** Still present (`main` at `8846dbbf6`, 20 November 2026). Found during the book’s move to 15.2.0, and shown live in the chapter Legacy kern.

---

## 86. An AAT DontAdvance loop that runs out of the operations budget doesn’t mark the buffer failed, unlike OpenType

**ID:** `HHW-86` · **Tag:** `hhw-20261120`

**Summary.** When OpenType shaping runs out of the buffer’s operations budget (`max_ops`), it marks the buffer unsuccessful (`src/hb-ot-layout-gsubgpos.hh:1244`–`:1246`, `:2163`–`:2164`), and every later lookup stops. When an AAT state machine’s DontAdvance loop runs it out, the driver just advances to the next glyph (`src/hb-aat-layout-common.hh:1498`): the buffer stays successful, later subtables run, and `hb_buffer_allocation_successful ()` reports success, so a client can’t tell that the budget ran out. Before 15.2.0 the per-subtable pass charge (removed by #6664) failed the buffer in this case, so the two paths agreed.

AAT’s way gives the better output here, and may be what HarfBuzz wants; but the two shaping paths now treat the same exhausted budget differently. A question for HarfBuzz’s developers rather than a clear bug.

**Repro.** `fonts/aat-inside-harfbuzz/loop.ttf` from the book’s repository: a `morx` whose first subtable loops on `a` with DontAdvance, and whose second turns `e` into `e.sc`.

```
$ hb-shape fonts/aat-inside-harfbuzz/loop.ttf ae
[a=0+500|e.sc=1+450]
```

The loop spends the whole budget on the `a`, and the second subtable still runs. In 15.1.0 the buffer was marked failed and the result was `[a=0+500|e=1+500]`.

**Expected.** Either the buffer marked unsuccessful when an AAT loop exhausts the budget, as in OpenType, or the difference documented.

**On main.** Still present (`main` at `8846dbbf6`, 20 November 2026). Found during the book’s move to 15.2.0, by reading the code; the chapter AAT inside HarfBuzz shows the result live.

---

Note from chapter 41: `test/fuzzing/README.md:13` lists the fuzzers without the gpu fuzzer (add to draft 42 if filed). Fixed on `main` by `202b3e2c2` (#6692), with report 42’s README item. Draft 43 is related to the closed issue #5993.
