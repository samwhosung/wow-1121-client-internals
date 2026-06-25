# Image decode — BLP2 and TGA, palette/DXT/dither

The image-decode codec turns on-disk texture bytes into the pixel format the graphics device wants to
upload. It comprises two decoders — BLP2 (`engine\Source\BLPFile\blp.cpp`) and TGA
(`ENGINE\Source\Images\tga.cpp`) — plus a set of paletted/DXT/raw format converters. What must be exact here is
integer behaviour, not float: palette channel order (BGRA), DXT endpoint interpolation, and the 16-bit
quantize-and-dither all decide the exact displayed pixel, so a wrong decode is a wrong colour
on-screen.

The codec sits just above the [graphics-device](graphics-device.md) layer. The texture *service*
(`Services\Texture.cpp`, `0x44xxxx`) drives it, choosing a pipeline by file extension; file I/O is
delegated to Storm (`SFile*`, see [MPQ](mpq.md)), and the final device-format upload goes back to the
graphics device.

## Decode architecture

Two pipelines, selected by extension:

- **TGA**: `Open`+parseHeader `0x5a3660` → `Decode 0x5a3a30(flags)` (dispatch on imageType) → flip
  `0x5a4070` → fetch `0x5a41f0`. (A generic loader path also reaches it via `0x52594f`.)
- **BLP2**: `Open 0x5a7b40` (Storm reads the whole file → parseHeader `0x5a83b0`) → the service queries
  the gx device format (`0x58a230`) → `DecodeAllMips 0x5a8240` → `Reset 0x5a7aa0`. Per level,
  `DecodeMip 0x5a7f00` dispatches on `compression` (`+0xC`): palettized → convert `0x5a8ce0`; DXTC →
  matrix `0x5a4f60`; raw → `memcpy`.

## State structures

### TGA `CImage` — header overlaid at `+0x08`

| Offset | Field |
|---|---|
| `+0x00` | `SFile*` |
| `+0x04` | pixels |
| `+0x08` | idLength u8 |
| `+0x09` | colorMapType u8 |
| `+0x0a` | imageType u8 (1=cmap, 2=RGB, 3=gray, 9/10/11=RLE of those) |
| `+0x0b` | colorMapFirstIndex u16 |
| `+0x0d` | colorMapLength u16 |
| `+0x0f` | colorMapEntrySize u8 (bits) |
| `+0x10` | xOrigin u16 |
| `+0x12` | yOrigin u16 |
| `+0x14` | width u16 |
| `+0x16` | height u16 |
| `+0x18` | bpp u8 (mutated during decode) |
| `+0x19` | descriptor u8 (`&0xf` = alpha/attr bits, bit4 = horiz-flip, bit5 = top-origin) |
| `+0x1c` | idBuf |
| `+0x20..+0x39` | 26-byte TGA footer (`+0x28` = "TRUEVISION-XFILE" sig `0x85c854`) |
| `+0x3c` | sizeBytes u32 |
| `+0x40` | palette |

Shared idioms:

```
entryBytes    = (min(colorMapEntrySize / 3, 8) * 3) >> 3   // the /3 compiles to imul 0x55555556
bytesPerPixel = (bpp + 7) >> 3
```

### BLP `CBLPFile` — ~`0x4b4` bytes; on-disk 1172-byte header copied to `+0x04`

| Offset | Field |
|---|---|
| `+0x00` | decodedMips ptr (owned buffer: `[count ptrs][mip0 px][mip1 px]…`) |
| `+0x04` | magic "BLP2" = `0x32504c42` |
| `+0x08` | version == 1 |
| `+0x0C` | compression u8 (1=palettized, 2=DXTC, 3=raw-BGRA) |
| `+0x0D` | alphaDepth u8 (0/1/4/8) |
| `+0x0E` | alphaType u8 (DXT1/3/5 select) |
| `+0x0F` | hasMips u8 |
| `+0x10` | width u32 |
| `+0x14` | height u32 |
| `+0x18` | mipOffsets[16] u32 (file-relative) |
| `+0x58` | mipSizes[16] u32 |
| `+0x98` | palette[256] u32 **BGRA** (`[0]=B, [1]=G, [2]=R, [3]=A`) |
| `+0x498` | fileImage ptr (shared static read-buffer, not owned) |
| `+0x49c` | flag = 0 |
| `+0x4a0` | mipCount (= ComputeMipLevels(w,h) if hasMips else 1) |
| `+0x4b0` | pendingScratch ptr |

Module statics — one global read buffer: `0xc15c40` base / `0xc15c44` size / `0xc15c48` ptr /
`0xc15c4c` gx-alignment.

## Two distinct format enumerations

**gx device-format id** — the `format` argument threaded through BLP decode. The enum's meaning belongs
to the graphics device (queried via `0x58a230` / `0x59fc50`); this codec consumes and branches on it:

| id | meaning |
|---|---|
| 0, 1, 7 | raw passthrough (device eats DXT/native) |
| 2 | BGRA8888 |
| 3 | ARGB1555 (16-bit) |
| 4 | ARGB4444 (16-bit) |
| 5 | RGB565 (16-bit) |
| 9 | RGB565 + 2-bit alpha plane |

The bpp leaf `0x5a49b0` maps: `0→4`, `{1,6,7}→8`, `2→32`, `{3,4,5}→16`. The size leaf `0x5a62c0` uses
`shift[0,0,0,0,0,2,2,2,0]` and `mult[0,4,2,2,2,8,16,16,2]`.

**conversion-matrix class** — internal to the dispatcher `0x5a4f60`, keyed by src × dst × submode:
`1=BGRA8888, 2=ARGB4444, 3=ARGB1555, 4=RGB565, 5=DXT1, 6=DXT3, 7=DXT5, 8=other16`.

## TGA pipeline `[0x5a3600, 0x5a4810)`

`Open 0x5a3660` reads the 18-byte header into `+0x08`, then loads the colormap
(`entryBytes·colorMapLength`) and the ID field. `Decode 0x5a3a30` dispatches on imageType:

- 1 / 9 (colormapped) → `0x5a39d0`
- 2 (RGB) → `0x5a3c80`
- 10 (RLE-RGB) → `0x5a3d70`
- 3 / 11 (grayscale) — unsupported

Supported types are 1, 2, 9, 10. Component primitives:

- **paletted→RGB expand** `0x5a3820`
- **pixel-data file offset** `0x5a3b40` = `entryBytes·colorMapLength + idLength + 0x12`
- **append-alpha interleave** `0x5a3b80` (24→32: opaque-pad or alpha-source)
- **RLE decode loop** `0x5a3ed0`: run packet (hi-bit set) copies one pixel `(b & 0x7f) + 1` times; raw
  packet copies `b + 1` literal pixels; imageType is normalized `−= 8` on exact fill
- **vertical flip** `0x5a4070`: reverse-row memcpy by `stride = bytesPerPixel·width`, toggling descriptor
  bit5
- **32→24 alpha-strip** `0x5a4250`
- **CImage init-from-source** `0x5a43c0` (size / descriptor-pack / footer)
- **TGA write/serialize** `0x5a4810` (owns the colormap-size math; file I/O is a host `CFile` boundary)

## Shared mip-geometry `[0x5a49b0, 0x5a4bb0]`

Format-agnostic; called by both BLP decode and the texture service (which caches mip pixels in a
`CMipBitsCache` and dedupes loaded textures through a `CTextureHash`). A mip chain is one contiguous
block: a leading array of `count` 32-bit absolute pointers, then the concatenated mip pixel data:

```
mipTable[i] = dataBase + Σ_{j<i} levelSize(j)
```

Primitives:

- **bpp-by-format** `0x5a49b0`
- **per-level size** `0x5a4a00`: `wL = max(1, w >> L)`, `hL = max(1, h >> L)`; formats `{0,1,7}` clamp
  `≥4` (DXT 4×4 blocks); format 9 → `wL·hL·2 + max(1, wL·hL >> 2)`; otherwise `bpp·wL·hL >> 3`
- **Σ sizes** `0x5a4a80`
- **level count** `0x5a4ac0` = `floor(log2(max(w,h))) + 1`
- **alloc + populate** `0x5a4af0`
- **total block size** `0x5a4b80` = `Σ + 4·count`
- **populate-into-existing** `0x5a4bb0`

## Pixel-format conversion matrix `[0x5a4c30, 0x5a5c80]`

The dispatcher `0x5a4f60` indexes a function-pointer matrix at `0xc0f560` (lazily filled by `0x5a4fc0`,
guard `0xc0f558`):

```
slot = matrix[dst·4 + src·0x24 + submode]
```

Converter ABI: `__fastcall(ecx = dims{w,h}, edx = src, stk: srcStride, dst, dstStride)`. This matrix is
reached only from the BLP *uncompressed* path (`0x5a7f00` / `0x5a8590`). Converters:

| Converter | VA | Formula |
|---|---|---|
| BGRA8888 → ARGB4444 | `0x5a5070` | `(A>>4)<<12 \| (R>>4)<<8 \| (G>>4)<<4 \| (B>>4)` |
| BGRA8888 → ARGB1555 | `0x5a51d0` | `(A>>7)<<15 \| (R>>3)<<10 \| (G>>3)<<5 \| (B>>3)` |
| BGRA8888 → RGB565 | `0x5a5330` | `(R>>3)<<11 \| (G>>2)<<5 \| (B>>3)` |
| plain copy | `0x5a5480` | — |
| alpha-keyed copy | `0x5a5510` | — |
| source-over blend | `0x5a5590` | `dst_c + ((src_c − dst_c)·a >> 8)` |
| 16bpp copy | `0x5a56b0` | — |
| DXT1 block copy | `0x5a5740` | — |
| DXT3/5 block copy | `0x5a5780` | — |
| DXT1 → RGB565 decode | `0x5a57c0` | (driver) |
| DXT1 → ARGB1555 decode | `0x5a5a20` | (driver) |
| DXT1 → BGRA8888 decode | `0x5a5c80` | (driver) |
| box-filter mip downsample | `0x5a4c30` | alpha-weighted RGB averaging |

## DXT / S3TC decode `[0x5a5ef0, 0x5a79f0]`

Software decode covers **DXT1 and DXT3** into RGB565 / ARGB1555 / ARGB4444 / ARGB8888. There is **no
software DXT5**: DXT5 (format 7) is size-aware but wired only to the passthrough copier `0x5a5780`,
handed to the GPU as native S3TC.

The shared interpolation LUT `0xc0fa80` is generated once by `0x5a7910`, with sub-tables `(n·256)/3 + 1`
and `(n·512)/3 + 1`. Endpoint interpolation then reads as a combination of the two:

```
tblB[c0] + tblA[c1] >> 8  ==  (2·c0 + c1) / 3
```

DXT1 punch-through selection:

- `color0 > color1` → 4-colour palette `{ c0, c1, (2·c0+c1)/3, (c0+2·c1)/3 }`
- otherwise → 3-colour palette `{ c0, c1, (c0+c1)/2, transparent-black }`

Per-block decoder ABI: `__fastcall(ecx = block, edx = &dstRowPtrs, [+8] = &bounds, [+0xc] = alphaCb)`.

## Paletted → RGB converters `[0x5a83b0, 0x5a9db0)` — Floyd-Steinberg dithering

`parseHeader 0x5a83b0` copies the 1172-byte header (`rep movsd 0x125`), validates magic + version, and
sets mipCount. Per mip, Stage-1 dispatch `0x5a8590` branches on compression; Stage-2 dispatch `0x5a8ce0`
branches on gx-format to the converter:

| gx format | converter |
|---|---|
| 2 | `0x5a8790` (alpha==8) / `0x5a87d0` (alpha≠8) — palette-expand → BGRA8888 |
| 3 | `0x5a9580` |
| 4 | `0x5a92d0` |
| 5 | `0x5a9860` |
| 9 | `0x5a9a80` |

The four 16-bit converters dither with the classic Floyd-Steinberg kernel. They share a two-row
ping-pong error buffer in static BSS (bases `0xc0fc10` / `0xc1bc98` / `0xc21cd0` / `0xc15c68`; stride
`0x3018`; row-parity by `row & 1`; three ints R, G, B per pixel; zeroed on entry). Per pixel:

```
accum = (palChan << 16) + (incomingError >> 4)     // the /16 of the kernel is the >>4 on consume
accum += bias
value  = clamp(accum arith-shifted to the target depth)
```

The quantization error then distributes by the **{7, 5, 3, 1}/16** kernel: ×7 to the next pixel in the
same row; ×5, ×3, ×1 to the row below. Palette is read in the order R = `[+0x9a]`, G = `[+0x99]`,
B = `[+0x98]`. (Per-format bias/shift/clamp/pack values differ per converter.)

## Dead code (real, with zero in-binary callers)

These are fully-formed primitives that no live decode path reaches — the live build writes uncompressed
TGA and dithers only via Floyd-Steinberg:

- the TGA RLE-*encode* subtree `0x5a4710` → `0x5a4570` → `0x5a44f0`, plus orient re-init `0x5a4350`
- the no-dither converter family: dispatcher `0x5a8c00` + `0x5a8910` / `0x5a8a40` / `0x5a8b80`
- the ordered-dither family `0x5a8ea0` / `0x5a9030` / `0x5a91d0`

They remain deterministic and behave identically if invoked directly; they simply never run during a
normal texture load.
