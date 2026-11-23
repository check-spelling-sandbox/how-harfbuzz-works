# Dotted circles and broken clusters

A combining mark needs something to combine with. When a vowel sign begins a line, or follows a space in a script whose signs can’t stand alone, or comes in an order the script doesn’t allow, there is nothing for it to sit on, and a renderer that drew it anyway would hang it over whatever came before, or over nothing. The convention, which the Unicode code charts use for every combining mark, is to show such a mark on U+25CC DOTTED CIRCLE, so that the reader sees a mark and sees that it is out of place. HarfBuzz follows the convention. It doesn’t change the text: the circle is only a glyph in the output, given a cluster of the text it stands in.

There are four places in HarfBuzz that insert one. They were written at different times, for different shapers, and they differ in when they run, in what they test, in what they give the circle, and in whether they check that the font has one at all. The script chapters met each of them in passing. This chapter puts them side by side.

The examples use three small test fonts, built from `fonts/dotted-circles-and-broken-clusters/circles.fea`. Their glyphs are boxes: a Latin *a* and a combining acute, and a few Devanagari, Khmer, Myanmar, and Brahmi letters and vowel signs. `circle.ttf` has a dotted circle, a GDEF table, and a `mark` feature that attaches the acute and Devanagari vowel sign U to a base, the dotted circle among them. `no-circle.ttf` is the same font with U+25CC left out of its `cmap` (the glyph is still there, but no character reaches it). `bare.ttf` is `circle.ttf` without GDEF and GPOS. A few examples borrow fonts from the script chapters. Every result on this page was produced by HarfBuzz {{hb}} when the page was built.

## Four paths

| | Function | When | Shapers | When it inserts | Checks the font | What it inserts |
|---|---|---|---|---|---|---|
| 1 | `hb_insert_dotted_circle` (`hb-ot-shape.cc:551`) | step 3, before `preprocess_text` | all | a mark at the start of a paragraph | yes | the character, with U+25CC’s own properties |
| 2 | `_hb_preprocess_text_vowel_constraints` (`hb-ot-shaper-vowel-constraints.cc:62`) | `preprocess_text` | Indic, USE | inside a listed sequence of characters | no | the character, with U+25CC’s own properties |
| 3 | `preprocess_text_hangul` (`hb-ot-shaper-hangul.cc:232`) | `preprocess_text` | Hangul | a tone mark with no syllable | yes | the character, with U+25CC’s own properties |
| 4 | `hb_syllabic_insert_dotted_circles` (`hb-ot-shaper-syllabic.cc:33`) | in GSUB, at the first reordering pause | Indic, Khmer, Myanmar, USE | a broken cluster | yes | the glyph, with a category of the shaper’s |

So the vowel constraints are not a fifth way, beside four others: they are one of the four. No other code in {{hb}} inserts U+25CC. `NEWS` first mentions them in 0.9.4: “Implement dotted-circle insertion.”

All four obey the buffer flag `HB_BUFFER_FLAG_DO_NOT_INSERT_DOTTED_CIRCLE`, new in 2.4.0, which turns them all off. `hb-shape` has no option for it. The order in the table is the order they run in, and an earlier one can settle a case before a later one sees it: a circle inserted at step 3 is a base by the time the syllable grammars run, so it leaves nothing broken.

## At the start of a paragraph

The first path is the only one outside the script shapers, and the strictest about when it applies. Here it is whole (`hb-ot-shape.cc:551`):

```c++
static void
hb_insert_dotted_circle (hb_buffer_t *buffer, hb_font_t *font)
{
  if (unlikely (buffer->flags & HB_BUFFER_FLAG_DO_NOT_INSERT_DOTTED_CIRCLE))
    return;

  if (!(buffer->flags & HB_BUFFER_FLAG_BOT) ||
      buffer->context_len[0] ||
      !_hb_glyph_info_is_unicode_mark (&buffer->info[0]))
    return;

  if (!font->has_glyph (0x25CCu))
    return;

  hb_glyph_info_t dottedcircle = {0};
  dottedcircle.codepoint = 0x25CCu;
  _hb_glyph_info_set_unicode_props (&dottedcircle, buffer);

  buffer->clear_output ();

  buffer->idx = 0;
  hb_glyph_info_t info = dottedcircle;
  info.cluster = buffer->cur().cluster;
  info.mask = buffer->cur().mask;
  (void) buffer->output_info (info);

  buffer->sync ();
}
```

Four conditions: the buffer must be marked as the beginning of a paragraph (`BOT`), it must have no pre-context, its first character must be a mark (any general category M), and the font must map U+25CC. [Buffers](buffers.html#buffer-flags) explains why `BOT` is needed and why pre-context overrides it. What goes in is the character U+25CC, before normalization has mapped anything to glyphs, with its own Unicode properties, and with the first character’s cluster and mask:

```shape
circle.ttf  \u0301  --bot
circle.ttf  \u0301
no-circle.ttf  \u0301  --bot
bare.ttf  \u0301  --bot
```

The circle is an ordinary character from here on. In `circle.ttf`, the font’s GPOS attaches the acute to it. In `bare.ttf` there is no GPOS, and [the fallback positioner](fallbacks.html#placing-marks) places the acute over the circle as over any base, because the circle’s general category, So, makes it one. Only the first character is looked at: a mark later in the text, after a space say, gets nothing from this path.

“A mark” is read from the general category alone, so it includes the variation selectors and COMBINING GRAPHEME JOINER, which are default ignorables:

```shape
circle.ttf  \uFE00  --bot
circle.ttf  \u034F  --bot
```

The selector is then hidden, as default ignorables are, and leaves a visible circle with nothing on it. A paragraph rarely starts with one, but an application that sets `BOT` on every run of a paragraph and gives no pre-context would get it at the start of every run (inferred from the code).

This path runs for every script and every shaper, including `dumber`, which a font with `morx` gets ([Choosing a script shaper](choosing-a-script-shaper.html#aat-fonts)), and the default shaper that Devanagari falls back to in a font with only `DFLT` or `latn` lookups. In the syllabic shapers it comes first. A vowel sign at the start of a paragraph already has its circle when the Indic shaper looks for broken clusters:

```trace
font: circle.ttf
text: \u093F
options: --language=hi --bot
from: start preprocess-text
to: end reordering indic initial
```

The circle is there in the first line, before any of the shaper’s steps; the Indic grammar sees a dotted circle and a vowel sign, a standalone cluster, and its own insertion finds “no broken syllables”.

## Broken clusters

The four syllabic shapers each divide the text into syllables with a grammar, as [the Indic model](indic-model.html#finding-syllables), [Khmer and Myanmar](khmer-and-myanmar.html), and [the Universal Shaping Engine](universal-shaping-engine.html) describe. Each grammar has a catch-all for marks with no base: a **broken cluster**, a syllable made of what may follow a base, with no base before it. Indic’s is `reph? n? complex_syllable_tail` (`hb-ot-shaper-indic-machine.rl:107`), Khmer’s `Robatic? (H.cn)* (H | syllable_tail)`, Myanmar’s `k? VS? syllable_tail`, and USE’s a tail with an optional leading repha. A lone vowel sign is one in all four. When the scanner finds one, it also sets a buffer scratch flag, `HB_BUFFER_SCRATCH_FLAG_HAS_BROKEN_SYLLABLE`, so a buffer with none can skip the next step at once.

Which sequences count as broken is the grammars’ business, and it changes with them. `NEWS` for 10.2.0, for instance, says “Don’t insert U+25CC DOTTED CIRCLE before superscript/subscript digits”.

### Where the circle goes

Each shaper calls the same function at the start of its reordering, in the GSUB pause after its first features: Indic in `initial_reordering_indic` (`hb-ot-shaper-indic.cc:986`), Khmer in `reorder_khmer` (`hb-ot-shaper-khmer.cc:312`), Myanmar in `reorder_myanmar` (`hb-ot-shaper-myanmar.cc:333`), and USE in `reorder_use` (`hb-ot-shaper-use.cc:476`). Trimmed, it is (`hb-ot-shaper-syllabic.cc:33`):

```c++
  if (unlikely (buffer->flags & HB_BUFFER_FLAG_DO_NOT_INSERT_DOTTED_CIRCLE))
    return false;
  if (likely (!(buffer->scratch_flags & HB_BUFFER_SCRATCH_FLAG_HAS_BROKEN_SYLLABLE)))
    return false;

  hb_codepoint_t dottedcircle_glyph;
  if (!font->get_nominal_glyph (0x25CCu, &dottedcircle_glyph))
    return false;

  hb_glyph_info_t dottedcircle = {0};
  dottedcircle.ot_shaper_var_u8_category() = dottedcircle_category;
  if (dottedcircle_position != -1)
    dottedcircle.ot_shaper_var_u8_auxiliary() = dottedcircle_position;
  dottedcircle.codepoint = dottedcircle_glyph;

  buffer->clear_output ();
  buffer->idx = 0;
  unsigned int last_syllable = 0;
  while (buffer->idx < buffer->len && buffer->successful)
  {
    unsigned int syllable = buffer->cur().syllable();
    if (unlikely (last_syllable != syllable && (syllable & 0x0F) == broken_syllable_type))
    {
      last_syllable = syllable;

      hb_glyph_info_t ginfo = dottedcircle;
      ginfo.cluster = buffer->cur().cluster;
      ginfo.mask = buffer->cur().mask;
      ginfo.syllable() = buffer->cur().syllable();

      /* Insert dottedcircle after possible Repha. */
      if (repha_category != -1)
      {
	while (buffer->idx < buffer->len && buffer->successful &&
	       last_syllable == buffer->cur().syllable() &&
	       buffer->cur().ot_shaper_var_u8_category() == (unsigned) repha_category)
	  (void) buffer->next_glyph ();
      }

      (void) buffer->output_info (ginfo);
    }
    else
      (void) buffer->next_glyph ();
  }
  buffer->sync ();
```

It walks the buffer once and puts one circle at the start of each broken syllable, recognized by the low four bits of its `syllable` byte. The four shapers differ only in the arguments:

| Shaper | Broken type | Circle’s category | Skipped first | Position |
|---|---|---|---|---|
| Indic | `indic_broken_cluster` | `DOTTEDCIRCLE` | `Repha` | `POS_END` |
| Khmer | `khmer_broken_cluster` | `DOTTEDCIRCLE` | nothing | none |
| Myanmar | `myanmar_broken_cluster` | `DOTTEDCIRCLE` | nothing | none |
| USE | `use_broken_cluster` | `B` | `R` | none |

The category is the one a typed U+25CC gets from the shaper’s table, a base in every case, so once the circle is in, the broken cluster is reordered like any other syllable with a base. In the Indic shaper it goes through the code for standalone clusters (`hb-ot-shaper-indic.cc:965`, “We already inserted dotted-circles, so just call the standalone_cluster”), and the circle becomes the base. The Indic position `POS_END` differs from the `POS_BASE_C` a typed circle has, but the base search sets the base’s position to `POS_BASE_C` anyway (`:638`).

One lone vowel sign in each script, with and without a circle in the font:

```shape
circle.ttf  \u093F  --language=hi
circle.ttf  \u17B6  --language=km
circle.ttf  \u102D  --language=my
circle.ttf  \u{11038}  --language=sa
no-circle.ttf  \u093F  --language=hi
no-circle.ttf  \u17B6  --language=km
no-circle.ttf  \u102D  --language=my
no-circle.ttf  \u{11038}  --language=sa
```

Devanagari vowel sign I is a pre-base sign, and the reordering moves it before the circle, as it would before a consonant. Without a circle in the font, nothing is inserted, and the broken cluster is reordered without a base. The trace of the first line shows the step:

```trace
font: circle.ttf
text: \u093F
options: --language=hi
from: start reordering indic initial
to: end reordering indic initial
```

### After the repha

A circle goes after any glyphs at the start of the syllable that have the shaper’s repha category: `Repha` in the Indic shaper, `R` in USE. Both are categories of characters, not of sequences. In the Indic table one character has `Repha`, U+0D4E MALAYALAM LETTER DOT REPH (`hb-ot-shaper-indic-table.cc:96`); in USE, `R` is given to characters whose Indic syllabic category is `Consonant_Preceding_Repha` or `Consonant_Prefixed` (`gen-use-table.py:283`). With the Malayalam glyphs of [the Indic model](indic-model.html)’s `scripts.ttf`, the dot reph is passed over, and then moved after its new base:

```shape
indic-model/scripts.ttf  \u0D4E\u0D3E  --language=ml
indic-model/scripts.ttf  \u0D4E\u0D3E  --language=ml  --cluster-level=1
```

The reph that RA and a virama make is not a character, and the insertion doesn’t pass over it, though the Indic grammar puts it in the broken cluster (`reph?`). So RA, virama, and a vowel sign get their circle before the RA. The circle becomes the base, the RA a consonant after it, and the reph is never made: the Indic shaper makes one only of RA + virama at the start of a syllable. A circle typed in the same place gives a reph on the circle:

```shape
circle.ttf  र\u094D\u0941  --language=hi
circle.ttf  र\u094D\u25CC\u0941  --language=hi
universal-shaping-engine/deva3.ttf  र\u094D\u0941  --language=hi
```

The first line shows RA with a visible virama after the circle, where the second has the reph; it looks like a bug (the comment “Insert dottedcircle after possible Repha” suggests the reph was meant to be kept). When the circle comes in before the shaper looks, from the vowel constraints below, the reph does form on it, and HarfBuzz’s own test, `indic-vowel-letter-spoofing.tests`, expects exactly that for RA, virama, and LETTER I. The third line is USE, under `dev3`, where RA + virama is a syllable of its own, a virama-terminated cluster, so the vowel sign after it is broken by itself and the reph, formed in the first syllable, stays before the circle.

> Since {{hb}}: fixed on HarfBuzz main by [#6830](https://github.com/harfbuzz/harfbuzz/pull/6830) (20 November 2026), so main puts the circle after RA + virama, and the first line has the reph on the circle, as the second does.

### Clusters

The circle takes the cluster and the mask of the first glyph of the syllable, even when it goes in after a repha, as the second Malayalam line shows: the circle is in cluster 0 with the dot reph, not cluster 1 with the vowel sign. After a space, a vowel sign is a broken cluster in the Indic shaper (and in USE, as [Spaces and symbols](universal-shaping-engine.html#spaces-and-symbols) shows):

```shape
circle.ttf  क \u093F  --language=hi
circle.ttf  क \u093F  --language=hi  --cluster-level=1
```

At the default cluster level, the vowel sign was already merged into the space’s cluster at step 4 ([The pipeline, step by step](pipeline-step-by-step.html#properties-and-clusters)), because a mark continues the grapheme before it, so the circle gets cluster 1. At level 1 nothing was merged, and it gets the vowel sign’s cluster, 2.

### What the circle is to GDEF

The circle goes in as a glyph ID, after GSUB has begun, so it misses two things every other glyph got. Its Unicode properties are all zero, which reads as general category Cc, a control character, with combining class 0. And its glyph properties are zero, because GDEF classes, or synthesized classes in a font without GDEF, were given out at the start of GSUB (`hb_ot_layout_substitute_start`, and `hb_synthesize_glyph_classes`, `hb-ot-shape.cc:858`), before the pause. So the inserted circle is neither a base nor a mark to HarfBuzz, whatever the font’s GDEF says about the dotted circle glyph, until a substitution replaces it.

That mostly doesn’t matter. GPOS mark attachment looks back for the nearest glyph that isn’t a mark (`MarkBasePosFormat1.hh:105`), and the circle isn’t one, so a mark attaches to it as to a typed circle. The syllabic shapers have no fallback positioning, so without GPOS a mark is left where it is:

```shape
circle.ttf  \u0941  --language=hi
bare.ttf  \u0941  --language=hi
```

Where it would matter is a lookup flag that skips base glyphs: such a lookup skips a typed circle the font classes as a base, but not an inserted one (inferred from the code; not shown).

### The pre-context isn’t consulted

The grammars run over the buffer only. A vowel sign at the start of a run is a broken cluster even when the pre-context ends in a consonant it belongs to:

```shape
circle.ttf  \u093F  --language=hi  --text-before=क
```

The first path checks for pre-context, and this one doesn’t. So an application that splits a syllable between two runs, for a change of font or style, gets a dotted circle at the split, whatever context it passes.

## Hangul’s tone marks

The Hangul shaper does its own composition in `preprocess_text`, and there it gives a dotted circle to a tone mark that doesn’t follow a syllable, if the flag allows and the font has U+25CC (`hb-ot-shaper-hangul.cc:232`). A tone mark with an advance goes before the circle, a zero-width one after it:

```shape
hebrew-thai-lao-and-hangul/hangul.ttf  \u302E  --language=ko
hebrew-thai-lao-and-hangul/hangul.ttf  \u302E  --language=ko  --text-before=가
```

It doesn’t look at the pre-context either. The circle is made with `replace_glyphs`, which copies the tone mark’s entry, and is then given U+25CC’s own Unicode properties (`:247`), so it is a base, not a spacing mark like the tone mark. With `BOT`, path 1 has already put a circle before the tone mark, and the shaper takes a dotted circle as a tone mark’s base (`:402`), so the text gets one circle, not two. [Tone marks](hebrew-thai-lao-and-hangul.html#tone-marks) shows both. Before 15.2.0 the circle kept the tone mark’s properties, and with `BOT` there were two ([HHW-19](bugs-this-book-found.html#hhw-19), [HHW-20](bugs-this-book-found.html#hhw-20)).

## Vowel constraints

Some scripts can write the same vowel in two ways that look alike. In Devanagari, LETTER A with VOWEL SIGN AA looks like LETTER AA, but it isn’t LETTER AA to a search, a sort, or a spelling checker, and HarfBuzz’s tests of it are called “vowel letter spoofing”. Microsoft lists such sequences as invalid clusters, and HarfBuzz shows a dotted circle inside them, since issue #1019, in what the code calls an “UGLY UGLY UGLY business of adding dotted-circle in the middle of vowel-sequences that look like another vowel”.

### The data

The list is Microsoft’s: `src/ms-use/IndicShapingInvalidCluster.txt`, whose header gives the dates 2015-03-12 and 2019-11-08 and which defines one property, `Indic_Shaping_Invalid_Cluster`: “sequences of characters that should be treated as invalid clusters”. It has 103 sequences in 16 scripts: Devanagari, Bengali, Gurmukhi, Gujarati, Oriya, Tamil, Telugu, Kannada, Malayalam, Sinhala, Brahmi, Khojki, Khudawadi, Tirhuta, Modi, and Takri. Most are an independent vowel followed by a vowel sign, but not all:

| Sequence | Kind |
|---|---|
| U+0905 U+093E, DEVANAGARI LETTER A + VOWEL SIGN AA | independent vowel + vowel sign |
| U+0930 U+094D U+0907, DEVANAGARI RA + VIRAMA + LETTER I | a reph on an independent vowel |
| U+0AC5 U+0ABE, GUJARATI VOWEL SIGN CANDRA E + VOWEL SIGN AA | two vowel signs |
| U+0C3F U+0C55, TELUGU VOWEL SIGN I + LENGTH MARK | a vowel sign + a length mark |
| U+0D91 U+0DCA, SINHALA LETTER EYANNA + AL-LAKUNA | independent vowel + virama |
| U+114AA U+114B5, TIRHUTA LETTER LA + VOWEL SIGN VOCALIC R | consonant + vowel sign |

`gen-vowel-constraints.py` turns the list into C++ (and, with `--rust`, into Rust for HarfRust). It reads `Scripts.txt` too, to find each sequence’s script from its first character (`:189`), because the generated function switches on the buffer’s script. It builds a tree of the sequences by their first characters (`ConstraintSet.add`, `:71`), and a sequence that begins with a shorter one is absorbed by it: Gujarati A + AA + CANDRA E is dropped, since A + AA already matches. The output, `src/hb-ot-shaper-vowel-constraints.cc`, records the headers of both files it was made from; in {{hb}}, `Scripts-18.0.0.txt`.

### The code

The function is the Indic and USE shapers’ `preprocess_text` (`hb-ot-shaper-indic.cc:1478`, `hb-ot-shaper-use.cc:495`), so it runs before normalization, on characters. Its Devanagari case begins (trimmed):

```c++
    case HB_SCRIPT_DEVANAGARI:
      for (buffer->idx = 0; buffer->idx + 1 < count && buffer->successful;)
      {
	unsigned int start = buffer->out_len;
	bool matched = false;
	switch (_vowel_constraint_glyph (buffer, 0))
	{
	  case 0x0905u:
	    switch (_vowel_constraint_glyph (buffer, 1))
	    {
	      case 0x093Au: case 0x093Bu: case 0x093Eu: case 0x0945u:
	      ...
		matched = true;
		break;
	    }
	    break;
	  ...
	  case 0x0930u:
	    if (0x094Du == _vowel_constraint_glyph (buffer, 1) &&
		buffer->idx + 2 < count &&
		0x0907u == _vowel_constraint_glyph (buffer, 2))
	    {
	      _next_vowel_constraint_glyph (buffer);
	      matched = true;
	    }
	    break;
	}
	_next_vowel_constraint_glyph (buffer);
	if (matched)
	{
	  _output_with_dotted_circle (buffer);
	  buffer->unsafe_to_break_in_outbuffer (start, buffer->out_len);
	}
      }
      break;
```

The circle goes before the last character of the sequence: between A and AA, and between the virama and I. `_output_with_dotted_circle` (`hb-ot-shaper-vowel-constraints.cc:56`) outputs U+25CC with `output_glyph`, then copies the next character. It checks the flag (`:70`), but nothing checks the font:

```shape
circle.ttf  अ\u093E  --language=hi
circle.ttf  अ\u093E  --language=hi  --cluster-level=1
circle.ttf  र\u094Dइ  --language=hi
no-circle.ttf  अ\u093E  --language=hi
```

In a font without a dotted circle, the inserted U+25CC is mapped like any other character, to `.notdef`. The trace shows the character arriving inside `preprocess_text`:

```trace
font: circle.ttf
text: अ\u093E
options: --language=hi
from: start preprocess-text
to: end preprocess-text
```

After RA + virama the circle is the base of a standalone cluster that begins with a reph, so the reph forms on it, as the test expects.

### What the circle is given

`output_glyph` makes the new entry by copying the current one, which is the character the circle goes before (`hb-buffer.hh:313`). So the circle gets that character’s cluster and mask. `_output_dotted_circle` (`hb-ot-shaper-vowel-constraints.cc:47`) then gives it U+25CC’s own Unicode properties, as path 1 does, and clears the continuation bit, so that the circle starts a grapheme of its own. At the default cluster level, step 4 had already merged the vowel sign into the cluster of the letter before it, so all three glyphs share cluster 0; at level 1 the circle has the vowel sign’s cluster, 1.

Its general category is So, a base, so the circle keeps its advance even in a font without GDEF, where HarfBuzz synthesizes classes from the general category. Brahmi is shaped by USE, which zeroes marks by class before GPOS ([Zeroing marks](fallbacks.html#zeroing-marks)):

```shape
circle.ttf  \u{11005}\u{11038}  --language=sa
bare.ttf  \u{11005}\u{11038}  --language=sa
bare.ttf  \u{11038}  --language=sa
```

In `bare.ttf`, which has no GDEF, the circle between A and the vowel sign keeps its advance, as the one path 4 gives a lone vowel sign does. Before 15.2.0 the circle copied the vowel sign’s properties, became a mark, and lost its advance ([HHW-35](bugs-this-book-found.html#hhw-35)).

### Only adjacent characters

The match is on characters next to each other in the buffer, except that default ignorables between them are passed over (`_vowel_constraint_glyph`, `:24`). A ZWJ, which the Indic grammar allows inside a syllable, doesn’t hide the sequence, nor does CGJ, which ends the syllable:

```shape
circle.ttf  अ\u200D\u093E  --language=hi
circle.ttf  अ\u034F\u093E  --language=hi
```

In both, the circle goes in after the joiner, before the vowel sign, and the matched characters are made unsafe to break between (`:138`), since the circle depends on all of them. Before 15.2.0 the ZWJ hid the sequence, and LETTER A and VOWEL SIGN AA came out together with no circle ([HHW-34](bugs-this-book-found.html#hhw-34)). Anything else between the two characters breaks the match, and so does a split between runs: the function doesn’t look at the pre-context, though there, path 4 catches the vowel sign as a broken cluster.

> For type designers: give the font a dotted circle, mapped to U+25CC and classed as a base in GDEF, with the anchors of a base for every mark class. Three of the four paths do nothing without it, and the fourth puts `.notdef` in its place. Expect the inserted circle in places a typed one would be, and test the shaping of lone vowel signs as well as complete syllables.

> For programmers: set `BOT` on the first run of a paragraph and pass the text before every other run as pre-context: path 1 depends on both. The syllabic and Hangul paths don’t read context, so don’t split a syllable between runs. To show text exactly as typed, with no circles, set `HB_BUFFER_FLAG_DO_NOT_INSERT_DOTTED_CIRCLE`.

## Configuration

`HB_NO_OT_SHAPER_VOWEL_CONSTRAINTS` empties the vowel constraints: the function returns at once (`hb-ot-shaper-vowel-constraints.cc:67`). It is implied by `HB_NO_OT_SHAPE_FALLBACK` (`hb-config.hh:174`), which `HB_NO_LEGACY` implies (`:156`). The other three paths have no macro of their own; they go with the `ot` shaper (`HB_NO_OT_SHAPE` leaves out `hb-ot-shaper-syllabic.cc`).

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-ot-shape.cc` | `hb_insert_dotted_circle`, and where it is called, step 3 of `hb_ot_shape_internal` |
| `src/hb-ot-shaper-syllabic.cc` | `hb_syllabic_insert_dotted_circles`, shared by the four syllabic shapers |
| `src/hb-ot-shaper-indic.cc`, `-khmer.cc`, `-myanmar.cc`, `-use.cc` | the calls, at the start of each shaper’s reordering, with its arguments |
| `src/hb-ot-shaper-*-machine.rl` | the grammars, and their `broken_cluster` rules |
| `src/hb-ot-shaper-hangul.cc` | the tone-mark circle, in `preprocess_text_hangul` |
| `src/hb-ot-shaper-vowel-constraints.cc` | the vowel constraints (generated) |
| `src/gen-vowel-constraints.py` | its generator |
| `src/ms-use/IndicShapingInvalidCluster.txt` | Microsoft’s list of invalid sequences |
| `src/hb-buffer.h` | `HB_BUFFER_FLAG_DO_NOT_INSERT_DOTTED_CIRCLE` |
| `test/shape/data/in-house/tests/indic-vowel-letter-spoofing.tests`, `use-vowel-letter-spoofing.tests` | HarfBuzz’s tests of the vowel constraints |

## In short

* Four functions insert U+25CC DOTTED CIRCLE: one at the start of a paragraph, for every shaper; the vowel constraints, for Indic and USE; the Hangul shaper’s, for tone marks; and one for broken clusters, shared by Indic, Khmer, Myanmar, and USE. One buffer flag turns them all off.
* The first needs `BOT`, no pre-context, and a leading mark, and runs before the others, so its circle leaves nothing broken. It inserts the character, with U+25CC’s own properties.
* A broken cluster is a syllable with no base. Its circle goes in at the syllable’s start, after a repha character but not after RA + virama, as a glyph with the shaper’s base category, the syllable’s first cluster, and no Unicode or GDEF properties.
* The vowel constraints are Microsoft’s 103 sequences in 16 scripts, generated into code, matched across default ignorables such as ZWJ. They insert the character, with U+25CC’s own properties, without checking the font, so a font without a dotted circle gets `.notdef`.
* Only the first path reads the pre-context. A syllable split between runs gets a circle at the split.
