# Unicode inside HarfBuzz

A font knows about glyphs. Unicode knows about characters. Before HarfBuzz can apply a single lookup, it needs to know a few things about every character in the buffer that no font says: whether it is a letter or a mark, how strongly a mark binds, whether it has a mirror image, which script it belongs to, and how it decomposes. It also needs to know which characters are meant to be invisible, and which of those still have work to do.

This chapter is about where those facts come from and where HarfBuzz keeps them. They come from a small table of callbacks, the **Unicode funcs**, which by default read HarfBuzz’s own compressed copy of the Unicode Character Database. They are kept in sixteen bits of each glyph, the **Unicode properties**, worked out once at the start of shaping. And a few properties aren’t taken from Unicode at all, but from lists written by hand in HarfBuzz’s source. The most important is HarfBuzz’s own idea of a *default ignorable* character, and of the few ignorables that are *hidden*.

The examples use one small test font, `unicode.ttf`, built from `fonts/unicode-inside-harfbuzz/unicode.fea` by a script beside it. It has a few Latin letters with an *fi* ligature and one kerning pair, a soft hyphen that it can draw, parentheses, two Hebrew letters and two Hebrew points, a Thai consonant with two marks below it, and one variation sequence. Its glyphs are boxes. Every shaping result and trace on this page was produced by running HarfBuzz {{hb}} on that font when the page was built.

## What HarfBuzz asks of Unicode

The questions are listed once, in an X macro at `hb-unicode.hh:43`, in the same way as the font funcs’ callbacks (see [Fonts and font functions](fonts-and-font-functions.html#one-list-of-callbacks)):

```c++
#define HB_UNICODE_FUNCS_IMPLEMENT_CALLBACKS \
  HB_UNICODE_FUNC_IMPLEMENT (combining_class) \
  HB_IF_NOT_DEPRECATED (HB_UNICODE_FUNC_IMPLEMENT (eastasian_width)) \
  HB_UNICODE_FUNC_IMPLEMENT (general_category) \
  HB_UNICODE_FUNC_IMPLEMENT (mirroring) \
  HB_UNICODE_FUNC_IMPLEMENT (script) \
  HB_UNICODE_FUNC_IMPLEMENT (compose) \
  HB_UNICODE_FUNC_IMPLEMENT (decompose) \
  HB_IF_NOT_DEPRECATED (HB_UNICODE_FUNC_IMPLEMENT (decompose_compatibility)) \
  /* ^--- Add new callbacks here */
```

Two of the eight are deprecated, since 2.0.0, and the shaper never calls them. The other six, and where the shaper asks them:

| Callback | Unicode property | Where HarfBuzz uses it |
|---|---|---|
| `general_category` | General_Category (Lu, Mn, Cf…) | for every character, when its properties are set up (`hb-ot-layout.hh:228`); by the Arabic shaper for the characters of pre- and post-context; by the Indic, Khmer and USE shapers’ `compose` |
| `combining_class` | Canonical_Combining_Class | for every mark, through `modified_combining_class` (`hb-ot-layout.hh:263`) |
| `mirroring` | Bidi_Mirroring_Glyph | in right-to-left text, before normalization (`hb_ot_rotate_chars`, `hb-ot-shape.cc:667`) |
| `script` | Script | only when guessing the buffer’s script (`hb-buffer.cc:753`) |
| `decompose` | canonical Decomposition_Mapping, one step | in normalization, and in the Indic, Khmer and USE shapers’ own versions of it |
| `compose` | the canonical composition of a pair | the same, and in the Hebrew shaper |

Each answers for one code point, or for a pair in the case of `compose`. `decompose` splits a character into at most two: a character whose canonical decomposition is longer is decomposed again, recursively, by the normalizer. That process is the subject of [Normalization](normalization.html).

`script` is asked less than you might expect. Only `hb_buffer_guess_segment_properties` uses it, to take the first character whose script isn’t Common, Inherited or Unknown as the script of the whole buffer. Nothing in the pipeline asks for a character’s script after that: splitting text into runs of one script is the application’s job, as [Buffers](buffers.html) explains.

`mirroring` shows how little HarfBuzz needs. In right-to-left text, each character with a mirror image is replaced by it, if the font has a glyph for it. If the font doesn’t, the character keeps its own glyph and gets the mask bit of the `rtlm` feature instead, so that the font can mirror it itself:

```shape
unicode.ttf  (a)
unicode.ttf  (א)
```

The Hebrew letter makes the second run right-to-left. The output is in visual order, so it reads `parenleft`, `alef`, `parenright`, as it should on the page, but the cluster numbers show that the glyph on the left is the last character typed. The first character, U+0028 LEFT PARENTHESIS, was turned into U+0029 before the `cmap` was asked for it. Without mirroring, the reversed run would have its parentheses facing outwards.

### What HarfBuzz doesn’t ask

The callbacks are a small part of the Unicode data HarfBuzz uses. The rest is compiled in, and doesn’t change whichever callbacks are installed:

* Arabic joining types, Indic syllabic and positional categories, the Universal Shaping Engine’s categories and the vowel constraints come from tables generated from the Unicode data files by their own scripts (`gen-arabic-table.py`, `gen-indic-table.py`, `gen-use-table.py`, `gen-vowel-constraints.py`). Those are covered with the shapers that use them, and the scripts in [Regenerating the Unicode and shaping tables](regenerating-the-unicode-and-shaping-tables.html).
* Extended_Pictographic, for emoji sequences, comes from `hb-unicode-emoji-table.hh`, generated from `emoji-data.txt`.
* Four properties are hand-written functions in `hb-unicode.hh` itself: `is_default_ignorable` (`:167`), `is_variation_selector` (`:119`), `space_fallback_type` (`:219`), which says how wide each space character should be when the font has no glyph for it, and `vertical_char_for` (`:245`), which maps punctuation to its vertical presentation form when a font has no `vert` feature.

So an application that installs ICU’s Unicode functions changes where general categories and combining classes come from, but not which characters are hidden, and not how Arabic joins.

## Unicode funcs objects

An `hb_unicode_funcs_t` holds one function pointer, one user-data pointer and one destroy callback per callback in the list. Every buffer has one. A new buffer gets the default (`hb-buffer.cc:289`), and `hb_buffer_set_unicode_funcs` replaces it.

The *nil* functions in `hb-unicode.cc:60` say what a buffer gets when nothing answers. They are worth knowing, because they are what any callback left unset falls back on:

| Callback | Nil answer |
|---|---|
| `general_category` | Lo, other letter |
| `combining_class` | 0, not reordered |
| `mirroring` | the character itself |
| `script` | Unknown |
| `compose`, `decompose` | false |

So with nil functions every character is a letter, nothing is a mark, and nothing is normalized, but the font’s lookups still apply. `hb_unicode_funcs_get_empty` returns this nil object (`hb-unicode.cc:228`), unlike `hb_font_funcs_get_empty`, which returns the *default* font funcs.

`hb_unicode_funcs_create` takes a parent (`hb-unicode.cc:182`). The new object starts as a copy of the parent’s function and user-data pointers, and keeps a reference to the parent, which is made immutable. Setting a callback replaces one slot, and setting it to `NULL` copies the parent’s back (`hb-unicode.cc:386`). So, to change one property, a client creates funcs with the default as parent and sets one callback. Unlike a font’s parent, this parent is never asked anything while shaping: its pointers were copied when the child was made.

### Three implementations

Which implementation is the default is decided when HarfBuzz is compiled (`hb_unicode_funcs_get_default`, `hb-unicode.cc:152`; [Integrations](integrations.html#unicode-from-icu-or-glib-hb-icu-and-hb-glib) covers the ICU and GLib layers):

```c++
#if !defined(HB_NO_UNICODE_FUNCS) && !defined(HB_NO_UCD)
  return hb_ucd_get_unicode_funcs ();
#elif !defined(HB_NO_UNICODE_FUNCS) && defined(HAVE_GLIB)
  return hb_glib_get_unicode_funcs ();
#elif !defined(HB_NO_UNICODE_FUNCS) && defined(HAVE_ICU) && defined(HAVE_ICU_BUILTIN)
  return hb_icu_get_unicode_funcs ();
#else
  /* … the nil funcs … */
#endif
```

That is: HarfBuzz’s own tables, unless the build leaves them out with `HB_NO_UCD`, then GLib’s, then ICU’s. A build with none of them stops with an `#error` (`:166`), unless `HB_NO_UNICODE_FUNCS` is defined. Each is a static, immutable object, created lazily and freed at exit (`hb-ucd.cc:224`, `hb-glib.cc:146`, `hb-icu.cc:242`):

* **hb-ucd** (`hb-ucd.cc`) reads the tables in `hb-ucd-table.hh`, described in the next section.
* **GLib** (`hb-glib.cc`) calls `g_unichar_type`, `g_unichar_combining_class` and the rest, converting GLib’s script codes to HarfBuzz’s.
* **ICU** (`hb-icu.cc`) maps ICU’s categories onto HarfBuzz’s enum, and composes and decomposes with ICU’s NFC normalizer: `unorm2_composePair`, and `unorm2_getRawDecomposition`, which gives the one-step mapping HarfBuzz wants. It is a separate library, `libharfbuzz-icu`, unless HarfBuzz is configured with `icu_builtin`, the only way ICU can become the default.

> For programmers: the three needn’t agree. Each knows the version of Unicode its library knows, and a system’s ICU or GLib may be older than HarfBuzz’s tables, so that a character one calls unassigned another calls a mark. HarfBuzz’s own tables make shaping depend on nothing but the HarfBuzz version. Installing ICU’s funcs makes HarfBuzz agree with an ICU-based text stack about categories and combining classes, but changes nothing about default ignorables, joining or the script shapers’ categories.

## The built-in tables

HarfBuzz’s own Unicode data is in `hb-ucd-table.hh`, a generated header of more than 400 KB. Its first lines say how it was made:

```c++
/*
 * The following table is generated by running:
 *
 *   ./gen-ucd-table.py ucd.nounihan.grouped.xml hb-script-list.h
 *
 * on file with this description: Unicode 18.0.0
 */
```

The input is the XML form of the Unicode Character Database, without the Unihan data. The script, `gen-ucd-table.py`, extracts five properties for every code point and packs them with [packTab](https://github.com/harfbuzz/packtab), a table packer from the HarfBuzz project. The comment at `gen-ucd-table.py:39` says how each is encoded:

* **gc**, General_Category, as its value in HarfBuzz’s enum, which lists the 30 categories alphabetically, Cc to Zs, so it fits in five bits;
* **ccc**, Canonical_Combining_Class, as itself;
* **sc**, Script, as an index into the 179 values listed in `hb-script-list.h`;
* **bmg**, Bidi_Mirroring_Glyph, as the difference between the mirror and the character, through a palette of the 39 differences that occur;
* **dm**, Decomposition_Mapping, as an index into four lists of decompositions, described below.

HarfBuzz first carried a copy of UCDN, a small Unicode library, from 0.9.5, and made it the default in 1.4.1, in January 2017. 2.5.0, in May 2019, replaced it with “a new Unicode Character Database implementation that is half the size of previously-used UCDN” (`NEWS`). That is this one.

### Looking a property up

Each property is read by one inline function. Here is the general category (`hb-ucd-table.hh:2736`):

```c++
static inline uint8_t _hb_ucd_gc (unsigned u)
{
  /* packtab: [2^8,2^5,2^3,2^1] */
  return u<1114110u ? (uint8_t)(_hb_ucd_u8[7944u+((_hb_ucd_u8[2176u+_hb_ucd_u16[((_hb_ucd_u8[((((((u)>>1))>>3))>>5)])<<5)+((((((u)>>1))>>3))&31)]+((((u)>>1))&7)])<<1)+((u)&1)]) : 2;
}
```

It is a multi-level table. The code point is cut into fields of bits. The highest bits index a first array; what is found there, plus the next field, indexes a second; and so on, until the last array gives the value. The saving comes from sharing: identical blocks, such as those of the hundreds of thousands of unassigned code points, are stored once. All the properties’ arrays share two pools, `_hb_ucd_u8` and `_hb_ucd_u16`. Past the last value stored, the answer is the default, here 2, Cn. A lookup is a few loads and shifts, with one branch.

### Three sizes

The header holds three complete sets of these arrays, and the build chooses one:

| Build | Selected by | gc, ccc, sc, bmg, dm arrays | With the decomposition lists |
|---|---|---|---|
| default | nothing | 42,627 bytes | 51,029 bytes |
| compact | `HB_OPTIMIZE_SIZE` | 37,735 bytes | 46,137 bytes |
| sloppy | `HB_OPTIMIZE_SIZE` and `HB_NO_UCD_UNASSIGNED` | 26,427 bytes | 34,829 bytes |

The sizes are worked out from the array lengths. The compact set holds the same data as the default, packed harder, with deeper lookups. The sloppy one doesn’t: for it, `gen-ucd-table.py:257` first gives every unassigned code point the general category and script of its neighbor, within each block of 128. Identical values pack better, and an unassigned code point gets a plausible answer instead of Cn and Unknown. `HB_TINY` turns on both macros (`hb-config.hh`); see [Compile-time configuration](compile-time-configuration.html).

### Decomposition and composition

Only canonical decompositions are stored (`gen-ucd-table.py:63`), each to one character or two. Compatibility decompositions, such as *ﬁ* to *f i*, aren’t, which is why `hb-ucd` doesn’t implement the deprecated `decompose_compatibility`. Hangul syllables are left out too: `hb-ucd.cc:56` composes and decomposes them by arithmetic, as Unicode defines it.

The `dm` index points into four lists: one-character mappings into plane 0, then into plane 2, then pairs packed into 32 bits, then pairs packed into 64. A pair fits in 32 bits when its first character is below U+0800, its second is in U+0300 to U+037F, and the composite is small enough. That covers most accented Latin, Greek and Cyrillic. The same pair lists, sorted by pair, serve composition, by binary search (`hb_ucd_compose`, `hb-ucd.cc:128`). A composite that mustn’t be recomposed, because it is a composition exclusion or has a non-zero combining class, is stored as 0, and composing its pair fails. `gen-ucd-table.py --rust` writes the same tables as Rust.

## The properties each glyph carries

The callbacks are asked once per character, at the start of shaping. The answers are kept in the glyph itself, in 16 bits of its private `var2` field, named `unicode_props()` (`hb-ot-layout.hh:154`). It is allocated first in `hb_ot_shape_internal` and freed last (`hb-ot-shape.cc:1176`), so it lasts the whole of shaping, long after `codepoint` has become a glyph ID. That is how GSUB knows that a glyph came from a zero-width joiner. The buffer’s other scratch fields are in [Buffers](buffers.html).

The design comment at `hb-ot-layout.hh:187` explains how the bits are shared out. The low byte means the same for every character. The high byte’s meaning depends on the general category:

| Bits | Mask | Meaning |
|---|---|---|
| 0–4 | `UPROPS_MASK_GEN_CAT` | the general category, slightly modified |
| 5 | `UPROPS_MASK_IGNORABLE` | default ignorable, by HarfBuzz’s definition |
| 6 | `UPROPS_MASK_HIDDEN` | hidden: ignorable, but not to be passed over in GSUB |
| 7 | `UPROPS_MASK_CONTINUATION` | continues the grapheme before it |
| 8–15, for Mn, Mc, Me | | the modified combining class |
| 8–15, for Cf | `UPROPS_MASK_Cf_ZWJ`, `_ZWNJ`, `_VS`, `_AAT_DELETED` | a zero-width joiner, a zero-width non-joiner, a variation selector, or a glyph deleted by `morx` |
| 8–15, for Zs | | which kind of space, if it needs the space fallback |

Worked out by hand, as an illustration: U+0301 COMBINING ACUTE ACCENT is Mn (12), a continuation, with combining class 230, which makes 0xE68C. U+200D ZERO WIDTH JOINER is Cf (1), ignorable, a ZWJ and a continuation: 0x01A1. The getters check the category before they trust the high byte, so `_hb_glyph_info_is_zwj` is false for anything that isn’t Cf, and a non-mark’s combining class is 0.

### Setting them

`_hb_glyph_info_set_unicode_props` (`hb-ot-layout.hh:224`) does it, for one glyph. Here it is with its comments shortened:

```c++
  hb_unicode_funcs_t *unicode = buffer->unicode;
  unsigned int u = info->codepoint;
  unsigned int gen_cat = (unsigned int) unicode->general_category (u);
  unsigned int props = gen_cat;

  if (u >= 0x80u)
  {
    if (unlikely (unicode->is_default_ignorable (u)))
    {
      buffer->scratch_flags |= HB_BUFFER_SCRATCH_FLAG_HAS_DEFAULT_IGNORABLES;
      props |=  UPROPS_MASK_IGNORABLE;
      if (u == 0x200Cu) props |= UPROPS_MASK_Cf_ZWNJ;
      else if (u == 0x200Du) props |= UPROPS_MASK_Cf_ZWJ;
      /* Mongolian Free Variation Selectors: issue #234 */
      else if (unlikely (hb_in_ranges<hb_codepoint_t> (u, 0x180Bu, 0x180Du, 0x180Fu, 0x180Fu))) props |= UPROPS_MASK_HIDDEN;
      /* TAG characters: issue #463 */
      else if (unlikely (hb_in_range<hb_codepoint_t> (u, 0xE0020u, 0xE007Fu))) props |= UPROPS_MASK_HIDDEN;
      /* COMBINING GRAPHEME JOINER: issue #554 */
      else if (unlikely (u == 0x034Fu))
      {
        buffer->scratch_flags |= HB_BUFFER_SCRATCH_FLAG_HAS_CGJ;
        props |= UPROPS_MASK_HIDDEN;
      }
    }

    if (unlikely (HB_UNICODE_GENERAL_CATEGORY_IS_MARK (gen_cat)))
    {
      buffer->scratch_flags |= HB_BUFFER_SCRATCH_FLAG_HAS_CONTINUATIONS;
      props |= UPROPS_MASK_CONTINUATION;
      props |= unicode->modified_combining_class (u)<<8;
    }
  }

  info->unicode_props() = props;
```

ASCII is never ignorable and never a mark, so for it only the category is looked up. `is_default_ignorable` and the modified class table are HarfBuzz’s own, whichever funcs are installed. And the function sets **scratch flags** on the buffer: “has default ignorables”, “has a CGJ”, “has continuations”. Later steps check them first, so a buffer of plain Latin letters never runs the code that hides ignorables.

### When properties change

The category is “modified” because HarfBuzz changes it in a few places. A variation selector that the `cmap` doesn’t consume moves from Mn to Cf, so that the `_VS` flag fits, and moves back after positioning. A ligature formed from a non-spacing mark becomes Lo, as the [GSUB chapter](gsub.html#edge-cases-the-code-handles) says. The Thai shaper makes the NIKHAHIT it splits from SARA AM a mark (`hb-ot-shaper-thai.cc:342`). A glyph that `morx` deletes becomes Cf with the `_AAT_DELETED` and hidden bits until it is removed (`hb-ot-layout.hh:431`). The combining class changes too: the Arabic shaper renumbers marks it moves (below), and for a font with no GPOS, fallback mark positioning replaces script-specific classes with the positional ones it can place (`hb-ot-shape-fallback.cc:183`, and [Fallbacks](fallbacks.html)), after normalization has used them.

## Continuations: just enough of graphemes

The continuation bit says that a character belongs with the one before it, in one *grapheme*: what a reader sees as one character. Forming clusters merges each grapheme into one cluster at the cluster levels that work by graphemes, including the default, and text reversed into its native direction is reversed grapheme by grapheme, so that a mark stays after its base. Both are in [Buffers](buffers.html).

Unicode defines graphemes in UAX #29. HarfBuzz implements only “enough of Unicode Graphemes here that shaping in reverse-direction wouldn’t break graphemes” (`hb_set_unicode_props`, `hb-ot-shape.cc:474`). It marks as continuations:

* every mark, Mn, Mc or Me (done in `_hb_glyph_info_set_unicode_props`, above);
* the emoji skin-tone modifiers, U+1F3FB to U+1F3FF;
* the second of a pair of regional indicators, which together make a flag (HarfBuzz issue #2265);
* a zero-width joiner, and an Extended_Pictographic character straight after it, which is how emoji ZWJ sequences hold together;
* the halfwidth katakana voiced sound marks, U+FF9E and U+FF9F (issue #3844), and the TAG characters, U+E0020 to U+E007F, which make emoji subdivision flags (issue #1556).

The last group is Unicode’s Other_Grapheme_Extend, less ZWNJ, because “keeping it separate results in more granular clusters”. (The same loop notices U+2044 FRACTION SLASH, so that `frac` masks are only set up for text that has one.) The difference shows in the clusters:

```shape
unicode.ttf  ❤\u200D❤
unicode.ttf  a\u200Da
```

The second heart follows a ZWJ and is Extended_Pictographic, so all three characters are cluster 0. After *a*, the ZWJ joins the *a*’s cluster, but the next *a* starts its own, 2. (The hearts are .notdef, and the ZWJ is a zero-width space glyph, for reasons explained below.)

## Modified combining classes

Unicode’s canonical combining classes exist to put marks into a canonical order, so that equivalent strings compare equal. They weren’t chosen with fonts in mind, and for some scripts the canonical order isn’t the one fonts expect. So HarfBuzz’s normalizer sorts marks by its own **modified combining classes**: Unicode’s class, passed through a table of 256 entries (`_hb_modified_combining_class`, `hb-unicode.cc:500`), plus three single-character exceptions (`hb-unicode.hh:106`):

```c++
    /* Reorder SAKOT to ensure it comes after any tone marks. */
    if (unlikely (u == 0x1A60u)) return 254;
    /* Reorder PADMA to ensure it comes after any vowel marks. */
    if (unlikely (u == 0x0FC6u)) return 254;
    /* Reorder TSA -PHRU to reorder before U+0F74 */
    if (unlikely (u == 0x0F39u)) return 127;
```

Most of the table is the identity. The changes, as the comments in `hb-unicode.hh` explain them:

| Script | Unicode classes | What HarfBuzz does |
|---|---|---|
| Hebrew | 10–26, one per point | permutes them into the order of the SBL Hebrew manual: shin and sin dots first, then dagesh, rafe, holam, the vowels, and meteg and varika last |
| Arabic | 27–35 | moves shadda (33) before the other harakat |
| Telugu | 84, 91 | makes the two length marks 4 and 5, so that they no longer reorder with the virama (9) |
| Thai | 103 | makes SARA U and SARA UU 3, so that they come before PHINTHU (9), as Uniscribe does |
| Tibetan | 130, 132 | swaps the vowel signs I and U, so that U comes first, for Dzongkha |

The effect, with Hebrew and Thai:

```shape
unicode.ttf  ב\u05B8\u05BC  --no-positions
unicode.ttf  ב\u05BC\u05B8  --no-positions
unicode.ttf  ก\u0E3A\u0E38  --no-positions
unicode.ttf  ก\u0E38\u0E3A  --no-positions
```

In canonical order QAMATS (class 18) comes before DAGESH (21), as in the first line and in any normalized text. HarfBuzz gives dagesh 12 and qamats 21, so both lines come out the same, with the dagesh first: read the visual-order output from the right, `bet`, `dagesh`, `qamats`. Both Thai lines come out with SARA U before PHINTHU, the reverse of Unicode’s order. The trace shows the sort, in normalization’s second round:

```trace
font: unicode.ttf
text: ב\u05B8\u05BC
from: start reorder
to: end reorder
```

The result isn’t in canonical order, but only the font’s lookups ever see it. What matters is that every canonically equivalent input gives the same order, which sorting by any fixed set of classes does.

> For type designers: when you write contextual lookups or mark-to-mark attachment for a sequence of marks, the sequence your lookups see is the one sorted by HarfBuzz’s modified classes, not the one typed and not Unicode’s canonical order. `hb-shape --trace` shows it, at “end reorder”.

The modified class is what the high byte of `unicode_props` holds. The Arabic shaper’s `reorder_marks` hook changes it once more: it moves the “modifier combining marks”, such as hamza above and below, to the front of their run, and renumbers them 25 or 26 (the modified classes HarfBuzz names `CCC22` and `CCC26`) so that the run stays sorted, because the CGJ handling below relies on that (`hb-ot-shaper-arabic.cc:762`). [Joining scripts](joining-scripts.html) has the rest.

## Default ignorables

Unicode’s Default_Ignorable_Code_Point property marks characters that should be invisible if a renderer doesn’t support them: joiners, the soft hyphen, bidirectional controls, variation selectors, invisible operators, TAG characters, and reserved code points for more of the same. Fonts rarely have glyphs for them, and drawing .notdef would be wrong.

HarfBuzz doesn’t get this property from the Unicode funcs. `is_default_ignorable` (`hb-unicode.hh:167`) is a hand-written switch, whose comment lists Unicode’s set as of Unicode 14.0, and the differences:

> Note: While U+115F, U+1160, U+3164 and U+FFA0 are Default_Ignorable, we do NOT want to hide them, as the way Uniscribe has implemented them is with regular spacing glyphs, and that’s the way fonts are made to work. As such, we make exceptions for those four. Also ignoring U+1BCA0..1BCA3.

The four are the Hangul fillers. The last four are the shorthand format controls, used with Duployan (HarfBuzz issue #503). Everything else in that list is in HarfBuzz’s. Being in the source, the list changes only when someone edits it, not when the tables are regenerated.

Being default ignorable does three things to a glyph, at three points in the pipeline:

1. **In GSUB and GPOS matching**, it may be passed over. Most default ignorables are skipped when a lookup looks for the next glyph, with special rules for the joiners, as the [GSUB chapter](gsub.html#joiners) explains.
2. **After positioning**, its advance is set to zero, and its offset along the line too (`hb_ot_zero_width_default_ignorables`, `hb-ot-shape.cc:772`).
3. **At the very end**, its glyph is replaced with an invisible one, or removed (`hb_ot_hide_default_ignorables`, `hb-ot-shape.cc:817`).

So until the end of positioning, a default ignorable is an ordinary glyph, with whatever glyph and advance the font gives it. The test font has a glyph for the soft hyphen, 300 units wide, and kerning between *a* and *b*:

```trace
font: unicode.ttf
text: a\u00ADb
from: start table GSUB
to: end table GPOS
```

```shape
unicode.ttf  a\u00ADb
```

Through GSUB and GPOS, the soft hyphen is the glyph `softhyphen`, 300 units wide. The kerning lookup passes over it to kern the *a* against the *b*. Only then is it zeroed and replaced. Doing it this late lets invisible characters take part in lookups that ask for them.

### Hidden, preserved or removed

Step 3 puts in the buffer’s **invisible glyph**, set with `hb_buffer_set_invisible_glyph` (since 2.0.0), or if there is none, the font’s glyph for the space character: hence the zero-width `space` glyphs in this chapter. If the font has no space glyph either, the ignorables are deleted.

Two buffer flags change this:

* `HB_BUFFER_FLAG_PRESERVE_DEFAULT_IGNORABLES` turns steps 2 and 3 off. The characters keep the glyphs and advances the font gave them.
* `HB_BUFFER_FLAG_REMOVE_DEFAULT_IGNORABLES` (since 1.8.0) turns step 2 off and makes step 3 delete the glyphs. Where a deletion would leave a character with no glyph in its cluster, clusters are merged (`delete_glyphs_inplace`, `hb-buffer.cc:694`).

If both are set, preserving wins. `hb-shape` has an option for each, and `--invisible-glyph` to set the invisible glyph:

```shape
unicode.ttf  a\u00ADa
unicode.ttf  a\u00ADa  --preserve-default-ignorables
unicode.ttf  a\u00ADa  --remove-default-ignorables
unicode.ttf  a\u200Ba
unicode.ttf  a\u200Ba  --preserve-default-ignorables
unicode.ttf  a\u3164a
```

The soft hyphen is hidden, then drawn with the font’s own glyph, then gone. U+200B ZERO WIDTH SPACE has no glyph in this font, so preserving it draws .notdef, at full width. U+3164 HANGUL FILLER is default ignorable to Unicode but not to HarfBuzz, so it is never hidden.

### Substituted, so no longer ignorable

A default ignorable that GSUB has substituted is not default ignorable anymore. The test is in `_hb_glyph_info_is_default_ignorable` itself (`hb-ot-layout.hh:349`):

```c++
  return (info->unicode_props() & UPROPS_MASK_IGNORABLE) &&
         !_hb_glyph_info_substituted (info);
```

The bit is still set, but a glyph with the `SUBSTITUTED` property, which GSUB gives every glyph it writes, is neither skipped by later lookups nor hidden: a font that substitutes an invisible character has something to draw for it. The test font’s `ss01` replaces the soft hyphen with a hyphen:

```shape
unicode.ttf  a\u00ADa  --features=ss01
```

> For type designers: this is how a font can make an invisible character visible, in HarfBuzz at least: substitute it, even with a copy of itself under another name. Mapping it in the `cmap` isn’t enough, because the glyph it maps to is hidden unless the application asks to preserve default ignorables.

The `fallback` backend hides ignorables too. It replaces them with the space glyph, at zero width, using the same hand-written list (`hb-fallback-shape.cc:90`). See [hb_shape and its backends](hb_shape-and-its-backends.html).

## The hidden ones

A few default ignorables are there to change the shaping of the characters around them. If GSUB passed over them, they couldn’t. These get the **hidden** bit as well as the ignorable one:

* the Mongolian free variation selectors, FVS1 to FVS4: U+180B to U+180D, and U+180F (HarfBuzz issue #234);
* the TAG characters, U+E0020 to U+E007F (issue #463);
* U+034F COMBINING GRAPHEME JOINER, but only some of the time (issue #554);
* and, internally, glyphs that `morx` has deleted.

The bit is used in the matcher’s `may_skip` (`hb-ot-layout-gsubgpos.hh:891`), which the GSUB chapter describes. A default ignorable may be skipped only if it isn’t a ZWNJ or ZWJ that the lookup must respect, and isn’t hidden:

```c++
    if (unlikely (_hb_glyph_info_is_default_ignorable (&info) &&
                  (ignore_zwnj || !_hb_glyph_info_is_zwnj (&info)) &&
                  (ignore_zwj || !_hb_glyph_info_is_zwj (&info)) &&
                  (ignore_hidden || !_hb_glyph_info_is_hidden (&info))))
      return SKIP_MAYBE;
```

`ignore_hidden` is true only for GPOS (`hb-ot-layout-gsubgpos.hh:848`). So in GSUB a hidden character always stops a match. Here is the *fi* ligature with an invisible character between the letters each time:

```shape
unicode.ttf  fi
unicode.ttf  f\u2060i
unicode.ttf  f\u00ADi
unicode.ttf  f\u180Bi
unicode.ttf  f\u{E0067}i
unicode.ttf  f\u200Ci
```

U+2060 WORD JOINER and the soft hyphen are ordinary default ignorables, and the ligature forms across them, leaving the invisible character after it, in its cluster. FVS1 and the TAG character U+E0067 are hidden, and block it, as ZWNJ does for its own reasons. In the output they are hidden like any other default ignorable, in the *f*’s cluster, since both are continuations.

Each of these characters says something about the one before it. A Mongolian FVS selects a form of the letter it follows, which a font does with lookups that match the two together. If GSUB passed over the FVS, lookups written for the letter alone would apply as though it weren’t there. (Mongolian is shaped by the Universal Shaping Engine, which uses the Arabic shaper’s joining code, and that code copies each letter’s joining form onto the FVS after it, so that both get the same feature masks: `hb-ot-shaper-arabic.cc:380`; [Joining scripts](joining-scripts.html).) TAG characters likewise follow an emoji, as in subdivision flags, and spell out what it should become.

### CGJ, sometimes

The combining grapheme joiner is a mark with combining class 0. A class-0 character between two marks stops reordering from moving one past the other, and that is CGJ’s main use: to keep marks in an order that normalization would otherwise change, in Hebrew for example. The GSUB chapter’s table lists CGJ as always blocking. That is true of a CGJ doing this job. After reordering, HarfBuzz checks whether each CGJ kept anything apart (`hb-ot-shape-normalize.cc:403`):

```c++
    for (unsigned int i = 1; i + 1 < count; i++)
      if (info[i].codepoint == 0x034Fu/*CGJ*/ &&
          (info_cc(info[i+1]) == 0 || info_cc(info[i-1]) <= info_cc(info[i+1])))
      {
        _hb_glyph_info_unhide (&info[i]);
      }
```

If the character after it isn’t a mark, or the marks on either side were in order anyway, the CGJ prevented nothing, and loses its hidden bit. GSUB can then pass over it. The comparison uses the modified classes. Both cases:

```shape
unicode.ttf  f\u034Fi
unicode.ttf  ב\u05B8\u034F\u05BC  --no-positions
```

Between two letters, a CGJ blocks no reordering, so the ligature forms across it. Between QAMATS and DAGESH, which HarfBuzz would otherwise swap, it stays hidden, and the marks keep their typed order: from the right, `bet`, `qamats`, the CGJ, `dagesh`.

### In GPOS

GPOS passes over all of them. Until 10.0.0, hidden characters blocked positioning lookups as well as substitutions. That release’s `NEWS` says: “CGJ and Mongolian Variation Selectors are now ignored during glyph positioning, previously they would block both glyph substitution and positioning across them.” GPOS also passes over ZWNJ, always, and ZWJ, unless the feature has manual ZWJ, as only `mark` and `mkmk` do for the default shaper. So kerning works across any invisible character:

```shape
unicode.ttf  ab
unicode.ttf  a\u180Bb
unicode.ttf  a\u{E0067}b
unicode.ttf  a\u200Cb
```

Each *a* is kerned against the *b* by −100, whatever is between them. Fallback mark positioning, too, passes over hidden and ignorable glyphs when it gathers a base’s marks (`hb-ot-shape-fallback.cc:442`). And when HarfBuzz synthesizes glyph classes for a font without GDEF, it never makes a default ignorable a mark, though CGJ and the FVSes are Mn (`hb-ot-shape.cc:856`), as the GSUB chapter mentions.

## Variation selectors

The variation selectors VS1 to VS256, U+FE00 to U+FE0F and U+E0100 to U+E01EF, follow a base character to ask for a particular form of it. The Mongolian FVSes don’t count: `is_variation_selector` (`hb-unicode.hh:119`) leaves them to the Arabic shaper.

Normalization gives up on a cluster that holds a variation selector (“it’s just too hard”, `hb-ot-shape-normalize.cc:208`). It asks the font’s `variation_glyph` func, which for hb-ot reads a format 14 `cmap` subtable, for a glyph for the pair. If there is one, the base and the selector become that glyph. If not, both go on with their nominal glyphs, for GSUB to deal with, and the selector is marked with the `_VS` bit. If it is still there at the end, it is hidden like any default ignorable. Since 10.0.0 an application can ask for a glyph of its choice instead, with `hb_buffer_set_not_found_variation_selector_glyph`: the selector then isn’t ignorable (`hb-ot-shape-normalize.cc:227`), and gets that glyph at zero width (`hb-ot-shape.cc:795`), so the client can see that the font didn’t support the sequence and use “a different font for example” (`NEWS`). The test font maps *a* with VS1 to `a.vs1`, and nothing else:

```shape
unicode.ttf  a\uFE00
unicode.ttf  a\uFE01
unicode.ttf  a\uFE01  --not-found-variation-selector-glyph=0
```

With VS1 the pair becomes one glyph. With VS2 the base keeps its usual glyph and the selector is hidden. With a not-found glyph set, here glyph 0, the selector shows as that glyph, in the base’s cluster.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-unicode.h` | the public API: the callbacks’ types, the general category and combining class enums |
| `src/hb-unicode.hh` | `hb_unicode_funcs_t`; the hard-coded `is_default_ignorable`, `is_variation_selector`, `space_fallback_type` and `vertical_char_for`; `modified_combining_class` and the definitions of the modified classes |
| `src/hb-unicode.cc` | the nil funcs, choosing the default, creating funcs from a parent, the modified combining class table |
| `src/hb-ucd.cc`, `src/hb-ucd-table.hh` | the built-in implementation, and its generated tables in three sizes |
| `src/gen-ucd-table.py` | the generator, with the encoding of each property |
| `src/hb-icu.cc`, `src/hb-glib.cc` | the ICU and GLib implementations |
| `src/hb-ot-layout.hh` | `unicode_props()`, its bits, `_hb_glyph_info_set_unicode_props`, and the getters and setters |
| `src/hb-ot-shape.cc` | `hb_set_unicode_props` and continuations, mirroring, zeroing and hiding default ignorables, variation selectors not found |
| `src/hb-ot-shape-normalize.cc` | reordering by modified class, unhiding CGJ, variation selector clusters |
| `src/hb-ot-layout-gsubgpos.hh` | `may_skip`, where ignorable and hidden glyphs are passed over or not |
| `src/hb-unicode-emoji-table.hh` | Extended_Pictographic |

## In short

* HarfBuzz asks Unicode six questions: general category, combining class, mirroring, script, decomposition and composition. They are answered by an `hb_unicode_funcs_t`, normally HarfBuzz’s own, with GLib and ICU as alternatives.
* The built-in tables are generated from the UCD (here Unicode 18.0.0) by `gen-ucd-table.py` and packed into multi-level tables of about 35 to 51 KB, depending on the build.
* Default ignorables, variation selectors, space widths, vertical forms, emoji, joining and the script shapers’ categories don’t come from the funcs. They are compiled in, and the same whatever funcs are installed.
* Each glyph carries 16 bits of Unicode properties for the whole of shaping: a general category, ignorable, hidden and continuation bits, and a byte whose meaning depends on the category.
* Marks are reordered by HarfBuzz’s modified combining classes, which suit fonts better than Unicode’s for Hebrew, Arabic, Telugu, Thai and Tibetan.
* Default ignorables take part in GSUB and GPOS, then lose their advance, then are replaced by an invisible glyph, unless the buffer says to preserve or remove them. One that GSUB has substituted is no longer ignorable.
* Hidden characters, the Mongolian FVSes, TAG characters, and a CGJ that kept marks apart, are never passed over in GSUB. Since 10.0.0 they are passed over in GPOS.
