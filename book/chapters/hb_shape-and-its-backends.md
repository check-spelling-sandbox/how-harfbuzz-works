# hb_shape and its backends

An application that wants glyphs calls `hb_shape`. It hands over a font, a buffer of Unicode text whose script, direction and language are set, and perhaps some features, and it gets the buffer back full of positioned glyphs. All of the shaping this book describes happens inside that call, but `hb_shape` itself does very little. It finds a shape plan, and the plan names a **backend**: a complete shaping engine with one entry point, which does the actual work. HarfBuzz can be built with up to nine of them.

In practice one backend, `ot`, does nearly all the shaping in the world. It is HarfBuzz’s own engine for OpenType and AAT fonts, the one that Parts III to VI take apart. The others are there for particular fonts (Graphite fonts, and fonts that carry their own WebAssembly shaper), for comparison with other engines (HarfRust, Core Text, DirectWrite, Uniscribe, kb_text_shape), and as a last resort (`fallback`). This chapter is about how the choice between them is made, why it almost always comes out as `ot`, what each of the others is and gives up, and the small piece of machinery that lets every backend keep data of its own on faces and fonts.

The examples use one small test font, `backends.ttf`, built from `fonts/hb_shape-and-its-backends/backends.fea`. It has an *fi* ligature, a kerning pair `A V`, a combining acute with an anchor on `e` (but no precomposed `é`), and the Arabic letter beh in its four joining forms. Its glyphs are boxes. Two fonts from HarfBuzz’s own test suite appear too. Every result on this page was produced by running HarfBuzz {{hb}} when the page was built, with the book’s build of it, which has none of the optional backends compiled in.

## Two things called a shaper

HarfBuzz uses the word “shaper” for two different things, and it helps to separate them first.

A **backend shaper** is one of the entries in `hb-shaper-list.hh`: `ot`, `fallback`, `graphite2` and the rest. Each is a function with the same signature, `hb_shape_func_t` (`hb-shaper.hh:33`), which takes a plan, a font, a buffer and the user’s features and does all the shaping. This chapter is about these.

A **script shaper** is one of the parts of the `ot` backend that knows a family of scripts: the Arabic shaper, the Indic shaper, the Universal Shaping Engine. They are `hb_ot_shaper_t` structs (`hb-ot-shaper.hh`), and they only exist inside `ot`. [Why shaping exists](why-shaping-exists.html#runs-and-the-three-things-harfbuzz-must-be-told) introduced them, and [Choosing a script shaper](choosing-a-script-shaper.html) explains how one is picked. None of the other backends uses them.

The source keeps the two apart by name: backend files are `hb-<name>-shape.cc` or `hb-<name>.cc`, script shapers are `hb-ot-shaper-<name>.cc`. This chapter says “backend” for the first kind wherever there could be doubt.

## hb_shape and hb_shape_full

`hb_shape` is a one-line wrapper (`hb-shape.cc:185`). It calls `hb_shape_full` with no shaper list, and throws away the result:

```c++
void
hb_shape (hb_font_t *font, hb_buffer_t *buffer,
          const hb_feature_t *features, unsigned int num_features)
{
  hb_shape_full (font, buffer, features, num_features, nullptr);
}
```

`hb_shape_full` (`hb-shape.cc:127`) is the real entry point. Here it is with the verification details trimmed:

```c++
hb_bool_t
hb_shape_full (hb_font_t *font, hb_buffer_t *buffer,
               const hb_feature_t *features, unsigned int num_features,
               const char * const *shaper_list)
{
  if (unlikely (!buffer->len))
    return true;

  hb_buffer_t *text_buffer = nullptr;
  if (buffer->flags & HB_BUFFER_FLAG_VERIFY)
  { /* keep a copy of the text, to check the result against */ }

  hb_shape_plan_t *shape_plan = hb_shape_plan_create_cached2 (font->face, &buffer->props,
                                    features, num_features,
                                    font->coords, font->has_nonzero_coords ? font->num_coords : 0,
                                    shaper_list);

  hb_bool_t res = hb_shape_plan_execute (shape_plan, font, buffer, features, num_features);

  hb_shape_plan_destroy (shape_plan);

  if (text_buffer)
  { /* buffer->verify (): reshape in pieces and compare */ }

  return res;
}
```

That is the whole of it. There are two steps: get a plan, and run it.

The plan comes from a cache on the face, keyed by the buffer’s segment properties, the user’s features, the font’s variation coordinates (only when some are non-zero, so a variable font at its default shares plans with a static one) and the shaper list. What a plan holds, and how the cache compares keys, is the subject of [Shape plans and their cache](shape-plans-and-their-cache.html). What matters here is one field of the plan’s key: `shaper_func`, the backend that will run it (`hb-shape-plan.hh`). The backend is chosen when the plan is made, and every later call that finds the same plan in the cache uses the same backend without choosing again.

Running the plan, `hb_shape_plan_execute`, calls that one backend. If it succeeds and the buffer still says it holds Unicode, its content type is set to glyphs (`hb-shape-plan.cc:478`).

The buffer has to be ready before any of this. `_hb_shape_plan_execute_internal` asserts that it holds Unicode text (`hb-shape-plan.cc:421`), and a plan can’t be made for a buffer whose direction isn’t set: `hb_shape_plan_create2` returns the empty plan (`:255`), which can’t be executed, so shaping fails. That is why applications call `hb_buffer_guess_segment_properties` or set the properties themselves. [Buffers](buffers.html) covers both.

> For programmers: `hb_shape` returns nothing. If you need to know whether shaping worked, call `hb_shape_full` with a null shaper list, which does exactly what `hb_shape` does, and check its result *and* `hb_buffer_allocation_successful`. As the section on failure below explains, the `ot` backend always returns true and reports trouble only through the buffer.

## The shaper list

Every backend HarfBuzz knows about is named once, in `hb-shaper-list.hh`. Without its license header, the whole file is this:

```c++
#ifndef HB_NO_SHAPER

/* v--- Add new shapers in the right place here. */

#ifdef HAVE_WASM
/* Only picks up fonts that have a "Wasm" table. */
HB_SHAPER_IMPLEMENT (wasm)
#endif

#ifdef HAVE_GRAPHITE2
/* Only picks up fonts that have a "Silf" table. */
HB_SHAPER_IMPLEMENT (graphite2)
#endif

#ifndef HB_NO_OT_SHAPE
HB_SHAPER_IMPLEMENT (ot) /* <--- This is our main shaper. */
#endif

#ifdef HAVE_UNISCRIBE
HB_SHAPER_IMPLEMENT (uniscribe)
#endif
#ifdef HAVE_DIRECTWRITE
HB_SHAPER_IMPLEMENT (directwrite)
#endif
#ifdef HAVE_CORETEXT
HB_SHAPER_IMPLEMENT (coretext)
#endif

#ifdef HAVE_HARFRUST
HB_SHAPER_IMPLEMENT (harfrust)
#endif

#ifdef HAVE_KBTS
HB_SHAPER_IMPLEMENT (kbts)
#endif

#ifndef HB_NO_FALLBACK_SHAPE
HB_SHAPER_IMPLEMENT (fallback) /* <--- This should be last. */
#endif

#endif
```

The file has no header guard of its own (its guard lines are marked “Dummy header guards”). It is an *X-macro* list: other files define `HB_SHAPER_IMPLEMENT` to mean something, include the list, and undefine it again, so the list is expanded into whatever code they need. It is included this way eight times:

| Where | `HB_SHAPER_IMPLEMENT(name)` becomes |
|---|---|
| `hb-shaper.hh:39` | a declaration of the backend’s entry point, `_hb_name_shape` |
| `hb-shaper.hh:61` | an enumerator `HB_SHAPER_ORDER_name`, numbering the backends from 1 |
| `hb-shaper.hh:117`, `:123`, `:129` | a lazy loader for the backend’s data, and its `init0` and `fini` |
| `hb-face.hh:43`, `hb-font.hh:107` | the types and create/destroy functions of that data, for faces and for fonts |
| `hb-shaper.cc:33` | an entry `{"name", _hb_name_shape}` in the array `_hb_all_shapers` |
| `hb-shape-plan.cc:129`, `:142` | a test in the loops that choose a backend |
| `hb-shape-plan.cc:444` | a branch in the code that runs the chosen one |

So adding a backend means adding one line here, writing its shape function and its four data functions, and nothing else. The order of the lines is the order of preference, which is why the comment says “in the right place”.

Each line is guarded by a build option, and only two are on by default. HarfBuzz is built with [Meson](building-harfbuzz.html#meson-and-ninja), which reads its options from `meson_options.txt`. There the options `wasm`, `graphite2`, `harfrust`, `kbts`, `coretext`, `directwrite` and `gdi` (which brings Uniscribe) are all `disabled` unless asked for, while `ot` and `fallback` are left out only by the compile-time macros `HB_NO_OT_SHAPE` (implied by `HB_NO_SHAPER` and `HB_NO_OT`) and `HB_NO_FALLBACK_SHAPE` (implied by `HB_NO_LEGACY`), in `hb-config.hh:125–175`. A build with every backend compiled out fails to compile, on purpose: “No shaper enabled.” (`hb-shaper.cc:38`).

`hb_shape_list_shapers` returns the names in order (`hb-shape.cc:101`), and `hb-shape --list-shapers` prints them. The book’s build has the default two:

```shape
backends.ttf  fi  --list-shapers
```

(The font and text are ignored: `--list-shapers` prints the list and stops.)

## How the backend is chosen

The choice is made in `hb_shape_plan_key_t::init` (`hb-shape-plan.cc:75`), which fills in the key of a new plan. After the segment properties, features and variation state are copied into the key, this runs (trimmed slightly):

```c++
#define HB_SHAPER_PLAN(shaper) \
  if (face->data.shaper) \
  { \
    this->shaper_func = _hb_##shaper##_shape; \
    this->shaper_name = #shaper; \
    return true; \
  }

  if (unlikely (shaper_list))
  {
    for (; *shaper_list; shaper_list++)
      if (false) ;
#define HB_SHAPER_IMPLEMENT(shaper) \
      else if (0 == strcmp (*shaper_list, #shaper)) \
        HB_SHAPER_PLAN (shaper);
#include "hb-shaper-list.hh"
  }
  else
  {
    const hb_shaper_entry_t *shapers = _hb_shapers_get ();
    for (unsigned int i = 0; i < HB_SHAPERS_COUNT; i++)
      if (false) ;
#define HB_SHAPER_IMPLEMENT(shaper) \
      else if (shapers[i].func == _hb_##shaper##_shape) \
        HB_SHAPER_PLAN (shaper);
#include "hb-shaper-list.hh"
  }
```

Both loops do the same thing. They go through a list of backends in order, and for each one they test `face->data.shaper`: does this backend have data for this face? The first that does is chosen. The only difference is where the list comes from: the caller’s list of names, or HarfBuzz’s own list.

`face->data.shaper` looks like a field, but it is a lazy loader, of the same kind as the table loaders in [Blobs, faces and lazy tables](blobs-faces-and-lazy-tables.html#lazy-tables). Testing it as a boolean creates the data if it hasn’t been created yet, by calling the backend’s `_hb_<name>_shaper_face_data_create`. So the question “will you shape this face?” is put to each backend in the form “make your data for this face”, and a backend says no by returning `NULL`. The answer is remembered, so each backend is asked once per face, not once per plan.

Only *face* data is asked for here. The backend’s *font* data, which depends on size and variation coordinates, isn’t created until the plan is run.

### Why `ot` always wins

Now the list and the test together explain the behavior. Here is what each backend’s face data needs:

| Order | Backend | Its face data is created if… | Where |
|---|---|---|---|
| 1 | `wasm` | the face has a non-empty `Wasm` table, and the WebAssembly runtime loads it as a module | `hb-wasm-shape.cc:168` |
| 2 | `graphite2` | the face has a non-empty `Silf` table, and libgraphite2 accepts the face | `hb-graphite2.cc:114` |
| 3 | `ot` | always | `hb-ot-shape.cc:413` |
| 4 | `uniscribe` | Windows GDI accepts a renamed copy of the font file | `hb-uniscribe.cc:418` |
| 5 | `directwrite` | DirectWrite makes an `IDWriteFontFace` from the font file | `hb-directwrite-shape.cc:41` |
| 6 | `coretext` | Core Text makes a `CGFont` from the font file | `hb-coretext-shape.cc:39` |
| 7 | `harfrust` | HarfRust can read the face’s tables | `rust/shape.rs:74` |
| 8 | `kbts` | kb_text_shape can read the font file | `hb-kbts.cc:173` |
| 9 | `fallback` | always | `hb-fallback-shape.cc:38` |

The `ot` backend’s face data is no data at all:

```c++
struct hb_ot_face_data_t {};

hb_ot_face_data_t *
_hb_ot_shaper_face_data_create (hb_face_t *face)
{
  return (hb_ot_face_data_t *) HB_SHAPER_DATA_SUCCEEDED;
}
```

`HB_SHAPER_DATA_SUCCEEDED` is the pointer value 2 (`hb-shaper.hh:72`): not a real object, just something that isn’t `NULL`. The `ot` backend keeps everything it needs about a face in the face’s own lazy tables, so it has nothing to make. It never says no, whatever the font: one with no `GSUB`, no `GPOS`, no `cmap`, or a face made from an empty blob. The one face it can’t take is the static empty face, `hb_face_get_empty ()`. That face is inert: its lazy loaders build nothing, for any backend, so no backend accepts it and shaping with it fails.

So, with no shaper list:

* Only the two backends *before* `ot` can ever be chosen instead of it, and both decline every font that lacks their own table. A font is shaped by `graphite2` only if HarfBuzz was built with Graphite *and* the font has `Silf`, and by `wasm` only if HarfBuzz was built with WebAssembly *and* the font has `Wasm`.
* No backend *after* `ot` is ever reached. Uniscribe, DirectWrite, Core Text, HarfBuzz’s Rust twin and kb_text_shape run only when they are asked for by name.
* `fallback` is reached only in a build without `ot`. Its comment says it “should be last”, and since its face data, like `ot`’s, always succeeds, anything after it could never be reached even by name.

That is the whole of “why `ot` always wins”. It isn’t a rule written anywhere: it is the position of one line in a list, and a create function that can’t fail.

It also means the presence of a table decides nothing unless its backend is compiled in. Here is the font HarfBuzz’s own tests use for the Graphite backend, `graphite2-expansion.ttf`. It is three glyphs and a `Silf` table, with no OpenType layout at all:

```info
hb:test/api/fonts/graphite2-expansion.ttf  --list-tables
```

Its Graphite rule replaces every `a` with 41 glyphs (`test/api/fonts/README`), and the test that uses it asks for `graphite2` by name and checks that it gets 164 glyphs from “aaaa” (`test-graphite2-buffer.cc`). The book’s build has no Graphite, so `graphite2` isn’t in its list, `ot` takes the font, and `ot` sees only the `cmap` and the advances:

```shape
hb:test/api/fonts/graphite2-expansion.ttf  aa  --language=en
```

> For type designers: a Graphite font, or a font with a WebAssembly shaper in it, will meet many builds of HarfBuzz that lack the backend for it. None of the optional backends is on in a default build. Such a font should carry OpenType tables as well, so that `ot` has something to work with. SIL’s fonts do: Harmattan, in HarfBuzz’s subsetter tests, has `Silf`, `Glat` and `Gloc` beside a full `GSUB` and `GPOS`, which is why the `ot` backend can join its Arabic here.

```shape
hb:test/subset/data/fonts/Harmattan-Regular.ttf  ببب  --language=ar
```

## Asking for a backend by name

There are two ways to ask.

### The shaper list

`hb_shape_full`’s last argument, and the same argument of `hb_shape_plan_create`, `hb_shape_plan_create2` and the cached versions, is a `NULL`-terminated list of backend names. The first loop above walks it in the caller’s order. Names that aren’t compiled into this build match nothing and are passed over without complaint. The first named backend whose face data can be created is chosen.

If none can, the key isn’t made, `hb_shape_plan_create_cached2` returns the empty plan (`hb-shape-plan.cc:566`), and `hb_shape_full` returns false with the buffer untouched: still Unicode, still unshaped. HarfBuzz doesn’t add `ot` to the end of a list on its own. A caller who wants “Core Text if possible, otherwise HarfBuzz” writes `{"coretext", "ot", NULL}`.

`hb-shape --shapers` passes its argument as this list. It is stricter than the library: it checks every name against `hb_shape_list_shapers` and stops with “Unknown or unsupported shaper” if one isn’t there (`util/shape-options.hh:276`). With the two backends of this build, the order of the list is all that matters, since both always accept the face:

```shape
backends.ttf  fi  --shapers=ot,fallback  --language=en
backends.ttf  fi  --shapers=fallback,ot  --language=en
```

The plan’s key compares backends by `shaper_func` (`hb-shape-plan.cc:184`), not by the list that produced it. So two lists that end up choosing the same backend share cached plans, and the same text shaped with `ot` and with `fallback` gets two plans.

### `HB_SHAPER_LIST`

The second way changes the default list itself. The first time HarfBuzz needs the list, `_hb_shapers_get` reads the environment variable `HB_SHAPER_LIST` (`hb-shaper.cc:48`). If it is set, HarfBuzz copies `_hb_all_shapers` and moves the backends it names, in the order named, to the front. Names that aren’t compiled in are ignored, and the backends not named keep their order after them.

The variable reorders the list; it doesn’t replace it. `HB_SHAPER_LIST=fallback` gives the list `fallback, ot`, in which `fallback` takes every font, while `HB_SHAPER_LIST=coretext` on a Mac build gives `coretext` followed by the rest in their usual order, in which Core Text takes every font it can read, and `ot` still takes the rest. The reordered list is also what `hb_shape_list_shapers` returns, and so what `hb-shape --list-shapers` prints.

The list is made once per process, by a lazy loader, and freed at exit (`hb-shaper.cc:43–90`). Changing the variable afterward has no effect. Builds with `HB_NO_GETENV`, which `HB_LEAN` and `HB_TINY` imply, don’t read it at all (`hb.hh:440`). It exists for testing: running a whole program, or a whole test suite, through another backend without changing a line of it. `hb-shape`’s own option code describes it as the variable that “Overrides the default shaper list” (`util/shape-options.hh:470`).

## Running the plan, and failing

The plan is run by `_hb_shape_plan_execute_internal` (`hb-shape-plan.cc:406`). After its checks, it dispatches on the backend recorded in the key:

```c++
#define HB_SHAPER_EXECUTE(shaper) \
  HB_STMT_START { \
    buffer->enter (); \
    bool ret = font->data.shaper && \
               _hb_##shaper##_shape (shape_plan, font, buffer, features, num_features); \
    buffer->leave (); \
    return ret; \
  } HB_STMT_END

  if (false) ;
#define HB_SHAPER_IMPLEMENT(shaper) \
  else if (shape_plan->key.shaper_func == _hb_##shaper##_shape) \
    HB_SHAPER_EXECUTE (shaper);
#include "hb-shaper-list.hh"

  return false;
```

Three things follow from these few lines.

**The font data is made here.** `font->data.shaper` is the lazy loader for the backend’s font data, created on first use for each font. If it can’t be made (Core Text refusing to make a `CTFont` at this size, say), the call returns false.

**There is no second choice.** If the font data fails, or the backend returns false, `execute` returns false. It doesn’t go back to the list and try the next backend. The list was consulted once, with the face, when the plan was made.

**The buffer’s limits apply to every backend.** `buffer->enter()` sets the per-text limits `max_len` and `max_ops` from the buffer’s length, and `leave()` restores the defaults (`hb-buffer.cc:326`). A comment there explains that `enter()` used to run only from `hb_shape`, and now runs here, on every shape, so that calling `hb_shape_plan_execute` directly is as bounded as calling `hb_shape`.

What a backend’s return value means differs between them. The `ot` backend always returns true (`hb-ot-shape.cc:1214`). When it runs out of memory or of its operation budget, it says so by setting the buffer’s `successful` flag to false, which callers read with `hb_buffer_allocation_successful`, and the buffer may be left half shaped. The HarfRust backend does the same, so that callers of `hb_shape`, which can’t see a return value, can find out (a comment at `hb-harfrust.cc:176` explains this; `NEWS` lists it for 15.1.0). The other backends return false when they fail. A program that wants to know whether shaping worked has to check both.

## Per-shaper data

Each backend may keep one object per face and one per font. `hb_face_t` and `hb_font_t` each have a member `data` (`hb-face.hh:63`, `hb-font.hh:155`) of type `hb_shaper_object_dataset_t`, which is, again, made from the list (`hb-shaper.hh:111`, trimmed):

```c++
template <typename Object>
struct hb_shaper_object_dataset_t
{
  Object *parent_data; /* MUST be JUST before the lazy loaders. */
#define HB_SHAPER_IMPLEMENT(shaper) \
        hb_shaper_lazy_loader_t<Object, HB_SHAPER_ORDER(shaper), \
                                hb_##shaper##_##object##_data_t> shaper;
#include "hb-shaper-list.hh"
};
```

So a face in a default build has `face->data.ot` and `face->data.fallback`, and a build with Graphite also has `face->data.graphite2`. Each member is a single pointer, null until first use.

Two details make this cheap.

A loader has to know which face or font it belongs to, in order to pass it to the create function. Storing a pointer back in each loader would cost one pointer per backend per object. Instead, the one pointer `parent_data` comes first, and each loader finds it by counting back from its own address. Its position in the list is its `HB_SHAPER_ORDER`, numbered from 1, so the *n*th loader is *n* pointers after `parent_data`, and `hb_data_wrapper_t::get_data` reads `*(((Data **) this) - WheresData)` (`hb-machinery.hh:160`). That is why the comment insists the pointer be “JUST before the lazy loaders”.

A failed create has to be remembered, or the `graphite2` loader would look for `Silf` again on every plan for every non-Graphite font. But a loader’s empty value is `NULL`, so `NULL` can’t also mean “tried and failed”. The loaders therefore set `tag_null` (`hb-shaper.hh:104`): a failure is stored as the null value with its lowest bit set, which isn’t null, and is read back as null. [Blobs, faces and lazy tables](blobs-faces-and-lazy-tables.html#failure-is-cached-as-null) describes the same trick for table accelerators. It is also why `HB_SHAPER_DATA_SUCCEEDED` is 2 and not 1: “Low bit is taken by the lazy-loader’s tag_null” (`hb-shaper.hh:71`).

Creation uses the same lock-free compare-and-swap as the table loaders ([Loading without a lock](blobs-faces-and-lazy-tables.html#loading-without-a-lock)): two threads may both create the data, and the one that loses the race destroys its copy. A face’s shaper data lives as long as the face. A font’s is thrown away whenever the font changes: `hb_font_t::changed()` ends with `data.fini ()` (`hb-font.hh:1159`), so a new size or new variation coordinates mean every backend makes its font data again on the next shape. [Fonts and font functions](fonts-and-font-functions.html#serial-numbers) lists the setters that call it.

The data has one more user besides the backends. HarfBuzz’s Core Text and DirectWrite *font functions* get their native font objects from the matching backend’s font data, `font->data.coretext` and `font->data.directwrite`, so the two layers share one object, as [Fonts and font functions](fonts-and-font-functions.html#the-other-funcs) explains.

## The backends

### `ot`

HarfBuzz’s own engine. It shapes OpenType fonts with `GSUB` and `GPOS`, and AAT fonts with `morx`, `kerx` and `trak` in their place: there is no separate AAT backend, and the choice between the two kinds of table is made inside `ot`, as [AAT inside HarfBuzz](aat-inside-harfbuzz.html) explains. Its font data is a cache for the variation store in `GDEF`, which holds the deltas that a variable font’s `GPOS` values and `GDEF` carets refer to (`hb-ot-shape.cc:433`). A font whose `GDEF` has no variation store gets the shared Null cache, which is still not `NULL`, so this can’t fail either. Its entry point, `_hb_ot_shape` (`hb-ot-shape.cc:1205`), wraps everything in a context and calls `hb_ot_shape_internal`, which [The pipeline, step by step](the-pipeline-step-by-step.html) follows from start to end.

It also builds every plan’s `ot` half, whichever backend is chosen. `hb_shape_plan_create2` calls `shape_plan->ot.init0` unconditionally once the key is made (`hb-shape-plan.cc:280`), so even a plan that `graphite2` will run has its OpenType features compiled into lookups.

### `fallback`

The simplest backend there could be. Here it is, nearly whole (`hb-fallback-shape.cc:72`):

```c++
  hb_codepoint_t space;
  bool has_space = (bool) font->get_nominal_glyph (' ', &space);

  buffer->clear_positions ();

  for (unsigned int i = 0; i < count; i++)
  {
    if (has_space && unicode->is_default_ignorable (info[i].codepoint)) {
      info[i].codepoint = space;
      pos[i].x_advance = 0;
      pos[i].y_advance = 0;
      continue;
    }
    (void) font->get_nominal_glyph (info[i].codepoint, &info[i].codepoint);
    font->get_glyph_advance_for_direction (info[i].codepoint, direction,
                                           &pos[i].x_advance, &pos[i].y_advance);
    font->subtract_glyph_origin_for_direction (info[i].codepoint, direction,
                                               &pos[i].x_offset, &pos[i].y_offset);
  }

  if (HB_DIRECTION_IS_BACKWARD (direction))
    hb_buffer_reverse (buffer);

  buffer->clear_glyph_flags ();
```

Each character becomes the glyph `cmap` gives it, or glyph 0. Each glyph gets its advance from the font funcs. Default ignorables become a zero-width space, if the font has a space. Right-to-left text is reversed into visual order. That is all. There is no normalization, no ligature, no kerning, no mark positioning, no joining and no reordering, and each character stays in a cluster of its own. Side by side with `ot` on the test font:

```shape
backends.ttf  fi  --shapers=ot  --language=en
backends.ttf  fi  --shapers=fallback  --language=en
backends.ttf  AV  --shapers=ot  --language=en
backends.ttf  AV  --shapers=fallback  --language=en
backends.ttf  \u00E9  --shapers=ot  --language=en
backends.ttf  \u00E9  --shapers=fallback  --language=en
backends.ttf  ببب  --shapers=ot  --language=ar
backends.ttf  ببب  --shapers=fallback  --language=ar
```

The `é` shows the cost of skipping normalization. The font has no precomposed `é`, so `ot` decomposes it into `e` and the combining acute, both of which it has, and then positions the acute on its anchor. `fallback` asks `cmap` for U+00E9 and gets `.notdef`. The behs come out in the right order, because reversing is the one thing `fallback` does for right-to-left text, but isolated.

Clearing the glyph flags is correct here. Each glyph depends only on its own character, so the text can be broken and reshaped anywhere and come out the same.

This backend is not what HarfBuzz means elsewhere by “fallback”. The `ot` backend has fallbacks of its own, for fonts without some table: mark positioning without `GPOS`, Arabic shaping from presentation forms without `GSUB`, spaces from Unicode properties. Those are in [Fallbacks](fallbacks.html), and they run inside the full pipeline. The `fallback` backend is a different and much simpler thing (its file’s copyright is from 2011), whose job is only to make sure a build without `ot` still turns characters into glyphs.

### `graphite2`

SIL’s Graphite puts a font’s shaping rules in its own tables (`Silf` for the rules, `Glat` and `Gloc` for glyph attributes, `Feat` for features) and runs them with its own library, libgraphite2. HarfBuzz has had a backend for it since 0.9.2, whose `NEWS` calls it “New experimental Graphite2 backend”. It is built with `-Dgraphite2=enabled`.

Its face data checks for `Silf`, then makes a `gr_face` whose table callback fetches tables through the HarfBuzz face and keeps a list of the blobs it has handed out, added to without a lock (`hb-graphite2.cc:74`, `:114`). Its font data is nothing.

The shape function (`:233`) turns the buffer’s language into Graphite’s feature defaults, sets any of the user’s features the font defines, and makes a Graphite segment from the buffer’s code points. It then has to translate Graphite’s results into HarfBuzz’s. Graphite reports slots and their origins, and which characters each slot came from. HarfBuzz rebuilds clusters from those (merging any that Graphite reorders across one another), gives each cluster one advance, turns origins into offsets, and reverses for right-to-left text. At the end it marks the whole buffer unsafe to break (`:482`), since nothing is known about where it could be broken.

What it gives up is the HarfBuzz font. The segment is made with no Graphite font (`gr_make_seg (nullptr, …)`, `:282`), so Graphite works in design units from its own reading of `cmap` and `hmtx`, and HarfBuzz scales the result by `x_scale / upem` (`:340`). The font funcs aren’t consulted: not for glyphs, not for advances, and not for variations, which Graphite doesn’t have. The `TODO vertical` comment at `:255` marks the other gap.

### `wasm`

The WebAssembly backend lets a font carry its own shaping engine, compiled to WebAssembly and stored in a table tagged `Wasm`. It arrived in 8.0.0 and is still called experimental, in `meson_options.txt` and in `NEWS`. It needs the wasm-micro-runtime library and `-Dwasm=enabled`, and `docs/wasm-shaper.md` documents it in full.

Its face data loads the table as a WebAssembly module (`hb-wasm-shape.cc:168`). To shape, it needs an *instance* of the module, with its own memory: 32 KiB of stack and 2 MiB of heap (`:217`). Instances are made as needed, and one is kept in the face data for reuse. When an instance is made, HarfBuzz calls the module’s `shape_plan_create(face)` if it exports one, so the module can prepare per-face state. Then for each run it calls the module’s `shape(plan, font, buffer, features, num_features)`, which returns zero for failure. If it fails while the buffer still holds Unicode, HarfBuzz throws the instance away and tries once more with a fresh one (`:436`).

The module works through an API that HarfBuzz exports to it (`hb-wasm-api-*.hh`): copying the buffer’s contents in and out, asking the font for glyphs, advances, extents, outlines and coordinates, reading tables. And one more function, `shape_with`, which runs another backend on the same font and buffer (`hb-wasm-api-shape.hh:36`). It calls `hb_shape_full` with a one-name shaper list, and refuses the name `wasm`, so a module can’t start itself again. The documentation’s examples use it to let `ot` do the ordinary work and then change the result. HarfBuzz’s own source has a larger example: `src/wasm/graphite/` builds Graphite itself into a module and adds it to fonts such as Charis SIL, so that they shape with Graphite in a HarfBuzz that has WebAssembly but no `graphite2`.

The backend trusts the module with the buffer. A comment after the call says “TODO Regularize clusters according to direction & cluster level, such that client doesn’t crash with unmet expectations” (`:439`), and the documentation asks modules to keep clusters monotonic. Glyph flags are cleared and the whole buffer is marked unsafe to break (`:452`). Another comment admits that initializing the runtime isn’t thread-safe (`:122`). `NEWS` for 15.0.0 records hardening “against malicious fonts”, which is what a font that runs code invites.

> For type designers: with `wasm` first in the list, a `Wasm` table overrides everything else in the font, in a HarfBuzz built with it. In any other build, and in every other engine, the table is ignored, so the font’s OpenType tables are what most readers will see.

### `harfrust`

HarfRust is a port of HarfBuzz’s shaping to Rust, from the HarfBuzz project itself. `NEWS` for 11.2.1 introduced the backend as “harfruzz”, “an ongoing Rust port of HarfBuzz shaping… mainly used for testing the output of the Rust implementation”, and 11.3.0 renamed it after the upstream rename. It is built with `-Dharfrust=enabled`, and this release uses HarfRust 0.15 (`src/rust/Cargo.toml`). [HarfRust](harfrust.html) is about HarfRust itself.

The glue is in `hb-harfrust.cc` and `rust/shape.rs`. Face data is a HarfRust `Font` that fetches tables from the HarfBuzz face (`shape.rs:74`). Font data is an instance of it at the font’s normalized coordinates, with its scale and point size (`:220`). HarfRust’s own plan is cached in the HarfBuzz plan, in a field that exists only for it (`hb_shape_plan_t::harfrust_data`, `hb-shape-plan.hh`). For each run, the buffer’s code points, clusters, flags, cluster level and pre- and post-context are copied into a HarfRust buffer, shaped, and the glyphs and positions copied back (`shape.rs:336`).

Unlike the other backends, it uses the HarfBuzz font. `HBHarfBuzzFontFuncs` (`shape.rs:116`) answers HarfRust’s questions about nominal and variation glyphs, advances, vertical origins and extents by calling `hb_font_get_*`, so custom font funcs and synthetic bold reach it just as they reach `ot`. That is what makes it useful for testing: with the same font answers going in, any difference in the output is a difference between the two shaping implementations.

### `kbts`

kb_text_shape is a single-header C shaping library by another author, compiled straight into HarfBuzz (`hb-kbts.cc`). `NEWS` for 11.4.0 is blunt about it: “purely for testing and performance evaluation and we do NOT recommend using it for any other purposes.” 15.0.0 updated it to kb_text_shape 2.28g and added variations.

Its face data is the whole font file, read by kb_text_shape (`hb-kbts.cc:173`), and its font data the variation coordinates (`:220`). It takes advances from the font itself, not from the font funcs, and scales them (`:345`). It refuses vertical text (`:278`), and marks the whole buffer unsafe to break (`:362`).

### The platform backends: `coretext`, `directwrite`, `uniscribe`

These three hand the text to the operating system’s own shaping engine. (The faces and font funcs that wrap the same platform libraries are another matter: [Integrations](integrations.html#the-platforms-core-text-directwrite-gdi-and-uniscribe).) The Uniscribe and Core Text backends came with 0.9.2, whose `NEWS` calls both “primarily for testing”, and DirectWrite later. Their purpose is comparison: HarfBuzz has to match what these engines do with the same fonts, and the quickest way to find out is to ask them.

* **`coretext`** (Apple platforms, `-Dcoretext=enabled`) makes a `CGFont` from the face’s file, or reuses the one a Core Text face was made from (`hb-coretext.cc:261`). It can’t open any face of a collection but the first: “TTC index %u not supported” (`:222`). Its font data is a `CTFont` at the font’s `ptem` size, or 12 points if none is set (`HB_CORETEXT_DEFAULT_FONT_SIZE`, `hb-aat-layout.hh:36`), with the font’s variation coordinates. OpenType feature tags are translated into AAT feature types and selectors where a mapping exists (`hb-coretext-shape.cc:220`).
* **`directwrite`** (Windows, `-Ddirectwrite=enabled`) makes an `IDWriteFontFace` from the face’s file. It supports variations and user features since 11.0.0, when `NEWS` says it “graduated from experimental”.
* **`uniscribe`** (Windows, `-Dgdi=enabled`) has the oldest problem to solve: GDI selects fonts by name, so HarfBuzz copies the font, gives it a new `name` table naming it with a random name made from a UUID (`F_` and 24 characters, `hb-uniscribe.cc:300`), and installs it privately with `AddFontMemResourceEx` (`:443`). Its font data is a GDI font at the font’s `y_ppem`, or at the units per em. It has no variations.

All three take glyphs and positions from the system, not from the font funcs, and HarfBuzz only scales the system’s numbers into its own. All three mark the whole buffer unsafe to break, and none handles vertical text.

## The backends side by side

| Backend | Build option | Chosen by default for | Uses the font funcs | Variations | Glyph flags |
|---|---|---|---|---|---|
| `wasm` | `-Dwasm` (experimental) | fonts with `Wasm` | if the module asks | through the font | all unsafe to break |
| `graphite2` | `-Dgraphite2` | fonts with `Silf` | no (scale only) | no | all unsafe to break |
| `ot` | on | every other font | yes | yes | computed |
| `uniscribe` | `-Dgdi` | none | no | no | all unsafe to break |
| `directwrite` | `-Ddirectwrite` | none | no | yes | all unsafe to break |
| `coretext` | `-Dcoretext` | none | no | yes | all unsafe to break |
| `harfrust` | `-Dharfrust` | none | yes | yes | computed by HarfRust |
| `kbts` | `-Dkbts` | none | no | yes | all unsafe to break |
| `fallback` | on | none, unless `ot` is compiled out | yes | through the font | none (all safe) |

The “glyph flags” column matters to applications that break lines by reshaping only the part of a paragraph that changed. With the backends that mark everything unsafe to break, every change means reshaping the whole run. [For applications](for-applications.html) covers what the flags are for.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-shape.cc` | `hb_shape`, `hb_shape_full`, `hb_shape_list_shapers` |
| `src/hb-shaper-list.hh` | the list of backends, in order of preference |
| `src/hb-shaper.hh`, `src/hb-shaper.cc` | the shape function type, the per-shaper data loaders and dataset, `_hb_all_shapers`, `HB_SHAPER_LIST` |
| `src/hb-shape-plan.cc` | choosing a backend (`hb_shape_plan_key_t::init`) and running it (`_hb_shape_plan_execute_internal`) |
| `src/hb-machinery.hh` | the lazy loader, `get_data`, and `tag_null` |
| `src/hb-ot-shape.cc` | the `ot` backend’s data and entry point |
| `src/hb-fallback-shape.cc` | the `fallback` backend |
| `src/hb-graphite2.cc` | the `graphite2` backend |
| `src/hb-wasm-shape.cc`, `src/hb-wasm-api*.hh`, `docs/wasm-shaper.md`, `src/wasm/` | the `wasm` backend, the API it gives modules, its documentation and examples |
| `src/hb-harfrust.cc`, `src/rust/shape.rs` | the `harfrust` backend and its Rust glue |
| `src/hb-kbts.cc` | the `kbts` backend |
| `src/hb-coretext-shape.cc`, `src/hb-directwrite-shape.cc`, `src/hb-uniscribe.cc` | the platform backends |
| `meson_options.txt` | which backends a build includes |

To see which backends a build has, run `hb-shape --list-shapers`. To shape with one, use `hb-shape --shapers=NAME`, or set `HB_SHAPER_LIST` to run a whole program through it.

## In short

* `hb_shape` gets a plan from the face’s cache and runs it. The plan names one backend, chosen when the plan is made.
* With no shaper list, the first backend in `hb-shaper-list.hh` that can make data for the face wins. Only `wasm` and `graphite2` come before `ot`, and they accept only fonts with a `Wasm` or `Silf` table, in builds that include them. `ot` accepts every face, so nothing after it is chosen by default.
* Uniscribe, DirectWrite, Core Text, HarfRust and kb_text_shape run only when named, in a shaper list or in `HB_SHAPER_LIST`, which reorders the default list for the whole process.
* There is no falling through: if the chosen backend fails, shaping fails. `ot` reports failure through the buffer, not its return value, so check `hb_buffer_allocation_successful`.
* Each backend keeps lazily made data on faces and fonts. Font data is rebuilt whenever the font changes, and failures are remembered.
* The `fallback` backend maps characters to glyphs and does nothing else. It is not the `ot` backend’s fallback shaping.
