# Normalization

Text arrives as Unicode, and Unicode often has more than one way to write the same thing. *é* can be one character, U+00E9, or two, *e* followed by U+0301 COMBINING ACUTE ACCENT. A font may have a glyph for one spelling and not the other, or for neither. Before any lookup runs, HarfBuzz has to settle which characters it will ask the font for, and it does so by asking the font.

That is HarfBuzz’s normalization. It decomposes characters, puts marks into a fixed order, and composes them again, much as Unicode’s own normalization forms do. The difference is that each step is taken only if the font has glyphs for the result. The `cmap` lookups this needs are the real ones: the glyph IDs that GSUB later starts from are the ones found here.

This chapter follows the three rounds in `hb-ot-shape-normalize.cc`, and the choices a script shaper can make about them. The examples use one test font, `norm.ttf`, built from `fonts/normalization/norm.fea`, with the coverage below. Its glyphs are boxes, and it has no GPOS, so most examples leave positions out. Every result on this page was made by HarfBuzz {{hb}} when the page was built.

| The font has | It doesn’t have |
|---|---|
| *a e o u A*, *á â é ü ǜ ȫ Å*, Ω, the angstrom sign Å (U+212B) | *à ê ö ǘ ạ ụ ñ x* |
| combining grave, acute, macron, dieresis, ring above, dot below | combining circumflex (U+0302) |
| U+2010 HYPHEN, the space | U+2011 NON-BREAKING HYPHEN, every other space |
| Devanagari NA, NNNA, NUKTA and the vowel sign AA | |

## Why normalize, and why against the font

Two strings are canonically equivalent when Unicode says they mean the same text, as with the two spellings of *é* ([Why shaping exists](why-shaping-exists.html#one-character-several-glyphs-or-the-other-way-round-decomposition) has the first example). Unicode’s normalization forms turn every equivalent string into one chosen spelling: NFD decomposes everything, and NFC decomposes and then composes everything that can be composed. Neither is what a font wants. A font with *á* but no *é* is best served by composed *a*+acute and decomposed *é*, and no normal form gives both.

So HarfBuzz uses the font’s `cmap` as its guide. The comment at the top of `hb-ot-shape-normalize.cc` sets out what it is for (`:36`, trimmed):

```c++
 * In general what happens is that: each grapheme is decomposed in a chain
 * of 1:2 decompositions, marks reordered, and then recomposed if desired,
 * so far it's like Unicode Normalization.  However, the decomposition and
 * recomposition only happens if the font supports the resulting characters.
 *
 * The goals are:
 *
 *   - Try to render all canonically equivalent strings similarly.  To really
 *     achieve this we have to always do the full decomposition and then
 *     selectively recompose from there.  It's kinda too expensive though, so
 *     we skip some cases.  [...]
 *
 *   - When a font has a precomposed character for a sequence but the 'ccmp'
 *     feature in the font is not adequate, use the precomposed character
 *     which typically has better mark positioning.
 *
 *   - When a font does not support a combining mark, but supports it precomposed
 *     with previous base, use that.  [...]
 *
 *   - When a font does not support a character but supports its canonical
 *     decomposition, well, use the decomposition.
 *
 *   - The shapers can customize the compose and decompose functions to
 *     offload some of their requirements to the normalizer.  [...]
```

Here is the first goal at work. *ǘ* is U+01D8, and its canonical decomposition is *u*, U+0308 COMBINING DIAERESIS, U+0301 COMBINING ACUTE ACCENT. It can be typed in three ways, all equivalent. A fourth line adds a different letter, *ǖ*, with a macron in place of the acute:

```shape
norm.ttf  ǘ  --no-positions
norm.ttf  ü\u0301  --no-positions
norm.ttf  u\u0308\u0301  --no-positions
norm.ttf  ǖ  --no-positions
```

The first three come out the same: *ü*, which the font has, and the acute, which it also has. The font has no *ǖ* either, so it gets *ü* and the macron.

HarfBuzz’s normalization changes only the characters it asks the font for. The text the application passed in is untouched, and clusters still point into it.

## Where it runs

Normalization is the second thing `hb_ot_substitute_default` does (`hb-ot-shape.cc:882`), after mirroring and vertical forms (`hb_ot_rotate_chars`) and before the shaper sets its feature masks. By then the text is in the script’s native direction and the shaper’s `preprocess_text` has run. [The pipeline, step by step](the-pipeline-step-by-step.html) shows where this sits in the whole of shaping. In outline (`:882`, trimmed):

```c++
  hb_ot_rotate_chars (c);
  HB_BUFFER_ALLOCATE_VAR (buffer, normalizer_glyph_index);
  _hb_ot_shape_normalize (c->plan, buffer, c->font);
  hb_ot_shape_setup_masks (c);
  [...]
  hb_ot_map_glyphs_fast (buffer);
  HB_BUFFER_DEALLOCATE_VAR (buffer, normalizer_glyph_index);
```

Throughout normalization the buffer holds Unicode: each `codepoint` is a character. Whenever the normalizer asks the font whether it has a character, it keeps the answer in the glyph’s scratch variable **`normalizer_glyph_index`**, the whole of `var1` (`hb-ot-shape-normalize.hh:35`; [Buffers](buffers.html#who-has-which-byte-and-when) has the table of who uses which byte when). After the masks are set, `hb_ot_map_glyphs_fast` copies it into `codepoint` for every glyph and changes the buffer’s content type to glyphs (`hb-ot-shape.cc:844`). There is no second `cmap` lookup: the “mapping” step is a copy.

The questions go to the font funcs `nominal_glyph`, `nominal_glyphs` and `variation_glyph` ([Fonts and font functions](fonts-and-font-functions.html)), which for hb-ot read `cmap`.

The trace shows the three rounds by name. Each “start” message is a chance for a message callback to skip the round ([Buffers](buffers.html#messages)):

```trace
font: norm.ttf
text: ǘ\u0323
from: start decompose
to: end compose
```

The text is *ǘ* and a dot below. The first round splits *ǘ* all the way down. The second puts the dot below (class 220) before the marks above (230). The third composes *u* with the dieresis into *ü*, and stops there: the font has no *ǘ* and no *ụ*.

## The modes

How far HarfBuzz composes is chosen by the script shaper, in its `normalization_preference` ([Choosing a script shaper](choosing-a-script-shaper.html#what-the-choice-decides)). The **normalization modes** are `hb_ot_shape_normalization_mode_t` (`hb-ot-shape-normalize.hh:39`):

| Mode | Used by | Round 1: single characters | Round 1: base and marks | Round 3 |
|---|---|---|---|---|
| `NONE` | Hangul, Zawgyi | kept if the font has them | kept if the font has them | no |
| `DECOMPOSED` | no shaper | decomposed if the font allows | decomposed if the font allows | no |
| `COMPOSED_DIACRITICS` | (through `AUTO`) | kept if the font has them | decomposed if the font allows | yes |
| `COMPOSED_DIACRITICS_NO_SHORT_CIRCUIT` | Indic, Khmer, Myanmar, USE | decomposed if the font allows | decomposed if the font allows | yes |
| `AUTO` (= `DEFAULT`) | default, Arabic, Hebrew, Thai, and `dumber` | as `COMPOSED_DIACRITICS` | as `COMPOSED_DIACRITICS` | yes |

`AUTO` is resolved when normalization starts (`hb-ot-shape-normalize.cc:288`):

```c++
  hb_ot_shape_normalization_mode_t mode = plan->shaper->normalization_preference;
  if (mode == HB_OT_SHAPE_NORMALIZATION_MODE_AUTO)
  {
    if (plan->has_gpos_mark)
      // https://github.com/harfbuzz/harfbuzz/issues/653#issuecomment-423905920
      //mode = HB_OT_SHAPE_NORMALIZATION_MODE_DECOMPOSED;
      mode = HB_OT_SHAPE_NORMALIZATION_MODE_COMPOSED_DIACRITICS;
    else
      mode = HB_OT_SHAPE_NORMALIZATION_MODE_COMPOSED_DIACRITICS;
  }
```

Both branches give `COMPOSED_DIACRITICS`. The code still has the shape of a choice: a font whose plan has a `mark` feature (`has_gpos_mark`, `hb-ot-shape.cc:196`) once got full decomposition, on the theory that a font able to position marks would rather have them separate. The commented-out line and its link to issue #653 record that this was given up. So a description of HarfBuzz that says it decomposes text for fonts with GPOS mark positioning is out of date. `has_gpos_mark` still matters to the Hebrew shaper, below. Nothing in {{hb}} asks for `DECOMPOSED`.

The table’s columns come from two flags worked out from the mode (`:309`):

```c++
  bool always_short_circuit = mode == HB_OT_SHAPE_NORMALIZATION_MODE_NONE;
  bool might_short_circuit = always_short_circuit ||
                             (mode != HB_OT_SHAPE_NORMALIZATION_MODE_DECOMPOSED &&
                              mode != HB_OT_SHAPE_NORMALIZATION_MODE_COMPOSED_DIACRITICS_NO_SHORT_CIRCUIT);
```

To **short-circuit** is to keep a character the font has, without looking at its decomposition. `might_short_circuit` applies to characters with no marks after them; `always_short_circuit` to characters that marks follow too. The names, as the table shows, are misleading in one way: `NONE` doesn’t mean no normalization. It still decomposes a character the font lacks, and still reorders marks. It only never composes.

## Round one: decompose

The first round copies the buffer to its output side, one character or one cluster at a time, deciding for each what to write ([Buffers](buffers.html#the-buffers-three-arrays) explains the input and output arrays).

### Runs without marks

Most text has long stretches with no combining marks at all, and the round treats those first (`:330`). It looks ahead for the next mark, and stops one character short of it, “Leave one base for the marks to cluster with” (`:338`). Everything before that is a run of single characters. In the short-circuiting modes, the whole run goes to the font in one call, `get_nominal_glyphs`, which writes each glyph straight into `normalizer_glyph_index` and stops at the first character the font doesn’t have (`:343`). That character, and the rest of the run, go one at a time through `decompose_current_character`.

If no mark turns up in the whole buffer, the round sets nothing more, and the variable `all_simple` stays true. The second and third rounds are then skipped altogether: there is nothing to reorder or to compose with. For most Latin text, normalization is one batch `cmap` lookup.

### One character

`decompose_current_character` (`:150`) tries these, in order, and takes the first that works:

1. In shortest mode, the character itself, if the font has it.
2. Its decomposition, if the font has every piece (`decompose`, below).
3. Otherwise, the character itself, if the font has it.
4. For a space character (general category Zs), the font’s U+0020 SPACE.
5. For U+2011 NON-BREAKING HYPHEN, the font’s U+2010 HYPHEN.
6. The not-found glyph.

**Shortest** mode is the short circuit: the `shortest` argument is `might_short_circuit` for a lone character, and `always_short_circuit` for one followed by marks. In shortest mode, step 3 can’t be reached, since step 1 asked the same question.

`decompose` (`:108`) is recursive. Unicode’s decomposition data is stored one step at a time, each step one character into one or two (`decompose` in [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#decomposition-and-composition)). For a character *ab* that splits into *a* and *b*:

```c++
  if (!c->decompose (c, ab, &a, &b) ||
      (b && !font->get_nominal_glyph (b, &b_glyph)))
    return 0;

  bool has_a = (bool) font->get_nominal_glyph (a, &a_glyph);
  if (shortest && has_a) {
    /* Output a and b */
    [...]
  }

  if (unsigned ret = decompose (c, shortest, a)) {
    if (b) { output_char (buffer, b, b_glyph); return ret + 1; }
    return ret;
  }

  if (has_a) { /* output a, then b */ [...] }

  return 0;
```

(trimmed). The second character, nearly always a mark, has to be in the font: if it isn’t, the decomposition is refused at this level and no deeper one is tried. The first may be decomposed further. In shortest mode, HarfBuzz stops at the first level where the font has *a*. Otherwise it goes as deep as the font allows, and only falls back on *a* if *a* can’t be split. So *ǘ* in a lone character gives *ü* + acute, the shortest spelling the font supports, while *ǘ* followed by a mark, which isn’t in shortest mode, gives *u* + dieresis + acute, as in the trace above.

The same function handles **singleton** decompositions, where *b* is zero. U+2126 OHM SIGN decomposes to U+03A9 GREEK CAPITAL LETTER OMEGA, and U+212B ANGSTROM SIGN to *Å*. The test font has Omega but no ohm sign:

```shape
norm.ttf  \u2126  --no-positions
```

A decomposition can also need a mark the font lacks. The font has no combining circumflex, so *ê* has nowhere to go, and neither does *ñ*, whose base the font lacks too:

```shape
norm.ttf  ê  --no-positions
norm.ttf  ñ  --no-positions
```

Each stays one character, with no glyph.

### Spaces, and one hyphen

A font with no glyph for a space character still gets a space. For each Zs character that `space_fallback_type` knows (`hb-unicode.hh:219`), the normalizer uses the glyph for U+0020, or the buffer’s invisible glyph if the font has no space either. It writes the kind of space into the character’s properties, in the byte that holds a mark’s combining class, and sets the buffer flag `HAS_SPACE_FALLBACK` (`hb-ot-shape-normalize.cc:174`). The **space fallback** is completed in positioning, where each such space is given a width that suits its kind (`_hb_ot_shape_fallback_spaces`, `hb-ot-shape-fallback.cc:559`): an em for EM SPACE, a fifth of one for THIN SPACE, half the space for NARROW NO-BREAK SPACE, and so on. That part is in [Fallbacks](fallbacks.html).

```shape
norm.ttf  a\u2003a
norm.ttf  a\u2009a
norm.ttf  a\u202Fa
norm.ttf  a\u1680a
```

All four are Zs. The last, U+1680 OGHAM SPACE MARK, is the one Zs character that `space_fallback_type` leaves out, so it gets the not-found glyph. (It is usually drawn as a visible stroke, which may be why.)

U+2011 is a special case of its own, the only character that is “a no-break version of another character and not a space”, as the comment says (`:190`). If the font lacks it, HarfBuzz uses U+2010 HYPHEN, as it has since 1.1.0, according to `NEWS`. The test font has only U+2010:

```shape
norm.ttf  a\u2011a  --no-positions
```

Only the glyph changes: the cluster still points at U+2011.

### Not found

If nothing works, the glyph is the buffer’s **not-found glyph**: `.notdef`, glyph 0, unless the application has set another with `hb_buffer_set_not_found_glyph` (since 3.1.0). The default argument of `get_nominal_glyph` is what puts it there: the call stores its third argument as the answer before asking the font (`hb-font.hh:322`), so a failed lookup leaves it in place. `hb-shape` sets it with `--not-found-glyph`. The test font has an unencoded glyph, `missing`, as glyph 27:

```shape
norm.ttf  x\u0301  --not-found-glyph=27  --no-positions
norm.ttf  x\uFE00  --not-found-glyph=27  --no-positions
```

The second line shows a gap, described next.

### Clusters with marks

After the run of single characters comes a character with marks after it: one base and every mark up to the next non-mark (`:359`). “Mark” here means a general category of Mn, Mc or Me. The cluster goes to `decompose_multi_char_cluster` (`:255`), which hands each character in turn to `decompose_current_character`, now with `shortest` false except in `NONE` mode. A precomposed character followed by marks is therefore decomposed as far as the font allows, and the third round builds it back up. That is what makes *é* followed by a dot below come out the same as *e*, dot below, acute.

If the cluster contains a variation selector, none of this happens. “Currently if there’s a variation-selector we give-up on normalization, it’s just too hard” (`:208`). `handle_variation_selector_cluster` asks the font’s `variation_glyph` func for each base and selector pair, and replaces the two with one glyph if there is one. Otherwise each character gets its nominal glyph, with no decomposition. [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#variation-selectors) covers what then happens to the selector. Here is the cost, with *ö*, which the font has only as *o* and a dieresis:

```shape
norm.ttf  ö  --no-positions
norm.ttf  ö\uFE00  --no-positions
```

With a selector after it, *ö* is not decomposed, and has no glyph. The selector, unmatched, becomes an invisible space glyph.

The gap in the earlier example is here too. This path looks glyphs up with `set_glyph` (`:79`), which calls `get_nominal_glyph` without the not-found argument, so a missing character in a cluster with a variation selector gets glyph 0 whatever the buffer’s not-found glyph is. In the second line above it, `x` came out as `.notdef`, not `missing`. That looks like an oversight in HarfBuzz (inferred: nothing in the source says it is intended).

> Since {{hb}}: fixed on HarfBuzz main by [`97d8794`](https://github.com/harfbuzz/harfbuzz/commit/97d879446fe4de6f09ece771ef645127947f7795) (17 November 2026), so on main the `x` in the second line is `missing`.

## Round two: reorder

Marks of different combining classes can be typed in any order, and all the orders are equivalent. *a* with an acute (class 230) and a dot below (220) is the same text whichever is typed first. To make equivalent strings look the same to the font, the second round sorts each run of marks by class, in place (`:372`):

```c++
    for (unsigned int i = 0; i < count; i++)
    {
      if (_hb_glyph_info_get_modified_combining_class (&info[i]) == 0)
        continue;

      unsigned int end;
      for (end = i + 1; end < count; end++)
        if (_hb_glyph_info_get_modified_combining_class (&info[end]) == 0)
          break;

      /* We are going to do a O(n^2).  Only do this if the sequence is short. */
      if (end - i > HB_OT_SHAPE_MAX_COMBINING_MARKS) {
        i = end;
        continue;
      }

      buffer->sort (i, end, compare_combining_class);

      if (plan->shaper->reorder_marks)
        plan->shaper->reorder_marks (plan, buffer, i, end);

      i = end;
    }
```

A run is a sequence of characters with a non-zero class. The class is HarfBuzz’s modified combining class, Unicode’s class renumbered for some scripts so that the order suits fonts, and zero for anything that isn’t a mark. [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#modified-combining-classes) lists the changes and shows Hebrew and Thai marks being reordered.

The sort is `hb_buffer_t::sort` (`hb-buffer.cc:2239`), an insertion sort, which is stable: marks of the same class keep their order, as Unicode requires, since swapping them would change the meaning. It is also why runs are limited. Insertion sort takes time proportional to the square of the run’s length, so a run longer than `HB_OT_SHAPE_MAX_COMBINING_MARKS`, 32 (`hb-ot-shaper.hh:42`), is left as typed, so that no text can make the sort slow.

Each move merges the clusters of the characters it passes over (`merge_clusters`, [Buffers](buffers.html#merging-clusters)). At the default cluster level, marks are in their base’s cluster already and nothing visible happens. At level 1 they are merged; at level 2 they keep their own clusters, and the glyphs are marked unsafe to break instead:

```shape
norm.ttf  a\u0301\u0323  --no-positions
norm.ttf  a\u0323\u0301  --no-positions
norm.ttf  a\u0301\u0323  --cluster-level=1  --no-positions
norm.ttf  a\u0301\u0323  --cluster-level=2  --no-positions  --show-flags
```

The two orders give the same glyphs. (The acute ends up inside *á* in the next round.)

After sorting a run, the shaper’s `reorder_marks` hook may move marks again. Two shapers have one. The Arabic shaper moves the “modifier combining marks”, such as hamza above, to the front of their run, and renumbers them to keep the run sorted (`hb-ot-shaper-arabic.cc:721`; [Joining scripts](joining-scripts.html)). The Hebrew shaper swaps a sheva or hiriq with a following meteg or below mark when they come after a patah or qamats (`hb-ot-shaper-hebrew.cc:166`; [Hebrew, Thai, Lao and Hangul](hebrew-thai-lao-and-hangul.html)).

A mark of class zero ends a run. The combining grapheme joiner, U+034F, is such a mark, and is how a writer stops two marks being reordered. Straight after this round, HarfBuzz looks at each CGJ to see whether it kept anything apart, and lets later lookups pass over the ones that didn’t (`hb-ot-shape-normalize.cc:403`). That check, and an example, are in [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#cgj-sometimes).

## Round three: recompose

The last round runs only in the two `COMPOSED_DIACRITICS` modes, and only if the first round found a mark somewhere (`!all_simple`). It walks the buffer again, writing to the output, and keeps track of the **starter**: the last character written whose modified class is zero, the one a mark can compose with. For each character it comes to (`:435`, trimmed):

```c++
      if (_hb_glyph_info_is_unicode_mark(&buffer->cur()))
      {
        if (/* If there's anything between the starter and this char, they should have CCC
             * smaller than this character's. */
            (starter == buffer->out_len - 1 ||
             info_cc (buffer->prev()) < info_cc (buffer->cur())) &&
            /* And compose. */
            c.compose (&c,
                       buffer->out_info[starter].codepoint,
                       buffer->cur().codepoint,
                       &composed) &&
            /* And the font has glyph for the composite. */
            font->get_nominal_glyph (composed, &glyph))
        {
          /* Composes. */
          [...]
          buffer->merge_out_clusters (starter, buffer->out_len);
          buffer->out_len--; /* Remove the second composable. */
          /* Modify starter and carry on. */
          buffer->out_info[starter].codepoint = composed;
          buffer->out_info[starter].normalizer_glyph_index() = glyph;
          [...]
          continue;
        }
      }

      /* Blocked, or doesn't compose. */
      if (unlikely (!buffer->next_glyph ())) break;

      if (info_cc (buffer->prev()) == 0)
        starter = buffer->out_len - 1;
```

So a character is composed into the starter only if all of these hold:

* **It is a mark.** Two base characters are never composed, “both an optimization to avoid trying to compose every two neighboring glyphs in most scripts AND a desired feature for Hangul”, whose fonts “are not designed to mix-and-match pre-composed syllables and Jamo” (`:438`). The Hangul shaper does its own composing, before normalization ([Hebrew, Thai, Lao and Hangul](hebrew-thai-lao-and-hangul.html)).
* **It isn’t blocked.** Either it comes straight after the starter, or the mark before it has a lower class. A mark with the same class or a higher one in between is **blocked**: Unicode’s rule, applied with the modified classes.
* **The pair composes.** By Unicode’s composition data, or by the shaper’s `compose` hook.
* **The font has the result.**

When all hold, the composite replaces the starter, with its glyph, and the mark is dropped. The new starter can compose again with a later mark. *ǜ* is *ü* plus a grave, and the font has both *ü* and *ǜ*:

```shape
norm.ttf  u\u0308\u0300  --no-positions
norm.ttf  u\u0308\u0301  --no-positions
```

The first composes twice. In the second, *u* and the dieresis compose, but *ü* and the acute would make *ǘ*, which the font lacks, so the acute stays a mark: a **partial composition**, the same result the first section reached by decomposing *ǘ*.

Blocking, with a mark the font can’t compose:

```shape
norm.ttf  a\u0300\u0301  --no-positions
norm.ttf  a\u0301\u0300  --no-positions
norm.ttf  a\u0323\u0301  --no-positions
```

In the first line, *a* and the grave can’t compose, since the font has no *à*, and the grave, of the same class 230, then blocks the acute from reaching the *a*. In the second, the acute comes first and composes. The two lines are different text, so a different result is right. In the third, the dot below has a lower class than the acute, so it doesn’t block it: *á* is formed past it, and the dot below follows. This was also the result of the reordering examples above.

The third goal in the design comment, using a precomposed glyph for a mark the font lacks, comes for free:

```shape
norm.ttf  a\u0302  --no-positions
norm.ttf  e\u0302  --no-positions
```

The font has no combining circumflex. It has *â*, so *a* + circumflex composes; it has no *ê*, so the circumflex after *e* has no glyph.

The composition is greedy, and needs every step to exist in the font. *ȫ* is *ö* plus a macron. The font has *ȫ* but not *ö*:

```shape
norm.ttf  ȫ  --no-positions
norm.ttf  o\u0308\u0304  --no-positions
```

Typed precomposed, *ȫ* has its glyph. Typed as *o*, dieresis, macron, it doesn’t: *o* and the dieresis would make *ö*, which the font lacks, so they stay apart, and the dieresis then blocks the macron. This is the case the design comment means when it says that rendering all equivalent strings alike would need a search HarfBuzz doesn’t do.

> For type designers: if a font has a precomposed glyph for a letter with two marks, it should also have the letter with the first of them. Otherwise, text typed with combining marks won’t reach the precomposed glyph.

### Clusters when composing

Composing merges the clusters of the starter and the mark, and of everything between them (`merge_out_clusters`). At level 2, `merge_out_clusters` does nothing (`hb-buffer.hh:419`), and the mark’s cluster number disappears along with the mark, as the level 2 line in the reordering example shows: the *á* has cluster 0, and the dot below cluster 2, but no glyph has cluster 1. Decomposition, the other way, gives every piece the cluster of the character it came from.

### The shapers’ compose and decompose

A shaper can replace both of the Unicode funcs’ functions used here, with its `compose` and `decompose` hooks (`hb-ot-shape-normalize.cc:299`). What they change:

* **Indic.** The shaper (`hb-ot-shaper-indic.cc:1473`) doesn’t decompose four letters: DEVANAGARI LETTER RRA, the Bengali RRA and RHA (issue #779), and TAMIL LETTER AU. Its `compose` refuses any pair whose first character is a mark, so that the parts of a split vowel sign aren’t joined back together once the first round has split it. It also composes one pair that Unicode excludes, Bengali YA + NUKTA into YYA.
* **Khmer.** The shaper (`hb-ot-shaper-khmer.cc:326`) gives five two-part vowel signs, which Unicode doesn’t decompose, a decomposition of its own: U+17C1 KHMER VOWEL SIGN E, the part written before the consonant, followed by the sign itself. Its `compose` has Indic’s rule about marks.
* **USE.** Its `compose` (`hb-ot-shaper-use.cc:482`) has the same rule about marks.
* **Hebrew.** The shaper composes the presentation forms that Unicode’s composition excludes, such as YOD + HIRIQ into U+FB1D, for “old fonts”: only when Unicode composition fails, and only if the plan has no `mark` feature (`hb-ot-shaper-hebrew.cc:81`). That is the one use of `has_gpos_mark` left in normalization. `HB_NO_OT_SHAPER_HEBREW_FALLBACK` leaves it out.

The script chapters, [The Indic model](the-indic-model.html), [Khmer and Myanmar](khmer-and-myanmar.html) and [The Universal Shaping Engine](the-universal-shaping-engine.html), say why each script needs them.

## The short circuit, and its price

The design comment admits that the short circuit is a compromise: “we simply don’t touch 1-character clusters that are supported by the font, even though their NFC may be different”. The angstrom sign shows it. Its NFC is *Å*, and the test font has both:

```shape
norm.ttf  \u212B  --no-positions
norm.ttf  \u212B  --script=deva  --no-positions
norm.ttf  \u212Ba\u0301  --script=deva  --no-positions
```

The default shaper keeps the font’s angstrom glyph. The second and third lines force the Indic shaper, which uses `COMPOSED_DIACRITICS_NO_SHORT_CIRCUIT`, on the same text. In the third line, the angstrom sign is decomposed to *A* and ring above, and composed back into *Å*, which is what that mode is for.

The second line shows something else. The angstrom sign is decomposed into *A* and ring, and left that way. Recomposing is skipped when `all_simple` is true, that is, when the *input* had no marks, but in this mode the first round can make marks out of characters that had none. A buffer with a single *é*, shaped by any of the four shapers that use the mode, gets *e* and an acute; add a combining mark anywhere else in the buffer, and the same *é* comes out precomposed. With real Devanagari:

```shape
norm.ttf  ऩ  --no-positions
norm.ttf  ऩ\u093E  --no-positions
```

NNNA, which the font has, comes out as NA and NUKTA when it is alone, and as NNNA when a vowel sign (a mark, category Mc) follows it. The difference is seldom visible, because Indic fonts usually compose the nukta forms themselves, with `nukt`, and because Indic text rarely goes long without a mark. It still looks unintended (inferred: the source gives no reason for it), since the mode’s own comment promises “Always fully decomposes and then recompose back” (`hb-ot-shape-normalize.hh:43`).

> Since {{hb}}: fixed on HarfBuzz main by [`bc7dff9`](https://github.com/harfbuzz/harfbuzz/commit/bc7dff9a51e900d25d35ee1a903ab496caf0feae) (17 November 2026), so main recomposes what the first round decomposed: the second line gives *Å*, and NNNA alone stays NNNA.

The `NONE` mode, in contrast, does short-circuit everywhere and never composes. Zawgyi, shaped as script `Qaag`, uses it ([Choosing a script shaper](choosing-a-script-shaper.html#myanmar-and-zawgyi)):

```shape
norm.ttf  a\u0301  --script=Qaag  --no-positions
norm.ttf  ǘ  --script=Qaag  --no-positions
```

*a* and the acute stay apart, though the font has *á*. *ǘ*, which the font lacks, is still decomposed.

## What normalization doesn’t do

* **Compatibility decompositions.** Only canonical ones are used. A font with no glyph for *ﬁ* (U+FB01) gets `.notdef`, not *f i*. The Unicode funcs once had a `decompose_compatibility` callback; HarfBuzz stopped using it in 1.1.0 and deprecated it in 2.0.0 (`NEWS`).
* **Composition for its own sake.** Nothing is composed unless the font has the result, and lone characters the font has are left alone even when NFC would change them.
* **Closure.** `hb_ot_shape_glyphs_closure`, which works out the glyphs a piece of text can use, maps each character with `nominal_glyph` and its mirror and doesn’t decompose. A comment above it says so: “TODO Move this to hb-ot-shape-normalize, make it do decompose, and make it public” (`hb-ot-shape.cc:1268`). So for *ǘ* in the test font, it would find no glyph at all, where shaping uses *ü* and the acute (inferred from the code; not run here).

Fonts shaped with `morx` are normalized like any other, by the `dumber` shaper, in `AUTO` mode, before `morx` runs.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-ot-shape-normalize.cc` | the design comment, the three rounds, `decompose`, `decompose_current_character`, the variation-selector clusters, the CGJ check |
| `src/hb-ot-shape-normalize.hh` | the modes, `normalizer_glyph_index`, the context with the `compose` and `decompose` hooks |
| `src/hb-ot-shape.cc` | `hb_ot_substitute_default`, which calls normalization; `hb_ot_map_glyphs_fast`; `has_gpos_mark` |
| `src/hb-ot-shaper.hh` | the shaper hooks and `normalization_preference`; `HB_OT_SHAPE_MAX_COMBINING_MARKS` |
| `src/hb-ot-shaper-{indic,khmer,use,hebrew,arabic}.cc` | the shapers’ `compose`, `decompose` and `reorder_marks` |
| `src/hb-unicode.hh` | `space_fallback_type`, `is_variation_selector`, the modified combining classes |
| `src/hb-ot-shape-fallback.cc` | `_hb_ot_shape_fallback_spaces`, which gives fallback spaces their widths |
| `src/hb-buffer.cc` | `sort`, the stable sort that reorders marks |

## In short

* HarfBuzz normalizes against the font: a character is decomposed only if the font has the pieces, and composed only if the font has the result. The result is neither NFC nor NFD.
* This is where the `cmap` lookup happens. Its answers are kept in `normalizer_glyph_index`, and `hb_ot_map_glyphs_fast` copies them into the buffer.
* There are three rounds. Decompose, recursively and as deep as needed, stopping early for lone characters the font has. Reorder runs of up to 32 marks by modified combining class. Recompose marks into the last starter, unless blocked by a mark of the same or higher class.
* `AUTO` always means `COMPOSED_DIACRITICS` now. The Indic family and USE decompose everything first; Hangul and Zawgyi never compose.
* Missing spaces become the space glyph, with a width set later; a missing U+2011 becomes U+2010; anything else missing becomes the not-found glyph. A cluster with a variation selector isn’t normalized at all.
