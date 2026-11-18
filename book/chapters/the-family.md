# The family

The rest of this book has looked inside one program. This chapter looks around it: at the Rust port that the HarfBuzz project now maintains next to the C++ library, at a few smaller relatives, and at how some of the best-known programs that use HarfBuzz actually call it.

Everything here is about software other than HarfBuzz {{hb}}, so none of it can be checked against the code in `.cache/harfbuzz/` or shown live: the book’s build can’t run a browser. (HarfRust, which the book does build, has a chapter of its own: [HarfRust](harfrust.html).) Each claim is taken from the other project’s own repository, documentation or announcement, linked where it is made, and read in November 2026. Repositories move on, so a version number given here is the one that was there when it was read.

## HarfRust

HarfRust is, in its [README](https://github.com/harfbuzz/harfrust)’s words, “a Rust port of HarfBuzz text shaping engine”: the HarfBuzz project’s own port of shaping, which began as a fork of rustybuzz and reads fonts with Fontations’ `read-fonts` crate. Each release names the HarfBuzz release it matches, and HarfBuzz can call it as its `harfrust` shaper. [HarfRust](harfrust.html) covers its history, how it is kept in step, its APIs, the Fontations crates, and every example in this book run through it.

## Other relatives

`fontations` is the font-funcs backend that reads fonts with Fontations’ Rust crates ([Fonts and font functions](fonts-and-font-functions.html#the-other-funcs)). The crates are a project of their own at Google Fonts. Their [README](https://github.com/googlefonts/fontations) describes `read-fonts` as “intended to be a high performance parser, suitable for shaping”, one that “performs no allocation and no copying”, which is HarfBuzz’s own approach ([Fonts as bytes](fonts-as-bytes.html)), and says of Skrifa: “It’s primary purpose is to replace FreeType in Google applications.” The HarfBuzz README sends readers who need hinting to “FreeType or Skrifa”.

Several bindings live in the HarfBuzz organization on GitHub ([Integrations](integrations.html#other-language-bindings) covers bindings as a way to use HarfBuzz):

* **harfbuzzjs** offers “HarfBuzz shaping library for client/server side JavaScript projects” ([README](https://github.com/harfbuzz/harfbuzzjs)). It is HarfBuzz compiled to WebAssembly with Emscripten, published on npm, and its API keeps HarfBuzz’s objects: blob, face, font, buffer, `shape`.
* **uharfbuzz** is “Streamlined Cython bindings for the HarfBuzz shaping engine” for Python ([README](https://github.com/harfbuzz/uharfbuzz)). It normally builds its own copy of HarfBuzz, and with `USE_SYSTEM_LIBS=1` uses the system’s. The HarfBuzz README points Python users to it, and to GObject introspection (`README.python.md`). This book’s tools read fonts with fontTools and run `hb-shape` itself, not uharfbuzz.
* **harfbuzz_rs** is a Rust interface over HarfBuzz’s C library, as opposed to a port of it like [HarfRust](harfrust.html). HarfRust uses it to benchmark against HarfBuzz (HARFBUZZ.md).

## How applications call HarfBuzz

HarfBuzz’s README lists users from Android and Chrome to car displays (`README.md:174`). Among them are some of the most widely used programs for setting type: Adobe Photoshop, Illustrator and InDesign, QuarkXPress and Scribus, as well as Microsoft Edge, Figma, Canva, Amazon’s Kindle, the PlayStation, Flutter and the Unreal game engine. Most of those are closed source, so how they call HarfBuzz can’t be examined. What follows is the open-source ones whose code could be read, and only what their source shows: whether they keep their own copy of HarfBuzz (**vendoring**: copying its source into their tree and building it with their own code), which version that was, which font funcs they give it, which shapers they ask for, and which [cluster level](buffers.html#the-four-cluster-levels) they set.

| Project | Copy of HarfBuzz | Version read | Font funcs | Shapers asked for | Cluster level |
|---|---|---|---|---|---|
| Chromium (Blink) | vendored | 15.1.0 and 148 commits | callbacks into Skia | `ot`, or `harfrust` in tests | default |
| Skia (SkShaper) | through its DEPS | a commit, not a release | hb-ot, with a sub-font calling Skia | default | monotone characters |
| Firefox | vendored | 15.1.0 | its own, reading tables | default | monotone characters |
| Android (minikin) | vendored | 10.2.0 in AOSP | a sub-font calling Skia for advances | default | default |
| Pango | a dependency, or built from a wrap | at least 11.0.0 | | | |
| LibreOffice | bundled tarball, or system | 15.1.0 | hb-ot, with a sub-font for `cmap` | `graphite2`, `ot`, `fallback` | characters |
| Qt | bundled, or system | 15.1.0 | its own, calling the font engine | `graphite2`, `ot`, `fallback` | default |
| OpenJDK | vendored | 15.0.0 | its own, calling Java’s font code | default | monotone characters |
| Godot | vendored | 15.0.0 | hb-ft (FreeType) | default | default |

“Default” in the last column means the code read never sets a cluster level, so HarfBuzz’s level 0 applies. Blank cells are questions the source read for this chapter didn’t answer.

### Chrome and Skia

Chromium keeps HarfBuzz in `third_party/harfbuzz`. Its [README.chromium](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/harfbuzz/README.chromium) gives the version as `15.1.0-148`, rolled on 16 November 2026: Chromium follows HarfBuzz’s main branch, not its releases. It lists files it doesn’t build, among them the Core Text, DirectWrite and Uniscribe backends, “as we rely on HarfBuzz’ built-in AAT shaping”.

The [BUILD.gn](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/harfbuzz/BUILD.gn) is a worked example of [compile-time configuration](compile-time-configuration.html). It uses ICU for Unicode data (`HB_NO_UCD`, with `HAVE_ICU`). It leaves out the `fallback` backend: “Fallback shaper not required, we only use the HarfBuzz internal OT shaper.” It drops drawing, painting, buffer messages and verification, and the experimental formats (`HB_NO_BEYOND_64K`, `HB_NO_CUBIC_GLYF`, and `HB_NO_VAR_COMPOSITES` with the comment “Don’t ship VARC yet”). It keeps the subsetter, because Skia’s PDF output needs it, but without layout or CFF subsetting unless Paint Previews are built. It defines `HAVE_FONTATIONS`, and in [harfbuzz.gni](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/harfbuzz/harfbuzz.gni) `enable_harfrust` is true, so Chromium builds HarfRust 0.15 into HarfBuzz as the `harfrust` backend.

Blink’s [harfbuzz_shaper.cc](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/platform/fonts/shaping/harfbuzz_shaper.cc) decides between the two engines on each call to `hb_shape_full` (trimmed):

```c++
const char* const harfrust_shaper_list[] = {"harfrust"};
const char* const ot_shaper_list[] = {"ot"};

inline const char* const* ShapingBackend() {
  return RuntimeEnabledFeatures::HarfRustShapingEnabled() ? harfrust_shaper_list
                                                          : ot_shaper_list;
}
```

The `HarfRustShaping` feature has the status `test` in [runtime_enabled_features.json5](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/platform/runtime_enabled_features.json5), which the file defines as “Enabled in ContentShell for testing, otherwise off.” So Chrome shapes with `ot`, and the Rust port runs in Blink’s test shell.

Blink’s font funcs, in [harfbuzz_face.cc](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/platform/fonts/shaping/harfbuzz_face.cc), answer the nominal and variation glyph, advance, vertical origin and extents callbacks from Skia, and it gets table data from the Skia typeface. On Apple platforms one exception is made: for a font with a `trak` table and no `sbix`, the horizontal advances are left to HarfBuzz. Since the `harfrust` backend asks the HarfBuzz font for these answers, the same callbacks serve either engine (inferred, from [hb_shape and its backends](hb_shape-and-its-backends.html)).

Below Blink, Skia’s own text layout module has a HarfBuzz shaper, [SkShaper_harfbuzz.cpp](https://github.com/google/skia/blob/main/modules/skshaper/src/SkShaper_harfbuzz.cpp). It builds a face from the typeface’s bytes, or through a table callback when it can’t, makes a font with hb-ot funcs and the typeface’s variation coordinates, and then puts a sub-font on top whose funcs ask Skia for nominal glyphs, advances and extents. Its comment is the [parents and sub-fonts](fonts-and-font-functions.html#parents-and-sub-fonts) model in one line: “Creating a sub font means that non-available functions are found from the parent.” It shapes at `HB_BUFFER_CLUSTER_LEVEL_MONOTONE_CHARACTERS`.

What Skia reads fonts with is changing. Chrome’s blog post [Memory safety for web fonts](https://developer.chrome.com/blog/memory-safety-fonts) (19 March 2025) describes a Skia font backend based on Skrifa, used for web fonts on Linux, Android and ChromeOS from Chrome 133. A note added in 2026 says it was enabled for system fonts on Linux and Android in Chrome 136, and that “In Chrome 145 we removed FreeType from Blink.”

### Firefox

Firefox keeps HarfBuzz in `gfx/harfbuzz`. Its [moz.yaml](https://github.com/mozilla-firefox/firefox/blob/main/gfx/harfbuzz/moz.yaml) gives the release, 15.1.0, tracks HarfBuzz by tag, and leaves out the Core Text, DirectWrite, `harfrust` and `kbts` backends and the paint, raster and vector code.

[gfxHarfBuzzShaper.cpp](https://github.com/mozilla-firefox/firefox/blob/main/gfx/thebes/gfxHarfBuzzShaper.cpp) gives HarfBuzz its own Unicode funcs (`mozilla_harfbuzz_glue_set_up_unicode_funcs`) and its own font funcs: nominal and variation glyphs, horizontal and vertical advances, vertical origin, extents, contour points, font extents, and `glyph_h_kerning`, for which it reads `kern` tables itself. That is the callback HarfBuzz’s [fallback kerning](fallbacks.html#kerning-through-the-font-funcs) asks. For a font with a `CFF ` table it does less: it makes a sub-font that supplies only the nominal glyph and font extents, and leaves the rest to the parent. It sets the scale in 16.16 fixed point from the font size, passes the variation settings, and shapes at the monotone-characters level.

### Android

Android’s copy is `external/harfbuzz_ng`. In AOSP’s public main branch its [METADATA](https://android.googlesource.com/platform/external/harfbuzz_ng/+/refs/heads/main/METADATA) gives version 10.2.0, last upgraded in January 2025, and its [README.android](https://android.googlesource.com/platform/external/harfbuzz_ng/+/refs/heads/main/README.android) notes that “Chromium updates Harfbuzz more frequently”.

The text layout library minikin calls it in [LayoutCore.cpp](https://android.googlesource.com/platform/frameworks/minikin/+/refs/heads/main/libs/minikin/LayoutCore.cpp). For each font it makes a sub-font, because “We override some functions which are not thread safe.” The funcs on it supply advances and origins from Skia, for a stated reason: “we can’t use HarfBuzz’s implementation. It may return the wrong value if the font uses hinting aggressively.” Color bitmap fonts (those with `CBDT`) get funcs that keep HarfBuzz’s own advances, “for performance reasons”.

### Pango and GNOME

Pango’s [NEWS](https://github.com/GNOME/pango/blob/main/NEWS) (read on the GitHub mirror) records the whole relationship. Version 1.13.0 “Moved the OpenType Layout code into a new project called HarfBuzz”. 1.31.0 ported Pango to HarfBuzz 0.9, used it as an external library, and removed Pango’s Arabic, Syriac, Hangul, Hebrew, Khmer, Tibetan, Thai and Indic shapers as “replaced by harfbuzz”. 1.44.0 said “Use harfbuzz for shaping on all platforms” and stopped using FreeType for font loading, which “drops support for type1 and bitmap fonts”. 1.44.4 switched to HarfBuzz for metrics too. Pango links HarfBuzz as an ordinary dependency: its [meson.build](https://github.com/GNOME/pango/blob/main/meson.build) asks for `harfbuzz >= 11.0.0`, and a Meson wrap file in `subprojects/` points at HarfBuzz 14.4.0 for building it from source.

### LibreOffice

LibreOffice’s [download.lst](https://github.com/LibreOffice/core/blob/master/download.lst) names `harfbuzz-15.1.0.tar.xz` among the third-party tarballs. In [LogicalFontInstance.cxx](https://github.com/LibreOffice/core/blob/master/vcl/source/font/LogicalFontInstance.cxx) it creates a font with hb-ot funcs, its scale set to the font’s units per em, and passes variations. When it slants a font artificially it tells HarfBuzz to do the same, “so that mark positioning is also transformed” ([synthetic slant](fonts-and-font-functions.html#slant)). On top it puts a sub-font that replaces only the nominal and variation glyph callbacks, so that a hyphen the font has no glyph for can use the minus sign’s glyph instead (bug tdf#171555).

[CommonSalLayout.cxx](https://github.com/LibreOffice/core/blob/master/vcl/source/gdi/CommonSalLayout.cxx) shapes each run with an explicit list, “in the order of preference”: `{ "graphite2", "ot", "fallback", nullptr }`. It sets the BOT and EOT flags at the ends of the text, asks for `HB_BUFFER_FLAG_PRODUCE_SAFE_TO_INSERT_TATWEEL`, which marks where a tatweel may be inserted ([Buffers](buffers.html#glyph-flags)), and uses the characters cluster level, not monotone characters.

### Qt

Qt bundles HarfBuzz in `src/3rdparty/harfbuzz-ng`. Its [qt_attribution.json](https://github.com/qt/qtbase/blob/dev/src/3rdparty/harfbuzz-ng/qt_attribution.json) gives version 15.1.0, and says HarfBuzz is “Optionally used in Qt GUI”, with `-system-harfbuzz` or `-qt-harfbuzz` choosing between the system library and the bundled copy. Its copyright list still includes “Nokia Corporation and/or its subsidiary(-ies)”, from the history in [chapter 1](why-shaping-exists.html#freetype-pango-and-qt-the-first-harfbuzz).

[qharfbuzzng.cpp](https://github.com/qt/qtbase/blob/dev/src/gui/text/qharfbuzzng.cpp) builds the face from a table callback into Qt’s font engine, and gives HarfBuzz Qt’s own Unicode funcs and font funcs, including `glyph_h_kerning` and `glyph_contour_point`. [qtextengine.cpp](https://github.com/qt/qtbase/blob/dev/src/gui/text/qtextengine.cpp) passes the same shaper list as LibreOffice, `graphite2`, `ot`, `fallback`, under the comment “whitelist cross-platforms shapers only”, and sets `HB_BUFFER_FLAG_PRESERVE_DEFAULT_IGNORABLES` for symbol fonts or when the application asks to show default ignorables.

### OpenJDK

Java took HarfBuzz in JDK 9. [JEP 258](https://openjdk.org/jeps/258) proposed to “Replace the existing ICU OpenType font-layout engine with HarfBuzz”, because ICU’s layout engine “is no longer actively developed and IBM, the project owner, intends to replace it with HarfBuzz in the ICU project”. It found “no viable open-source alternatives to HarfBuzz”. The JDK keeps its copy in `src/java.desktop/share/native/libharfbuzz`, at 15.0.0 according to its [legal notice](https://github.com/openjdk/jdk/blob/master/src/java.desktop/share/legal/harfbuzz.md).

Its glue, [hb-jdk-font.cc](https://github.com/openjdk/jdk/blob/master/src/java.desktop/share/native/libfontmanager/hb-jdk-font.cc), builds the face from a table callback and gives HarfBuzz a full set of font funcs that call back into Java’s font code, including kerning, contour points and glyph names. [HBShaper.c](https://github.com/openjdk/jdk/blob/master/src/java.desktop/share/native/libfontmanager/HBShaper.c) shapes at the monotone-characters level, with no shaper list.

### Godot

The Godot game engine vendors HarfBuzz in `thirdparty/harfbuzz`. Its [thirdparty/README.md](https://github.com/godotengine/godot/blob/master/thirdparty/README.md) gives version 15.0.0, and lists what it leaves out: among others `hb-harfrust.cc`, `rust/*`, `hb-wasm*` and `hb-gpu*`. Its advanced text server, [text_server_adv.cpp](https://github.com/godotengine/godot/blob/master/modules/text_server_adv/text_server_adv.cpp), is the only project here that uses HarfBuzz’s FreeType funcs, `hb_ft_font_create`, and asks HarfBuzz for synthetic bold when a font is emboldened. For its bitmap fonts it supplies font funcs of its own.

### What they have in common

Most of these programs let HarfBuzz read the layout tables but answer some of its questions about glyphs themselves, from the code that also renders them: Skia, Android and LibreOffice with a sub-font over hb-ot ([Fonts and font functions](fonts-and-font-functions.html#parents-and-sub-fonts)), Firefox, Qt and OpenJDK with whole sets of funcs. Android’s comment gives one reason, hinted advances; keeping shaping and drawing in agreement is the general one (inferred). The platform backends hardly appear: Chromium stopped building Core Text in favor of HarfBuzz’s own AAT code, Firefox leaves the Core Text and DirectWrite files out, and Qt and LibreOffice name only cross-platform shapers. And while Chromium builds HarfRust in and tests with it, Firefox and Godot leave `hb-harfrust.cc` out of their copies.

## Where to read the source

| Where | What’s there |
|---|---|
| `src/hb-harfrust.cc`, `src/rust/shape.rs`, `src/rust/Cargo.toml` | HarfBuzz’s side of the `harfrust` backend, and the HarfRust version it builds |
| `src/rust/font.rs` | the `fontations` font funcs |
| `NEWS`, `README.md`, `README.python.md` | HarfBuzz’s record of the HarfRust backend, its list of users, and its Python bindings |
| harfrust: `README.md`, `CHANGELOG.md`, `HARFBUZZ.md`, `harfrust_capi/README.md` | what HarfRust ports, which HarfBuzz each release matches, how it is tested against HarfBuzz, and its C API |
| Chromium: `third_party/harfbuzz/`, `third_party/blink/renderer/platform/fonts/shaping/` | Chromium’s copy and build of HarfBuzz, and Blink’s font funcs and shaper |
| Skia: `modules/skshaper/src/SkShaper_harfbuzz.cpp` | Skia’s HarfBuzz shaper |
| Firefox: `gfx/harfbuzz/moz.yaml`, `gfx/thebes/gfxHarfBuzzShaper.cpp` | Firefox’s copy and its shaper |
| Android: `external/harfbuzz_ng/`, `frameworks/minikin/libs/minikin/LayoutCore.cpp` | Android’s copy, and minikin’s calls |
| LibreOffice: `vcl/source/font/LogicalFontInstance.cxx`, `vcl/source/gdi/CommonSalLayout.cxx` | LibreOffice’s fonts and shaping |
| Qt: `src/gui/text/qharfbuzzng.cpp`, `src/gui/text/qtextengine.cpp` | Qt’s glue and shaping |
| OpenJDK: `src/java.desktop/share/native/libfontmanager/` | the JDK’s glue and shaping |

## In short

* HarfRust is the HarfBuzz project’s Rust port of shaping. It began as a fork of rustybuzz that reads fonts with Fontations instead of ttf-parser, and is now developed with Google Fonts.
* HarfBuzz leads and HarfRust follows: each HarfRust release names the HarfBuzz release it matches, and 0.16.0 matches 15.1.0. HarfBuzz 15.1.0 itself builds HarfRust 0.15.
* HarfRust ports shaping and Unicode, and leaves font parsing, subsetting and drawing to the Fontations crates. It has no ppem (it scales with `ShaperFont::set_scale`), no platform integrations and no Graphite, and it now has a C API with HarfBuzz’s names.
* Most large users vendor HarfBuzz: Chromium follows its main branch, and Firefox, Qt and LibreOffice were on 15.1.0 when read. Pango takes it as an ordinary dependency.
* Most of them give HarfBuzz font funcs that answer from their own rendering code, often on a sub-font over hb-ot, and shape with `ot`. Chromium builds the `harfrust` backend but uses it only in tests.

HarfBuzz {{hb}} is one release of a program that changes every few weeks, and some of what this book describes will have changed by the time it is read. The book’s examples run when it is built, so building it again against a newer HarfBuzz will show what has changed.
