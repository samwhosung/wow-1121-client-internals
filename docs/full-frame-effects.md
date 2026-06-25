# Full-frame effects — glow, bloom, death fade

The 1.12.1 client's full-frame post-processing, the `FFX::Pass` family from
`Source\Game\GameClient\FFXEffects.cpp`. Three effects run over the whole rendered frame: the
full-screen **glow** (the iconic vanilla "bloom"/over-bright shimmer), the **death-fade**
desaturation (the screen draining to grey when you die), and a **rectangle** pass. Each is toggled by
a CVar — `ffxGlow`, `ffxDeath`, `ffxRectangle`. The client computes the effect *parameters* on the
CPU; the actual per-pixel blur/glow/desaturation runs on the GPU as fragment programs (see
[Graphics device](graphics-device.md)).

## Structure and invocation

The subsystem occupies `[0x6ca9d0, 0x6cdf50)`. Per-world init constructs the three pass objects and
stores them in globals:

| Pass | Constructor | Global |
|---|---|---|
| Glow | `0x6cc130` | `0xb414c4` |
| Death | `0x6cc690` | `0xb41468` |
| Rectangle | `0x6ccfc0` | — |

Each frame, the apply hook **`0x6cd890`** (called from the render driver at `0x46fad3` / `0x48350e`)
composes the list of active passes and submits them to the graphics device. The active pass set is
selected by `0x6cde60` (state at `0xce8bb4`). Pass render methods are **vtable-dispatched** (the
`FFX::Pass` vtable family at `0x811480`–`0x8114fc`, base `0x81148c`).

## What the client computes (CPU side)

The per-pixel convolution is on the GPU; what the CPU computes is the geometry, the colour packing,
and the animation of the glow. These are the pieces an implementer must reproduce exactly.

### The glow-wave LUT (`0x6cbea0`)

The animated shimmer of the glow is driven by a 128-entry sine table:

```
lut[i] = sin(i · 2π / 128)        # x87 fsin, i in [0, 128)
```

packed in two formats:

```
signed:    round(lut[i] · 128)            clamped to [-128, 127]   -> s8
unsigned:  round((lut[i] · 0.5 + 0.5) · 255)  clamped to [0, 255]  -> u8
```

(The rounding is the x87 `__ftol` truncation-to-zero conversion.)

### The glow pass (`0x6cb310`)

The dominant render method. It derives a phase from the frame time, scrolls the glow-wave UVs by it,
and packs float colour to bytes with:

```
byte = (channel · 255 + 512) >> 14
```

The **death-fade desaturation** factor is `× 3.0 × 0.25` (= `× 0.75`).

Colour and death byte-packing also appear standalone in `ffx_glow_color_quantize` (`0x6cb020`) and
`ffx_death_pass_pack` (`0x6cb930`).

### Quad geometry

- **Fullscreen quad** (`0x6cd580`): NDC corners at `±0.5`, with a half-texel UV correction
  `uv = (src + 0.5) · (1 / dst)`.
- **Quad UV transform** (`0x6cd750`): affine `uv' = uv · scale + offset`.

### Render-target dimensions (`0x6cdb40`)

The glow is rendered at a reduced resolution and composited back up — the standard bloom pipeline. The
RT-dimension chain computes `1/w` and `1/h` reciprocals, the ½ and ¼ downsample sizes, and a
power-of-two / cap clamp.

## What runs on the GPU

The actual blur/glow/death convolution is delegated to the graphics device as **ARB fragment
programs** loaded from `.bls` files: `FFXBox4`, `FFXGauss4`, `FFXGlow`, `FFXGlowWave`, and the
`FFXDeath*` set. Worth noting for implementers: the 1.12.1 client is **not** purely fixed-function —
these full-frame effects are real programmable-shader passes (a box/gaussian blur of a downsampled
target, composited with the glow-wave animation; a desaturation pass for death). The CPU math above
feeds those programs their per-frame constants and geometry.
