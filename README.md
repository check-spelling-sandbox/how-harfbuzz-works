# How HarfBuzz Works


*From Unicode to positioned glyphs, and everything in between*

Read it [online](https://lorp.github.io/how-harfbuzz-works/) or as a [PDF](https://lorp.github.io/how-harfbuzz-works/print/how-harfbuzz-works.pdf).

**How HarfBuzz Works** is a book about the inside of [HarfBuzz](https://github.com/harfbuzz/harfbuzz), the text shaping engine in Chrome, Firefox, Android, GNOME, LibreOffice and most other modern software that puts text on a screen. It follows text through HarfBuzz 15.2.0, from Unicode characters to positioned glyphs, and on to the outlines, color glyphs and subset fonts HarfBuzz also makes. It’s for type designers and font engineers who want to know why their fonts shape the way they do, and for programmers who use HarfBuzz, port it or work on it.

“I skimmed a few chapters. Genuinely useful to aspiring HarfBuzz developers.”  
— *Behdad Esfahbod*

The book, started 17 November 2026, is still a draft: 50 chapters in twelve parts, and seven appendices. So far only Laurence Penney and Behdad Esfahbod, HarfBuzz’s author, have reviewed it. If you find a mistake, whether a false claim, a misleading phrase or something missing, please [open an issue](https://github.com/Lorp/how-harfbuzz-works/issues), quoting the version in the page’s footer.

## How it’s made

The book was drafted by Claude, Anthropic’s AI model, working in Claude Code. Laurence Penney planned it, directed its revisions, checked it, and added numerous edits by hand. Not a one-shot, but hundreds of instructions:

* **Organization:** the order of the parts and chapters, which sections moved between them, and who each chapter is for.
* **Wording:** headings, sentences and captions rewritten, plain wording chosen over clever wording, and errors of fact caught, down to a miscounted glyph.
* **Style:** typographic quotes, American spelling, no padding, bold only for definitions.

The rules that came out of those interventions are kept in the repository, so they hold for later drafting: `AGENTS.md`, and the style guide and method in `notes/writing-brief.md`.

HarfBuzz’s source code is the main source. The book quotes it by file and line at the version it describes. The code shows what HarfBuzz does, but not whether that is standard, so the book also checks the OpenType specification and its registries, the ISO Open Font Format, Microsoft’s script development specifications, Apple’s TrueType Reference Manual, and HarfBuzz’s issues and pull requests, and says where HarfBuzz goes its own way. The [bibliography](https://lorp.github.io/how-harfbuzz-works/bibliography.html) lists these sources. `tools/check-tags.mjs` checks every tag the book quotes against the registries.

Anything the book shows as HarfBuzz’s output is made by running HarfBuzz when the book is built, not typed in:

* shaping results and `--trace` logs, from small test fonts built for each chapter from feature files;
* views of font tables, made from the compiled fonts;
* the output of HarfBuzz’s command-line tools;
* whole example programs in C, JavaScript, Python and Rust, compiled and run, with what they draw.

So if HarfBuzz changes, rebuilding the book shows what has changed. Writing it this way has turned up bugs in HarfBuzz, which `notes/upstream-bug-reports.md` records and the appendix “Bugs this book found” lists. Behdad has fixed many of them.

## Layout

* `book/toc.txt`: the order of the book, including pages planned but not yet written
* `book/front/`, `book/chapters/`, `book/appendices/`, `book/end/`: one Markdown file per page
* `book/book.json`: title, subtitle, edition, and the HarfBuzz version described
* `book/cover.png`, `book/cover-back.png`: the front and back covers, for the PDF
* `fonts/<page>/`: each chapter’s feature files, the fonts built from them, and `build.sh`
* `examples/`: whole programs, in C, JavaScript, Python and Rust, which the build compiles and runs
* `tools/`:
  * `build-harfbuzz.sh` builds HarfBuzz;
  * `build-site.mjs`, `build-pdf.mjs` and `publish.mjs` build and publish the book;
  * `build-test-font.py` and `font-views.py` make the test fonts and their views;
  * `check-tags.mjs` and `index-terms.mjs` check tags and gather index terms
* `notes/`: the writing brief, the outline, the bugs found, and research toward the book, such as the survey of the source
* `AGENTS.md`: instructions for the AI agents that work on the book
* `resources/` (not committed): local copies of reference material, such as the specifications

## Building

You’ll need:

* **Node and npm**, for the site;
* **`meson`, `ninja` and a C++ compiler**, for HarfBuzz itself (`brew install meson ninja` on a Mac);
* **a C compiler, Python 3 and cargo**, for the example programs;
* **Google Chrome, `qpdf` and `pdftotext`** (from Poppler), and the network, for the PDF.

```sh
tools/build-harfbuzz.sh   # HarfBuzz 15.2.0 into .cache/harfbuzz (twice: plain, and with detailed traces)
npm install               # once
npm run build             # the book as HTML, into site/
npm run pdf               # the book as a PDF, with its .pages.txt, into site/print/
```

## Publishing

Commit first, then publish:

```sh
git commit …              # the book's changes
npm run publish           # makes the PDF if it's out of date, builds the site afresh, and pushes both to gh-pages
```

Every page and the PDF carry a version: the edition from `book/book.json` (“Draft”), and the date and short commit the book was built from, such as “Draft, 18 November 2026 · 30688e6”. The site shows it in each page’s footer, linked to the commit; the PDF on its imprint page; and the colophon in both. A commit’s hash can’t be known until it’s made, so the PDF has to be made after committing: `npm run publish` makes it again whenever the one in `site/print/` wasn’t made from exactly the current commit and files, which takes a couple of minutes. It refuses to publish uncommitted changes in `book/`, `examples/`, `fonts/` or `tools/`; `npm run publish -- --dirty` publishes them anyway, and the version then ends “+ changes”. `npm run pdf` on its own still makes a PDF to look at.

GitHub Pages serves the `gh-pages` branch at <https://lorp.github.io/how-harfbuzz-works/>. The branch is replaced on each publish by a single commit, so it never grows; `main` holds only the source.

`tools/build-harfbuzz.sh` needs `meson`, `ninja` and a C++ compiler (`brew install meson ninja` on a Mac).

The test fonts are committed, so Python is only needed when a feature file changes. Then run the chapter’s `build.sh`, which needs Python 3 with `fonttools`:

```sh
fonts/gsub/build.sh       # a chapter's test fonts, and their FONT.views.json
```

Each `FONT.views.json` holds the views made from the compiled font: spec-like, TTX and bytes. It’s committed beside the font, and the site build stops with a message if a font has changed since its views were written.

## Writing a chapter

A chapter is ordinary Markdown, in `book/chapters/`. What makes this book different is that its examples aren’t typed in by hand: you describe what you want to show, and the build runs HarfBuzz to produce it. If HarfBuzz’s behavior changes, the book changes with it.

Each chapter has its own small test fonts in `fonts/<chapter>/`, named after the chapter’s file (so `gsub.md` uses `fonts/gsub/`). The kinds of code block below use them.

### Show what HarfBuzz does: `shape`

Write one font and one piece of text per line, and the build puts HarfBuzz’s output in the page:

````markdown
```shape
ligatures.ttf  ffi
ligatures.ttf  ffi  --features=-liga
```
````

Put **two or more spaces** between the font, the text and any `hb-shape` options. Text can include `\u0301` or `\u{1F600}` for characters that are hard to type.

### Show how it got there: `trace`

This shows HarfBuzz’s own step-by-step log (`hb-shape --trace`) for one run, cut down to the part you’re talking about:

````markdown
```trace
font: ligatures.ttf
text: ffi
from: start lookup 0
to: end lookup 0
```
````

`from` and `to` are bits of text from the log’s lines. The extract starts at the first line containing `from` and ends at the first line after it containing `to`. Lookups that were skipped are left out; add `skipped: yes` to keep them. Add `options:` for any other `hb-shape` options, such as `options: --script=arab`.

### Show what’s in the font: `views`

This shows one lookup from the compiled font, with tabs to switch between a spec-like table, TTX, and the raw bytes:

````markdown
```views
font: ligatures.ttf
lookup: 0
fea: ligatures.fea liga
cpp: LigatureSubstFormat1_2
```
````

`fea` names the feature file and the block in it the lookup came from, and `cpp` names the HarfBuzz struct that reads it. In the PDF only one view can be shown; add `print: spec fea` (for example) to choose which. The default is the spec-like table.

These views come from a `FONT.views.json` file next to each font, which the chapter’s `build.sh` writes. So after changing a feature file, rebuild that chapter’s fonts (see above); the site build stops and tells you if you forget.

### Show a command and what it gives: `command`

This shows one command line exactly as a reader would type it, runs it when the book is built, and shows what it produces under it: `hb-shape`’s output as text, or the SVG or PNG the command writes, captioned with the file’s name.

````markdown
```command
font: quick-start/LibreBaskerville.ttf
$ hb-raster LibreBaskerville.ttf "office coffee" --font-size=32 -o office.png
```
````

`font:` says where the font named in the command really is (found as in the other blocks); a path starting `test/` is a font in HarfBuzz’s own source. `zoom:` enlarges a PNG (default 2), and `show: no` shows the command without running it. The tools it can run are `hb-shape`, `hb-vector`, `hb-raster` and `hb-info`.

### Show a whole program: `example`

This shows a program from `examples/`, and what it outputs. The build compiles and runs it (C against the HarfBuzz in `.cache/harfbuzz`, JavaScript with npm, Python in a virtual environment from its `requirements.txt`, Rust with cargo, all built in `.cache/examples/`), so the listing and the picture can’t disagree:

````markdown
```example
program: quick-start/c/text-to-svg.c
args: LibreBaskerville.ttf  office coffee
output: svg
```
````

`args` are the program’s arguments, with two or more spaces between them; the first is a font, found as in the other blocks (`hb:` means HarfBuzz’s own source tree). `output` is `svg`, `png` or `text`. For `text`, a line `same-as: hb-shape` (with any `hb-shape` options after it) makes the build fail unless the program prints exactly what `hb-shape` prints for the same font and text. Add `show: source` for the listing alone, or `show: output` for the picture alone. Outputs are kept in `.cache/examples/out/` and rerun only when the program, its `package.json` or `Cargo.toml`, or the arguments change.

### Let the reader try it: `demo`

This runs a JavaScript module from `examples/` in the reader’s browser, with a box to type text into. The module exports `drawText(canvas, fontData, text, pixels)`; the font is cut down with `hb-subset` and carried in the page, and the package comes from jsDelivr. Demos appear on screen only, so the text around one should still make sense in print.

````markdown
```demo
program: quick-start/js/text-to-canvas.mjs
font: LibreBaskerville.ttf
unicodes: 20-7E
text: office coffee
package: harfbuzzjs@1.6.4
```
````

### Show the order features run in: `order`

This shapes a pair of fonts from `fonts/feature-order/`, which have one lookup for every feature HarfBuzz’s shapers ask for, numbered in opposite orders, and shows the stages HarfBuzz runs them in, with the reorderings and other steps between them. It is what the appendix “Feature order” is made of:

````markdown
```order
fonts: order-dev2
text: कि
options: --features=smcp
```
````

The other blocks, `raster`, `draw`, `info` and `table`, are described with the rest at the top of `tools/build-site.mjs`.

`tools/build-site.mjs` has the full details at the top.
