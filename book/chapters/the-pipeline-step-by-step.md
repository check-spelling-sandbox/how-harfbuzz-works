# The pipeline, step by step

When `hb_shape` reaches the `ot` backend, the expensive decisions have been made. The plan holds the map of features and lookups, the script shaper and a few dozen settings, all worked out once for this font, script, language and direction ([Shape plans and their cache](shape-plans-and-their-cache.html)). What is left is to run it: to take one buffer of characters and turn it into positioned glyphs.

That is the job of one short function, `hb_ot_shape_internal` (`hb-ot-shape.cc:1171`). Everything the `ot` shaper does to a buffer happens inside it, in a fixed order, and most of the rest of this part of the book is about one step or another of it. This chapter goes through the whole function once, in order: what each step does, why it comes where it does, which fields of the buffer it reads and writes, and where the step is explained in depth.

It follows one run all the way, using `hb-shape --trace` ([The command-line tools](command-line-tools.html#traces)). The test font, `pipeline.ttf`, is built from `fonts/the-pipeline-step-by-step/pipeline.fea`. It has Arabic beh, lam and alef in their joining forms, the lam-alef ligature, a fatha that GPOS attaches to beh, a pair of parentheses, and the digits and fraction slash for an automatic fraction. Its glyphs are boxes. Every result and trace on this page was produced by running HarfBuzz {{hb}} on it when the page was built.

## The whole function

Here it is, with its comments left out:

```c++
static void
hb_ot_shape_internal (hb_ot_shape_context_t *c)
{
  c->target_direction = c->buffer->props.direction;

  _hb_buffer_allocate_unicode_vars (c->buffer);

  hb_set_unicode_props (c->buffer, c->plan->map.get_global_mask ());
  hb_insert_dotted_circle (c->buffer, c->font);

  hb_form_clusters (c->buffer);

  hb_ensure_native_direction (c->buffer);

  if (c->plan->shaper->preprocess_text &&
      c->buffer->message(c->font, "start preprocess-text"))
  {
    c->plan->shaper->preprocess_text (c->plan, c->buffer, c->font);
    (void) c->buffer->message(c->font, "end preprocess-text");
  }

  hb_ot_substitute_pre (c);
  hb_ot_position (c);
  hb_ot_substitute_post (c);

  hb_propagate_flags (c->buffer);

  _hb_buffer_deallocate_unicode_vars (c->buffer);

  c->buffer->props.direction = c->target_direction;
}
```

It falls into five parts. The first five calls prepare the characters: they give each one its Unicode properties, group them into clusters and put them in the order the script is written in. `hb_ot_substitute_pre` turns characters into glyphs and runs GSUB. `hb_ot_position` gives the glyphs advances and runs GPOS. `hb_ot_substitute_post`, despite its name, comes *after* positioning: it is where invisible characters are finally hidden. And the last three lines tidy up.

The `hb_ot_shape_context_t` that every step is passed holds the plan, the font, its face, the buffer and the user’s features, and one field of its own, `target_direction`: the direction the caller asked for, saved because the buffer’s direction may be flipped while shaping and is put back at the end.

## The run

The text is an opening parenthesis, a zero-width joiner, beh with a fatha, lam and alef, and a closing parenthesis, shaped as Arabic, right to left:

```shape
pipeline.ttf  (\u200Dب\u064Eلا)  --language=ar
```

Read the result from the right, because `hb-shape` lists glyphs in visual order. The opening parenthesis comes out as the `parenright` glyph, mirrored. The joiner has become a `space` with no advance. Beh is medial, because the joiner before it asks it to join, and the fatha sits on it with no advance of its own. Lam and alef have become one glyph, `lam_alef.fina`. Every one of those changes was made by a different step. (The font gives the fatha an advance of 200, which a real font wouldn’t, so that one of the steps has something to do. It has no glyph for the joiner.)

## The steps at a glance

Line numbers are in `hb-ot-shape.cc` unless another file is named; where there are two, the first is the call and the second the function.

| | Step | Source | What it changes | In depth |
|---|---|---|---|---|
| 1 | save the direction; allocate `unicode_props` | :1174, :1176 | `target_direction`; bytes 4–5 of each glyph | [Buffers](buffers.html#who-has-which-byte-and-when) |
| 2 | `hb_set_unicode_props` | :1178, :471 | `mask` (to the global mask), `unicode_props`, scratch flags | [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#the-properties-each-glyph-carries) |
| 3 | `hb_insert_dotted_circle` | :1179, :552 | may add one character | [Dotted circles and broken clusters](dotted-circles-and-broken-clusters.html) |
| 4 | `hb_form_clusters` | :1181, :581 | `cluster` | [Buffers](buffers.html#merging-clusters) |
| 5 | `hb_ensure_native_direction` | :1183, :591 | order; `props.direction` | below |
| 6 | shaper’s `preprocess_text` | :1185 | characters | the script chapters |
| 7 | `hb_ot_rotate_chars` | :886, :655 | `codepoint` (mirrors, vertical forms); `rtlm` mask bit | [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html), [Vertical text](vertical-text.html) |
| 8 | normalization, with the `cmap` lookup | :890, `hb-ot-shape-normalize.cc:280` | characters; `normalizer_glyph_index` (bytes 0–3) | [Normalization](normalization.html) |
| 9 | `hb_ot_shape_setup_masks` | :892, :750 | `mask`: fractions, shaper, ranged user features | [From features to lookups](from-features-to-lookups.html#mask-bits) |
| 10 | recategorize marks, if fallback positioning will run | :896 | combining classes in `unicode_props` | [Fallbacks](fallbacks.html) |
| 11 | `hb_ot_map_glyphs_fast` | :898, :844 | `codepoint` becomes a glyph ID; content type | [Buffers](buffers.html#content-types) |
| 12 | allocate `glyph_props`, `lig_props`; GDEF classes and digest, or synthesized classes | :940, :926, :929 | bytes 0–2; the buffer’s digest | [GSUB](gsub.html#glyph-properties) |
| 13 | GSUB, with the shaper’s pauses; or `morx` | :931, :916 | glyphs, clusters | [GSUB](gsub.html), [AAT inside HarfBuzz](aat-inside-harfbuzz.html) |
| 14 | `clear_positions` | :1096 | `pos` becomes positions | [Buffers](buffers.html#the-buffers-three-arrays) |
| 15 | default advances; origins; space widths | :1098, :1002 | advances, offsets | [Where metrics come from](where-metrics-come-from.html), [Fallbacks](fallbacks.html) |
| 16 | `position_start`; zero marks early | :1049, :1055 | `attach_chain`, `attach_type`; advances | [GPOS: positioning glyphs](positioning-glyphs-gpos.html) |
| 17 | GPOS, or `kerx`; `kern` or fallback kerning; `trak` | :1064, :271 | advances, offsets | GPOS, [Legacy kern](legacy-kern.html), [AAT positioning](aat-positioning.html) |
| 18 | zero marks late | :1070 | advances | GPOS |
| 19 | finish advances; zero default ignorables; finish offsets | :1080–:1082 | advances, offsets | GPOS, [Fonts and font functions](fonts-and-font-functions.html#slant) |
| 20 | fallback mark positioning | :1089 | offsets | [Fallbacks](fallbacks.html) |
| 21 | reverse into visual order; free bytes 0–2 | :1103, :1105 | order | below |
| 22 | remove glyphs `morx` deleted | :946 or :955 | glyphs | [AAT inside HarfBuzz](aat-inside-harfbuzz.html) |
| 23 | variation selectors not found | :958, :795 | `codepoint` | [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#variation-selectors) |
| 24 | `hb_ot_hide_default_ignorables` | :959, :817 | `codepoint`, or removes glyphs | [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#hidden-preserved-or-removed) |
| 25 | shaper’s `postprocess_glyphs` | :961 | anything | the script chapters |
| 26 | `hb_propagate_flags` | :1196, :1109 | glyph flags in `mask` | [Buffers](buffers.html#glyph-flags) |
| 27 | free `unicode_props`; restore the direction | :1198, :1200 | `props.direction` | |

Only some steps send a message to `--trace`: preprocess-text, the three rounds of normalization, GSUB and GPOS and their lookups, `morx` and `kerx`, legacy `kern`, fallback kerning and mark positioning, and postprocess-glyphs. The others show only in what the buffer looks like at the next message.

## Preparing the characters

### Properties and clusters

Steps 1 to 4 have no messages, so the first line of the trace already shows their work:

```trace
font: pipeline.ttf
text: (\u200Dب\u064Eلا)
options: --language=ar
from: start decompose
to: start decompose
```

`hb-shape` numbers clusters by character, so the seven characters arrived as clusters 0 to 6. Now the joiner is in cluster 0, with the parenthesis, and the fatha is in cluster 2, with the beh.

That is step 4. Step 2, `hb_set_unicode_props`, set every glyph’s mask to the plan’s global mask, worked out its Unicode properties, and marked some characters as **continuations** of the one before: marks, and also a zero-width joiner, which belongs with what precedes it, as in emoji sequences. It also set **scratch flags** on the buffer, one-bit notes such as “has continuations”, “has default ignorables” and “has a fraction slash”, which let later steps skip themselves when there is nothing to do. [Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#continuations-just-enough-of-graphemes) has the details of both. Step 4, `hb_form_clusters`, then merges each base with its continuations into one cluster, with `merge_grapheme_clusters`. It only merges at the cluster levels that work by graphemes, 0 (the default) and 3. At level 1 every character keeps its own cluster:

```shape
pipeline.ttf  (\u200Dب\u064Eلا)  --language=ar  --cluster-level=1
```

Step 3 inserts a dotted circle before a mark that begins the text. Only when the buffer says it is the beginning of the text, with no pre-context, and only if the font has U+25CC. This run doesn’t qualify on any count.

### Native direction

Step 5, `hb_ensure_native_direction`, compares the buffer’s direction with the script’s own. If a script written right to left is given as left to right, or the other way round, it reverses the buffer and flips `props.direction`, so that everything after it sees the text in the order the script is read. It reverses by graphemes, using the continuation bits, so a mark stays after its base. Vertical text always becomes top to bottom.

Our run is Arabic, right to left, so nothing happens. Given as left to right, the same kind of text is taken to be in visual order, and reversed:

```trace
font: pipeline.ttf
text: ب\u064Eل
options: --language=ar --direction=ltr
from: start decompose
to: start decompose
```

```shape
pipeline.ttf  ب\u064Eل  --language=ar  --direction=ltr
pipeline.ttf  لب\u064E  --language=ar
```

Lam comes first, with beh and its fatha after it, and lam takes the initial form. The glyphs and positions are the same as for lam, beh, fatha shaped right to left; only the cluster numbers differ, because they count the characters as they were given. (A left-to-right run in a right-to-left script is left alone if it has digits or regional indicators but no letters, so that numbers keep their order: HarfBuzz issues #501 and #3314.)

Why reverse at all, rather than let every lookup run backwards? Because fonts are written for the script’s own order: an Arabic font’s contextual rules say what comes *after* a letter in reading order. From here until step 21, HarfBuzz works in that order.

### Preprocessing

Step 6 is the first of the script shaper’s hooks to run on the text. `preprocess_text` changes the characters before anything else looks at them: Thai and Lao split SARA AM, Hangul composes and decomposes jamo, and the Indic shaper and USE insert dotted circles where vowel signs mustn’t follow each other. The Arabic shaper has no such hook, so this run has no “start preprocess-text” in its trace.

## Characters to glyphs

`hb_ot_substitute_pre` (`:936`) first calls `hb_ot_substitute_default` (`:882`), steps 7 to 11, which end with a buffer of glyphs.

### Mirroring

Step 7, `hb_ot_rotate_chars`, is why the trace above starts with U+0029 where the text has U+0028. When the *target* direction is backward, each character with a Bidi_Mirroring_Glyph is replaced by its mirror, if the font has a glyph for it. Every other character gets the mask bit of `rtlm`, the feature a font can use to mirror glyphs itself. It reads `target_direction`, not the buffer’s direction: what matters is how the text will be displayed, so the left-to-right run above, reversed into right to left, is not mirrored. In vertical text, if the plan found no `vert` feature, characters are replaced by their vertical presentation forms instead (U+FE10 and the rest, `vertical_char_for`), a subject for [Vertical text](vertical-text.html).

Mirroring is done on characters, before the `cmap` is consulted, so a font needs only the two glyphs it has anyway.

### Normalization

Step 8 claims all of `var1` for `normalizer_glyph_index` and runs normalization: decompose, reorder marks, recompose. Normalization is where the `cmap` is read: whether a character decomposes depends on which glyphs the font has, so each character’s glyph is looked up there and kept in `normalizer_glyph_index`. The shaper’s `decompose`, `compose` and `reorder_marks` hooks are called from inside it. All of that is the subject of [Normalization](normalization.html). In this run nothing changes:

```trace
font: pipeline.ttf
text: (\u200Dب\u064Eلا)
options: --language=ar
from: start decompose
to: end compose
```

### Masks

Step 9, `hb_ot_shape_setup_masks`, sets the feature bits of each glyph’s mask that aren’t simply on everywhere. Mask bits were handed out when the plan was made ([From features to lookups](from-features-to-lookups.html#mask-bits)); every glyph has had the global ones since step 2. Three things add more, in this order:

1. **Fractions.** If step 2 saw a U+2044 FRACTION SLASH and the font has `frac`, or both `numr` and `dnom`, each run of digits before a fraction slash gets `numr` (and `frac`), the slash `frac`, and the digits after it `dnom` (and `frac`). The range is marked unsafe to break.
2. **The shaper’s `setup_masks`.** The Arabic shaper works out each letter’s joining form here and sets the bit for `init`, `medi`, `fina` or `isol`. It is why the beh in our run will be medial: the joiner before it is join-causing.
3. **User features with a range**, such as `--features=dnom[2:3]`. Their start and end are compared with cluster values, not glyph positions.

The fraction needs no feature to be asked for:

```shape
pipeline.ttf  1⁄2  --language=en
pipeline.ttf  1/2  --language=en
```

Masks are set on characters, before any glyph exists, because they are about the text: which character starts a word, which digit is a numerator. Lookups that change the glyphs later keep the masks of the glyphs they replace.

### Glyphs at last

Step 10 only runs when the plan will position marks without GPOS or `kerx`: it changes combining classes into the few positional classes the fallback positioner understands ([Fallbacks](fallbacks.html)). It has to come after normalization, which needs the real classes.

Step 11, `hb_ot_map_glyphs_fast`, copies each `normalizer_glyph_index` into `codepoint`, sets the content type to glyphs, and gives `var1` back. The next line of the trace prints square brackets instead of angle brackets: these are glyph names now, and the joiner, which the font has no glyph for, is `.notdef`.

## Substituting

Step 12 claims three of the bytes `normalizer_glyph_index` just gave back, for `glyph_props` and `lig_props` (`_hb_buffer_allocate_gsubgpos_vars`, `:940`). `hb_ot_layout_substitute_start_with_digest` (`hb-ot-layout.cc:1631`) reads each glyph’s class from GDEF into `glyph_props`, clears `lig_props`, and builds the buffer’s digest of glyph IDs, which lets GSUB skip lookups that can’t match. If the font has no GDEF glyph classes, they are made up from the Unicode general categories instead ([GSUB](gsub.html#glyph-properties)).

Step 13 is GSUB itself, `hb_ot_map_t::substitute`, which runs the plan’s GSUB lookups stage by stage and calls the shaper’s pause functions between stages ([From features to lookups](from-features-to-lookups.html#stages-and-pauses)):

```trace
font: pipeline.ttf
text: (\u200Dب\u064Eلا)
options: --language=ar
from: start table GSUB
to: end table GSUB
```

The order is the Arabic shaper’s, not the font’s: `fina` (lookup 2), `medi` (1), `init` (0), each in a stage of its own, then `rlig` (3). The `init` lookup runs, because its glyphs are in the buffer, but changes nothing: the masks set in step 9 gave its bit to no glyph. `rlig` makes the ligature, passing over nothing here, and the ligature takes cluster 4. How each lookup runs is the [GSUB chapter](gsub.html)’s subject.

If the plan uses `morx` instead, step 13 is `hb_aat_layout_substitute`, and GSUB isn’t used at all ([AAT inside HarfBuzz](aat-inside-harfbuzz.html)). Glyphs that `morx` deletes are only marked as deleted, and removed later: here, if GPOS is to follow, or after positioning if not (step 22).

## Positioning

`hb_ot_position` (`:1094`) holds steps 14 to 21.

### Advances

Step 14, `clear_positions`, zeroes the positions array. Until now it was free for GSUB to use as its output buffer ([Buffers](buffers.html#the-buffers-three-arrays)); from now on it holds positions.

Step 15, `hb_ot_position_default`, asks the font funcs for every glyph’s advance, in one batch call, from `hmtx` with the default funcs ([Where metrics come from](where-metrics-come-from.html)). In horizontal text, it subtracts each glyph’s horizontal origin only if the font funcs provide origins, which the default funcs don’t. In vertical text it fetches vertical advances, and always subtracts the vertical origins, because those default to something other than zero. Last, if normalization mapped a space character that the font lacks to the space glyph, it gives that glyph the right width now: an em-space an em, a thin space a fifth of one, and so on ([Fallbacks](fallbacks.html)).

The trace shows the advances at the start of GPOS. The joiner’s `.notdef` has the full 500 units of `.notdef`, and the fatha has its 200:

```trace
font: pipeline.ttf
text: (\u200Dب\u064Eلا)
options: --language=ar
from: start table GPOS
to: end table GPOS
```

### GPOS, and what surrounds it

`hb_ot_position_plan` (`:1029`) runs GPOS with a sequence of steps around it whose order is fixed:

* If the font funcs provide horizontal origins, they are added back, because GPOS works from the horizontal origin.
* Step 16: `GPOS::position_start` clears each glyph’s `attach_chain` and `attach_type`, which GPOS keeps in the spare field of the positions. If the shaper zeroes marks “early”, before GPOS, it sets every GDEF mark’s advance to zero now. USE and the Myanmar shaper do.
* Step 17: `plan->position` (`:271`) runs GPOS, or `kerx`. Then, if the plan says so, legacy `kern` ([Legacy kern](legacy-kern.html)) or fallback kerning, and `trak` ([AAT positioning](aat-positioning.html)). Which of them the plan uses was decided when it was made, from the tables the font has. Here, the fatha is attached to beh at offset (100, 50).
* Step 18: shapers that zero marks “late” do it now. The default, Arabic, Hebrew and Thai shapers are among them; the Indic, Khmer and Hangul shapers don’t zero marks at all. If the plan has neither GPOS nor `kerx` and the run is forward, the mark’s offset is moved back by its old advance at the same time, so that it hangs over the glyph before it.
* Step 19: `position_finish_advances`, which does nothing in {{hb}}; then `hb_ot_zero_width_default_ignorables`, which gives every default ignorable no advance and no offset along the line; then `position_finish_offsets`, which resolves attachments. GPOS stores a mark’s offset relative to its base, and this adds up the base’s offset and the advances between them. Then, for a synthetically slanted font, each glyph with a vertical offset is moved sideways to match ([Fonts and font functions](fonts-and-font-functions.html#slant)).
* Horizontal origins are subtracted again, if there are any.
* Step 20: if the plan has neither GPOS nor `kerx`, and the shaper allows it, marks are placed from glyph extents (“start fallback mark” in a trace; [Fallbacks](fallbacks.html)).

The order in step 19 is the one the source comment insists on: “Has to follow a certain order.” Resolving an attachment adds up advances, so the advances must be final first. In a right-to-left run, the advances added to a mark’s offset include the mark’s own (`OT/Layout/GPOS/GPOS.hh:126`). The fatha’s 200 was zeroed in step 18, so its offset stays at 100. In the other order it would have moved 200 units (read from the code).

### Into visual order

Step 21: if the buffer’s direction is now backward, `hb_buffer_reverse` reverses the whole buffer. In practice that means right to left: a bottom-to-top run was turned top to bottom at step 5, so it isn’t backward anymore, and is never reversed back. Its glyphs come out in top-to-bottom order ([Vertical text](vertical-text.html)). This is the only place HarfBuzz turns its output into visual order, and it is why `hb-shape` lists the parenthesis that was typed first last. The reversal is of glyphs, not graphemes: the fatha, typed after the beh, now comes before it, as it would be drawn in a left-to-right list. `glyph_props` and `lig_props` are given back.

## After positioning

`hb_ot_substitute_post` (`:951`) changes glyph IDs, which is why it is called “substitute”, but runs after positioning. Each of its steps needs positioning to have happened first.

Step 22 removes glyphs that `morx` deleted, if they weren’t removed before GPOS. Step 23 replaces any variation selector that the `cmap` didn’t consume with the buffer’s `not_found_variation_selector` glyph, if the application set one, with no advance or offset ([Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#variation-selectors)).

Step 24, `hb_ot_hide_default_ignorables`, replaces each default ignorable with the buffer’s invisible glyph, or the font’s space glyph, or deletes it if there is neither, or if the buffer asks for that. Until now the joiner was an ordinary glyph, `.notdef`, which GSUB and GPOS could see and match, and had an advance until step 19. Hiding it this late is what lets invisible characters take part in shaping ([Unicode inside HarfBuzz](unicode-inside-harfbuzz.html#default-ignorables)).

Step 25 is the shaper’s `postprocess_glyphs`, the last of its hooks. For Arabic it stretches glyphs for `stch`. Its message is the last in the trace, and shows the buffer as it will be returned, in visual order, with the joiner hidden and the fatha’s advance gone:

```trace
font: pipeline.ttf
text: (\u200Dب\u064Eلا)
options: --language=ar
from: start postprocess-glyphs
to: end postprocess-glyphs
```

## Finishing

Step 26, `hb_propagate_flags`, makes the glyph flags the same on every glyph of a cluster, by OR-ing them together. It also clears `UNSAFE_TO_CONCAT` unless the buffer asked for it, and reconciles `SAFE_TO_INSERT_TATWEEL` with `UNSAFE_TO_BREAK` ([Buffers](buffers.html#glyph-flags)). The flags were set, glyph by glyph, by the steps that looked across a boundary between glyphs. After this, an application can read any glyph of a cluster and get the same answer:

```shape
pipeline.ttf  (\u200Dب\u064Eلا)  --language=ar  --show-flags
```

The `#1` is `UNSAFE_TO_BREAK`. Step 27 gives back `unicode_props`, the last of the scratch bytes, and puts back the direction the caller asked for, which step 5 may have flipped.

## Where the shaper’s hooks run

[Choosing a script shaper](choosing-a-script-shaper.html#what-the-choice-decides) lists a shaper’s hooks and settings. This is where each takes effect:

| Hook or setting | Where | Step |
|---|---|---|
| `collect_features`, `override_features`, `gpos_tag` | planning, `hb_ot_shape_collect_features` and `compile` | |
| `data_create` | the end of planning (`:240`) | |
| `preprocess_text` | after the native direction (`:1185`) | 6 |
| `decompose`, `compose`, `reorder_marks`, `normalization_preference` | inside normalization | 8 |
| `setup_masks` | after the fraction masks, before ranged user features (`:758`) | 9 |
| pause functions | between GSUB stages | 13 |
| `zero_width_marks` | before or after GPOS | 16, 18 |
| `fallback_position` | whether steps 10 and 20, and legacy `kern`, may happen | 10, 17, 20 |
| `postprocess_glyphs` | after hiding default ignorables (`:961`) | 25 |

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-ot-shape.cc` | `hb_ot_shape_internal` and every step it calls; the planner decisions (`compile`) that switch steps on and off |
| `src/hb-ot-shape-normalize.cc` | normalization and the `cmap` lookup |
| `src/hb-ot-layout.cc` | glyph properties and the digest, `hb_ot_map_t::apply`, GSUB and GPOS entry points |
| `src/OT/Layout/GPOS/GPOS.hh` | `position_start`, `position_finish_offsets` and `propagate_attachment_offsets` |
| `src/hb-ot-shape-fallback.cc` | fallback mark positioning, kerning and space widths |
| `src/hb-ot-shaper.hh` | the shaper hooks |
| `src/hb-ot-shaper-arabic.cc` | the Arabic shaper this run used |

## In short

* `hb_ot_shape_internal` prepares the characters, maps and substitutes them, positions the glyphs, hides the invisible ones, and propagates flags, in that order.
* The text is put into the script’s native order at the start, and into visual order only after positioning.
* Mirroring and masks are worked out on characters, before the `cmap` is read; the `cmap` is read during normalization.
* GSUB sees glyphs without positions; positions start from the font’s advances, then GPOS, then the zeroing of marks and default ignorables, then attachments.
* Default ignorables are ordinary glyphs until after positioning, so lookups can match them.
* The shaper’s hooks run at fixed points: preprocess before normalization, masks after it, pauses inside GSUB, postprocess at the end.
