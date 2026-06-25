# DBC — the WDBC static data tables

The client's read-only static data lives in `DBFilesClient\*.dbc` files in the `WDBC` format, read out
of the MPQ asset VFS. They hold the data the client needs but does not compute: localized startup
strings, spells, items, maps, area/light/terrain-type tables, sounds, races/classes, and more. **151
distinct `DBFilesClient\*.dbc` tables** are referenced in the binary. Every one is the same fixed-layout
container; only the per-table row schema differs. This chapter documents the container format, the
generic loader, and how a table's per-column schema is recovered.

## The WDBC container format

A **20-byte little-endian header** (no byte-swap on x86), then fixed-width records, then a string block:

| Offset | Field | Type |
|---|---|---|
| `+0x00` | magic `'WDBC'` = `0x43424457` | u32 |
| `+0x04` | recordCount | u32 |
| `+0x08` | fieldCount | u32 |
| `+0x0c` | recordSize | u32 |
| `+0x10` | stringBlockSize | u32 |

The magic is compared at `0x4025bc` (`cmp [ebp-0x18], 0x43424457`).

After the header come `recordCount × recordSize` bytes of **records**. Each record is `fieldCount`
columns, and columns are **4-byte little-endian cells** — interpreted as u32, i32, f32, or a u32 *offset
into the string block* for string columns. The cells are untyped in the file; what a cell *means* is
decided by the table's row schema, not by the container.

After the records comes a **string block** of `stringBlockSize` bytes: NUL-terminated UTF-8/locale
strings. **Offset 0 is the empty string.** A string column's value is a byte offset into this block.

A valid file satisfies `header + records + string block == file length` (`20 + recordCount·recordSize +
stringBlockSize`).

The first column of every record is the record **id**; the loader builds an id→record index from it (see
below). Two concrete examples to validate against real data:

- `Startup_Strings.dbc`: **fieldCount == 11** (`cmp eax, 0xb` at `0x402618`), **recordSize == 44**
  (= 11 × 4).
- `Spell.dbc`: 22357 records × 173 fields, recordSize 692.

## Loading: the generic `WowClientDB<T>` loader

A DBC is the **first asset the client reads through the VFS at startup**. The main startup function
`0x402350`, after mounting MPQ and setting up config/CVars and locale, opens
`DBFilesClient\Startup_Strings.dbc` and WDBC-parses it inline using the MPQ VFS primitives: open
`0x6477a0`, read `0x648460`, close `0x648730`. The bulk of the other tables load later through the
generic templated loader `WowClientDB<T>` (binary source paths `Source\DB\WowClientDB.hpp` / `.h`,
string table `0x82e210`).

The loader entry points:

- `WowClientDB<T>::Load` @ `0x53f8b0` — WDBC-parse the file, `SMemAlloc` the record array, then build a
  **dense id index** (`idIndex[id] = &record`).
- `ClientDBInitialize` @ `0x53f4f0` — builds the consumer-feeding secondary cross-indexes.

Each loaded table is a **0x14-byte in-memory POD**:

```
{ recordsPtr, recordCount, idIndexPtr, maxId, loadedFlag }
```

There are ~155 such instances in `.bss` at `0xc0e0xx`. For example, AreaTable's record array (100-byte
stride) is at `.bss 0xc0e040` with its count at `0xc0e044`.

The DBC files resolve through the [MPQ](mpq.md) VFS, so the patch chain applies — a patched
`Startup_Strings.dbc` is served from `patch.MPQ`. The `.dbc` files are MPQ-only (`./WoW/Data/dbc.MPQ`),
not loose on disk.

## Localized strings

A localized-string field is **not** a single column. It is a set of per-locale string-block offsets:
**8 locale columns** in vanilla 1.12.1, immediately followed by a **flags column** (9 columns total).
The runtime picks which of the 8 offsets to read by **locale index**.

The locale index is at `ds:0xc0e080`. It is computed from the `Wow.ini` Region/Language settings into a
locale tag at `ds:0xc2a2a4`, which selects the column. The column read is:

```
value_offset = record[ base*4 + locale*4 ]      // i.e. record + base*4 + locale*4
```

This is the `0x4027c0` access `mov eax, [eax + edx*4 + 8]` (here base field 2, byte offset `+8`). For
`Startup_Strings`, the localized base is **field 2**; record 0's enUS column (locale 0) resolves to
`"World of Warcraft"`.

The 8-locale column order is the vanilla convention:

```
enUS, koKR, frFR, deDE, zhCN, zhTW, esES, esMX
```

An enUS-only install fills only column 0; the other seven are empty string-block offsets. The mechanism
to reproduce is: a localized field is a fixed run of 8 offset columns + a flags column, and the active
column is `locale` (from the locale tag), read with `record + base*4 + locale*4`.

## Per-table row schemas (`T::Read`)

Each table has a generated decode method, `T::Read`, that fully describes its column layout. The schema
of any of the 151 tables can be recovered from it.

**Finding a table's `Read`.** Each table has an error string `Error reading <Name>Rec` (in `.data` from
`0x857e6c`), and each is referenced by exactly one `.text` site — inside that type's `T::Read`. For
example `AreaTableRec`'s error string `0x857ee8` is referenced at `0x574348`, inside `AreaTableRec::Read`
@ `0x574070`. `Read` is a direct callee of that table's `WowClientDB<T>` load loop (AreaTable's load loop
is `0x53fc00`).

**The schema is the `Read` call schedule.** `Read` has the signature:

```
T::Read(this = &record, ctx = column-cursor, stringBlockBase)
```

and issues one call to the generic typed-column reader per logical column:

```
0x648460(ctx, dest, size)
```

`0x648460` dispatches on the cursor's per-column type tag `[ctx+0]` via a 5-way jump table at `0x6485d8`
(lock-bracketed by `0x6579b0` / `0x6579c0`). The *sequence* of `(struct-offset, size, is-string)` reads
is the table's schema.

**Deriving `(fieldCount, recordSize)` from the schedule.** These are deterministic from the read sizes:

```
recordSize  = Σ(read sizes)
fieldCount  = Σ( size/4  for each read with size ≥ 4  [u32 cells / arrays]
                 1       otherwise                    [byte cells] )
```

Across all 151 tables this matches the real file headers: field counts range 2…173, record sizes 2…692.
The byte-cell case matters for two byte-packed tables that a uniform-4-byte assumption would miss:
`CharBaseInfo` (2 columns / 2 bytes) and `CharStartOutfit` (41 columns / 152 bytes). `Startup_Strings`
derives to 11 columns / 44 bytes, matching the `cmp eax, 0xb` @ `0x402618`.

**String vs scalar columns.** A string-block-offset column is read into a **stack temp** and then fixed
up in the success path via `add reg, stringBlockBase` — that pattern marks a column as a string. 91 of
the 151 tables carry string columns (621 columns total). A **run of 8 consecutive string columns** is a
localized string (the 8 locale offsets described above; the localized tail also reads the flags cell and
applies `stringBlockBase` to each of the 8 offsets in the success tail, e.g. `0x57435c`). Single or other
runs are plain `char*` or `char*` arrays — for example `SoundEntries`' `SoundFile[10]` and `Map`'s
`Directory`. Note `Startup_Strings` reads one offset directly to the struct (not via a temp), an
off-by-one against the temp-read pattern.

The distinction between a localized 8-offset run and a `char*[8]` array is not separable from `Read`
alone (both are offset + fixup), and int-vs-float intent on a scalar cell is likewise undecidable from
the container — the cell is an untyped 4 bytes and the cursor types are data-driven. Those are consumer
concerns, decided by the consuming subsystem ([Lighting](lighting.md) for `Light*.dbc`,
[Terrain](terrain.md) for `TerrainType.dbc`, etc.).

### Worked example: `AreaTable`

`AreaTableRec::Read` @ `0x574070` yields **25 columns / 100-byte records**:

| Struct offset | Columns | Meaning |
|---|---|---|
| `[0x00 .. 0x28]` | 11 × u32 | scalar fields |
| `[0x2c .. 0x48]` | 8 offsets | `AreaName` localized string (base field 11) |
| `[0x4c]` | flags | the localized field's flags cell |
| `[0x50]` | 1 × u32 | scalar |
| `[0x54 .. 0x63]` | 4 columns / 16 bytes | tail |

The real `AreaTable.dbc` has field_count = 25, record_size = 100, 1081 rows; the `AreaName` localized
field (base field 11), record 0, reads `"Dun Morogh"`.

## DBCache — the server-streamed path is separate

There is a second, distinct way the client gets table-like data: the **server-streamed DBCache**
(`0x554ff0`), which registers query/response SMSG handlers over the network rather than reading `.dbc`
files. It is a different code path with its own decoders and is **not** the static `.dbc` path — the
static path is `0x648460`, the DBCache path is `0x554ff0`. The DBCache handlers are registered through
the net layer (`0x5ab650` → `0x537a60` `NetClient::RegisterHandler`, 14 query-response handlers). See the
[DBCache](dbcache.md) and [Net](net.md) chapters; do not conflate the two.
