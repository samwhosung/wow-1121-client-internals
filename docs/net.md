# Network protocol — auth + world wire protocol

How the 1.12.1 client talks to the realm: a logon flow against the auth (realmd) server that
runs an SRP6 password exchange, a framed-and-ciphered world stream, and a net↔state bridge that
hands decoded packets from a dedicated network thread to the main thread. This chapter covers the
session lifecycle, the threading seam, the wire framing and stream cipher, the SRP6 crypto, and the
on-the-wire layouts of the opcodes the client parses. The raw TCP socket below the framing layer is
the host's concern (Storm OsNet, `0x438240`–`0x4420b0`); everything above it — framing, cipher,
dispatch, the protocol — is described here.

## Entry points and threads

The networking has four roots spread across two threads. There is no single top function; all four
anchor the system.

**1. Construction / lifecycle.** Glue-init `0x402b0c` → singleton factory `0x5ab1d0` (the object at
`DAT_00c28128`) → `0x5aa370` → `0x5b3a80`. The latter wires the base constructor `0x537410`,
`Initialize` `0x537600` (which lazily calls `0x5b7080` `InitOsNet` → `0x5bd5f0` and starts the
network thread via `0x5bd910`), and registers ten SMSG handlers through
`0x537a60(opcode, 0x5b3ea0, this)` for opcodes `0x1ec`, `0x1ee`, `0x2ef`, `0x3b`, `0x3a`, `0x41`,
`0x4d`, `0x4f`, `0x4c`, `0x3c`.

**2. Main-thread receive pump.** `0x403620`, registered on event-bus **category 6** at `0x402b84`
(via core's `0x41fca0`). The pump runs `0x5b3db0` `PollEventQueue` → `0x538040` → `0x5384d0`, which
drains the critical-section-guarded event queue. Data events route through `vtable+0x30` =
`0x537c50` → `0x537aa0` (the opcode dispatcher over the `+0x74`/`+0xd64` tables) → `0x5b3ea0`
(`RealmConnection::MessageHandler`, by its own debug string) → the per-SMSG parsers.

**3. The dedicated "Network" thread.** `0x5bd910` `Start` (threads named "Network" / "Net
Thread %d") → `0x659ac0` → `0x64bd20` → `CreateThread`(shim `0x64bc20`) → proc `0x5bda10` →
`0x5bdfe0` → `0x5bcb00` `PlatformRun`: a select/recv loop calling WSOCK32 by ordinal (18 = `select`,
151 = `__WSAFDIsSet`, 16 = `recv`), exiting only on the stop flag. **Socket ingestion lives here;
parsing and dispatch live on the main thread (root 2).**

**4. Send boundary.** `0x5ab630` (the game-wide fan-in) → `0x5379a0` `NetClient::Send` (gated on
state `+0x70 == 6`, cipher block at `+0x120`) → `0x5b5630` (size-header framing, outbound cipher,
WSOCK32 `send`). Sends run synchronously on the caller's thread, with a 1-byte self-pipe wake
(`0x5bcad0`) to break the network thread out of `select`.

## The net thread ↔ main thread seam

The connection base class plus a typed event queue form the seam between the network thread (which
reads the socket) and the main thread (which parses). Three locks protect it: a global registry lock
at `0xc2a314`, a per-queue lock at `queue+4`, and a per-connection stats lock at `+0x1ac0`.

Producers on the network thread enqueue **deep-copied typed nodes**, tagged `0x12` (data), `0x13`,
`0x14`, `0x15`, `0x16` (abort). The main-thread drain `0x5384d0` dispatches each via vtable slots
`+0x30..+0x3c`, using interlocked deferred-delete refcounting (`+0x1a5c` / `+0x1a60`). The opcode
dispatcher `0x537aa0` then routes the u16 opcode through the **828-slot** handler/context tables at
`+0x74` / `+0xd64`.

PONG `0x1dd` is the one opcode handled inline on the network thread. The ping it answers,
`CMSG_PING`, has the layout `{ u16 0x1dc, u32 seq, u32 lastRtt }`.

All packet bytes are composed/decomposed by the engine's `CDataStore`, not by hand in the net code.

## Wire framing — the big-endian size-header varint

Every world frame is prefixed by a variable-length big-endian size header. `N` counts the opcode
plus the body:

```
N <= 0x7fff  ->  2-byte header
N >  0x7fff  ->  3-byte header, top bit of the first byte set as a flag
```

- **Encode**: `header_encode` at `0x5b5450` (selects the form, writes the big-endian length).
- **Decode / deframe**: inline in the recv path `0x5b65f0`, at `0x5b6730`. It form-selects on
  `buf[0] & 0x80`, then computes `frame_len = N + hdr_len`. When fewer than `hdr_len` bytes are
  buffered it returns the `0xffffffff` incomplete-header sentinel and waits for more.

On the receive side the SMSG opcode is **2 bytes**.

## The stream cipher

The world stream is enciphered with an **additive feedback cipher (not RC4)**. Only the
header+opcode span is ciphered; the body travels in clear.

```
send:  c = (key[j] ^ p) + prev      prev = c   (ciphertext feedback)
recv:  p =  key[j] ^ (c - prev)      prev = c
       j = (j + 1) % 40
```

The key is the 40-byte block at `NetClient+0x48` — the session key `K` from SRP6. The cipher is
armed by `0x5b73c0`; the key, length, and ciphered spans are set by `NetClient::Send`. Implementing
functions: `cipher_send` `0x5b5630` / `0x5b5af0` / `0x5b5dd0`, `cipher_recv` `0x5b65f0`. Send and
recv are exact inverses.

The send and recv socket/framing/cipher live in `WowConnection`; the network thread, select loop,
and worker pool live in `WowConnectionNet`; realm-address configuration lives in `AddressList`.

### Outbound movement rides this path

`CMSG_MOVE` and the movement heartbeat are **consumers** of this transport, not part of the net
surface. The `MSG_MOVE` state opcodes (`0xb5`–`0xbb`), the heartbeat `0xee`, and the move packet
`0x2c9` are built and ~500 ms-paced by the movement driver (see [Collision](collision.md), "the
move-send cadence"). They ride the framing + additive cipher like any other outbound opcode. Net
owns the transport; collision owns the cadence.

## The realm session

`RealmConnection` (the SMSG dispatcher), `ClientServices` (the `COP_*` / `AUTH_*` operation API over
the singleton), `FriendList`, and `Login` / `CLoginMgr` (the Grunt-front state machine) make up the
session layer. Key struct sizes: `RealmConnection` fields `+0x1adc..+0x1b20`, `FriendList` `0x728`,
`Logon` `0x1f8`.

The constructor `0x5b3a80` registers all ten SMSG handlers on `0x5b3ea0`
(`RealmConnection::MessageHandler`), which dispatches by opcode to the per-SMSG parsers.
`CMSG_AUTH_SESSION` (opcode `0x1ed`) is built at `0x5b4000` (build `0x16f3`; account + seeds + a
40-byte session-key digest = `SHA1`, the same `NETCLIENT+0x48` key that arms the stream cipher).

### SMSG parser ABI and environment

A connection-coupled SMSG parser is:

```
__thiscall( ecx = RealmConnection*, [ebp+8] = ?, [ebp+0xc] = ?, [ebp+0x10] = CDataStore* ),  ret 0xc
```

The `CDataStore` (`[ebp+0x10]`) holds its size at `+0x10` and read cursor at `+0x14`. The getters are
**cds-relative only — no global**: `0x418cb0` `GetInt8`, `0x418eb0` `GetInt32`, `0x4190b0` `GetInt64`,
`0x419130` `GetFloat`, `0x4191b0` `GetString` (C-string, ≤ `0x30`). Each reads `[cds+0x14]` against
`[cds+0x10]`.

`RealmConnection*` is a ≥ `0x1b00`-byte object. The parser **virtual-calls a notify sub-object at
`this+0x1adc`**: it loads `[this+0x1adc]`, then its vtable, and calls `[vtable+8]` **unconditionally**
in the tail (`0x5b424a`), plus `[vtable+0x14]` conditionally (`0x5b41ce`, with a debug-string arg)
and `[vtable+0xc]` in a sibling. (These are the `ClientServices` status-notify virtuals; they return
void.)

### SMSG_AUTH_RESPONSE (`0x5b41b0`, opcode `0x1ee`)

The least-coupled parser: it reads **no global**, only the cds and the `this+0x1adc` notify vtable.
Parsed output is written back into `this`: queue-flag at `+0x1aec`, fields at `+0x1af0` / `+0x1af4` /
`+0x1af8` / `+0x1afc`.

(Other parsers do read globals — e.g. the sibling `0x5b4260` calls `0x51da70`, which reads the global
`[0xbe1b6c]` — so each parser's global dependencies must be checked individually.)

### SMSG_CHAR_ENUM (`0x5b42a0`, opcode `0x3b`) and the AUCHARACTER_INFO layout

Same ABI. Here the notify-vtable slots used are `[vtable+0x14]` (entry-notify) and `[vtable+0x10]`
(finalize-notify). The parser manages an `AUCHARACTER_INFO` array via a 3-dword control at
`this+0x1ae0`:

| Offset | Field |
|---|---|
| `+0x1ae0` | self pointer |
| `+0x1ae4` | count |
| `+0x1ae8` | data pointer |

The leading `GetInt8` is the character count (≤ 10; a value `> 10` zeroes the control and bails with
an empty list). The resize `0x5b4840(ctrl, count)` sets `[ctrl+8] = realloc/malloc(count · 0x110)`
(the runtime record is **0x110 bytes**) via `0x646320` / `0x6462e0`. The per-slot init `0x5b47f0`
zeroes the slot's `+0x44` / `+0x48` / `+0x4c` (its **position**), and the entry dtor `0x5b47b0`
frees `[ctrl+8]` when non-null.

The **wire read sequence per record** is the getter call order (the runtime
stores some fields out of order, but the wire order is the call order):

```
guid:        u64
name:        cstr (null-terminated, <= 0x30)
race:        u8
class:       u8
gender:      u8
skin:        u8
face:        u8
hairStyle:   u8
hairColor:   u8
facialHair:  u8
level:       u8
zoneId:      u32
mapId:       u32
x:           f32
y:           f32
z:           f32
guildId:     u32
charFlags:   u32
firstLogin:  u8
petDisplayId: u32
petLevel:    u32
petFamily:   u32
equip[20]:   { displayId:u32, invType:u8 }
```

Notes that resolve common ambiguities: `name` is a null-terminated C-string (read by `0x4191b0`
`GetString`), not a fixed-width field; `guildId` is a single `u32`; the wire order is
`zoneId → mapId → x/y/z → guildId` even though the runtime stores them out of order
(zone @ `+0x3c`, map @ `+0x38`, guild @ `+0x40`); the equip array is `{u32 displayId, u8 invType} × 20`
(the `+0x44`/`+0x48`/`+0x4c` zeroed by `0x5b47f0` is the per-record **position**, not an equip field).

## The Grunt auth-logon protocol

`Grunt.cpp` holds two peer classes, split at the bytes:

- **`ClientLink`** (vtable @ `+0x198`) is the **live** client auth-logon path. It handles
  `CMD_AUTH_LOGON_CHALLENGE` (`0x00`), `PROOF` (`0x01`), `RECONNECT_*`, `REALM_LIST` (`0x10`), and
  `XFER_*`. It is driven by `CLoginMgr`: `SendLogonChallenge` `0x5bb540` (called from `0x5b21b3`),
  and `OnLogonChallenge` `0x5baa20`, which parses the server→client `{B, g, N, salt}` challenge and
  runs SRP6 `0x5d3650` to derive the 40-byte session key `K` at `0xc2a428`. `K` fills `NETCLIENT+0x48`
  / `Logon+0x54` and arms the world stream cipher.
- **`ServerLink`** (`CMD_GRUNT_*`, vtable @ `+0x108`) is the realmd↔realm-server protocol — **dead
  surface in the client** (its constructor `0x5b8980` has zero references in `WoW.exe`). It exists in
  the binary but the client never instantiates it.

The CMSG builders and SMSG parsers in this band own no wire math — all byte composition goes through
`CDataStore`.

## Crypto primitives

### Hashes and checksums

| Primitive | Addresses | Notes |
|---|---|---|
| SHA-1 (family A) | `0x5d1420`, `0x5d1a40`, `0x5d1a80`, `0x5d1b60` | count-first context, `__thiscall` update; all four round constants present |
| SHA-1 (family B) | `0x5d1db0`, `0x5d1de0`, `0x5d1ec0`, `0x5d32b0` | state-first context, `__fastcall` update |
| MD-5 | `0x661da0`, `0x661dd0`, `0x661e90`, `0x661f00` | `K1 = 0xd76aa478`; little-endian schedule/length |
| CRC-32 | `0x662860` | reflected, poly `0xEDB88320` |
| SHA1-PRNG | `0x5d1c70`, `0x5d1cf0` | feeds the SRP ephemeral; a SHA1-based generator, **not ARC4**; `seed` / `generate` over a 0x40-byte state |

### SRP6

The client uses a `k = 3` SRP-6 exchange:

| Step | Address | Notes |
|---|---|---|
| Client core | `0x5d3650` | computes `x`, `A = g^a mod N`, `u`, `S`, `K`, `M1` |
| Session-key interleave | `0x5d3360` | RFC 2945 `SHA_Interleave` |
| Credential hash | `0x5d3590` | `SHA1(USER)` at `this+0x5c` and `SHA1(USER:PASS)` at `this+0x70` |
| M2 verify | `0x5d3ad0` | finalizes the caller's in-progress SHA-1 context (not a standalone primitive) |
| File HMAC | `0x5bba80` | `SHA1(chunk ‖ key ‖ record[0xc..0x20])` |

The ephemeral `a` is `rand + bitlen(N)`. SBig (big-number) serialization is **little-endian**.

The multi-precision integer arithmetic itself — StormLib `BigNumber` add/sub/mul/divmod/cmp/modexp,
`0x663280`–`0x666900` (modexp at `0x665160`) — is delegated host plumbing; `SSignature` `0x66cf10`
(asset signature) likewise.

### Login proofs

- **PIN-grid shuffle** `Logon::PinGridShuffle` `0x5b1e70`: a factorial-base Lehmer permutation of the
  digits `0..9` (a base-10 Fisher-Yates). ABI `__thiscall(ecx = this, [esp] = seed)`, output at
  `this+0x1ae`.
- **HMAC-SHA1** `0x5b1170`: RFC-2104 HMAC, with **no long-key shortening** (keys are not pre-hashed).
- **Checksum proof** `0x5b31e0`: `SHA1(k2 ‖ SHA1(k1 ‖ data))`.

## Inbound wire decodes — SET_TIMESPEED and MONSTER_MOVE

### SMSG_LOGIN_SETTIMESPEED (`0x42`, handler `0x6c5e80`)

Body: `{ u32 packedDate (GetInt32 0x418eb0), f32 timeRate (GetFloat 0x419130) }`. The
`secsToTimeBitFields` unpack `0x642b30` (wrapper `0x642c30`) then splits `packedDate` into 6 fields,
each `v = (packedDate >> shift) & mask`, where an all-ones value maps to the sentinel `0xFFFFFFFF`:

| Field | Bits |
|---|---|
| minute | `[0:6)` |
| hour | `[6:11)` |
| weekday | `[11:14)` |
| day | `[14:20)` |
| month | `[20:24)` |
| year | `[24:29)` |

See [Game time](game-time.md) for how the day-phase feeds the lighting clock.

### SMSG_MONSTER_MOVE (`0xdd`, handler `0x603f00` → decoder `0x6018f0`)

A packed-GUID (`0x642ed0`) plus a unit lookup, then the spline body:

```
startPos    : 3 x f32
splineId    : u32
moveType    : u8
target      : switched on moveType (jumptable 0x602114)
                1 = stop / none
                2 = vec3
                3 = guid
                4 = facing
flags        : u32  \
duration     : u32   |  skipped entirely when moveType == 1
pointCount   : u32  /
endpoint     : 3 x f32
points       : pointCount - 1 intermediate points
                 flags & 0x200  -> explicit vec3 each
                 else           -> packed int32 deltas
```

The packed-delta decode `0x602130` is shared with collision (x/y as signed-11, z as signed-10, scaled
by `×0.25`, `point = endpoint − delta` — see [Collision](collision.md)). Only the outer field
sequence above belongs to the network layer.
