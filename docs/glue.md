# Glue screens — login, realm list, character select/create

The "glue" is everything the client shows before the world loads: the login screen, the realm
list, and character select/create. It is driven by a single manager, `CGlueMgr`, which runs a
small screen-flow state machine, hosts the GlueXML UI (a FrameScript/Lua binding layer), manages
the realm list, and drives the 3D character preview. The manager itself owns only a small set of
look-defining math functions — viewport aspect, realm-population statistics, and a handful of customization/preview
scalars — and delegates the network wire to [Net](net.md), the preview render to [Models](models.md)
and the [Graphics device](graphics-device.md), and the widget framework plus the Lua VM to [UI](ui.md).

The glue code lives in one contiguous translation-unit cluster, the virtual-address range
`[0x46a400, 0x476020)`, spanning six source files: `CGlueMgr.cpp` (the manager and state machine),
`ClientServices.h` (the GlueXML binding layer), `RealmList.cpp`, `CSimpleModelFFX.cpp` (the preview
controller), `SurveyDownloadGlue.cpp`, and `PatchDownloadGlue.cpp`.

## The screen-flow state machine

`CGlueMgr` is a singleton derived from `CSimpleTop`. It is brought up by `CGlueMgr::Initialize`
(`0x46a400`), called from client startup at `0x402b36`. Initialize does three things:

- Polls the four network-connection state objects (`0x882720`, `0x882688`, `0x8826d8`, `0x88265c`).
  Each object's connection state is the dword at `+0x28`.
- Builds the GlueXML UI.
- Registers the per-frame handler on event-bus category `5`. The registration is
  `mov edx, 0x46b930; mov ecx, 5; call 0x41fca0` at `0x46a52d` — i.e. it binds `CGlueMgr::Update`
  (`0x46b930`) as the category-5 callback.

`CGlueMgr::Update` (`0x46b930`) is the state machine. It switches on the current screen state,
`screenState`, a global at `[0xb41da0]` taking values `0..0xb` (login, realm select, charselect,
charcreate, the patch/survey download screens, and so on).

## Viewport and aspect ratio

The glue screens compute their own viewport aspect rather than reading a global render aspect.

`glue_aspect` (`0x46ab40`) computes the aspect as an integer-to-float divide — `fild` the width,
`fidiv` by the height — defaulting to 4:3:

```
aspect = (f32)width / (f32)height        // default 4:3
```

`glue_onresize` (`0x46abe0`) is the resize change-detector: it only reacts when the aspect actually
moves, using a fixed threshold:

```
changed = |oldAspect - newAspect| >= 0.01
```

`glue_init_viewport` (`0x46a7b0`) converts a viewport's floating pixel extent to an integer screen
dimension by truncation:

```
dimension = (i32)(hi - lo)
```

The same `(i32)(hi - lo)` truncation appears inline in `CGlueMgr::Initialize` (`0x46a400`); there it
produces an integer window dimension that feeds window setup rather than any look-defining float.

## Realm-list population statistics

The realm list shows each realm's relative population. The client derives this from raw population
counts in two steps.

`realm_load_stats` (`0x46e510`) computes the mean and a scaled standard deviation over the `n` realm
population values:

```
mean   = Σ pop / n
stddev = sqrt( Σ (pop - mean)² / (n - 1) ) · 0.6
```

The `0.6` scale is applied to the standard deviation as stored. This is the one `fsqrt` in the glue
code. For a degenerate list (`n ≤ 1`) the function short-circuits to `(mean, stddev) = (1.0, 0.0)`.

`realm_load_classify` (`0x46ec60`) maps a single realm's population to a load band, first honouring
the flag sentinels `-3`, `-2`, `2`, then bucketing against the mean ± stddev window:

```
pop < mean - stddev   →  -1   (low)
pop > mean + stddev   →   1   (high)
otherwise             →   0   (medium)
```

## Character customization and preview

The character-create screen normalizes packed colours and drives the preview model's scale and
rotation through a few small scalars.

`char_customize` (`0x4727f0`) normalizes a packed RGB colour — each of the three channels is a `u8`
multiplied by `1/255` — and clamps a scalar to `[0, 1]`:

```
channel = (u8)byte · (1/255)            // ×3 channels
scalar  = clamp(value, 0.0, 1.0)
```

`charcreate_blend` (`0x472950`) computes the preview scale as a simple product and interpolates a
frame across a range:

```
scale = a · b
frame = lo + (hi - lo) · (idx / total)
```

The Lua bindings for the preview convert between radians and degrees and scale the model in world
units. The degree conversions differ in their internal precision:

- `lua_rad2deg` (`0x471910`, `0x473650`): `g · 180/π`, computed and pushed as `f64`.
- `lua_deg2rad` (`0x471930`, `0x473670`): `deg · π/180`, stored as `f32`.
- `lua_model_scale` (`0x46dad0`, `0x46dd30`, `0x46dd80`): `scale · 1024 · coord`.

The preview controller `CSimpleModelFFX` is a UI `CSimpleModel` subclass; it authors no transform of
its own — it positions and scales the M2 through these bindings and delegates the actual render.

## What the glue delegates

The glue layer is deliberately thin. Three large areas live elsewhere:

- **The network wire** — realm list, character enumeration, and login — belongs to [Net](net.md). The
  four connection objects (`0x882720`, `0x882688`, `0x8826d8`, `0x88265c`, each with its state at
  `+0x28`), the `RealmList`/`Login.cpp` data, and the query/response message handlers are net's;
  glue is a caller and consumer.
- **The 3D character preview render** delegates to [Models](models.md) (the M2 render), the
  [Graphics device](graphics-device.md), and [UI](ui.md). The character-model components the preview
  drives live in `CharacterComponent.cpp` (the translation unit at `0x476020+`, just past the glue
  cluster) — see [Character model](character-model.md).
- **The widget framework and the Lua VM** are [UI](ui.md)'s; the GlueXML bindings are just glue's
  C-functions registered into that VM.

## Runtime-only behaviour

The end-to-end screen-flow — what the realm list actually shows, what characters appear, whether
login succeeds — depends on server-fed data (the realm-list response, `SMSG_CHAR_ENUM`, and the login
responses). That flow is read from live network state at runtime, not produced by any closed formula
in the glue code itself; only the per-screen math above (aspect, statistics, customization scalars)
is self-contained.
