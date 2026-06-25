# System messages — the structured client-message service

The client's structured-message ("SysMsg") service — `ENGINE\Source\Services\SysMessage.cpp` — is the
diagnostic/system-message facility that roughly 35 producer sites emit through (e.g. `"NOSPELLIDFOUND|%d"`,
`"Camera view %s"`, object-cache traces). It is small: it formats a message string, validates and classifies
it by a small type code, and looks up that type's colour and on-screen duration. The actual on-screen
message store, display, and fade are delegated to an external store; in the retail 5875 build the print path
formats the string and then discards it (display compiled out).

## Translation unit and startup

The translation unit is `[0x44cb20, 0x44cd80)` (~19 functions), with `__FILE__` at `0x835934`. Its init is
wired during startup at `0x402ad0`, immediately after the [console](console.md) (`0x402350`).

Note that the lower band `[0x44c040, 0x44cb20)` is a *separate* `Texture.cpp`
`TSExplicitList<CTextureHash>` TU with zero call edges into sysmsg — it is not part of this subsystem.

## The producer path

```
sys_msg(type, channel, fmt, ...)   0x44cbd0
    → format    0x44cb90   vsnprintf into a 256-byte buffer
    → validate  0x44cb20   accept only if  text != 0 && type < 4
```

The `type < 4` gate matches the four records in the colour table below.

## The colour primitive

The one look-defining computation the service owns is the per-type colour normalize, `color_lookup`
(`0x44cca0`). For a given type index it converts the table's stored 8-bit channels to normalized floats:

```
*{r, g, b} = (float)(u8) colorTable[idx].{R, G, B} · (1/255)
```

In the binary this is `fild ; fmul [0x8026c8] ; fstp`, repeated for each of R, G, B, where `0x8026c8` holds
the reciprocal `0.0039215689` (= 1/255 as f32).

The result is consumed by the external glue status renderer (`0x40924f`), which re-packs the floats back
`×255` into BGRA. The round-trip through floats is the binary's actual path; an implementer reproducing the
on-screen colour should follow it rather than passing the bytes through directly.

## The SYSMSGCOLOR table

The colour/duration data is a 4-record table at `0x835908`. Each record is 8 bytes: `{u8 R, u8 G, u8 B, u8
pad, f32 duration}`.

| idx | R | G | B | duration (s) | colour |
|---|---|---|---|---|---|
| 0 | 255 | 255 | 255 | 10.0 | white |
| 1 | 255 | 255 | 127 | 15.0 | yellow |
| 2 | 255 | 127 | 127 | 15.0 | red |
| 3 | 127 | 255 | 255 | 20.0 | cyan |

The `duration` field is read by the f32-duration getter `0x44cd00`. The `idx < 4` validation gate
corresponds exactly to these four records.

## The scalar state machine

The remaining state is plain integer bookkeeping — loaded, stored, and clamped, with no look-defining arithmetic:

| State | Address | Init | Notes |
|---|---|---|---|
| enabled | `0x835928` | 1 | |
| window lo | `0xb05d30` | 0 | set with a `lo ≤ hi` clamp |
| window hi | `0x83592c` | 3 | set with a `lo ≤ hi` clamp |
| sel | `0x835930` | -1 | |

Entry points: init `0x44cd10` (called from startup `0x402ad0`); disable `0x44cd40`.

## Delegated work

- **Storm runtime**: `vsnprintf` (`0x73f6ad`), `SErrSetLastError` (`0x64e850`), `SMemFree` (`0x646430`).
- **The on-screen message store, display, and fade**: a 268-byte ring buffer at `0x882e28`, owned by the
  later CGlue/world TU around `0x409xxx`. The sysmsg service's `color_lookup` output feeds this store; the
  store itself, and the duration/fade rendering that consume the colour/duration table, live in the
  [glue](glue.md) layer. In the retail 5875 build the print path formats the message and then discards it —
  the display side is compiled out.
