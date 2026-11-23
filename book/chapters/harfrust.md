# HarfRust

**HarfRust** is, in its [README](https://github.com/harfbuzz/harfrust)’s words, “a Rust port of HarfBuzz text shaping engine”. It is published as the `harfrust` crate, it is developed in the HarfBuzz organization on GitHub, and HarfBuzz itself can call it as one of its shapers. It takes the same text and the same font as HarfBuzz’s `ot` shaper and should give the same glyphs and positions, so most of what this book says about shaping is also a description of HarfRust.

HarfRust has a chapter of its own, and HarfBuzz in other languages doesn’t, because it is a different program. harfbuzzjs and uharfbuzz are HarfBuzz itself: the same C++, compiled to WebAssembly for JavaScript or wrapped with Cython for Python. Everything the rest of this book says about HarfBuzz is true of them, and [The family](family.html#other-relatives) and [Integrations](integrations.html#other-language-bindings) list them with the other bindings. HarfRust is HarfBuzz rewritten in Rust. It follows HarfBuzz’s code closely, but it is separate code, with its own gaps and, in a few places, its own results.

What differs is everything around the shaping. HarfRust doesn’t read fonts the way HarfBuzz does: it has no sanitizer, and uses a Rust parser, `read-fonts`, which checks each read as it happens. `read-fonts` is part of [Fontations](https://github.com/googlefonts/fontations), Google Fonts’ collection of crates for reading and writing fonts, and HarfRust leaves drawing, hinting, and subsetting to other Fontations crates. And it follows HarfBuzz release by release, a short way behind.

This chapter covers where HarfRust came from, how it is kept in step, how it reads fonts, what it leaves out, and its Rust and C APIs and the HarfBuzz backend. It ends with every `shape` example in this book run through HarfRust as well, and what came out different. Everything about HarfRust is from its repository at release 0.16.1 (tag `0.16.1`, 19 November 2026), the one its README says “Matches HarfBuzz 15.2.0”, and about Fontations from its repository at `read-fonts` 0.47.0 and `skrifa` 0.50.0, the versions HarfRust 0.16.1 uses. Paths without `src/` are in those repositories.

## Where it came from

Behdad Esfahbod’s study [Introducing HarfRust](https://docs.google.com/document/d/1aH_waagdEM5UhslQxCeFEb82ECBhPlZjy5_MwLNLBYo/preview), dated 10 June 2025 and linked from HarfRust’s README, gives the history. It lists earlier attempts at text shaping in Rust: harfbuzz_rs (2017), a safe Rust interface to the C++ library rather than a port; Allsorts (2019), a separate implementation by YesLogic; RustyBuzz; and swash (2021).

**rustybuzz** was a pure Rust port of HarfBuzz by Yevhenii Reizner, built on his own font parser, ttf-parser, and “initially written to fulfill a need by resvg”, his SVG renderer. The study quotes him declaring it deprecated in August 2023, and says Laurenz Stampfl then brought it from HarfBuzz 2.7.1 up to 10.1.0. Its [README](https://github.com/harfbuzz/rustybuzz) now begins: “This project is not developed further, unmaintained, and archived”, and recommends HarfRust.

Google Fonts wanted the same port to read fonts with its own crates, [Fontations](https://github.com/googlefonts/fontations), to avoid, as HarfRust’s README puts it (`README.md:16`), “shipping (and maintaining) multiple implementations of core font parsing for `skrifa` consumers”, and points to [Fontations issue 956](https://github.com/googlefonts/fontations/issues/956). The study says that when no agreement was reached with rustybuzz’s author in 2024, Chad Brokaw started a fork, first called HarfRuzz, and that it was renamed because the name looked too much like HarfBuzz. Behdad Esfahbod started contributing in May 2025 and brought it up to HarfBuzz 11.2.1, and the fork dropped ttf-parser for `read-fonts`. “HarfRust is an official HarfBuzz project”, the study says, developed by Chad Brokaw of the Google Fonts team with Behdad Esfahbod’s contributions.

The name is a pun, as HarfBuzz’s is. *Harf* (حرف) is “type”, and the study explains *rāst* (راست) as Persian for “true”, “right”, or “honest”: “Type–True”, a nod to TrueType, beside HarfBuzz’s “Type–Open”.

## Keeping in step with HarfBuzz

The study says which project leads: “Until further notice, all development will primarily happen in HarfBuzz, with the intention to quickly bring HarfRust up to date with each and every HarfBuzz release.” HarfRust’s `CHANGELOG.md` shows it happening. Every release names the HarfBuzz release it matches:

| HarfRust | Released | Matches HarfBuzz |
|---|---|---|
| 0.1.0 | 10 June 2025 | 11.2.1 |
| 0.5.0 | 7 January 2026 | 12.3.0 |
| 0.13.3 | 25 August 2026 | 14.3.1 |
| 0.14.0 | 3 October 2026 | 14.5.1 |
| 0.15.0 | 21 October 2026 | 15.0.0 |
| 0.16.0 | 11 November 2026 | 15.1.0 |
| 0.16.1 | 19 November 2026 | 15.2.0 |

HarfBuzz 15.1.0 came out on 4 November 2026 (`NEWS`), and HarfRust 0.16.0 followed a week later. HarfRust 0.16.1 came out a day *before* the release it matches, HarfBuzz 15.2.0 (20 November 2026): its changelog lists the same fixes as HarfBuzz’s `NEWS`, ported while they were still on HarfBuzz’s main branch (inferred from the dates). The README’s release checklist makes the matched version part of every release: it goes in the README, the changelog, and the C API’s version macros (`README.md:113`).

The changelog also shows the gaps being closed. rustybuzz’s README lists what it never did: no `mort` table, and no Arabic fallback shaping, because building lookups on the fly needs “serialization code that is associated with subsetting”. HarfRust 0.13.3 added both: “Add support for shaping legacy AAT `mort` tables” and “Synthesize Arabic fallback GSUB lookups” ([Joining scripts](joining-scripts.html#shaping-without-gsub) describes what HarfBuzz does there). 0.15.0 added fonts with more than 65,535 glyphs and FeatureVariations 1.1 lookup variations.

### How the code matches

The port keeps HarfBuzz’s structure, so a reader of one can find their way in the other. The names of functions mostly survive: HarfRust’s `normalize.rs` has `decompose_current_character`, `handle_variation_selector_cluster`, and `decompose_multi_char_cluster` (`:187`, `:249`, `:311`), as `hb-ot-shape-normalize.cc` does (`:150`, `:204`, `:255`). Where a Rust type stands for a HarfBuzz one, a comment often says which: 50 comments of the form `/// HB: hb_aat_map_t` (`harfrust/src/aat/map.rs:10`). Code duplicated in HarfBuzz is duplicated here too, with HarfBuzz’s warning: “This block copied from NoncontextualSubtable::apply. Keep in sync.” (`aat/morx.rs:571`). Roughly:

| HarfBuzz (`src/`) | HarfRust (`harfrust/src/`) |
|---|---|
| `hb-buffer.cc`, `hb-buffer.hh` | `buffer.rs` |
| `hb-ot-shape.cc`, `hb-ot-shape-fallback.cc` | `shape.rs`, `plan.rs`, `fallback.rs` |
| `hb-ot-shape-normalize.cc` | `normalize.rs` |
| `hb-ot-map.cc` | `ot/map.rs` |
| `hb-ot-layout-gsubgpos.hh` (`match_input`, `apply_lookup`, the skipping iterator) | `ot/apply.rs`, `ot/lookup/context.rs` |
| `OT/Layout/GSUB/`, `OT/Layout/GPOS/` | `ot/lookup/gsub.rs`, `ot/lookup/gpos.rs` |
| `hb-ot-shaper-*.cc` | `ot/shaper/*.rs` |
| `hb-ot-shaper-*-machine.rl` | `ot/shaper/*_machine.rl` and the generated `*_machine.rs` |
| `hb-aat-layout-*-table.hh` | `aat/morx.rs`, `kerx.rs`, `kern.rs`, `trak.rs` |
| `hb-ucd-table.hh`, `hb-ot-tag-table.hh`, `hb-ot-shaper-indic-table.cc`, … | `unicode/ucd_table.rs`, `tag_table.rs`, `ot/shaper/indic_table.rs`, … |

The tables in the last row are generated by HarfBuzz’s own scripts, which have a `--rust` mode (“generated by running: ./gen-indic-table.py --rust …”, at the top of `indic_table.rs`; [The Universal Shaping Engine](universal-shaping-engine.html) and [Dotted circles and broken clusters](dotted-circles-and-broken-clusters.html) describe them). HarfRust’s `scripts/README.md` says they “live in `~/harfbuzz/src` and should be run from there”. The syllable machines are generated from Ragel sources, as in HarfBuzz, with a Ragel that writes Rust (`docs/ragel.md`).

The tests are HarfBuzz’s too. `scripts/gen-shaping-tests.py` reads HarfBuzz’s test files, runs HarfBuzz’s `hb-shape` on each case for the expected output (`:130`–`151`), and writes Rust tests: about 5,700 in `harfrust/tests/shaping/in_house.rs` alone. It skips a few whole files, among them `synthetic.tests` (“Not implemented”) and `harfbust.tests` (“We behave differently”, `:20`). [Testing HarfBuzz](testing-harfbuzz.html) describes those suites. `HARFBUZZ.md` runs them the other way round, through HarfBuzz’s backend with `HB_SHAPER_LIST=harfrust`. On 24 August 2026 that gave 800 of 800 AOTS tests, 437 of 437 text-rendering tests, and 5,012 of 5,021 in-house tests, the nine failures all Arabic fallback shaping, which 0.13.3 added the next day.

## Reading fonts without a sanitizer

HarfBuzz reads a table by laying C++ structs over its bytes. That is safe only because the sanitizer has first checked every offset and every length the code will follow, once, when the table is loaded; a table that fails is dropped whole ([Trust nothing: the sanitizer](trust-nothing-the-sanitizer.html)).

HarfRust reads through `read-fonts` instead, and there is no such pass. A table in `read-fonts` is a wrapper around a byte slice. Its README (`read-fonts/README.md:28`) describes what happens when it is parsed: “we perform _minimal validation_: that is, we ensure that the provided data is at least long enough to contain all of the required, non-version-dependent fields. All other validation occurs at runtime. If table data is malformed, and a field cannot be read, we will always return a default value.” Each read is bounds-checked where it happens. The basic one, trimmed:

```rust
// read-fonts/src/font_data.rs:124, trimmed
pub fn read_at<T: Scalar>(&self, offset: usize) -> Result<T, ReadError> {
    let end = offset.checked_add(T::RAW_BYTE_LEN).ok_or(ReadError::OutOfBounds)?;
    self.bytes.get(offset..end).and_then(T::read).ok_or(ReadError::OutOfBounds)
}
```

The bytes are not copied: records are “zerocopy types that can be cast from raw bytes” (`:54`), and the Fontations README says `read-fonts` “performs no allocation and no copying”, which is HarfBuzz’s own approach ([Fonts as bytes](fonts-as-bytes.html)). Rust’s slice indexing would stop an out-of-bounds read anyway, with a panic; `get` and `?` turn it into an ordinary error that the caller handles, typically by treating the item as missing. The price is a check at every read, where HarfBuzz pays once, when a table is loaded (inferred: neither project gives a figure for it). And the compiler holds the crates to it: `read-fonts` and `skrifa` begin with `#![forbid(unsafe_code)]` (`read-fonts/src/lib.rs:60`, `skrifa/src/lib.rs:97`), as does HarfRust (`harfrust/src/lib.rs:11`), whose README says “There are no `unsafe` in this library and in most of its dependencies (excluding `bytemuck`).”

The two designs agree on good fonts. On broken ones they part ways: HarfBuzz drops a table that fails, and HarfRust uses what it can still read. HarfBuzz’s `NEWS` for 12.3.0, when it began rejecting whole tables, says so itself: “This behavior is different from DirectWrite and HarfRust, and is in line with CoreText” ([HarfBust](trust-nothing-the-sanitizer.html#harfbust) tells the story). The fonts that [Trust nothing](trust-nothing-the-sanitizer.html#pass-or-fail) and [Fonts as bytes](fonts-as-bytes.html) break on purpose show it; [the comparison below](#running-this-books-examples-through-harfrust) has them. HarfRust’s README lists one more difference, from the time it was rustybuzz: “Malformed fonts will cause an error. HarfBuzz uses fallback/dummy shaper in this case.”

## What it covers

The README divides HarfBuzz into six parts and says what became of each (`README.md:82`):

1. shaping, ported to HarfRust;
2. Unicode routines, ported to HarfRust;
3. font parsing, handled by `read-fonts`;
4. subsetting, handled by `skera`;
5. custom containers and utilities, reimplemented in Fontations where appropriate;
6. glue for system and third-party libraries, not ported.

So the `harfrust` crate shapes and does nothing else. It doesn’t draw, paint, rasterize, or subset: those are other crates’ jobs. In a Rust text stack they come from Fontations, the collection `read-fonts` belongs to: `skrifa` draws outlines at a size, gives metrics, and hints them ([hb-raster](hb-raster.html#what-hb-raster-doesnt-do) covers it as a hinting engine), and `skera` subsets ([below](#drawing-hinting-and-subsetting)). Fontations also has `font-types`, `write-fonts`, `font-codegen`, `fauntlet`, and `incremental-font-transfer` (its `README.md`). It has no FreeType, Core Text, DirectWrite, ICU, or GLib integration and no Graphite (`README.md:31`). And, the README says, “Experimental HarfBuzz features like most of the boring-expansion-spec are not supported yet.”

The README also says “No font size property. Shaping is always using UnitsPerEm. You should scale the result manually.” That line is older than the code. 0.8.1 “Added internal scaling support”, and since 0.14.0 a scale is set on the font (`ShaperFont::set_scale`, `harfrust/src/shaper_font.rs:90`). What HarfRust has no notion of is pixels per em: nothing in its source reads a ppem, so GPOS Device tables, which [Fonts and font functions](fonts-and-font-functions.html) shows taking a pixel off a kerning pair at 12 ppem, have nothing to apply to. Nor does it slant mark offsets for a synthetically slanted font, as HarfBuzz’s GPOS does (`src/OT/Layout/GPOS/GPOS.hh:198`).

## The Rust API

The crate’s documentation is one sentence: “Create a `ShaperFont` from a `Font`, add Unicode text to a `Buffer`, and call `shape` to replace the text with positioned glyphs” (`harfrust/src/lib.rs:4`). The types, all re-exported from the crate root (`:76`–`106`):

| Type | What it is | HarfBuzz’s nearest |
|---|---|---|
| `Font` | a font at one location in its variation space; `read-fonts`’ font model, re-exported | `hb_face_t`, plus coordinates |
| `ShaperFont` | a short-lived view of a `Font` for shaping, with a scale and optional `FontFuncs` | `hb_font_t` |
| `FontFuncs` | a trait whose methods override character mapping, advances, origins, and extents | `hb_font_funcs_t` |
| `Buffer` | the text before shaping, glyphs and positions after; `ContentType` says which | `hb_buffer_t` |
| `GlyphInfo`, `GlyphPosition` | a glyph ID and cluster; advances and offsets | the same names |
| `Feature`, `Direction`, `Script`, `Language` | as in HarfBuzz, parsed from the same strings | the same names |
| `ShapeOptions` | features, an optional plan, and a point size for one call | the arguments of `hb_shape` |
| `ShapePlan`, `ShapePlanKey` | a reusable plan and its key, for an application’s own cache | `hb_shape_plan_t` |

`shape` (`harfrust/src/shape.rs:10`) returns a `Result`: it fails if the buffer already holds glyphs, or “cannot be shaped with the supplied plan or lacks the properties needed to build one”. Running out of budget or memory is reported as in HarfBuzz, by `Buffer::allocation_successful`.

A small program, written for this chapter (it is not HarfBuzz output, and isn’t run when the book is built). It shapes “IHV” at weight 700 with kerning off, in the variable test font of [Design space](design-space.html):

```rust
use harfrust::{shape, Buffer, Feature, Font, ShapeOptions, ShaperFont};

fn main() {
    let data = std::fs::read("var.ttf").expect("can't read the file");
    let font = Font::new(data, 0).expect("not a font");
    let bold = font.instance_builder().variations([("wght", 700.0)]).build();
    let shaper_font = ShaperFont::new(&bold);

    let mut buffer = Buffer::new();
    buffer.push_str("IHV");
    buffer.guess_segment_properties();

    let features: [Feature; 1] = ["-kern".parse().unwrap()];
    let options = ShapeOptions::new().features(&features);
    shape(&shaper_font, &mut buffer, options).expect("shaping failed");

    for (info, pos) in buffer.glyph_infos().iter().zip(buffer.glyph_positions()) {
        let name = shaper_font.glyph_name(info.glyph_id.into()).unwrap();
        println!("{}={}+{}", name, info.cluster, pos.x_advance);
    }
}
```

Built against HarfRust 0.16.1 and run on `fonts/variations/var.ttf`, it printed `I=0+370`, `H=1+668`, and `V=2+448`, the advances `hb-shape --variations=wght=700` gives. Without a scale, positions are in font units, as here.

The same `Font` type is the one Skrifa draws from. Skrifa’s documentation ends with “Need shaping?”: HarfRust “reads fonts through the same font model, so a `Font` can be shaped with HarfRust and drawn with a `ScalerFont`” (`skrifa/src/lib.rs:88`). HarfRust has a command-line tool too, `hr-shape`, described as “equivalent to HarfBuzz’s `hb-shape`” (`hr-shape/README.md`), with most of its options and the same output format.

## The C API

Since 0.14.0 HarfRust has a C library as well, [`harfrust_capi`](https://github.com/harfbuzz/harfrust/blob/main/harfrust_capi/README.md). It mirrors “the shaping, OpenType BASE baseline, and MATH query APIs of HarfBuzz with an `hr_` prefix in place of `hb_`. Every type, function, enumerator and constant is named and numbered to match its HarfBuzz counterpart, so C code can usually be ported by renaming `hb_` to `hr_`.” The prefix lets it be linked into the same process as HarfBuzz.

A header, `hr-hb.h`, maps HarfBuzz’s names onto HarfRust’s, so that “existing HarfBuzz shaping code builds unchanged”; calls it doesn’t cover fail to compile. Optional features add drawing and painting backed by Skrifa and subsetting backed by skera, through `hr-hb-draw.h`, `hr-hb-paint.h`, and `hr-hb-subset.h`. The README lists what is left out, among them custom Unicode funcs, buffer message callbacks, `hb_buffer_diff`, and JSON serialization, and the deliberate differences, such as “Allocation failure aborts” and “Misusing the shaping calls aborts”. Through `hr-hb.h`, `HB_VERSION_STRING` is the HarfBuzz release HarfRust matches, 15.2.0 (`harfrust_capi/cbindgen.toml:71`), though the README warns that this “does not imply that every API from that HarfBuzz release is available”.

## The harfrust backend in HarfBuzz

HarfBuzz can shape with HarfRust itself, as its `harfrust` shaper ([hb_shape and its backends](hb_shape-and-its-backends.html#harfrust) covers the glue). It is off unless HarfBuzz is configured with `-Dharfrust=enabled` (`meson_options.txt:30`), and then comes after `ot` in the shaper list (`src/hb-shaper-list.hh:61`), so it runs only when asked for: `hb-shape --shapers=harfrust`, or `HB_SHAPER_LIST=harfrust`.

Building it needs more than a Rust compiler. Meson runs bindgen over `hb.h` (`src/rust/meson.build:3`) and Cargo with the unstable flags `-Z build-std` and `--artifact-dir` (`:38`, `:66`), which only a nightly Rust accepts; `HARFBUZZ.md` gives the same requirements. The HarfRust it builds is whatever `src/rust/Cargo.toml:7` names, and in {{hb}} that is `harfrust = { version = "0.16.1" }`, which Cargo takes as 0.16.1 or any later 0.16 release. HarfRust 0.16.1 matches HarfBuzz 15.2.0, so {{hb}}’s backend runs the HarfRust in step with it. It hasn’t always: HarfBuzz 15.1.0 asked for 0.15.0, the release that matches 15.0.0, and ran a HarfRust one release behind. (`HARFBUZZ.md` says HarfBuzz uses HarfRust’s git main branch by default; in {{hb}} that line is commented out, `:8`.) This book’s `tools/build-harfbuzz.sh` builds it as `build-harfrust/`.

The backend gives HarfRust the HarfBuzz font’s answers. Its `FontFuncs` call `hb_font_get_nominal_glyph`, the advances, origins, and extents (`src/rust/shape.rs:120`–`198`), and it passes the scale, the point size, the variation coordinates, the buffer flags, the cluster level, and the context (`:220`, `:336`). The fonts’ tables come through `hb_face_reference_table` (`:66`), unsanitized. Since {{hb}} it passes the buffer’s not-found glyph too (`:374`), which HarfRust 0.16.1 added; it doesn’t pass the invisible glyph, or the font’s ppem or slant. Since 15.1.0, a HarfRust failure leaves the buffer marked unsuccessful, as `ot` would, rather than unshaped but apparently fine (`src/hb-harfrust.cc:175`).

## Drawing, hinting, and subsetting

HarfRust stops at positioned glyph IDs. In a Rust stack, Skrifa draws and hints those glyphs, and skera subsets the font. Shaping and drawing use one font object: `harfrust::font` and `skrifa::font` both re-export `read-fonts`’ font model (`harfrust/src/lib.rs:98`, `skrifa/src/font.rs:3`), so the `Font` of [the Rust API](#the-rust-api), already at its variation location, goes to Skrifa unchanged. skera works on `read-fonts`’ lower-level `FontRef` and writes its output with `write-fonts` (`skera/src/lib.rs:1433`, `skera/Cargo.toml`). In the C API, the `draw`, `paint`, and `subset` features (`harfrust_capi/Cargo.toml`) turn `hb_font_draw_glyph`, `hb_font_paint_glyph`, and `hb_subset_or_fail` into calls to Skrifa and skera (`harfrust_capi/include/hr-hb-draw.h:37`, `hr-hb-paint.h:25`, `hr-hb-subset.h:26`). There the shaping data isn’t shared: on first use, a face copies its rendering tables into a new SFNT for Skrifa (`harfrust_capi/src/rendering.rs:1`, `:47`), and the subsetter copies its input tables the same way (`SUBSETTING.md`).

### Drawing

Skrifa draws through a `ScalerFont`: a `Font` at a size in pixels per em, hinted or not, or in font units with no size (`skrifa/src/lib.rs:49`). It draws from `VARC`, `glyf`, `CFF2`, `CFF`, or Type 1, whichever comes first in that order (`:43`). An outline draws into a `PathSink`, a trait with `move_to`, `line_to`, `quad_to`, `curve_to`, and `close` (`read-fonts/src/model/glyph/outline/path.rs:11`): HarfBuzz’s draw funcs as a Rust trait ([Drawing outlines](drawing-outlines.html)). Continuing the program above (written for this chapter, not run):

```rust
use skrifa::{font::glyph::outline::SvgSink, DrawMemory, ScalerFont};

let scaler = ScalerFont::unhinted(bold.clone(), Some(16.0));
for info in buffer.glyph_infos() {
    if let Some(outline) = scaler.outline(info.glyph_id.into()) {
        let mut path = SvgSink::new();
        outline.draw(DrawMemory::Scratch, &mut path)?;
    }
}
```

Color glyphs work the same way. `font.color_glyphs().get(glyph)` returns a `COLR` glyph, version 0 or 1, and its `paint` method calls a `PaintSink`: push and pop transforms, clip to a glyph or a box, fill with a brush, push and pop layers with a composite mode (`read-fonts/src/model/glyph/color.rs:136`, `:365`). Those are HarfBuzz’s paint funcs ([Color](color.html)), except that the values are in font units and the sink does the scaling. Bitmap glyphs come back, its README says, “as coverage masks, BGRA pixels or PNG data for the caller to decode”.

Skrifa doesn’t rasterize: it gives paths, paint calls, and bitmap data, and no crate in Fontations’ README turns them into pixels as [hb-raster](hb-raster.html) does for HarfBuzz. The only rasterizer the README names is Chromium’s: “Fontations + Skia path rendering” (`README.md:195`).

`DrawSettings` and `OutlinePen`, which older Skrifa code uses and its `outline` module still documents, belong to the API that `ScalerFont` replaced, “kept so that existing code builds, and hidden from the documentation” (`skrifa/src/lib.rs:125`). The C API still uses it. `hr_font_draw_glyph_or_fail` draws unhinted with `DrawSettings`, at the font’s coordinates, with `PathStyle::HarfBuzz` (`harfrust_capi/src/draw.rs:575`). That path style picks a contour’s starting point the way hb-draw does when the first point is off the curve (`skrifa/src/outline/pen.rs:28`). Painting goes through the older `ColorPainter` (`harfrust_capi/src/paint/backend.rs:205`, `:500`).

### Hinting

Hinting is part of Skrifa’s `ScalerFont` too. [hb-raster](hb-raster.html#what-hb-raster-doesnt-do) describes its engines beside FreeType’s; in the API they are `HintOptions::interpreter()`, which runs the font’s TrueType instructions or CFF hints, `HintOptions::auto()`, the autohinter, and the default `auto_fallback()`, which uses the font’s own hints where it has any. A target is one of `Mono`, `Normal`, `Light`, `Lcd`, and `VerticalLcd` (`skrifa/src/scaler/hinting.rs:7`, `:87`–`108`). Hinted drawing asks for a size:

```rust
// Written for this chapter, not run.
use skrifa::{HintOptions, RasterTarget};

let options = HintOptions::interpreter().with_target(RasterTarget::Lcd);
let scaler = ScalerFont::hinted(bold.clone(), 16.0, options);
let metrics = scaler.outline(glyph_id).unwrap().draw(DrawMemory::Scratch, &mut path)?;
```

“Hinting can change a glyph’s advance, so hinted glyph metrics come from drawing the outline” (`skrifa/src/lib.rs:80`), and `metrics` holds that advance. HarfRust, with no ppem, never sees it; an application that wants hinted advances in its layout would supply them through `FontFuncs` (inferred: neither project shows it). The C API doesn’t hint: its README lists hinting among what isn’t exposed.

### Subsetting

skera is “a Rust library and binary for subsetting a font file according to provided input” (`skera/README.md`), at 0.10.0 here. Neither README calls it a port of hb-subset, but it follows hb-subset closely. Its source cites HarfBuzz’s about 85 times, as in “ref: <https://github.com/harfbuzz/harfbuzz/blob/a69b43c1f/src/hb-subset-plan.cc#L566>” (`skera/src/lib.rs:579`). Its default list of tables to drop is HarfBuzz’s, in the same order and with the same comments (`:209`; `src/hb-subset-input.cc:46`). Its name overrides are “matching HarfBuzz’s override API” (`:459`). A `Plan` is built from glyphs, Unicodes, flags, tables to drop, layout scripts and features, and name IDs and languages, and `subset_font` runs it (`:533`, `:1433`), as `hb_subset_plan_create_or_fail` and `hb_subset_plan_execute_or_fail` do ([Subsetting](subsetting.html)).

It subsets `glyf`, `CFF` and `CFF2`, `gvar`, `HVAR` and `VVAR`, the layout tables, `COLR` and `CPAL`, the bitmap tables, `MATH`, `BASE`, `cmap`, `name`, `OS/2`, `post`, and the metrics tables (`subset_table`, `:1590`, with CFF at `:1523`). What it lacks is instancing ([Instancing](instancing.html)). `fvar`, `avar`, and `cvar` pass through unchanged (`:1768`), and pinning axes appears only in TODO comments (`:1114`, `:1480`, `:1505`). The C API leaves out axis pinning too (`SUBSETTING.md`), and refuses any font with `CFF`, `CFF2`, or `VARC`, on the grounds that skera can’t rewrite them (`harfrust_capi/src/subset/mod.rs:362`), though skera’s README says it subsets both CFF tables.

## Speed

HarfRust has become much faster since it began, and is now very close to HarfBuzz:

* In June 2025 Behdad Esfahbod’s study had it “over 3x slower than HarfBuzz for major workloads”, and planned to port HarfBuzz’s caches. HarfBuzz’s study of those, [Caching](https://docs.google.com/document/d/1_VgObf6Je0J8byMLsi7HCQHnKo2emGnx_ib_sHo-bt4/preview), is linked from HarfRust’s README.
* In August 2025 HarfRust’s README said “HarfRust is less than 25% slower than HarfBuzz on most common fonts”, pointing to a spreadsheet kept over time.
* In November 2026 the README of 0.16.1 dropped the figure: “Recent comparisons show similar performance on common Latin workloads, with HarfRust faster on some workloads and HarfBuzz faster on others; there is no single percentage that describes all fonts and scripts.”
* In November 2026, Behdad Esfahbod said that earlier editions of this book had been underselling HarfRust performance: it is now “almost on par” with HarfBuzz, and “sometimes surpasses” it, depending on how the compiler inlines the code.

To measure it yourself, `HARFBUZZ.md` explains two ways. [benchmark-harf](https://github.com/harfbuzz/benchmark-harf) runs one C++ harness against HarfBuzz’s C API and HarfRust’s, so that both are called the same way. HarfBuzz’s own `benchmark-shape` runs both shapers through HarfBuzz, with `--benchmark_filter='/(ot|harfrust)$'`, which measures the backend’s glue as well. HarfRust’s own `cargo bench` compares it with HarfBuzz through harfbuzz_rs (`harfrust/benches/shaping.rs:100`). This book hasn’t measured speed.

## Running this book’s examples through HarfRust

Every line of every `shape` block in this book, 913 in all, was run through HarfBuzz {{hb}}’s `ot` shaper and through its `harfrust` backend, which in {{hb}} is HarfRust 0.16.1, the release in step with it, and separately through `hr-shape` from the same release. The lines were also run through HarfBuzz 15.1.0, to see what had changed on HarfBuzz’s side. The script, the counts, and every differing output are in `notes/harfrust-differences.md`. These runs were made by hand on 20 November 2026; nothing on this page is run when the book is built.

| Through the `harfrust` backend (HarfRust 0.16.1) | Lines |
|---|---|
| not comparable: the line chooses its own shaper, or lists them | 11 |
| identical to `ot` | 887 |
| an option the backend doesn’t pass on | 6 |
| real divergences | 9 |

None of the differences is lag anymore. With HarfBuzz 15.1.0, whose backend built HarfRust 0.15.0, four lines were HarfRust one release behind; with the backend and HarfRust in step, there are none. The six options are the invisible glyph, ppem, and synthetic slant (see [above](#the-harfrust-backend-in-harfbuzz)). The not-found glyph now reaches HarfRust, and synthetic bold always did, so those lines are identical.

The nine real divergences are fewer than in the same comparison with 15.1.0 and HarfRust 0.16.0. Two AAT bugs in HarfRust are fixed in 0.16.1: `smcp[1:2]` on “aaa” in a `morx` font made none of the *a*s small, because HarfRust looked up each glyph’s feature range by the cluster of the glyph at the buffer’s cursor, not the glyph’s own; and a `kerx` format 4 control-point action that can’t find its points attached the mark at its base’s origin, where HarfBuzz attaches nothing. One malformed font now agrees from HarfBuzz’s side: a null coverage offset no longer exhausts {{hb}}’s sanitizer budget ([Fonts as bytes](fonts-as-bytes.html)), so HarfBuzz keeps the table and applies the other lookups, as HarfRust did. What is left:

* **Seven malformed fonts.** The broken tables of [Trust nothing](trust-nothing-the-sanitizer.html#pass-or-fail), [Fonts as bytes](fonts-as-bytes.html), and the FeatureVariations font of [From features to lookups](from-features-to-lookups.html#lookup-variations). HarfBuzz drops the whole table; HarfRust applies what it can read. For `bad-gsub.ttf` and the text “ab”, HarfBuzz gives `[a=0+500|b=1+500]` and HarfRust `[a.alt=0+500|b=1+500]`: the broken ligature is lost, but the `calt` lookup still works.
* **An unsorted kerning pair set**, from the subsetting chapter’s [custom glyph map](subsetting.html). Both shapers binary-search an array the specification requires to be sorted, and reach different pairs. The font is invalid, so neither is wrong.
* **One in AAT.** A `kerx` subtable with tuples gives −80 in HarfBuzz and −100 in HarfRust, because HarfRust counts the tuple offset from the end of the subtable’s header, HarfBuzz from its start (`harfrust/src/aat/kerx.rs:215`; `src/hb-aat-layout-kerx-table.hh:118`). Apple’s specification doesn’t say which, so this is a disagreement rather than clearly a bug in either.

Here it is, as run:

| Font, text, options | HarfBuzz {{hb}} `ot` | HarfRust 0.16.1 |
|---|---|---|
| `tuples.ttf  AVTo` | `[A=0+560\|V=1@-40,0+560\|T=2+550\|o=3@-50,0+450]` | `[A=0+550\|V=1@-50,0+550\|T=2+550\|o=3@-50,0+450]` |

`hr-shape` 0.16.1 couldn’t run 30 lines, for options it lacks (`--shapers`, `--font-bold`, `--font-ppem`, `--invisible-glyph`, JSON output, and others) or a face index with named-instance bits. Of the 883 it ran, 870 were identical. Nine of the rest are the nine above. The other four differ only in metrics, since `hr-shape` uses HarfRust’s built-in metrics rather than HarfBuzz’s: it keeps the fraction of a variable advance, where HarfBuzz rounds it to a whole font unit first ([Where metrics come from](where-metrics-come-from.html)), and gives a glyph past the end of `hmtx`’s side bearings the shared advance, as the specification says, where HarfBuzz gives 0.

> For programmers: to see a difference for yourself, build HarfBuzz with `tools/build-harfbuzz.sh` and run the same line through `.cache/harfbuzz/build/util/hb-shape` and `.cache/harfbuzz/build-harfrust/util/hb-shape --shapers=harfrust`. Since both use the same font funcs, any difference is in the shaping.

## Where to read the source

| Where | What’s there |
|---|---|
| `src/hb-harfrust.cc`, `src/rust/shape.rs` | HarfBuzz’s side of the `harfrust` backend |
| `src/rust/Cargo.toml`, `src/rust/meson.build`, `meson_options.txt:30` | which HarfRust it builds, and how |
| harfrust: `README.md`, `CHANGELOG.md`, `HARFBUZZ.md` | what is ported, which HarfBuzz each release matches, testing and benchmarking against HarfBuzz |
| harfrust: `harfrust/src/lib.rs`, `shape.rs`, `shaper_font.rs`, `buffer.rs` | the Rust API |
| harfrust: `harfrust/src/ot/`, `aat/`, `normalize.rs` | the ported shaping |
| harfrust: `scripts/gen-shaping-tests.py`, `docs/ragel.md` | tests made from HarfBuzz’s, and the state machines |
| harfrust: `harfrust_capi/README.md`, `harfrust_capi/include/hr-hb.h` | the C API |
| harfrust: `hr-shape/` | the command-line tool |
| Fontations: `README.md`, `read-fonts/README.md`, `skrifa/README.md`, `skrifa/src/lib.rs` | the crates, and how `read-fonts` validates |
| Fontations: `read-fonts/src/font_data.rs` | bounds-checked reads |
| Fontations: `skrifa/src/scaler/`, `read-fonts/src/model/glyph/` | drawing, hinting, and painting |
| Fontations: `skera/README.md`, `skera/src/lib.rs` | subsetting |
| harfrust: `harfrust_capi/src/rendering.rs`, `draw.rs`, `paint/backend.rs`, `subset/mod.rs`, `SUBSETTING.md` | the C API’s drawing, painting, and subsetting |

## In short

* HarfRust is the HarfBuzz project’s Rust port of shaping, a fork of rustybuzz that reads fonts with Fontations’ `read-fonts`. HarfBuzz leads; each HarfRust release names the HarfBuzz release it matches, and 0.16.1 matches 15.2.0.
* Its code follows HarfBuzz’s file by file and function by function, its tables come from HarfBuzz’s generators, and its tests from HarfBuzz’s suites.
* It has no sanitizer. `read-fonts` checks every read as it is made, copies nothing, and has no `unsafe` code; on a broken font HarfRust uses what it can read where HarfBuzz drops the table.
* It shapes and nothing else: drawing, hinting, and subsetting belong to Skrifa and skera. It has a scale but no ppem, and a C API with HarfBuzz’s names.
* Skrifa draws, paints, and hints the same `Font` that HarfRust shapes, but doesn’t rasterize. skera follows hb-subset closely but doesn’t instance. The C API puts both behind HarfBuzz’s draw, paint, and subset calls, without hinting.
* HarfBuzz {{hb}} builds HarfRust 0.16.1, the release in step with it, as its `harfrust` shaper, which needs nightly Rust and bindgen, and gives it HarfBuzz’s own font funcs.
* Of the book’s 913 `shape` lines, 887 of 902 comparable ones came out the same through the backend. Six differences are options the backend doesn’t pass on, and nine are real: seven malformed fonts, an invalid one, and one AAT case.
