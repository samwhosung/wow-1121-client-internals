# Loading screen — the world-enter loading bar and taxi animation

The world-enter loading screen (`Source\LoadingScreen.cpp`, the function band `[0x406640, 0x409040)`) is
what the client shows during the glue→world transition: a progress bar that fills 0→1 as the zone streams
in, a full-Azeroth overview map drawn from twelve tiles, and — when the player is travelling by taxi — a
"flying ship on a route" ribbon animation drawn over that map. Everything is composed under a single
letterboxed full-frame orthographic projection. This chapter covers the geometry and the math the client
computes; the actual full-frame composite is assembled at render time from live device and world state.

## Entry points and lifecycle

`begin_transition_by_area` `0x4072c0` (and its world-enter variant `_variant` `0x407de0`) starts the screen.
It records the entered area into `0x82f00c` and resolves the player's taxi node / zone — this selects **which
taxi route** to highlight and **which UV region** of the world map to show, *not* a background directory (see
the gotcha below). `init` `0x406800` resets the progress globals and registers the per-frame render callback
`0x406a60` via the UI callback registrar `0x442800`.

Progress is pushed in by three setters `0x407f70` / `0x407fa0` / `0x407fd0`. These are monotonic latches: they
compare and store, with no floating-point arithmetic. Teardown is `0x407e80`.

## The global state cluster (`0x882xxx`)

All loading-screen state lives in a single cluster of globals, initialised by the translation unit's static
initialisers.

| Address | Type | Meaning |
|---|---|---|
| `0x882be4` | f32 | combined bar fill fraction `[0,1]` — output of `progress_fraction`, read by both the bar and the ribbon reveal |
| `0x882bd0` | f32 | progress input A |
| `0x882df4` | f32 | progress input B |
| `0x882894` | f32 | progress "phase" flag (0 / 1), monotonic-latched |
| `0x8828e0` / `e4` / `e8` / `ec` | — | `TSGrowableArray<DYNAMICELEMENTVERT>` header `{capacity, count, data*, growHint}` |
| `0x8828f8` | `[12×0xc]` | world-map tile atlas-UV array |
| `0x882be8` | `[12×8]` | world-map tile screen-quad corner array |
| `0x8828a4` | u32 | ribbon / ship vertex tint |
| `0x8828b8` | `[4×0xc]` | ship-sprite local-quad template (written by a sibling TU — see below) |
| `0x882bac` | `[4×8]` | ship-sprite template texcoords (sibling TU) |
| `0x882da8` | `[2]` | bar texture handles, order `{fill, border}` |
| `0x882b70` | handle | taxi-route ribbon texture |
| `0x882d70` | `[12]` | world-map tile textures |
| `0x882e04` | handle | dynamic-element texture |
| `0x882be0` | handle | active texture handle |
| `0x882e08` / `0c` / `14` | — | loading-text font / string / object |
| `0x882b54` / `58` / `68` / `64` | — | taxi zone-graph (count / zone-array / node-array / default) |

x87 math runs at PC_53: intermediates are f64, stores are f32. The formulas below are computed in that
precision.

## The progress fraction (`progress_fraction` `0x406920`)

The combined fill fraction is a two-input weighted sum plus an optional phase base, then clamped:

```
phase = (0x882894 != 0)
base  = phase ? 0x882894 · 0.5 : 0
(w1, w2) = phase ? (0.35, 0.15) : (0.70, 0.30)
f = 0x882df4·w2 + 0x882bd0·w1 + base
f = (f < 0) ? 0 : (f >= 1) ? 1 : f       // clamp to [0,1]
→ store to 0x882be4
```

The weights are stored as exact f32 constants: `0.70` = `0x7ffd7c`, `0.30` = `0x7ffd78` (unphased);
`0.35` = `0x7ffd70`, `0.15` = `0x7ffd6c` (phased).

## The progress bar

### The bar descriptor table (`0x7ffd34`)

The bar is driven by a two-entry table at `0x7ffd34`, stride `0x18`. The renderer iterates base →
`0x7ffd64` (`(0x7ffd64 − 0x7ffd34) / 0x18 = 2`), skipping any entry whose parallel texture handle
`0x882da8[i]` is null.

```
struct BarDescriptor {        // stride 0x18 @ 0x7ffd34
  const char* texturePath;    // +0x00  (loaded by 0x406820 -> 0x882da8[i])
  int32_t     isFill;         // +0x04  (read as a byte at 0x4071d4)
  float       cx;             // +0x08
  float       cy;             // +0x0c
  float       w;              // +0x10  (full width;  half-extent = w·0.5)
  float       h;              // +0x14  (full height)
};
```

| # | role | texturePath | isFill | cx | cy | w | h |
|---|---|---|---|---|---|---|---|
| 0 | **fill** | `Interface\Glues\LoadingBar\Loading-BarFill` | 1 | 0.5 | 0.075 | 0.525 | 0.025 |
| 1 | **border** | `Interface\Glues\LoadingBar\Loading-BarBorder` | 0 | 0.5 | 0.075 | 0.6 | 0.05 |

Both quads are centred at `(0.5, 0.075)` in the `[0,1]` letterboxed ortho. Draw order is fill (entry 0) then
border (entry 1); the handle array `0x882da8` is therefore `{fill, border}`.

### The bar quad (`progress_bar_quad` `0x407150`)

For each descriptor, the quad corners are the centre ± half-extent. For the fill quad, the right edge is
pulled in proportional to the fraction:

```
x0 = cx − w·0.5      x1 = cx + w·0.5
y0 = cy − h·0.5      y1 = cy + h·0.5
if isFill:  x1 = x0 + progress(0x882be4)·w
```

(`0.5` = `0x7ffa24`.) The fill's right edge is computed at `0x407217` as `fld [0x882be4]; fmul [esi+0x10];
fadd x0`.

## The letterboxed ortho (`aspect_fit_ortho` `0x406a60`)

The per-frame render callback fits a `[0,1]×[0,1]` design space into the screen, letterboxing on the longer
axis:

```
a = ScreenW / ScreenH
if a > 1:  w = 1/a;  xlo = (1−w)·0.5;  h = 1;  ylo = 0      // pillarbox
if a < 1:  h = a;    ylo = (1−a)·0.5;  w = 1;  xlo = 0      // letterbox
else:      full 0..1
ortho far = 500.0
```

## The loading text (`loadingbar_text_layout` `0x406640`)

The loading-text label is scaled to a fixed pixel size and placed just above the bar:

```
a    = ScreenW / ScreenH
s    = cmath_pixel_snap(515.0 / (a · 1024.0))      // 515.0=0x7ffd64, 1024.0=0x7ffd68
yoff = (a < 1) ? (1 − a)·0.5 : 0
pos.x = 0.5 − s·0.5
pos.y = 0.025 + 0.075 + yoff
```

`cmath_pixel_snap` is the pixelize/normalize round-trip (`0x41ae40` / `0x41ae60`) that reads the live gx
pixel-scale (`0x832a44` / `0x832a48`). The font string is created and measured at the font/device boundary
(`0x44d040` / `0x44d340`). Because the scale and aspect come from live device state, the final position is
resolved at render time, not from a closed formula. See [Text layout](text-layout.md).

## The world-map background

The background is **fixed** — it is always the full-Azeroth overview map, independent of the area being
entered. The loader `0x4074b0` builds twelve tile paths with `snprintf("Interface\WorldMap\World\World%d",
i+1)` for `i = 0..11` (the format string is `0x82f1d4`), creating each texture into `0x882d70[i]`. The `%d` is
the **tile ordinal**, not an area id, so the tiles are simply `World1` … `World12` (the loader appends
`.blp`).

The twelve tiles form a 4-column × 3-row, row-major grid; tile index `t = r·4 + c`. That same `t` indexes the
atlas-UV array `0x8828f8` and the screen-quad corner array `0x882be8`.

### Tile geometry (`worldmap_tile_geometry` `0x4081e0`)

The atlas is `1002×668` pixels with `256`-pixel tiles (so the right column and bottom row are clamped), and V
is flipped:

```
u0 = (c·256) / 1002
u1 = min(c·256 + 256, 1002) / 1002
v0 = 1 − (r·256) / 668
v1 = 1 − min(r·256 + 256, 668) / 668
```

The reciprocal scales are exact f32 constants: `1/1002` = `0x7ffdb8`, `1/668` = `0x7ffdb4`. The screen-quad
corner array uses constant edges — columns at `1.0` / `0.9140625`, rows at `1.0` / `0.609375`.

The taxi ribbon texture is a separate asset, `Interface\Glues\LoadingScreens\DynamicElements` (`0x882b70`),
not one of the tiles.

## The taxi route ribbon

When the entered area resolves to a taxi route, a ribbon ("flying ship on a route") is tessellated along an
interpolated spline and revealed proportionally to the loading progress. The spline itself is a shared
interpolation helper: arc-length at `0x453360` and point-at-`t` (which performs the floating-point
interpolation and clamps `t` to `[0,1]`) at `0x453390`.

### Tessellation count (`route_tessellate` `0x4075a0`)

```
segs      = max(__ftol(arclen · 66.6667), 1)        // arclen from 0x453360
vertCount = segs·4 + 12
```

The taxi density `66.6667` is `0x7ffd80`.

### Ribbon vertices (`route_ribbon_tessellate` `0x407c80`)

```
step = 1 / segs
for t in [0,1] (clamped):
    out.xyz = spline(t)(0x453390) + template.xyz(0x8828b8)
    out.uv  = template.uv (copied)
```

Each ribbon vertex is the spline point offset by the corresponding ship-sprite template corner. This loop
runs over the live spline output and the sibling-supplied template (see below), so it is evaluated at render
time.

### Endpoint markers (`route_endpoint_quads` `0x407900`)

Two endpoint marker quads, each centred at the route end with half-extents `±(0.02, 0.0265)`, `z = 0`. Their
UVs are the left/right halves of the ship sprite. The half-extents are exact f32 constants: `0.0265` =
`0x7ffda8`, `0.02` = `0x7ffdac`.

### Proportional reveal (`route_progress_extent` `0x406fb0`)

The ribbon is revealed in step with the bar. The function tints the ribbon vertex buffer in place while
`progress(0x882be4) < i / (quadCount − 1)` (a single `fdiv`), then builds the triangle index buffer and draws.
The meaningful output — how far along the live ribbon buffer the reveal reaches — is produced at render time
over the live vertex buffer.

## The ship-sprite template (input from a sibling TU)

The ship-sprite billboard template `0x8828b8[4×0xc]`, its texcoords `0x882bac[4×8]`, and the tint `0x8828a4`
are **not** written inside `LoadingScreen.cpp` — they are populated by a sibling translation unit (the
`CLoadingScreenShipInfo` / `TaxiPathNode` static-table code) and consumed here by `route_ribbon_tessellate`,
`route_endpoint_quads`, and `route_progress_extent`. An implementer must supply this template as an input to
the ribbon math.

## Constants

All are exact f32 bit patterns at the listed address.

| Value | Address | Use |
|---|---|---|
| `0.5` | `0x7ffa24` | bar half-extents |
| `1.0` | `0x7ff9d8` | — |
| `0.0` | `0x7ffd74` | — |
| `0.70` / `0.30` | `0x7ffd7c` / `0x7ffd78` | progress weights (unphased) |
| `0.35` / `0.15` | `0x7ffd70` / `0x7ffd6c` | progress weights (phased) |
| `515.0` / `1024.0` | `0x7ffd64` / `0x7ffd68` | text aspect-fit |
| `500.0` | — | ortho far plane |
| `66.6667` | `0x7ffd80` | taxi tessellation density |
| `0.0265` / `0.02` | `0x7ffda8` / `0x7ffdac` | ship-sprite half-extents |
| `1/1002` / `1/668` | `0x7ffdb8` / `0x7ffdb4` | world-map U / V scale |

## Dead ends worth noting

- **The area does not select the background.** `begin_transition_by_area` resolves the player's taxi node /
  zone to choose the taxi route and the UV region of the world map; the background tiles are always the fixed
  `World1`…`World12` overview. The per-zone glue artwork under `Interface\Glues\LoadingScreens\…` is a
  different subsystem — the glue / screen manager driven by `Map.dbc` — and is not part of this code. See
  [Glue](glue.md).
- **Bar handle order is `{fill, border}`** in `0x882da8`, matching the draw order.
- **The ship-sprite template is external** to this translation unit; see the section above.
</content>
</invoke>
