# Character model — compositing a player from body + equipment

How the 1.12.1 client turns a player's race/sex/appearance bytes plus equipped-item display IDs into a
single rendered character: a `CCharacterComponent` object selects which M2 geosets are visible, bakes
all the skin/face/hair/equipment texture layers into one 256×256 composite, and attaches held-item
sub-models to bone points. This is a distinct subsystem from [Models](models.md) (which owns M2
file-format decode and skeletal animation) — it consumes an already-decoded model handle and makes the
*composition* decisions. Texture upload and blit go out to the [graphics device](graphics-device.md);
the appearance/equipment rows come from [DBC](dbc.md). The whole pipeline is integer/index/pointer
arithmetic — no floating-point.

## The `CCharacterComponent` object

The component object is `0x4e4` bytes. Its fields fall into appearance indices, dirty/state flags, and
the baked handle grids.

**Appearance indices** (decoded from the player's appearance bytes):

| Offset | Field |
|---|---|
| `+0x18` | race |
| `+0x1c` | sex |
| `+0x20` | hairColor |
| `+0x24` | skinColor |
| `+0x2c` | faceType |
| `+0x30` | facialHair |
| `+0x34` | hairStyle |
| `+0x38` | gx M2 render handle |
| `+0x148` / `+0x14c` / `+0x150` | runtime-selected geoset region bases |
| `+0x2f0 .. +0x358` | per-region equip-section texture handles |

**Flags / state:**

| Offset | Meaning |
|---|---|
| `+0x08` | full-rebuild flag |
| `+0x0a` | geoset-dirty (gates `0x477520`) |
| `+0x10` | 10-bit dirty mask — bit `i` gates dispatch handler `[0xb422f4 + i*4]`; `-1` = all dirty |
| `+0x3c` | paperdoll / bake-target |

**Data blocks:**

| Offset | Contents |
|---|---|
| `+0x144 .. +0x180` | 16-entry geoset region-base const table (below) |
| `+0x184` | the 256×256 composite render target |
| `+0x188` | skin-section handle grid (5 region rows × 8 columns) |
| `+0x228` | equipment section-handle grid (10 rows × 16 cols, cell `(slot<<4)+sub`) |
| `+0x248` | parallel ItemDisplayInfo-ptr grid |
| `+0x4a8` | equip-slot → ItemDisplayInfo array, indexed `[+0x4a8 + bodyslot*4]` |

The 16-entry region-base table at `+0x144` is the set of M2 geoset group bases (spacing `0x64`):

```
{ 1, 0x65, 0xc9, 0x12d, 0x191, 0x1f5, 0x259, 0x2be,
  0x321, 0x385, 0x3e9, 0x44d, 0x4b1, 0x515, 0x579, 0x5dd }
```

## Init and the dispatch table

`0x476020` is the one-time init. It binds the model-manager `[0xb42720]`, opens the CharSections DBC
`[0xb4231c]`, builds the CharSections cache grid (below), and installs a runtime dispatch table at BSS
`[0xb422f4 .. 0xb42440]` — ten composite handlers, slots 0..9:

```
0x477170, 0x4771b0, 0x4771f0, 0x4772f0, 0x477360,
0x4773a0, 0x477400, 0x477440, 0x477210, 0x477280
```

## The rebuild pipeline — driver `0x477860(cc, bForce)`

The driver rebuilds a character into the composite target. Its steps:

1. **Fast-out.** On `+0x3c` set / nothing dirty, run only the geoset selector `0x477520` and return.
2. **Load sections.** `0x476eb0` (skin grid) + `0x476f30` (equip grid) walk their grids; if any section
   load is still pending the driver aborts and retries next call.
3. **Lazy-alloc the target.** The 256×256 render target `+0x184` is created on first need (`0x449cc0`,
   bound via `0x710ec0(1, tex)`).
4. **Geoset selection.** `0x477520` chooses the visible geoset set.
5. **Dispatch walk.** For `i = 0..9`, if `+0x10 & (1<<i)`, call `[0xb422f4 + i*4]` (thiscall, `ecx = cc`)
   then blit that handler's region rect `bbox[i]` into the target.
6. **Finalize.** On a full rebuild (`+0x08`), white-clear and release the transient equip handles
   (`0x477ba0`), then clear `+0x08` and `+0x10`.

`bForce` is **not** a blit-forcer — it only bypasses the throttle early-out (`0x4778cb` →
`0x47b200` / `[0xb42764] ≥ 0x100`). The per-handler blit of the base skin is emitted unconditionally;
its sole gate is cell 0 of the skin grid resolving non-null.

`0x477b50` is the race/sex-change reset: it force-loads and invokes **every** handler in the dispatch
table unconditionally. The dirty mask `+0x10` is otherwise set by the 8 equipment layer-choosers (bits
`0x1..0x80`) plus `0x47a610` (`|= 0x18`), and consumed/cleared each driver pass.

## The visible geoset set — `0x477520`

Gated by `[cc+0xa]` (geoset-dirty), this selects which M2 submeshes are shown. Every enable/disable
resolves to `0x7110d0(lo, hi, enable)` in [Models](models.md), which toggles each M2 submesh whose
geoset id lies in `[lo, hi]`. The sequence:

1. Master-disable all geosets: `0x47a880(0, 0x6a4)`.
2. Enable geoset 0.
3. Enable the 16 region bases `+0x144[i]` (the `{1, 0x65, 0xc9, …, 0x5dd}` group bases).
4. Run **8 per-item branches**. Each branch gate is `0x4774f0([cc+0x4bN], sub)` (= ItemDisplayInfo
   section-`sub` present); each add is `enable(base + rec.field)` or `disable(lo, hi)`:

| Branch | Gate | Effect |
|---|---|---|
| B1 | scalp `[+0x4c8],0` | disable `[0x191,0x1f3]` + enable `0x191 + rec[0x18]` (else chest `[+0x4b4],0` → `0x321 + rec[0x18]`) |
| B3 | body `[+0x4b0],0` (only if `[cc+0x26c..0x284]` all 0) | `0x321 + rec[0x18]` |
| B4 | robe-bit guards `[+0x4b4]`/`[+0x4bc]` sub 2 | long path: disable `[0x1f5,0x257]`/`[0x386,0x3e7]`/`[0x44c,0x4af]`/`[0x514,0x577]` + trouser `0x515 + rec[0x20]`; else boot `[+0x4c0],0` → `0x385` + `0x1f5 + rec[0x18]` / legs `[+0x4bc],1` → `0x385 + rec[0x1c]` / `0x385` |
| B5 | glove `[+0x4cc],0` (skip if robe) | `0x4b1 + rec[0x18]` |
| B6 | `[cc+0xc]` helm-skirt | `0x4b1` (+ `0x4b2` if no robe) |
| B7 | `[+0x4b0],1` | `0x3e9 + rec[0x1c]`; `[+0x4bc],0` → `0x44e + rec[0x18]` (skip if robe/glove) |
| B8 | cloak `[+0x4d0],0` | disable `[0x5dc,0x63f]` + enable `0x5dd + rec[0x18]` |

5. Commit `0x47a8e0`, which clears `[cc+0xa]`.

`0x477520` reads only flags `+0xa` and `+0xc`. `0x47a880(lo,hi,0)` disables a range; `0x47a8a0(id,id,1)`
enables one.

## The CharSections grid — build and lookup

CharSections rows (DBC, see below) are aggregated once by `0x476020` from the pre-sorted source array
`[0xc0df64]` (stride `0x28`) into a grid via a **three-pass group-by**. Precondition: rows are sorted
by race / sex / sectionType / variation / color, and `variation` must be dense within each group (else
the build returns 0).

The grid is `(2·maxRace + 2)·5` entries × 8 columns, where `maxRace = [0xc0dee4]`. The entry index is:

```
entryIdx = sectionType + 5·(sex + 2·race)
```

Each grid entry points at a record (stride `0xC`): `{ field0 = colorCount, field4 = baseCount, texidArr @ +8 }`.

- **Pass A** sets `entry.count` = number of distinct variations and allocates `recPtr`.
- **Pass B** sets `record.field0` = total rows (colorCount), and `record.field4` = count of **base**
  rows (those with `flags & 1 == 0`) — the overlay bound.
- **Pass C** sets `idArr[slot] = row.ID` with `slot = (flags & 1) ? field4 + colorIndex : colorIndex`.

So base colors fill `idArr[0, field4)` and overlays fill `idArr[field4, field0)`. `flags & 1` is the
**overlay discriminator, not a skip** — every row is stored. The stored `texid` is the row's **ID**
column, later resolved as an ID-index into `[0xc0df6c]`, not a raw-row ordinal.

**Lookup — `0x477c20(cc, region, secType, variant)`.** Indexes the grid `[0xb4231c]` (record stride 32
bytes) and returns a has-section bool that the texture apply gates on:

```
idx  = sex + 2*race
cell = [0xb4231c] + 8·(5·idx + region)      (region ∈ [0,5))
```

A passing cell yields a `rowId` → `row = [0xc0df6c + rowId*4]` → texture name `[row+0x18 + sub*4]` →
path build `0x58a980` → `TextureCreate 0x449d90`, stored into the `+0x188`/`+0x228` grid.

The cell guard checks `field0` (so cell 0 always fills), but the skin-detail and face-overlay builders
additionally gate on `field4`: an overlay cell fills only when `cc+0x24 < region0.records[0].field4`.
That `field4` (base-row count) is exactly the base/overlay split established by pass B.

## The CharSections section-apply — filling grid cells

The skin grid at `cc+0x188` is `wrapper*[5][8]` (5 region rows, 8 columns, row stride `0x20`, cell stride
4). Region rows are the **section types**: `0 = skin, 1 = face, 2 = facialhair, 3 = hair, 4 = underwear`.
Cell 0 of region 0 is the base skin.

Each cell holds a **texture wrapper**, not a raw `gxImage*`. The blit gate reloads `ecx = [wrapper+0x18]`
(a live indirection) before resolving the source, so the source-mip array is:

```
srcMipArr = *(*(*(cell) + 0x18) + 0x120)
```

i.e. `cell → wrapper`, `wrapper+0x18 → gxImage`, `gxImage+0x120 → src`. The descriptor dims/format live
on the **wrapper** at `+0x1c..+0x30`, filled lazily by `0x47b150 → 0x44b430`.

**Core fill — `0x4782e0(cc; region, sub, recordIdx, colorIdx, bitNo)`.** Fills the cell
`cc+0x188 + (sub + region*8)*4` with a wrapper (texture kind 5 if `region == 0`, else 9), sourced from
`record[0x18 + sub*4]` via the DBC chain `[0xb4231c] → [0xc0df6c]` with `idx = region + 5·(sex + 2·race)`,
then sets `cc+0x10 |= 1 << bitNo` (the dispatch group that consumes the cell).

The functions that fan out to this core:

| Fn | Role | Cells it fills |
|---|---|---|
| `0x478790` | skin | cell 0 (region 0) via bitNo 3, plus face overlays `cc+0x208`/`+0x20c`, plus body-skin bind `0x478160` |
| `0x4785b0` | face / skin-detail | `cc+0x1a8`/`+0x1ac` (region-1 face pair, gated by detailFlag) |
| `0x4783f0` | hairColor recolor | region-2 `cc+0x1c8`/`+0x1cc` (region-3 gated by detailFlag) |
| `0x4784c0` | hair | region-3 texture + the hair geoset `cc+0x144`; tail-calls underwear |

`0x478160` resolves the 3D body skin and binds it to **model texture unit 8** — it fills no composite
cell. The underwear apply-helper `0x478220` builds a direct model texture bound to **model texture
unit 6**; it likewise writes no composite cell.

## The composite target and the CPU blit kernels

The composite target `cc+0x184` is **256×256 16-bit RGB565**, byte pitch `0x200` (256×2). Every
destination address is:

```
dst = dstArr[mip] + Y·0x200 + X·2
```

`[0xb422d8]` (the global destination) is a **mip-pointer array** (`{ &buffer256, … }`), not the buffer
itself — both source and destination args are mip-pointer arrays and the handler indexes `dstArr[mip]` /
`srcArr[mip]`.

**The bbox table** `bbox[g] = 0xb42450 + g*0x10 = { X @+0, Y @+4, W @+8, H @+0xc }` (pixels) is a static
const table written once by the static init `0x475c50` — **not** a per-group fill. All ten rects have
`W = 128`: the 256-wide target is split into a left column (`X = 0`) for groups `{0,1,2,8,9}` and a right
column (`X = 128`) for groups `{3,4,5,6,7}`, each tiling `Y ∈ [0,256)`.

`bbox[g]` is the **destination** position + extent. Source position differs by layer:

- **Base skin** (`0x477070`) pushes `&bbox[idx]` as *both* src and dst (`srcPos == dstPos == bbox`) — the
  base BLP is authored in the full composite layout.
- **Overlay** (`0x4770f0`) sets `srcPos = (0,0)`, `dstPos = bbox[idx].xy`, `size = bbox[idx].wh` — the
  overlay BLP is authored standalone at its own origin.

**Handler selection.** The kernel is picked by the source BLP's format `srcTex+0x30` via the trampoline
`0x477460`:

| `format` | Handler | Operation |
|---|---|---|
| 0 | `0x477c80` | REPLACE (opaque base skin) |
| 1 | `0x477db0` | 1-bit alpha-key |
| ≥2 | `0x477f20` | 4-level alpha-blend |

The 4-level blend uses RGB565 lerp `(src·SW + dst·DW) >> 16` with source weights `SW = {0, 0x5555,
0xaaaa, 0x10000}` (levels 0, ⅓, ⅔, 1).

The handlers are **stdcall, 6 args, `ret 0x18`**, with no internal calls:

```
handler(srcMipArr, dstMipArr=[0xb422d8], &dstPos, &srcPos, &(w,h), descriptor)
```

The 6-dword `descriptor` carries `srcWidth @+0x1c`, `mipCount @+0x28`, `format @+0x30`, and `srcMipArr` —
these are the **wrapper's** `+0x1c..+0x30`. The 2-bit coverage plane is appended in the source mip buffer
**after** the RGB565 plane (at byte offset `h·srcPitch`).

`mipCount` is **computed**, not copied: on the wrapper-fill path (`0x47b150`, entered when
`wrapper+0x1c == 0` and `wrapper+0xc4 == 0`), `0x44ae70(inner+0x144, inner+0x148) = 1 + floor(log2(max(w,h)))`,
clamped `≥ 1`. A bypassed fill leaves `mipCount = 0`, and REPLACE then skips its copy (`0x477cb2 jbe`).
The inner gxImage `[wrapper+0x18]` supplies the descriptor via `0x44b430`: `+0x144` → width, `+0x148` →
height, `+0x11c` → format, `+0x120` → srcMipArr (its `+0x14c` is read but discarded).

## Producing the source buffer — palette → RGB565 + coverage

CharSections textures are **palettised** BLPs, created with gx format 9 (`0x476e93`). The decode chain
`0x47ae30 → 0x449840 → 0x5a8ce0 → 0x5a9a80` ends in the gx format-9 packer `0x5a9a80`, which builds the
source buffer the blit kernels consume:

**(a) RGB565 colour plane** — palette BGRA converted with **round-to-nearest** (not truncation):

```
R5 = (R8 + 4) >> 3        (0x5a9b71)
G6 = (G8 + 2) >> 2        (0x5a9b6b)
```

followed by **Floyd–Steinberg 7/5/3/1 dither** (`0x5a9c04+`) on every section pixel.

**(b) 2-bit coverage plane** — the alpha MSBs at the relevant depth: `alpha8 >> 6` (`0x5a9cd2`) /
`nibble4 >> 2` / `bit1 << 1`, packed with the mask LUT `0x85c9b0 = [0x03, 0x0c, 0x30, 0xc0]` (byte-for-byte
identical to the consumer's mask LUT `0x838ae4`).

**(c) Format selector** — `inner+0x11c` = the BLP **alphaDepth** (`0x4498c4`) → `wrapper+0x30` → the
handler choice (0 → REPLACE, 1 → 1-bit, ≥2 → 4-level). alphaDepth 0 → fmt 5 / REPLACE; alphaDepth 1/4/8
→ fmt 9.

## Customization — face, hair, facial hair

The CharCreate `CycleCustomization` driver `0x470f80` is a 5-way jump table mapping a customization
category to its apply-fn and source cc-field:

| Category | Apply fn | cc field |
|---|---|---|
| 0 skinColor | `0x478790` | `cc+0x24` |
| 1 faceType | `0x4785b0` | `cc+0x2c` |
| 2 hairStyle | `0x4784c0` | `cc+0x34` |
| 3 hairColor | `0x4783f0` | `cc+0x20` |
| 4 facialHair | `0x478660` | `cc+0x30` |

The resulting **head overlay cells**:

| Cell | Region | recordIdx | colorIdx |
|---|---|---|---|
| `cc+0x1a8` / `+0x1ac` | 1 = face | `cc+0x2c` faceType | `cc+0x24` skinColor |
| `cc+0x1c8` / `+0x1cc` | 2 = facialhair | `cc+0x30` facialHair | `cc+0x20` hairColor |
| `cc+0x1ec` / `+0x1f0` | 3 = hair | `cc+0x34` hairStyle | `cc+0x20` hairColor |

Sub 0 maps to bitNo 9 (group g9); sub 1/2 map to bitNo 8 (group g8). The region-2 (facialhair) cells are
filled by `0x478660` (geoset + texture) and recolored by `0x4783f0`; they are null only when the
facialhair section has no row for that race+sex (clean-shaven) — a correct data condition, not an error.

**Geoset region-offset selection — `0x478660`.** Reads source `[eax+0x18/0x1c/0x20]` (the facial-hair
geoset region bases), adds the M2 group base constants `0x64` / `0x12c` / `0xc8`, and stores to
`[cc+0x148]` / `[cc+0x150]` / `[cc+0x14c]`. `0x478660` also fills the region-2 texture, so it owns both
the facialhair geoset and the facialhair section.

The simple hair geoset comes from `0x478540`, which reads `CharHairGeosets` and returns `max(1, geosetId)`.

## Equipment and attach points

**Equip — `0x479010(invType, displayId)`.** Clears the prior occupant (`0x478f20`), maps `invType →
bodyslot` via the **SET LUT `[0x479158]`** (the clear path uses a separate LUT `[0x478c98]`), then
`0x478aa0` dereferences ItemDisplayInfo `[0xc0dc10][displayId]` → `0x478ad0`, which sets `+0x0a`, stores
the record at `[+0x4a8 + bodyslot*4]`, and — for body armor — walks the 8 layer-choosers.

For each layer with a present section (`rec[+0x38 + layer]`) **and** a valid region cell
(`[0x803bf8 + (layer + 10*slot)*4] != -1`), it calls the chooser `[0xb42424 + layer*4]`. The choosers
(`0x4791c0 .. 0x479520`) read their region value, store the section into the `+0x228` grid (`0x478900`,
cell `(layer<<4)+slot`) or clear it (`0x479180`), and OR their dirty bit (`0x1 .. 0x80`) into `+0x10`.

**The `[0x803bf8]` region table** maps `(bodyslot, layer)` to a CharSections texture-region code, `-1` =
unused (10 bodyslots × 10 dwords, stride `0x28`; layers 0..7 are walked). Slots 0/1 are all `-1`;
slots 2..9:

| Slot | layer 0..7 |
|---|---|
| 2 | `0, 0, -, 0, 0, -, -, -` |
| 3 | `1, 1, -, 1, 1, 1, 1, -` |
| 4 | `-, -, -, -, -, 2, -, -` |
| 5 | `-, -, -, -, -, 0, 0, -` |
| 6 | `-, -, -, -, -, -, 2, 0` |
| 7 | `-, 2, -, -, -, -, -, -` |
| 8 | `-, 3, 0, -, -, -, -, -` |
| 9 | `-, -, -, 4, 4, -, -, -` |

**Held items** (bodyslots `0xf`/`0x10`/`0x11`) route to `0x47a0c0` (name table `0x838d50`), which
resolves an M2 attach-point id via `0x47a070` (jump table `[0x47a0a8]`: `0x1b`/`0x1f`/`0x21 − (dl!=0)`,
or `0x1c`), loads the sub-model (`0x4798c0`), and installs the attach transforms (`0x47a380`).
`0x47a230` resets all attach points to identity; `0x47a610` installs the attach transforms (and OR's
`0x18` into `+0x10`). `0x47a4c0` is a race-id table indexer (`[0xb4244c][edx + 2·ecx]`), not an attach
function.

## Section loading

`0x476eb0` (skin grid) and `0x476f30` (equip grid) walk the `+0x188`/`+0x228` grids calling `0x47b150`
per cell. `0x47b150` ensures the underlying model record is loaded (`0x47ac10`, else `0x44b430` +
`0x44ae70`) before its geosets/textures are read, returning 0 (retry) while a load is in flight. That model
record is a **`CACHEENTRY`** — the model manager's refcounted, name-keyed cache record: header `0xc8` bytes
plus a variable tail (`SMemAlloc(size + 0xc8)`), keyed by the model-name string at `+0x38`, with the model id
at `+0x34` (the value `0x47ac10` loads) and a refcount at `+0xbc`. The manager keeps them in a `TSExplicitList`
hash table (get-or-create `0x47ae30`, bucket `= key & [0xb4274c]`, rehash `0x47b7d0`); `0x47b000` adds a
reference and `0x47b010` releases one, destroying the record (`0x47ad70`) when the count reaches zero.
`0x476e20` builds an equipment texture filename with the gender-letter patch: `[0x838ab4 + sex*4]`
written at `[0xb42510 + strlen − 5]`.

## The DBC tables

The drivers read in-memory DBC records whose field offsets are **not** straight `col*4` column copies —
string columns reorder, so offset ≠ column index. The structural schema (field count / record size) is
owned by [DBC](dbc.md).

**ItemDisplayInfo** — reader `0x57b150`, container `[0xc0dc10][displayId]`, stride `0x5c`, 23 columns:

| Offset | Field |
|---|---|
| `+0x00` | id |
| `+0x04 .. +0x14` | 5 model strings |
| `+0x18` / `+0x1c` / `+0x20` | geosetGroup[0..2] (the ints the geoset dispatch reads / `0x4774f0`-gates) |
| `+0x24 .. +0x34` | 5 ints |
| `+0x38 .. +0x54` | 8 texture strings |
| `+0x58` | int |

**CharSections** — reader `0x575540`, stride `0x28`, 10 columns:

| Offset | Field |
|---|---|
| `+0x00` | id |
| `+0x04` | race |
| `+0x08` | sex |
| `+0x0c` | sectionType (0..4) |
| `+0x10` | variationIndex |
| `+0x14` | colorIndex |
| `+0x18` / `+0x1c` / `+0x20` | textureName[0..2] |
| `+0x24` | flags (bit 0 = non-displayable / overlay discriminator) |

**CharHairGeosets.dbc** — `[0xc0df78]`, stride `0x18`, getter `0x575380 → 0x858184`, loader `0x541840`
(6 cols / `0x18`-byte rows, raw copy):

| Offset | Field |
|---|---|
| `+0x00` | id |
| `+0x04` | race |
| `+0x08` | sex |
| `+0x0c` | variation |
| `+0x10` | geosetId (`0x478540` returns `max(1, ·)`) |
| `+0x14` | (uncited) |

**CharacterFacialHairStyles.dbc** — `[0xc0df28]`, stride `0x28`, getter `0x575a50 → 0x8582a8`, loader
`0x541ef0` (9 cols / `0x24`-byte rows + a synthesized row-ordinal at `+0x24` → `0x28` in memory):

| Offset | Field |
|---|---|
| `+0x00` | race |
| `+0x04` | sex |
| `+0x08` | variation |
| `+0x0c` / `+0x10` / `+0x14` | (uncited) |
| `+0x18` / `+0x1c` / `+0x20` | geoset region bases (`0x478660` adds the M2 group bases `+0x64`/`+0x12c`/`+0xc8` → `cc+0x148`/`+0x150`/`+0x14c`) |

Both customization tables are field-by-field built copies (no string columns), loaded by the master
loader `0x540300` (`mov ecx, <desc>; call <loader>`). The 20-byte descriptor is
`base / count / id-index / maxId / loaded-flag`.

## Component copy and clone

- `0x476b90` builds a fresh component from a packed 91-dword index descriptor (`rep movsd 0x5b` into
  `+0x18`), zeroes the handle grids, and re-derives appearance via `0x478790` / `0x4784c0` / `0x478660`.
- `0x476cb0` clones an already-built component: same index copy, but AddRef-copies the `+0x188` /
  `+0x228` / `+0x248` handle grids and sets `+0x10 = -1` to force a full re-bake.

## What this subsystem owns vs delegates

It owns the **composition** decisions: geoset/region selection, the CharSections layer/region
arithmetic, the attach-slot and race-index logic, and the manager pool / hash containers (the `CACHEENTRY` records). It delegates
texture upload/blit and scene nodes to the [graphics device](graphics-device.md) (`0x449d90`,
`0x58a980`, the `0x71xxxx` cluster); model-name/path and object allocation to the CGModel object API
(`0x64a7f0` / `0x6462e0` / `0x646430`); and CharSections rows to [DBC](dbc.md). All of its own logic is
integer / index / bitfield / pointer arithmetic — there is no floating-point math in the composition
itself.
