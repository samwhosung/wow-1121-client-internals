# Text layout — per-glyph advance, wrap, measure, glyph quads

The text-layout subsystem (the `GxuFont*` C API over the `CGxFont`, `CGxString`, and `CGxStringBatch`
classes) is where the client turns a byte string into placed glyphs: it computes per-glyph advances and
kerning, wraps and measures text against a box, lays a string out into multiple justified lines, builds the
per-glyph textured quads (position + atlas UV + colour), and manages the glyph atlas the quads sample. The
UI, console, and overlay code delegate *all* text measurement and layout here; the resulting glyph quads are
handed to the [graphics device](graphics-device.md) for drawing. The whole subsystem spans the address
range `[0x5c1ae0, 0x5d1420)`.

This is distinct from the device that draws the quads — the layout math (where each glyph lands, how text
wraps) is computed here in screen-fraction and pixel units; only the final quads cross the device boundary.

## Coordinate model and global state

Text sizes are expressed as a **fraction of the viewport height** (a value in `(0,1)`), and most layout math
converts between that fraction and integer device pixels. Two module globals hold the current viewport pixel
dimensions:

| Global | Meaning |
|---|---|
| `0xc2b9a0` | viewport **height** in pixels (default 480) — divisor of all point-size↔fraction math |
| `0xc2b9a4` | viewport **width** in pixels (default 640) — divisor of horizontal-extent normalization |

These are recomputed by `0x5c2b50` from the live device rect: it reads the rect, substitutes the default rect
`{0, 0, 480.0, 640.0}` if the device reports the unset sentinel, then `__ftol`s the extents into the two
globals (`0xc2b9a4 = ftol(y1−y0)`, `0xc2b9a0 = ftol(x1−x0)`). When the rect changes it stores the new rect
(cache at `0xc2b82c..0xc2b838`) and walks the global live-`CGxFont` registry (`0xc2b978`/`0xc2b97c`),
re-laying every font at the new dimensions.

### Constants

All of these are f32 read from `.rdata`; the layout math depends on the exact bit values.

| Address | f32 | Role |
|---|---|---|
| `0x7ffd74` | `0.0` | the "unset/invalid" sentinel — every `x == [0x7ffd74]` guard is `x == 0.0` |
| `0x7ff9d8` | `1.0` (`0x3f800000`) | unit; upper bound on font size; per-glyph `+1.0` outline metric |
| `0x801628` | `2.0` (`0x40000000`) | shadow pad `+2.0`; `2.0/height` minimum-size floor numerator |
| `0x80306c` | `4.0` (`0x40800000`) | outline pad `+4.0` |
| `0x8026bc` | `2⁻²⁰` (`0x35800000`, 9.5367e-07) | vertical wrap-fit epsilon |
| `0x808120` | `0.99999` (`0x3f7fff58`) | ceiling bias: `__ftol(x + 0.99999) ≈ ceil(x)` |
| `0x7ffa24` | `0.5` (`0x3f000000`) | round-half bias: `__ftol(x ± 0.5)` = round-half-away-from-zero |
| `0x80a92c` | `1/64` (`0x3c800000`) | 26.6 fixed-point → float pixel scale |
| `0x3b800000` | `1/256` | atlas-cell coordinate → normalized UV |
| `0x3b000000` | `+1/512` | shadow UV texel nudge (stored at `0xc2b9f8`/`0xc2b9fc`) |
| `0xbb000000` | `−1/512` | shadow UV texel nudge (stored at `0xc2ba00`/`0xc2ba04`) |
| imm `0x461c4000` | `10000.0` | "infinite" wrap width used by the max-chars search |

`__ftol` (`0x40a2b0`) is the CRT float→int helper: it sets the x87 rounding mode to truncate-toward-zero, so
`__ftol(x + 0.5)` rounds half away from zero and `__ftol(x + 0.99999)` is effectively `ceil(x)`. Two libm
helpers also appear: `0x73fdf5` = `floorf`, `0x73ff3f` = `roundf`.

## The structures

### `CGxFont` — the font object (size `0x250`)

Created by `0x5c1ae0` (`GxuFontCreate`), constructed by `0x5ca920`, configured by `0x5ca1d0`
(`Initialize`/`SetFont`).

| Offset | Field |
|---|---|
| `+0x00`/`+0x04` | live-font registry link (→ `0xc2b978`) |
| `+0x08`/`+0x0c`/`+0x10` | per-font `CGxString` list (link offset 8, head, tail) |
| `+0x14` | glyph-hash container vtable (`0x80a8cc`) |
| `+0x30`/`+0x38` | glyph-`CharCodeDesc` hash bucket array / mask |
| `+0x3c` | kerning-hash container vtable (`0x80a8bc`) |
| `+0x58`/`+0x60` | kerning-pair hash bucket array / mask (`KERNNODE` records) |
| `+0x64`/`+0x68`/`+0x6c` | glyph-LRU list (link offset `0x20`, head, front) |
| `+0x70` | font face / typeface handle (`FACEDATA`) |
| `+0x74` | font name/path, `char[0x104]` (MAX_PATH) |
| `+0x178` | rasterization pixel/em size (the atlas cell size) |
| `+0x17c` | rasterizer load parameter |
| `+0x180` | flags |
| `+0x184` | requested size (the create parameter) |
| `+0x188` | f32 advance-adjust (bold/outline) factor |
| `+0x18c` | glyph-atlas page array, 8 × `0x18` |
| `+0x24c` | rendered pixel size (the em pixel height — denominator of all scale math) |

Flags at `+0x180`: bit0 `0x1` = shadow (`+2.0` pad), bit2 `0x4` = antialias rasterize, bit3 `0x8` = outline
(`+4.0` pad, and forces bit0 on). `GxuFontCreate` validates the requested size is a screen fraction in
`(0,1)` and applies `flags & 8 → flags | 1` (outline forces shadow).

### `CGxString` — a laid-out string (size `0xc4`)

Pool-allocated by `0x5cd830`, built by `0x5cd6d0` (`Setup`/`SetText`), constructed by `0x5cda00`.

| Offset | Field |
|---|---|
| `+0x00`/`+0x04` | global render-list link (→ `0xc2b998`) |
| `+0x08`/`+0x0c` | per-font string-list link |
| `+0x18` | resolved size |
| `+0x1c` | clamped size = `max(size, 2.0/height)` |
| `+0x20`/`+0x24`/`+0x28` | normalized anchor position (x, y, z) |
| `+0x2c` | packed colour (per-glyph vertex colour); `+0x2f` colour/alpha byte |
| `+0x3c`/`+0x40` | x / y position |
| `+0x44` | owning `CGxFont*` |
| `+0x48`/`+0x4c` | text buffer / capacity |
| `+0x50`/`+0x54` | justify selectors (see the layout gotcha below) |
| `+0x58` | wrap-box width |
| `+0x5c` | flags byte (bit7 `0x80` = pre-snapped/rotated, bit5 `0x20` = has-selection, bit4 `0x10` = monospace advance, bit3 `0x8` = outline, bit1 `0x2` = stop-after-first-line) |
| `+0x60` | dirty page bitmask |
| `+0x68`/`+0x6c` | selection start/end (init −1) |
| `+0x70`/`+0x74`/`+0x78` | pixel-snapped screen anchor (x, y, z) |
| `+0x80`/`+0x84`/`+0x88` | vertex-record growable array |
| `+0x90`/`+0x94`/`+0x98` | hyperlink-info growable array |
| `+0x9c` | rendered-glyph count (also the "already built" flag) |
| `+0xa0..+0xbc` | 8 atlas-page vertex lists |
| `+0xc0` | frame-age counter |

The constructor seeds float defaults (no arithmetic): `+0x3c`/`+0x40` = `1.0`, colour `+0x2c` = `0xff000000`
(opaque black), unit scales = `1.0`.

### `CharCodeDesc` — a per-glyph metric record (alloc `0x70`)

Looked up / allocated by `0x5cabd0` (`NewCodeDesc`), filled by the rasterizer (`0x5c81b0`/`0x5d1120`).

| Offset | Field |
|---|---|
| `+0x00` | codepoint key |
| `+0x14` | code high/style byte |
| `+0x20`/`+0x24` | glyph-LRU link (→ font `+0x68`) |
| `+0x28` | atlas page index |
| `+0x2c` | glyph source/bitmap pointer |
| `+0x30`/`+0x34` | atlas cell x-start / x-end |
| `+0x48` | s32 integer pixel advance (cell width) |
| `+0x4c` | f32 step-base metric (read by the advance path) |
| `+0x50` | f32 glyph width metric (font units, scaled) |
| `+0x58` | glyph top bearing (rows) |
| `+0x60`/`+0x64`/`+0x68`/`+0x6c` | atlas UV rect: `v_bottom`, `u_left`, `v_top`, `u_right` |

The two metric floats `+0x4c` (step base) and `+0x50` (width) are distinct. The UV fields are precomputed
(in `0x5c7f50`/`0x5c7f80`) as `cell_coordinate / 256` (the `1/256` constant `0x3b800000`), since the atlas is
256×256.

### `CGxStringBatch` — a batched run set (size `0x34`)

A batch (`0x5c1d60` create, `0x5c1f00` clear, `0x5c1f70` destroy) holds an embedded
`TSHashTable<CGxFont* → run>` so that many strings sharing a font draw together. Its element-list manager
vtable (`0x80a884`) sits at `+0x0c`; the run hashtable is `+0x24` (count) / `+0x28` (buckets) / `+0x30`
(mask). The container bodies are pure intrusive-list/hashtable plumbing.

## Screen ↔ pixel size math

The core converters round a screen-height fraction to whole device pixels, half away from zero, with a
passthrough when the caller's `0x80` "already in raw pixels" flag is set:

```
ScreenToPixelHeight(v)  [0x5c6fa0] = round_half_away_from_zero(480 · v)   ; ftol(480·v ± 0.5)
ScreenToPixelWidth(v)   [0x5c7010] = round_half_away_from_zero(640 · v)   ; ftol(640·v ± 0.5)
PixelToScreenHeight(px) [0x5c7080] = px / 480                            ; integer divide
```

The font's natural 1:1 height, `GetOneToOneHeight` (`0x5c2540`), and the default-size used when a call passes
size 0 or the `flags & 4` "use default" bit, are the same value:

```
defaultSize = (float)font[0x24c] / 480.0
```

`GetPixelSize` (`0x5cae90`) is the trivial accessor `return font[0x24c]`.

A `CGxString` floors its size to two device pixels when built (`0x5ccb80` / `0x5cd6d0`):

```
clamped_size@+0x1c = max(size, 2.0 / 480)
```

`min_metric_screen_height` (`0x5caea0`) takes the larger of a structure-derived minimum (a min-reduction over
an `i16` array, divided by the viewport height) and a supplied floor fraction.

## Per-glyph advance and kerning

`GetCharAdvance` (`0x5c6b70`) returns one glyph's advance in raster-pixel units. It skips leading control
codes, and if an explicit kerning entry resolves it returns that; otherwise it falls to the glyph metric:

```
advance(glyph) = GlyphWidth(desc, mode, size) + (float)(s32)desc[0x48]
```

`GlyphWidth` (`0x5cb080`) has a pixel mode and a scaled mode:

```
mode 0 (pixel): round(desc[0x50])
mode ≠ 0:       (ScreenToPixelHeight(size) / font[0x24c]) · desc[0x50]
```

The **inter-glyph escapement** (the advance carried from the previous glyph plus its kerning) is `ComputeStep`
(`0x5ca2d0`), memoized in the kerning hash. The pair key and bucketing are byte-exact:

```
pairKey     = (cur << 16) ^ (prev & 0xffff)
bucketIndex = font.kernMask(+0x60) & cur
```

A cache node matches when `node[+0] == cur` **and** `node[+0x14] == pairKey`; if its `+0x18` bit `0x2` is set
the cached step `node[+0x1c]` is returned. On a miss the raw kerning amount comes from the OS/FreeType face;
the font owns the clamp-negative, scale, add, and round:

```
glyphStepBase(g)  [0x5ca2b0] = desc[0x4c] + (font.flags & 0x8 ? 1.0 : 0)   ; +1.0 outline bias
step(cur, prev)              = round( glyphStepBase(cur) + min(kern(cur,prev), 0) · font[0x188] )
```

Only negative kerning is kept (`min(kern, 0)`), scaled by the advance-adjust factor `font[0x188]`.

The fixed-width (monospace) variant `ComputeStepFixedWidth` (`0x5ca4b0`) centers each glyph in a fixed cell of
width `R = font[0x178]` (the raster pixel size), in integer arithmetic:

```
m1 = ftol(glyphStepBase(g1)); if (m1 >= R) return R
center = R − floor((R − m1) / 2)
m2 = ftol(glyphStepBase(g2)); if (m2 >= R) return center
return center + floor((R − m2) / 2)
```

Gotcha: the glyph-found branch never writes the cache, so fixed-width is recomputed every call; the *only*
cache write on this path is the glyph-miss case, which latches a cached `0`. The two cached values share one
`KernNode` (`+0x1c` proportional step with flag bit `0x2`, `+0x20` fixed-width with flag bit `0x1`).

## Measuring and wrapping

The shared accumulation identity for a line's width across all the measure/step kernels is:

```
lineWidth = Σ escapement(prev → cur) + lastGlyphAdvance
```

The escapement carries the previous glyph's advance+kerning, and the final `+ lastGlyphAdvance` supplies the
last glyph. Before glyphs, each per-line pen is seeded with a pad: `+4.0` when the outline flag (`0x8`) is
set, else `+2.0` when the shadow flag (`0x1`) is set, plus `floor(ScreenToPixelWidth(extraSpacing))`.

### Horizontal extent — `0x5c6940`

Walks tokens (decoding control codes), tracks the per-line maximum, then scales to device pixels and
normalizes:

```
size = (size == 0 || flags & 4) ? defaultSize : size
width = max(lineMax, lastLine)                        ; per the accumulation identity
width = (ScreenToPixelHeight(size) / font[0x24c]) · width
*out  = (flags & 0x80) ? width : width / 640
```

### Max chars within a width — `0x5c2320` (`GetMaxCharsWithinWidth`)

This is the wrap-fit search. It `_alloca`s a per-char cumulative-width array, then measures the whole string
with the wrap width set to `10000.0` (effectively unbounded), which fills the array and writes the total
width back. Then a subtractive search finds the fit:

```
if (total ≤ boxW) return all cnt chars, *outW = total
else find first i with (total − boxW) < arr[i]; return cnt − i, *outW = total − arr[i]
```

### Wrap / max-chars kernel — `0x5c6c50`

Returns the count of glyphs that fit in `boxWidthFrac` and fills the optional cumulative-width array. The
integer line-width threshold (in glyph units) uses the ceiling bias:

```
maxW      = font[0x24c] · 640 · boxWidthFrac
hf        = max(ScreenToPixelHeight(size), 1.0)
threshold = ftol(maxW / hf + 0.99999)                 ; ceil
scale     = ScreenToPixelHeight(size) / font[0x24c]
```

Per glyph the fit test is `(kern + adv + run) > threshold` — if it exceeds, the glyph is not counted and the
walk stops. Accepted glyphs accumulate `run += kern`, advance the pen, and (optionally) write
`perCharCumWidth[i] = run · scale` (`/640` unless `0x80`). The output width is `(lastAdv + run) · scale`.

### `ComputeStep` dispatch and the two step kernels

`ComputeStep` (`0x5c7260`) validates `size ≥ 0`, `boxWidth ≥ 0`, non-empty text, then routes by the sign of
the flags byte: to the fixed-width kernel (`0x5c7300`) or the proportional kernel (`0x5c7470`).

The proportional step (`0x5c7470`, the word-wrap step) finds how much fits one line, honoring break
opportunities (`0x5c7780`):

```
threshold = floor( (font[0x24c] / ScreenToPixelHeight(size)) · boxWidthFrac · 640 )
fit test  : (kern + adv + run) > threshold ? stop
outWidth  = ((lastAdv + run) · (1/640)) · (ScreenToPixelHeight(size) / font[0x24c])
```

The fixed-width step (`0x5c7300`) measures the whole remaining run with no box, scaling by **raw `size`**, not
`ScreenToPixel(size)`:

```
*outWidth = (escSum + lastAdv) · (size / font[0x24c])
```

then advances past trailing whitespace.

### Vertical fit and block height

`GetTextBlockHeight` (`0x5c2070`) returns the height of `N` laid-out lines plus `N−1` pixel-quantized gaps,
in screen-fraction units:

```
gapPx  = ftol(lineGap · 480 + 0.99999)                ; ceil(gap · height)
height = N · size + (gapPx / 480) · (N − 1)
```

`GetMaxCharsWithinHeight` (`0x5c21c0`) accumulates lines until the box is full, quantizing the gap to whole
pixels (`gap = ftol(gap·480 + 0.99999)/480`) and breaking when a line makes no progress or when
`boxH + 2⁻²⁰ < accumH + lineH` (the `0x8026bc` epsilon guards the boundary comparison).

## Line-break classification

`0x5c7780` decides whether a line break is permitted between a codepoint pair (the kinsoku ruleset, integer
comparisons only). Quick-yes for `-` (0x2d), `:` (0x3a), `/` (0x2f), and when `next == 0xffffffff`. A large
branch tree returns **0** (no break) for "no-break-after openers" (`( [ { " ' 《 「 『 〈 ﹙ ＄ …`) and
"no-break-before closers" tested on `next` (`) ] } 。 、 ！ ？ ： ； ・ ° ' " 」 』 〉 …`), and returns **1**
(breakable) for the CJK/Hangul/Kana script ranges `0x1100–0x11ff`, `0x3000–0xd7af`, `0xf900–0xfaff`,
`0xff00–0xff9f`. Default is 0.

## Markup / control codes

`DetermineQuotedCode` (`0x5c2810`) decodes one token at the cursor and returns its class plus the byte length
consumed (and any parsed colour/codepoint). It is integer/string logic with no floating point. The classes:

| Return | Meaning |
|---|---|
| 0 | colour set (`\|cRRGGBBAA`, 10 bytes; 8 hex nibbles parsed, byteswapped with forced alpha) |
| 1 | colour restore (`\|r`) |
| 2 | newline (CR, LF, or `\|n`) |
| 3 | escaped literal pipe (`\|\|`) |
| 4 | hyperlink open (`\|H…\|h`) |
| 5 | hyperlink close (`\|h`) |
| 6 | literal/normal glyph |

Each control sequence is gated by a flag bit that can disable it (`0x100` colour/restore, `0x200` newline,
`0x400` hyperlink, `0x800` the whole `|` mechanism). The measure kernels treat classes 0/1/4/5 as zero-width,
2 as a line break, and 6 as a measurable glyph. Related helpers: `GxuFontStripControlCodes` (`0x5c2590`,
table-driven keep/skip via `0x85f4d8`) and `GxuFontFormatColor` (`0x5c2700`, prints `%2.2x%2.2x%2.2x%2.2x`
BGRA hex).

## Laying out a string

`string_layout_lines` (`0x5cdc20`) builds the whole `CGxString` by iterating lines. It bails if already built
(`+0x9c ≠ 0`) or the text is empty.

```
lineMetric = ScreenToPixelHeight(−clamped_size)
wrapPx     = ftol(480 · wrapWidth + 0.99999)          ; non-rotated
lineStep   = (float)wrapPx / lineMetric
penYseed   = ScreenToPixelHeight(clamped_size) + wrapPx
```

Per line it stops when the pen-Y exceeds the box bottom (`+0x40`), advances the pen by
`clamped_size + lineStep`, computes the line's justify X-offset, calls the line measurer (`0x5c7260`) and the
glyph-quad builder (`0x5ccbe0`), appends any closed hyperlink span, and finally calls the anchor snap. When a
selection is active (`flags & 0x20`) it repaints the highlight (`0x5cd4d0`).

`anchor_justify_snap` (`0x5cdf70`) copies the normalized anchor (`+0x20..+0x28` → `+0x70..+0x78`), applies the
justify offsets, then pixel-snaps:

```
blockHeight h = (numLines · clamped_size) + (480 / linesPx) · (numLines − 1)
anchor.x = round(640 · anchor.x)                      ; roundf
anchor.y = round(480 · anchor.y)
anchor.z = 0
```

Layout gotcha (faithful to the bytes): the justify fields cross. The selector at `+0x54` drives the
**horizontal** offset (`==2` → add `pos_x`; `==1` → add `pos_x · 0.5`), and the selector at `+0x50` drives the
**vertical** placement (`==0` → use `pos_y`; `==1` → add `(pos_y − h) · 0.5`; else add `h`). Implement by
offset and behavior, not by the "h"/"v" names.

The font's render-time metrics are derived once at `Initialize` by `0x5ca030`:

```
s            = max(2.0/480, screenHeightFraction)
font[0x24c]  = min(32, ftol(ScreenToPixelHeight(s)))                  ; rendered pixel size
font[0x17c]  = ftol(renderH · face.s16[0x46] / (|face.s16[0x48]| + face.s16[0x46]) + 0.5)
font[0x178]  = renderH + (flags & 8 ? 4 : flags & 1 ? 2 : 0)         ; atlas cell size
font[0x188]  = (float)face[0x58].u16[0xc] / (float)face.u16[0x44]    ; advance-adjust
```

## Building the glyph quads

`glyph_quad_build` (`0x5ccbe0`) is the core geometry. For each glyph in the (already line-broken) run it
appends a 4-vertex quad to the glyph's atlas-page vertex list and a colour quad to the page colour list. Each
vertex is `0x14` bytes: `{x@+0, y@+4, z@+8, u@+0xc, v@+0x10}`.

Per-call setup:

```
scale      = ScreenToPixelHeight(clamped_size) / font[0x24c]
lineHeight = ScreenToPixelHeight(...) + (flags & 8 ? 4.0 : flags & 1 ? 2.0 : 0)
```

Per-glyph pen advance and the prototype vertex:

```
penAdvance = kern · scale                              ; ComputeStep / ComputeStepFixedWidth when flags&0x10
penAdvance = round(penAdvance)                         ; non-shadow only (+0.5, ftol)
pen       += penAdvance
proto.x    = pen + GlyphWidth(glyph)                   ; left placement
cellW      = (float)desc[0x48] · scale                 ; non-shadow floored
proto.y    = originY − vpad + desc[0x58] · scale       ; vpad = 2.0 outline / 1.0 shadow
```

The four corners (v1 holds the proto base) span `cellW` horizontally and `lineHeight` vertically (the shadow
path uses `sizeMetric` rather than `lineHeight` for the vertical extent):

```
v0 = (base.x,         base.y)
v1 = (base.x,         lineHeight + base.y)
v2 = (cellW + base.x, base.y)
v3 = (cellW + base.x, lineHeight + base.y)
```

The atlas UV rect is copied from the descriptor — left/right column = `u_left`/`u_right` (`desc[0x64]` /
`desc[0x6c]`), top/bottom row = `v_top`/`v_bottom` (`desc[0x68]` / `desc[0x60]`) — matched to the x/y corners.
When the shadow flag is set, the UVs are nudged by `±1/512` (`v_bottom`/`u_left += 1/512`,
`v_top`/`u_right += −1/512`). Each vertex's colour is the string colour word (`+0x2c`), and the glyph's atlas
page bit is OR'd into the caller's page mask.

The index buffer (`0x5c9330`, 512 quads) emits six indices per quad — two triangles — as
`{c, c+2, c+1, c+2, c+3, c+1}` with `c += 4` per quad.

The per-page submit (`0x5ce0c0`) hands a page's vertex list, colour, gradient, the pixel-snapped anchor
(`+0x70`), and shadow flag to the device draw (`0x5c8710`). The per-frame flush (`0x5c2ce0`) walks the global
string render list (`0xc2b99c`) drawing each.

### Selection highlight

`selection_highlight_fade` (`0x5cd4d0`) fills the selection rectangle with a fading alpha. The base alpha is
`+0x2f`; the fade step is `ftol(startCol / endCol)`, and every odd in-selection cell decrements the alpha by
that step (clamped at 0).

## The glyph atlas

Glyphs are rasterized once and packed into a 256×256 ARGB4444 texture. The atlas is partitioned into bucket
rows by cell height:

```
rows           [0x5cf360] = 256 / cellsize             ; cellsize = font[0x178]
per-row free width         = 256                        ; initialized at row create
cellTexelAddr  [0x5cf5f0] = 0xc2ba48 + 2·(page · cellsize · 256 + x0)
```

`0xc2ba48` is the shared staging buffer (256 texels per row, 2 bytes per texel). Glyph placement scans the
bucket rows for a free slot; when the atlas is full the glyph-LRU front is evicted and the freed cell reused.

### Blit variants

`0x5cf310` dispatches to one of four blit routines by the low two flag bits of `font[0x180]`. Each writes
ARGB4444 texels:

| Selector | Routine | Pixel format |
|---|---|---|
| 0 | `0x5cf220` (plain) | `texel = (src8 << 8) \| 0x0fff` → A = `src>>4`, RGB = white |
| 1 | `0x5cea30` (AA + outline) | AA coverage → `fill = (glyphByte·0xff)>>0xc`; outline alpha from the 8-neighbour count via LUT `0x80a8ec` |
| 2 | `0x5ce910` (mono) | 1-bit glyph bitmap → `0xffff` (set) or `0` |
| 3 | `0x5ce440` (outline) | dilation map → `1`→`0xffff`, `2`→`0x7000`, `4`→`0xf000`, else `0` |

The AA coverage LUT `0x80a8ec` maps neighbour-count 0..9 to a 4-bit alpha nibble:
`{0, 1, 1, 3, 5, 7, 9, B, D, F}`. The plain variant pads the cell: `top blank rows`, the glyph rows, then
`font[0x178] − topBlank − glyphRows` trailing rows, each row stepping `0x200` bytes (256 texels × 2).

### Rasterization

`rasterize_glyph` (`0x5d1120`) drives FreeType (load + render at `0x5d1300`, with flags `0x208a` unhinted /
`0x20c2` hinted), copies the bitmap out, and converts the FreeType 26.6 fixed-point metrics to float pixels:

```
advance: out[5] = (float)(advance26.6 >> 6) + 1.0      ; round-toward-zero with negative bias, then +1.0
bearing: out[6] = (float)bearing26.6 · (1/64)          ; 0x80a92c, unrounded
width:   out[4] = bitmapWidth + advanceAdjust
```

For an empty bitmap it synthesizes a blank cell: `width = (code+3) >> 2`, pitch `= (width+7) & ~7` when
hinted. `glyph_vplace` (`0x5d1360`) then computes the glyph's vertical placement and clip against the face
ascender `[[face+0x54]+0x68]`.

### The font face

`FontFaceGetOrCreate` (`0x5d0060`) is the typeface factory behind `CGxFont+0x70`. It caches faces by name in a
`TSHashTable` (mask global `0xc4baa0`); on a miss it opens the font file, builds the FreeType face from memory,
and allocates an `HFACE` (size `0x2c`, vtable `0x80a91c`) holding the file handle, FreeType face, and a
duplicated name. FreeType itself (`0x7cdb40` load, `0x7cdde0`/`0x7ce8d0` face-create, `0x7cec20` render,
`0x7ce2f0` done-face) is the statically-linked rasterization engine — the boundary of the font's own math.
