# Game time — the server-synced day/night clock

The client keeps a single server-authoritative clock that tracks the position of the day/night cycle: how far through a 24-hour day the world currently is. It is a global singleton, constructed at C++ static-init, advanced by a configurable time-scale, and re-anchored whenever the server sends a time packet. Its consumers are the UI clock display, the console, the Lua/glue layer, and — through a bridge — the [lighting](lighting.md) subsystem, which keeps its own separate, per-frame-interpolated clock and re-syncs to this one only on a time packet.

This subsystem owns the clock *model*; the wire format that feeds it belongs to [net](net.md), and the millisecond tick and intrusive-list primitives it builds on belong to [core](core.md). The class methods occupy `[0x643300, 0x643c00)`.

## The GAMETIMECBSTRUCT singleton (`0xce8538`)

All game-time state lives in one global object at `0xce8538`. Relevant fields:

| Offset | Field |
|---|---|
| `+0x18 .. +0x6c` | embedded `TSExplicitList<TIMESTAMPSTRUCT>` observer list — callbacks fired on each minute boundary |
| `+0x2c` | time-scale, in game-minutes per real-second (default `1/60`) |
| `+0x38` | tick baseline — the value of `0x42c010()` captured at the last anchor |
| `+0x3c` | minute offset — the absolute game-minute count at the baseline |

The companion id `0xce853c` is passed alongside `0xce8538` to the registered callbacks (see the bridge below).

## The day-phase computation (`0x643a70`)

The core output — the day/night cycle position — is produced by `0x643a70`:

```
phase = frac( ((tick − [+0x38]) · 0.001 · [+0x2c] + [+0x3c]) mod 1440 ) · (1/1440)   →  [0, 1)
```

Reading the units left to right:

- `tick` is the millisecond tick `0x42c010()`; subtracting the baseline `[+0x38]` and multiplying by `0.001` gives real seconds elapsed since the anchor.
- `· [+0x2c]` (time-scale, game-min/real-sec) converts that to game-minutes elapsed.
- `+ [+0x3c]` (minute offset) yields the absolute game-minute count.
- `mod 1440` reduces to the minute-of-day (1440 = minutes in a 24-hour day).
- `· (1/1440)` normalizes to a `[0, 1)` day phase.

This is the value the lighting bridge publishes; everything downstream of "how far through the day are we" reads from here.

## Advancing, scaling, and syncing the clock

The integer side of the clock works in whole game-minutes within a 1440-minute day:

- **`0x6434d0` SetGameTime** — installs a new clock value (called as `0x6434d0(1, &time)` by the net handlers).
- **`0x6435a0` minute-accumulate** — accumulates elapsed time, applies the `__ftol` float-to-int correction, and runs a reduce loop; the per-minute advance is `0x643ad0`.
- **`0x643ad0` advance-one-minute / HH:MM** — advances the minute count as `(cur+1) mod 1440` (via `idiv 1440`) with day-rollover, and decomposes into hours:minutes.
- **`0x6436c0` GameTimeSync** — applies a server→local game-minute differential (`mod 1440`) plus the per-minute advance, reconciling the local clock with the server's authoritative minute.
- **`0x643820` SetTimeScale** — sets `+0x2c`, clamped to `[1/60, 60]`. **`0x643810`** is the corresponding getter.

## How the server sets it (net)

The clock is driven by two SMSG opcodes, dispatched through net's registrar `0x5ab650`:

| Opcode | Meaning | Handler | Action |
|---|---|---|---|
| `0x42` | `SET_TIMESPEED` | `0x6c5e80` | parse, `SetGameTime 0x6434d0(1, &time)`, set time-scale, then the bridge |
| `0x43` | time-sync | `0x6c6010` (family `0x6c6120`) | parse, `SetGameTime`/`GameTimeSync`, then the bridge |

The clock model is game-time's; the wire decode is net's.

## Consumers

Roughly 21 cross-references read the singleton `0xce8538`:

- **UI** — the day-phase read at `0x482bdd`, plus `0x457811` and `0x483dd7`.
- **Console** — `0x515ef4`.
- **The Lua/glue cluster** at `0x6c5xxx`.
- **Lighting**, via the bridge `0x6c5f50` below.

## The lighting bridge (`0x6c5f50`)

`0x6c5f50` is the publish point from game-time into [lighting](lighting.md). It is glue code (it does no day-phase arithmetic of its own — it calls game-time's `0x643a70`), but it is the only path by which the two clocks are reconciled. Three facts matter:

**It runs per time-packet, not per-frame.** Its only two callers are the net SMSG handlers above: the `0x42` `SET_TIMESPEED` handler `0x6c5e80` and the `0x43` time-sync handler family (`0x6c6010` / `0x6c6120`). Each parses its packet, sets game-time's clock via `0x6434d0(1, &time)`, then calls the bridge. It never runs from the render loop.

**It is a publish/copy, not a recompute.** The bridge:

1. Fetches lighting's `DNState` pointer (`0x6d48b0` → `0xce9b60`).
2. Calls game-time's own day-phase `0x643a70` and stores the result straight into lighting's clock `0xce9b64` (= `DNState+0x4`).
3. Writes two sibling `DNState` fields: `0xce9b60 ← 0x6424c0` and `0xce9b68 ← (float)0x642320`.
4. Runs the world/weather + scene-fog refresh `0x66ff60(1, 0)`.
5. Fires the four registered time-changed callbacks — the table at `0xce85a8`, stride 4, each called with `0xce8538` / `0xce853c`.
6. Formats and publishes the clock string (`0x642c80` → `SStrPrintf 0x64a7f0` → `0x63cb50`).

**Between syncs, lighting advances its own clock.** `0xce9b64` is written inside lighting's own code (`0x6d1be9`) and read by the `DNState` track and sky evaluators every frame. The bridge re-anchors that client-side, per-frame-interpolated clock to the server's authoritative time on each time packet — which is why the two subsystems keep separate clocks at all.
