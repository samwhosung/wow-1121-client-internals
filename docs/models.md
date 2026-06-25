# Models — M2 and WMO geometry and materials

The client renders two kinds of art asset besides the [terrain](terrain.md): **M2** models (every
animated thing — creatures, players, doodads, and the small placed scenery) and **WMO** map objects
(the large static geometry — buildings, caves, bridges, whole interiors). This chapter covers how both
formats are parsed, how their geometry and materials are laid out, how an M2 file is assembled into a
runtime model, and the matrix math that places a doodad or WMO into the world. The skeletal-animation
sampling that drives M2 bones is covered separately in [Animation](animation.md).

There are three model families, with three different object models:

- **M2** classes are all **non-polymorphic** (no vtables) — a cache of shared file images plus per-scene
  instances.
- **WMO** classes are also **non-polymorphic**, registered in a fixed entity pool.
- **Particle and ribbon emitters** *are* **polymorphic** (the only models that are), with a small class
  hierarchy and vtable dispatch.

A note on identifying these in your own binary: 1.12.1 ships with **RTTI disabled**, so the strings
`.?AVCMapObjDef@@` (`0x86a7f0`) and `.?AVCMapDoodadDef@@` (`0x86a7d4`) are **allocation-tag strings, not
vftables** — do not mistake them for class type-info.

## M2 — the runtime classes

M2 loading splits a shared, reference-counted **file image** from the per-instance **model** that
animates and draws it.

- **M2Cache** — the static singleton at `0xceefb0`. Holds a 1021-bucket hash of `CM2Shared` keyed by
  filename (head at `+0x14`), an LRU list, the async load pump, and the vertex/pixel-shader constant
  pools.
- **M2Scene** — heap objects, `0x1d6c` bytes, allocated by the factory `0x706e10`. There are seven
  creation sites; the principal two are the world-init scene (`0x66f720` → stored at `0xc7b298`) and a
  lazily-created scene (`0x6d4640` → `0xce9d54`). Each scene back-points to the cache at `+0x4` and owns
  its instance list at `+0x130`, a callback array, and the draw buffers.
- **CM2Shared** — `0x164` bytes per file. Holds the tagged instance-list head at `+0x8`, the MD20 file
  image at `+0x130` (fixed up in place by `0x71cdf0`), the skin/LOD profile at `+0x138`, and the GPU
  buffers.
- **CM2Model** — `0x428` bytes per instance. Points to its own `CM2Shared` at `+0x2c`, the skeleton's
  shared image at `+0x30`, its parent at `+0x34`, its world matrix at `+0xbc`, and the runtime arrays
  (sized from the MD20 element counts, built by `0x70ebd0`).

**Load path.** Cache lookup `0x706a50` → async begin/complete `0x71d4e0` / `0x71d640` → MD20 header
fixup `0x71cdf0` → profile (skin/LOD) select `0x71d6c0` → per-instance build `0x70ebd0`. The GPU
index/vertex uploads are deferred to the first draw (`0x71dab0` / `0x71dc80`).

**Per-frame draw.** The draw entry is `0x707680`: it animates each node, builds batch records, and emits
leaves (`0x70d330` / `0x719b20`, where the vertex-transform runs). The per-bone quad-strip emit is
`0x719370`. The scene ray-intersect queries used for picking and collision are the pair `0x7089c0` /
`0x7099e0`.

## M2 — file format and accept/reject

The header validator is `0x71cdf0` (called from `0x71d656`). A file is **rejected** unless:

- magic `[+0x00] == 'MD20'` (`0x3032444d`),
- version `[+0x04] ∈ {256, 257}`,
- the `0x154`-byte header fits the buffer, and
- for **every** array descriptor `(count, offset, elem)`: `offset ≤ size` **and**
  `offset + count*elem ≤ size`.

When an array passes, its on-disk file offset is relocated to an absolute pointer in place (or nulled
when `count == 0`).

Six descriptors are checked inline in `0x71cdf0` with explicit element sizes:

| Field | count @ | offset @ | elem size |
|---|---|---|---|
| name | 0x08 | 0x0c | 1 |
| globalSequences | 0x14 | 0x18 | 4 |
| animations | 0x1c | 0x20 | 68 |
| animationLookup | 0x24 | 0x28 | 2 |
| playableAnimationLookup | 0x2c | 0x30 | 4 |
| bones | 0x34 | 0x38 | 108 |

The remaining header arrays are validated by sub-helpers (`0x71e0c0` / `0x71e110` / `0x71ef40` /
`0x71f210`, and `0x71e160` for the **v257-only** array at `[+0x144]`). Each sub-helper's first gate is
`offset ≤ size`.

The shipped 1.12.1 `.m2` corpus is 10196 files: 10170 load (10165 v256, 5 v257), 26 are 0-byte stubs
(unused/beta content, never loaded), and none are rejected — i.e. the predicate matches the entire
shipped data set with no silent fallback path.

## M2 — the header array map

The MD20 top of file is 32 `M2Array` descriptors, each `{count @ base, offset @ base+4}`:

| base | array | base | array |
|---|---|---|---|
| 0x08 | name | 0x9c | transLookup |
| 0x14 | globalSequences | 0xa4 | texAnimLookup |
| 0x1c | animations | 0xac | (unnamed) |
| 0x24 | animationLookup | 0xec | boundingTriangles |
| 0x2c | playableAnimationLookup | 0xf4 | boundingVertices |
| 0x34 | bones | 0xfc | boundingNormals |
| 0x3c | keyBoneLookup | 0x104 | attachments |
| 0x44 | vertices | 0x10c | attachLookup |
| 0x4c | views | 0x114 | events |
| 0x54 | colors | 0x11c | lights |
| 0x5c | textures | 0x124 | cameras |
| 0x64 | transparency | 0x12c | cameraLookup |
| 0x6c | textureAnims | 0x134 | ribbons |
| 0x74 | texReplace | 0x13c | particles |
| 0x7c | renderFlags | 0x144 | (v257-only) |
| 0x84 | boneLookup | | |
| 0x8c | texLookup | | |
| 0x94 | texUnitLookup | | |

## M2 — geometry decode (vertex / view-skin / triangle)

**Vertices** are the `M2Array {count @ 0x44, offset @ 0x48}`, **stride 0x30** (48 bytes per vertex). The
per-vertex block:

| Offset | Field |
|---|---|
| 0x00 | position (C3Vector) |
| 0x0c | bone weights |
| 0x10 | bone indices |
| 0x14 | normal |
| 0x20 | texcoord |

**Views / skins** are the `M2Array {count @ 0x4c, offset @ 0x50}`. Vanilla 1.12 **inlines** the skin in
the MD20 (there is no separate `.skin` file), so each element is a **ModelView record of stride 0x2c**.
Each ModelView holds five M2Arrays plus a trailing u32:

| Offset (count, offset) | Array | element stride |
|---|---|---|
| +0x00, +0x04 | vertexIndex | 2 |
| +0x08, +0x0c | triangle | 2 |
| +0x10, +0x14 | properties | 4 |
| +0x18, +0x1c | submesh / M2Batch | 0x20 |
| +0x20, +0x24 | texUnit | 0x18 |
| +0x28 | (trailing u32) | — |

**Draw-time index indirection** is two levels:

```
vertex = verts[ vertexIndex[ triangle[i] ] ]
```

The triangle u16s (view `+0x0c`) index the view's vertexIndex list (view `+0x04`), which indexes the
global vertex array (header `+0x48`). This is confirmed at the GPU index upload `0x71dab0`
(`mov eax,[eax+0xc]`) and the vertex upload `0x71dc80` (which multiplies the resolved index by `0x30`
and adds `[md20+0x48]`).

### Submesh (M2Batch) and texture units

The **M2Batch** record (the submesh; stride `0x20`):

| Offset | Field | Notes |
|---|---|---|
| +0x00 | geosetId | geoset SELECTION key |
| +0x04 | vertexStart | |
| +0x06 | vertexCount | |
| +0x08 | triangleStart | |
| +0x0a | indexCount | # of u16 **indices**, not triangles — #tris = `/3` |
| +0x0c | boneCount | |
| +0x0e | boneComboIndex | |
| +0x10..+0x1f | (unread) | |

`indexCount` is a count of index entries, not triangles: the upload loop `0x71dbab`–`0x71dbe2` copies
exactly `[+0xa]` u16s. The selector `0x7110d0` range-compares `lo ≤ geosetId ≤ hi` to set the
per-submesh enable array at `[ctx+0x98]` — this is how geosets (hair styles, armor pieces, etc.) are
turned on and off.

The **texUnit** record (stride `0x18`):

| Offset | Field | Notes |
|---|---|---|
| +0x00 | flags | |
| +0x02 | shaderId | |
| +0x04 | submeshIndex | pairs texUnit → submesh |
| +0x08 | colorIndex | |
| +0x0a | materialIndex | → renderFlags `[model+0x88]` |
| +0x0e | textureCount | |
| +0x10 | textureComboIndex | |
| +0x12 | texCoordSet | |

At draw time the submesh is read via the render-command field `cmd+0x30`, while `cmd+0x2c` is the
texUnit (`texUnit+0xa` is its materialIndex) — the two are easy to swap if you misread the command
layout.

## M2 — animation-data layouts

The header fixup `0x71cdf0` also recurses each bone record's M2Tracks and relativizes their arrays.
These are the version-256 layouts.

**Bone record** — `bones` M2Array `{count @ 0x34, offset @ 0x38}`, **stride 0x6c**, count = `header[0xd]`.
The fixup loop (`base += 0x6c`) relativizes 9 M2Arrays = three M2Tracks × three arrays each, at
bone-relative `+0x10/+0x18/+0x20`, `+0x2c/+0x34/+0x3c`, `+0x48/+0x50/+0x58`. The three tracks and the
pivot sit at:

| Offset | Field |
|---|---|
| +0x00 | keyBoneId (i32) |
| +0x04 | flags (u32) |
| +0x08 | parentBone (i16) |
| +0x0a | submeshId (u16) |
| +0x0c | translation track (M2Track, 0x1c) |
| +0x28 | rotation track (M2Track, 0x1c) |
| +0x44 | scale track (M2Track, 0x1c) |
| +0x60 | pivot (C3Vector) |

**M2Track** (track-version 0x100/0x101, **stride 0x1c**):

| Offset (count, offset) | Field | element |
|---|---|---|
| +0x00 | interpType (u16) | |
| +0x02 | globalSeq (i16) | |
| +0x04, +0x08 | ranges | u32 pair (relativizer `0x71e0c0`) |
| +0x0c, +0x10 | timestamps | u32, stride 4 (relativizer `0x71e110`) |
| +0x14, +0x18 | values | see below |

The track is the **version-256 ranges-based** form (interpolation ranges precede timestamps and values).
The `values` element type depends on which track — a typed relativizer fixes it:

- translation, scale: `0x71e1b0` → **C3Vector**, stride `0xc` (12 bytes/key)
- rotation: `0x71f650` → **`[f32;4]` quaternion**, stride `0x10` (16 bytes/key)

So in **v256 (1.12.1) the rotation track is an uncompressed float quaternion** (`[f32;4]`, 16 bytes per
key) — **not** the int16×4 "Quat16" of later model versions. The runtime quaternion sampler `0x713ea0`
reads the 16-byte/key array directly and lerps (`(b−a)·frac + a`); there is no int16→float quaternion
decompression in this build, so a reader marshals the file values straight through (decode stride 16,
not 8).

**Sequence record** — `animations` M2Array `{count @ 0x1c, offset @ 0x20}`, **stride 0x44**, count =
`header[7]`. The fixup relativizes the array offset but does **not** recurse it: v256 sequences are pure
scalars (no M2Arrays inside).

| Offset | Field |
|---|---|
| 0x00 | id (u16) |
| 0x02 | variationIndex (u16) |
| 0x04 | startTimestamp (u32) |
| 0x08 | endTimestamp (u32) |
| 0x0c | moveSpeed (f32) |
| 0x10 | flags (u32) |
| 0x14 | frequency (i16) |
| 0x18 | replay.min (u32) |
| 0x1c | replay.max (u32) |
| 0x20 | blendTime (u32) |
| 0x24 | bounds.min (C3Vector) |
| 0x30 | bounds.max (C3Vector) |
| 0x3c | bounds.radius (f32) |
| 0x40 | variationNext (i16) |
| 0x42 | aliasNext (u16) |

Note v256 uses **start/end timestamps**, not the duration form of later versions.

## M2 — model assembly (MD20 → runtime)

After the header fixup, `0x70ebd0(Model* this)` builds the runtime arrays from the MD20 element counts.
The MD20 image is at `*(this+0x30 [CM2Shared]) + 0x130`, header size `+0x138`. For each element array it
allocates a buffer sized `count × stride` (with a 4-byte count header, via `0x6462e0`), zero- or
ctor-initialises it, and stores the pointer and count into the `Model`.

The buffer-sizing map (MD20 header count offset → runtime stride → `Model` field):

| MD20 count @ | element | runtime stride | → Model buf @ | count @ |
|---|---|---|---|---|
| 0x14 | globalSequences | 4 (u32) | 0x64 | — |
| 0x1c | sequences | 4 (u32) | 0x6c | — |
| 0x34 | bones | 0x118 (+4 hdr) | 0x90 | 0x94 |
| 0x50→view+0x18 | view geosets | 4 (init 1) | 0x9c | — |
| 0x54 | colors | 0x50 (+4) | 0xa0 | — |
| 0x5c | textures | 4 | 0xa4 | 0xa8 |
| 0x6c | textureAnims | — | — | 0xac |
| 0x74 | textureTransforms | 0x98 (+4) | 0xb0 | 0xb4 |
| 0x104 | attachments | — | — | 0x1c8 |
| 0x114 | events | — | — | 0x1f0 |
| 0x11c | lights | 0x170 (+4) | 0x200 | — |

The **bone block** (runtime stride `0x118`, ctor `0x71b0a0`) copies the MD20 bone flags (`bone+0x04`)
into block `+0xf4`. The **light block** (`0x170`) initialises `+0x3c = 1.0f`, `+0x8c = 1.0f`,
`+0xec = 1`.

**Finalize:**

- visibility flag `Model+0xb8 ← 1` iff `!(this+4 & 2)` **and** skin profile > 1 **and** (no parent or
  parent visible);
- bone-hierarchy + identity-transform init (`0x711bf0` / `0x7121a0`, with `1.0f` scale);
- parent propagate (`0x7103a0`, then `Model+0x34 := 0`);
- finally `Model+0x10 := 1` (loaded), `Model+0x14 := 0`.

## M2 — element definitions

The non-geometry element arrays, with their on-disk layouts (file strides and scalar fields recovered
from `0x70ebd0`'s copy loops; the inner M2Tracks are the `0x1c`-byte M2Track above):

- **color** (header `0x54`, runtime `0x50`) — 2 M2Tracks: `color` (C3Vector) + `alpha` (fix16).
- **textureTransform** (header `0x74`, runtime `0x98`) — 3 M2Tracks: translation (vec3) / rotation
  (quat) / scaling (vec3).
- **light** (header `0x11c`, **file stride 0xd4**) — `type(u16)@0x00 · bone(i16)@0x02 ·
  position(C3Vector)@0x04`, then 7 M2Tracks at `0x10/0x2c/0x48/0x64/0x80/0x9c/0xb8` (ambient
  color/intensity · diffuse color/intensity · attenuation start/end · visibility).
- **camera** (header `0x124`, **file stride 0x7c**) — `type(i32)@0x00 · fov(f32)@0x04 ·
  farClip(f32)@0x08 · nearClip(f32)@0x0c`, then position/target/roll M2Tracks plus base C3Vectors.
- **texture** (header `0x5c`, file `0x10`) — `{type:u32, flags:u32, filename:M2Array}`; load resolves
  the name to a texture handle (`0x41af50`).
- **attachment** (header `0x104`) — the v256 M2Attachment `{id, bone, position, animateTrack}`.
- **ribbon** (header `0x134`, **file stride 0xdc**) → runtime `CRibbonEmitter` `0x168`
  (`Model+0x3c8` block array `0xd0` + pointer array `+0x3cc`); load reads its
  textureIndices/materialIndices M2Arrays at file `+0x14`/`+0x18`/`+0x20` and color/edge params at
  `+0x94..+0xa2`.
- **particle** (header `0x13c`, **file stride 0x1f8**) → runtime `CParticleEmitter` (type-variant alloc
  `0x2a0`/`0x2a4`/`0x2d0`; `Model+0x3d0` block `0x16c` + pointer array `+0x3d4`).

The ribbon and particle records are the most version-divergent in MD20; their inner field offsets are
derivable from the same copy loops, and they drive render-only (cosmetic) effects rather than the
skeletal hierarchy.

## Particle and ribbon emitters

Unlike M2 and WMO, the emitters are **polymorphic**:

- **CParticleEmitter2** — base class, vtable `0x81d83c` (slots: Preallocate / spawn-finalize /
  type-spawn / kill). Subclasses by spawn shape: **Plane** `0x2a0`, **Sphere** `0x2a4`, **Spline**
  `0x2d0` (multiple-inheritance; second vtable `0x81d90c` at `+0x2a8`). Created by the factory legs
  `0x7ae3f0` / `0x7ae520` / `0x7ae650` into the GfxSingletonManager registry (`DAT_00cf575c`).
- **CRibbonEmitter** — `0x168` bytes; simulation records `0x14` at `+0x8`, render records `0x28` at
  `+0x40`.
- **CLightning** — `0x74` bytes.
- A **legacy CParticleEmitter** subsystem also exists (`0x7b9f80`–`0x7ba910`, `CParticle` records of
  `0x28` bytes).

**Update:** `0x7b5230` → `0x7b5880` (ages each particle; kill bound = `emitter+0xac` lifespan) →
`0x7b5a10` → spawn `0x7b5550` → vtable type-spawn.

**Render:** `0x7b3d20` → depth-sort `0x7b3a10` → quad emit `0x7b2a50` (which has both a plain and a
tiled-and-rotated expansion mode).

The M2 wiring is the constructor `0x7b14a0`, which clones the M2 emitter definition; the flag word is at
`+0x1ac`.

## WMO — the runtime classes

The WENTITY pool registration (`0x691dfc..0x691ff4`) fixes the class set and object sizes. All are
non-polymorphic:

| Class | size | vftable | ctor | dtor | notes |
|---|---|---|---|---|---|
| CMapObj | 0x7f4 | (created by record class, vtable 0x810998) | | | the map object |
| CMapDoodadDef | 0x198 | 0x810a74 | 0x6a7d00 | 0x6a7f30 | WENTITY-derived; holds its M2 instance |
| CMapObjDef | 0x168 | 0x810928 | 0x6a2200 | | |
| CMapObjDefGroup | 0xdc | 0x810918 | 0x6a1ca0 | 0x6a1e00 | |
| WENTITY (base) | | 0x810824 | 0x69d590 | 0x69f120 | |
| common base | | 0x8110a4 | 0x6c3020 | | |

The parsed-MOGP group struct **is** the WMAPOBJGROUP pool object: `0x170` bytes, link at `+0x168`, pool
`0xca7e0c`, allocator `0x6a03a0`.

**Load path:** `0x6c3e20` (loader) → `0x6c3a60` (root chunk table) → `0x6c3f80` / `0x6c40c0` /
`0x6c4530` per group.

**Query / render:** `0x6b3f90` render driver; `0x6b3dd0` / `0x6b3b20` portal visibility (outside /
inside); `0x6b41c0` portal flood; `0x6a37b0` segment-intersect wrapper. Doodad-def enumerations are
`0x6abe60` / `0x6abc40`; the WMO-list query over `DAT_00ca7da4` is `0x69b4a0`.

## WMO — file format

WMO files are IFF chunk sequences `[tag:u32 LE][size:u32][data]`, with the 4CC stored **reversed**
(e.g. MVER reads as `0x4d564552`). A root `.wmo` carries the materials, group index, and portals; each
group's geometry lives in a separate `_NNN.wmo` group file.

### Root file

The root parser `0x6c3a60` (under loader `0x6c3e20`) walks chunks in **fixed order** using the
`+8`-advance helper `0x6c3a00`. The cursor starts at `file+0xc` (past MVER's 8-byte header + 4-byte
version), and each chunk is skipped by `cursor += size`. The chunks store their pointers/counts into
`CMapObj`:

| Chunk | stored @ | size/stride |
|---|---|---|
| MOHD | +0x120 | header |
| MOTX | +0x124 | texture-name block (size @ +0x160) |
| MOMT | +0x1d8 | 64 B/material (`shr 6`; count @ +0x1dc) |
| MOGN | +0x128 | group-name block |
| MOGI | +0x130 | 32 B/group (`shr 5`; count @ +0x168) |

The parser reads chunks while the 8-byte header is in-bounds and clamps the last chunk to EOF (it never
requires exact tiling — some files run a byte past EOF and are tolerated). The shipped corpus is 6332
`.wmo` files: 820 roots + 5373 groups load, 139 are 0-byte stubs, none are rejected. MVER is v17
throughout.

The **MOMT records pointer (`CMapObj+0x1d8`) points directly into the loaded file buffer** (count =
`MOMT.size >> 6`), so the on-disk material record layout *is* the runtime layout.

### MOMT material record (64 B)

Material realization is `0x6c3db0`:

| Offset | Field |
|---|---|
| +0x0c | texture1 name offset (into MOTX block at `[edi+0x124]`) |
| +0x18 | texture2 name offset |
| +0x38 | resolved texture1 handle (written over the on-disk bytes) |
| +0x3c | resolved texture2 handle |

There is **no texture3** in 1.12: `0x6c3db0` resolves exactly the two offsets (`+0x0c`, `+0x18`), and
`0x6c3d70` only zero-inits the `+0x38` / `+0x3c` handle slots. The blend-mode/flags/colour fields of the
record are read by the render material-bind (which takes a record pointer), not by the parser. In the
corpus, every root's material texture offsets point at a valid string start in MOTX (820 roots, 10435
materials, 9613 non-empty texture refs).

### Group file (MOGP) and geometry sub-chunks

A group `_NNN.wmo` is `MVER + MOGP{ [MOGP header][MOPY MOVI MOVT MONR MOTV MOBA …] }`. In `0x6c3f80` the
cursor (`eax = [group+0x150]`, the file buffer) advances `+0x14` to MOGP.data start (MVER `0xc` + MOGP
tag/size `8`), reads the MOGP header across `[eax+0x00..+0x38]`, then does `add eax,0x44` to reach the
first sub-chunk. So the **MOGP header is 0x44 bytes** and the geometry sub-chunks begin at
**MOGP.data + 0x44**.

The mandatory-sub-chunk fixup `0x6c40c0` walks them **sequentially by size** (each step: `+8` header,
then `+= size`), deriving each count as `size / stride`:

| Chunk | stride | count stored @ | content |
|---|---|---|---|
| MOPY | 2 | +0x124 | material/flag per triangle |
| MOVI | 2 | +0x128 | u16 vertex index |
| MOVT | 0xc | +0x130 | 3×f32 vertex |
| MONR | 0xc | +0x134 | 3×f32 normal |
| MOTV | 8 | +0x138 | 2×f32 uv |
| MOBA | 0x18 | +0x13c | render batch |

## Placement — the doodad/WMO world matrix

`mddf_create_xform 0x694bc0` builds each placed doodad's **column-major 4×4** model matrix (stored at
`inst+0x24`) from the MDDF Euler triple, position, and scale. Draw applies `v' = M·v`:

```
M = T(worldpos) · ( s · Rz(aZ) · Ry(aY) · Rx(aX) )      // column-major; the vertex sees Rx first, then Ry, then Rz
worldpos = (−mddf.x, −mddf.y, +mddf.z) + tile_origin
aZ = mddf[+0x18]·(π/180) + π     // world +Z
aY = mddf[+0x14]·(π/180)         // +Y
aX = mddf[+0x1c]·(π/180)         // +X
s  = u16[mddf+0x20] / 1024
```

The composition convention is the load-bearing part:

- `0x7bdb00` is a standard **right-handed active Rodrigues rotation** `R(axis, +θ)` (column-major, not
  transposed, the angle never negated).
- `0x7bdd60` **post-multiplies** `M ← M·R` (via `0x7bc6a0`, column-major), so issuing the three rotates
  in Z, Y, X call order yields net 3×3 = `Rz·Ry·Rx`.
- Scale (`0x7bdd00`, `mat34_scale3`) is uniform and applied after.
- The matrix is initialised to identity (`0x694d55`).

The MDDF rotation field → axis mapping: `+0x14 → Y`, `+0x18 → Z` (with the `+π` offset), `+0x1c → X`.
The `+π` applies to the Z-axis angle only.

For WMOs, `wmo_world_transform 0x695650` builds the analogous matrix from the MODF placement using the
same primitives (`0x7bdc40` / `0x7bdd60` / `0x7bdd00`) and the same convention.

## Cross-references and dead ends

- The segment-vs-triangle intersection used by both the WMO BSP leaf predicate and the M2 pick path is a
  Möller-Trumbore test at `0x7c29f0` (ε = `0.002`); it lives in the shared math TU. See
  [Collision](collision.md).
- The cached MOHD ambient colour at `CMapObj+0x198` is written but has no reader in this subsystem — WMO
  interior ambient is sourced elsewhere. See [Lighting](lighting.md).
