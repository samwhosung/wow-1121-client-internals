# Animation — M2 skeletal animation

How the 1.12.1 client animates `.m2` (MD20) models: each frame, a per-instance kernel resolves a
playing sequence into a per-bone clock, binary-searches each bone's keyframe tracks at that clock,
interpolates and cross-fades the keys, builds a 4×4 transform per bone, and propagates those down the
bone hierarchy into a matrix palette. The same machinery drives translation/rotation/scale on bones,
plus per-element tracks (texture transforms, colour, alpha, visibility) and attachment points. There
is no separate animation module — the evaluator is a functional seam inside the M2 model code, built
around one 17.7 KB kernel and six small helpers.

This chapter is mechanism: the data-structure layouts, the keyframe-search and interpolation math, the
sequence-arming contract, and the bone-matrix assembly. See [Models](models.md) for how the model
instance and its buffers are allocated and loaded, and [math-primitives](math-primitives.md) for the
shared matrix/quaternion bodies.

## The evaluator functions

Seven functions carry the whole skeletal evaluator. Everything else they reach is CRT/cmath or model
plumbing.

| VA | Role |
|---|---|
| `0x714260` | The animate kernel (~17.7 KB). The sequence/blend state machine, the dual per-bone clock, the λ cross-fade, the per-bone matrix build, and the billboard switch. Calls the keyframe search at 58 inline sites. |
| `0x713d50` | Keyframe binary-search + dual-time-base resolution + global-sequence resolution. Returns `{k0, k1, t}`. |
| `0x713ea0` | Quaternion (vec4) track sampler — component-wise lerp of the two keys, then global-sequence slerp blend. |
| `0x714000` | Bone matrix propagate — parent-relative → world, recursive up the parent chain. |
| `0x711bf0` | Bone hierarchy walk / billboard-chain classifier (leaf, integer-only). |
| `0x71ae90` | Byte-payload track sampler (step, no interpolation). |
| `0x71af20` | Scalar-float track sampler (linear interpolation + λ blend). |

## CM2Model instance state the evaluator reads

The kernel operates on a `CM2Model` instance (`this`, usually in `ebx`/`ecx`). All the per-frame
animation state lives in model-instance memory; these offsets are read across the seven functions:

| Offset | Width | Meaning |
|---|---|---|
| `+0x10` | dword | "live" / bone-buffer-present gate. The kernel returns immediately if this is 0. |
| `+0x2c` | ptr | `CM2Shared` / model-data pointer. `[[+0x2c]+0xc]` is the running per-anim clock (`dur`); `[[+0x2c]+0x10]` is the global frame tag. |
| `+0x30` | ptr | Model-data root. `[[+0x30]+0x130]` is the MD20 animation header (below). |
| `+0x40` | dword | Last-animated frame tag (the once-per-frame marker). |
| `+0x4c` | dword | Duration tracker; gates the cursor wrap-delta add (zero on the first frame). |
| `+0x64` | ptr | Global-sequence-duration table base; `[+0x64][gseq*4]` is a global sequence's own clock. |
| `+0x8c` | dword | "Max key index" / sequence-length gate, compared against a track's key count to skip empty tracks. |
| `+0x90` | ptr | Per-bone animation block array, stride **`0x118`**, one block per bone (layout below). |
| `+0x94` | ptr | Bone-matrix palette base; a bone's slot is `[+0x94] + lookup*0x40`. |
| `+0xbc` | 16 dwords | Base/world 4×4 matrix (root fallback). |
| `+0xfc` | 16 dwords | The root bone's computed world matrix; also `[+0xfc]/[+0x10c]/[+0x11c]` are the root TRS rows. |
| `+0x12c..+0x134` | vec3 | Model pivot (used by the `flags&1` pivot-translate). |
| `+0x19c` | f32 | Playback-rate-scaled tick (used to scale element tracks). |
| `+0x1c8` | ptr | Parallel payload base for the byte-track element loop. |
| `+0x1cc` | ptr | Parent-bone object pointer (propagate recursion driver). |
| `+0x200` | ptr | Secondary per-bone state array, stride `0x170` (used by the op8 setter and one element loop). |
| `+0x3c4`, `+0x3c8`, `+0x3d0` | ptr | Element track destination arrays. |

## The MD20 animation header

The animation header lives at `[[this+0x30]+0x130]`. The kernel and the sequence-arming path read:

| Offset | Field |
|---|---|
| `+0x1c` | animation (sequence) count |
| `+0x20` | sequence/animation records array, stride **`0x44`** |
| `+0x24` / `+0x28` | `animationLookup` (count / pointer) |
| `+0x2c` / `+0x30` | `playableAnimationLookup` (count / pointer) |
| `+0x34` | bone count |
| `+0x38` | bone-def array, stride **`0x6c`** |
| `+0x3c` | `keyBoneLookup` (key-bone id → bone index) |

A sequence (animation) record is stride `0x44`. The two fields the clock reads are `rec[+4]` =
sequence start tick and `rec[+8]` = sequence end tick; `rec[+0x10] & 1` is the loop/clamp flag.

## The M2 bone-def record

Bone-def records sit in the array at `MD20+0x38`, stride `0x6c`. The kernel reads:

| Offset | Field |
|---|---|
| `+0x04` | bone flags (OR'd with the per-bone block's `+0xf4` to form the working flags) |
| `+0x08` | parent / lookup index (u16; `0xffff` = root) |
| `+0x0c` | a vec3 track descriptor (basis-offset track) |
| `+0x18` | that track's key count |
| `+0x28` | rotation (quaternion) track descriptor |
| `+0x34` | rotation key count |
| `+0x44` | translation track descriptor |
| `+0x50` | translation key count |
| `+0x60` / `+0x64` / `+0x68` | bone basis / pivot vec3 |

## The per-bone animation block (stride 0x118)

`this+0x90` is an array of one block per bone, stride `0x118`. This block holds the entire per-bone
playback and cross-fade state — both a **primary** clock and a **blend** clock, the cross-fade weight,
and the sampled-value scratch. Offsets:

| Offset | Field |
|---|---|
| `+0x98` | primary time (current animation tick) |
| `+0x9c` | primary range/sequence slot (the active key-range selector) |
| `+0xa0` | bone-id frame stamp |
| `+0xa4` | **primary armed record index** (the master arm bit; `-1` = disarmed / inherit) |
| `+0xa8` / `+0xac` | primary cursor window lo / hi (absolute ticks) |
| `+0xb0` | op6 payload |
| `+0xb4` | primary rate (f32) |
| `+0xb8` | primary bias (int) |
| `+0xc4` | blend time |
| `+0xc8` | blend range/sequence slot |
| `+0xd0` | blend armed record index (`-1` = disarmed) |
| `+0xd4` / `+0xd8` | blend cursor window lo / hi |
| `+0xdc` | op6 payload |
| `+0xe0` | blend rate (f32) |
| `+0xe4` | blend bias (int) |
| `+0xf0` | optional pre-multiply matrix pointer (gated by `flags&0x80`) |
| `+0xf4` | per-bone flag word |
| `+0x100` | transition-start tick |
| `+0x104` | 1 / transition-duration |
| `+0x108` | target blend amplitude (the max λ) |
| `+0x10c` | **λ** — the current sequence cross-fade weight |

All per-bone blend state is in this `0x118` block. The hypothetical sampled values land at flavour-
specific offsets relative to a per-track output sub-block (primary value at `+0xc..`, blend value at
`+0x1c..`/`+0x24..`).

## Arming a sequence — the command opcodes

The kernel does **not** loop a bone unless that bone's block is armed (`+0xa4 != -1`). Arming and
re-seeking happen through a small set of command opcodes; the kernel then distributes the armed bone's
clock down the hierarchy.

### The kernel's once-per-frame gate

`0x714260` opens with two guards and commits the frame tag on exit:

```
if (*(this+0x10) == 0) return;                       // model not live
if (*(this+0x40) == *(*(this+0x2c)+0x10)) return;    // already animated this frame
... run the bone loop ...
*(this+0x40) = *(*(this+0x2c)+0x10);                 // stamp the global frame tag (once per frame)
```

The per-bone skip is the inner `*(block+0xa4) == -1` test — a bone with no armed track inherits its
parent's clock (see below) rather than re-running a search.

### op4 (`0x7121a0`) arms exactly one bone

`op4` is the arm command. It takes a **key-bone id** and a **sequence**:

- `param_2` is a key-bone id, resolved through `keyBoneLookup` (`hdr+0x3c`) to a bone index. `-1`
  selects bone 0.
- `param_3` is the sequence, resolved through `0x711bf0` → `playableAnimationLookup` (`hdr+0x2c`) →
  `animationLookup` (`hdr+0x24`).

It writes exactly **one** bone's block: `+0xa4` = the resolved record index, the cursor window
(`+0xa8`/`+0xac`), rate (`+0xb4`), bias (`+0xb8`), the blend setup, and the link. `param_3 == -1`
disarms (`+0xa4 = -1`). There is no loop over bones and no whole-model arm function.

### "Play sequence S" = one op4 on bone 0; the kernel distributes it

A typical play call is a single `op4` on bone 0. Most bones remain `+0xa4 = -1` (set by the per-block
constructor `0x71b0a0` at load). The kernel's bone loop, for each `+0xa4 == -1` bone, **inherits its
parent's resolved time** (`+0x98`) and **sequence slot** (`+0x9c`) — the parent being
`(u16)bonedef[+0x08]` — and then samples that bone's *own* keyframe tracks (`bonedef+0x28` …) at the
inherited time. Because bones are stored parent-before-child, bone 0's clock cascades to the whole
skeleton: unarmed bones still animate (they inherit the *time*, not a static pose). Arming a non-`-1`
key-bone instead arms just that key-bone's subtree, an additive override (e.g. turn-head).

### The arm precondition: linkFlag = 1 plus a resolving lookup

`op4` writes an arm (`+0xa4` set to a record index) only when **both** hold:

1. The two-stage lookup resolves: `animationLookup[playableAnimationLookup[seq]] < hdr+0x1c`. The arm
   gate at `0x712482` (`cmp uVar4,[hdr+0x1c]; jae`) returns otherwise.
2. It is called with `linkFlag (param_8) = 1`. The fork at `0x712514` routes `linkFlag == 0` to the
   blend-snapshot slot `+0xc4` (`0x71268c`, leaving `+0xa4 = -1`), and `linkFlag != 0` to the arm at
   `blk+0x98` (`+0xa4 = record`).

The loader-idle seed call is `0x7121a0(this, -1, seq, 0, 0, 1.0f, 0, 1)`. With `seq=0, linkFlag=1`
this arms bone 0 with `+0xa4 = 0` and window `+0xa8/+0xac = 1000/2000`.

### Re-seek setters and deferred dispatch

`op5`/`op6`/`op8` poke an already-armed block (each guarded by `+0xa4 != -1`):

- `0x7127f0` (op5) re-seeks the window `+0xa8/+0xac`.
- `0x712910` (op6) sets the window plus `+0xb0` and `+0xb4`.
- `0x713430` (op8) pokes the secondary array, `*(this+0x200) + boneIdx*0x170 + 0x100`.

Before a model goes live, commands queue. The deferred dispatcher `0x7101a0` (jump table `0x7102e8`)
replays each queued command through the **same** setter (case == opcode: 4 → `0x7121a0` … 8 →
`0x713430`), restoring each command's request time.

`op4` selects among animation variations with the CRT `_rand` (`0x7400e5`).

## The keyframe search (`0x713d50`)

The search resolves a `{lower key, upper key, fraction}` triple for one track at one time. It is
`fastcall`-ish: `ecx` = the model, then four pushed args — `time`, `range-index`, `track ptr`,
`result-out ptr`; `ret 0x10`.

The track descriptor it reads:

| Offset | Field |
|---|---|
| `+0x2` | global-sequence id (u16); `== 0xffff` ⇒ use the passed-in `time`, else index `[this+0x64][id]` |
| `+0x4` | "has ranges" flag / range count; 0 ⇒ single-range fallback |
| `+0x8` | ranges array; `[+8][idx*8 + {0,4}]` is a `(lo, hi)` key-index pair |
| `+0xc` | total key count |
| `+0x10` | timestamps array (the int32 key-time array searched) |

The result-out is three dwords: `+0` = lower key `k0`, `+4` = upper key `k1`, `+8` = fraction `t`
(f32, in `[0,1]`).

**Range selection.** If `[track+4] != 0`, the search window is the `(lo, hi)` pair from
`[track+8][rangeIndex*8]`; otherwise `lo = 0`, `hi = [track+0xc] - 1`. If `lo >= hi` the result is the
degenerate `{lo, lo, 0.0}`.

**Dual time base.** If `[track+2] != 0xffff` the search key is the global sequence's own clock
`[this+0x64][gseq*4]`; otherwise it is the passed-in `time`. This is the *dual time base*: a track
keyed off a global sequence runs on that sequence's clock, independent of the per-anim time.

**Search heuristic.** The search seeds from last frame's index (a hot-path optimisation) and picks one
of three strategies by `delta = time - prevKey`:

- `delta < 500` (`0x1f4`): linear forward scan from the seed.
- `delta < -500` (`0xfffffe0c` unsigned): linear backward scan, bounded below by `lo` (it reads
  `ts[k]` only for `k ∈ (lo, seed]` and exits at `lo` without reading `ts[lo]`).
- otherwise: re-seed from `lo`, try a short forward scan, else **binary search**:
  `mid=(hi+lo)>>1`; `time < ts[mid]` ⇒ `hi=mid-1`; `time < ts[mid+1]` ⇒ found; else `lo=mid+1`. On a
  collapse exit the result index is the final `lo`.

**Fraction.** With `k0` found and `k1 = k0+1`: if `k1 >= [track+0xc]` the leg clamps to `{k0,k0,0}`.
Otherwise:

```
t = (float)(time - ts[k0]) / (ts[k1] - ts[k0])
```

computed `fild qword / fidiv dword`, narrowed to f32 only at the `fstp`. The intermediate runs at the
binary's 53-bit precision (FPCW `0x027F`); only the f32 store narrows.

## The dual per-bone clock

At the head of the bone loop, the kernel computes two clocks per bone — a **primary** clock
(`+0x98` time, `+0x9c` slot) and a **blend** clock (`+0xc4` time, `+0xc8` slot) — from the bone block
and its armed sequence record (`MD20+0x20 + trackIdx*0x44`). The two legs are byte-identical; the
following is the primary leg (the blend mirrors it on `+0xd0/+0xd4/+0xd8/+0xe0/+0xe4 → +0xc4/+0xc8`).

If `+0xa4 == -1`, the bone inherits its parent's clock (below). Otherwise, with
`lo=+0xa8, hi=+0xac, rate=+0xb4, bias=+0xb8, seqStart=rec[+4], seqEnd=rec[+8], dur=[[this+0x2c]+0xc]`:

```c
// cursor wrap-delta, only when [this+0x4c] != 0:
if (this[0x4c] != 0) { lo += wrapDelta; hi += wrapDelta; }   // +0xa8 then +0xac

if (rec[0x10] & 1) {                 // flag SET -> CLAMP (one-shot), unless hi > dur
    if (hi > dur) goto MODULO(base = (lo > dur) ? lo : dur);
    scaled = trunc((float)(hi - lo) * rate) + bias;          // truncate toward zero
    if      (scaled < 0)                  time = seqStart;
    else if (scaled > seqEnd - seqStart)  time = seqEnd;
    else                                  time = scaled + seqStart;
} else {                             // flag CLEAR -> MODULO (looping)
MODULO(base = dur):
    if (seqEnd <= seqStart) time = seqStart;
    else {
        scaled = trunc((float)(base - lo) * rate) + bias;
        time = (u32)scaled % (u32)(seqEnd - seqStart) + seqStart;
    }
}
boneBlock[0x9c] = trackIdx;          // range/sequence slot
boneBlock[0x98] = time;
boneBlock[0xa0] = boneIdx;           // frame stamp
```

Key points an implementer must get right (these were each a real trap in the binary):

- The loop/clamp flag is `rec[+0x10] & 1`; **clear** ⇒ the modulo (looping) path, **set** ⇒ the clamp
  (one-shot) path.
- The modulo divisor is the **sequence span** `seqEnd - seqStart` (`rec[+8] - rec[+4]`), and the
  remainder is rebased by **`seqStart`** — not by the per-bone cursor window `hi-lo`/`lo`. The cursor
  window feeds only the `fild` numerator.
- The float scale uses MSVC's `__ftol` (`0x40a2b0`): truncate toward zero. There is no `fstp`
  narrowing here — the scaled value is consumed as an integer; the clocks are computed in integer
  ticks.
- The cursors `+0xa8/+0xac` and `+0xd4/+0xd8` are written **only** by the wrap-delta add, gated on
  `[this+0x4c] != 0` (so on the first frame they don't shift).

**Inheritance (the `-1` leg).** When `+0xa4 == -1`, with `parentIdx = (u16)bonedef[+8]`:

- `parentIdx < boneCount` ⇒ copy the parent block's `+0x98/+0x9c/+0xa0` into this bone.
- else if `boneIdx > 0` ⇒ copy **bone 0's** `+0x98/+0x9c/+0xa0`.
- else (`boneIdx == 0`) ⇒ leave the primary clock unchanged.

The blend inherit mirrors this on `+0xc4/+0xc8` (reloading the bone count from `MD20+0x34`), with one
asymmetry: when out-of-range and `boneIdx == 0`, the blend clock copies **this bone's primary** clock
(`+0x98/+0x9c → +0xc4/+0xc8`).

**Transition expiry.** After computing the blend clock, the kernel reads the transition-start tick
`+0x100` and, if `dur - [+0x100] >= 0` (the transition has elapsed), writes `+0xd0 = -1` — the blend
disables its own track once the cross-fade completes.

## The blend weight λ (smoothstep)

λ (`+0x10c`) is the per-frame sequence cross-fade weight. It is selected from the two armed indices and
the elapsed-since-transition counter `elapsed = dur - [+0x100]`:

- If **both** `+0xa4 == -1` and `+0xd0 == -1`: λ is taken from the bone (no-blend path).
- If `elapsed <= 0`, or the two clocks are identical (`+0x98==+0xc4 && +0x9c==+0xc8`): `λ = 0`.
- Otherwise `t = (float)elapsed * [+0x104]` (1/transition-duration), then:

```
t <= 0  ->  λ = 0.0          * [+0x108]
t >= 1  ->  λ = 1.0          * [+0x108]
else    ->  λ = (3.0 - 2t)·t·t · [+0x108]      // smoothstep s(t) = 3t² - 2t³
```

`[+0x108]` is the target blend amplitude (the max weight); the `3.0` is `[0x80297c]` = `0x40400000`.
So λ ramps `0 → [+0x108]` over the transition via the classic Hermite smoothstep.

The blend leg in every sampler fires only when λ is **ordered and non-zero** — the gate is
`fcomp [+0x10c]; fnstsw; test ah,0x44; jnp`, which skips only when λ == 0 or NaN. A **negative** λ
fires the blend leg. The leg additionally requires the track to be global-sequence-tagged
(`word[track+2] == 0xffff`).

## Track interpolation — value flavours and the cubic basis

A track block follows one skeleton: search primary → interp-dispatch → write dest → (optional) search
blend → interp → λ-blend into the primary in place. The interp type is `word[track+0]`: `0` = step,
`1` = linear, `2` = Bézier, `3` = spline/Hermite. The full 4-way switch is emitted **only** in the
cubic (float) loops; TRS and scalar loops collapse to a 2-way step/linear dispatch.

The λ-blend, where it applies, is the in-place lerp `dest = primary + (secondary - primary)·λ`.

Five value flavours share the skeleton:

- **vec3 linear** (stride 12): `dest[i] = a[i] + (b[i]-a[i])·t` for i∈{0,1,2}, `a=P+k0*12, b=P+k1*12`.
  Step copies the three dwords of `P+k0*12`.
- **scalar f32 linear** (stride 4): `dest = a + (b-a)·t`. This is the inline form of `0x71af20`.
- **int16 fixed-point** (stride 2): `dest = (float)(int16 P[k0]) · (1/0x7fff)`, the scale const at
  `[0x811610]` = `3.0518509e-05`. The linear leg reads two int16 via the helpers `0x71aff0`
  (returns `*(track+4) + idx*2`, an element address) and `0x71b010` (copies one int16), then lerps.
- **raw word / raw byte** (step only): `dest = P[k0]` as-is. The blend leg fetches a second raw value
  into the secondary slot but never lerps it — a discrete value can't interpolate.
- **vec3 cubic** (stride `0x24`, key = `{value@+0, inTan@+0xc, outTan@+0x18}`): the full
  step/linear/Bézier/spline switch.

The cubic bases (with `t = fraction`, control points `value[k0], outTan[k0], inTan[k1], value[k1]`):

```
Hermite (spline, case 3):
  h00 = 2t³ - 3t² + 1
  h10 = t³  - 2t² + t
  h01 = 3t² - 2t³
  h11 = t³  - t²
  result[j] = h00·value[k0][j] + h10·outTan[k0][j] + h01·value[k1][j] + h11·inTan[k1][j]

Bézier (case 2):
  B0 = (1-t)³        B1 = 3t(1-t)²        B2 = 3t²(1-t)        B3 = t³
  result[j] = B0·value[k0][j] + B1·outTan[k0][j] + B2·inTan[k1][j] + B3·value[k1][j]
```

Constants: `3.0 = [0x80297c]`, `6.0 = [0x802990]`, `1.0 = [0x7ff9d8]`. There is also a **scalar-float
cubic** (stride `0xc`, key `{value@+0, inTan@+4, outTan@+8}`) with the same coefficients; its basis is
kept entirely on the x87 stack (no f32 spill), so it is strictly higher precision than the vec3 cubic,
which spills `h00`/`h10` (and Bézier's `B0`/`B3`) to f32. There is **no int16-fixed cubic** — int16
tracks are always step or linear.

## The quaternion track and slerp

Rotation tracks are sampled out-of-line by `0x713ea0` (vec4 stride `0x10`). It searches with
`0x713d50`, then on the interpolation leg does a **component-wise lerp** of the two quaternion keys:

```
out[i] = a[i] + (b[i] - a[i]) · t        // i = 0,4,8,0xc
```

If the track is global-sequence-tagged and λ is ordered/non-zero, it samples a second quaternion the
same way and **slerps** the first toward the second by λ via the body at `0x74d11a`.

The slerp (`0x74d11a`, args `out, A, B, t`):

```
d = A·B                                       // 4-component dot, chain order ((a0·b0+a2·b2)+a1·b1)+a3·b3
sign = (d >= 0) ? +1.0 : -1.0                 // shortest arc
cosθ = d · sign
if !(1 - cosθ > 2^-23)  -> LERP fallback:     // near-parallel; eps [0x818868] = 1.1920928955078125e-07
      coeffA = (1 - t),  coeffB = t · sign
else:                                          // general slerp
      s   = sqrt(1 - cosθ²)
      θ   = atan2(s, cosθ)                     // θ narrowed to f32
      inv = 1 / s                              // narrowed to f32 (note: 1/s, not 1/sinθ)
      coeffA = sin(θ·(1-t)) · inv
      coeffB = sin(θ·t)     · inv · sign
out[i] = coeffA·A[i] + coeffB·B[i]            // 4 components
```

This is the one transcendental site in the subsystem (`fsin`/`fpatan`/`fsqrt`); every other
interpolation leg is pure x87 multiply/add and is exactly reproducible bit-for-bit. The `2^-23`
near-parallel epsilon and the f32 narrowing of `θ` and `inv` matter for the exact result.

## Byte and scalar element samplers

Per-element tracks (texture transforms, colour, alpha, visibility, attachment scalars) are sampled by
two named helpers, sharing the search/blend skeleton:

- **`0x71ae90` (byte):** searches, then `byte[out+0xc] = payload[k0]` — a pure step sample, stride 1.
  The blend leg fetches a second byte into `out+0x1c` but does **not** blend it (a byte can't lerp).
- **`0x71af20` (scalar f32):** step copies `payload[k0]` (stride 4) or lerps `a + (b-a)·t`. Its blend
  leg, gated on λ, does blend: `out+0xc = lerp(primarySample, blendSample, λ)`. Element loops call
  this for four consecutive scalar channels per element (e.g. colour RGBA / alpha).

## Per-bone matrix assembly

For each bone, after the clocks and λ are set, the kernel builds the bone's transform. The combined
flags are `[block+0xf4] | [bonedef+0x04]`. A root bone (lookup `0xffff`) writes into `CM2Model+0xfc`;
others into the palette slot `[this+0x94] + lookup*0x40`. If `combined & 7 == 0` the bone skips the
normalize/pivot block (no animated basis).

`combined & 6` selects a row-normalize variant on the working 3×3:

| `flags & 6` | Behaviour |
|---|---|
| 0 | no row processing |
| 2 | **unit-normalize** each row; eps `[0x8029d4]` = `2^-22` (`2.384185791015625e-07`) |
| 4 | **ratio-normalize**: replace each row with the root direction scaled to the local row's magnitude (`row = (\|local\|/\|root\|)·root`); eps `[0x80c5c8]` = `1e-05` |
| 6 | copy the root rows `[ebx+0xfc]/[+0x10c]/[+0x11c]` |

The unit-normalize idiom is shared with the billboard arms: `len = sqrt(rowK·rowK)`; if `len > eps`,
`recip = 1.0/len` (`fdivr [0x7ff9d8]`) and scale the row; if `len <= eps`, leave the row unscaled (the
eps guard, no divide). Note: this is row-normalize, not quaternion renormalize — there is no quaternion
renormalize in the rotation post-processing.

`flags & 1` chooses the translation: set ⇒ the model pivot `[ebx+0x12c..]`; clear ⇒ the pivot-fold
`translation[j] = pivot[j] - (M_col_j · b)` (b = `[bonedef+0x60..]`), keeping the bone's pivot fixed
across re-orthonormalization.

The TRS path then: samples rotation (`0x713ea0`) into a quaternion at `block+0x3c`, converts it to a
matrix at a scratch via `0x74b6b5` (body `0x74b6bb`); if no rotation keys, builds identity. It scales
the rows by the sampled scale vec3 (`0x7bdca0`); optionally pre-multiplies by `[block+0xf0]` when
`flags & 0x80` is set (`0x74a7c0`, body `0x74a7c6`); folds the `[bonedef+0xc]` track vec3 into the
basis; builds the translation row by the same pivot-fold; and composes
`palette[boneIdx] = TRS · sourceMatrix` via `0x74a7c0`.

The TRS-vs-billboard route is `flags & 0x280`: **set** ⇒ the animated-TRS path above; **clear** ⇒ the
camera-basis billboard switch. Both paths converge at the `flags & 0x78` billboard selector.

## Bone matrix propagate (`0x714000`)

`0x714000` turns parent-relative transforms into world matrices by recursing up the parent chain
(`thiscall`, `ecx` = bone object). It is short-circuited per node by the same frame-tag done-guard the
kernel uses (`[this+0x40] == [[this+0x2c]+0x10]`).

- If the bone has a parent (`[this+0x1cc]`), recurse with `ecx` = the parent first (ensure-parent-
  computed). If no parent, build an identity transform (`0x3f800000` = 1.0f for the diagonal) and call
  the kernel.
- The compose leg resolves a per-bone animation record (`[parent+0x30]→+0x130→+0x108`, record stride
  `0x30`; a `0x40`-stride source matrix at `[parent+0x94] + (u16 record[+4])<<6`), copies that 4×4 into
  a scratch, then does a matrix-vector accumulate (`scratch_translate += M·boneVec`) transforming the
  bone's local offset by the parent record's 3×3 + translate.
- Copy-out: if a parent exists, the bone's world matrix `[this+0xfc]` is copied from `[parent+0xfc]`;
  otherwise `0x7bc6a0` (a cmath 4×4 multiply) composes the local matrix against the base matrix
  `[this+0xbc]`, and that result is copied to `[this+0xfc]`.

## The billboard switch (`flags & 0x280 == 0`)

When the TRS bit is clear, the bone matrix is built from the camera basis. The selector is
`(flags & 0x78) - 8`; if it is `> 0x38` (unsigned) the default arm is taken. Otherwise a 24-byte index
map at `0x71879c` and a 5-entry jump table at `0x718788` pick the arm:

| `flags & 0x78` | Arm | Behaviour |
|---|---|---|
| `0x08` | arm 0 (`0x7152f8`) | spherical — three independent row normalizes (or, on the billboard path, a fixed basis: row0=(0,0,-1), row1=(1,0,0), row2=(0,1,0)) |
| `0x10` | arm 1 (`0x715531`) | cylindrical, locked axis row0; row1 = `(y, -x, 0)` normalized; row2 = row0 × row1 |
| `0x18` | arm 2 (`0x715639`) | cylindrical, locked axis row1; row0 = `(-x, +y, 0)` normalized (note the negated first component); row2 = row0 × row1 |
| `0x20` | arm 3 (`0x715742`) | cylindrical, locked axis row2 (inline squared-length); row1 = `(y, -x, 0)`; row0 = row1 × row2 |
| other (incl. `0x00`) | arm 4 (`0x715868`) | default tail |

All arms fall through the common tail (`0x715868`): scale the three rows by the per-axis scales, zero
the w-column, build the translation column `rowScaled·cameraBasis - translate`, set the bottom-right to
`1.0`, then `boneIdx++` against the bone count (`MD20+0x34`).

The shared squared-length helper is `0x4549f0` (`thiscall`, returns `x²+y²+z²` on st0).

## Shared matrix and quaternion bodies

These leaf bodies are reached through delay-linked import thunks; their natural home is the math
library (see [math-primitives](math-primitives.md)). Each is pure multiply/add/sub — no transcendentals
— and so exactly reproducible.

- **`0x74a7c6` — 4×4 × 4×4 multiply** (`a = b · c`, `cdecl`, `ret 0xc`). Row-major. Has an
  aliasing fast path: if `out` aliases an input it writes a local scratch and `rep movsd` 16 dwords
  back; otherwise it writes `out` directly. 16 `fstp dword` narrowing points.
- **`0x74b6bb` — quaternion → 4×4 matrix** (`cdecl`, `ret 8`). Loads `2.0f` from `[0x801628]`
  (`0x40000000`), forms the doubled-component products, writes the standard
  `out[0] = 1 - 2(yy+zz)` etc. rows, zeroes the off-diagonal w terms, and sets `out+0x3c = 1.0`. The
  exact per-element f32-spill-vs-register narrowing matters: e.g. `2x`/`2y` are spilled to f32 while
  `2z` stays on the stack, and element `m10` (`out+0x28`) recomputes `1 - 2xx` from a spilled f32 — a
  distinct codegen from the symmetric diagonal terms.
- **`0x7bdc40` — translate a 4×4 by a local offset** (`[ecx+0x30] += v·col0`, etc.).
- **`0x7bdca0` — scale matrix rows by a vec3** (row0·=s.x, row1·=s.y, row2·=s.z).
- **`0x7bddb0` — quaternion → matrix-rows** (same family as `0x74b6bb`, used by the texture-transform
  pre-pass).

One element loop applies a texture-transform rotation about the UV pivot `(0.5, 0.5, 0.0)` (globals
`[0xcf043c]/[0xcf0440]/[0xcf0444]`): translate to pivot (`0x7bdc40`), rotate (`0x7bddb0`), translate
back by the negated pivot. Two element loops tick-scale a sampled scalar by the playback tick
`[ebx+0x19c]` and apply it to a direction vector (e.g. UV scroll / particle velocity).

## The bone hierarchy classifier (`0x711bf0`)

`0x711bf0` classifies a bone within the billboard/parent chain (`thiscall`, args = out ptr, bone
index; leaf, integer-only). It resolves the animation header at `[[this+0x30]+0x130]`, and:

- **Fast path:** if the bone index is in range, return a precomputed per-bone descriptor dword
  directly.
- **Chain walk:** otherwise it walks a global flag table at `0xc0e070` (count `0xc0e074`), following
  each entry's link (`entry+0x18`) and accumulating a signed direction from the entry flags
  (`cl&0x10` flips direction, `cl&0x20` stops it), tracking visited nodes in a 208-dword scratch map.
  The default seed is `0x93` (147) when the first lookup word is `0xffff`.

The output packs both results: `out = (classCode << 16) | value`, where the low word is the resolved
bone/value and the high word is a 2-bit classification (direction sign / accumulator sign).
