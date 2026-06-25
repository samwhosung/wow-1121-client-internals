# Rendering math — projection, frustum, ray/picking

The applied 3-D geometry layer of the client: the projection and view matrix builders, frustum
corner/plane/AABB extraction and point-in-frustum culling, viewport project/unproject, ray↔sphere
and ray↔triangle intersection with mesh ray-pickers, a spatial point-hash, projected-decal UV
generation, and a nearest-neighbour image resample. This layer sits *above* the low-level linear
algebra in [Math primitives](math-primitives.md) (the C44Matrix/C33Matrix/C3Vector library), to which
it delegates every elementwise matrix/vector operation. It defines where geometry lands on screen and
what the cursor picks. The whole module is a single MSVC translation unit spanning
`[0x5c3cc0, 0x5c6680)`.

## Conventions

These hold throughout the module and an implementer must match them to reproduce results.

- **C44Matrix** is 16 `f32`, **column-major**, at struct offsets `0x00..0x3c`. For a column vector `v`,
  `r = M·v`, and element `M[row][col]` lives at storage index `col*4 + row`. So **row `r`** of the
  matrix is `(m[r], m[r+4], m[r+8], m[r+12])`.
- **C3Vector** is 3 `f32` at `{0,4,8}`; corner arrays stride `0xc`. **C4Plane / C4Vector** is 4 `f32`
  at `{0,4,8,0xc}`; plane arrays stride `0x10`.
- **The x87 precision idiom.** The FPU runs at `_PC_53` (control word `0x027F`): every
  `fmul`/`fadd`/`fsub`/`fdiv`/`fsqrt` rounds to `f64`, and each `fstp m32` rounds that intermediate to
  `f32`. So the math is `f64` arithmetic with an `f32` round at exactly the stores the binary makes.
  *Which* intermediates get spilled to `f32` versus kept live in the register is load-bearing for the
  exact bits — several functions below have deliberate asymmetries (one component spilled, another kept
  in `f64`) that change the last-bit result, and they are called out where they occur.
- **Input validation** is a hard abort: a bad argument calls `0x64e850` with `push 0x57` (87). The
  abort produces no output, so the value-returning descriptions below model only the valid path.

### Module constants

All read directly from `.rdata`/`.data` (little-endian `f32` unless noted):

| VA | Value | Role |
|---|---|---|
| `0x7ffd74` | `0.0` | zero / validity comparison |
| `0x7ff9d8` | `1.0` | ubiquitous 1.0 (numerator for `1/x`, `1/√`, matrix 1-entries) |
| `0x7ffa24` | `0.5` | perspective half-angle scale; viewport `(x+1)·0.5` |
| `0x80a898` | `π = 3.14159274` | perspective FOV upper bound |
| `0x8038dc` | `−2.0` | perspective z numerator `−2·n·f/(f−n)` |
| `0x801628` | `2.0` | ortho span numerator `2/(r−l)` etc. |
| `0x8029d0` | `0.01` | lookAt degenerate-length ε (len² gate on dir & up) |
| `0x8029d4` | `2.384185791e-07 = 2⁻²²` (`0x34800000`) | affine-vs-perspective detect ε; Möller–Trumbore det ε |
| `0x8090d8` | `2⁻¹⁰ = 0.0009765625` | unit-vector ε (ray dir asserts) |
| `0x801360` | `0.001` | `w ≈ 0` reject ε (unproject) |
| `0x8029b0` | `0.25` | decal centroid weight |
| `0x80308c` | `1.5` | decal mip threshold factor |
| `0x8066d8` | `16.0` | decal quantizer scale |
| `0x802998` | `1/16 = 0.0625` | decal quantizer inverse scale |
| `0x8026c8` | `1/255 ≈ 0.003921569` (`0x3b808081`) | skinned byte-weight normaliser |
| `0x803078` | `0.5` (qword) | decal quantizer pre-add |
| `0x00c2b850` | `+inf` | the picker no-hit sentinel, initialised once at startup (from `0x0080a894 = +inf` via the thunk at `0x5c43a0`) |

## Projection matrices

### Perspective — `0x5c3cc0`

Builds a column-major, OpenGL-style perspective matrix from `(fov, aspect, near, far)`. Validates
`0 < fov < π`, `aspect > 0`, `near < far`. The mapping sends `z ∈ [near, far]` to `z_ndc ∈ [−1, 1]`
and sets `w_clip = +z_eye` (`m[11] = 1`, a left-handed view convention, +z forward).

The half-angle uses a **diagonal-FOV convention**: x is divided by aspect and the angle is scaled by
`1/√(aspect²+1)`. The one transcendental is `fptan` (libm regime).

```
t      = tan( (fov / √(aspect² + 1)) · 0.5 )
tn     = t · near        // computed once, reused by m[0] and m[5]
m[0]   = near / (aspect · tn)
m[5]   = near / tn
m[10]  = (near + far) / (far − near)
m[14]  = (near · far · −2.0) / (far − near)
m[11]  = 1.0
// all other entries 0, including m[15] = 0
```

Precision note: the binary computes `t·near` and reuses it, so `m[0]`/`m[5]` are the **un-cancelled**
`near/(aspect·t·near)` and `near/(t·near)` — not the algebraically reduced `1/(aspect·t)` / `1/t`. The
un-cancelled form is what reproduces the binary's exact result.

The resulting NDC: `x_ndc = x/(aspect·t·z)`, `y_ndc = y/(t·z)`.

### Orthographic — `0x5c3d90`

The standard column-major `glOrtho` from `(left, right, bottom, top, near, far)`. Validates `l ≠ r`,
`b ≠ t`, `n < f`. Affine, with w-row `(0,0,0,1)`.

```
m[0]  = 2/(r − l)      m[12] = −(l + r)/(r − l)
m[5]  = 2/(t − b)      m[13] = −(b + t)/(t − b)
m[10] = 2/(f − n)      m[14] = −(n + f)/(f − n)
m[15] = 1.0
// all other entries 0
```

A thiscall thunk at `0x5c3e50` re-pushes a stored matrix's 6 fields and jumps to `0x5c3d90` — an
inline-reload wrapper in the same TU.

## View matrices (lookAt)

### Variant 1 — `0x5c3e70`

Builds a view matrix from `(eye, target, up)`. Forward `f = target − eye`; the function validates
`len²(f) ≥ 0.01` and `len²(up) ≥ 0.01`, else aborts. It then builds an orthonormal basis:

```
f = normalize(target − eye)
s = normalize(f × up)
u = normalize(s × f)
```

with the standard cross product `a × b = (a.y·b.z − a.z·b.y, a.z·b.x − a.x·b.z, a.x·b.y − a.y·b.x)`.
The basis is stored as **rows** over an identity:

| Row | Stored at | Vector |
|---|---|---|
| 0 | `m[0], m[4], m[8]` | `s` |
| 1 | `m[1], m[5], m[9]` | `u` |
| 2 | `m[2], m[6], m[10]` | `f` (forward = 3rd row) |

The translation column is then composed by the cmath translate (`0x7bdc40`) of `−eye` against the
half-built rotation, giving `final = R · T(−eye)`, i.e. `m[12..14] = R·(−eye)`.

Precision note: the **forward** normalize rounds each component square to `f32` first, then sums
`(fy² + fz²) + fx²`; the `s`/`u` normalizes keep the squares in `f64`. That asymmetry is real and
affects the last bits.

### Variant 2 — `0x5c4100`

Byte-identical to variant 1 except the two cross products are sign-flipped (`s = up × f = −(f × up)`,
`u = f × s`), producing the mirror-handed view basis. This variant is statically linked but has **zero
call/pointer references in build 5875** — unused at runtime, though deterministic.

## Frustum

### 8-corner unproject — `0x5c43b0`

Computes the 8 world-space frustum corners from a view matrix and a projection matrix. It first builds
the inverse view-projection:

```
inv(VP) = inv(proj) · inv(view)
```

(each inverse and the product delegated to cmath). Then it picks the corner set by detecting whether
the projection is affine or perspective:

```
affine  iff  |proj.m15 − 1.0| < 2⁻²²   (proj.m15 @ offset 0x3c)
```

- **Affine/ortho branch:** transform the 8 corners of the ±1 NDC cube `(±1, ±1, ±1, w = 1)` by
  `inv(VP)`, keep `.xyz`.
- **Perspective branch:** recover near/far from the projection matrix, then transform the
  **w-prescaled** clip corners `(±w, ±w, ∓w, w)`:

  ```
  n = −proj.m14 / (proj.m10 + 1.0)     // proj.m10 @ 0x28, proj.m14 @ 0x38
  f = −proj.m14 / (proj.m10 − 1.0)
  // near cap uses w = n (z = −n); far cap uses w = f (z = +f)
  ```

  Pre-scaling by `w` replaces the post-perspective-divide, so the linear inverse lands the exact world
  point.

Corner order (both branches): near cap `(−,−) (−,+) (+,+) (+,−)` in (x,y), then far cap in the same
order.

### Gribb–Hartmann plane extraction — `0x5c4930`

Extracts 6 normalized frustum planes from a clip (view·proj) matrix. With rows
`R_k = (m[k], m[k+4], m[k+8], m[k+12])`:

| Plane | Output offset | Formula | Side |
|---|---|---|---|
| `p0` | `+0x00` | `R0 − R3` | RIGHT |
| `p1` | `+0x10` | `−(R0 + R3)` | LEFT |
| `p2` | `+0x20` | `R1 − R3` | TOP |
| `p3` | `+0x30` | `−(R1 + R3)` | BOTTOM |
| `p4` | `+0x40` | `R2 − R3` | FAR |
| `p5` | `+0x50` | `−(R2 + R3)` | NEAR |

The `−(R + R3)` planes are computed as `(−R) − R3` (an exact `fchs` then `fsub`). Each plane is then
normalized: `(a,b,c,d) *= 1/√(a²+b²+c²)` with the sum-of-squares `((a²+b²)+c²)` in `f64`, the
reciprocal rounded to `f32`, and **all four components** (including `d`) scaled. Convention is
outward-normal: `plane·(p,1) ≤ 0` means inside (matches the point-in-frustum test below).

Statically linked but **zero references in build 5875** — the live frustum class at `0x68xxxx` rolls
its own plane extraction.

### Frustum AABB — `0x5c4c70`

Axis-aligned bounding box of the 8 corners (which it gets by calling `0x5c43b0`). Seeds both min and
max with corner 0, then component-wise: min takes the corner when `corner ≤ min`, max takes it when
`max ≤ corner`, across all 8. Statically linked, **zero references in build 5875**.

### Point-in-frustum — `0x5c54f0`

Tests a point against 6 outward-pointing planes with a slack margin. For each plane, the signed
distance accumulates in the binary's exact x87 order:

```
d = (n.y·p.y + n.x·p.x) + n.z·p.z + n.w
if (d − margin) > 0.0  →  return 0 (outside)
// all 6 within margin  →  return 1 (inside)
```

This is a point / radius-`margin` sphere containment test. Statically linked, **zero references in
build 5875**.

## Viewport project / unproject

Both read the live `CGxDevice` (singleton `0xc0ed38`) through plain accessors — these viewport rect,
viewport scalars, and matrix-stack values are runtime device state, not a closed formula. See
[Graphics device](graphics-device.md). The accessors used: rect getter `0x58a240`, viewport scalars
`0x58b060` (device `[0xf38..0xf4c]`: origin/far per axis + the z pair), combined modelview `0x58b0c0`,
and (unproject only) the projection-stack matrix `0x58b280`.

### project — `0x5c4d70`

World/clip → screen for an array of `(x, y)` points, with **no perspective divide**. Per point it
takes the dot of clip rows 0/1 with `(x, y, 1)`, then the viewport map:

```
clipx  = m[0]·x + m[4]·y + m[12]        // each clip stored f32
clipy  = m[1]·x + m[5]·y + m[13]
screenx = (clipx + 1)·extent_x·0.5 + origin_x   // extent = far − origin
screeny = (clipy + 1)·extent_y·0.5 + origin_y
out = (screenx·rect[3],  rect[2] − screeny·rect[2],  1.0)   // depth fold; z ≡ 1.0
```

`z` of the input is ignored. The combined matrix `M` is assumed already pre-divided (or fed an affine
matrix) since no `1/w` is applied. This function's only caller is the (unused) decal generator, so it
is **dead in build 5875**.

### unproject — `0x5c4f30`

Screen → world for an array of `(x, y, z)` points — this is the **live mouse-ray picker** path (caller
`0x7e55ac`). It builds the total inverse `M_total = inv(proj·view·viewport)` as a cmath mat4-mul of the
two device matrices, then per point:

```
clip = M_total · (sx, sy, sz, 1)        // (x', y', z', w')
if |w'| < 0.001  →  return 0 (reject the whole call)
invw = 1.0 / w'
out.x = (x'·invw + 1)·0.5·extent_x + origin_x
out.y = (y'·invw + 1)·0.5·extent_y + origin_y
out.z = (z'·invw + 1)·0.5·z_extent + z_origin
```

The x/y origin/extent come from the rect getter; the z-range comes from device fields `[0xf48]`
(near/subtract) and `[0xf4c]` (far/scale). Returns 1 on success, 0 if any point's `w'` is degenerate.

## Ray intersection and picking

The pickers return a no-hit distance equal to the `+inf` sentinel at `0x00c2b850`. All ray functions
assert the direction is a **unit vector**: `|dir·dir − 1| < 2⁻¹⁰` (else abort).

### ray↔sphere — `0x5c53e0`

Projects the sphere centre onto the unit ray:

```
t = dot(center − origin, dir)        // projection distance
if t ≥ −radius:
    closest = origin + t·dir
    if |closest − center|² ≤ radius²  →  hit, *outT = t
```

Note it returns the **projection distance `t`**, not the surface-entry distance. Statically linked,
**zero references in build 5875**.

### Möller–Trumbore ray↔triangle — `0x5c5560`

The core intersection primitive (the mesh pickers call it). `dir` must be unit.

```
e1 = v1 − v0
e2 = v2 − v0
pvec = cross(dir, e2)
det  = dot(pvec, e1)
if |det| < 2⁻²²  →  miss
inv_det = 1/det                      // spilled to f32
tvec = origin − v0
u = dot(tvec, pvec)·inv_det
if !(0 ≤ u ≤ 1)  →  miss
qvec = cross(tvec, e1)
v = dot(qvec, dir)·inv_det
if !(0 ≤ v)  →  miss
if !(u + v ≤ 1)  →  miss
t = dot(qvec, e2)·inv_det            // hit
```

Precision note: the `u ≤ 1` and `u + v ≤ 1` tests reload the `f32`-rounded `u`, while the `u ≥ 0` and
`v ≥ 0` tests use the `f64` register value. The barycentric tests are exactly `0 ≤ u`, `u ≤ 1`,
`0 ≤ v`, `u + v ≤ 1`. It is only ever called from the two (unused) pickers, so it is **dead in build
5875**.

### Mesh pickers

Both walk an index buffer by primitive mode and track the nearest hit; the topology walk:

| `prim_mode` | Topology | Triangles |
|---|---|---|
| 3 | list | `idx[3k .. 3k+3]`, `(nIdx−1)/3 + 1` triangles, face = `k` |
| 4 | strip | `idx[i .. i+3]`, fixed winding, face = `i` |
| 5 | fan | pivot `idx[0]`, `idx[j−1]`, `idx[j]`, face = `j−2` |

**static_pick — `0x5c5bf0`** ray-picks over an already-transformed, tightly-packed (stride 3) vertex
buffer. It owns no geometry math beyond the unit-dir guard — every triangle goes to the
Möller–Trumbore core. Nearest hit is strict `t < best`; there is **no `t ≥ 0` filter** (that belongs to
the skinned picker).

**skinned_pick — `0x5c57a0`** first transforms each vertex by a byte-weighted blend of up to 4 bone
matrices into a scratch buffer, then runs the same topology walk. The blend, per vertex:

```
acc = Σ_{bones, stop at first zero weight byte}  weight_b · (M_b · v)
acc *= 1/255                          // byte-weight normalise (0x8026c8)
```

Each bone transforms the *same* source vertex by its 4×4 matrix (cmath transform-point). Weights are
bytes; bones are consecutive within a vertex (per-vertex byte strides for index/weight arrays). The
walk here **does** filter `t ≥ 0` before comparing to the best. Precision note: bone 0 spills all three
products to `f32`; later bones keep the `x` product in `f64` but spill `y`/`z`.

Both pickers are statically linked with **zero references in build 5875**.

## Spatial point-hash — `0x5c5e30`

Transforms a point as `(x, y, z, 1)` by a matrix (cmath vec4-transform), then folds the six homogeneous
face quantities into a 6-bit (0..63) bucket by an IEEE-bit mix. Operating on the raw `f32` bit patterns:

```
a = bits(w − x) >> 1
a = ((bits(x + w) & 0xbfffffff) | a) >> 1
a = ((bits(w − y) & 0x9fffffff) | a) >> 1
a = ((bits(y + w) & 0x8fffffff) | a) >> 1
a = ((bits(w − z) & 0x87ffffff) | a) >> 1
a = ((bits(z + w) & 0x83ffffff) | a) >> 26   // final >>26 → 6-bit bucket
return a
```

The progressively deepening AND masks clear high mantissa/exponent bits before each OR-and-shift fold.
Statically linked, **zero references in build 5875**.

## Projected-decal UV generation — `0x5c5ee0`

Generates texture coordinates for a projected decal over a **4-vertex** polygon. `poly3d` (4×`(x,y,z)`,
in `ecx`) is the 3-D polygon; `poly2d` (4×`(u,v)`, in `edx`) is read for the AABB/coordinate system and
then **overwritten in place** with the final UVs (its only output). `resX`/`resY` are the decal target
resolution. Statically linked, **dead in build 5875**, but the heaviest owned arithmetic in the module.

Steps:

1. **Project** the 3-D polygon to screen via `project` (`0x5c4d70`): `proj[i] = (sx·rect[3],
   rect[2] − sy·rect[2], 1.0)`, with `proj[i].z ≡ 1.0`.
2. **Per vertex:** apply a floor/ceil axis remap (writes only when a device flag is set — see below),
   accumulate the **centroid** `Σ axis·0.25`, and the 2-D AABB (min/max of `u,v`) over the **input**
   `poly2d`. (The centroid's z is computed but never read.)
3. **Area:** `area = (maxV − minV)·(maxU − minU)·resX·resY` (the resolutions enter as signed-32 → `f64`).
4. **Wrap** each axis-mapped vertex toward the centroid: subtract `1.0` from a component when it is
   strictly greater than the centroid component.
5. **Cross product / reject:** with `B = v0 − v1`, `A = v2 − v1` of the wrapped verts, compute the cross
   `B × A`; **reject (leave `poly2d` untouched) if `|cross|² < 1.0`**. (Because `z ≡ 1`, the only
   surviving cross component is in z, so this reduces to `|c| < 1`.)
6. **Area-based mip reduction:** `count = 0; while ((2·|cross|)·1.5 ≤ area) { area *= 0.5; count++; }`,
   then `resX >>= count`, `resY >>= count` (an x86 `shr` masks the shift count to 5 bits). **Reject if
   either resolution shifts to 0.**
7. **Two 3×3 systems** (rows 0..2): `M1[i] = (wrapped_x, wrapped_y, 1)` and
   `M2[i] = (u·resXf ± 0.5, v·resYf ± 0.5, 1)`, where the `±0.5` is relative to the scaled AABB centre
   `CU = (maxU+minU)·resXf·0.5`, `CV = (maxV+minV)·resYf·0.5`. Precision asymmetry: col0 uses the
   un-rounded `f64` product `resXf·u`; col1 uses the `f32`-spilled `resYf·v`.
8. **Inverse map:** `R = inv3(M1) · M2` (cmath 3×3 det/inverse/multiply).
9. **Final UV** per vertex, written in place to `poly2d` (R rows 0/1, third row unused):

   ```
   invResX = 1/resXf,  invResY = 1/resYf
   u_out = invResX · (R[0]·px + R[3]·py + R[6]·pz)     // pz ≡ 1
   v_out = invResY · (R[1]·px + R[4]·py + R[7]·pz)
   ```

### The axis remap helper — `0x5c63d0`

A flag-selected non-linear remap of a 2-D coord to a 3-vector. The two flags come from the device
struct (`device[0x240]` and `device[0x244]`, fetched via `0x58a230`). When the second flag is non-zero:
`out = (quant(in.x), quant(in.y), 1.0)`; otherwise nothing is written. The first flag picks between two
**byte-identical** quantizer instantiations, so it makes no value difference (its only observable effect
is that "flag4 set, flag8 clear" still writes nothing).

### The quantizer — `0x5c6470` ≡ `0x5c64b0`

Two byte-identical clones. Despite the address range overlapping the CRT transcendental band, both are
the MSVC single-`double` math shell around the same `frndint` rounding kernel (`0x7458d2`), differing
only in a precomputed FPU control word:

```
quant(x) = ceil( floor(16·x + 0.5) · (1/16) )
```

- `0x73ff3f` loads control word `[0x875f00] = 0x173f` (rounding control 01, toward −∞) → **floor**.
- `0x73fdf5` loads control word `[0x875efc] = 0x1b3f` (rounding control 10, toward +∞) → **ceil**.

The pre-add constant `[0x803078]` is `0.5` (not `0.0`). `16·x + 0.5` and `×(1/16)` are exact in `f64`
for any `f32` `x`.

## Image resample — `0x5c51f0`

Nearest-neighbour image rescale by 16.16 fixed-point DDA — pure integer arithmetic plus an element
copy, no floating point. The element size is **4 bytes iff `src_fmt == 1 && dst_fmt == 1`, else 2
bytes** (the formats are compared for equality, never converted). The binary asserts both formats `< 9`.

```
step_x = (src_w << 16) / dst_w        // unsigned, once
step_y = (src_h << 16) / dst_h
// per dst row:   src_row = src + (acc_y >> 16)·src_stride;   acc_y += step_y
// per dst col:   copy elem from src_row + (acc_x >> 16)·elem  to  dst + col·elem;  acc_x += step_x
```

Strides are byte counts. `dst_w`/`dst_h` of zero divide-fault, so a caller must pass non-zero.
Statically linked, **zero references in build 5875**.

## C4Vector / C44Matrix helpers

Small leaf helpers, mostly used by the (dead) plane extractor; `c4_set` is the one with live external
callers (the runtime frustum class at `0x68xxxx` and a scene/camera module — see [Camera](camera.md)).

| VA | Helper | Operation |
|---|---|---|
| `0x5c6540` | `vec4_set` | `dst = (x, y, z, w)` — pure stores, no arithmetic |
| `0x5c6560` | `vec4_scale` | `v[i] *= s` |
| `0x5c6590` | `vec4_negate` | `dst[i] = −src[i]` (`fchs`, exact) |
| `0x5c65c0` | `vec4_sub` | `dst[i] = a[i] − b[i]` |
| `0x5c65f0` | `c44_get_row2` | row index 2 of a column-major 4×4: `(m[2], m[6], m[10], m[14])` |
| `0x5c6620` | `c44_get_row3` | row index 3 (the w-row): `(m[3], m[7], m[11], m[15])` |

## What is statically linked but unwired in build 5875

A large part of this module compiles into the binary but has zero call/pointer references at runtime —
a static-library artifact, not a bug. These remain deterministic, well-defined math, but the live
client never calls them (the runtime frustum class at `0x68xxxx` rolls its own plane extraction and
containment): the second lookAt handedness variant (`0x5c4100`), plane extraction (`0x5c4930`), frustum
AABB (`0x5c4c70`), point-in-frustum (`0x5c54f0`), `project` (`0x5c4d70`), image resample (`0x5c51f0`),
ray↔sphere (`0x5c53e0`), Möller–Trumbore (`0x5c5560`, internal-only to the pickers), both mesh pickers
(`0x5c57a0`/`0x5c5bf0`), the spatial hash (`0x5c5e30`), and the whole decal-UV chain
(`0x5c5ee0`/`0x5c63d0`/`0x5c6470`/`0x5c64b0`). The two live entry points an outside reader will
actually hit are **unproject** (`0x5c4f30`, mouse-ray picking) and **`vec4_set`** (`0x5c6540`).
