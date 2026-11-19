# Blobs, faces and lazy tables

The [last two chapters](fonts-as-bytes.html) showed how HarfBuzz reads a table: it lays a struct over the bytes, and [the sanitizer](trust-nothing-the-sanitizer.html) checks those bytes once so that the struct can be trusted afterward. This chapter is about where the bytes come from, and when each table is checked. The answer to the second question is: as late as possible, and only once.

Two objects hold a font before it has a size. A **blob** is a run of bytes with a reference count and a way to free them. A **face** is a blob seen as one font in an sfnt file: it knows how to find a table by its tag, and it keeps, for each table HarfBuzz uses, a slot that stays empty until something asks for that table. The first request fills the slot, sanitizing the table and often building a faster structure to read it with, and every later request just reads the pointer. There are no locks. If two threads ask at once, both do the work and one of them throws its copy away. If the table is missing or broken, the slot is filled with an empty table, so the work isn’t done again.

The result is that opening a font costs almost nothing, a font that is never used for vertical text never has its `vmtx` read, and one face can be shared by every thread in a program. The `hb_font_t` built on a face, with its size and variation coordinates, is the subject of the [next chapter](fonts-and-font-functions.html).

## Blobs

A blob is small. Here are its fields, from `hb-blob.hh:39`, without the helpers that some encoders use to reuse their output buffers:

```c++
struct hb_blob_t
{
  hb_object_header_t header;

  const char *data = nullptr;
  unsigned int length = 0;
  hb_memory_mode_t mode = (hb_memory_mode_t) 0;

  void *user_data = nullptr;
  hb_destroy_func_t destroy = nullptr;
  /* … */
};
```

`data` and `length` say where the bytes are. `destroy` is called with `user_data` when the last reference to the blob goes, and it’s how the bytes get freed, unmapped, or handed back to whoever owns them. HarfBuzz never frees `data` itself.

`hb_blob_create` takes all five, and it takes ownership of `user_data` straight away: if it can’t make the blob, because the length is zero, or 2<sup>31</sup> bytes or more, or memory runs out, it calls `destroy` before returning (`hb-blob.cc:69`, `:107`). A caller never has to work out whether to clean up after a failure. What it gets back on failure is the **empty blob**, `hb_blob_get_empty()`, which is the blob’s Null object (`hb-blob.cc:226`): no data, length zero. The `_or_fail` variant returns `NULL` instead.

### Memory modes

The mode tells HarfBuzz what it may do with the bytes:

| Mode | Meaning |
|---|---|
| `HB_MEMORY_MODE_DUPLICATE` | Copy the data now. The blob owns the copy, and the caller can reuse its buffer at once. |
| `HB_MEMORY_MODE_READONLY` | Use the caller’s bytes in place, and never write to them. |
| `HB_MEMORY_MODE_WRITABLE` | Use the caller’s bytes in place. HarfBuzz may write to them. |
| `HB_MEMORY_MODE_READONLY_MAY_MAKE_WRITABLE` | Read-only, but HarfBuzz may try to make the pages writable in place, with `mprotect`. |

The comment above the enum in `hb-blob.h` gives the advice: use `DUPLICATE` if the caller might change or free the data, `READONLY` otherwise, `WRITABLE` only for a copy made just for HarfBuzz, and not the last mode at all (“using that mode correctly is very tricky. Use `HB_MEMORY_MODE_READONLY` instead”). `DUPLICATE` isn’t really a mode of its own: `hb_blob_create_or_fail` starts the blob as `READONLY` and immediately makes it writable, which copies it.

### From a file

`hb_blob_create_from_file_or_fail` (`hb-blob.cc:789`) first tries to map the file into memory: with `mmap` (`PROT_READ`, `MAP_PRIVATE`) on POSIX systems, or with `CreateFileMapping` and `MapViewOfFile` on Windows (`_hb_blob_try_mmap`, `:620` and `:659`). The blob’s destroy function unmaps it. Nothing is read from disk at this point: the operating system brings pages in as they are touched, and pages that are never touched are never read. On a Mac, if the file’s data fork is empty, HarfBuzz maps its resource fork instead, which is where old dfont-style fonts keep their data.

If mapping fails, or the system has no `mmap`, HarfBuzz reads the file into memory in growing chunks (`_hb_blob_read_file`, `:732`). That route works on pipes too, and it stops at about 512 MiB. The mapped blob is created with the “very tricky” mode, `READONLY_MAY_MAKE_WRITABLE`. HarfBuzz owns the mapping, and it is private, so even a write to it would never reach the file.

### Sub-blobs

Every table HarfBuzz reads from a file is a **sub-blob**: a blob that points into another blob’s bytes. Here is all of `hb_blob_create_sub_blob` (`hb-blob.cc:169`), without the declarations:

```c++
if (!length || !parent || offset >= parent->length)
  return hb_blob_get_empty ();

hb_blob_make_immutable (parent);

blob = hb_blob_create (parent->data + offset,
                       hb_min (length, parent->length - offset),
                       HB_MEMORY_MODE_READONLY,
                       hb_blob_reference (parent),
                       _hb_blob_destroy);
```

Three things happen. The parent becomes immutable, so its bytes can no longer be swapped out from under its children. The child’s length is cut to what the parent actually has, so a table record that claims more bytes than the file holds gets only what is there. And the child holds a reference to the parent as its `user_data`, with `hb_blob_destroy` as its destroy function, so a table blob keeps the whole file alive for as long as anyone holds it. No bytes are copied.

### Writable copies, and why little needs them now

A blob can be made writable: `hb_blob_get_data_writable` asks for it, and `try_make_writable` (`hb-blob.cc:473`) first tries `mprotect` on the pages, if the mode allows it, and otherwise copies the data into memory the blob owns. `hb_blob_copy_writable_or_fail` always copies.

This used to matter. The sanitizer once repaired broken tables by editing them, and it needed writable bytes to do it. Since HarfBuzz 12.3 it repairs nothing (the [sanitizer chapter](trust-nothing-the-sanitizer.html) tells that story): a table either passes, and is made immutable, or is dropped. So nothing on the path from a font file to a shaped run writes to a blob anymore, and the `mprotect` route is, in practice, unused: the file blob is made immutable as soon as a face is made from it, and `hb_blob_get_data_writable` refuses an immutable blob. The few places in the library that need writable bytes make a copy. `cvar`, for example, copies the `cvt` table with `hb_blob_copy_writable_or_fail` before applying its deltas to it (`hb-ot-var-cvar-table.hh:178`).

## Objects, references and user data

Blobs, faces, fonts, buffers and the other public objects all start with the same header (`hb-object.hh:213`):

```c++
struct hb_object_header_t
{
  hb_reference_count_t ref_count;
  mutable hb_atomic_t<bool> writable = false;
  hb_atomic_t<hb_user_data_array_t *> user_data;
};
```

The reference count is an atomic integer. `hb_object_create` starts it at one, `reference` adds one, and `destroy` takes one away and frees the object when it reaches zero. A count of *zero* has a special meaning: it marks a static object, such as an empty singleton, which `reference` and `destroy` leave alone. The empty blob and the empty face are made with `HB_OBJECT_HEADER_STATIC`, which is all zeros, so they are never counted, never freed, and, since `writable` is false, always immutable. `hb_face_create_or_fail` uses that last fact to recognize the empty face: an immutable face straight from `hb_face_create` can only be the singleton.

`writable` is the immutability flag: `make_immutable` clears it, and setters check it. `user_data` lets an application hang its own pointers on any object, each under a key, with a destroy function that runs when the object dies. The array is allocated on first use, published with the same compare-and-swap that the lazy loaders below use, and the array itself is guarded by a mutex. It is the one place in this chapter with a lock in it.

## Faces

A face is the struct in `hb-face.hh:47`. Trimmed of the per-shaper data and the table-tag callback:

```c++
struct hb_face_t
{
  hb_object_header_t header;

  unsigned int index;                       /* Face index in a collection */
  mutable hb_atomic_t<unsigned> upem;
  mutable hb_atomic_t<unsigned> num_glyphs;

  hb_reference_table_func_t  reference_table_func;
  void                      *user_data;
  hb_destroy_func_t          destroy;

  hb_ot_face_t table;                       /* All the face's tables. */
  /* … and the shape-plan cache */
};
```

There is no blob in it. A face doesn’t know where its bytes are: it has a function, `reference_table_func`, which it calls with a tag and gets a blob back, and that is the only way any table gets into a face. `face->reference_table (tag)` is a thin wrapper that turns a missing function or a `NULL` result into the empty blob (`hb-face.hh:80`).

### hb_face_create

The usual face is made from a blob holding a font file. `hb_face_create` (`hb-face.cc:244`) does very little:

```c++
blob = hb_sanitize_context_t ().sanitize_blob<OT::OpenTypeFontFile> (hb_blob_reference (blob));

hb_face_for_data_closure_t *closure = _hb_face_for_data_closure_create (blob, index);

face = hb_face_create_for_tables (_hb_face_for_data_reference_table,
                                  closure,
                                  _hb_face_for_data_closure_destroy);
face->index = index;
```

It sanitizes the file as an `OpenTypeFontFile`, which means only the outer container: the sfnt header and its table records, or a TTC header and each face’s directory, or a dfont resource map. No table is looked at. Then it stores the blob and the face index in a small closure and makes a face whose `reference_table_func` reads from that closure. The face holds its own reference to the blob, so the caller can destroy its reference as soon as the face exists.

If the file fails that check, the sanitizer hands back the empty blob, and the face is made anyway. `hb_face_create` never fails, except for lack of memory: given rubbish, it returns a working face with no tables in it, whose upem is 1000 and whose glyph count is zero. `hb_face_create_or_fail` (`:289`) is the version that says no. It counts the faces in the blob first, and returns `NULL` if the index is out of range.

### Finding a table

The function that every table of a file-based face passes through is `_hb_face_for_data_reference_table` (`hb-face.cc:185`):

```c++
if (tag == HB_TAG_NONE)
  return hb_blob_reference (data->blob);

const OT::OpenTypeFontFile &ot_file = *data->blob->as<OT::OpenTypeFontFile> ();
unsigned int base_offset;
const OT::OpenTypeFontFace &ot_face = ot_file.get_face (data->index, &base_offset);

const OT::OpenTypeTable &table = ot_face.get_table_by_tag (tag);

hb_blob_t *blob = hb_blob_create_sub_blob (data->blob, base_offset + table.offset, table.length);
```

The tag `HB_TAG_NONE` asks for the whole file. Any other tag is looked up in the face’s table directory, as [Fonts as bytes](fonts-as-bytes.html#finding-a-table) describes. A tag that isn’t there finds the Null table record, whose offset and length are zero, and a sub-blob of length zero is the empty blob. A table whose record points past the end of the file gets the empty blob too, or a shortened one, from the clamping in `hb_blob_create_sub_blob`. None of this touches the table’s contents, so a call to `reference_table` costs a directory search and one small allocation.

> For programmers: `hb_face_reference_table` is this function, with nothing added. It returns the table’s bytes *unsanitized*. Only the face’s lazy loaders, below, check what they load. A program that reads a table itself through the public API has to check it itself.

### Collections and the face index

A TrueType Collection (TTC) holds several faces in one file, each with its own table directory. Identical tables can be stored once and pointed at from several directories. The test file `pair.ttc` is a collection of two of the fonts from the [GSUB chapter](gsub.html), `ligatures.ttf` and `ccmp-first.ttf`. Its two directories point at the same bytes for every table except `GSUB` and `head`, so the two faces share their glyphs, `cmap` and metrics, and differ only in the order of their lookups. HarfBuzz counts the faces with `hb_face_count` (`hb-face.cc:80`), which sanitizes the container and asks it:

```info
pair.ttc  --show-face-count
```

The face index picks one. Face 1’s `ccmp` lookup comes before its `liga` lookup, so the `f i` that `ccmp` makes from `x` can still ligate:

```shape
pair.ttc  x  --face-index=0
pair.ttc  x  --face-index=1
```

In a collection, table offsets count from the start of the file, so `base_offset` is zero. In a dfont, they count from the start of each face’s resource, and `get_face` returns where that is. A single sfnt ignores the index altogether.

The index is a 32-bit number, but only its low 16 bits choose a face: the closure stores `index & 0xFFFF` (`hb-face.cc:170`). The high 16 bits, if they aren’t zero, choose a **named instance** of a variable font, counting from one. The face only keeps them. It is `hb_font_create` that reads them, and sets the font to named instance `(index >> 16) - 1` (`hb-font.cc:2155`). The test font `instances.ttf` is the variable font from the [design space chapter](design-space.html) with three named instances added: Thin, Bold and Black. `hb-shape` passes `--face-index` straight to the face, so 65536 (1 << 16) asks for instance 0, 131072 for instance 1, and so on:

```shape
instances.ttf  IHV
instances.ttf  IHV  --face-index=65536
instances.ttf  IHV  --face-index=131072
instances.ttf  IHV  --face-index=196608
instances.ttf  IHV  --named-instance=1
instances.ttf  IHV  --face-index=262144
```

Index 131072 gives the same result as `--named-instance=1`, which sets the instance on the font directly. Index 262144 asks for a fourth instance that the font doesn’t have, and gets the default one. This packing gives an index alone enough room to name both a face in a collection and an instance in it, which is how some font systems identify a font. Until HarfBuzz 15.1, `hb_face_create_or_fail` compared the whole index with the face count, so it refused any index with the high bits set. The `hb-shape` runs above, which go through it, would have failed.

### Faces from somewhere else

Because tables only come in through a callback, a face doesn’t need a font file at all. `hb_face_create_for_tables` (`hb-face.cc:130`) takes the callback, its `user_data`, and a destroy function, and makes a face around them. It’s how an application whose fonts live elsewhere (a font editor’s tables in memory, or a system font API that hands out tables one at a time) gives HarfBuzz just the tables it asks for. Such a face can’t list its tables unless `hb_face_set_get_table_tags_func` is given one more callback that can.

The **face builder** is a face made this way. `hb_face_builder_create` (`hb-face-builder.cc:215`) returns a face whose callback reads from a hash map of tag to blob, and `hb_face_builder_add_table` adds to the map. Asking a builder face for `HB_TAG_NONE`, through `hb_face_reference_blob`, makes it write all its tables out as a new font file. The subsetter uses it to assemble its output, and `hb_face_reference_blob` uses it for any face that can list its tables but can’t give its file as a whole (`hb-face.cc:742`).

> For programmers: a `reference_table_func` may be called more than once for the same tag, and from several threads at once. Two threads that race to load a table both call it (see below), and some accelerators load tables directly rather than through the face’s slot for them. It should return a new reference each time, and be safe to call from any thread.

### Face loaders

Since HarfBuzz 11.0 there is a second way to choose how a face is read: by a **face loader** name. `hb_face_create_from_file_or_fail_using` and `hb_face_create_or_fail_using` take a file or a blob, a face index, and a loader name, and look the name up in a short table (`supported_face_loaders`, `hb-face.cc:335`):

| Name | Built when | Reads the font with |
|---|---|---|
| `ot` | always | HarfBuzz itself: `hb_face_create_or_fail` |
| `ft` | with FreeType | FreeType, which, depending on how it was built, can read WOFF and WOFF2 too |
| `coretext` | with Core Text | Core Text |
| `directwrite` | with DirectWrite | DirectWrite |

With no name, or an empty one, the `HB_FACE_LOADER` environment variable is tried, and then the first loader in the table, which is always `ot`. A name from the environment that isn’t in the table falls back to the default, but one passed in by the caller makes the call fail. `hb_face_list_loaders` lists what was built in, and the command-line tools take `--face-loader`. The book’s build of HarfBuzz has only `ot`. Whatever the loader, the result is an ordinary `hb_face_t`, and the rest of HarfBuzz can’t tell the difference. Some loaders end up wrapping the font’s bytes in a face much like `hb_face_create`’s; others make a face whose `reference_table_func` asks their library for each table.

### Units per em and the glyph count

Two numbers are needed so often that the face keeps them in fields of its own: `upem` and `num_glyphs`. Both are atomics and both are filled in lazily, with a sentinel value meaning “not loaded yet”: zero for `upem`, and `UINT_MAX` for `num_glyphs`, which `hb_face_create_for_tables` sets. The first call to `get_upem` reads `head`, and the first call to `get_num_glyphs` reads `maxp` (`hb-static.cc:79` and `:86`). If two threads do this at once, they both store the same number, which is harmless.

`head`’s `get_upem` returns 1000 for any value outside 16 to 16,384, including the zero that a missing `head` gives. The empty face has 1000 and zero glyphs built in, so it never loads anything. `hb_face_set_upem` and `hb_face_set_glyph_count` can set either number by hand, for faces made from tables that don’t have them.

### Immutability

`hb_font_create` makes its face immutable (`hb-font.cc:2113`), and so does `hb_shape_plan_create2` (`hb-shape-plan.cc:267`). From then on the face’s setters do nothing: `hb_face_set_index`, `hb_face_set_upem`, `hb_face_set_glyph_count` and `hb_face_set_get_table_tags_func` all check the flag first. A face is meant to be set up and then only read.

Immutable doesn’t mean nothing inside it changes. Its tables are still loaded on first use, its shape-plan cache still grows, and both are written to by whichever thread gets there first. What immutability promises is that nothing a caller can *see* changes, which is what makes lazy loading safe to do behind the caller’s back.

## Lazy tables

### One pointer per table

The `table` field of a face is an `hb_ot_face_t` (`hb-ot-face.hh:48`), and nearly all of it is generated from a list. `hb-ot-face-table-list.hh` names every table that a face can load, each with one of two macros:

```c++
HB_OT_TABLE (OT, head)
HB_OT_ACCELERATOR (OT, maxp)
HB_OT_ACCELERATOR (OT, cmap)
/* … */
HB_OT_ACCELERATOR (OT, GSUB)
HB_OT_ACCELERATOR (OT, GPOS)
```

`hb-ot-face.hh` includes the list twice. The first time it makes an enum, with one entry per table, starting from one. The second time it makes a member per table:

```c++
hb_face_t *face; /* MUST be JUST before the lazy loaders. */
#define HB_OT_TABLE(Namespace, Type) \
  hb_table_lazy_loader_t<Namespace::Type, HB_OT_TABLE_ORDER (Namespace, Type)> Type;
#define HB_OT_ACCELERATOR(Namespace, Type) \
  hb_face_lazy_loader_t<Namespace::Type##_accelerator_t, HB_OT_TABLE_ORDER (Namespace, Type)> Type;
#include "hb-ot-face-table-list.hh"
```

So `face->table.head` is a loader for the `head` table itself, and `face->table.GSUB` is a loader for a `GSUB_accelerator_t`, a structure built from the table. Each loader is exactly one pointer, and it starts as `NULL`.

A loader needs the face to load anything, but it doesn’t store a pointer to it. It works out where the face pointer is from its own address. Its position in the enum, `WheresData`, is the number of pointers between it and the `face` field just before the loaders, so (`hb-machinery.hh:160`):

```c++
Data * get_data () const
{ return *(((Data **) (void *) this) - WheresData); }
```

`head` is first in the enum, so its loader is one pointer after `face`. That is why the comment insists that `face` comes “JUST before” the loaders, and why the enum starts with a dummy `ORDER_ZERO`. With about forty tables on the list in a full build, the trick halves the size of the table slots. The list’s own header comment is blunt about the cost that remains: “Don’t add a table unless it’s used though. This is not exactly zero-cost.”

The list is cut down at compile time. A build with `HB_NO_VERTICAL` has no `vhea`, `vmtx` or `VORG` slots, one with `HB_NO_COLOR` none for the color tables, and so on. Some tables a face uses aren’t on it at all, because an accelerator loads them for itself: `HVAR` and `VVAR` are loaded by the `hmtx` and `vmtx` accelerators, and the 24-bit `GLYF` and `LOCA` by the `glyf` accelerator.

### Loading without a lock

Every loader is a `hb_lazy_loader_t` (`hb-machinery.hh:185`), and every access, whether `face->table.GSUB->…`, `*face->table.head` or a plain `get()`, comes down to `get_stored` (`:230`):

```c++
Stored * get_stored () const
{
retry:
  Stored *p = this->instance.get_acquire ();
  if (unlikely (!p))
  {
    if (unlikely (this->is_inert ()))
      return const_cast<Stored *> (Funcs::get_null ());

    p = this->template call_create<Stored, Funcs> ();
    if (unlikely (!p))
      p = tagged_null (const_cast<Stored *> (Funcs::get_null ()));

    if (unlikely (!cmpexch (nullptr, p)))
    {
      do_destroy (p);
      goto retry;
    }
  }
  return untagged (p);
}
```

Once a table is loaded, this is one atomic load and a test. The load is an *acquire*, which guarantees that a thread that sees the pointer also sees everything that was written into the object before the pointer was published, so a half-built accelerator can never be seen.

When the pointer is `NULL`, the thread builds the object itself, then tries to publish it with a compare-and-swap: “if the slot is still `NULL`, put my pointer there”. If that works, the object is the face’s. If it doesn’t, another thread got there first. The loser destroys what it built and starts again from the top, where it now finds the winner’s pointer and returns that. Both threads end up with the same object, and nothing leaks.

The cost is that, in a race, the work is done twice, which for `GSUB` means sanitizing it twice. That’s rare, since it can only happen the first time a table is used, and the alternative, a lock taken on every access or a lock held while a large table is sanitized, would cost every caller every time. (`cmpexch` is a `compare_exchange_weak`, which is allowed to fail even when the slot is empty. That just means one more trip round the loop.)

No locks also means no deadlocks. A loader’s `create` often needs other tables: loading any table first asks for the glyph count, which loads `maxp`, and the `GDEF` accelerator checks the lengths of `GSUB` and `GPOS` against its list of known broken fonts, which loads them. Each of those is just another call to `get_stored`, on another slot. The one loop that would never end, `maxp` needing the glyph count to load `maxp`, is broken by hand: the `maxp` accelerator tells the sanitizer the glyph count is zero rather than asking (`hb-ot-maxp-table.hh:193`).

`is_inert` is true when the face pointer is `NULL`. That is the case only for the empty face, whose fields are all zero and live in read-only memory: its loaders return the Null object without trying to build or store anything.

### Two kinds of loader

`hb_table_lazy_loader_t` (`hb-machinery.hh:325`) stores a blob. Its `create` is two lines:

```c++
static hb_blob_t *create (hb_face_t *face)
{
  hb_sanitize_context_t c;
  return c.reference_table<T> (face);
}
```

and reading it turns the blob into the struct with `blob->as<T> ()`, which is a cast, or, if the blob is shorter than `T`’s minimum size, `T`’s Null object. `face->table.head` is used like a pointer to `OT::head`.

`hb_face_lazy_loader_t` (`:313`) stores an **accelerator**: an object built from one or more tables, with whatever the code that reads them wants to have ready. Its `create` is the default one, which allocates the accelerator and runs its constructor with the face. The constructor loads and sanitizes the table it’s built on, then does the rest of its setup.

| Accelerator | What it prepares |
|---|---|
| `cmap` | chooses the best subtable once, finds the variation-selector subtable, prepares format 4 for fast searching, and allocates a 1 KB cache of recent lookups |
| `GSUB`, `GPOS` | the table, its lookup count, and an array of empty slots, one per lookup, for the lookup accelerators |
| `GDEF` | the table, minus a few known-bad ones; small mark-filtering sets flattened into bitmaps |
| `glyf` | chooses `GLYF`/`LOCA` or `glyf`/`loca`, reads the `loca` format from `head`, and keeps pointers to the `gvar`, `hmtx` and `vmtx` accelerators it will need |
| `hmtx`, `vmtx` | the metrics table and its `HVAR` or `VVAR` |
| `maxp` | the glyph count, from `MAXP` if there is one, else `maxp` |

The accelerators are lazy inside, too. A `GSUB` accelerator doesn’t build the per-lookup accelerators (the set digests and subtable caches the [GSUB chapter](gsub.html) relies on) until a lookup is first applied. `get_accel` (`hb-ot-layout-gsubgpos.hh:6747`) fills each lookup’s slot with exactly the same acquire, build, compare-and-swap, destroy-and-retry pattern. So a face whose font has 500 lookups, of which a given text uses 20, builds 20 lookup accelerators. The `cmap` cache shared by every thread is an `hb_cache_t` (`hb-cache.hh`), whose entries are single atomic integers that hold both a key and a value, so a thread reading an entry that another is writing sees either the old pair or the new one, and at worst misses.

### Failure is cached as Null

A table that is missing, or that fails sanitizing, doesn’t leave its slot empty. `reference_table` returns the empty blob in both cases, and the empty blob is stored like any other, so the next access finds a non-`NULL` pointer and returns at once. The table is never looked for again, and never sanitized again, for the life of the face. Every caller then reads the Null object of the table’s type, all zeros, which (as [Fonts as bytes](fonts-as-bytes.html) explains) reads as a valid, empty table: a `GSUB` with no scripts, features or lookups. No code that uses a table has to check whether it loaded.

The [sanitizer chapter](trust-nothing-the-sanitizer.html#pass-or-fail) shows what that does to shaping. Seen from the face, it looks like this. `broken-gsub.ttf` is `ligatures.ttf` from the GSUB chapter with one offset changed: the GSUB header’s offset to its lookup list points far past the end of the table. The table is still in the font’s directory, with its 274 bytes. But when `hb-info` asks for the features, what it gets is the Null `GSUB`, so it lists none:

```info
broken-gsub.ttf  --list-tables
broken-gsub.ttf  --list-features
```

> For type designers: if a font shapes as though it had no layout at all, though `hb-info --list-tables` shows the table, a table that failed sanitizing is the likely reason.

For accelerators, the failure that matters is of a different kind. An accelerator is built even when its table is missing: it’s an accelerator for an empty table. What can fail is allocating it. Then `create` returns `NULL`, which can’t be stored, since `NULL` means “not loaded yet”. So the loader stores a **tagged Null** instead: the address of the accelerator’s Null object with its lowest bit set (`hb-machinery.hh:273`). Readers clear the bit and get the Null accelerator, and the destroy code recognizes the tag and frees nothing. A comment explains why it checks the bit and not the address: the Null pool is internal to each of HarfBuzz’s libraries (`libharfbuzz`, `libharfbuzz-subset` and the rest each have their own copy), so another library sharing the face would not recognize the Null object by its address. The same tagging is used for the per-shaper data on faces and fonts, whose Null is `NULL` itself.

## The first GSUB access, step by step

To put the pieces together, here is what happens the first time anything asks a fresh face for its `GSUB`. In an ordinary `hb_shape` call, that’s while the shape-plan cache builds its key: the key records which of GSUB’s feature variations apply (`hb-shape-plan.cc:106`), and that calls `get_gsubgpos_table` (`hb-ot-layout.cc:433`):

```c++
case HB_OT_TAG_GSUB: return *face->table.GSUB->table;
```

**1. The loader.** `face->table.GSUB->` calls `get_stored`. The slot is `NULL`, and the face pointer, found by stepping back from the slot, isn’t, so the loader calls `create (face)`, which allocates a `GSUB_accelerator_t` and runs its constructor.

**2. The accelerator.** The constructor (`hb-ot-layout-gsubgpos.hh:6710`, trimmed) asks the sanitizer for the table:

```c++
hb_sanitize_context_t sc;
sc.lazy_some_gpos = true;
this->table = sc.reference_table<T> (face);

this->lookup_count = table->get_lookup_count ();
this->accels = (hb_atomic_t<hb_ot_layout_lookup_accelerator_t *> *)
               hb_calloc (this->lookup_count, sizeof (*accels));
```

(`lazy_some_gpos` lets GPOS leave some checks until its data is used. It makes no difference to GSUB.)

**3. The glyph count.** `reference_table` (`hb-sanitize.hh:441`) first gives the sanitizer the number of glyphs, whatever the table, because some tables’ checks (in `gvar`, CFF, `sbix` and the AAT tables) compare against it:

```c++
if (!num_glyphs_set)
  set_num_glyphs (hb_face_get_glyph_count (face));
return sanitize_blob<Type> (hb_face_reference_table (face, tableTag));
```

If the face doesn’t know it yet, this goes through `load_num_glyphs` to `face->table.maxp`: another lazy slot, filled the same way.

**4. The bytes.** `hb_face_reference_table` calls the face’s `reference_table_func`, which for a face from a file is `_hb_face_for_data_reference_table`. It finds the face in the file, finds `GSUB` in its directory, and returns a sub-blob of the file blob: no copying, and no check of the contents yet.

**5. The check.** `sanitize_blob` (`hb-sanitize.hh:405`) casts the sub-blob’s bytes to `GSUB` and calls its `sanitize`, which walks every script, feature, lookup and subtable offset. The outcome is one of two:

```c++
if (sane)
{
  hb_blob_make_immutable (blob);
  return blob;
}
else
{
  hb_blob_destroy (blob);
  return hb_blob_get_empty ();
}
```

**6. The rest of the setup.** Back in the constructor, the lookup count is read from the table (zero from the Null table), and the array of lookup slots is allocated, all `NULL`. If that allocation fails, the accelerator gives up its table and holds the empty blob instead.

**7. Publishing.** `get_stored` tries to swap its new accelerator into the slot. If another thread has published one meanwhile, this thread destroys its own, which releases its reference to the table blob, which releases its reference to the file blob, and returns the other thread’s.

**8. Every call after.** The next access, from any thread, is the acquire load and nothing else. Code that runs during a GSUB lookup and needs the table again, such as a context lookup calling a nested one, uses `face->table.GSUB.get_relaxed ()` (`hb-ot-layout-gsub-table.hh:49`), a plain load without even the acquire, since by then this thread has certainly seen the accelerator.

## Sharing a face between threads

These pieces are what make a face safe to share. Everything a caller can change is changed before the face is made immutable, and the face is made immutable as soon as a font or a shape plan uses it. After that, the only writes are the lazy loaders’ (each a single compare-and-swap into a slot that starts `NULL` and is written once), the `upem` and `num_glyphs` fields (written with the same value by everyone), the caches inside accelerators (whose entries are atomic), and the shape-plan cache, a list that grows with the same kind of compare-and-swap and is covered in the chapter on [shape plans](shape-plans-and-their-cache.html). None of these needs a lock. User data, which an application adds and removes as it likes, has one.

So the pattern HarfBuzz is built for is: one blob per file, one face per face in it, shared by every thread, and a font for each size and set of variation settings, which [the next chapter](fonts-and-font-functions.html) covers. Fonts are cheap because faces carry everything that is expensive, and faces are cheap to open because nothing expensive happens until it’s needed. A build with `HB_NO_MT` replaces the atomics and mutexes with plain variables, for programs that use only one thread.

> For programmers: “immutable” is the contract to keep in mind. Set the face up (its index, upem, glyph count, table-tags function) before passing it to `hb_font_create` or to another thread. After that, it can be used from any number of threads without locks of your own. `hb_face_builder_add_table` doesn’t check the immutable flag, but a builder face is for assembling a font file, not for shaping with: a slot that has already loaded a table won’t notice a new one.

## Where to read the source

| File | What’s there |
|---|---|
| `src/hb-blob.h`, `src/hb-blob.hh`, `src/hb-blob.cc` | blobs: memory modes, `hb_blob_create_sub_blob`, making blobs writable, mapping and reading files |
| `src/hb-object.hh` | the object header: reference counts, the immutable flag, user data |
| `src/hb-face.hh`, `src/hb-face.cc` | the face, `hb_face_create`, `_hb_face_for_data_reference_table`, `hb_face_count`, face loaders, immutability |
| `src/hb-face-builder.cc` | the face builder, and writing its tables out as a font file |
| `src/hb-open-file.hh` | the containers: sfnt, TTC and dfont, and the table directory |
| `src/hb-machinery.hh` | `hb_lazy_loader_t`, `hb_table_lazy_loader_t`, `hb_face_lazy_loader_t`, tagged nulls |
| `src/hb-ot-face.hh`, `src/hb-ot-face-table-list.hh`, `src/hb-ot-face.cc` | the per-face table slots, generated from the table list |
| `src/hb-sanitize.hh` | `reference_table` and `sanitize_blob`, where loading meets checking |
| `src/hb-ot-layout-gsubgpos.hh` | the GSUB/GPOS accelerator and its lazy lookup accelerators |
| `src/hb-static.cc` | loading `upem` and the glyph count |

## In short

* A blob is bytes with a reference count and a destroy function. Font files are memory-mapped, and every table is a sub-blob of the file: nothing is copied.
* Nothing writes to font data anymore, so writable blobs, and the `mprotect` trick, are now rarely needed.
* A face gets its tables only through `reference_table_func`. `hb_face_create` checks only the file’s container, and never fails; `hb_face_create_or_fail` does.
* The low 16 bits of a face index pick a face in a collection. The high 16 bits, minus one, pick a named instance, which `hb_font_create` applies.
* Each table a face uses has a one-pointer slot, filled on first use. Racing threads both build, one wins the compare-and-swap, and the other destroys its copy.
* A missing or broken table is stored as the empty blob, so it is never retried, and code reads it as an empty table.
* Accelerators are built lazily in the same way, and so are the lookup accelerators inside GSUB and GPOS. Together with immutability, this lets one face be shared by every thread without locks.
