# WoW 1.12.1 Client Internals

**What the World of Warcraft 1.12.1 (build 5875) game *client* computes from its bytes — reverse-engineered, with the math, the constants, and the structures written down.**

This is a knowledge resource. It documents the *behaviour* of the retail vanilla client: how it
renders terrain and sky, how it lights a scene, how it decodes the wire protocol, how movement and
collision resolve, how it lays out text — the logic that defines how the game looks and feels. Every
claim is traced to a specific function in the client binary, by address.

It exists because nobody else writes this down. The public WoW ecosystem is split between **server
emulators** that reimplement game *logic* (vmangos, CMaNGOS, TrinityCore) and **file-format parsers**
that read the *assets* (warcraft-rs, StormLib, the wowdev.wiki). Neither answers the question this
resource answers: **given the bytes, what does the client itself compute?**

## What this is / is not

- **It is** a per-subsystem written account of the 1.12.1 client's internal mechanisms.
- **It is not** a game, a client, a server, or an asset parser. It ships no Blizzard code or assets.
- **It is for** people implementing their own client or tooling, emulator developers who need
  client-accurate behaviour, and anyone studying how the vanilla client works.

## How to read a chapter

Chapters cite **virtual addresses** like `0x6d2260`. These are addresses in the **en-US 1.12.1
build 5875 `WoW.exe`** — the specific retail binary this work was derived from. They are provenance:
load your own copy of that binary in a disassembler (Ghidra, IDA, objdump) and you can find the same
function and confirm what is written here. You do **not** need the binary to read and use the
knowledge — the formulas and structures stand on their own — but the addresses are why you can trust
it rather than take it on faith.

A few conventions you'll see (full list in [GLOSSARY.md](GLOSSARY.md)):
- Addresses are absolute VAs in the 5875 image.
- Struct fields are given as offsets, e.g. `DNState+0x40`.
- `f32`/`f64` distinctions are called out where the client's float precision is observable (it
  often is, and it matters for byte-faithful reproduction).

## Chapters

### World & rendering
- [Terrain](docs/terrain.md) — the ADT world surface: mesh, textures, liquid
- [Lighting](docs/lighting.md) — time-of-day light, sky dome, fog, weather
- [Models](docs/models.md) — M2 and WMO geometry and materials
- [Animation](docs/animation.md) — M2 skeletal animation
- [Character model](docs/character-model.md) — compositing a player from body + equipment

### Rendering engine
- [Graphics device](docs/graphics-device.md) — the render device and state submission
- [Camera](docs/camera.md) — the view transform: perspective / lookAt / frustum
- [Rendering math](docs/rendering-math.md) — projection, frustum, ray/picking math
- [Full-frame effects](docs/full-frame-effects.md) — glow/bloom, the death-fade desaturation, the post-process passes

### Movement & simulation
- [Collision & movement](docs/collision.md) — player vs world geometry, the movement controller
- [Object model](docs/object-model.md) — the CGObject hierarchy, UpdateFields, the object manager
- [Spells](docs/spells.md) — client cast lifecycle, cooldowns, targeting, spell visuals
- [Minigame](docs/minigame.md) — the client model of a server board game (TicTacToe)

### Networking
- [Network protocol](docs/net.md) — auth + world wire protocol, the session

### User interface
- [UI engine](docs/ui.md) — the FrameScript/Lua binding layer and the widget/text engine
- [Console](docs/console.md) — the in-game console and command registry
- [Glue screens](docs/glue.md) — login → realm list → character select/create
- [Minimap](docs/minimap.md) — the circular HUD minimap
- [World text](docs/world-text.md) — nameplates, floating combat text, raid icons
- [Loading screen](docs/loading-screen.md) — the world-enter loading bar and taxi animation
- [Text layout](docs/text-layout.md) — per-glyph advance, wrap, measure, glyph quads

### Audio
- [Sound](docs/sound.md) — sound selection/scheduling over the FMOD mixer

### Assets & data
- [MPQ asset VFS](docs/mpq.md) — the archive filesystem everything loads through
- [DBC tables](docs/dbc.md) — the client database tables
- [DBCache](docs/dbcache.md) — the server-streamed record cache
- [Image decode](docs/image.md) — BLP2 and TGA decoders, palette/DXT/dither

### Core & configuration
- [Core loop](docs/core.md) — the per-frame event scheduler and event bus
- [CVars](docs/cvars.md) — the console-variable configuration registry
- [System messages](docs/sysmsg.md) — the structured client-message service
- [Game time](docs/game-time.md) — the server-synced day/night clock

### Math & determinism
- [Math primitives](docs/math-primitives.md) — matrix/vector LA, software fast-math, the PRNG, splines

## Provenance & method

This account was reverse-engineered from a single legally-obtained copy of the en-US 1.12.1 (5875)
`WoW.exe` using disassembly and a differential test harness: candidate transcriptions of each function
were executed against the original under CPU emulation and compared output-for-output until they
matched bit-for-bit. So the mechanisms here are not guesses or guesswork from behaviour — they are
what the binary provably computes, over a declared range of inputs, with any non-reproducible cases
(hardware-approximate instructions, externally-fed state) called out explicitly.

The harness, the proofs, and the working notes live in a separate private workshop; this repository
is the distilled, human-readable result. The proof artifacts themselves — the bit-exact reference
implementations and the differential test harness that gates them — may be released separately at a
later date. An AI assistant was used throughout to help transcribe and cross-check against the binary;
every claim was gated by the objective differential test, not accepted on the model's word.

## Legal

World of Warcraft, Warcraft, and Blizzard are trademarks of Blizzard Entertainment, Inc.; this is an
independent work, not affiliated with or endorsed by Blizzard. Full trademark, provenance, and
no-redistribution details are in [NOTICE](NOTICE).

## License

The documentation in this repository is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE) — use it, quote it, build on it;
just credit the source. Any code snippets are additionally available under the MIT license.
