# Lighting — time-of-day light, sky, fog, weather

How the 1.12.1 client lights the world: a time-of-day state drives a table of colours and scalars,
which are blended per frame into a single block of light parameters that all geometry reads. Lighting
is **CPU vertex-lit** — there is no fixed-function `D3DRS_AMBIENT`; the only lighting value pushed to
the GPU as render state is fog. The data behind it comes from five `Light*` DBC tables.

## The day/night state and the per-frame world-light block

The day/night state (`DNState`) lives at `0xce9b60`. Its time-of-day clock is `0xce9b64`, with a
domain of `[0, 2880)` half-minutes — a full 24-hour day in half-minute ticks.

Each frame, the refresh `0x66ff60` copies `DNState`'s colour slots into a **world-light block** at
`[0xc89ef0]+0x7c`. All geometry — terrain, M2, WMO — reads its light from this block, so the block is
the single point of truth for the current frame's lighting.

- **Global ambient** = `DNState+0x178` (`0xce9cd8`) → block `+0xac` → the ambient accumulator
  `0x71bc70`. Terrain (`0x71bf90`), M2 (`0x6a8115`/`0x6a812f`), and WMO all **share** this one ambient
  value; it is not per-object-type.
- **Directional (sun/moon) diffuse colour** = colour `table[8]` (`0xce9c2c`) → the sun diffuse slot
  `0xce97c0` and moon slot `0xce9710` (broadcast at `0x6d2928`/`0x6d2934`, applied at `0x7e57e0`). The
  light direction is the gated sun/moon direction (`sun_direction_at`).

## The sky dome

The sky is a **multi-band vertical gradient dome**, not a single clear colour. The gradient is six
contiguous colour bands, `table[2..7]`, drawn top-to-bottom by the dome loop `0x6d0f50`
(`esi = 0xce9c14` = `table[2]` base, six iterations, `+4` per band):

- `table[2]` (`0xce9c14`) = the apex / overhead colour (top of the dome)
- … through …
- `table[7]` (`0xce9c28`) = the horizon colour (which is also the fog colour, see below)

The sky-array dome `0x6cfb00` layers `table[8]` (`0xce9c2c`) plus `table[9..11]`
(`0xce9c30`/`0x34`/`0x38`) over this base gradient.

## Fog

Fog is the one lighting output that becomes GPU render state:

| Value | Source | Destination | Render state |
|---|---|---|---|
| Fog colour | colour `table[7]` (`0xce9c28`) | `0xce9bd0` | `0xd` |
| Fog end | scalar band sub-0 (`0xce9c50`) | `0xce9bd8` | `0xb` |
| Fog start | scalar band sub-1 (`0xce9c54`) **× fog end** | `0xce9bd4` | `0xa` |

Fog start is stored as a **fraction of fog end** in the data and multiplied through at update
(`0x6cee30`/`0x66ff20`).

## Band addressing

The light data is organised into bands selected by a band-group id `G` (from `LightParams`, 1-based):

- **Colour band id** = `18·(G−1) + 1 + sub`, indexing into `[0xce9da4]`
- **Scalar band id** = `6·(G−1) + 1 + sub`, indexing into `[0xce9d90]`

So a group has 18 colour bands and 6 scalar bands; `sub` selects within the group.

## The day/night blend weight (`bcc`)

`bcc` (`[0xce9bcc]`) is the runtime weight that blends the **day** light record over the **night**
record. It is **not** sun elevation — a common misreading.

```
bcc = min(1.0, density · 4.0)
```

where `density` = `DNState+0x40` (`[0xce9ba0]`), the clamped cloud/weather density. The weather
intensity ramp (`0x67bc70`) writes `density`; `cloud_density_clamp` (`0x6d4500`) computes `bcc` from
it (`fld [0xce9ba0]; fmul 4.0; fcomp 1.0; clamp`) — there is no sun term in it at all.

The colour-table builder `0x6d2260` consumes `bcc`:

- `bcc > 0` → blend the day record over the night record with weight `bcc` (`0x6d67c0(dst, src, bcc)`)
- `bcc != 0` → stamp the alpha leg

The two axes are orthogonal: **sun elevation selects *which* records** (day vs night) are gathered;
**`bcc` (cloud density) selects *how much* day is blended in**. To reproduce, compute `bcc` from the
weather density, never from `sun_dir.z`.

## The Light DBC tables (the data source)

Five tables feed lighting: `Light`, `LightParams`, `LightIntBand`, `LightFloatBand` (plus
`LightSkybox` for skyboxes). All are loaded by the DBC subsystem.

**Container/loader template (all five).** Each container is `{[+0]=rowBase, [+4]=count, [+8]=idMap,
[+0xc]=maxId, [+0x10]=loaded}`. The loader validates the `"WDBC"` magic, checks field-count and
record-size, allocates `count · stride` rows, and builds an **ID-indexed sparse map**: allocate
`(maxId+1)·4`, zero it, set `map[row.ID] = &row` (holes stay null). Note that the container's `[+0xc]`
bound is the **maximum ID, not the row count**.

Container globals: `Light` `0xce9d60..d70`, `LightParams` `0xce9d74..d84`, scalar-band
`0xce9d88..d98`, colour-band `0xce9d9c..dac`.

**`Light.dbc` — 12 columns / 48 bytes** (reader `0x589220`):

| Offset | Field |
|---|---|
| `+0` | ID |
| `+4` | continent / map |
| `+8`, `+0xc`, `+0x10` | X / Y / Z position |
| `+0x14` | falloff start (inner radius) |
| `+0x18` | falloff end (outer radius) |
| `+0x1c .. +0x2c` | 5 `LightParams` references |

The five references are read as one 20-byte block. Reference `k`: `k=0` night, `k=1` day, `k=2,3` a
secondary pair, `k=4` (`+0x2c`) is copied but unused.

**`LightParams.dbc` — 9 columns / 36 bytes** (reader `0x589030`):

| Offset | Field |
|---|---|
| `+0` | band-group id (1-based) |
| `+4` | highlight-sky scalar (i32, used as `(f32)(i32)`) |
| `+8`, `+0xc` | scalars |
| `+0x10 .. +0x20` | a 5-dword scalar tail (cloud/fog/water density block, blended in `0x6d67c0`) |

It has no reference columns; skybox references are the separate `LightSkybox` table.

**The cross-reference rule.** Every reference in this subsystem is a **DBC ID used directly as an array
index** into the ID-indexed map — there is no separate remap step:

- A `Light` row's `+0x1c[k]` is a `LightParams` **ID**, resolved as `[[0xce9d7c] + 4·id]` (bound
  `[0xce9d80]` = maxId).
- A `LightParams` group-id `G` selects its bands by **band ID** `18·(G−1)+1` (colours, into
  `[0xce9da4]`) / `6·(G−1)+1` (scalars, into `[0xce9d90]`).

So an implementer should ID-key all four tables (sizing each map to `maxId+1`) and resolve every
cross-reference as a direct ID lookup. The in-memory strides are **48 B (`Light`) / 36 B
(`LightParams`)**.

## Weather

Weather state is two globals:

- The **`CMapWeather` state manager** `[0xc6326c]` — non-polymorphic; constructor `0x67b740`. The
  current type key is `+0x20`; effect slots are `+0x28`/`+0x2c`/`+0x30`/`+0x34`.
- A **4000-particle drift cloud** `[0xc63180]` (`0xfa40` bytes; constructor `0x68e5a0`).

Rain, snow, and sand are plain tagged records (constructors `0x674620`/`0x677420`/`0x679140`). The
**mist** effect is polymorphic (one slot, the spawn `0x67a990`), and the **fog-density grid sampler**
is a second polymorphic class (six slots, at manager `+0x34`).

Drive path: the `CWorld` frame calls the driver `0x67be40` (a switch on the weather type) with a
camera-delta pre-pass `0x67c150`; particles tick via the gated `0x6701e0`. `SetWeather` (`0x67baf0`)
is called from the network handler `0x48fa5f`. Weather takes **no DBC input**.

## Clocks

Lighting keeps its **own** time-of-day clock at `0xce9b64`, advanced per frame (writer `0x6d1be9`). It
re-anchors to the server-synced game clock **only on a time-sync packet**: the bridge `0x6c5f50`
copies the game-time day-phase straight into `0xce9b64`. See the [Game time](game-time.md) chapter for
the day-phase computation.

## Dead ends worth noting

- The cached MOHD ambient colour at `CMapObj+0x198` is written (`0x6c39aa`) but **never read**. WMO
  interior ambient comes from the MOCV vertex colours, not from `MOHD+0x198`.
