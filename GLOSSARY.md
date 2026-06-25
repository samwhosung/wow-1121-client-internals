# Conventions & glossary

A short guide to how chapters are written. This resource avoids project-internal jargon; what remains
is standard reverse-engineering and WoW terminology.

## Notation

- **Virtual address (VA)** — a hex address like `0x6d2260` is the location of a function or datum in
  the **en-US 1.12.1 build 5875 `WoW.exe`**. It is provenance: find the same address in your own copy
  to verify the claim.
- **Struct offset** — `DNState+0x40` means the field at byte offset `0x40` within the structure named
  `DNState`. A bare global like `[0xce9b60]` is a fixed address in the binary's data segment.
- **`f32` / `f64`** — single- vs double-precision float. Where a chapter says a value "stays in `f64`"
  or is "stored as `f32`", that precision is observable in the binary and affects the exact result. It
  is recorded because byte-faithful reproduction depends on it.
- **GPU/render state `0xNN`** — a fixed-function render-state slot the client sets (fog, etc.).

## WoW terms used throughout

- **ADT** — a terrain map tile (the world surface chunk format).
- **M2** — the model format for creatures, characters, doodads, and animated objects.
- **WMO** — the world map object format for large structures (buildings, caves, cities).
- **MPQ** — Blizzard's archive format; all game assets are stored in `.MPQ` files.
- **DBC** — `DBFilesClient\*.dbc`, the client's static database tables (spells, items, light
  definitions, etc.).
- **BLP** — Blizzard's texture format; **DXT** — the block-compression schemes BLP can wrap.
- **FrameScript / FrameXML** — the client's Lua + XML UI system.
- **CGObject / CGUnit / CGPlayer / …** — the client's object class hierarchy (the `C`-prefixed C++
  classes recovered from the binary).
- **UpdateFields** — the server-replicated object state, a flat indexed field array per object.

## Reading the addresses against your own binary

The cited addresses assume the image base and layout of the retail en-US 5875 `WoW.exe`. Other locales
or repacks may differ. If you have the matching binary, every function address here resolves to the
function described; if it doesn't, you likely have a different build.
