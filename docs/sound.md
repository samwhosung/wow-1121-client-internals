# Sound — selection/scheduling over the FMOD mixer

The 1.12.1 client does not mix audio itself. The mixer — 3D spatialization, distance attenuation,
doppler, reverb, and sample/stream playback — is **FMOD** (`fmod.dll`, the FSOUND C API). What the
client owns and computes is the layer on the WoW side of the FMOD call: which sound plays when, the
coordinate and parameter math fed to FMOD before it produces the audible mix, and channel / priority /
volume management. This chapter covers that owned layer and the boundary it sits behind.

## The FMOD boundary — what is delegated

`WoW.exe` imports `fmod.dll` and drives the FSOUND 3D-audio API (53 FSOUND imports). The functions it
calls include:

- `FSOUND_3D_SetAttributes`, `FSOUND_3D_Listener_SetAttributes` / `GetAttributes`
- `FSOUND_3D_SetDistanceFactor` / `SetDopplerFactor` / `SetRolloffFactor`
- `FSOUND_Sample_SetMinMaxDistance`
- `FSOUND_SetVolume` / `SetFrequency` / `SetPaused`
- `FSOUND_Reverb_SetProperties` / `SetChannelProperties`
- the `FSOUND_Stream_*` family

Spatialization, attenuation rolloff, doppler pitch-shift, reverb, and per-sample mixing are FMOD's: WoW
hands it listener/source attributes and min/max distances, and FMOD computes the result.

A decisive negative tells you where the line is: WoW imports the full 3D suite but **no
`FSOUND_SetPan`, no `FSOUND_PlaySound`, and no `Surround`**. The client therefore cannot and does not
compute pan, attenuation, or doppler directly — it feeds FMOD 3D positions and FMOD computes the
audible mix.

## Entry point and structure

There is **no `CSound*` singleton** — the manager is C-style file-scope globals (the FSOUND C-API
idiom). Three anchors define the subsystem:

- **Subsystem init: `0x456fe0`** ("SoundMgr::Initialize"), sole caller `0x402ae9` — one slot in the
  WinMain subsystem-init call-run. It registers the sound CVars, gates on `-nosound`, calls the FMOD
  device init, and builds the `SoundEntries` caches.
- **FMOD device init: `0x7a4330`** → `FSOUND_Init` @ `0x7a45f5`. Configures memory / output / driver /
  mixer / file-callbacks (streaming is routed through WoW's MPQ layer) and sets the distance, doppler,
  and rolloff factors.
- **Per-frame pump: `0x7a4ad0`** ("FmodSystem::Update") → the per-channel distance cull `0x7a5000` →
  `FSOUND_Update` @ `0x7a4b4a`. The pump is reached **only via a callback**: `0x7a4330` registers the
  trampoline `0x7a4850` on the core async-task register `0x41fc90`
  (`7a474e: mov edx,0x7a4850; mov ecx,5; call 0x41fc90`), with the type-2 focus/pause handler
  `0x7a4860` registered alongside.

The FMOD-binding wrapper layer is the translation unit **`0x7a4180`–`0x7a6970`** — one thin wrapper per
FSOUND op. In-band sound logic (`0x453000`–`0x464000`) calls those wrappers, never the IAT thunks
directly. Sound proper begins around `0x456820`; the low edge `0x453000`–`~0x4568xx` is shared
vector/RNG math owned elsewhere, not sound code (`0x4531e0` `CRandom::uint32`, `0x4549a0`
`C3Vector::Set`, `0x456280` vec2-normalize).

## The coordinate transform — WoW → FMOD

WoW and FMOD use different axis conventions, so the client remaps coordinates itself before every 3D
attribute call. The transform is an axis swap plus a Y-negation:

```
(x, y, z)_WoW  →  (−y, z, x)_FMOD
```

It is applied to position and velocity (and, for the listener, the forward/up orientation vectors as
well):

| VA | Role |
|---|---|
| `0x7a5eb0` | listener 3D attributes → `FSOUND_3D_Listener_SetAttributes` |
| `0x7a5b10` | source 3D attributes → `FSOUND_3D_SetAttributes` |
| `0x7a5f50` | inverse transform, FMOD → WoW (negates Y) |
| `0x7a4e00` | direction-only 3D position synthesis: `1/√` normalize + scale + Y-negation |

The listener position is *sourced* from camera/player world state and fed in at `0x482d70`; sound only
transforms a position once it is given one. The per-emitter source position is likewise sourced from
world/object state.

## Distance, culling, and near-field attenuation

The per-frame pump runs a squared-distance test per channel in `0x7a5000`:

```
d² = Σ (chanᵢ − listenerᵢ)²
cull / virtualize the channel when  d² > maxdist²
```

In the falloff band the client computes its own near-field rolloff (stored at channel `+0x78`), layered
on top of FMOD's own 3D attenuation:

```
atten = 1 − clamp( √d² − maxdist·k₁ ) / (maxdist·k₂)
```

The `√` here is the **precise square root**, not the `rsqrtss` reciprocal-sqrt approximation — this
matters for reproducing the exact attenuation value.

Two more distance gates exist outside the pump:

- `0x45cdf0` — selection-time gate `d² ≤ maxDist²` (decide whether to start the sound at all).
- `0x458380` — environment-sound gate `Σ pos² ≤ r²`.

A per-frame source slew at `0x462960` moves a source toward its target without snapping:

```
cur += (Δ / |Δ|) · min(|Δ|, maxstep)
```

## Volume

Volume is a chain of small curves and category mixes, each clamping then applying a fixed-point or
linear scale before the value reaches `FSOUND_SetVolume`. The category factor (`cat`) is selected from
master / SFX / music / ambient flag bits.

| VA | What it computes |
|---|---|
| `0x7a5dc0` | channel master/category mix: `cat · ch_vol · ch_atten · k` |
| `0x7a5d70` | per-channel volume curve: `(v·k + b) >> 14` |
| `0x7a5730` | channel volume rate: clamp level to `[0, 255]`; `+0x48 = level / (param · k)` |
| `0x7a5a50` | stream volume rate: `curve / (p · k)` |
| `0x7a5da0` | volume-ramp increment |
| `0x7a6500` | SFX master-volume curve: clamp to `[≈0, ≈1]`; FMOD value `(uint)(v·k + b) >> 14 & 0xff` |
| `0x457960` | music crossfade lerp |

The volume-category slider arithmetic (master/SFX/music/ambient slider → the scalar handed to FMOD) is
the trio `0x7a6390` / `0x7a6400` / `0x7a6500`.

## Sound selection, variation, and pitch

A `SoundEntries.dbc` row can carry several interchangeable samples; the client picks one and applies
per-shot random variation to volume and pitch.

- **Variation pick** — `0x45bb70`, a weighted-random selection over the entry's variants (integer
  arithmetic over the engine RNG).
- **Per-shot volume variation** — `0x458c60`, via the engine PRNG, clamped to `[0, 1]`.
- **Playback pitch** — `0x458da0`: `(k · (rng + 0x55)) / 100`.
- **Frequency / pitch scale** — `0x7a57b0` (and the `vol · const / 255` scale at `0x7a5730`).

Per-entry **cooldown** is throttled against the millisecond clock, so a repeatedly-triggered sound does
not retrigger every frame.
