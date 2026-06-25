# Terrain — the ADT world surface

How the 1.12.1 client builds and draws the outdoor world surface: it streams **ADT** tiles from MPQ,
turns each tile's heightmap into a camera-relative mesh, layers blended textures and baked shadow over
it, fills animated liquid surfaces, answers height/liquid/ground-effect queries against the grid, and
emits an ordered draw list for the world-scene frame. The geometry math is pure 2004-era format and
coordinate work; the GPU submit underneath it is the [graphics device](graphics-device.md)'s.

## The class hierarchy and file layout

WoW 1.12.1 terrain is the **ADT** tile system: `CMap`/`CWorld` (world/tile manager) → `CMapArea` (one
ADT tile, 16×16 chunks) → `CMapChunk` (one cell), plus `CMapAreaLow` (the WDL low-detail tile). The
file pattern is `World\Maps\<map>\<map>_%d_%d.adt` (format string `%s\%s_%d_%d.adt` at `0x86c368`).

The relevant classes and constructors:

| Class | vtable | ctor | pool elem |
|---|---|---|---|
| `CMapArea` | `0x810fdc` | `0x6c1c80` | WAREA `0x6a8` |
| `CMapChunk` | `0x810bf8` | `0x6af1d0` | WCHUNK `0xf34` |
| `CMapLight` | `0x810ef4` | `0x6c0920` | WLIGHT |
| (shared async base) | `0x8110a4` | `0x6c3020` | — |

`CMapAreaLow` (the WDL tile) is inline-constructed in `CMap::LoadWdl` `0x6944a0` and has no vtable of its
own. The WMO/doodad placement classes (`CMapObj`, `CMapObjGroup`, `CMapDoodadDef`, `CMapObjDef`) live in
the same code window but belong to [Models](models.md).

## The on-disk format

The loader is **offset-table-driven and never reads a chunk tag**. Every record is `{u32 tag (unread),
u32 size}` followed by its data; stored offsets point at a sub-record's 8-byte header, and the code adds
`+8` to reach the payload. The tag names (MCNK, MCVT, …) below are labels from corroborating files, not
strings the code matches.

### WDT — the tile presence table (`0x694760`)

`MVER` · `MPHD` (0x20 bytes; only dword0 bit0 is read = WMO-only map) · `MAIN` (0x8000 bytes = 64×64×8B
`{u32 flags, u32 unused}`) → stored at `0xc89f50`, indexed `y·64 + x`. `flags` bit0 = tile exists; bit1
and the dword1 slot are reused at runtime. The WMO-only branch follows the `MWMO` path and one `0x40`
`MODF` entry → `0x695650`.

### WDL — the low-detail far tile (`0x6944a0`)

`MAOF` is a 64×64 table of `u32` absolute offsets; each present tile has 545 `s16` heights, loaded with
`fild` directly (no scale) into `CMapAreaLow+0x3c`. The vertex order is the 17×17 outer grid row-major,
then the 16×16 inner grid (consumer `0x6bd8c0`).

### ADT — the full tile (prepare `0x6c2720` → async `0x6c1e70` → parse `0x6c2010`, callback `0x6c21f0`)

The whole file is held resident at `area+0x678`. `MHDR = buf + *(buf+4) + 0x10`, a `0x40` table of
offsets into:

- **MCIN** — 256×16B; only `+0x00` (the absolute MCNK offset) is read.
- **MTEX** — NUL-separated strings → texture handles at `area+0xec`.
- **MMDX / MMID / MWMO / MWID** — model/WMO name and index tables.
- **MDDF** — doodad placements, 36-byte entries.
- **MODF** — WMO placements, 64-byte entries.

Placement transform (doodad/WMO): `world = (17066.666 − f[+0x10], 17066.666 − f[+0x08], f[+0x0c])`;
rotation is degrees `× π/180` (with `+π` added on the second axis); scale is `u16 / 1024`. The ADT
origin computation (`0x6c2720`) lands the tileY origin leg at `+0x7c` — note the axis swap.

### MCNK — the per-chunk header (build `0x6c2880` → `0x6af5f0`, fixup `0x6af970`)

`0x80`-byte header. Sub-record offsets are relative to the record start and get fixed up into
`chunk+0xf14 .. +0xf30`. Header fields:

| Offset | Field |
|---|---|
| `+0x3c` | `u16` holes (4×4 quad mask, table `0x8103f8`) |
| `+0x40` | `u16[8]` 2-bit low-res layer map |
| `+0x50` | `u8[8]` no-doodad bits |
| `+0x68 .. +0x70` | chunk position (`z` = the height base) |

Flags: bit0 = shadow present; bits2–5 = which of four liquid slots are present; bit15 = alpha-full.

`ix`/`iy` (`+0x04`/`+0x08`), `sizeAlpha` (`+0x28`), and `sizeLiquid` (`+0x64`) are **dead — never read**.

### The MCNK arrays

| Array | Layout | Destination / consumer |
|---|---|---|
| MCVT | 145 relative `f32`, 9-row/8-row interleave | world verts at `chunk+0x83c` (consumer `0x6b0e50`) |
| MCNR | 145×3 `s8`, scaled `×1/127` | `chunk+0x170` (`mcnr_normals` `0x6aff10`) |
| MCLY | 16B `{texIdx, flags (bit8 = alpha), alphaOfs, effectId}` | per-layer texture binding |
| MCAL | 2048B, 4-bit, 64×64, low-nibble-first | alpha blend (bit15 clear → 63×63 + duplicated last row/col, `0x6b0bf0`) |
| MCSH | 512B, 1-bit, LSB-first | baked shadow |
| MCRF | doodad-then-WMO `u32` refs (×36 / ×64 into MDDF/MODF) | object references |
| MCSE | 52B entries | sound emitters |
| MCLQ | one `0x324`B block per set liquid bit | liquid surface (consumer `0x68d540`) |

The MCLQ block is `{min, max f32; 81×8B = 9×9 vertices (f32 height at +4); 8×8 cell flags; flow}`.

The field layout above is confirmed from the binary except for a few inferred labels — the
`areaId`/`impassible`/`doodadSet`/`nameSet` field names, the MCSE/MCLQ field semantics, and the MCNK
header-bit→liquid-class ordering (bits 2–5) remain inferred. The *rendered* per-tile liquid type
selection (below) is confirmed.

## Building the chunk mesh

`mcvt_world_verts` `0x6b0e50` turns the 145 relative MCVT heights into 145×12B world vertices at
`chunk+0x83c`, and also computes the chunk AABB and bounding-sphere center/radius. The 9×9 outer grid
steps by the cell width and the 8×8 inner grid sits at the half-step.

The X/Y grid coordinates come from a shared step derivation that **`f32`-stores every intermediate** — a
precision detail that matters because the rounding accumulates differently than a pure-`f64`
computation:

```
prod(i)   = f32( f32(i) · 33.33333206 )
origin(i) = f32( −prod + 17066.666015625 )
d         = f32( origin(i) − origin(i+1) )
step      = f32( −d / 8 )
```

The product store is load-bearing: at `i = 512`, `512·C` is exactly the map-center constant, and `513·C`
rounds **up** to `17100.0` under `f32`, so the step there is the same `-4.166748046875` as at `i = 0`. A
pure-`f64` transcription diverges. The general rule throughout terrain geometry: do the arithmetic in
`f64`, then narrow to `f32` at exactly the points the binary does an `fstp dword`; heights enter the
vertices as raw bit copies.

Supporting leaves: `vec3_min` `0x699250`, `vec3_max` `0x6992c0`, `vec3_max_inplace` `0x6b11f0`. The grid
constant `17066.666` appears at `0x7ffab4`; the recurring `×0.24` query scale is at `0x8107a4`.

## Index strips

Four strip-emitters generate the triangle indices, all using a degenerate-triangle-stitched strip:

- **`strip_indices` `0x68b990`** — the chunk mesh: a degenerate-stitched double pass over two running
  counter slots, both mutated; the caller threads them across successive strips.
- **`liquid_strip_indices` `0x68d9b0`** — a restartable strip over the 8×8 MCLQ cell flags
  (`low-nibble == type`). A run emits `(i, i, i+9)` then `(i+1, i+10)` pairs, re-emitting the last index
  on run end; returns the emitted count.
- **`hole_strip_indices` `0x6b61a0`** — the same weave over `rows×cols` cells with vertex stride; a cell
  participates when its low nibble `≠ 0xf` **and** (bit7 clear, or the `DAT_00ce269c` global is set).
- **`wdl_strip_indices` `0x6bd9b0`** — the WDL far mesh: a fixed 16×16 layout of a 4-triangle fan per
  quad around centers `0x121..`, using `i16` arithmetic.

The GX vertex/index buffer lock/unlock entry points (`0x58a080`/`0x58a0a0`/`0x58a030`) belong to the
device; the emitters compute index values only.

## Textures and alpha (MCAL / MCSH decode)

A family of decoders unpack the per-chunk alpha and shadow maps into RGBA texels:
`mcal_decode64`/`32`/`63`/`32_63`, `mcsh_decode`/`decode32`, and the combined packers
`alpha_texel_64`/`32_edge`/`dup`/`32`. The output texel is **RGBA4444**, with the four channels =
`(shadow, layer1, layer2, layer3)`.

Lookup tables:

| Table | Value | Role |
|---|---|---|
| `0x810ba4` / `0x810bac` | `[0xf, 0xf0]` / `[4, 0]` | nibble select (LOW nibble first, amplified into the byte's top 4 bits) |
| `0x810bb4` / `0x810bd4` | (bit tables) | LSB-first bit selection |
| `0x810bf4` | `[0xff, 0x00]` | shadow LUT |
| `0x86b590` | `[0, 0xffff]` | MCSH expand LUT |

Absent layer/shadow inputs read `0xff`. Two routines are 32×32 samplers, not the sizes a naïve read
suggests: `0x6b0510` is the **32×32 edge-sampling combine** (not 65×65), and `0x6b0da0` is the **32×32
MCSH decode** (not a 2-bit layer expand).

## Height and liquid queries

These kernels answer "what is the surface height / liquid state at this point?" against a loaded chunk.

**`chunk_get_height` `0x6b72a0`** — the **x87 path**. It picks one of a 4-triangle fan around the cell
center vertex (`+9`) using the corner LUTs `0x81047c`/`0x810480` (`q0=(17,0) q1=(0,1) q2=(18,17)
q3=(1,18)`), builds the plane from the **normalized** cross product, and `f32`-stores `dx`/`dy` into the
(otherwise dead) argument slots. There is also an `rsqrtss` fast path selected by `DAT_00c9e388 ≠ 0`;
that path is **not reproduced** — a hardware reciprocal-sqrt approximation has no canonical value and
differs per platform. The precise x87 path is the canonical one.

**`liquid_height_sample` `0x6b7500`** — `cell = round_ties_even(c − 0.5) & 7`, with the whole bilinear
interpolation carried on the x87 stack and exactly one final `f32` store. `(flags & 3) == 1`
short-circuits to height `0`.

**`liquid_status` `0x69b6d0`** — the terrain liquid query (the `0x69b520` WMO pre-query is
[Models](models.md)'s). Coordinates are scaled (`×0.24 = 0x3e75c28f`) and `f32`-stored before the
`fistp(−0.5)` cell rounding; the index splits into area/chunk/cell by `>>7` / `>>3 & 0xf` / `& 7`. The
fishable bit `b>>7` is written for **every** inspected non-null slot. A type-1 slot below the `0.01` eps
short-circuits to height `0`. The bilinear is single-store. Acceptance is `z < h + 0.01`; otherwise the
slot loop **continues** to the next slot. Flow cases 1/2 read the wave objects at `slot+0x18` — note that
this is the liquid slot's `+0x18` field, **not** a RadWave object's own `+0x18` (which is its rows
field). The count-1/2 underwater sound-zone flow (`liquid_sound_flow`) handles the inside/outside
branches including the `dist² == radius²` gate boundary.

**`ground_effect_lookup` `0x69b1c0`** — bounds `[0, 34133.332)`, gated by the holes mask. It reads the
2-bit low-res layer index via the `u16` masks `0x8104b4` and byte-stride-4 shifts `0x8104c4`; if the
layer's `effectId ≠ 0xffff` and `≤ the DAT_00c0dc64 count`, it returns the `DAT_00c0dc60` row's `+0x18`
dword. The function returns **AL only — the high bytes are garbage**.

Vector leaves used by the queries: `vec3_cross_world` `0x672130`, `vec3_normalize_world` `0x6720f0` (the
reciprocal **stays in `st0`** — `f64`, never `f32`-stored). Note `0x4549a0` (in cmath) is a plain vec3
**store** of its three stack arguments, not a scale — the call-site `fstp`s round each component to `f32`.

## Liquid type → texture and material selection

The water type at a tile is **the per-tile MCLQ cell-flag low nibble** (`block+0x290`, an 8×8 array,
masked `& 0xf`; `0xf` = no liquid / hole). The nibble (0–8) **is** the type and indexes the texture table
directly — confirmed at the consumers `0x6ba970` (`and al,0xf` / `cmp 0xf`) and `0x68d9b0`. It is **not**
the MCNK header bit and **not** the slot ordinal. The chunk `+0x118 .. +0x124` slot pointers still just
locate one MCLQ block per set header bit (2–5); they do not carry the type.

**Texture name table `0x86a000`** — `char*[12]`, stride 4, indexed `type*4` (read at `0x68ab44`; the `%d`
is the frame, 1..30):

| Type | Texture | Type | Texture |
|---|---|---|---|
| 0 | `XTextures\river\lake_a.%d.blp` | 5 | NULL |
| 1 | `ocean\ocean_h` | 6 | `lava\lava` (dup) |
| 2 | `lava\lava` | 7 | `slime\slime` (dup) |
| 3 | `slime\slime` | 8 | `river\fast_a` |
| 4 | `river\lake_a` (dup) | | |

Class = `nibble & 3` `{0 river, 1 ocean, 2 magma, 3 slime}`; bit2 selects a same-class variant.

**Animation `0x68aac0`** (`ecx` = type → GX handle): 30 frames, period **1250 ms** (speed `1.25 × 1000`);
`frame = round((now_ms % 1250)/1250 · 30 − 0.5) ∈ [0,29]`. The handle cache is `0xc811c0[type*30 +
frame]`; the per-type scroll phase is `0xc81188[type]`.

**ADT render uses three fixed queues**, each hard-coding its texture-cache type (the cell nibble selects
*membership* in a queue, not the call argument):

- `0x6851b0` → type **4** river (`mov ecx,4` @ `0x68db9d`)
- `0x685010` → type **1** ocean (`mov ecx,1` @ `0x68dabb`)
- `0x6855a0` → type **6** magma (`mov ecx,6` @ `0x68dcab`)

Slime (3/7) has **no ADT queue** — it renders only on the WMO liquid path.

**Per-type material:**

- **River** vert-fill `0x68d790` → depth-LUT `0xc81768`, alpha-blended (plus the fancy-water shader
  `ocean0_s.bls` under `0xc9a324`).
- **Ocean** `0x68d690` → depth-LUT `0xc7fcd8`, alpha-blended.
- **Magma** `0x68d890` → constant `0x3f800000` (1.0), **no LUT = opaque, full-bright, no fog/texgen**;
  its setup `0x68dca0` uniquely toggles GX state `0x37`.

The depth→LUT input scale is `0x8029b0 = 0.25`; the LUTs themselves are built in `0x68c4c0`.

The parse-time classifier that writes the cell nibble from MCNK header bits 2–5 is **not confirmed from
the binary** — so the header-bit→class ordering (bit2 river / bit3 ocean / bit4 magma / bit5 slime) is
unproven, even though the rendered nibble→texture selection is.

### The depth → color/alpha LUTs

The vert-fill's `vert.color = lut[depth_byte]` reads three **runtime-built but static** LUTs, built once
by `0x68c4c0` (one-time init via `0x691b10`, not per-frame):

- **`0xc7fcd8` OCEAN** — 256-entry **float** depth→color/brightness: `ocean[i] = min(i/255, 1.0)` (write
  `0x68c57c`; read `0x68d718`).
- **`0xc81768` RIVER** — 256-entry **float** depth→color/brightness: `river[i] = min(i/42, 1.0)`
  (saturates ~6× faster; writes `0x68c598`/`0x68c5a7`; read `0x68d818`).
- **`0xc7fbc0` ALPHA** — 64-entry **packed `<<24`** depth→opacity:
  `alpha[i] = round(clamp(1.6·(i/63)^8, 0, 1)·255) << 24` (the x⁸ chain `0x68c5d1..0x68c5d9`; write
  `0x68c629`). The constants are all `.rdata` (`1/9`, `4.6667`, `3/14`, `1/63`, `1.6`, `255`, `512`).

There is **no dynamic input** to these LUTs: it is a fixed depth-shading curve, with no day/night color
table, no per-zone or per-liquid-type water color, and no time term (the lone non-`.rdata` read
`[0xc81184] = 0.5833` is itself a compile-time constant that cancels). Any time-of-day water tint is
applied **downstream** — standard CPU vertex-lighting plus fog (see [Lighting](lighting.md)), and
`ocean0_s.bls` for ocean — combining this static factor with the lit base color. The LUTs contribute zero
day/night dependence.

## The world-scene render frame

The world frame is driven by `CWorldScene` `0x681070`, which first builds the GX mesh/liquid pools
(`0x6ae640` `CMapChunk_vtx`/`_idx`, `0x68ca50` `CChunkLiquid` vtx/idx) and computes the frame's fog
distances. It is otherwise pure composition over the per-pass routines; its only non-delegated computation is the
fog-distance/near-bias math.

### Drain order

The per-frame draw order is **not** `0x681070`'s alone. The top-level scene driver **`0x483460`**
dispatches three drain phases through `jmp`-thunks (`0x6701b0`/`c0`/`d0` = `jmp 0x681070`/`0x6813d0`/
`0x6816d0`); `0x681070` is only phase 1.

- **Phase 1 — `0x681070`** (thunk `0x6701b0`, `0x48361d`): six back-to-back **unconditional** drains, in
  order — **`0x684510` terrain → `0x684840` WMO → `0x6855a0` liquid-2 → `0x683f80` doodad → `0x683dd0`
  M2 → `0x6841a0` far-band**. **Far is drained LAST.** `0x6855a0` (liquid type-2) is the only liquid
  bucket in phase 1.
- **Phase 2 — `0x6813d0`** (thunk `0x6701c0`, `0x4836ca`): a separate detail-doodad sub-scene pass — its
  own phase, with zero direct callers (reached only via the thunk).
- **Phase 3 — `0x6816d0`** (thunk `0x6701d0`): **conditional**, gated by `0x672470`'s return
  (`0x4836d6: cmp eax,0xf`, two mutually-exclusive sites `0x4836e4`/`0x4836fe`). Drains the secondary
  liquid layers — **`0x685010` liquid-1 → `0x6851b0` liquid-0 → `0x684cd0` WMO-liquid** — then tail-jumps
  `0x68fae0` (a 4th liquid pass). Runs after phases 1 and 2.

### Per-pass setup

Each pass drains its own GX draw-list global and renders camera-relative against the camera-position
globals `0xc7cf20`/`24`/`28`. Phase placement is noted inline.

1. **Fog distances + sky/indoor setup.** `render_fog_dist` (`0x681316` + 5 byte-identical clones) derives
   the near-bias and fog distance from the far clip (`fov·0.5`, `fcos`). The scene fog end
   `0xc62964 = min(outdoor +0x78, indoor/underwater +0x88)` of the DayNight singleton `0xce9b60` (set in
   `0x66fd50`). Then the sky-dome / indoor-portal legs.
2. **Far-band pass — `0x6841a0`** (drains `0xc7cad4`; phase 1, drained last — `0x6812ca`): a special
   projection (far plane `33.0`) and a compressed **depth range `[0.955, 0.96]`** so distant hulls sort to
   the back of the depth buffer; unlinks the sort buckets and draws the cached distant fog hulls
   (`0x6bd780`) under flag `0x4000000`.
3. **Terrain-chunk pass — `0x684510`** (drains `0xc7cae0`): per visible chunk, a camera-relative
   transform (delta vs `0xc7cf20`/`24`/`28`), light gather/apply, draw via `0x6beb50`. It also manages
   detail-doodads — unlink at distance `≥ 70.0` (`[0x867958]`), else build via `0x6bfc10`.
4. **Detail-doodad pass — `0x6813d0`** (drains `0xc7cae0`; phase 2): per shared-tile record, draws the
   per-tile doodad sub-list (gated by record field `+0xc0`), camera-relative transform, light
   gather/apply, draw via `0x6c02d0`.
5. **Chunk-liquid passes** (the `CChunkLiquid` pools): type 2 — `0x6855a0` (drains `0xc7cb34`; phase 1,
   `0x6812bb`): render-state push, per-layer queue drain, `0x68de40(2)`. Type 1 — `0x685010` (drains
   `0xc7cb28`; phase 3): camera-offset, queue drain, `0x68de40(1)`. Type 0 — `0x6851b0` (drains
   `0xc7cb1c`; phase 3): fancy-water texgen one-time init (y-scale `0.225`, anim callbacks), queue drain,
   `0x68de40(0)`.
6. **Map-object (WMO) passes:** WMO — `0x684840` (drains `0xc7caf8`; phase 1, `0x6812b6`): camera
   transform, fog/light setup, group batches via `0x6b3f90`, defers its chunk list to `0xc7cd94`.
   WMO-liquid — `0x684cd0` (drains `0xc7cb04`; phase 3, `0x6816da`): LUT rebuild (`0x6b6b60`) when dirty,
   per-def camera transform + light gather/apply, draws the MLIQ group via `0x6b4060`.
7. **Weather / glare passes** — gated behind `CWorld` flag `0x2000000` (weather): the sun/moon billboard
   and glare staging (`0x68df70`).
8. **Fog color by underwater mask — `0xc7f288`.** Selects the frame fog color — outdoor (`0xce9b60
   +0x70`/`+0x74`/`+0x78`) vs indoor/underwater (`+0x80`/`+0x84`/`+0x88`) — by the underwater mask.
9. **Debug flush** closes the frame.

## Render-list buckets

Each pass bucket is an **intrusive sentinel deque** with a 3-dword header `{+0 stride, +4 tail, +8 head}`;
the "drain global" named per pass is the **head** (`base+8`), so the base is 8 below it. The `+0` "stride"
is the **in-record offset of that record's link node** — far `0x8c0`, terrain/doodad `0xd4`, WMO `0xcc`,
WMO-liquid `0xd4`, liquid `0x20` — and a record's link node is `{+0, +4}` at `record+stride`.

The **empty-list sentinel** (init helper `0x6878e0`, or inline `0x67f152`), all 3 dwords:
`+0 = stride`; `+4 tail = base+4` (untagged self-pointer); `+8 head = (base+4)|1` (the low-bit-tagged
empty/end marker — `test ptr,1` set ⇒ stop).

The pointer convention is **asymmetric**: the forward/drain chain stores **record bases** —
`head(+8)` = first record, `[record+stride+4]` = next record, terminated by `(base+4)|1`; the
backward/insert chain stores **node addresses** — `tail(+4)` = last node, `[record+stride]` = previous
node, terminated at `base+4`.

Read-back (the draw order):

```
cur = [base+8]
while (cur & 1) == 0 && cur != 0:
    emit record at cur       # cur is a record base — no −stride adjustment
    cur = [cur + stride + 4]
```

Records come out in **insertion order (FIFO, oldest first)** — there is no sort key.

### Populate — the scene walk (`0x682fa0`)

The per-region scene walk `0x682fa0` runs inside `0x681070` before its drains (`0x682f60` is a cull-query
wrapper, not the walk). It loops the **32 cells** at `0xc7bd40` (stride `0x6c`); each cell calls the
sub-walks in the fixed order **M2 `0x683340` → doodad `0x683700` → WMO `0x6856c0` → liquid `0x683ab0`
(layers in order 1, 0, 2) → terrain `0x683bf0`**, then the far walk `0x683040` once (a camera-centered
±3-tile window of the 64×64 grid `0xc92078`, row-major).

Per record it culls then tail-appends: skip if already linked (`[rec+nextoff] != 0`); frustum-cull via
`0x682f40` (AABB) or `0x682ef0` (M2/doodad point); occlusion-cull via `0x686180`/`0x686000` (gated by
`0xc7b2a4 & 0x20`); if both pass, FIFO tail-append into the kind's bucket. This deterministic visit order
**is** the draw order: region ascending → 32 cells ascending → fixed per-cell kind order → native
container order → far last.

Per-kind buckets (base / head / stride · drain):

| Kind | walk | base/head/stride | drain |
|---|---|---|---|
| M2 | `0x683340` | `0xc7cb08` / `0xc7cb10` / `0xfc` | `0x683dd0` |
| doodad | `0x683700` | `0xc7cae4` / `0xc7caec` / `0x16c` | `0x683f80` |
| WMO | `0x6856c0` | `0xc7caf8` | `0x684840` |
| liquid | `0x683ab0` | `0xc7cb1c`/`28`/`34` | (per layer) |
| terrain | `0x683bf0` | `0xc7cae0` | `0x684510` |
| far | `0x683040` | `0xc7cad4` | `0x6841a0` |

### The frustum planes

The 6-plane build `0x686640` assembles the context's planes (stride `0x10` `{n, d}`, slots 0..5) from the
8 camera corners (`c0..c7` at `ctx+0x60`, stride `0xc`):

- `plane0 = cross(c5−c1, c6−c1)` anchored at c1 (using the standard cross `0x637480`).
- 4 side planes use the vendored, non-textbook-order cross `0x672130`: side1 `(c7−c0)×(c4−c0)`/c0,
  side2 `(c4−c0)×(c5−c0)`/c0, side3 `(c6−c3)×(c7−c3)`/c3, side4 `(c4−c5)×(c6−c5)`/c5 — each `d = −(n·anchor)`.
- `plane5 = −plane4.n` anchored at c2.

The 8 corners are written into the global `0xc7bcd8` (stride `0xc`) directly by `gx::frustum_corners`
`0x5c43b0` (called from the scene-capture `0x680bc0` @ `0x680e3c` with the staged view `0xc7d088` and proj
`0xc7d280` matrices, out-buffer `0xc7bcd8`). It is an **identity** fill — slot `c_N` ← the unproject of
NDC corner N, **no per-corner `fchs`**. The corners are the 8 NDC-cube vertices `(±1, ±1, z)` with
**z = −1 near (c0–c3), +1 far (c4–c7)**, each quad wound BL→TL→TR→BR: c0 nBL, c1 nTL, c2 nTR, c3 nBR,
c4 fBL, c5 fTL, c6 fTR, c7 fBR. So the plane mapping is: plane0 = top, side1 = bottom, side2 = left,
side3 = right, side4 (plane4) = far, plane5 = near; every cross points inward. (The 7 `fchs` in `0x680bc0`
negate the camera translation into the view-matrix staging, not corner slots; the matrix multiply bakes
them uniformly into every corner.) `0x682490` lerps `0xc7bcd8` 1:1 and `0x6865f0` copies the 8 corners
into `ctx+0x60` unchanged.

### Drain mechanics

The drain (e.g. far `0x6841a0`) walks the head-forward chain, saves `next` before unlinking, splices the
record's geometry into the GX pool (`0x687890`), clears the node, and — gated by `0xc7b2a4 & <pass-bit>` —
calls the pass draw routine. The far-band entry is `{ geometry hull record+0x8c8 · camera-relative
transform (−0xc7cf20/24/28) · projection far = [0xc7b49c] / fwd − 33.0 · depth-range [0.955, 0.96]
([0x80febc] = 0.96, −[0x80fec4] = 0.005) · fog/material state · insertion draw-order }`; `0x6bd780` emits
it, and the GX submit `0x58a830` (device singleton `[0xc0ed38]`) is the device boundary.

The geometry passes share this machinery — only the stride, payload, draw routine, and `0xc7b2a4` feature
bit differ:

| Pass | draw routine | feature bit (`0xc7b2a4`) |
|---|---|---|
| far | `0x6bd780` | `0x4000000` |
| terrain | `0x6beb50` | `0x2` |
| liquid | `0x68de40(layer)` | `0x1000000` |
| WMO | `0x6b3f90` | `0x100` |
| WMO-liquid | `0x6b4060` | `0x100` |
| doodad-LOD | `0x6c02d0` | `0x100000` |

### The M2 / doodad model passes own no draw transform

The two **model** passes — doodad `0x683f80` and M2 `0x683dd0` — render placed model objects, so the
terrain drain applies **no model matrix and no camera-relative draw transform**. They compute scalars and
flags, stage the model object, and register a **deferred render callback**; the matrix is consumed and
the draw issued later inside the model object's own vtable (the [Models](models.md) surface). Per record
(`obj = [rec+0x88]`):

- **Doodad `0x683f80`** (bucket head `0xc7caec`, stride `0x16c`) owns a **2D distance-fade alpha**:
  `d = sqrt((rec.x−cam.x)² + (rec.y−cam.y)²) − rec.radius` (`[rec+0x5c]`/`[0x60]` vs camera
  `0xc7cf20`/`24`, radius `[rec+0x68]` — a scalar for the fade, not a vertex transform), banded
  (`0x810188`/`8c`/`90` = `0.5`/`2.5`/`7.0`; divisors `0x810194..9c`) and clamped `[0,1]`. It does a
  **verbatim** 4×4-matrix blit `[rec+0xcc] → obj+0xbc` (`0x710620`, copied not computed), then `show
  0x710b90(1)`, `visible 0x710c50(1)`, **register `0x7134b0(0x672a20, rec)`**, and `set alpha
  0x710cb0(alpha) → obj+0x180`.
- **M2 `0x683dd0`** (bucket `0xc7cb08` / tail `0xc7cb10`, stride `0xfc`) owns: unlink + show/hide
  (`0x710b90(1)` / `0x710c50((~[rec+0x90]>>3)&1)`) + **register `0x7134b0(0x672a20, rec)`** + forward to
  the record's own `[rec+0xb4]` method (`edx=1`; the second list `0xc7cb4c` uses `edx=2`). No alpha, no
  matrix; it never reads the camera globals.

`0x7134b0` is a **setter, not a draw**: `mov [obj+0x3bc], 0x672a20; mov [obj+0x3c0], rec; ret 8` — it
registers the deferred callback. `0x672a20` (run later) calls the record's own vtable `[rec]→+4` (setup),
fog-gated `+8` (`[rec+0x78] − [rec+0x68] ≥ [0xc62964]`), `+0xc` (draw): the model matrix's use and the
draw live inside the model object.

## Ray-walk, intersection, and collision gather

The terrain ray query walks the grid by a DDA, then intersects per cell.

**DDA major-axis steppers** `0x69c600` / `0x69c780` keep the slope on the x87 stack. `0x69c600`
`f32`-stores a **copy** (`fst`) whose reload drives the per-row multiply, while `b` derives from the
`f64` original; the twin `0x69c780` has no `f32` copy (both `b` and `inv = 1/m` use the `f64`). A major
walk's final push is one cell **past** the end cell (`end+dir`) before the endpoint fixup appends
`(yb, xb)`. The visited-cell list is a pointer at `0xc962c8` plus an int write-count at `0xc89f40`,
storing `(y, x)` pairs. The drivers call their per-cell intersector **once** after the walk (it iterates
the list itself).

**Per-cell intersectors** `0x69c920` / `0x6aa600` resolve the packed cell two-level (`c1` = the major
axis): grid `(c1>>7&0x3f)*0x40 + (c0>>7&0x3f)`, chunk `(c1>>3&0xf)*0x10 + (c0>>3&0xf)`, fan index
`(c1&7)*0x11 + (c0&7)`. A same-chunk cache is keyed `(c0&0x1ff8, c1&0x1ff8)` seeded `0x2000`. The two
functions use **opposite fan windings** (`0x81047c` vs `0x810480` — the sign of the determinant matters
at `eps=0`), and `0x6aa600` stores the **raw** ray-tri `t` (the `inv_len` rescale is the driver
`0x6aa3e0`'s).

The ray-triangle test `0x7c2c40` is Möller–Trumbore with per-dot sum orders `det/u/v = z,y,x` but
`t = y,z,x`.

The collision facet record is `0x34` bytes = 13 `f32` `(n, d, v0, v1, v2)`; the quad far corner is
single-rounded in record 1, double-rounded in record 2. The plane-from-triangle `0x637480`: `v = b−a` has
x/y `f32`-stored but z `f64`, `u = c−a` wholly `f64`; `1/sqrt` stays `f64` (never stored); `d` reuses the
`f64` normalized products.

**The collision gathers.** The 5-vertex outcode is the sign bit of `f32((coord − bound) + eps@0x8101b8)`.
`chunk_gather_aabb` translates the whole box by `−origin` (min via the inline `fadd`, max via the vec3-add
`0x4841c0`), so both outcode faces run chunk-relative. It has a **second output** beside the `0x34`-stride
records: parallel **slide-velocity rows** (`param_5+0x18`, stride 8 = `[f32;2]`, the resolvers'
`outVecB` @ `0xc4e544[rec_idx]`). After the record sweep it sizes the rows to the record count
(`param_5+0x14`) and **zeroes** the newly-added ones — so every terrain slide-velocity row is `[0,0]`.
(The terrain wall-slide reproject is the resolver's slide-response `0x635090` `outVecA`, not this row;
non-zero rows come only from model/doodad collision.) See [Collision](collision.md).

**Ray range threading.** The chunk driver hands its intersector the rebuilt ray `{A, A+(B−A)·t0}` and
rescales the raw first-hit `t` by `t0` — so `t0` is the max **range**: acceptance is `raw_t < t_io` (the
down-ray consumer shape).

## Waves

The wave FFT `0x68b600` (Numerical Recipes `fourn` shape) carries `wr`/`wi` as `f64` across each whole
dimension (only `wpr`/`wpi` are `f32`-stored). The synthesis peaks `0x68b410` are aliases into the
`0xc81d60` grid with a per-peak fresh-`f64`-vs-`f32`-reload split.

`radwave_update` `0x68bee0`, despite looking stochastic, is **shipped-deterministic**: the `tanh`
argument folds to the `0.1` clamp (`0.0·seed`; the seed global `0xc89d8c = 2.75/150` is never advanced
here) — visible in the bytes at `0x68bf37..` (`[0x801620] = 0.1`, `[0x8102f4] = 150.0`).

## The WDL horizon test

Both horizon test functions bias `col_hi` by `+0xa1` (vs `+0xa0` everywhere else — a real binary
off-by-one) and return `0` = visible / `2` = occluded. The view 4×4 at `0xc7b700` and the 320-column
buffer at `0xc7b750` do **not** overlap (`0xc7b748` holds an unrelated pointer).

## The batch protocol

The GX batch staging: headers `0xc62550` (stride `0x20`), matrices `0xc62980` (stride `0x40`) — both
capped at `0x20`; `u16` indices `0xc63298` with cursor `0xc7b378`, cap `0xc000`; overflow sets
`0xc63258 |= 1`. Batch-open reserves the **full worst-case** index span (advancing the cursor) but writes
only `index_count` entries — the tail is reserved-but-uninitialized.
