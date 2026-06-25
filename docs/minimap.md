# Minimap — the circular HUD minimap

The minimap engine (`Source\Game\GameClient\Minimap.cpp`) is the low-level core behind the circular HUD
minimap: the world↔ADT-tile coordinate math, the streaming of minimap tile textures
(`Textures\Minimap\…\map%d_%d.blp`, indexed through `md5translate.trs`), and the blip placement, distance,
zoom and edge-clamp logic. It computes exactly where the map and every blip appear; the FrameXML
`MinimapFrame` / Lua minimap API and the `CWorldFrame` render path are consumers above it. It is distinct
from the full-screen WorldMap, which is a separate UI translation unit.

The translation unit spans 52 functions at **`[0x6d8e10, 0x6dbce8)`**. The per-frame entry point is
the driver **`0x6d93a0`**, called from the `CWorldFrame` minimap render dispatch at `0x4ecc14`.

## Asset paths

The translation unit's string pool gives the file layout the streamer uses:

- `Textures\Minimap` — the tile-texture root.
- `%s\map%d_%d.blp` — a single minimap tile, formatted as `<continent>\map<col>_<row>.blp`.
- `%s\md5translate.trs` — the translation file mapping logical tile names to hashed on-disk directory names.
- `World\` and the continent name form the `<continent>` prefix (`NOCONTINENTNAME|%d` is the failure tag).
- `MINIMAPCHUNKNOTFOUND|%d|%d` — warning emitted when a requested tile is absent.

## The coordinate grid (tile ↔ world)

The minimap works in the standard 64×64 ADT tile grid (see [Terrain](terrain.md)). A tile is `533.333`
yards on a side, and the map origin is `17066.666 = 32 · 533.333`.

**`tile_to_world` (`0x6da5b0`)** — given an integer `tile` (`[0]=row`, `[1]=col`):

```
out.x = 17066.666 − (float)tile[1] · 533.333
out.y = mapOriginY   − (float)tile[0] · 533.333
out.z = 0
```

`mapOriginY` is the runtime per-map Y origin at `0xceaa88`; the X origin is the constant `17066.666`. The
computation is done with the x87 sequence `fild; fmul[0x80654c]; fsubr`.

**`world_to_tile` (`0x6da540`)** — the inverse, per axis:

```
t = (17066.666 − world) · 0.001875        // 0.001875 = 1/533.333
if (t < 0)  t -= 1.0                       // floor-toward-−∞ correction for the truncating __ftol
tile = __ftol(t)                           // truncate to int  →  floor((17066.666 − world)/533.333)
```

The explicit `−1.0` when `t < 0` compensates for `__ftol` (`0x40a2b0`) truncating toward zero rather than
flooring, so the result is a true floor.

## World → minimap-screen scale

**`world_to_screen_scale` (`0x6da380`)** first ensures the current tile window is built (calling
`0x6da540` / `0x6da5e0` / `0x6da5b0`), then emits the blip→screen scale. With `1066.666 = 2 · 533.333`:

```
scaleDenom = 1.0 / 1066.666                                  // 1066.666 at 0x86f6b8
*param4    = (float)(chunkTable[zoom] >> 1) · 33.333 · scaleDenom   // 33.333 = sub-chunk yd (0x80fee4)
*param3    = (tileWorld.y − player.y) · scaleDenom
param3[1]  = (tileWorld.x − player.x) / 1066.666             // 1066.666 at 0x86f6bc
```

`chunkTable` is the per-zoom chunk-count table (below). The three scale terms are what the function returns;
before computing them it rebuilds the current tile window (via the `0x6da540` / `0x6da5e0` / `0x6da5b0` calls
noted above) so the scale is taken against the up-to-date window.

## Constant pool

All values are `.rdata` constants; the addresses are the in-binary locations of these constants.

| Address | Value | Meaning |
|---|---|---|
| `0x80654c` | `533.333` | ADT tile size (yards) |
| `0x811734` | `0.001875` | `1 / 533.333` |
| `0x7ffab4` | `17066.666` | `32 · 533.333` — map X origin |
| `0xceaa88` | (runtime) | per-map Y origin |
| `0x80fee4` | `33.333` | sub-chunk size (yards) |
| `0x86f6b8` / `0x86f6bc` | `1066.666` | `2 · 533.333` — screen-scale denominator |
| `0x8116cc` | `694.444` | blip max world distance (rank gate) |
| `0x811730` | `0.8` | edge-clamp ratio |
| `0x806b10` | `100` | ping auto-clear distance² (`= 10 yd²`) |
| `0x8116e8` | `{150,120,90,60,40,25}` | zoom radius table (orientation-locked) |
| `0x8116d0` | `{14,12,10,8,6,4}` | zoom chunk-count table |
| `0x7ffa24` | `0.5` | |
| `0x7ff9d8` | `1.0` | |
| `0x7ffd74` | `0.0` | |
| `0x8029d4` | `2.384e-7` | orientation renormalize epsilon |
| `0x7ffaac` | `0.0174533` | degrees→radians |

## State globals

The minimap is a file-scope singleton; its "struct" is a cluster of globals.

### Player, view and dirty state

| Address | Type | Meaning |
|---|---|---|
| `0xceaa7c` / `0xceaa80` / `0xceaa84` | f32 | player world position (x, y, z) sampled by the last update |
| `0x86f694` | s32 | current continent / map id (`−1` = none) |
| `0x86f6a0` | f32 | last facing / yaw |
| `0xcea8bc` | u32 | dirty bitmask — bit0 = grid dirty, bit1 = tiles-ready |
| `0xceaa60` | u8 | orientation-locked flag (the rotate-minimap CVar) |
| `0x86f698` / `0x86f69c` | u32 | zoom index (0..5), unlocked / locked variants |

The orientation-locked flag at `0xceaa60` selects which zoom index applies: `0x86f698` when unlocked,
`0x86f69c` when locked.

### Tile-window bounds

| Address | Type | Meaning |
|---|---|---|
| `0xcea9c4` / `c8` / `cc` / `d0` / `d4` / `d8` | f32 | current tile-window world bounds (min.xyz / max.xyz) |
| `0xceaa6c` / `70` / `74` / `78` | s32 | last tile-window indices (lo col/row, hi col/row) — used to detect crossing |

The bounds are recomputed when the player crosses into a new tile (see the driver below).

## The tile LOD cache

The driver clears and refills a **512-entry LOD record array** living at `param_6+0x90` — `param_6` is the
`CWorldFrame` render buffer, so this array is owned by the caller, not by the minimap globals. Each record
has **stride `0xa4` (0x29 dwords)**:

| Record offset | Type | Meaning |
|---|---|---|
| `+0x70 .. +0x87` | 6 × f32 | tile world AABB |
| `+0x8c` | f32 | LOD-select distance |
| `+0x90` | u32 | texture handle |
| `+0x94` | u32 | flags — bit1 = present, bit2 = loaded |
| `+0x98` / `+0x9c` / `+0xa0` | u32 | tile key (col, row, mapId) |

`clear_lod_slots` (`0x6d9e10`) stamps all 512 records with `−1` keys / `FLT_MAX` distances.

**`blip_lod_distance` (`0x6d9e30`)** sets a record's LOD-select distance by matching it on tile key:

```
rec+0x8c = (tile.maxZ + tile.minZ) · 0.5 − bias        // tile.maxZ = p3[8], tile.minZ = p3[5]
```

or `FLT_MAX` (`0x7f7fffff`) when the key is the player's own tile. `load_place_tile` (`0x6d9ed0`) performs
the analogous single `rec+0x8c = z − bias` composition over a boundary-produced position vector while it
builds the path, hash-looks-up the md5 name, MPQ-loads the texture, and fills the record.

## Tile texture streaming + md5translate

The streaming helpers build paths and resolve them through the translation table (its entries are
`MINIMAPMD5NAME` records):

- `build_world_name` (`0x6da330`) — forms the `World\<continent>` prefix.
- `build_tile_path` (`0x6da7b0`) — formats `%s\map%d_%d.blp` and resolves it via the md5translate hash table.
- `tile_grid_shift` (`0x6da5e0`) — applies an integer tile-offset table (`0x811700`), warns
  `MINIMAPCHUNKNOTFOUND` on a miss, and triggers the MPQ load.
- `find_free_lod_slot` (`0x6da150`) — finds an empty record in the 512-entry cache.
- `md5translate_load` (`0x6d9040`) — parses `md5translate.trs` into the hash table.
- `zone_rebuild` (`0x6d8e10`) — rescans the area-POI table and reloads the md5 translation.
- `reset_clear` (`0x6d92c0`) — stamps the caches with `FLT_MAX` / `−1` and frees the hash table.

### The md5translate hash table

A Storm `TSHashTable<MINIMAPMD5NAME>` whose state is the globals `0xcea628`–`0xcea64c`:

| Address | Type | Meaning |
|---|---|---|
| `0xcea628` | ptr | element allocator / manager (vtable `0x811720`) |
| `0xcea62c` | u32 | intrusive link offset (`= 0xc`) |
| `0xcea630` / `0xcea634` | ptr | global used-list head / tail (intrusive `TSLink`) |
| `0xcea638` | u32 | iteration cursor |
| `0xcea640` | u32 | bucket count |
| `0xcea644` | ptr | bucket array (stride `0xc`, `TSExplicitList`) |
| `0xcea64c` | u32 | hash mask (lazily 3, grows ×2, capped below `0x1fff`) |

Each **`MINIMAPMD5NAME`** record is `SMemAlloc(0x40 + nameLen)`:

| Record offset | Meaning |
|---|---|
| `+0x00` | u32 key hash (`SStrHash`) |
| `+0x04` | chain link |
| `+0x14` | ptr to key string (strdup'd) |
| `+0x18` | 0x28-byte value — the hashed md5 directory name |

## POI blips and the nearest-3 selection

The POI working set lives in another block of globals:

| Address | Type | Meaning |
|---|---|---|
| `0xcea650[3]` | f32 | nearest-3 POI screen angles (atan2 results) |
| `0xcea65c[3]` | s32 | nearest-3 POI indices |
| `0xceaaac` | u32 | nearest count |
| `0xceaab0` | u8 | nearest-changed flag |
| `0xcea69c` / `0xcea698` / `0xcea694` | — | POI-pointer `TSGrowableArray<AreaPOIRec*>` (data / size / cap, stride 4) |
| `0xcea688` / `68c` / `690` and `0xcea67c` / `680` / `684` | — | two further pointer arrays |
| `0xceaaa8` | u32 | POI count |

**`blip_distance_nearest3` (`0x6d9a90`)** selects, per frame, the three nearest points of interest. For each
POI:

```
d = √((px − player.x)² + (py − player.y)²)
in-range  iff  d / zoomScale ≤ 0.8   (cull)   and   d ≤ 694.444   (rank)
```

It keeps the nearest three ranked by `(areaId ascending, d ascending)`, and for each kept POI computes the
screen angle via `atan2` (the engine-util transcendental at `0x47f220`; see [Math primitives](math-primitives.md)).

This routine reads the fully world-loaded POI table (`0xcea69c`, fed from the area-POI world data at
`0xc0e054`); the 2-D distance itself is never stored — it feeds the cull/rank comparisons and the per-POI
angle. The selection therefore depends on live world state and is exercised at runtime, not by a closed
formula over fixed inputs.

## Party / raid / corpse blips and edge-clamp

The blip slots are five records at `0xcea760`, **stride `0x74`**:

| Slot offset | Meaning |
|---|---|
| `+0x10` / `+0x14` / `+0x18` | position |
| `+0x1c` | mapId |
| `+0x28` | 0x40-byte data |
| `+0x48` | source kind (party/raid index, or `−2`) |

The list handed to consumers uses a separate **stride `0x4c`** output record: 0x40 bytes of data, then
`angle` at `+0x40`, `isSelf` at `+0x44`, and `kind` at `+0x48`.

**`place_party_raid_blips` (`0x6dad10`)** — for each party/raid member and the corpse:

```
d = √(dx² + dy²)
if  d / zoomScale ≤ 0.8 :  place inside  (record the unit position)
else                     :  clamp to the edge at angle atan2(...)   // edge-clamp ratio 0.8 = 0x811730
```

This reads the entity/object manager — the party/raid roster (`0x468550` / `0x468460`) and unit positions
via the unit vtable's `GetPosition`. Like the POI selection, it depends on fully-loaded world/entity state
and is exercised at runtime rather than computed from fixed inputs.

## Zoom → scale

**`zoom_to_scale` (`0x6da9b0`)**, with a byte-identical `float10` twin at **`0x6dab70`**:

```
if (orientation-locked flag 0xceaa60 != 0):
    return radiusTable[0x8116e8][zoom_locked]          // {150,120,90,60,40,25}
else:
    return chunkTable[0x8116d0][zoom] · 0.5 · 33.333   // {14,12,10,8,6,4} · 0.5 · 33.333
```

## Orientation basis renormalization

**`orientation_renormalize` (`0x6da180`)** keeps the 3×3 minimap orientation basis at `0xceaa20`–`0xceaa48`
orthonormal. When `|Σ row0² − 1.0| ≥ eps` (`eps = 2.384e-7` at `0x8029d4`) it Gram-renormalizes:

```
f = 1.0 / √(Σ row0²)
each row k is scaled by  f / √(Σ rowk²)
```

It then sets the facing column from `facing · 0.0174533` (degrees→radians, `0x7ffaac`; facing source
`0x86f6b4`) via `0x7bdd60`. This operates entirely on the minimap's own orientation globals — it is local to
the minimap, not a shared math helper.

## The per-frame driver

**`minimap_update` (`0x6d93a0`)** is the per-frame entry called from `CWorldFrame` (`0x4ecc14`). It sequences
orientation, coordinate, blip and tile-stream work and sets the dirty bitmask (`0xcea8bc`: `&1` clears grid
dirty, `|2` marks tiles-ready). It owns two FP computations:

**Tile-window snap** — when the player crosses into a new tile, with `c = chunkW`:

```
cx = round((1.0/c) · player.x − 0x86f690) · c
cy = round((1.0/c) · player.y − 0x86f690) · c
bounds = [cx, cx + c] × [cy, cy + c]        // written to 0xcea9c4 .. 0xcea9d8
```

**Ping auto-clear** — the ping is dropped once the player reaches it:

```
if (player − ping)² < 100:   clear the ping        // 100 = (10 yd)², at 0x806b10
```

The ping state lives at `0xcea7e4` / `0xcea7e8` (ping world x / y) with the expiry tick at `0xceaaa4`
(`GetTime + 0x1e0`); `0xcea7d4` is the accessor target read by `Minimap_GetPing`.

## API surface

The accessors and mutators the Lua/UI layer and the render path call into:

- **Zoom / flags**: `get_zoom_index` (`0x6da980`), `get_zoom_levels` (`0x6da9a0`, returns 6), `set_zoom`
  (`0x6da8e0` — clamps to 5, marks dirty, persists via `CVar::Set`), `get_update_flags` (`0x6daa00`),
  `get_ping_ptr` (`0x6dad00`).
- **Blips / POI**: `build_blip_list` (`0x6daa20` — memcpy of the nearest-N records, stride `0x4c`),
  `refresh_for_unit` (`0x6dabc0`), `set_blip` (`0x6dac10` — fills a slot at stride `0x74`, sets ping expiry
  `GetTime + 0x1e0`), `set_blip_wrap` (`0x6dacb0`), `set_corpse_blip` (`0x6dacd0`).

The MD5 / POI Storm containers (the `TSHashTable` / `TSGrowableArray` / `TSExplicitList` machinery over
`MINIMAPMD5NAME` and `AreaPOIRec*`) are pure pointer/integer surgery with no floating-point arithmetic —
node dtor `0x6dafd0`, list insert `0x6db040`, array grow `0x6db310`, rehash `0x6db820`, and the rest of the
`0x6db0xx`–`0x6dbbxx` band.
