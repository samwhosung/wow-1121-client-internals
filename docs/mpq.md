# MPQ asset VFS — the archive filesystem

The read-only virtual filesystem the client mounts at startup and reads every asset through — DBCs,
textures (BLP), models (M2/WMO), terrain (ADT), sound, interface. It is Blizzard's Storm `SFile`
layer over MPQ archives, statically linked into `WoW.exe` (the binary carries the source paths
`E:\build\buildWoW\Storm\Source\SFile.cpp` / `SFile2.cpp`; there is no `storm.dll` import). It is the
first foundational subsystem brought up: mounted before config, locale, DBC, and the graphics device,
so every downstream subsystem resolves names through it. The client uses the original MPQ **v0**
format.

## Mount

The startup function (`0x402350`) brings the VFS up first:

1. `0x5aa2d0` opens `Data\base.MPQ` and reads `telemetry.dat` — the earliest `SFile` use.
2. **`0x403740`** mounts the asset set. It walks an archive-name pointer table at `0x82e12c` and opens
   each via `SFileOpenArchive` (`0x648dd0`, which branches on a `.mpq` name to the real open
   `0x655670`), searching the roots **`Data\`** then **`..\Data\`**; failures log
   `"Failed to open archive %s: %s"`.
3. `speech2.MPQ` is optional: the `disableOptionalSpeech` CVar unmounts it (`SFileCloseArchive`). See
   [CVars](cvars.md).

The archive list, in mount order:

```
model.MPQ, texture.MPQ, terrain.MPQ, wmo.MPQ, sound.MPQ, misc.MPQ,
interface.MPQ, fonts.MPQ, speech.MPQ, dbc.MPQ, speech2.MPQ
```

plus the **patch chain** (`patch.MPQ`, `patch-?.MPQ`) from a separate table at `0x82edb0`.

## Override resolution — patches win

Two mechanisms coexist; their net effect is that **a patch archive overrides a base archive's copy of
a file**.

**(a) The global open-archive list** at `0xc53fe0`, kept **sorted descending by mount priority**
(`archive+0x148`). The insert at `0x656012` walks past every node whose priority is `>` the new one
and links before the first `<=`. A bare-name lookup walks this list **head-first**, so among
independently-mounted archives the **later-mounted (higher priority number) one wins**.

**(b) A per-base-archive patch-overlay chain** at `archive+0x170`, walked **recursively** by
`0x6549a0` — the mechanism by which `patch.MPQ` / `patch-?.MPQ` override a base archive's copy of a
file.

The mounter (`0x403740`) opens the **patch chain first** (priority counter starting from `0x40`, the
`0x82edb0` loop), then the **base archives** (the `0x82e12c` loop, which get higher priorities). The
net result is that patches take precedence: a file present in both a base archive and a patch (for
example `Character\NIGHTELF\Female\NIGHTELFFEMALEEYEGLOW.BLP`, or even
`DBFilesClient\Startup_Strings.dbc` itself) resolves to `patch.MPQ`.

Subsystems request files by path (`DBFilesClient\*.dbc`, `World\…\*.adt`, `*.blp`, `*.m2`) and receive
bytes; they never see archives or the patch chain. The mount order plus this override priority are the
VFS's contract to the rest of the client.

## Read API

The Storm `SFile` entry points in the binary — the behavioural contract a reader must match:

| Function | VA |
|---|---|
| `SFileOpenArchive` | `0x648dd0` (→ `0x655670`) |
| `SFileOpenFile` / `SFileOpenFileEx` | `0x6477a0` / `0x6477c0` |
| `SFileReadFile` | `0x648460` |
| `SFileLoadFile` | `0x648620` |
| `SFileCloseFile` | `0x648730` |
| `SFileCloseArchive` | `0x648ef0` |
| name-hash table backing | `0x63db90` |

## MPQ on-disk format

### Header (`0x655bf0`)

A **32-byte v0 header**, located by scanning **0x200-aligned** offsets for the magic. All table and
file offsets are relative to the accepted header base.

| Offset | Field | Notes |
|---|---|---|
| `+0x00` | magic `'MPQ\x1a'` = `0x1a51504d` | compared at `0x655c7f` |
| `+0x04` | header size | checked `>= 0x20` |
| `+0x08` | archive size | |
| `+0x0c` | format version (u16) | read but never validated; this client is v0-only |
| `+0x0e` | sector-size shift (u16, low byte) | `sectorSize = 0x200 << shift` (`0x655d0a`) |
| `+0x10` | hash-table offset | |
| `+0x14` | block-table offset | |
| `+0x18` | hash count | checked `<= 0x10000` |
| `+0x1c` | block count | |

### Hash table

`count` × **16-byte** entries, decrypted with key `HashString("(hash table)", 3)`.

| Offset | Field |
|---|---|
| `+0x00` | nameHashA (u32, hash type 1) |
| `+0x04` | nameHashB (u32, hash type 2) |
| `+0x08` | locale (u16) |
| `+0x0a` | platform (u8) |
| `+0x0c` | block index (u32) |

A block index of `0xffffffff` marks a free slot (and ends a probe); `0xfffffffe` marks deleted.

**Lookup** (`0x6477c0`): `start = HashString(name, 0) & (count − 1)` — a power-of-two **mask**, not a
modulo — then linear-probe matching `(nameHashA, nameHashB)` plus locale/platform (a value of `0` is a
wildcard) to recover the block index.

### Block table

`count` × **16-byte** on-disk entries (expanded to `0x2c` bytes in memory at `0x656130`), decrypted
with key `HashString("(block table)", 3)`.

| Offset | Field |
|---|---|
| `+0x00` | file offset (relative to header base) |
| `+0x04` | packed size |
| `+0x08` | unpacked size |
| `+0x0c` | flags |

Flags (each confirmed at the point it is tested):

| Flag | Meaning |
|---|---|
| `0x00000100` | IMPLODE (PKWARE DCL) |
| `0x00000200` | COMPRESS (multi-codec) |
| `0x00010000` | ENCRYPTED |
| `0x00020000` | FIX_KEY |
| `0x01000000` | SINGLE_UNIT |
| `0x80000000` | EXISTS |

### Sector layout (`0x651450`)

A compressed or imploded file is split into `sectorSize` chunks:

- A `(numSectors + 1)` × u32 **sector-offset table** sits at the file's start. `table[i+1] − table[i]`
  is sector `i`'s packed size; `table[numSectors]` is the total packed size.
- A SINGLE_UNIT file is a single chunk with no offset table.
- A sector whose packed size equals `sectorSize` is stored **verbatim** (incompressible).

### Compression

For a COMPRESS sector the **first byte is a codec mask**; the dispatcher (`0x661a80`, table at
`0x80fa1c`) applies one codec per set mask bit:

| Mask | Codec | Function |
|---|---|---|
| `0x01` | Huffman | `0x661350` |
| `0x02` | Zlib | `0x660800` |
| `0x10` | 6th-codec | `0x660c10` |
| `0x20` | Sparse | `0x660ac0` |
| `0x40` | ADPCM-mono | `0x660f60` |
| `0x80` | ADPCM-stereo | `0x6611e0` |

PKWARE (`0x08`) is **absent** from this multi-mask table — imploded files use the IMPLODE flag
(`0x100`) and a separate explode path. So in practice vanilla data is **Zlib (`0x02`) or uncompressed
for almost everything**, PKWARE-implode for legacy archives, and ADPCM/Huffman/Sparse appear only
inside WAVE sound data. There is no bzip2 or LZMA on this path (a deviation from the later generic MPQ
spec).

### Encryption

Storm's crypt table plus a key-derived block cipher.

- **Crypt table**: a 0x500-entry `u32` table built from seed `0x00100001` with the LCG
  `seed = (seed·125 + 3) mod 0x2AAAAB`, run in three passes (stored at `0xc53fbc`).
- **`HashString(s, type)`**: seeds `0x7fed7fed` / `0xeeeeeeee`, multiplier `0x21`, with the name first
  **uppercased and `/`→`\`** normalised. `type` selects the table quadrant: `0` = hash-table index,
  `1`/`2` = name-A / name-B verify, `3` = key.
- **DecryptBlock** (`0x651aa0`): the key-stream draws from table offset `+0x400`, advancing per dword
  as `key = ((~key << 0x15) + 0x11111111) | (key >> 0x0b)`.
- **File key** = `HashString(baseName, 3)`. If FIX_KEY is set:
  `key = (key + fileOffset) ^ fileSize` (`0x65687b`). The **sector-offset table** is decrypted with
  `fileKey − 1`, and **sector `i`** with `fileKey + i`.

## Downstream

Every other subsystem reads its data through this VFS: [DBC](dbc.md) tables, [BLP](image.md) textures,
[Models](models.md) (M2/WMO), [Terrain](terrain.md) (ADT), [Sound](sound.md), and the interface. DBC
table parsing and BLP texture decoding are themselves downstream formats read out of this VFS, opened
after the VFS is up. See also [Core](core.md) for the startup function that mounts it.
