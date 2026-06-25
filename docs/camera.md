# Camera — the view transform

The `CCamera` object is the client's camera/view → device-transform layer. It holds the camera's
eye/target/orientation as a set of lazily-evaluated properties, and once per use it assembles the
**projection**, **view (lookAt)**, **orthographic**, and **frustum** matrices and pushes them onto the
graphics device's matrix stack. It owns the camera-specific *assembly* arithmetic — viewport aspect,
the lookAt direction/up basis, frustum-corner interpolation, ortho extents/depth, the roll sin/cos,
and ray unprojection — and delegates the matrix kernels themselves to the math layer
([Rendering math](rendering-math.md)) and the matrix-stack push to the device
([Graphics device](graphics-device.md)). Camera *control* (the `cam_*` script bindings) lives in the
[UI](ui.md); this object is purely the view transform.

The TU occupies `[0x7ac640, 0x7ae010)`. `CCamera` is a `CBaseManaged` subclass managed through an
`HCAMERA` handle.

## The `CCamera` state struct

The object is `0x170` bytes (368). It begins with the `CBaseManaged` header and a small array, then
embeds nine **managed-property sub-objects** that each cache a value and a producer callback:

| Offset | Field |
|---|---|
| `+0x00` | vptr (final `CCamera` vtable `0x81d7c0`) |
| `+0x04` | base field |
| `+0x08` / `+0x0c` | intrusive managed-property list head (self-linked) |
| `+0x10` / `+0x14` | property array count / pointer (9 entries) |

Each managed-property sub-object has this shape:

| Offset (within sub-object) | Field |
|---|---|
| `+0x00` | vptr |
| `+0x04` / `+0x08` | list next / prev |
| `+0x0d` | dirty byte — bit `0x8` = changed, bit `0x4` = eval-active |
| `+0x10` / `+0x14` / `+0x18` | producer-callback triplet (recomputes the value lazily) |
| `+0x1c` | value.x |
| `+0x20` | value.y, or **cos** for an angle property |
| `+0x24` | value.z, or **sin** for an angle property |

There are three kinds, distinguished by vtable: 2 vec3-verify properties (`0x81d7d8`), 3 vec3
properties (`0x81d7c8`) at `+0x70`/`+0x90`/`+0xb0`, and 4 angle properties (`0x81d7fc`) at
`+0xd0`/`+0xf8`/`+0x120`/`+0x148`.

### Absolute fields the transforms read

The matrix-assembly functions read the cached value fields by absolute `this`-offset:

| Offset | Meaning |
|---|---|
| `+0x3c` / `+0x40` / `+0x44` | eye x / y / z |
| `+0x64` / `+0x68` / `+0x6c` | target x / y / z |
| `+0x8c` | dist |
| `+0xac` | far plane |
| `+0xcc` | near plane |
| `+0xf0` / `+0xf4` | projection factors |
| `+0x114` | fov |
| `+0x140` | roll-z / `+0x144` up-scale |
| `+0x168` sin(roll) / `+0x16c` cos(roll) | produced by the angle properties |

Six vtables sit in `.rdata`: `0x81d7c0` (CCamera), `0x81d7c8` (vec3 prop), `0x81d7d8` (vec3-verify
prop), `0x81d7e8` (array), `0x81d7f0` (register-entry base), `0x81d7fc` (angle prop).

## Roll-angle 2π reduction (`0x7abbe0`)

Before the roll angle is fed to `fsincos`, it is reduced into the range `(−2π, 2π)` by a small leaf
(`__cdecl`, one `f32` argument, returns `st0`):

```
q = (1/2π) · x
n = trunc(q)            ; __ftol (0x40a2b0): round toward zero, double → i64, low dword
m = n · 2π
r = (x < 0) ? m − 2π : m
return x − r
```

The `x < 0` leg subtracts an extra `2π` so negative inputs keep a negative residue. The sign test is a
parity branch on the FPU compare flags; `x == 0`, `x > 0`, and NaN all take the non-subtracting leg,
so NaN propagates to a NaN result.

Constants (the two reciprocal-related ones are runtime-initialized — they are BSS-zero at load and
written by the binary's own static initializers `0x7adf30`/`0x7adf50`):

| Address | Value | Bits |
|---|---|---|
| `[0xcf5734]` | 2π | `0x40c90fdb` (computed as `f32(π) + f32(π)`) |
| `[0xcf5730]` | 1/2π | `0x3e22f983` |

The `__ftol` chop runs the binary's own integer-conversion bytes (no transcendental, no host route);
model it as a round-toward-zero `as i32` where NaN / `|q| ≥ 2^31` saturate to `i32::MIN`.

## Angle properties — cos/sin (`0x7acbc0` / `0x7acc40`)

An angle property stores a reduced angle and its cosine/sine. `0x7acbc0` is the angle-property
constructor; `0x7acc40` is the property's set slot. Both compute the same thing:

```
a = camera_normalize_2pi(input)
if a != stored_angle:  stored_angle = a;  mark dirty (|= 0x8)
(cos, sin) = fsincos(a)   ; single x87 op: cos → sub-object +0x20, sin → +0x24
```

`fsincos` is the x87 transcendental — both outputs come from one FPU stack operation — so an
independent reimplementation will match only to within the libm structural difference, not bit-for-bit.

## Projection + view (lookAt) (`0x7ac640`)

`camera_view_lookat` (`__thiscall`) takes a four-float viewport `[x0, y0, x1, y1]` and builds both the
projection and the view matrix, pushing each to the device.

**Projection.** The aspect ratio is computed directly from the viewport:

```
aspect = (vp[3] − vp[1]) / (vp[2] − vp[0])
```

then the perspective kernel `0x5c3cc0(fov = +0x114, aspect, near = +0xcc, far = +0xac)` builds the
matrix, pushed via device `0x58b040`.

**View (lookAt).** The direction is `target − eye` and the up basis is built from the roll cos/sin and
the up-scale:

```
dir.x = +0x64 − +0x3c      ; target.x − eye.x
dir.y = +0x68 − +0x40
dir.z = +0x6c − +0x44

up.x  =  cos(roll) · upscale   ; +0x16c · +0x144
up.y  = −(sin(roll) · upscale) ; −(+0x168 · +0x144)
up.z  =  roll-z                ; +0x140
```

These feed the lookAt kernel `0x5c3e70` (against an identity matrix fill of `1.0`/`0`), pushed via
device `0x58b050`.

## Orthographic (`0x7ad7f0`)

`camera_ortho` (`__fastcall`) takes a rect `param_1[4]` and a translation `param_2`. It centers the
rect on its midpoints, builds an ortho matrix with a fixed `±500` depth range, applies a Z-flip, and
applies the view translation:

```
my = (param_1[1] + param_1[3]) · 0.5
mx = (param_1[0] + param_1[2]) · 0.5

ortho(0x5c3d90):  left/right/top/bottom = the rect re-centered on (mx, my),
                  near = −500.0, far = +500.0

scale (1, 1, −1)            ; flip-Z, via 0x7bdca0  (−1.0 = 0xbf800000)
if device 0x589ca0() == 1:  recenter via 0x58a250()
push view (0x58b040)
translate (param_2[0] − mx, param_2[1] − my, 0)   ; via 0x7bdc40
push (0x58b050)
```

## Frustum split (`0x7ad2c0`)

`camera_frustum_split` (`__fastcall`) builds a sub-frustum between two normalized split fractions
`p3`, `p4`. Both are validated to lie in `[0, 1)` (`0.0 ≤ p` and `p < 1.0`); an out-of-range argument
fails the validation gate (the parity-flag branch described below).

It zeroes 8 corner slots, reads the current matrices from the device (`0x58b0b0`, `0x58b090`), gets the
8 frustum corners from the kernel `0x5c43b0`, then interpolates along the frustum edges. Each
interpolation is `base + dir · t` with `t` = `p3` or `p4`, written through the C3Vector store helper
`0x4549a0` (a pure three-dword store). The two output points are the depth-split near/far centers,
returned in `param_1` and `param_2`.

The validate gates compile to a parity-flag branch (`test ah, 0x41; jnp`) reading the FPU compare
flags — a detail to reproduce exactly if porting the comparison.

## Ray unprojection (`0x7ac810` / `0x7ac8a0`)

Two `__fastcall` helpers compute a world point at distance `dist` (`+0x8c`) along the camera ray, using
the roll sin/cos and the two projection factors (`+0xf0`/`+0xf4`).

`camera_ray_neg` (`0x7ac810`) goes **backward from the target** (`+0x64`/`+0x68`/`+0x6c`):

```
out.x = −(sin · projf.x) · dist + target.x
out.y = −(cos · projf.x) · dist + target.y
out.z = −(projf.y)       · dist + target.z
```

`camera_ray_pos` (`0x7ac8a0`) goes **forward from the eye** (`+0x3c`/`+0x40`/`+0x44`) — the same
formula without the leading negation:

```
out.x = (sin · projf.x) · dist + eye.x
out.y = (cos · projf.x) · dist + eye.y
out.z = (projf.y)       · dist + eye.z
```

## The clamp-01 global (`0x7adfb0` / `0x7ae000`)

`camera_clamp01` (`__cdecl void(float)`) clamps its argument to `[0, 1]` and stores it in the global
`[0x87d5fc]`:

```
if      x < 0.0:  [0x87d5fc] = 0.0
else if x < 1.0:  [0x87d5fc] = x
else:             [0x87d5fc] = 1.0
```

Thresholds are `0.0` and `1.0`; the high store is `1.0`
(`0x3f800000`). The boundary behaviour at exactly `0` and `1` is load-bearing. The getter `0x7ae000`
simply returns `[0x87d5fc]`.

## Where the matrices go, and who drives the camera

The matrix kernels and device push are delegated:

- **Matrix kernels** ([Rendering math](rendering-math.md)): perspective `0x5c3cc0`, ortho `0x5c3d90`,
  lookAt `0x5c3e70`, frustum-corners `0x5c43b0`.
- **Device matrix stack** ([Graphics device](graphics-device.md)): push `0x58b040`/`0x58b050`, get
  `0x58b090`/`0x58b0b0`, query `0x589ca0`, recenter `0x58a250`.
- **Vector/matrix helpers** ([Math primitives](math-primitives.md)): C3Vector store `0x4549a0`,
  matrix scale `0x7bdca0`, matrix translate `0x7bdc40`.

None of the transform entry points is vtable-registered; they are called directly from UI code:

- `camera_frustum_split` (`0x7ad2c0`) ← `0x48144d` (WorldFrame).
- `camera_ortho` (`0x7ad7f0`) ← `0x4ec950` (unproject/render), `0x76d448` (model preview), `0x787457`
  (scroll frame), `0x7657f1` (strata).
- `camera_view_lookat` (`0x7ac640`) ← `0x7ada57` (in-TU, via the guarded wrapper `0x7ada40`).
