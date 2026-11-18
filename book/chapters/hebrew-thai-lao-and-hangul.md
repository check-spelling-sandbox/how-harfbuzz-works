# Hebrew, Thai, Lao and Hangul

Hebrew, Thai, Lao and Korean need no syllable machine and no reordering of letters. A font’s own GSUB and GPOS can do nearly everything their text needs, and HarfBuzz’s shapers for them are small: three files, none of them 500 lines long, license included. Each exists for a job the font can’t do on its own, either because it has to be done to the characters before they become glyphs, or because it depends on what the font has.

| Shaper | Scripts | Hooks it fills | Its job |
|---|---|---|---|
| Hebrew | Hebrew | `compose`, `reorder_marks`, `gpos_tag` | puts one sequence of points in a better order; makes presentation forms for old fonts |
| Thai | Thai, Lao | `preprocess_text` | splits SARA AM; for old Thai fonts, swaps marks for the font’s private-use glyphs |
| Hangul | Hangul | `collect_features`, `override_features`, `data_create`, `preprocess_text`, `setup_masks` | decides, from the font, whether a syllable is one glyph or several jamo |

How a script gets one of these shapers is in [Choosing a script shaper](choosing-a-script-shaper.html), and where each hook runs is in [The pipeline, step by step](the-pipeline-step-by-step.html#where-the-shapers-hooks-run). The examples use test fonts in `fonts/hebrew-thai-lao-and-hangul/`, built by `build-fonts.py` there from three small feature files. Most glyphs are boxes, named after their characters. The Thai ones are drawn so that a mark’s different forms can be told apart. Every result on this page was produced by HarfBuzz {{hb}} when the page was built.

## Hebrew

### One sequence of points

Normalization sorts each run of marks by HarfBuzz’s modified combining classes, which for Hebrew follow the order of the SBL Hebrew manual rather than Unicode’s ([Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#modified-combining-classes)). Then it gives the shaper’s `reorder_marks` hook the sorted run ([Normalization](normalization.html)). The Hebrew shaper’s is short (`hb-ot-shaper-hebrew.cc:166`, trimmed):

```c++
for (unsigned i = start + 2; i < end; i++)
{
  unsigned c0 = info_cc (info[i - 2]);
  unsigned c1 = info_cc (info[i - 1]);
  unsigned c2 = info_cc (info[i - 0]);

  if ((c0 == CCC17 || c0 == CCC18) /* patah or qamats */ &&
      (c1 == CCC10 || c1 == CCC14) /* sheva or hiriq */ &&
      (c2 == CCC22 || c2 == BELOW) /* meteg or below */)
  {
    buffer->merge_clusters (i - 1, i + 1);
    hb_swap (info[i - 1], info[i]);
    break;
  }
}
```

In the sorted order, patah and qamats (modified classes 20 and 21) come just before sheva and hiriq (22 and 23), and those come before meteg (25) and the cantillation marks below, such as etnahta (220). Where all three are there in a row, the second and third swap, so the meteg or accent comes straight after the patah or qamats. It happens once in a run, at the first place it matches:

```shape
hebrew-mark.ttf  ל\u05B7\u05B4\u05BD  --language=he  --no-positions
hebrew-mark.ttf  ל\u05BD\u05B4\u05B7  --language=he  --no-positions
hebrew-mark.ttf  ל\u05B8\u05B0\u0591  --language=he  --no-positions
hebrew-mark.ttf  ל\u05B4\u05BD  --language=he  --no-positions
hebrew-mark.ttf  ל\u05B7\u05B4\u034F\u05BD  --language=he  --no-positions
```

The output is in visual order, so read it from the right: `lamed`, `patah`, `meteg`, `hiriq`, whichever order the points were typed in. A qamats, sheva and etnahta swap the same way. Without a patah or qamats, hiriq and meteg keep their sorted order. A combining grapheme joiner between the hiriq and the meteg puts them in different runs, and nothing moves (the joiner is the `space` glyph: [GSUB](gsub.html#joiners)).

The source gives no reason beyond the names in its comment, and the swap’s history is no clearer: `NEWS` for 5.0.0 says only “Improve multiple mark interactions in Hebrew”. The pattern is the one in the traditional spelling of Jerusalem, יְרוּשָׁלִַם, whose lamed carries both a patah and a hiriq, and the swap keeps a meteg or accent next to the first of the two vowels (inferred).

### Presentation forms for old fonts

Unicode’s Alphabetic Presentation Forms block has Hebrew **presentation forms** from U+FB1D to U+FB4F, most of them letters with their points, such as SHIN WITH SHIN DOT (U+FB2A) or VAV WITH HOLAM (U+FB4B). They have canonical decompositions, but they are excluded from composition, so normalization never makes one from its parts. Old fonts, with no GPOS to place the points, drew these forms instead and mapped them in their `cmap`.

The Hebrew shaper’s `compose` hook makes them anyway, for those fonts. When Unicode’s composition finds nothing, and the plan has no `mark` feature, it composes about three dozen pairs into presentation forms: a dagesh with each letter that has a dagesh form, shin and sin dots with SHIN, rafe with BET, KAF and PE, and five more with particular vowels (`hb-ot-shaper-hebrew.cc:81`). Normalization’s last round calls the hook, and as with any composition, only uses the result if the font has a glyph for it ([Normalization](normalization.html#round-three-recompose)). A composed form can compose again, so SHIN, dagesh and shin dot become U+FB2C in two steps.

`hebrew-old.ttf` has the letters, the points and seven presentation forms, and no layout tables at all. `hebrew-mark.ttf` adds a GPOS with a `mark` feature under `hebr`, and `hebrew-mark-DFLT.ttf` has the same GPOS under `DFLT`:

```shape
hebrew-old.ttf  ש\u05C1  --language=he
hebrew-old.ttf  ש\u05BC\u05C1  --language=he
hebrew-old.ttf  ו\u05B9  --language=he
hebrew-old.ttf  ח\u05BC  --language=he
hebrew-mark.ttf  ש\u05BC\u05C1  --language=he
hebrew-mark.ttf  ש\u05BC\u05C1  --language=he  --features=-mark
hebrew-mark-DFLT.ttf  ש\u05BC\u05C1  --language=he
```

The old font gets its single glyphs. HET has no dagesh form in Unicode, so its dagesh stays a mark, placed by HarfBuzz’s [fallback](fallbacks.html#hebrew-and-arabic) positioner. With a `mark` feature, the letters and points stay apart for GPOS to place, unless the feature is turned off, which takes it out of the plan.

The last line is the odd one. The test is whether the plan has a `mark` feature (`has_gpos_mark`, `hb-ot-shape.cc:196`), and planning finds the `DFLT` one. But the Hebrew shaper’s `gpos_tag` says that GPOS is only used if the script it chose is `hebr` ([Choosing a script shaper](choosing-a-script-shaper.html#hebrew-and-its-gpos-tag)). So that font’s GPOS is ignored, and its presentation forms aren’t used either: the points are placed by the fallback positioner, though the font has glyphs that would do better. The two checks look at different things (read from the code, and shown live here).

> For type designers: a Hebrew font gets presentation forms only if it has no `mark` feature at all, in any script, and gets its GPOS only if GPOS has a `hebr` script. A font that has both kinds of data should put its GPOS under `hebr`.

`HB_NO_OT_SHAPER_HEBREW_FALLBACK` leaves the presentation forms out, and `HB_NO_OT_SHAPE_FALLBACK` implies it.

## Thai and Lao

### SARA AM

**SARA AM**, U+0E33 ำ, is a vowel written as a small circle over the consonant and a spacing SARA AA after it. Unicode gives it a compatibility decomposition to NIKHAHIT (U+0E4D) and SARA AA (U+0E32), but a compatibility decomposition is not one that normalization applies. When a tone mark goes on the same consonant, it is typed before SARA AM but drawn above the circle.

The Thai shaper splits every SARA AM, and moves the NIKHAHIT back over any above-base marks before it. A long comment in `preprocess_text_thai` says this isn’t in Microsoft’s Thai specification, but is what Uniscribe and other engines do, and gives Eric Muller’s example: `<0E14, 0E4B, 0E33>` becomes `<0E14, 0E4D, 0E4B, 0E32>`. The loop, trimmed (`hb-ot-shaper-thai.cc:324`):

```c++
if (!IS_SARA_AM (u)) { buffer->next_glyph (); continue; }

/* Is SARA AM. Decompose and reorder. */
buffer->output_glyph (NIKHAHIT_FROM_SARA_AM (u));
_hb_glyph_info_set_continuation (&buffer->prev(), buffer);
buffer->replace_glyph (SARA_AA_FROM_SARA_AM (u));

/* Make Nikhahit be recognized as a ccc=0 mark when zeroing widths. */
unsigned int end = buffer->out_len;
_hb_glyph_info_set_general_category (&buffer->out_info[end - 2],
                                     HB_UNICODE_GENERAL_CATEGORY_NON_SPACING_MARK);

unsigned int start = end - 2;
while (start > 0 && IS_ABOVE_BASE_MARK (buffer->out_info[start - 1].codepoint))
  start--;
/* ...move the NIKHAHIT from end - 2 to start, and merge clusters... */
```

The two new characters are copies of SARA AM, so the NIKHAHIT is given the general category of a non-spacing mark, which the copy lacked. Then it moves back over the run of above-base marks: MAI HAN-AKAT, the upper vowels SARA I to SARA UEE, and MAITAIKHU to YAMAKKAN, which includes the tone marks. The clusters from the first of those to the SARA AA are merged, and then merged with the consonant’s cluster too.

```shape
thai.ttf  กำ  --language=th
thai.ttf  ด\u0E4Bำ  --language=th
thai.ttf  ด\u0E34\u0E4Bำ  --language=th
thai.ttf  ด\u0E4B\u0E4Dา  --language=th
```

In the second line the NIKHAHIT now comes before MAI CHATTAWA, so a font sees the order it sees for a consonant with an upper vowel and a tone mark, and can stack them the same way. In the third it goes back over SARA I as well. The fourth line types NIKHAHIT and SARA AA separately, after the tone mark. They aren’t moved: the comment says the reordering “is legit only when the NIKHAHIT comes from a SARA AM”. They aren’t merged into the consonant’s cluster either, as SARA AA’s cluster number, 3, shows.

The trace shows the change in the characters, before normalization or the `cmap` sees them:

```trace
font: thai.ttf
text: ด\u0E4Bำ
options: --language=th
from: start preprocess-text
to: end preprocess-text
```

This happens whether or not the font has Thai layout tables, and the font’s own glyph for SARA AM is never used. `thai-am-only.ttf` has one, and no glyphs for NIKHAHIT or SARA AA:

```shape
thai-am-only.ttf  กำ  --language=th
```

> For type designers: a Thai font needs glyphs for NIKHAHIT and SARA AA, and a Lao font for their Lao counterparts. A glyph for SARA AM is never reached in HarfBuzz.

### Old fonts and the private use area

Before OpenType, Thai fonts for Windows and the Mac had no GPOS to move a mark out of the way of another, or of a consonant’s ascender or descender. They drew the moved forms as extra glyphs, mapped in the `cmap` to code points in Unicode’s private use area (PUA): a tone mark drawn lower, for when there’s no vowel above, or further left, to clear PO PLA’s ascender; YO YING without its descender, for when a vowel goes below it. Windows fonts put them at U+F700 to U+F71A, Mac fonts at U+F884 to U+F89E. The software chose the form.

HarfBuzz chooses them too, if the text is Thai and GSUB didn’t find the `thai` script, which includes fonts with no GSUB at all (`hb-ot-shaper-thai.cc:368`). The rules follow Theppitak Karoonboonyanan’s description of Thai shaping, which the source cites (`linux.thai.net/~thep/th-otf/shaping.html`). `do_thai_pua_shaping` walks the text with two small state machines, one for the space above the consonant and one for the space below, both reset at each character that isn’t a Thai mark. A consonant sets their starting states by its shape:

| Consonants | Kind | Above | Below |
|---|---|---|---|
| PO PLA, FO FA, FO FAN | ascender (AC) | starts in T1 | B0 |
| YO YING, THO THAN | removable descender (RC) | T0 | B1 |
| DO CHADA, TO PATAK | strict descender (DC) | T0 | B2 |
| the other consonants, U+0E01 to U+0E2E | normal (NC) | T0 | B0 |
| anything else | | T3 | B2 |

Each mark is an upper vowel or sign (MAI HAN-AKAT, SARA I to SARA UEE, MAITAIKHU, NIKHAHIT, YAMAKKAN), a lower vowel (SARA U, SARA UU, PHINTHU) or a tone mark (MAI EK to THANTHAKHAT), and moves both machines on. The transitions that do something are:

| Where | Mark | Action |
|---|---|---|
| T0: nothing above yet | tone mark | shift down |
| T1: an ascender, nothing above yet | upper vowel | shift left |
| T1 | tone mark | shift down and left |
| T2: an ascender and a shifted upper vowel | tone mark | shift left |
| B1: a removable descender | lower vowel | take the descender off the consonant |
| B2: a strict descender, or a lower vowel already | lower vowel | shift down |

Any other mark changes nothing, and once a tone mark or a plain upper vowel has been placed, the state above is T3, where nothing changes again. An action is carried out by looking the character up in one of four small tables, which give its Windows and Mac PUA code points. If the font maps the Windows one, the character becomes that; if not, the Mac one; if neither, it stays as it was (`thai_pua_shape`, `:98`). “Take the descender off” changes the consonant, not the mark.

`thai-pua.ttf` has the same glyphs as `thai.ttf` and no layout tables, and maps the moved forms at the Windows code points. `thai-pua-mac.ttf` maps them at the Mac ones. `thai.ttf` has a GSUB under `thai`, whose one lookup removes YO YING’s descender before SARA U, as a modern font’s `ccmp` might:

```shape
thai.ttf  ก\u0E48 ป\u0E48 ป\u0E34\u0E48 ญ\u0E38 ฎ\u0E38  --language=th
thai-pua.ttf  ก\u0E48 ป\u0E48 ป\u0E34\u0E48 ญ\u0E38 ฎ\u0E38  --language=th
thai-pua-mac.ttf  ก\u0E48 ป\u0E48 ป\u0E34\u0E48 ญ\u0E38 ฎ\u0E38  --language=th
thai-pua.ttf  ก\u0E34\u0E48 ก\u0E48ำ  --language=th
```

In the PUA fonts, MAI EK on KO KAI comes down into the space a vowel would have used, and on PO PLA it also moves left. With SARA I before it on PO PLA, both move left and the tone mark stays high. YO YING loses its descender and DO CHADA’s SARA U goes lower. The Windows and Mac fonts get the same glyphs. In the last line nothing is shifted: an upper vowel is already there, and so is a NIKHAHIT from SARA AM, moved before the tone mark, which counts as one. Drawn, with `thai-pua.ttf` first and `thai.ttf`, which has no GPOS to move anything, second:

```draw
font: thai-pua.ttf
text: ก\u0E48 ป\u0E48 ป\u0E34\u0E48 ญ\u0E38 ฎ\u0E38
options: --language=th
```

```draw
font: thai.ttf
text: ก\u0E48 ป\u0E48 ป\u0E34\u0E48 ญ\u0E38 ฎ\u0E38
options: --language=th
```

In the second drawing, the tone marks on PO PLA run into its ascender and SARA U on DO CHADA runs into its descender. A modern font would move them with GPOS.

The new code points replace the characters in the buffer, before normalization, so the `cmap` is asked for them in the ordinary way:

```trace
font: thai-pua.ttf
text: ป\u0E34\u0E48
options: --language=th
from: start preprocess-text
to: end preprocess-text
```

Only the code point changes. The Unicode properties were set before this step, so the PUA character keeps the general category and combining class of the mark it replaced: it is still zeroed as a mark, and still sorted with the other marks (read from the code: the hook writes only `codepoint`). Each mark the machines look at also marks the text from its consonant to it as unsafe to break.

`HB_NO_OT_SHAPER_THAI_FALLBACK` (implied by `HB_NO_OT_SHAPE_FALLBACK`) leaves the PUA shaping out, but not SARA AM’s.

### Why Lao shares the shaper

The Lao block was laid out after the Thai one, and the characters the shaper cares about are at the same places, 0x80 higher. The comment in `preprocess_text_thai` puts it as “Note how the Lao versions are the same as Thai + 0x80”, and the code tests characters with that bit masked off:

```c++
#define IS_SARA_AM(x) (((x) & ~0x0080u) == 0x0E33u)
```

So Lao’s SARA AM, U+0EB3, is split and its NIKHAHIT moved the same way:

```shape
thai.ttf  ກ\u0EC8ຳ  --language=lo
```

The PUA fallback is Thai only: its tables hold only Thai code points, and the test is for `HB_SCRIPT_THAI`. The modified combining class that puts SARA U and SARA UU before PHINTHU is Thai only too, since “Lao does NOT have a U+0E3A equivalent” (`:297`). Neither script gets fallback mark positioning or a legacy `kern` table, as [Fallbacks](fallbacks.html#who-decides) and [Legacy kern](legacy-kern.html) explain: the Thai shaper’s `fallback_position` is false.

## Hangul

### Syllables and jamo

A Korean syllable is written as a block of two or three letters, **jamo**: a leading consonant (L), a vowel (V) and an optional trailing consonant (T). Unicode encodes the jamo, U+1100 to U+11FF and two extension blocks, and also 11,172 **precomposed syllables**, U+AC00 to U+D7A3, one for each combination of the 19 modern leading consonants, 21 vowels and 28 trailing choices (27 consonants or none). The syllables decompose to jamo by arithmetic, and the jamo compose back. The other jamo, in U+1100 to U+11FF and the extension blocks U+A960 and U+D7B0, are for **Old Hangul**, Korean as written before the twentieth century, whose syllables have no precomposed characters. A font shows an Old Hangul syllable by combining jamo glyphs, chosen by the features `ljmo`, `vjmo` and `tjmo`.

So one syllable can come as a precomposed character `<LV>` or `<LVT>`, as jamo `<L,V>` or `<L,V,T>`, or as `<LV,T>`, and a font may have a glyph for the precomposed syllable, or jamo that combine, or both. Fonts, a comment in normalization says, “are not designed to mix-and-match pre-composed syllables and Jamo” (`hb-ot-shape-normalize.cc:440`). The Hangul shaper’s job is to pick one for each syllable, from what the font has.

### Composing and decomposing, with the font in mind

It uses normalization mode `NONE`, so that normalization neither composes nor decomposes ([Normalization](normalization.html)), and does both itself in `preprocess_text_hangul` (`hb-ot-shaper-hangul.cc:144`). Its comment states the aim:

> - If the whole syllable can be precomposed, do that,
> - Otherwise, fully decompose and apply ljmo/vjmo/tjmo features.

In more detail, the loop looks at each character in turn:

* **`<L,V>` or `<L,V,T>`.** If all are modern jamo that compose, and the font has the precomposed syllable, they become that one character. Otherwise they stay jamo, marked for the three features.
* **`<LV>` or `<LVT>`.** If the next character is a T that composes and the font has the result, they become that. Otherwise, if the font lacks the syllable, or the next character is a T that can’t compose with it, the syllable is decomposed into jamo, if the font has all of them, and marked for the features. A T that followed is taken into the syllable.

“Compose” here means only the modern jamo: L from U+1100 to U+1112, V from U+1161 to U+1175, T from U+11A8 to U+11C2. And “the whole syllable” means it: if the font has GA but not GAK, then GA followed by the trailing KIYEOK is decomposed into three jamo, not left as GA and one jamo. `hangul.ttf` has the syllables GA (가) and GAK (각) and the jamo they’re made of. `hangul-no-gak.ttf` is the same without GAK. Each line spells GA or GAK one of the possible ways:

```shape
hangul.ttf  가  --language=ko
hangul.ttf  가  --language=ko
hangul.ttf  각  --language=ko
hangul.ttf  각  --language=ko
hangul.ttf  각  --language=ko
hangul-no-gak.ttf  가  --language=ko
hangul-no-gak.ttf  각  --language=ko
hangul-no-gak.ttf  각  --language=ko
hangul-no-gak.ttf  각  --language=ko
```

Each spelling of a syllable gives the same glyphs in the same font: GAK where the font has it, and otherwise the three jamo in their `ljmo`, `vjmo` and `tjmo` forms. The trace shows the decomposition in the characters:

```trace
font: hangul-no-gak.ttf
text: 각
options: --language=ko
from: start preprocess-text
to: end preprocess-text
```

An Old Hangul syllable never composes. Neither does a modern one with an Old Hangul trailing consonant, such as U+11EB PANSIOS, even where the font has the syllable before it:

```shape
hangul.ttf  ᄒᆞᆫ  --language=ko
hangul.ttf  가ᇫ  --language=ko
```

### The jamo features

`collect_features_hangul` adds `ljmo`, `vjmo` and `tjmo` to the plan. They aren’t global: each gets its own mask bit ([From features to lookups](from-features-to-lookups.html#mask-bits)). While it composes and decomposes, the shaper writes into a spare byte of each jamo it leaves (`hangul_shaping_feature`, a buffer variable that [Buffers](buffers.html) lists) which of the three it should get, and `setup_masks_hangul` turns that into the feature’s mask bit (`:408`). So a font’s `ljmo` lookup only touches an L that begins a syllable of jamo. A leading consonant on its own isn’t one:

```shape
hangul.ttf  ᄀ  --language=ko
hangul.ttf  가ᇫ  --language=ko
```

### `calt`, kept off jamo

Uniscribe doesn’t apply `calt` to Hangul, and some fonts, Noto Sans CJK and Source Han Sans among them, put all their jamo lookups in `calt`, where they aren’t wanted (the comment at `:69`, citing HarfBuzz discussion #4853; [Why shaping exists](why-shaping-exists.html) has more on Uniscribe’s influence). Until 14.3.0 HarfBuzz turned `calt` off for the whole run. Now `override_features_hangul` asks for `calt` once more, which gives it a mask bit of its own ([From features to lookups](from-features-to-lookups.html)), and `setup_masks_hangul` clears that bit on every jamo character, L, V or T. Everything else in the run, syllables included, keeps `calt`. The test font’s `calt` changes the first of two KIYEOKs, and the first of two GAs:

```shape
hangul.ttf  ᄀᄀ  --language=ko
hangul.ttf  가가  --language=ko
hangul.ttf  가가  --language=ko
```

The two jamo are left alone. The syllables, including the ones composed from jamo, get the rule. Since the check is on the character, jamo still count as context for other glyphs’ `calt` rules: a context needn’t have the lookup’s mask ([GSUB](gsub.html#context-and-chained-context)).

### Tone marks

Middle Korean marked pitch with dots beside the syllable, encoded as U+302E HANGUL SINGLE DOT TONE MARK and U+302F HANGUL DOUBLE DOT TONE MARK. They follow the syllable in the text but are written before it, to its left. The shaper moves a **tone mark** that follows a syllable it has just recognized to the front of it, and merges their clusters, unless the font gives the tone mark zero advance, in which case “we leave it untouched, assuming it’s designed to overstrike” (`:174`). In `hangul.ttf` the single dot has an advance and the double dot doesn’t:

```shape
hangul.ttf  가\u302E  --language=ko
hangul.ttf  ᄒᆞᆫ\u302E  --language=ko
hangul.ttf  가\u302F  --language=ko
```

The double dot stays where it was typed, and since the font has no GPOS, HarfBuzz’s fallback positioner centers it over the syllable, as it does any mark of its class ([Fallbacks](fallbacks.html#what-it-doesnt-take-into-account)).

A tone mark that doesn’t follow a recognized syllable gets a dotted circle, unless the buffer says not to or the font has none. It is the Hangul shaper’s own way of inserting one, one of four in HarfBuzz ([Dotted circles and broken clusters](dotted-circles-and-broken-clusters.html)). A tone mark with an advance goes before the circle, and a zero-width one after it. Two things about it look like bugs:

```shape
hangul.ttf  \u302E  --language=ko
hangul.ttf  ᄀ\u302E  --language=ko
hangul.ttf  \u302E  --language=ko  --bot
```

The dotted circle is made by `replace_glyphs`, which copies the tone mark’s buffer entry, Unicode properties and all (`:245`). So the circle is a spacing mark too, and when it follows a base, as it does after a lone KIYEOK in the second line, the fallback positioner treats it as one: it loses its advance and is placed over the KIYEOK. The generic dotted circle, inserted at the start of text, gets the properties of U+25CC instead (`hb-ot-shape.cc:567`). And that generic circle comes first: at the beginning of text (`--bot`), HarfBuzz has already put a dotted circle before the tone mark, but to the Hangul shaper that circle isn’t a syllable, so it adds a second.

> Since {{hb}}: fixed on HarfBuzz main by [`2831a8a`](https://github.com/harfbuzz/harfbuzz/commit/2831a8a45c77c180373987312958d31b3575a72c) and [`59fa8f0`](https://github.com/harfbuzz/harfbuzz/commit/59fa8f0c28df047b6786a60febf2c8b53a694e05) (17 November 2026), so main gives the circle U+25CC’s own properties, and it keeps its advance; with `--bot`, the circle already there serves as the tone mark’s base, and there is one.

### Old Hangul and `KOH`

The shaper never looks at the language. OpenType has a language system tag for Old Hangul, `KOH`, and HarfBuzz maps the language `ko` to `KOR` and then `KOH`, and `okm`, Middle Korean, to `KOH` (`hb-ot-tag-table.hh:113`, `:933`; [Choosing a script shaper](choosing-a-script-shaper.html#from-a-language-to-language-system-tags)). So a font that keeps its jamo features under `KOH` alone still gets them for Korean text, unless it also has a `KOR` language system. `hangul-koh.ttf` registers `ljmo`, `vjmo` and `tjmo` only under `KOH`:

```shape
hangul-koh.ttf  ᄒᆞᆫ  --language=ko
hangul-koh.ttf  ᄒᆞᆫ  --language=okm
hangul-koh.ttf  ᄒᆞᆫ  --language=en
```

> For type designers: the Hangul shaper uses a precomposed syllable whenever the font has it, and decomposes it whenever the font lacks it and has all its jamo. A font with jamo features for Old Hangul still gets precomposed glyphs for modern syllables. Put jamo lookups in `ljmo`, `vjmo` and `tjmo`, not `calt`, and don’t rely on `calt` reaching jamo.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-ot-shaper-hebrew.cc` | `compose_hebrew` (presentation forms), `reorder_marks_hebrew`, the `gpos_tag` |
| `src/hb-ot-shaper-thai.cc` | `preprocess_text_thai` (SARA AM), `do_thai_pua_shaping` and `thai_pua_shape` (the PUA fallback) |
| `src/hb-ot-shaper-hangul.cc` | `preprocess_text_hangul` (composing, decomposing, tone marks), the jamo features and `calt` masks |
| `src/hb-unicode.hh` | the modified combining classes for Hebrew, Thai and Lao |
| `src/hb-ot-shape-normalize.cc` | where `compose` and `reorder_marks` are called, and mode `NONE` |
| `src/hb-ot-shape.cc` | `has_gpos_mark` and the `gpos_tag` test, in the planner |
| `src/hb-ot-tag-table.hh` | `ko` to `KOR` and `KOH`, generated by `gen-tag-table.py` |

None of this chapter’s tables is generated: the Hebrew presentation forms, the Thai state machines and PUA mappings, and the Hangul constants are written out in the shaper files.

## In short

* Hebrew: after normalization sorts the points, one sequence is swapped so that a meteg or accent below follows a patah or qamats. Old fonts with no `mark` feature get Hebrew presentation forms, composed from letters and points. A `mark` feature under `DFLT` stops that, though the GPOS it is in is ignored.
* Thai and Lao: SARA AM is always split into NIKHAHIT and SARA AA, and the NIKHAHIT moved back before any tone mark, so the font needs those two glyphs.
* Thai fonts without a `thai` GSUB script get marks and consonants swapped for the Windows or Mac private-use glyphs that old fonts used, chosen by two small state machines.
* Lao shares the Thai shaper because its block mirrors Thai’s, 0x80 higher; the PUA fallback is Thai only.
* Hangul: each syllable becomes a precomposed glyph if the font has the whole of it, and jamo with `ljmo`, `vjmo` and `tjmo` otherwise. `calt` is kept off jamo only.
* Hangul tone marks move before their syllable unless they have zero width. The dotted circle the shaper adds for a stray tone mark is given the tone mark’s properties, which looks like a bug.

> Since {{hb}}: fixed on HarfBuzz main by [`2831a8a`](https://github.com/harfbuzz/harfbuzz/commit/2831a8a45c77c180373987312958d31b3575a72c) (17 November 2026), so main gives the tone mark’s dotted circle U+25CC’s own properties (the item on Hangul tone marks).
