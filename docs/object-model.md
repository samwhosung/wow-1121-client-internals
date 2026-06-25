# Object model — the CGObject hierarchy and UpdateFields

Every entity the server tells the client about — players, NPCs, items, game objects, dynamic
objects (spell visuals), and corpses — is a C++ object descended from a single base class,
`CGObject_C`. Each carries a server-replicated array of **UpdateFields** (a flat `u32[]` keyed by
GUID and type), kept in sync by the `SMSG_UPDATE_OBJECT` packet and indexed through the client
object manager `ClntObjMgr`. This chapter covers the class hierarchy, the in-memory object layout,
the update pipeline that decodes the wire packet into the field array, the name→index map for the
fields themselves, and the per-object machinery (render transform, animation selection) that reads
those fields.

## The CGObject kind hierarchy

Eight concrete classes, all sharing the `CGObject_C` header. Each has a primary vtable and a
constructor; the in-memory size is `0x110 + fieldcount·4` (the header followed by the UpdateField
descriptor block):

| Class | vtable | ctor | TYPEID | TYPEMASK |
|---|---|---|---|---|
| `CGObject_C` | `0x803640` | `0x6139a0` | 0 | `1` |
| Item | `0x80ada8` | `0x5d81f0` | 1 | `3` |
| Container | `0x80ac88` | `0x5d7730` | 2 | `7` |
| Unit | `0x80c4f8` | `0x5fad50` | 3 | `9` |
| Player | `0x80af78` | `0x5dd2e0` | 4 | `0x19` |
| GameObject | `0x80c330` | `0x5f6f30` | 5 | `0x21` |
| DynObject | `0x80aa80` | `0x5d5140` | 6 | `0x41` |
| Corpse | `0x80ab80` | `0x5d6050` | 7 | `0x81` |

The TYPEMASK is the bitmask written into UpdateField index 2 (the TYPE field)
by the type-writer `0x613880`; it is a cumulative inheritance mask (Player `0x19` = `0x10 | 0x08 |
0x01`).

Construction goes through **two per-TYPEID dispatches**: `0x466a20` installs the correct vtable
(the ctor dispatch), and `0x465c50`'s dispatch runs the per-class initialize. The base
`CGObject_C` ctor `0x6139a0` builds 6 intrusive sub-lists at `+0x48`, a flag inventory at `+0xe4`,
and registers the object into the scene tree via `0x613e10` → `0x670db0`.

## Object memory layout

Two header pointers govern every object, regardless of class:

- **`obj+0x8`** = base pointer of the **live UpdateField array** (the descriptor base). Its first
  dwords are the object's own header: GUID at `+0/+4`, TYPEMASK at `+8`, ENTRY at `+0xc`.
- **`obj+0xc`** = the **shadow/mirror** copy used by the change-notification system (see below). It
  is *not* the "end" of the live array; it is a separate parallel buffer.

For a bare `CGObject_C`, the field block is inline at `this+0x110`. Unit/Player/GameObject instead
hold **cached view pointers** so their getters can index with class-relative offsets:

```
*(obj+0x110) = *(obj+8) + 0x18      # skips the 6 base-OBJECT fields (6·4 = 0x18)
```

set by the field-ctors `0x5fad10` (Unit) / `0x5f6ef0` (GameObject) (`add ebx,0x18; mov [+0x110],ebx`).
Unit also caches the inline movement block at `+0x118 = this+0x9a8`. Player caches a pointer to its
fields/local-data block at `obj+0xe68`, with a dynamic-field spill region pointer at `+0x1c68`.

## UpdateFields — the live array

The field array is a flat `u32[]`. Fields are **global-contiguous**: indices 0..5 are the base
OBJECT fields shared by every class, then the object's single TYPEID-specific section begins at
index 6. The per-TYPEID field count is the table at `DAT_00b4143c`:

```
{ 6, 48, 122, 188, 1282, 26, 16, 38 }   # OBJECT, ITEM, CONTAINER, UNIT, PLAYER, GAMEOBJECT, DYNOBJECT, CORPSE
```

The Player count splits by whether the object is the active player (the "self" view): **1282 self
/ 486 other** — the server sends a large private field set only for your own character.

Writes go through **`SetUpdateField` (`0x6142e0`)**, which is write-only: `*(obj+8) + idx·4 =
value`, where `idx` is the raw global index. Because Unit/Player/GameObject getters read via the
cached view `[obj+0x110]+off`, a getter at byte offset `off` reads **absolute index `6 + off/4`**.

### The shadow copy and change notification

`obj+0xc` is a parallel buffer maintained by **CMirrorHandler** (`0x467d40`–`0x468410`), a
per-object per-field change-notification registry. The shadow is written only by the copy kernel
`0x4667a0` (`*(obj+8)+rec.src → *(obj+0xc)+field_dynamic_offset`) and re-read by a memcmp
change-detect (`0x465570`). Its change-record list is a global node pool at `0xb23310`
(`0x2810` × 12-byte list heads, in BSS).

This whole observer path is **inert when no watchers are registered**: `0x465970` returns 0, so the
copy `0x4667a0` is skipped and the live array is the only state that changes. An implementer with no
field-watch callbacks can ignore the shadow buffer entirely.

### Descriptor-offset router (`field_dynamic_offset`, `0x466830`)

The shadow copy needs a *byte* offset within the per-class descriptor block for a given field index.
`field_dynamic_offset` computes it, dispatching on block-id via the jump table at `0x4669dc`:

```
result = (field_index & 3) + (A + B)·4
```

`A` is a per-class **table-scan getter**: the position of `(field_index - threshold) >> 2` within a
static descriptor table, or a per-table default if absent. `B` is a constant base getter
(`0x47f2b0()=3` · `0x47e5d0()=0x23` · `0x47db10()=0xaa`; block 6 has no table, `A=0x47c9f0()=10`).
Sub-threshold indices fall through to the parent class (PLAYER < `0x2f0` → UNIT, CONTAINER < `0xc0`
→ ITEM, any < `0x18` → DEFAULT_OBJ). The eight static descriptor tables are binary constants:

| Table | Address |
|---|---|
| ITEM | `0x83a220` |
| CONTAINER | `0x83a0f0` |
| UNIT | `0x839e48` |
| PLAYER_A | `0x838f90` |
| PLAYER_B | `0x839430` |
| C5 | `0x838f6c` |
| CORPSE | `0x838f50` |
| DEFAULT_OBJ | `0x83a2a8` |

### Descriptor-block reset (`zero_descriptor_block`, `0x466c70`)

On (re)construction, the per-TYPEID descriptor block is zeroed. The function calls the per-class
field-block cleanup ctor, then `rep stos` zeros `size` bytes at `obj+offset` from the table at
`0x466db8`:

| TYPEID | offset | size |
|---|---|---|
| OBJECT | `0x110` | `0x18` |
| ITEM | `0x348` | `0xc0` |
| CONTAINER | `0x6e0` | `0x1e8` |
| UNIT | `0xe68` | `0x2f0` |
| GAMEOBJECT | `0x288` | `0x68` |
| DYNOBJECT | `0x1a0` | `0x40` |
| CORPSE | `0x2b8` | `0x98` |

PLAYER selects `0x1d70`/`0x1408` (self) vs `0x1d70`/`0x798` (other) and writes the dynamic-field
spill pointer at `+0x1c68` (non-null only for self). TYPEID > 7 is a no-op.

## The field-index map — which index is which field

The base OBJECT fields (indices 0..5):

| Index | Field | Notes |
|---|---|---|
| 0, 1 | GUID lo / hi | array head (`obj+8`+0/+4) |
| 2 | TYPE (TYPEMASK) | written by `0x613880` |
| 3 | ENTRY | GO template-cache key (`[+0xc]`, `0x5f77a3`) |
| 4 | SCALE_X | float → render scale (`0x469f10`/`0x614c1a`) |
| 5 | (padding) | unproven |

UNIT section (from index 6; byte offsets are into the cached view `[unit+0x110]`):

| Index | Field | View offset |
|---|---|---|
| 22 | HEALTH | `+0x40` |
| 28 | MAXHEALTH | `+0x58` |
| 34 | LEVEL | `+0x70` |
| 36 | BYTES_0 | `+0x78` (race byte0, sex byte2) |
| 131 (`0x83`) | DISPLAYID | `+0x1f4` |
| 133 | MOUNTDISPLAYID | `+0x1fc` |

PLAYER section begins at index **188** (`descrBase+0x2f0`, cached at `[unit+0xe68]`):

| Index | Field | Offset | Byte layout |
|---|---|---|---|
| 193 (`0xC1`) | PLAYER_BYTES | `[+0x110]+0x2ec` / `[+0xe68]+0x14` | byte0 skinColor, byte1 faceType, byte2 hairStyle, byte3 hairColor |
| 194 (`0xC2`) | PLAYER_BYTES_2 | `[+0xe68]+0x18` | byte0 facialHair (decode `0x5fb200`) |

Race and sex are **not** in PLAYER_BYTES — they are in UNIT `BYTES_0` (index 36) byte 0 / byte 2.

GAMEOBJECT section (from index 6):

| Index | Field | View offset |
|---|---|---|
| 8 | DISPLAYID | `+0x8` |

**The display-id → model resolution chain** is the practical payload of this map:

```
UNIT DISPLAYID (131)  → [unit+0x110]+0x1f4 → CreatureDisplayInfo [0xc0de90]
                                            → CreatureModelData  [0xc0de68] → M2 model path
GAMEOBJECT DISPLAYID (8) → [go+0x110]+0x8  → GameObjectDisplayInfo [0xc0dcec]
```

See [Models](models.md) / [Character model](character-model.md) for what consumes the resolved path,
and [DBC](dbc.md) for the `CreatureDisplayInfo`/`CreatureModelData` column schemas.

**A unit's name is not a field.** It is fetched by an async name query and cached at `unit+0xb30`,
never stored in the UpdateField array.

## ClntObjMgr — the client object manager

A singleton at `DAT_00b41414` (a `0xe0`-byte struct; also republished at `[0xc28114]+0x1ad8`). Its
constructor `0x464ff0` builds two hash tables, a 7×`0xc` per-type list block, and the slot family at
`+0xc0..+0xd8`. It is constructed at game-init (`0x4015f3`) and world-enter (`0x401c22`); its
destructor `0x467700` clears `[0xb41414]`. (The separate pool teardown `0x468490` runs from
`0x401ee0` during `ClientDestroyGame`.)

The two hash tables:

- **Table #1** — the primary live GUID index (insert `0x4643b0`).
- **Table #2** — the lifecycle/staging-recycle index (promote `0x464530` / remove `0x464920` /
  destroy `0x4674a0`). Disjoint from table #1.

The manager slots:

| Slot | Meaning |
|---|---|
| `+0xc0/+0xc4` | active-player GUID (written only by the CREATE-self prepass `0x466245`/`0x46624d` on `UPDATEFLAG_SELF`) |
| `+0xc8`, `+0xd8` | no writer in `.text` |
| `+0xcc` | the ctor's owner arg |
| `+0xd0` | the client-app object `DAT_00c28114` (the net-registration `this`) |
| `+0xd4` | a [collision](collision.md)-owned slot, driven by its tick thunks `0x630a30`/`0x630a40` |

### The accessor API

- **Raw GUID lookup** `0x464870`/`0x464890` — table #1; object pointer at entry+0x08.
- **Typed lookup** `0x468460` — the workhorse; gates on the TYPEMASK at `object+0x8`.
- **Active-player getter** `0x468550` — reads slot `+0xc0/+0xc4`.
- **Slot-accessor cluster** `0x468550`–`0x468610` reads `+0xc0..+0xd8`; the collision band thunks
  in (`0x630a40 = jmp 0x4685e0`, reading slot `+0xd4`).

## The SMSG_UPDATE_OBJECT pipeline

The net→object bridge registers handlers at `0x465140` (thiscall on `*(mgr+0xd0)` via net's
`0x537a60`):

| Opcode | Handler | Meaning |
|---|---|---|
| `0xa9` | `0x4651a0` | `SMSG_UPDATE_OBJECT` — the apply entry |
| `0x1f6` | `0x4672f0` | compressed update: zlib inflate (`0x660740`) → re-enters `0x4651a0` at `0x4673b6` |
| `0xaa` | `0x4674a0` | destroy: packed-GUID unlink on table #2 |

The apply entry dispatches each block on a 1-byte UPDATETYPE via the 6-entry jump table at
`0x465314`:

| UPDATETYPE | Handler | Meaning |
|---|---|---|
| 0 | `0x465330` | VALUES |
| 1 | `0x465ea0` | MOVEMENT |
| 2, 3 | `0x465c50` | CREATE (arg bool selects which) |
| 4, 5 | `0x465fd0` | OUT_OF_RANGE / NEAR objects |

Wire formats:

- **VALUES** — a field-mask walk: `u8 maskDwordCount`, the mask bitmap, then one `u32` per set bit.
- **CREATE** — packed GUID + `u8 typeId` + an `updateFlags`-gated movement sub-block + a VALUES
  block.
- **OUT_OF_RANGE / NEAR** — `count` × packed GUID.

The packed-GUID codec is **`CDataStore::GetPackedGuid` (`0x642ed0`)**, shared with [net](net.md).

**Two-pass transport semantics:** transports are created first; the stream is then rewound and the
main apply runs with deferred callbacks. This ensures objects riding a transport resolve their
carrier before they are positioned.

### The writer and the notifier

Two functions handle the VALUES block, split by role:

- **`apply_update_fields` (`0x466590`) — the WRITER.** Reads the VALUES block and, for each field
  index `0..updatefield_count`, stores the wire value into the **live** array via `SetUpdateField`
  (`*(obj+8)+idx·4`). When the `clearUnset` flag is set, clear bits are forced to 0.
- **`apply_values_block` (`0x465330`) — the main-loop NOTIFIER.** Re-reads the same wire bytes,
  **discards** the values (it never calls `SetUpdateField` and writes no game state), runs the
  memcmp change-detect against the shadow, and fires watcher callbacks. `0x465bb0` is its
  VALUES-block discard twin.

### Field-layout geometry

The geometry sub-functions the pipeline composes:

| Function | Address | Computes |
|---|---|---|
| `updatemask_test_bit` | `0x465550` | is field N present in the wire mask |
| `updateblock_stride` | `0x465690` | block stride |
| `descriptor_section` | `0x4656e0` | which descriptor section a field belongs to |
| `updatefield_addr` | `0x465850` | address of field N |
| `updatefield_count` | `0x465a80` | field count for (TYPEMASK, guid, self) |
| `player_dynfield_base` | `0x466de0` | base of the player dynamic-field spill |

`updatefield_count` is where the **1282/486 player self/other split** is resolved (it inspects the
active-player GUID through `0x468550`).

## The movement block codec

The `updateFlags`-gated MovementInfo + spline block carried by CREATE/MOVEMENT has its own
flag-driven size and codec (callers `0x465c50`/`0x465ea0`):

| Function | Address |
|---|---|
| `movement_block_decode` | `0x47f050` |
| `movement_block_skip` | `0x47ede0` |
| `movement_block_encode` | `0x47ecb0` |
| `movement_info_size` | `0x47eb10` |

### The speeds[6] wire order

The movement block carries 6 consecutive f32 speeds at `+0x50..+0x64`, copied **1:1 in order** into
`CMovement +0x88..+0x9c` by the apply chain `0x467110 → 0x5ff030 → 0x619320`:

| Index | Speed | Block off → CMovement off |
|---|---|---|
| 0 | walk | `+0x50 → +0x88` |
| 1 | run | … |
| 2 | runBackward | … |
| 3 | swim | … |
| 4 | swimBackward | … |
| 5 | turnRate | `+0x64 → +0x9c` |

The speed getter is `0x7c4c90`; the animation selector queries it with param 0 (the directional
current speed, not a walk/run minimum).

## The per-frame transport tick — broadcast `0x630970`

The per-frame listener broadcast `0x630970` (owned by [collision](collision.md)) is
**transport-GameObject-exclusive**. Only two GameObject types ever register:

- type **15** (MO_TRANSPORT) — strategy vtable `0x80b798`
- type **11** (TRANSPORT) — strategy vtable `0x80ba58`

Registration happens in the strategy constructor (register `0x6308b0`), guarded by
`[strategy+0x10] != 0`. The registry is an intrusive sentinel-deque
(`{linkOffset 0x208, tail 0xc4e4b8, head (…)|1}`, init `0x6307c0`). No Unit/Player/Item/other-GO
registers.

Dispatch fires `[GO+0x210].vtable+0x74(now, delta)`, where `GO+0x210` is the per-type **strategy
object** from the 31-way factory `0x5f7036`. The two live `+0x74` slots are:

- `0x5f50a0` — transport-heading update (type 15)
- `0x5f5f10` — move-normalize (type 11)

The other 29 strategy types' `+0x74` is `0x5f9ed0` (`ret 8` no-op). Each tick touches only its own
GO+strategy state, so deque order is deterministic but immaterial. A client need only iterate
registered type-11/type-15 transports and run those two functions; Unit/Player ticking is a true
no-op.

**Do not confuse this with the per-kind `+0x74` slot in the primary object vtables**
(Object/Unit/Player `0x46a080`; Item/Container `0x5d9e10`; GameObject `0x5f5900`). That is slot 29 of
the *primary* vtable — a separate 1-arg dispatch (all `ret 4`), never fired by this 2-arg (`ret 8`)
broadcast. Those override bodies are lookup queries, not the transport tick.

## The CGUnit render world-transform — facing → M2 rotation

A unit's facing (radians, from MovementInfo orientation) builds its M2 model matrix in
**`0x613ef0`**, which composes `Rz(facing) · Scale(sx,sy,sz)` and installs it to the CM2Model at
`[unit+0xd8]`:

```
M = Rz(facing) · Scale(sx, sy, sz)        # column-major
```

The facing comes from the raw getter vtable+0x18 (`0x7c4ae0` = `fld [ebp+8]; ret 4`, **no negate or
offset**). The rotation uses the Rodrigues axis-angle builder (`0x7bdb00`, axis `(0,0,1)` = world-up
Z) then the scale (`0x7bdd00`) — see [Math primitives](math-primitives.md).

Rotation is `Rz(+facing)` — **positive** sign, zero offset. Rodrigues stores `+sin` at `0x7bdbd9`
and `−sin` at `0x7bdbec`; with axis `(0,0,1)` this gives, column-major:

```
col0 = ( c, +s, 0)
col1 = (−s,  c, 0)
col2 = ( 0,  0, 1)
```

The M2 model's **local forward is +X**: at `facing = 0` the model points world +X (east), and
increasing facing rotates +X counter-clockwise toward +Y (north) — WoW's facing convention. The host
composes `T · Rz(facing) · S` with +Z yaw, +facing, +X-forward.

This is **not** the weapon-swing path. `0x608560 → 0x60a110` (a `2π − facing` swing transform) is
gated on the combat animation phase (`test [[unit+0x118]+0x40], 0x200000` at `0x608572`) — it drives
the weapon trail, not the always-on body facing.

## Movement-state → animation-id selection

Which AnimationData id a CGUnit plays is resolved by a priority chain. The selector `0x5fd8b0`
(seeded with `0xD0` = "keep current") is driven by `RecomputeBaseAnim` (`0x5fd9e0`, called with
reqId = −1 from the move processor `0x6010e0`); if the result ≠ `0xD0`, it is played via `0x5fe2f0`
→ op4 `0x7121a0`. Inputs: move-state flags at `[unit+0x9e8]`, speed via `0x7c4c90`, walk-speed at
`[unit+0xa30]`.

The decision, in priority order:

```
airborne ([9e8]&0x2000 and vertical-velocity ≠ 0) → 0xD0 (keep current)
  (dead / mount / standState resolvers run first)

core ground (0x5fd100, gated [9e8]&0xf = any direction bit):
    SWIMMING        → 42 fwd / 45 back / 41 turn / 43–44 strafe
    backward        → 13
    speed ≥ 11.0    → 143                        # fast cutoff, [0x80c484]
    speed > 2·walk  → 5  (RUN)                   # walk-speed at [unit+0xa30]
    else            → 4  (WALK)

fallback idle (0x5fd830):
    swimming                              → 41
    jump/float ([[118]+0x40]&0x40000800)  → 193
    else                                  → 0  (STAND)

standState (sit/sleep/kneel) → DBC/table resolver 0x612990 / 0x5fd550  (ids ~96–116)
```

The walk-vs-run boundary is **speed-driven** (2× walk speed), *not* the WALK-mode bit `0x100` — that
bit only scales playback rate. See [Animation](animation.md) for the playback arm.

## SMSG_MESSAGECHAT wire format

The chat/system handler `0x5e7180` is registered under **SMSG opcode `0x92`** (`0x5e337a mov
ecx,0x92; 0x5e337f call 0x5ab650`) — not the often-cited `0x96`. Its wire contract:

```
u8  chatType
u8  strCount
strCount × CString          # NUL-terminated, ≤0x100; CDataStore Get-u8 0x418cb0 + GetString 0x4191b0
u64 GUID                     # ONLY for chatType 0xc / 0xd, read after the strings (0x4190b0)
```

`switch(chatType)` (jump table `0x5e74dc`) maps to a UI event id fired via `0x496720(eventId,
buf0..bufN)` (N = strCount, positional):

- ids `0x57`/`0x59`/`0x5a`/`0x5c`/`0x5d`/`0x69`/`0x6a`/`0x6b`/`0x6c` — direct
- `0xc`/`0xd` → `0x106`/`0x107` (name-resolve + ignore-check `0x5ae810`)
- `0x2` → `0x11c` (profanity-gated via `0x703f50`)
- `0xa` = channel-update (no event); `0xb` = silent

The `CHAT_MSG_*` event-name table is at `0xb4b498` (stride `0x14`), the bridge to FrameXML — see
[UI](ui.md).

## Base vtable default virtuals

The `CGObject_C` base vtable's default slot bodies (overridden as needed by subclasses):

| Slot | Body |
|---|---|
| 6 | `GetFacing` — returns `0.0f` |
| 7 | returns `desc+0x10` |
| 17 | tail-forwards to slot 6 |
| 27 | `GetScale` — returns `1.0f` |
| 29 | the primary-vtable `+0x74` per-frame slot — base no-op |

The remainder are `ret 0` / `ret 1` / no-op. (Slot 29's no-op is `0x46a080`, overridden by
Item/Container `0x5d9e10` and GameObject `0x5f5900` — the 1-arg dispatch, distinct from the
transport tick.)

## Other per-object primitives

- **`faction_template_reaction` (`0x606640`)** — the pure FactionTemplate comparator (record layout
  pinned); the `Can*`/`UnitReaction` wrappers build on it.
- **`gameobject_path_eval` (`0x5f6280`)** — quaternion → matrix plus spline lerp for transport paths.
- **`transport_speed_easing` (`0x5f8dc0`)** — transport speed easing curve.
- **`player_spell_power_cost` (`0x5ee150`)** — player spell power-cost computation; see
  [Spells](spells.md).

## Dead ends and gotchas

- **The shadow buffer (`obj+0xc`) is inert without registered watchers.** `0x465970` returns 0 with
  no watchers, the copy `0x4667a0` is skipped, and only the live array (`obj+0x8`) changes. The
  earlier instinct to treat `obj+0xc` as the live array (and `obj+0x8` as a shadow) is backwards.
- **Manager slots `+0xc8` and `+0xd8` have no writer in `.text`** — they are never assigned.
- **A unit's name is never an UpdateField** — it lives at `unit+0xb30` via async query.
- **The `0x630970` per-frame broadcast does nothing for non-transport objects** — only the two
  transport GO types register, so Unit/Player/Item have no per-frame tick on this path.
