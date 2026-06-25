# Math & determinism primitives

The client funnels its determinism-sensitive math through a handful of small, self-contained
translation units that every other subsystem delegates to. This chapter collects five of them:
the `C*Matrix`/`C*Vector` linear-algebra library, a software scalar log2/exp2 pair, the `CRandom`
deterministic PRNG and its distribution generators, the curve/spline evaluators, and the 2D
screen-coordinate scale singleton. They matter because the rest of the client — renderer, terrain,
models, collision, lighting, animation, UI — produces its exact output by calling into these, so
their precise arithmetic and rounding behavior is the substrate the higher-level systems inherit.

A note that applies to every section below: where a routine calls an 80-bit x87 transcendental
(`fsincos`, `fpatan`, `asin`, `fcos`), that result is **not bit-reproducible on a non-x87 target**.
On Apple Silicon there is no x87, so the host `libm` is the realizable behavior; the x87↔libm gap is
sub-ULP and unobservable. Everything else here is plain f32/f64 arithmetic with canonical bytes.

## Linear algebra — `C44Matrix` / `C33Matrix` / `C3Vector` / `C4Vector` (cmath)

The shared matrix/vector library. Pure 32-bit-float math with its own `fsqrt`/`fsincos`/`fpatan`
(no CRT `sqrt`). It is the leaf that the renderer's `proj` multiply (`0x59fe20` delegates to the 4×4
multiply `0x7bc6a0`) and later terrain/models/collision/lighting/animation all bottom out at.

In `.text` the library is emitted as a contiguous run **interleaved with foreign code** that shares
the address neighborhood: a color/HSV/gamma TU, ~15 linker accessor-thunk stubs of the form
`jmp pad → mov g,eax; mov eax,g; ret`, and a `std::vector<matrix>` container with its allocator. Only
the linear-algebra arithmetic is part of this library; the interleaved code is unrelated.

### Storage layout and calling conventions

- **Column-major** 4×4 at struct offsets `0x00..0x3c`; 3×3 at `0x00..0x20`.
- `C3Vector` = `{+0:x, +4:y, +8:z}`.
- Most ops are `__fastcall`/thiscall: `ecx` = this/dst, `edx`/stack = operands. Some are naked leaves
  with no `ebp` frame (e.g. the dot leaf `0x7bc8c0`).
- The `C33Matrix(9 floats)` constructor lives far from the rest at `0x5f8d20`; the 4×4 load-identity
  is right after it at `0x5f8d60`.

Calling-convention families:

| Family | `ecx` | `edx` | stack |
|---|---|---|---|
| elementwise/binary | dst | a | `[ebp+8]` = b-ptr or by-value scalar |
| by-value-return const method | src | — | `[ebp+8]` = dst-return ptr |
| in-place | matrix (read back) | — | — |
| compare | a | b | (bool in `eax`) |
| angle→matrix | dst | — | `[ebp+8]` = angle by value |
| determinant | m (by-pointer form) | — | 9 floats on stack (by-value form); result in st0 |

### The operations

**4×4 core:** add `0x7bc290` · sub `0x7bc500` · negate `0x7bc430` · ×scalar `0x7bc8e0` · ÷scalar
`0x7bccd0` · **× 4×4 multiply `0x7bc6a0`** · transpose `0x7bcef0` · `operator==` `0x7bc0b0` ·
**determinant `0x7bcf90`** (Laplace expansion via four 3×3 dets `0x7bc040`) · adjugate `0x7bd060` ·
**inverse `0x7bd390`** (cofactors ÷ det) · transform vec3-as-point `0x7bca80` · transform vec4
`0x7bcb40`.

**3×3 core:** multiply `0x7bdfc0` (the workhorse vec3-by-3×3) · determinant `0x7be0e0` · adjugate
`0x7be120` · inverse `0x7be320`/`0x7be3f0` · scales `0x7be610`/`0x7be670`/`0x7be6d0`.

**Vector:** the scalar-triple / 3×3 determinant `0x7bc040` (heavily reused) · **vec3 normalize
`0x7be490`** (len² → `fsqrt` → reciprocal → scale, bool-gated) · **plane from 3 points `0x7bf860`**
(cross + normalize + d; copies the three points by value) · dominant-axis index `0x7bf680`/`0x7bf700`.

**Rotation builders:** axis-angle → 4×4 `0x7bdb00` and the → 3×3 `0x7be490`-family (normalize +
`fsincos`, Rodrigues) · Z-rotation `0x7be5b0` · quaternion → matrix `0x7bddb0`/`0x7be770`.

**Euler family (12 functions):** 6 matrix → Euler extractors `0x7be930..0x7bec80` (one per rotation
order, with `fpatan` + `asin` and gimbal-lock branches) · 6 Euler → 3×3 builders `0x7bed30..0x7bf4b0`
(three `fsincos` each).

### Rounding model

All arithmetic runs in f64 (x87 PC_53) and narrows to f32 (`as f32`) **at each f32 store**. This
double-round is load-bearing: which intermediates stay in f64 registers versus round through f32
locals **varies per function** — there is no single split to copy across siblings.

Two specific rounding gotchas:

- The 3×3 determinant `0x7bc040` ends `fsubrp; ret`, so its result **stays in st0 at full 53-bit** —
  it does not narrow to f32. A caller that `call`s it and then `fmul`s consumes the un-rounded value.
  Its caller's accumulator does round per term when it is stored/reloaded through an f32 stack slot.
- The 4×4 multiply `0x7bc6a0` is 15 inline dot products plus the `dot_leaf` `0x7bc8c0` for `dst[0]`.

### Quaternion → rotation matrix

With `(x,y,z,w) = q[0..3]`, the column-major matrix `M` is:

```
M[0] = 1 − 2(y² + z²)   M[3] = 2xy − 2zw       M[6] = 2xz + 2yw
M[1] = 2xy + 2zw        M[4] = 1 − 2(x² + z²)  M[7] = 2yz − 2xw
M[2] = 2xz − 2yw        M[5] = 2xw + 2yz       M[8] = 1 − 2(x² + y²)
```

Rounding split: the off-diagonal products `2xw/2yw/2zw/2x²/2xy/2xz` round through f32 locals; `2yz`,
`2y²`, `2z²` stay in f64 registers; `2z` round-trips through f32. The Euler builders reuse this layout.
Note `dst` can be in/out: `0x7be770` left-multiplies the existing dst (`dst = M·dst_in`).

### Euler conversions

The 6 extractors take a 3×3 plus three `f32*` outputs plus an `al` gimbal-lock bool. The middle angle
is the `asin` of (depending on order) `m6`/`−m3`/`−m7`/`m1`/`m5`/`−m2`; the outer pair is two `atan2`.
The gimbal-locked branch uses the verbatim π/2 constant `0x3fc90fdb`. The 6 builders construct one
single-axis rotation per angle and store the `mat3_mul` product **transposed** via the `C33Matrix`
constructor `0x5f8d20`.

### `asin` (`0x7405a0`)

The binary's `asin` is implemented as `atan2(x, √((1+x)(1−x)))` for `|x|<1` — at `0x7405d8`:
`fld1; fadd; fld1; fsub; fmulp; fsqrt; fpatan`. Faithfully:

```
asin(x) = atan2(x, ((1 + x)·(1 − x)).sqrt())
```

The axis-angle builders share a Rodrigues core: conditional `fsqrt`-normalize, then the column-major
`R = cos·I + sin·[a]ₓ + (1 − cos)·a⊗a`.

### Adjacent color codec

The spatially-neighboring color TU packs a float color to unorm8 with a magic-number trick:
`c = x·255 + 512`, then `(bits ≫ 14) & 0xff`, in BGRA order; the unpack is `fild × 0x3b808081` (and a
byte→float unpack `fildl; fmuls [0x8026c8]` with `[0x8026c8] = 1/255`). This is not part of the
linear-algebra library proper but lives in the same address run.

### Scalar-interpolation leaves

Two shared scalar interpolators sit in a far TU and are included in this section:

- **lerp `0x5b7b90`** — `fld [b]; fsub [a]; fmul [t]; fadd [a]` ⇒ `a + (b − a)·t`. Returns the
  un-narrowed f64 in st0 (callers `fstp` to their own width).
- **cosine-smoothstep `0x5b7bb0`** — `a + (b − a)·((1 − cos(π·t))·0.5)`, with π the f32 constant at
  `0x80a714` and `cos` via the host transcendental.

See also [Rendering math](rendering-math.md), [Camera](camera.md), [Animation](animation.md),
[Models](models.md), and [Collision](collision.md), which consume these primitives.

## Software scalar fast-math — log2 / exp2 (fastmath)

A tiny TU of **hand-written software** scalar transcendentals (exponent extraction + table + Horner
polynomial), owned separately from the CRT `pow 0x73f940` (which is `fyl2x`/`f2xm1` hardware). Because
a software polynomial has canonical bytes, these are bit-reproducible on any target — the libm carve-out
above applies only to hardware instructions, not to these.

| Address | Function | Mechanism |
|---|---|---|
| `0x454c80` | `log2` | exponent extract + 64-entry reciprocal LUT `0x802be0` (`64/(64+i)`) + degree-6 Horner with `(1/ln2)/k` coefficients + a correction addend table `0x802de0` (`−log2(LUT[i])`). Returns `−inf` for `x ≤ 1e-307`. |
| `0x454d40` | `exp2` | the software `2^x` sibling: a second table `0x8029e0`, a polynomial `0x803028`–`0x803058`, and `__ftol` integer-exponent assembly — **no `f2xm1`**. |
| `0x40a2b0` | `__ftol` | the f64→i64 truncate that `exp2` calls (CRT). |

The `pow`/`exp` wrappers `0x456620`/`0x456640`/`0x456660`/`0x4566a0` are CRT-pow consumers that compose
this `log2` with the hardware CRT `pow`; they are not part of this library. The random library's
exponential and Gaussian generators call this `log2` for their `−ln(u)` leg.

## Deterministic PRNG — `CRandom` / `CRndSeed` (random)

The client's deterministic random-number service: a PRNG plus distribution generators that the whole
engine delegates to for stochastic effects (particle spawn velocities, weather drift, ground-doodad
placement, gradient effects, UI). A self-contained TU at `[0x44ea80, 0x4532b3)` over the shared
`.rdata` random table `0x802700`, with a Box-Muller spare-deviate cache in `.data` at
`0xb05d74`/`0xb05d78`. It owns the exact sequence rather than delegating to a host RNG, which is what
makes its output cross-platform-identical.

### Core and seed

- **`CRandom::uint32` `0x4531e0`** — the core advance: a 2-dword state (`[ecx]` / `[ecx+4]`),
  rotate-XOR over the shared table `0x802700`. This has 91 engine-wide callers.
- **`CRndSeed::SetSeed` `0x44ea80`** — string-hash-seeded via an `imul 0x7a2d` roll; 21 callers.

### Distribution generators

| Generator | Address | Mechanism |
|---|---|---|
| uniform | `0x44f270` | mantissa trick to `[1,2)`, then remap |
| exponential | `0x44f580` / `0x44f680` | `1−u → ln` (via `log2` at `0x454c80`) → scale |
| Gaussian (Box-Muller) | f32 `0x44f9b0` / f64 `0x44fa70` | `fsincos(2π·u)·√(−2 ln u)`, caching the spare deviate in `0xb05d74`/`0xb05d78` |
| Gaussian affine `σ·N+μ` | `0x44fb30` / `0x44fb50` | scale + shift of the above |
| random unit vector 2D | `0x44fd00` | `(cos, sin)` of `2π·hash` |
| random unit vector 3D | `0x44fe20` | z-hash + `√(1 − z²)` ring around the 2D form |

Plus Fisher-Yates shuffles for byte/word/dword/float at `0x450100`+ with batch array-fill wrappers, and
a 5-function gradient-noise stack. The `fsincos`/`ln` transcendentals are subject to the host-libm note
above.

### Lua's `math.random` is a different sequence

Stock **Lua 5.0's `math.random` delegates to C `rand()`**, which here is the **MSVC-CRT `rand` LCG** at
`0x7400e5`:

```
seed = _holdrand
seed = seed·214013 + 2531011      # the imul/add at 0x7400ed
_holdrand = seed
return (seed >> 16) & 0x7fff      # RAND_MAX = 0x7fff
```

`_holdrand` is a per-thread `int`, default 1. This is a separate, cross-platform-determinism hazard:
glibc/musl/MSVC `rand` all differ, so a faithful client must own *this* LCG rather than call the
platform `rand`. The Lua `math.random(m,n)` scaling of `rand()` into `[m,n]` is a separate Lua-stdlib
concern; the LCG above is the sequence generator. See [Spells](spells.md), [Net](net.md), and
[Models](models.md) for consumers.

## Curve / spline evaluation (curvemath)

A second math TU, distinct from the linear-algebra library above: higher-order math over stateful
container objects (control-point arrays), covering spline-basis curve evaluation, arc-length quadrature,
Frenet/orientation-frame construction, software reciprocal/rsqrt batch builders, and cubic interpolation.
It also co-locates the foundational `C3Vector` leaves: the store-xyz constructor `0x4549a0` (referenced
~400× binary-wide), ÷scalar `0x4549c0`, len² `0x4549f0`. The load-bearing consumer is path-following
(MONSTER_MOVE, transports, taxi/flight paths, MoveSplineToPoint).

### Scope

- **Curve/spline:** Catmull-Rom spline-basis evaluators `0x453580`/`0x453640`/`0x4536d0`
  (`Σ wᵢ·Pᵢ` over 4 control points × Horner basis-weight `0x453620`); arc-length quadrature `0x453760`;
  segment locator `0x453840`/`0x4538c0`; knot-sum reductions `0x453300`/`0x453920`; Frenet/orientation-
  frame builders `0x453b90`/`0x454580`; linear/spline position+length evaluators `0x4541b0`/`0x454320`.
- **Software transcendentals (polynomials):** atan `0x454ab0`, ln `0x454ba0`, log-base
  `0x454b30`/`0x454c10`.
- **Newton-Raphson batch builders:** reciprocal `0x454fd0` (seed `0x7fde61ee`, `x·(2 − a·x)`, 4
  iterations); sqrt `0x455260`; inv-sqrt `0x4555f0` (seed `0x5fe732f7`, `y·(1.5 − 0.5·a·y²)`); with
  integer magic-seed helpers `0x456330`/`0x456350`/`0x4563a0`/`0x4563b0`.
- **Cubic Catmull-Rom interpolators:** `0x455970` (f64) / `0x455a50` (f32).
- **C3Vector leaves:** len² `0x454980`/`0x4549f0`, ÷scalar `0x4549c0`, normalize
  `0x456280`/`0x4562b0`/`0x4562f0`.
- **Geometry:** ray-sphere `0x454e10`, quadratic solvers `0x454ea0`/`0x454f40`, integer-sqrt `0x454a10`.
- **Order-statistics (integer selection networks):** Min/Median/Max 3/5/9-way `0x455cc0`–`0x456130`.
- **Interpolation/scalar:** floor-fract `0x4563e0`–`0x4564c0`; step/ramp/smoothstep
  `0x456510`/`0x456570`/`0x4565c0`; window `0x456540`; pow/nth-root `0x456640`/`0x456660`; crossfade
  `0x4566a0`; sinc `0x456700`–`0x456760`.

It delegates `log2`/`exp2` to the fastmath library (`0x454c80`/`0x454d40`) and `pow` to CRT `0x73f940`.

### The curve container (`C3Spline`-class `this`)

| Offset | Field |
|---|---|
| `+0x00` | vtable (basis-method slots) |
| `+0x04` | cached arc-length / scale (f32) |
| `+0x0c` | control-point count (stored ×3) |
| `+0x10` | control-point array base, `C3Vector[]` (stride `0xc`, indexed `[seg*4 + i]`) |
| `+0x18` / `+0x1c` | capacity / segment count |
| `+0x20` | per-segment arc-length cache pointer (`f32[]`) |
| `+0x24` | alloc quantum (power-of-2, capped at `0x40`) |
| `+0x28` | mode flag (`0` = linear fast-path, `≠0` = spline-delegate) |

`C3Vector = {+0:x, +4:y, +8:z}`. The Frenet builders' output frame is `[0x0]` = T, `[0x10]` = N,
`[0x20]` = B.

### The spline-follow recipe

The inbound `SMSG_MONSTER_MOVE` creature follow (collision `0x7c5490`) reaches the evaluators through
three `__thiscall` clamp + virtual-dispatch wrappers on the curve object: `0x453390` (point-at-t),
`0x453480` (orientation/facing sample), `0x453360` (length-cache rebuild). These three own **zero**
floating-point arithmetic — they only load / compare-vs-`0.0`/`1.0` / store. They do endpoint
clamp-by-copy (t ≤ 0 → first control point, t ≥ 1 → last) plus a mode-keyed dispatch to the curve's
vtable.

The creature curve object carries vtable `0x7ffd84` (constructor `0x630c76`:
`mov [esi+0x2c], 0x7ffd84`; the curve is the spline's sub-object at `+0x2c` of `MovementInfo+0xa4`).
The slots and the floating-point computation leaves they reach:

- **point-at-t** (slot `+0x08` `0x454420` mode 1 / `+0x0c` `0x454460` mode 0) → segment locator
  `0x453840` (cumulative-length param) or `0x4538c0` (uniform param) → **`0x4541b0` `linear_pos_diff`**
  (the `P = A + frac·(B − A)` segment lerp). The curved-family blend is `0x453580` plus Horner basis
  `0x453620`.
- **arc-length** → per-segment chord **`0x454320` `seg_length_diff`** (`√(Δx² + Δy² + Δz²)`), summed by
  `0x453920`/`0x453300` (knot-sum). Distance↔t inversion is `0x453840`. The rebuild/sum slots
  `0x4543e0`/`0x4543d0` are loops over these.
- **orientation/facing** (slot `+0x18` `0x454580`) → the frame builder `0x454580`; the caller then
  applies its own `fpatan` for the heading.

### Building the control-point array

The evaluators assume a control-point array. The inbound path populates it via the collision
move-builder `0x6018f0` (through `0x6187a0`/`0x7c6a50`/`0x453360`). For a parsed path with `K_real`
real points in order (`[start, …intermediates…, endpoint]`), the array is **phantom-framed by
duplication**:

```
P[0]           = start            # leading phantom = dup(first real)   @0x601d30
P[1 .. K_real] = the path points  # forward order, start … endpoint
P[K_real+1]    = endpoint          # trailing phantom = dup(last real)  @0x601e66 (count++; P[count]=P[count-1])
count = K_real + 2  (-> [curve+0x0c]) ;  n_seg = count - 3 = K_real - 1
```

`P[0]` and the last slot are guard points the eval never lands on: it samples the 4-point window
`P[seg..seg+3]` for `seg ∈ [0, count-3)`, and the `t ≤ 0 → P[0]` / `t ≥ 1 → P[count-1]` clamps hit the
guards. The per-segment length table `[curve+0x20]` is filled with `seg_length_diff` (straight chord,
when `[curve+0x28] == 0`) and `[curve+0x04]` with their sum (knot-sum); the densifier `0x6021c0`
projects/inserts points to keep the polyline monotone. Timing lives on the movement-info `[unit+0xa4]`
(`+0x1c` start, `+0x24` duration, `+0x28` mode); the follow `0x7c5490` early-outs unless `[unit+0x40] &
3` is armed (set by `0x7c6ae0`). The eval runs at `t = elapsed/duration`.

See [Collision](collision.md) for the move-builder and follow path.

## 2D screen-coordinate scale (screencoord)

A 2D screen-coordinate scale singleton that converts between normalized-device coordinates (NDC) and
device-display coordinates (DDC), driven by the screen aspect ratio `s`. A tiny self-contained TU at
`[0x41ad10, 0x41ae7e)` over four private `.data` globals, referenced by the UI, glue, console, loading
screen, player-name, object-layer, collision, net, and spell subsystems. The renderer, terrain, and
models do **not** use it.

### State — four `.data` globals (default `s = 4/3`)

| Address | Value | Default |
|---|---|---|
| `0x832a40` | `s` (raw aspect ratio) | 1.3333 |
| `0x832a44` | `s/√(s²+1)` = **G44**, the x-axis scale factor | 0.8 |
| `0x832a48` | `1/√(s²+1)` = **G48**, the y-axis scale factor | 0.6 |
| `0x832a4c` | `s·0.75` (0.75 = inverse-4:3 reference aspect) | 1.0 |

`(G44, G48)` is the normalize of `(s, 1)` — the direction cosines for the aspect-axis remap.

### Functions

- **`0x41ad10` `SetScale(s)`** [`ret 4`] — caches all four globals. The `fst [0x832a48]` is a **no-pop
  store**: the subsequent `[0x832a44] = invlen·s` uses the **live f64** `invlen = 1/√(s²+1)`, not the
  f32-rounded value just written to `[0x832a48]`. Keep `invlen` in f64 to match.
- **`0x41ad60` `getScale`** (`fld [0x832a40]`) · **`0x41ad70` `getScale·0.75`** (`fld [0x832a4c]`) —
  pure loads.
- **`0x41ad80` `scaleXY`** (`[ecx] = G44·x` if `ecx≠0`; `[edx] = G48·y` if `edx≠0`) · **`0x41adb0`
  `scaleRect`** (`G44·{x,w}`, `G48·{y,h}`).
- **`0x41ade0` `unscaleXY`** (`x/G44`, `y/G48`) · **`0x41ae10` `unscaleRect`**.
- **`0x41ae40` `unscaleX`** (`v/G44`) · **`0x41ae50` `unscaleY`** (`v/G48`) · **`0x41ae60`
  `scaleX`/`NDCToDDCWidth`** (`G44·v`) · **`0x41ae70` `scaleY`/`NDCToDDCHeight`** (`G48·v`).

`0x41ad60`, `0x41adb0`, and `0x41ae10` have no live callers in 5875 (address-taken but unused).

`SetScale` is driven by glue `0x46a828` and UI `0x48fe2b`, each feeding a computed screen-geometry
ratio. See [UI](ui.md), [Glue](glue.md), and [Console](console.md).
