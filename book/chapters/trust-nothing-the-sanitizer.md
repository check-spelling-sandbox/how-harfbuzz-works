# Trust nothing: the sanitizer

HarfBuzz reads a font where it lies. As [Fonts as bytes](fonts-as-bytes.html) showed, a table is not parsed into objects: a struct is laid over its bytes, and an offset is followed by adding it to an address. That is fast, and it is only safe if every offset and every count in the table can be believed. A font is a file that came from somewhere: a web page, a PDF, an email attachment. Its numbers were chosen by whoever made it, and some of the people who make fonts are trying to break the software that reads them.

The **sanitizer** is what lets the rest of HarfBuzz believe the numbers. Before a table is used for the first time, it walks the table once and answers one question: is it safe to call this table’s methods? If the answer is yes, the table is used from then on with no further checks. If it is no, the table is thrown away, and HarfBuzz carries on as though the font didn’t have it. Since HarfBuzz 12.3 there is nothing in between: the sanitizer no longer tries to repair a broken table, so one bad offset anywhere in `GSUB` costs the whole of `GSUB`. A few checks are not made up front at all, but left until the data is used: in `glyf`, in the AAT tables, and in some parts of `GPOS`.

This chapter is about how those checks are written, what limits the work they do, what happens when one fails, and why HarfBuzz stopped repairing fonts. *When* a table is loaded, and how a failed load is remembered, is the subject of the [next chapter](blobs-faces-and-lazy-tables.html). The examples use one small test font, `good.ttf`, built from `fonts/trust-nothing-the-sanitizer/sanitizer.fea`, and three copies of it, each with one offset deliberately broken after it was compiled. Every shaping result and trace on this page was produced by running HarfBuzz {{hb}} on those fonts when the page was built.

## Why fonts are hostile input

Every number in a table is a claim about the rest of the file. An offset says “there is a Coverage table here”. A count says “there are this many records after me”. A length says “this subtable ends here”. A struct laid over the bytes takes all of them at their word. If an offset points past the end of the table, following it reads whatever memory comes next: at best a crash, at worst other data from the program’s memory, read back out through the shaping result. If a count is multiplied by a record size and the product overflows 32 bits, a check on the product passes while the records themselves run off the end.

Not every attack needs to read out of bounds. OpenType lets many offsets point at the same subtable, which is how a font shares one Coverage table between several lookups. Nothing stops a font from sharing on purpose, at every level, so that a table of a few kilobytes describes a graph with an enormous number of paths through it. Some structures nest without limit: a COLR paint can contain a paint that contains a paint. A checker that follows every path, or recurses as deep as the data asks, can be made to run for a very long time on a tiny file. So the sanitizer has to bound its own work, as well as the shaper’s.

None of this is hypothetical. HarfBuzz is fuzzed continuously: programs feed it endless mutated fonts, and when one finds a crash, a hang, or a runaway allocation, the font is kept as a test. The repository holds 532 fonts in `test/fuzzing/fonts`, and 471 of them are named after the fuzzer report they came from (the [testing chapter](testing-harfbuzz.html) has more).

## The contract

The long comment at the top of `hb-sanitize.hh` (lines 37–104) is the best description of the sanitizer there is, and this chapter follows it closely. Its central paragraph, lightly trimmed:

```c++
/*
 * The sanitize() method of each object type shall return `true` if it's safe to
 * call other methods of the object, and `false` otherwise.
 *
 * Note that what sanitize() checks for might align with what the specification
 * describes as valid table data, but does not have to be.  In particular, we
 * do NOT want to be pedantic and concern ourselves with validity checks that
 * are irrelevant to our use of the table.  On the contrary, we want to be
 * lenient with error handling and accept invalid data to the extent that it
 * does not impose extra burden on us.
 */
```

This is a definition of *safe*, not of *valid*. The sanitizer isn’t a font validator, and a font that passes it may still be wrong in many ways the OpenType specification cares about. What it guarantees is narrower and more useful: that the code which reads this table, written as it is, won’t read outside the table.

So what the sanitizer checks depends on what that code assumes. The comment goes on to describe two ways of dividing the work. In `GSUB`, `GPOS`, and `GDEF`, the reading code assumes that every offset points inside the table, so the sanitizer must check every offset before any of it is used. In `glyf` and `loca`, the reading code checks each glyph’s offsets itself, when it loads the glyph, so the sanitizer checks nothing. The choice is made table by table, for speed and for simplicity. Checking every glyph’s offsets at load would cost time in proportion to the number of glyphs, which is a lot for a CJK font of which a document uses a few hundred glyphs. Leaving them until use makes loading cost nothing and the total cost proportional to the glyphs actually used. For the layout tables the argument goes the other way: their structure is so deeply nested that checking everything once is far simpler than checking at each of the many places that follow an offset.

Being lenient shows up all over the table code. A few examples:

* **Unknown formats pass.** A Coverage table of format 7, which doesn’t exist, is accepted (`OT/Layout/Common/Coverage.hh:76`), and so is an unknown subtable format of any lookup type. The code that reads them treats them as empty: such a Coverage covers nothing, and such a subtable never applies. A font made for a future version of OpenType loses only the parts that HarfBuzz doesn’t understand.
* **Order isn’t checked.** The ranges in a Coverage table of format 2 should be sorted, and the sanitizer only checks that they are all inside the table (`CoverageFormat2.hh:55`). If they aren’t sorted, the binary search that reads them may miss a glyph. That is a wrong answer, not an unsafe one.
* **Unneeded fields are ignored.** A `cmap` format 4 subtable has a `length` field, which fonts have been known to get wrong. The sanitizer doesn’t look at it: it checks that the four segment arrays fit in the table, using only the segment count (`hb-ot-cmap-table.hh:745`). The glyph ID array after them has no count of its own, and the code that reads it bounds it by the real end of the `cmap` table (`:1907`).
* **Some tables check nothing.** `hmtx` accepts anything (`hb-ot-hmtx-table.hh:66`), with a comment explaining that “the users of the struct do all the hard work”: when its accelerator is built, it cuts the number of metrics down to what the table’s real length can hold (`:285–294`).

## Writing a `sanitize()` method

Nearly every struct that can be reached in a table has a `sanitize()` method, and the methods call each other, in the same shape as the structs. Here are the three for a ligature substitution, the lookup from the [GSUB chapter](gsub.html), whole:

```c++
/* LigatureSubstFormat1_2 */
bool sanitize (hb_sanitize_context_t *c) const
{
  TRACE_SANITIZE (this);
  return_trace (coverage.sanitize (c, this) && ligatureSet.sanitize (c, this));
}

/* LigatureSet */
bool sanitize (hb_sanitize_context_t *c) const
{
  TRACE_SANITIZE (this);
  return_trace (ligature.sanitize (c, this));
}

/* Ligature */
bool sanitize (hb_sanitize_context_t *c) const
{
  TRACE_SANITIZE (this);
  return_trace (ligGlyph.sanitize (c) && component.sanitize (c));
}
```

`TRACE_SANITIZE` and `return_trace` are for debugging, and compile to nothing in a normal build. What’s left is a chain of `&&`: the first check that fails makes the whole table fail, and nothing after it is looked at. Most methods are this short, because the work is done by the field types. `coverage` is an `OffsetTo<Coverage>`, and `ligatureSet` is an array of offsets to `LigatureSet`, and each of those knows how to check itself and then what it points at.

The offset’s method is the heart of it (`hb-open-type.hh:522` and `:536`, trimmed, with comments added):

```c++
bool sanitize_shallow (hb_sanitize_context_t *c, const BaseType *base) const
{
  if (unlikely (!c->check_struct (this))) return false;   /* the offset's own 2 bytes */
  hb_barrier ();
  if (unlikely ((const char *) base + (unsigned) *this < (const char *) base))
    return false;                                         /* the address wrapped round */
  return true;
}

bool sanitize (hb_sanitize_context_t *c, const BaseType *base, Ts&&... ds) const
{
  return sanitize_shallow (c, base) &&
         hb_barrier () &&
         (this->is_null () ||
          c->dispatch (StructAtOffset<Type> (base, *this), std::forward<Ts> (ds)...));
}
```

It checks that its own two bytes are in the table, then, unless it is zero (a null offset, which reads as the Null object), it hands the struct at `base + offset` to the context to be checked in turn. The offset doesn’t check whether its target is in range. The target does that itself, as the first thing its own `sanitize()` does. An array works the same way (`hb-open-type.hh:824`): it checks that its count and its whole run of elements fit, and then, unless the elements are plain data with nothing to follow, it dispatches each element.

### The primitives

Underneath, everything comes down to a few checks on the context, `hb_sanitize_context_t`:

| Check | What it asks |
|---|---|
| `check_struct (obj)` | do the struct’s fixed fields, `min_size` bytes, fit in the table? |
| `check_range (p, len)` | do `len` bytes from `p` fit? Charges `len` to the budget |
| `check_range (p, a, b)`, `check_array (p, count)` | the same for `a × b` bytes, or `count` elements, failing if the multiplication overflows |
| `check_range_fast`, `check_point` | the bounds alone, with no charge to the budget |
| `check_ops (n)` | charge `n` to the budget without checking any bytes |

`check_struct` is used on every struct with a fixed part (the `min_size` comes from the `DEFINE_SIZE_*` macros described in [Fonts as bytes](fonts-as-bytes.html)), so it is made as cheap as possible. On a 64-bit machine it only checks that the struct’s *end* is in the table (`hb-sanitize.hh:392`). Every multiplication of a count by a size goes through `hb_unsigned_mul_overflows`, so a count of 0x80000000 records of 2 bytes can’t pass as a small number.

After almost every check comes `hb_barrier ()`. It’s a compiler memory barrier (`hb-atomic.hh:188`) and generates no instructions. No comment says what it is for, but from the code its purpose appears to be to stop the compiler from moving a read of a field to before the check that makes the read safe, which an optimizer is otherwise entitled to do when it can see that the check doesn’t change the field.

### Formats and dispatch

Many tables start with a format number and mean different things for different formats. Those are unions with a `dispatch()` method, the same one that GSUB’s `apply` and the subsetter use (the dispatch pattern is in [Fonts as bytes](fonts-as-bytes.html)). Here is `LigatureSubst`’s (`OT/Layout/GSUB/LigatureSubst.hh:24`, trimmed):

```c++
template <typename context_t, typename ...Ts>
typename context_t::return_t dispatch (context_t *c, Ts&&... ds) const
{
  if (unlikely (!c->may_dispatch (this, &u.format.v))) return c->no_dispatch_return_value ();
  switch (u.format.v) {
  case 1: hb_barrier (); return c->dispatch (u.format1, std::forward<Ts> (ds)...);
  case 2: hb_barrier (); return c->dispatch (u.format2, std::forward<Ts> (ds)...);
  default:return c->default_return_value ();
  }
}
```

`may_dispatch` is a hook for the sanitizer. Every other context returns `true`, but the sanitizer checks the format field itself before the `switch` reads it (`hb-sanitize.hh:134`). `default_return_value ()` for the sanitizer is `true` (`:139`): that’s why unknown formats pass. `LigatureSubst` has no `sanitize()` of its own, and doesn’t need one. The sanitizer’s own `dispatch` prefers an object’s `sanitize()` if it has one and falls back on its `dispatch()` if not (`:150–159`), so the same `switch` serves for checking, applying, subsetting, and the rest.

## A window and a budget

The context holds very little: the start and end of the bytes being checked, their length, and the budgets. Here is `check_range`, which nearly every check comes down to (`hb-sanitize.hh:272`, without its debug message):

```c++
bool check_range (const void *base, unsigned int len) const
{
  const char *p = (const char *) base;
  bool ok = (uintptr_t) (p - this->start) <= this->length &&
            (unsigned int) (this->end - p) >= len &&
            ((this->max_ops -= len) > 0);
  return likely (ok);
}
```

The first line checks that `p` is in the window, with an unsigned comparison that also catches a `p` before the start. The second checks that `len` bytes fit after it. The third is the budget.

### `max_ops`

`max_ops` is set when checking starts (`start_processing`, `:225`) from the length of the table:

| Constant | Value | Meaning |
|---|---|---|
| `HB_SANITIZE_MAX_OPS_FACTOR` | 64 | the budget is 64 for each byte of the table… |
| `HB_SANITIZE_MAX_OPS_MIN` | 16,384 | …but never less than this… |
| `HB_SANITIZE_MAX_OPS_MAX` | 0x3FFFFFFF | …or more than this (about a billion) |
| `HB_SANITIZE_MAX_SUBTABLES` | 0x4000 (16,384) | lookup subtables, over the whole table |

The budget is counted in bytes, not checks: each `check_range` spends the number of bytes it checks. So a table can, in effect, be read through 64 times over. An honest table is read about once, since each struct is reached by one path or a few. A table that shares its subtables in order to be walked millions of times runs out, and once `max_ops` reaches zero every later check fails, so the table fails too. The `GSUB` in `good.ttf` is 156 bytes long, which would give it 9,984; it gets the minimum, 16,384.

A few structures cost more to use than their size suggests, and they spend extra with `check_ops`. Records whose ValueFormat is empty take no bytes, so a GPOS pair adjustment of format 2 with two empty formats charges its number of class pairs, and an array of empty ValueRecords its count (`PairPosFormat2.hh:189`, `ValueFormat.hh:448`). A chained context rule charges the bytes of the arrays its length checks skipped over, so that many offsets to one large rule are charged for each (`hb-ot-layout-gsubgpos.hh:5027`). And each COLR `PaintComposite` charges its own size (`OT/Color/COLR/COLR.hh:1915`), with the comment “PainComposite can get exponential”: it has two child paints, so a chain of them doubles at each step.

Before 15.2.0, a single substitution or positioning of format 1 also charged half the number of glyphs its Coverage covered, because a Coverage of a few bytes can name every glyph in the font as one range. That charge cost whole tables. A valid `GSUB` with a few single substitutions whose Coverages named many glyphs by range could run out of budget and be rejected ([HHW-47](bugs-this-book-found.html#hhw-47)), and a null Coverage, or one of an unknown format, counted as about four billion glyphs, which rejected the table for certain ([HHW-1](bugs-this-book-found.html#hhw-1); [Fonts as bytes](fonts-as-bytes.html#where-the-rule-broke)). 15.2.0 dropped it. The cost of a wide Coverage is now charged where the glyphs are walked: in the glyph closure and when subsetting (`SingleSubstFormat1.hh:81`, `Coverage.hh:304`).

The second budget, `max_subtables`, counts lookup subtables across the whole table. `Lookup::sanitize` adds each lookup’s subtable count to it (`hb-ot-layout-common.hh:1426`), and the table fails once it passes 16,384.

The sanitizer’s `max_ops` is a different budget from the buffer’s `max_ops`, which the [GSUB chapter](gsub.html) mentions and which limits the work of shaping. They share a name and an idea.

### Recursion

Most of a table can only nest as deep as its format allows: a GSUB lookup holds subtables that hold Coverage tables, and that’s the end of it. (A context lookup refers to other lookups by index, not by offset, so the sanitizer doesn’t follow it.) Two structures can nest themselves: COLR’s `Paint`, and the `Condition` of a FeatureVariations record, which can be the AND, OR, or negation of other conditions. Both count their depth with `check_start_recursion` and `end_recursion` (`hb-sanitize.hh:376`), and fail beyond `HB_MAX_NESTING_LEVEL`, which is 64 (`hb-ot-layout-common.hh:4867`, `COLR.hh:2275`). That limit is the same one that limits nested lookups during shaping (see the [Limits appendix](limits.html)).

### The glyph count

Some tables can’t be checked without knowing how many glyphs the font has. `gvar` has an offset for each glyph, plus one, and `sbix` the same. An AAT lookup table of format 0 has a value for each glyph, and a CFF charset names glyphs up to the count. So the context carries `num_glyphs` (`hb-sanitize.hh:180`), and `reference_table`, the usual way in, sets it from the face before checking anything (`:437`). The [next chapter](blobs-faces-and-lazy-tables.html#the-first-gsub-access-step-by-step) follows this step, including how `maxp`, which is where the glyph count comes from, avoids needing the glyph count to load itself.

### A smaller window

The window can also be narrowed to a single object with `hb_sanitize_with_object_t` (`hb-sanitize.hh:456`), which calls `set_object` on the way in and `reset_object` on the way out. While it’s in place, `start` and `end` are the bounds of that object, as its own length field states them, so nothing inside it can be checked as valid if it lies beyond its stated end. The `kern` and `kerx` tables use this to keep each subtable within its own length. But they make an exception for the last subtable, and the comment explains why (`hb-aat-layout-kerx-table.hh:1244`): the old `kern` table stores subtable lengths in 16 bits, some versions of Calibri have a `kern` subtable longer than 64 KB, and Microsoft’s implementation reads only one subtable anyway, so the length is evidently ignored in practice. HarfBuzz ignores it too, for the last subtable only.

## Pass or fail

`sanitize_blob` (`hb-sanitize.hh:401`) runs the top-level `sanitize()` and acts on the answer. If the table passed, its blob is made immutable and returned. If it failed, the blob is released and the **empty blob** is returned in its place. There’s no third outcome. A table that is missing from the font gives the empty blob too, so the code that uses a table can’t tell “missing” from “broken”, and doesn’t need to: the empty blob reads as the table’s Null object, all zeros, which is a valid, empty table. A failed `GSUB` reads as a `GSUB` with no scripts, no features, and no lookups. How that result is stored on the face, so that the table is never looked for or checked again, is in [Failure is cached as Null](blobs-faces-and-lazy-tables.html#failure-is-cached-as-null).

The test font `good.ttf` has two GSUB lookups that have nothing to do with each other: `liga` makes `f i` into `f_i`, and `calt` turns `a` into `a.alt` before `b`. Its GPOS has a kerning pair, `b a`, and a mark-to-base lookup that puts `acutecomb` over `a` and `b`. Here are the lookups, each with tabs for its views:

```views
font: good.ttf
lookup: 0
fea: sanitizer.fea liga
cpp: LigatureSubstFormat1_2
print: spec hex
```

In `bad-gsub.ttf`, exactly two bytes are different: the `coverageOffset` of that ligature subtable, `00 3a` at GSUB offset 0x005e in the Bytes view, has been changed to `ff ff`, which points far beyond the end of the 156-byte table. Nothing else in `GSUB`, and nothing in any other table, has changed:

```shape
good.ttf  fi
bad-gsub.ttf  fi
good.ttf  ab
bad-gsub.ttf  ab
good.ttf  ba
bad-gsub.ttf  ba
```

The ligature is gone, as you would expect. But so is the `calt` substitution, though its lookup is perfect and the bad offset is in a different lookup. HarfBuzz didn’t skip one subtable or one lookup: it dropped the whole table. The kerning of `b a` is unaffected, because `GPOS` is a different table, checked on its own, and it passed. The trace shows what GSUB looked like to the shaper:

```trace
font: bad-gsub.ttf
text: fi
from: start table GSUB
to: end table GSUB
```

Nothing runs between the start of the table and its end. There aren’t even any skipped lookups, since an empty table has no lookups to skip, and the script tag is empty because the table has no scripts to choose from.

> For type designers: a single bad offset loses the whole of `GSUB`, or the whole of `GPOS`, in HarfBuzz {{hb}}. If a font shapes as though one of its layout tables were missing, try decompiling that table with fontTools (`ttx -t GSUB font.ttf`). It fails on all three of this page’s broken fonts, and its error names the path to the bad offset, such as `('Coverage', 'SubTable', 'Lookup', 'LookupList')`. Don’t rely on what other engines do with the same font: as the next section says, they don’t agree.

### A list of known bad fonts

There’s one other way a sanitized table can be dropped. After `GDEF`, `morx`, and `mort` are sanitized, their accelerators ask `is_blocklisted` (`GDEF.hh:988`, `hb-aat-layout-morx-table.hh:1417`) whether this is one of a few specific fonts whose table is known to be harmful, identified by the lengths of their tables. The comment in `hb-ot-layout.cc:157` lists the `GDEF` cases: some versions of Times New Roman Italic and Bold Italic class the ASCII double quote as a mark, and many versions of Tahoma class some spacing IPA symbols as marks, so their `GDEF` is dropped to keep those glyphs from losing their width. The `morx` list (`hb-aat-layout.cc:215`) has one font, and the `mort` list none. `GSUB` and `GPOS` had empty lists until 15.2.0, which removed them. A build with `HB_NO_OT_LAYOUT_BLOCKLIST` defined has no `GDEF` list, and one with `HB_NO_AAT_LAYOUT_BLOCKLIST` none for `morx`; `HB_NO_LEGACY` defines both.

## Why it repairs nothing

The comment at the top of `hb-sanitize.hh` ends with a note on the name:

```c++
/*
 * Note:
 * Sanitize was named so because it used to try to recover from errors by
 * modifying the data to make it valid.  This is no longer the case, as it
 * could make HarfBuzz hallucinate new rules if there was aliasing in the
 * data.  However, the name stuck.  See: https://behdad.github.io/harfbust/
 */
```

For a long time, a bad offset didn’t lose the table. When the sanitizer found an offset whose target failed, it wrote zero over the offset, in the font’s own bytes. This was called **neutering** the offset. A zero offset is a null offset, which reads as an empty object, so the subtable or Coverage behind it simply vanished, and the rest of the table, now valid, was kept. The cost was that the sanitizer needed to write to the font. Older versions of `hb-sanitize.hh` counted the edits a table needed and checked whether editing was allowed (`may_edit`). If a table needed edits and its blob was read-only, they made the blob writable (copying it, or making a private memory mapping writable with `mprotect`) and checked it again from the start. There was a limit on edits, to stop a broken font from making checking slow: NEWS for 1.1.0, in 2015, reads “Reject font tables that need more than 8 edits”.

### Aliasing

The trouble is that a table’s bytes aren’t owned by one struct each. OpenType allows any number of offsets to point at the same bytes, and nothing in the format stops two *different* structs from overlapping, so that the same two bytes are an offset to one reader and a glyph ID or a count to another. When the sanitizer wrote a zero into such bytes, it didn’t only disable the bad offset. It changed whatever else was stored there, and the changed data could pass its own checks and be used.

Here is a hand-written illustration, not taken from a real font. An Extension lookup holds a 32-bit offset to its subtable. Suppose the font is arranged so that the high 16 bits of that offset are also read, by another lookup, as a 16-bit offset to a Coverage table, and that offset is bad. Neutering writes `00 00` into it. The second lookup loses its Coverage, as intended. But the Extension’s 32-bit offset has now lost its high half, and points somewhere much earlier in the table, where it finds whatever bytes happen to be there and reads them as a subtable. If those bytes pass the checks, the font has gained a lookup that its maker never wrote. And since the result depended on which offset the sanitizer checked first, it was hard to reason about at all.

### HarfBust

In late 2025, Behdad Esfahbod published a puzzle font, HarfBust, made from Anton, which forms a ligature for the word “HarfBust” in HarfBuzz, HarfRust, and DirectWrite but not in Core Text, and others for “HarfBuzz” and “HarfRust” in only one engine each. It doesn’t use any special features to do it: its `GSUB` contains deliberately invalid offsets, arranged so that each engine’s way of handling them produces a different font. His explanation, in a comment on HarfBuzz issue #5535, describes the three ways:

* **Core Text** discards the offending table, and uses the rest of the font.
* **DirectWrite** ignores the out-of-bounds offsets, and processes the rest of the table.
* **HarfBuzz** wrote zeros over the out-of-bounds offsets, which, with aliasing, could create new data.

The OpenType and ISO Open Font Format specifications say that such data is invalid, but not what an implementation should do with it. Any choice is reasonable as long as it is safe, and the comment adds that a rule requiring the whole *font* to be rejected would be very costly, since it would force every client to validate everything up front, outlines included.

HarfBuzz 12.3.0 made the change, and its NEWS entry puts it plainly:

> Invalid font tables (eg. GSUB/GPOS) are outright rejected, instead of partially validated and used. This behavior is different from DirectWrite and HarfRust, and is in line with Core Text.

So `sanitize()` became a pure check: it reads the table and returns `true` or `false`, and the bytes it was given are never touched. That also means the result no longer depends on the order of the checks.

The change had a cost, which NEWS also records. Any check that is stricter than it needs to be now costs a whole table, and 13.0.0 fixed one such case: “Don’t reject malformed `cmap` subtables, a regression from 12.3.0 when we stopped sanitizing malformed tables.”

### What’s left of repairing

A few things in the source make sense only as remains of the old design:

* `may_edit`, the edit counter, and the second, writable pass are gone. There’s no `try_set` or `neuter` anywhere in `src/`.
* The blob memory mode `HB_MEMORY_MODE_READONLY_MAY_MAKE_WRITABLE`, and `try_make_writable_inplace_unix`, which calls `mprotect` (`hb-blob.cc:415`), are still there, and fonts opened from a file are still mapped in that mode (`hb-blob.cc:654`). But the sanitizer never asks for writable bytes now. [Blobs, faces, and lazy tables](blobs-faces-and-lazy-tables.html#writable-copies-and-why-little-needs-them-now) covers what does.

Until 15.2.0 there was one more: `HB_SANITIZE_MAX_EDITS`, defined as 32 with the comment “This limits sanitizing time on really broken fonts”, and read by nothing.

## Checks that wait

The contract allows a table to leave checks to the code that reads it, as long as that code makes them. Some tables do that for everything, and `GPOS` does it for a few parts.

### glyf and loca

`glyf`’s `sanitize()` is the simplest in HarfBuzz (`OT/glyf/glyf.hh:56`):

```c++
bool sanitize (hb_sanitize_context_t *c HB_UNUSED) const
{
  TRACE_SANITIZE (this);
  /* Runtime checks as eager sanitizing each glyph is costly */
  return_trace (true);
}
```

`loca`’s is the same. The checks are in the `glyf` accelerator instead. It works out the number of glyphs from the length of `loca`, so that reading glyph *n*’s offset and the one after it is always in range (`glyf.hh:254`), takes the smaller of that and the face’s glyph count, and then, each time a glyph is loaded, refuses one whose offsets run backwards or past the end of `glyf` (`glyf.hh:595`). The glyph’s own data, its contours, points, and components, is then read with checks against its own length. A broken glyph is then a missing glyph, and every other glyph still works. [Drawing outlines](drawing-outlines.html) has the rest of that code.

CFF and CFF2 are similar in spirit. At load, the sanitizer checks the structure of the table: the INDEXes, the charset against the glyph count, the FDSelect. A glyph’s charstring is just an entry in an INDEX at that point. Its contents are a program, which can only be checked by running it, and the charstring interpreter does its own checking, with its own limit of operations, `HB_CFF_MAX_OPS` (200,000).

### The AAT tables

`morx` and `kerx` are driven by state machines. Which entry, action, or ligature component a machine will read depends on the glyphs being shaped, so most of their indexes can’t be checked until they are used. The AAT apply context therefore carries a sanitizer of its own, over the table being applied, with the glyph count set and the budget set to the maximum (`hb-aat-layout.cc:71–74`). The state-machine code calls it before each read that depends on the text, as in `hb-aat-layout-morx-table.hh:562`, where a ligature action is checked before it’s used, and gives up on that action if the check fails. The [AAT chapter](aat-inside-harfbuzz.html) covers those machines.

### Some of GPOS

`GSUB` and `GPOS` share one accelerator template, and its constructor sets a flag before it sanitizes the table (`hb-ot-layout-gsubgpos.hh:6992`):

```c++
hb_sanitize_context_t sc;
sc.lazy_some_gpos = true;
this->table = sc.reference_table<T> (face);
```

Nothing in `GSUB` looks at the flag. Six places in `GPOS` do, and each of them, when it is set, checks the bounds of an array but doesn’t follow the offsets inside it:

| Where | Checked at load | Left until use |
|---|---|---|
| `SinglePos` formats 1 and 2 (`ValueFormat.hh:431`, `:450`) | the value records | their Device offsets |
| `PairPos` format 1 (`PairSet.hh:112`) | the pair value records | their Device offsets |
| `PairPos` format 2 (`PairPosFormat2.hh:190`) | the class-by-class matrix of value records | their Device offsets |
| `CursivePos` (`CursivePosFormat1.hh:247`) | the array of entry and exit records | the entry and exit Anchors |
| `AnchorMatrix` (`AnchorMatrix.hh:99`), in `MarkBasePos`, `MarkLigPos`, and `MarkMarkPos` | the matrix of anchor offsets | the base, ligature-component, and mark2 Anchors |

The mark anchors, in each lookup’s `MarkArray`, are still checked at load, and so is everything else: coverage tables, class definitions, the value records’ own bytes.

No comment says why these were chosen, but the pattern is the one the header comment describes for `glyf`. These are the parts of GPOS that grow fastest with the font: a pair-kerning class matrix has a value record for every pair of classes, a mark-to-base table has an anchor offset for every base and every mark class, and any value record may carry Device offsets. Following every offset in them at load costs time in proportion to all of that, while shaping a line of text uses a handful. Their arrays are still checked whole, which is a single `check_range` each, so only following the offsets is deferred.

Those offsets are checked when they are used instead. The context that applies GSUB or GPOS lookups holds a sanitizer of its own, made over the same table blob (`hb-ot-layout-gsubgpos.hh:1262`, built at `:1299`), and a fresh one is made each time a table is applied to a buffer (`hb-ot-layout.cc:2091`). Before an anchor is read from a matrix (`AnchorMatrix.hh:115`), before a cursive entry or exit anchor (`CursivePosFormat1.hh:282`, `:296`), and before a Device table is read for a value record (`ValueFormat.hh:357`) or an anchor (`AnchorFormat3.hh:40`), the offset is sanitized there, fully, since this sanitizer doesn’t have the flag set. If the check fails, that one item is treated as missing: a Device table reads as Null and adds nothing, and an anchor that isn’t found means the attachment doesn’t happen, so the next subtable gets a chance at the glyph.

The two other broken fonts show the difference. In `bad-gpos.ttf`, the `markCoverage` offset of the mark-to-base subtable points past the end of `GPOS`. That is checked at load. In `bad-anchor.ttf`, the offset to the anchor of base glyph `b` in the same subtable’s base array points past the end. That is left until use:

```shape
good.ttf  ba
bad-gpos.ttf  ba
bad-anchor.ttf  ba
good.ttf  a\u0301
bad-gpos.ttf  a\u0301
bad-anchor.ttf  a\u0301
good.ttf  b\u0301
bad-gpos.ttf  b\u0301
bad-anchor.ttf  b\u0301
```

`bad-gpos.ttf` has lost all of `GPOS`. The kerning between `b` and `a` has gone, though the bad offset is in a different lookup, and the accents are no longer placed by the font’s anchors at all. They are placed by HarfBuzz’s fallback mark positioning instead, which is what HarfBuzz does for a font with no `GPOS` (see [Fallbacks](fallbacks.html)): above the base glyph’s outline, which here is the same box for `a` and `b`, so both accents end up at the same height.

`bad-anchor.ttf` keeps its `GPOS`. The kerning works, and the accent over `a` is placed by its anchor just as in `good.ttf`. Only the accent over `b` is wrong: its anchor failed its check when HarfBuzz tried to use it, so the mark wasn’t attached, and it stays where it was, at zero offset. The rest of the table never noticed.

> For type designers: the lazy checks make HarfBuzz forgiving of exactly one kind of mistake, a bad Anchor or Device offset inside a large positioning table. Don’t count on it. Other engines may reject the whole table, and so, it seems, may HarfBuzz’s own subsetter. Its code (not tested here) shows `hb-subset` loading `GPOS` with an ordinary sanitizer, without the flag (`hb-subset-plan.hh:243`), so the same font that shapes in HarfBuzz may lose its `GPOS` when it is subset for the web.

> For programmers: to watch the sanitizer work, build HarfBuzz with `HB_DEBUG_SANITIZE` defined to a depth, for example `-DHB_DEBUG_SANITIZE=100`. Every `TRACE_SANITIZE` then prints a line for the struct it is checking, and every `check_range` prints the range it checked and whether it was in bounds (`OK` or `OUT-OF-RANGE`), all on standard error. The table’s last line says `PASSED` or `FAILED`, and the first `OUT-OF-RANGE` before a `FAILED` is usually the check that lost the table.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-sanitize.hh` | the header comment (the contract, and the note on repairing), the limits, `hb_sanitize_context_t`, `sanitize_blob`, `reference_table`, `hb_sanitize_with_object_t` |
| `src/hb-open-type.hh` | `sanitize()` for offsets (`OffsetTo`) and the array types |
| `src/hb-dispatch.hh` | the dispatch context that the sanitizer is one kind of |
| `src/hb-ot-layout-gsubgpos.hh` | the GSUB/GPOS accelerator that sets `lazy_some_gpos`, and the apply context’s own sanitizer |
| `src/OT/Layout/GPOS/` | the deferred checks: `ValueFormat.hh`, `PairSet.hh`, `PairPosFormat2.hh`, `CursivePosFormat1.hh`, `AnchorMatrix.hh`, `AnchorFormat3.hh` |
| `src/hb-ot-layout-common.hh` | `Lookup::sanitize` (the subtable budget, Extension types) and `Condition` (recursion) |
| `src/OT/glyf/glyf.hh` | a table that checks at use instead of at load |
| `src/hb-aat-layout.cc`, `src/hb-aat-layout-kerx-table.hh` | the AAT apply context’s sanitizer; narrowing the window to one subtable |
| `src/hb-ot-layout.cc`, `src/hb-aat-layout.cc` | `is_blocklisted` |
| `NEWS` | 12.3.0 (tables rejected outright), 13.0.0 (the `cmap` regression), 1.1.0 (the old edit limit) |

## In short

* The sanitizer checks that a table is safe for HarfBuzz’s own code to read, not that it is valid. It is deliberately lenient: unknown formats pass, and anything the reading code checks for itself is left alone.
* `sanitize()` methods are chains of checks in the shape of the structs. Offsets check their own bytes and then dispatch their target, which checks its own bounds.
* Work is bounded: 64 bytes of checking for each byte of table (at least 16,384), 16,384 lookup subtables, and 64 levels of recursion.
* A table passes or fails as a whole. A failed table becomes the empty blob, and everything that reads it sees an empty table. Since 12.3, nothing is repaired, so one bad offset loses the whole of `GSUB` or `GPOS`.
* Repairing stopped because writing zeros over bad offsets could change other data that shared those bytes, and so invent rules. Core Text also drops the table. DirectWrite skips the bad offsets and keeps the rest of the table, and NEWS says HarfRust doesn’t drop the table either.
* `glyf`, `loca`, and `hmtx` are checked by the code that reads them, and CFF charstrings and the AAT state machines as they run. In `GPOS`, Anchor and Device offsets inside value records, cursive records, and anchor matrices are checked only when they are used, and a bad one loses only itself.
