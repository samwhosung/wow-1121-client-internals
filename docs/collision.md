# Collision & movement — player vs world geometry

How the 1.12.1 client moves the local player through the world and resolves that motion against terrain,
WMO, and M2 geometry. A per-frame movement tick drives a kinematic state machine over a single `CMovement`
object: input and network move-messages become a timestamped event queue, the queue is replayed in fixed
250 ms substeps, each substep integrates position/facing/pitch, and the integrated displacement is swept
against the world by a resolution layer that slides, steps up, and falls. The movement TU also reports the
player's motion to the server (the MSG_MOVE / heartbeat cadence). The query side — the actual ray/AABB
intersection against terrain and models — is owned by the [Terrain](terrain.md) and [Models](models.md)
subsystems; this chapter is the consumer that sweeps the player capsule against them.

## The entry point and driver chain

Movement runs from core's event bus as a **category-5 handler** (registered at `0x401af1` via the core
registration `0x41fca0`, dispatched by `0x4245b0`). The handler is the per-frame tick `0x616800`. The chain
from there to the integration loop:

| VA | Role |
|---|---|
| `0x616800` | Per-frame tick. Reads `OsGetAsyncTimeMs` (`0x42c010`), fetches the manager (`0x630a40`), runs one step per elapsed window gated on `mgr+0x128`. |
| `0x615b10` | Move-units loop over the manager list `mgr+0x118`. Per unit, `MOVEMENTFLAGS(+0x40) & 0x8000000` routes the free-advance `0x615ae0`, else the local integrator. |
| `0x616620` | **MoveLocalPlayer.** Clamps the elapsed window into fixed **250 ms substeps** (`cmp 0xfa` @`0x616642`), drives the loop, NaN-guards the result (`0x40a1b8`), queues the outbound move (opcode `0xee` via `0x600a30`). |
| `0x616de0` | The per-substep integrate loop (body detailed below). |
| `0x615c30` | Local jump-table dispatcher (table `0x616580`, the `0x26` opcode cases) that replays queued move events. |
| `0x618080` | Apply-to-wire dispatcher (table `0x618528`). |
| `0x7c5360` | The per-substep integrator/dispatcher (own-input branch). |

The movement subtree consumes the world through a **virtual interface** (`0x630ac0` / `0x630b70` → vtable
slots, resolved at runtime) plus the world-trace wrapper APIs (`0x672130`–`0x6721b0`, `0x480a50`,
`0x6821f0`).

## The kinematic state machine

Movement is a state machine over one struct — the `CMovement` object. The driver TU (`0x615020`–`0x61b000`)
turns input and network move-messages into **`0x50`-byte event nodes** queued on `CMovement+0x150`, sorted
by timestamp, and replays them through two jump-table dispatchers (`0x615c30` local via table `0x616580`,
`0x26` cases; `0x618080` apply-to-wire via `0x618528`) into the physics TU (`0x7bca00`–`0x7c7400`).
`0x7c5360` is the per-substep integrator/dispatcher (`dt = ms·0.001`, the `+0x40` flag decode, jump table
`0x7c5468`); it issues **no geometry queries**. The local player integrates in fixed 250 ms substeps
(`0x616620` → `0x616de0`).

The key `CMovement` fields:

| Offset | Field |
|---|---|
| `+0x10` / `+0x14` / `+0x18` | Live position (C3Vector, world yards) |
| `+0x1c` | Live facing (heading) |
| `+0x20` | Live pitch |
| `+0x40` | `MOVEMENTFLAGS` (keyboard/state flags) |
| `+0x44` / `+0x48` / `+0x4c` | Move-base position (C3Vector) |
| `+0x50` | Facing base |
| `+0x54` | Pitch base |
| `+0x58` | int32 substep-time / fall-timer accumulator (ms) |
| `+0x5c` / `+0x60` / `+0x64` | World-space velocity basis |
| `+0x68` / `+0x6c` / `+0x70` | Turn-arc rotation basis |
| `+0x78` | Airborne-ms accumulator (landing) |
| `+0x7c` | Predicted-Z reference (anti-warp / fall-damage) |
| `+0x84` | Move speed |
| `+0x9c` | Strafe speed |
| `+0xa0` | Fall velocity (**down-positive**, see below) |
| `+0xa4` | Movement-source descriptor (MI); NULL ⇒ pure own-input |
| `+0xb0` / `+0xb4` / `+0xb8` | Param scales (written by `0x6174b0`) |
| `+0x144` | Facing-interp angular velocity |
| `+0x148` | Facing-interp window flag (≠ 0 ⇒ live) |
| `+0x14c` | Prediction-reconcile window |
| `+0x150` | Move-event queue (`0x50`-byte nodes, timestamp-sorted) |

### Driver primitives

The feel-defining driver math:

| VA | Function | Detail |
|---|---|---|
| `0x616af0` | Anti-warp clamp | Horizontal speed² > (60 yd/s)² OR step dist² > (3 yd)² ⇒ snap-apply and abort. |
| `0x616bf0` | World-bounds validity | Per-axis `(−17064.666, +17064.666]`; z finite-only. |
| `0x616cb0` | Pos-apply | Build delta, ground vertical-zero, commit via `0x634040`. |
| `0x616d60` | 3-yd move dead-band | Commit only when accumulated move ≥ 3 yd (`dist² ≥ 9.0`). |
| `0x617080` | Angle wrap | Fold to `[0, 2π)` — a single ±2π adjustment per call (it does not loop). |
| `0x617460` | Turn-rate ceiling | `· 0.5407162`. |
| `0x6174b0` | Param scales | Writes `+0xb0` / `+0xb4` / `+0xb8`. |
| `0x618920` | Facing-interp turn-rate selector | Returns `&(+0x144)` angular velocity when the interp window is live (`+0x148 ≠ 0`), else 0. |
| `0x618f80` | Remote facing-interp angular velocity | Shortest-arc with ±π fold. |
| `0x619090` | Prediction-reconcile gate | Arms `+0x14c` when client-predicted pos disagrees with the queued server event by ≥ ~0.0278 yd (`dist² ≥ 7.716e-4`); 2-/3-axis by flag `0x200000`. |
| `0x6191c0` | Position extrapolation/lerp | `out = target + (cur − target)·frac` over the `+0x14c` window. |

## The per-substep integration sequence

This is the loop body an implementer writes; the math of every callee is detailed below, and what follows
is the **order** they run in. The substep loop is `0x616de0`; it owns only the `+0x58` integer accumulate —
all trajectory math is in the callees.

```
# prologue
if valid_position(0x616bf0) == 0:   return        # world-bounds + finite gate
if elapsed_ms == 0:                 return
consumed = 0 ;  carry = (0,0,0)                    # carry = accumulated newpos; filled+read ONLY by the spline-follow branch

# substep loop — runs while any move/jump/fall flag remains
while (CMovement+0x40 & 0x20ff) != 0:
    remaining = elapsed_ms - consumed
    CMovement+0x58 += remaining                    # +0x58 = int32 substep-time / fall-timer accumulator (ms)
    delta = (0,0,0)
    MI = CMovement+0xa4                             # movement-source descriptor (NULL ⇒ pure own-input)

    if MI != 0  AND  (MI.flags & 0x4) == 0:         # ── SPLINE / TRANSPORT-FOLLOW branch (fall-through @0x616e5c)
        if anti_warp(0x616af0, arg0, remaining, &delta) == 0:
            return                                  # 0x7c5490 sampled the path into delta; the speed²/dist² gate snapped+committed → abort
        carry = CMovement.base(+0x44) + delta
    else:                                           # ── OWN-INPUT (physics) branch — MI == 0 OR (MI.flags & 0x4) set (jump @0x616ea6)
        turn_rate = facing_interp_rate(0x618920)    # &(+0x144) angular-vel when the facing-interp window is live (+0x148≠0), else 0
        if integrate_dispatch(0x7c5360, this, fallaccum = CMovement+0x58, turn_rate) == 0:
            return                                  # 0x7c5360 reads the keyboard flags (+0x40) → linear/turn/pitch appliers
        if CMovement+0x14c != 0:                    # client-prediction reconcile window armed (0x619090 set it)
            pred = delta + CMovement.base(+0x44)
            if position_lerp(0x6191c0, now-remaining, &pred) != 0:
                delta = pred - CMovement.base(+0x44)        # reconciled delta replaces the integrated one

    # ── APPLY (every substep)
    consumed += build_delta_commit(0x616cb0, arg0, now-remaining, &delta)
    #   0x616cb0: out = base(+0x44)+delta − pos(+0x10) ; ground-z-zero ; commit via 0x634040 (advances +0x10)

# finalize
if valid_position(0x616bf0) == 0:                   return
if MI != 0 AND (MI.flags & 0x4) == 0:               anchor_snap(0x616d60, &carry)      # 3-yd dead-band — spline-follow only
if MI != 0 AND (MI.flags & 0x4) == 0 AND (MI.flags & 0x200): begin_move(0x7c5cd0)      # land-edge — spline-follow only
```

### The branch split

Each substep takes exactly one path, selected at `0x616e54` / `0x616e5a` on `MI(+0xa4)` and `MI.flags & 0x4`:

- **`MI != 0 AND (MI.flags & 0x4) == 0`** → the **spline / transport-follow** path (fall-through to
  `0x616af0` @`0x616e5c`).
- **`MI == 0 OR (MI.flags & 0x4) set`** → the **own-input (physics)** path (jump to `0x616ea6` → `0x7c5360`).

So a **null `MI`** (no movement-source descriptor) is *own-input*, not spline; and `MI.flags & 0x4` *clear*
selects the *spline* follow.

`0x7c5490` is a **spline sampler, never a speed·dt integrator**. It early-outs unless `+0x40 & 3`, then works
the spline-timing struct at `+0xa4` (`fidiv` progress fractions over `+0x18` / `+0x1c` / `+0x20` / `+0x24`),
samples the track via the vtable samplers `0x453480` / `0x453390`, sets the facing `+0x1c = atan2(y, x)`, and
writes `delta = sampled_pos − base(+0x44)`.

**Gotcha — the `+0xa4` descriptor's `+0x18` is a packed word.** It is read both ways within one substep: as a
**flag byte** by the driver (`0x616af0` / `0x616cb0` / `0x616de0`) and as a **signed spline-period int** by
`0x7c5490`. It is not a plain `MovementInfo`; it is populated across the network move-receive seam for the
MI-present spline/transport cases.

### Spline / transport-follow branch — `0x616af0` → `0x7c5490`

`0x616af0` is the sole caller of `0x7c5490` (@`0x616b02`). It first runs `0x7c5490` — the transport/spline
position+facing follow — which samples the queued path into `delta` (so `base(+0x44) + delta` recovers the
sampled world position) and sets the facing. `0x616af0` then runs the **anti-warp gate** on that sampled
displacement: horizontal speed² > (60 yd/s)² OR step dist² > (3 yd)² ⇒ snap-apply the position directly and
abort the loop (the guard against a bad server update, returning 0); else the small `delta` flows to the
apply phase. On the falling leg it refreshes the predicted fall-Z `+0x7c` via `0x7c5f30`. **Own-input
movement never reaches this gate.**

## Own-input physics — `0x7c5360` (ApplyMovement)

`0x7c5360` is the input-flag → position dispatcher, selected when `MI == 0 OR MI.flags & 0x4 set`. It is
preceded by `0x618920` (the facing-interp turn-rate selector, which returns the `+0x144` angular velocity
when the interp window is live, else 0). `0x7c5360`:

1. Converts the substep duration to seconds: `dt_s = (u32)ms · 0.001` (its one owned arithmetic). **The dt is
   the total `+0x58` accumulator, not the per-substep remaining** — `0x616de0` passes `arg0 = [esi+0x58]`
   (@`0x616eb2`).
2. Decodes the keyboard `MOVEMENTFLAGS(+0x40)`: move `0x3`, strafe `0xc`, turn `0x30`, pitch `0xc0`.
3. Dispatches through a two-level jump table to the displacement applier for the active motion mode.

### The dispatch structure

The flag decode builds a 4-bit **mode mask `m`** in `edi`:

```
MOVE  = 1   (fwd/back  0x1|0x2)
TURN  = 2   (turn      0x10|0x20)
STRAFE= 4   (strafe    0x4|0x8)
PITCH = 8   (pitch     0x40|0x80)
```

then dispatches `case = idxtab[m−1]; jmp target[case]` through a two-level table: a **15-entry byte index
table at `0x7c547c`** (`m−1 → case`) feeding a **5-entry dword target table at `0x7c5468`** (`0x7c53df
movzx ecx,[edi+0x7c547c]` · `0x7c53e6 jmp [ecx*4+0x7c5468]`). `m == 0` (e.g. falling-only) falls through
`dec edi; cmp edi,0xe; ja 0x7c545a` and returns `(0,0,0)`.

**Orientation is applied separately and before the table — never through it.** The facing applier `0x7c4f30`
is called at `0x7c53a7` whenever `(flags & 0x30)` [turn] **OR** `arg1 ≠ 0` [the `0x618920` facing-interp
override]; the pitch applier `0x7c4f80` at `0x7c53c0` whenever `(flags & 0xc0)` [pitch]. Both take the same
`dt_s` and write the live orientation recomputed from the fixed bases. The jump table that follows selects
**only the displacement applier**. So a **move+turn** runs BOTH `0x7c4f30` (heading integrate) and the case-2
arc `0x7c52c0` (curved delta), sharing one `dt_s`. A **move+turn+pitch** runs three appliers: `0x7c4f30`
(facing) + `0x7c4f80` (pitch) + the case-4 turn+pitch arc `0x7c4fd0` (displacement).

**The 5 displacement cases** (dword target table `0x7c5468`):

| Case | Target | Applier |
|---|---|---|
| 0 | `0x7c53f8` → `0x7c4ed0` | linear_move |
| 1 | `0x7c53ed` → none | returns 0 (orientation-only, no translation) |
| 2 | `0x7c5412` → `0x7c52c0` | simple_turn_move arc |
| 3 | `0x7c5430` → `0x7c5180` | pitch-only arc |
| 4 | `0x7c544a` → `0x7c4fd0` | turn+pitch arc |

Each returns `delta` to the caller's out-vec (`[ebp+0x10]`); the arc cases also take the turn override
(`[ebp+0xc]`).

**The combo → case map** (byte index table `0x7c547c`, by mode mask `m`):

| `m` | modes | displacement applier |
|----:|-------|----------------------|
| 1 | MOVE | `0x7c4ed0` linear |
| 2 | TURN | none (facing-only) |
| 3 | MOVE+TURN | `0x7c52c0` turn arc |
| 4 | STRAFE | `0x7c4ed0` linear |
| 5 | MOVE+STRAFE | `0x7c4ed0` linear |
| 6 | TURN+STRAFE | `0x7c52c0` turn arc |
| 7 | MOVE+TURN+STRAFE | `0x7c52c0` turn arc |
| 8 | PITCH | none (pitch-only) |
| 9 | MOVE+PITCH | `0x7c5180` pitch arc |
| 10 | TURN+PITCH | none (facing+pitch-only) |
| 11 | MOVE+TURN+PITCH | `0x7c4fd0` turn+pitch arc |
| 12 | STRAFE+PITCH | `0x7c4ed0` linear |
| 13 | MOVE+STRAFE+PITCH | `0x7c5180` pitch arc |
| 14 | TURN+STRAFE+PITCH | `0x7c52c0` turn arc |
| 15 | MOVE+TURN+STRAFE+PITCH | `0x7c4fd0` turn+pitch arc |

Reading it: **case 1 (no displacement)** is every orientation-only combo — no MOVE and no STRAFE
(`m ∈ {2,8,10}`). **TURN** with any translation curves (case 2 turn-arc), escalating to the **turn+pitch
arc** (case 4) only when MOVE+TURN+PITCH are *all* set (`m ∈ {11,15}`). **MOVE+PITCH without TURN** takes the
**pitch-only arc** (case 3, `m ∈ {9,13}`). Everything else translating is **linear** (case 0) — including
STRAFE-only and STRAFE+PITCH (no MOVE), because STRAFE folds into the velocity basis `+0x5c..` via `0x7c5a20`
rather than arcing; only TURN (heading curve) or MOVE+PITCH (pitched climb/dive) bends the path.

**Falling suppresses the TURN bit in the displacement index only — not the facing apply.** The `mov edi,2`
that sets the TURN bit (`0x7c53b4`) is skipped when the FALLING flag `0x2000` is set (`0x7c53af test ah,0x20;
jne`). So a falling move+turn still integrates the heading (`0x7c4f30` already ran at `0x7c53a7`, before the
gate) but its displacement drops to the non-turn case (no arc while airborne). The vertical fall delta is not
`0x7c5360`'s — it is the resolution-layer FALL resolver `0x635b00`'s `−fall_displacement`. So **fall+move**
routes the horizontal leg through `0x7c5360`'s table (turn-suppressed) and the z through `0x635b00`.

### The displacement appliers

| VA | Applier | Formula |
|---|---|---|
| `0x7c4ed0` | linear_move | `displacement = dir(+0x5c..) · speed(+0x84) · dt` |
| `0x7c4f30` | strafe_turn (heading) | `facing_live(+0x1c) = fmod(strafeSpeed·dt + facing_base(+0x50), 2π)`, re-folded to `[0,2π)` |
| `0x7c4f80` | pitch_integrate | `pitch_live(+0x20) = clamp(pitchRate·dt + pitch_base(+0x54), −π/2, +π/2)` |
| `0x7c52c0` | simple_turn_move | `R = speed/turnRate`, `θ = turnRate·dt`, `fsincos` rotate through basis `+0x68/+0x6c/+0x70` |
| `0x7c4fd0` | turn+pitch arc | `R = speed/turnRate`, `sp_pr = (1/pitchRate)·speed`, fsincos basis rotation, ±π/2 pitch clamp, `±clampTime·speed` Z-correction |
| `0x7c5180` | pitch-only arc | same fsincos basis rotation with the ±π/2 clamp and Z-correction |

**Precision note (linear_move).** The **z leg carries one extra f32 narrowing** the x/y legs do not — the
`dt·dir.z` spill/reload at `0x7c4ef1` / `0x7c4f05` — so the z displacement is computed at a different
intermediate precision from x/y. This affects the exact result and must be reproduced.

**strafe_turn details.** It **reads the base `+0x50`** (`fadd [esi+0x50]`) and **writes the live `+0x1c`**
(`fstp [esi+0x1c]` @`0x7c4f75`) — recompute-from-base, not an in-place `+0x50` accumulate. `strafeSpeed`
comes from `0x7c5c50`: `±speed(+0x9c)` by flag bit `0x10` / `0x20`, `×0.75` when `flags & 0x200f`. The
constant `2π` is the BSS global `[0xcf5d00]` = `(2·π_f32)`; the fold is `0x73f90a` fmod plus a `+2π` re-add
when the result is `< 0`.

**pitch_integrate details.** It **reads the base `+0x54`** (`fadd [esi+0x54]`) and **writes the live `+0x20`**
(`fstp [esi+0x20]` @`0x7c4fb9`, or the saturated bound stored directly @`0x7c4f9e` / `0x7c4fc1`). On
saturation it substitutes the bound itself: `−π/2 = 0xbfc90fdb` / `+π/2 = 0x3fc90fdb`. `pitchRate` comes
from `0x7c5c90`, bit `0x40` / `0x80`.

**The velocity basis** `+0x5c..` is built by `0x7c5880` (fsincos of facing, plus pitch when flag `0x200000`
set) and the input-flag → velocity combine `0x7c5a20` (forward/back `0x1`/`0x2`, strafe `0x4`/`0x8`,
diagonals `× 1/√2 = 0x81da54`) on each move-state change (callers `0x61701b` / `0x618e24` / `0x7c5d05`), not
per substep.

## The recompute-from-base model

Own-input position **and** orientation share one architecture: a **fixed base**, re-anchored only at
move-state transitions, plus a **live value** recomputed every substep as `base + rate · (total elapsed)`.
The dt for every applier is the **total `+0x58`** accumulator, converted once to `dt_s = (u32)[esi+0x58] ·
0.001`.

- **Facing:** `0x7c4f30` reads base `+0x50`, computes `fmod(rate·dt_s + base, 2π)` folded to `[0,2π)`, writes
  live `+0x1c`. Base `+0x50` is untouched.
- **Pitch:** `0x7c4f80` reads base `+0x54`, computes `clamp(rate·dt_s + base, ±π/2)`, writes live `+0x20`.
  Base `+0x54` untouched.
- **Position:** the translation appliers produce `delta` over `dt_s`; `0x616cb0` forms
  `out = base(+0x44) + delta − pos(+0x10)` and commits through `0x634040`, advancing the live position
  `+0x10`.

**The re-anchor — `0x7c5cd0` (UpdateAnchors)** snapshots all three bases from the live values at each
move-state transition: `+0x44 ← +0x10` (pos), `+0x50 ← +0x1c` (facing, `0x7c5cf5`), `+0x54 ← +0x20` (pitch,
`0x7c5cfb`), then resets `+0x58 := 0` (`0x7c5cfe`) and rebuilds the velocity basis (`0x7c5a20`).

So within one continuous move the bases (`+0x44` / `+0x50` / `+0x54`) are constant while `+0x58` grows, and
the live fields (`+0x10` / `+0x1c` / `+0x20`) are pure functions `base + rate·(+0x58 seconds)` —
**drift-free, with no per-substep f32 accumulation**. The simulation-observed orientation is `+0x1c` /
`+0x20`; `+0x50` / `+0x54` are the fixed bases.

### The move-base (+0x44) writer

The apply phase **reads** `base(+0x44/+0x48/+0x4c)` (a C3Vector, world yards) but never writes it (the loop
does `fadd`/`fsub` @`0x616e77` / `0x616f02`, never a store). The base is committed by the move-state machine.
Three writers:

- **ctor `0x7c4850`** seeds `base := spawn pos` — the same C3Vector it also stores at live pos `+0x10`
  (caller `0x616fa0`).
- **`UpdateAnchors 0x7c5cd0`** — the own-input writer — snapshots `base(+0x44..) := live pos(+0x10..)` and
  resets `+0x58 := 0` (then rebuilds the velocity basis `0x7c5a20`, notifies `0x6307a0`). It fires at every
  move-state transition: the own-input keyboard command setters (the `0x7c69a0..0x7c7300` cluster — each
  mutates `MOVEMENTFLAGS +0x40` then re-anchors), the state wrappers `0x619c50` / `0x61a770`, the fall
  starters `0x7c61f0` / `0x7c6230`, and the resolution-layer re-anchors `0x634942` / `0x635ae8` / `0x636f57`.
- **net deserialize `0x7c6420`** sets `base := inbound packet pos (+0x24/+0x28/+0x2c)` and `+0x58 := 0` (sole
  caller the move-message timing core `0x618c30` @`0x618dfc`).

So the base is written once per continuous move and stays fixed while `+0x58` accumulates and the integrated
`delta = velocity·(+0x58 as seconds)` grows. An empty-world own-input player (`MI +0xa4 == 0`) needs only the
ctor seed plus the per-move `0x7c5cd0` snapshot; the loop's own `0x7c5cd0` land-edge call (`0x616f89`) is
MI-gated and never runs for it.

## Gravity and falling kinematics

The fall is timed by the `+0x58` accumulator (advanced by `remaining` ms each substep). The falling
primitives:

| VA | Function | Detail |
|---|---|---|
| `0x7c5d20` | Gravity velocity integrate | `v += g·dt`; `g = 19.291105` (`[0x81da58]`); terminal-clamped `+60.148` land (`[0x87d894]`) / `+7.0` swim (`[0x87d898]`). |
| `0x7c5e70` | Fall displacement | `Vc·dt + ½g·dt²`; `½g = 9.645553` (`[0x81da60]`); terminal-reached piecewise. |
| `0x7c5f50` | Analytic fall-time solver | terminal-capped; `g = 19.29111`, `Vt = 60.148` land / `7.0` swim. |
| `0x7c5da0` | Swim fall velocity | |
| `0x7c5f00` / `0x7c6140` / `0x7c6110` | ms-wrapper / accessor chain into the kernels | |

**`fall_vel (+0xa0)` is down-positive.** The integrate adds `g` *positively* — `0x7c5ea5` `d9 45 08` fld dt ·
`0x7c5ea8` `d8 0d 58 da 81 00` fmul g · `0x7c5eae` `d8 c1` fadd ⇒ `vc' = vc + g·dt` — and the terminal clamp
is an *upper* bound at the positive cap (`0x7c5eb0` `d8 5d fc` fcomp cap; the clamp body is entered only when
`vc' > +cap`). So `fall_vel` grows toward `+60.148` during a fall: **a downward fall is `fall_vel > 0`**, an
upward jump `fall_vel < 0`. The kernel return is a positive downward magnitude (`½g·dt²` from rest).

**The applied vertical delta is `off.z = −fall_displacement`.** The own-input fall apply is the
resolution-layer **FALL-mode resolver `0x635b00`** (dispatched by the resolve entry `0x634040`). Per substep
it forms the move delta `(local.x = h·dir.x, local.y = h·dir.y, local.z = −fall_displacement)`, where `h` is
the horizontal move scalar (`[ebp+0x10]`), `dir` the unit move direction (`[ebp+0x14]`), and
`fall_displacement` the positive accumulated fall offset returned in `st0` by `0x7c6140(this, [esi+0x78]+dt)`.
The negation is the explicit **`fchs` @`0x635b4f`** (`d9 e0`) applied to that positive return, stored as
`local.z` by the no-pop **`fst [ebp-4]` @`0x635b51`** (`d9 55 fc`, kept live for the immediately-following
`local.z²` in the length ladder). World Z is up (a downward collision ray carries `z = −1.0` `0xbf800000`
@`0x63655a`), so `−displacement` lowers Z: a from-rest fall descends; a jump (`fall_vel < 0`) rises then
falls as `vc += g·dt` drives it back through zero.

**Gotcha — the neighbouring fall/LAND probe `0x635f80` is a different function**, not the gravity apply: its
`fchs` @`0x636014` negates a *vacated* `st0` (the `fstp [ebp-0x18]` @`0x636011` `d9 5d e8` pops first),
yielding a **horizontal** reflect direction `V.z = −0.0`, not `−fall_displacement`.

The per-substep commit then follows the normal apply: `0x616cb0` keeps `out.z` iff the body is airborne
(`MovementInfo.flags & 0x200` swim/fly OR `MOVEMENTFLAGS & 0x200800` bit 11 jump | bit 21), else zeroes it so
the body tracks terrain rather than the integrated height.

### The `+0x7c` predicted-Z is an ADD, not the apply

The plain `fadd` at `0x7c5f30` (`0x7c5f3c` `d8 41 18` `fadd [ecx+0x18]` = `fall_displacement(+0x78) + pos.z`),
stored to `[this+0x7c]` (`d9 5e 7c` fstp) at the four writers `0x616bc7` / `0x61a7d7` / `0x7c64ad` /
`0x7c696f`, is the **anti-warp / fall-damage predicted-Z reference**, consumed by the fall-distance metric
`0x7c60c0` (`(predicted_z − pos.z) + fall_vel²·½/g`, clamped ≥ 0; `+0x7c` is a near-fall-start reference Z so
the difference accumulates the distance descended). It is a *different quantity* from the applied `off.z`: an
add of the +magnitude is correct for a Z reference and opposite the applied (negated) delta — the two anchors
point opposite ways because they are two values, not a sign bug.

**`0x7c5360` (ApplyMovement) does not compose the fall kernels.** All six of its jump-table appliers
(`0x7c4ed0` / `0x7c4f30` / `0x7c4f80` / `0x7c4fd0` / `0x7c5180` / `0x7c52c0`) reference neither `fall_vel
(+0xa0)` nor any fall kernel; the dispatch keys only on the horizontal input flags
(`0x3` / `0xc` / `0x30` / `0xc0`) and falls to the default `return off=(0,0,0)` when only the falling bit
`0x2000` is set. The velocity kernel `0x7c5d20` is reached only from `0x7c72b8` (fall-start seed) and
`0x7c73f9` / `0x7c7407` (per-substep); the displacement kernel only via `0x7c5f30`→`+0x7c` and the resolvers
`0x635b00` / `0x635f80`. The gravity integrate+apply lives in the resolution-layer fall resolvers, not the
substep dispatcher.

## Apply / commit order and the reconcile leg

Per substep the commit is `0x616cb0` (build-delta → ground-z-zero → `0x634040`). After the loop, the finalize
phase re-validates (`0x616bf0`) and — **only on the spline-follow branch** (`MI != 0 AND MI.flags & 0x4
clear`, @`0x616f54`; the own-input branch never fills `carry`) — runs the **3-yard anchor dead-band
`0x616d60`** on the accumulated `carry`, committing (via `0x7c6290` anchor + `0x634030`) only when the move
is ≥ 3 yd (`dist² ≥ 9.0`), which suppresses sub-3-yard jitter. Then `0x7c5cd0` (BeginMove / land-edge) fires
on landing (same gate plus `0x200` set).

**The reconcile leg (own-input branch).** When `0x619090` has armed the window `+0x14c` (the client-predicted
position disagrees with the queued server event by ≥ ~0.0278 yd, `dist² ≥ 7.716e-4`), each own-input substep
lerps the predicted position toward the server target via `0x6191c0` (`out = target + (cur − target)·frac`,
`frac` counting down to 0 as `now → event_time`), and the reconciled delta replaces the integrated one before
the apply phase.

## The move-send cadence (CMSG_MOVE / heartbeat)

The driver reports the local player's movement to the server from this same TU; [Net](net.md) owns only the
wire transport (size-header framing + the additive stream cipher over the header+opcode span). The send side,
by trigger:

- **State-change broadcast — `0x61a820`.** On a movement-flag transition it snap-commits (`0x616d60`), runs
  the fall check (`0x619de0`), transforms to the transport frame (object vtable `[edx+0x14]`), selects the
  MSG_MOVE opcode from the flag delta via `0x619f00` (`+0x40 ^ arg` → one of
  `0xb5` / `0xb6` / `0xb7` / `0xb8` / `0xba` / `0xbb` — the start/stop forward / back / turn / strafe set),
  and emits it through the packet builder (`0x600a30` / `0x60e0a0`), opcodes `0xee` / `0xb7` / `0xcb`.
- **The packet builder — `0x600a30`** snapshots the fall data into `this+0xc70` and the swim pitch into
  `+0xc74` (gated on `MOVEMENTFLAGS & 0x200000`, default `[0x7ff9e4]`); the dedicated send `0x600b10` emits
  **opcode `0x2c9`** through the object-layer send `0x600860`.
- **Heartbeat — `0x618940` → `0x619e80`.** The trigger `0x618940` fires when the player is moving
  (`MOVEMENTFLAGS & 0x20ff`) AND the move-event queue is empty, calling `0x619e80` (heartbeat send / unlink —
  also reached from the fall trigger `0x619de0` and the per-opcode apply `0x617b60`). `MoveLocalPlayer`
  (`0x616620`) itself emits the heartbeat opcode **`0xee`** via `0x600a30` at the end of its substep run.
  Inbound: `0x6186b0` replays a received heartbeat (internal event-node opcode **`0x26`**), and `0x602c20`
  (OnMoveHeartBeat) routes to the producer `0x61a750`.

**Two opcode spaces.** The move-event queue uses small internal tags — the `0x24` / `0x25` / `0x26` producers
(`0x61a700` / `0x61a750` / …), where `0x26` is the heartbeat tag the reconcile gate `0x619090` filters on
(`ev[+0xc] != 0x26`). The wire opcodes are the MSG_MOVE family (`0xb5`–`0xbb`), the heartbeat `0xee`, and
`0xb7` / `0xcb` / `0x2c9`. The internal tag `0x26` and the wire heartbeat `0xee` are distinct numbers for the
same concept.

**The 500 ms send-deadline (`mgr+0x130`).** Distinct from the airborne-ms landing accumulator `+0x78` (also
500, a different field): the manager's `+0x130` is the local-player send-deadline. It is armed to
`clientTime + 500 ms` at `0x615b80` (`lea esi,[clientTime + 0x1f4]; mov [mgr+0x130],esi`, GUID-gated to the
local player via `0xc4da98` / `9c`) and advanced by the elapsed window in the free-advance accumulate
`0x615ae0` (`mgr[+0x130] += arg`). It is the ~500 ms pacing field for movement reports, independent of the
250 ms physics substeps: the integrator advances position every 250 ms, while the wire report is the
per-transition state-change broadcast plus this ~500 ms-paced heartbeat. The inbound move-message timing core
`0x618c30` clamps each message's timestamp into the `[500, 1500] ms` window (`0x1f4`..`0x5dc`) against
client/server clock skew.

## The resolution layer — world collision

The world-collision resolution lives in the Movement_C TU, rooted at the entry **`0x634040`**: normalize the
displacement into unit dir + speed, then a substep loop while `flags & 0x200f` and the `ms·0.001` budget
remain. Each substep runs the swept world-query **`0x633840`**, then dispatches by mode — **TRANSPORT
`0x634640`**, **FALL `0x635b00`**, **WALK `0x6367b0`** — and commits through the `0x61a820` / `0x7c4930` seam,
writing `+0x58`.

### The swept query — `0x633840`

The query builds the swept AABB into the **`0xc4e5a0`** global scratch (`min.z` raw), with three
mode-shaped growth arms:

- **FALLING:** slope-tilt growth.
- **SWIMMING:** `× 0.5 / √2`.
- **WALKING:** step-rise `max(collHeight·tan50°, r + 1/720)`; step-down `min.z −= collHeight + dist·tan50°`.

It then adds the **±1/720 skin**, selects the mask (`0x6315f0`), and fires the trace **`0x6721b0`** — the
boundary to terrain/models, which produce the `0x34`-byte hit records. Per hit it flips liquid
normals/distances and rotates transport records to world.

### The WALK resolver — `0x6367b0`

Flattens dir to horizontal and slides via the **earliest-contact finder `0x632ba0`** (the swept-prism
geometry core): the 9-plane open-bottom k-DOP, the Sutherland-Hodgman clip with the ±1/720 band, per-face
time-of-impact with the **−1/36** backface pass, the bevel collector, and a 5-axis-face scan accumulating the
banded manifold (the binary's own `idx_out` surfaced). The prism is built by **`0x631be0`** — the 9 vertices
plus a position/size-constant **face-index table** (5 quad faces = 4 side walls + 1 top cap, then 4 apex
triangles; the byte table spans `0x631de6`–`0x631e59`).

Each blocking hit is classified walkable (cos50° solid / cos80° terrain-water, by `0x5fa550`) → feature-
classify + speed-select reproject; or non-walkable → the **multipass step-up resolver `0x636100`** (back-
probe, up-probe, forward-step, fall/land `0x635f80`, down-snap; returns 0/1/2). Budget exhaustion
ground-settles (the **1.8493990**-scaled down-probe, the water `+1.0` leg); the water-surface arm probes DOWN
(`−1.0` @`0x636fe0`) and snaps `+z` by the clamped swim displacement.

### The FALL resolver — `0x635b00`

Solves the drop via the physics seam and either free-falls + land-snaps (`0x633240`: grounded iff fallen
≥ 1/9 yd below the start OR the `+0x78` airborne-ms accumulator ≥ 500) or runs the fall integrator `0x635600`
(snapshot/restore via `0x635ea0` / `0x635f10`, the walk-slide `0x6351a0` / fall-step `0x635450`
sub-resolvers, ≤ 5 reflects, the landing-damage leg). `0x635f80` re-enters `0x635600` over fall-time chunks
(the `V.z = −0.0` vacated-st0 reflect vector).

**Gotcha — the FALL collide path has no in-layer terminator.** A ground-abort multi-chunk sequence drives the
fall accumulator `T` up until `solved ≤ T` (fall_step leg 0 → `eax = 0`), and `0x635f80` then re-enters
`0x635600` **without bound** — the binary's own loop has no internal terminator on this path (it loops
indefinitely in isolation). The FALLING-clear exit lives across the movement-state seam; the binary's
in-image callers bound the condition, so **any reimplementation must impose the same bound** on this path.

### The TRANSPORT resolver — `0x634640`

Runs the same sweep in transport frame with ≤ 5 reflect (`0x634300`) iterations and the liquid second pass.

## The seam map — owned vs delegated

**Resolved within the movement/collision layer:** the driver primitives, the falling-physics pair, and the
entire resolution layer — the entry, the query, the three mode resolvers, the geometry core, and the
helper/sub-resolver tiers. Two cross-cut geometry kernels also belong here:

- **`0x7c29f0`** — the Möller-Trumbore segment-triangle kernel (eps `0.002`); every BSP face test funnels
  through it.
- **`0x6dc900`** — segment-segment distance squared (under `0x6b9950`).

**Consumed across seams:**

- **The world trace `0x6721b0`** — [Terrain](terrain.md)'s CMap segment/DDA drivers (`0x69c320`, the
  liquid-hit flush `0x671cc0`) and [Models](models.md)' WMO BSP machinery (`0x6bc370` descend, `0x6b88e0`
  leaf intersect, `0x6b91c0` / `0x6b92b0` drivers, `0x7089c0` M2Scene ray) produce the `0x34`-byte hit
  records.
- **The ClntObjMgr accessors:** `0x630ac0` (the transport 4×4 matrix), `0x5fa550` (the object-type
  predicate), `0x617430` (the `+0xb8`-or-`2.0` scalar consumed as collision height).
- **The movement-state seam:** `0x7c61f0` StartFalling, `0x7c5cd0` BeginMove, `0x6173b0` SetFacing,
  `0x7c61b0` swim-terminal, `0x61a820` / `0x7c4930` the broadcast/commit pair, `0x615ae0` free-advance.
- **cmath** (vector/matrix/plane helpers) and **core's event bus** (registration `0x401af1` via `0x41fca0`,
  dispatch `0x4245b0`).

On-transport static overlap transforms the position and displacement by the transport matrix `M`:
`pos := affine_xform(M, ·)`, `disp := rot_dir(M, ·)`.
