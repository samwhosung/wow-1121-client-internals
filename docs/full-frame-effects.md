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

The blur/glow/death convolution is delegated to the graphics device as **ARB fragment programs**
loaded from `.bls` files: `FFXBox4`, `FFXGauss4`, `FFXGlow`, `FFXGlowWave`, and the `FFXDeath*` set.
Worth noting for implementers: the 1.12.1 client is **not** purely fixed-function — these full-frame
effects are real programmable-shader passes. The programs ship as data in the MPQ, so the per-pixel
arithmetic below (kernel weights, the combine expression) is read directly out of `FFXBox4.bls` /
`FFXGauss4.bls` / `FFXGlow.bls`; the **tap offsets** are CPU-fed texcoords, i.e. part of the geometry
this document's subsystem computes, not the shader text.

### The glow shader chain

For the standard full-screen glow the GPU runs four passes — one downsample, a separable blur, and
the combine:

```
blur = Gauss4( Gauss4( Box4(scene → ¼), horizontal ), vertical )
out  = lerp(screen, blur, z) + w · blur²
```

**Downsample — one Box4, full → ¼.** Note that although the RT-dimension chain (`0x6cdb40`, above)
computes *both* ½ and ¼ sizes, the glow blur uses a **single** Box4 pass straight from full
resolution to ¼ — the ½ target is allocated but the glow never samples it. Box4 is 4 bilinear taps at
source-texel offsets `{−1.5, +0.5}²` (a 4×4 footprint), averaged `× 0.25` (tap table `0xce89cc`).

**Blur — separable Gauss4 at ¼ res**, horizontal then vertical. Four taps per axis (`FFXGauss4.bls`
weights; tap offsets from the CPU array at `0x6cad10`):

| offset (source texels) | weight |
|---|---|
| ±0.5 | 0.375 (⅜) |
| ±2.5 | 0.125 (⅛) |

i.e. the shipped `⅛ ⅜ ⅜ ⅛` kernel. Base UVs are texel-centre aligned with zero half-texel residual,
so the offsets transcribe as exact source-texel counts.

**Combine — the square-law.** `FFXGlow.bls` is exactly `out = lerp(screen, blur, z) + w · blur²`,
alpha 1:

- `w` is the per-zone glow gain — the `LightParams` glow weight, **authored data** (≈0.647 in
  Elwynn), *not* a tuned constant.
- `z` is the haze/mix term, **0** for the ordinary scene glow (a per-object attribute can raise it).

The **`blur²` square-law is the whole character of the vanilla glow**, and the detail most "bloom"
reimplementations miss: mid-tones barely bloom (a 0.5 blur contributes 0.25) while highlights bloom
fully. A linear additive composite — everything blooming in proportion to its brightness — cannot
express that curve.

**Colour space.** The pass runs in **gamma bytes**, not linear: the RTs are 8-bit gamma-encoded, the
blur convolves gamma values, and the combine is a **saturating byte add** (`out` clamps at 1.0 — the
framebuffer is LDR past this pass). Reproducing it in a linear-HDR renderer means encoding both terms
back to gamma, doing the byte math, clamping, and decoding once; run the square-law in linear space
and the weighting comes out wrong.
