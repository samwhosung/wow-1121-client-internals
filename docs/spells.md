# Spells — client cast lifecycle, cooldowns, targeting, visuals

The client-side spell system (the engine's internal **"Magic"** module — the `WoW\Source\Magic\…` RTTI
string lives at `0x86fa64`) drives everything the player sees and does around casting: starting a cast,
the targeting cursor, sending the cast to the server, reacting to the server's start/go/fail/channel
replies, tracking cooldowns and the global cooldown, the cast-time / duration / range / radius math, the
spell-modifier tables, the `Script::Spell*` Lua API, and the spell-visual / particle pipeline. It sits
*above* the object layer (which owns the unit-attribute side of spells) and is driven entirely by network
SMSG dispatch plus the Lua targeting API — it is not its own event-bus category. The DBC spell records,
the wire (de)serialization, and the unit/object relation checks belong to other subsystems
([dbc](dbc.md), [net](net.md), [object-model](object-model.md)); this chapter covers the logic the spell
module itself computes.

## Runtime state — the fixed statics

The casting state is not a heap singleton; it lives in several disjoint module statics (BSS/.data).

| region | range | what |
|---|---|---|
| pre-cast scalars | `0xceac30` / `0xceac34` | current/confirmed-cast key + RunOnce/CVar handle |
| **SPELLCAST** | `[0xceac48, 0xcead48)` | the player's active/pending cast request (256 B) |
| **SPELLMOD_FLAT** | `0xcead60` | `i32[64][29]` flat modifiers (set by SMSG `0x266`) |
| **SPELLMGR** | `0xceca60` | the singleton: cooldown/history container + scalar cast state |
| **SPELLMOD_PCT** | `0xcecb30` | `i32[64][29]` pct modifiers (set by SMSG `0x267`) |

`SPELLHISTORY` records are heap nodes living in two intrusive lists owned by SPELLMGR.

### SPELLCAST `0xceac48` — the pending cast request

A `SpellCastTargets`-shaped block (256 B). Getter `0x6e3d40` returns `&SPELLCAST`; it is cleared (0x40
dwords) at the top of a new cast and on CAST_RESULT. The status word `+0x14` records which sub-fields are
populated.

| off | abs | width | field |
|---|---|---|---|
| `+0x00` | `0xceac48` | u64 | caster_guid |
| `+0x08` | `0xceac50` | u64 | target_guid |
| `+0x10` | `0xceac58` | i32 | spell_id (index into the SpellRec table `0xc0d788`) |
| `+0x14` | `0xceac5c` | u16 | status flags: `0x2` unit-bound, `0x10` item-bound, `0x20` src-loc/string, `0x40` dest-loc (the unit-bound test is `& 0x8202`) |
| `+0x15` | `0xceac5d` | u8 | sub-status: `0x8` GO, `0x20` string, `0x80` corpse, `0x2` dead, `0x10` trade-item |
| `+0x18` | `0xceac60` | u64 | bound unit / GameObject target guid (also a world-destination) |
| `+0x20` | `0xceac68` | u64 | item / trade target — item-object guid, OR trade slot index `6` (`TRADE_SLOT_NONTRADED`) with hi-dword 0 |
| `+0x30` | `0xceac78` | f32×3 | source-location Vec3 (`+0x30/+0x34/+0x38`) |
| `+0x3c` | `0xceac84` | f32×3 | dest-location Vec3 (`+0x3c/+0x40/+0x44`) |
| `+0x70` | `0xceacb8` | char[0x80] | STRING-target name buffer |

`+0x28/+0x2c` and `+0x48..+0x6c` are only ever reset (`+0x4c` to `-1`, the rest to 0; reset routine
`0x6e11e0`); no typed write reaches them on the cast/targeting path. The standalone scalar `0xcead5c`
(outside SPELLCAST proper) holds the last-sent cast spell_id.

### SPELLMGR `0xceca60` — the singleton

Its head is the cooldown/history hash container (vtable `0x811904`); the rest is scalar cast state and
the modifier tables. All scalars are zeroed by `Spell_C::SystemInitialize 0x6e7150`.

| off | abs | field |
|---|---|---|
| `+0x00` | `0xceca60` | hist container vtable (`0x811904`) |
| `+0x04` | `0xceca64` | hist element size (init `0xc`) |
| `+0x0c` | `0xceca6c` | hist used-list head |
| `+0x10` | `0xceca70` | hist count |
| `+0x18` | `0xceca78` | hist bucket count |
| `+0x1c` | `0xceca7c` | hist buckets (`0xc`-byte slots) |
| `+0x24` | `0xceca84` | hist mask (init `-1`, set `3`) |
| `+0x28` | `0xceca88` | inflight_spell_id (awaiting CAST_RESULT) |
| `+0x2c` | `0xceca8c` | inflight_handle (heap; freed via `0x670d50`) |
| `+0x30` | `0xceca90` | ground_target_flag (AoE/TARGET) |
| `+0x48` | `0xcecaa8` | saved_spell_id (nested-cast save slot) |
| `+0x4c` | `0xcecaac` | modifier class_category (chrClasses[idx]+0x3c) |
| `+0x50` | `0xcecab0` | inflight_target_guid (u64) |
| `+0x60` | `0xcecac0` | **flag_word** — targeting/cast-active mask (u16) |
| `+0x74` | `0xcecad4` | targeting reticle orientation angle (f32) |
| `+0x78` | `0xcecad8` | autorepeat_aux |
| `+0x8c` | `0xcecaec` | self **SpellHistory list** (also the cooldown event queue head) |
| `+0xc0` | `0xcecb20` | saved_target_guid (u64) |

The nested-cast save/restore swaps `{inflight_spell_id, inflight_target_guid}` ↔ `{saved_spell_id,
saved_target_guid}` (routine `0x6e4ad0`). The reticle field `+0x74` was once mislabelled "pushback"; it is
the spell-pending cursor orientation (cached from the target's facing, advanced by `0x6e69e0`).

### SPELLHISTORY record (heap, 48 B; `SMemAlloc(0x30)`)

The per-cast record is `SPELLHISTORY` — the engine's own name, recovered from the `.?AUSPELLHISTORY@@`
type tag the compiler embeds in the binary. The "cooldown" the player sees is modelled internally as a
**spell history**: each cast appends a record, queries walk the list for the remaining time, and a garbage
collector reaps records whose timers have all elapsed. Appended by the history-add routine at `0x6e12c0`.

| off | width | field | populated from |
|---|---|---|---|
| `+0x00` | ptr | `prevLink` — intrusive list link | |
| `+0x04` | ptr | `NextValue` — intrusive list link | |
| `+0x08` | i32 | `spellID` | |
| `+0x0c` | i32 | `itemID` (set for an item-triggered cooldown) | |
| `+0x10` | u32 | `recoveryStart` | `OsGetAsyncTimeMs` |
| `+0x14` | u32 | `recoveryTime` | `SpellRec+0x4c` via modifier; `30000` for items |
| `+0x18` | i32 | `category` | `SpellRec+0x8` |
| `+0x1c` | u32 | `categoryRecoveryStart` | |
| `+0x20` | u32 | `categoryRecoveryTime` | `SpellRec+0x50` |
| `+0x24` | u8 | `onHold` (a single boolean, not a flags field) | `SpellRec+0x18` bit25 |
| `+0x28` | i32 | `startRecoveryCategory` | `SpellRec+0x274` |
| `+0x2c` | u32 | `startRecoveryTime` (the global cooldown) | `SpellRec+0x278` |

Item cooldowns additionally use a separate `ITEMCOOLDOWNHASHNODE`.

### The SpellHistory lists (`TSExplicitList`, 24 B; two instances)

`SPELLHISTORY` records live in two intrusive lists — self at `0xcecaec`, pet at `0xcecb04`, stride `0x18`.
The constructor `0x6e1050` writes the two
self-referential link sentinels per list: the `{+0x04,+0x08}` pair is the used-list sentinel (tail tagged
`|1`), the `{+0x10,+0x14}` pair the free-list sentinel. The list index is selected by
`(caster_guid == pet_guid [0xb714a0]) ? 1 : 0`.

## The data table — `SpellTableInitialize 0x6dda50`

Called from startup (`ClientInitializeGame` region `@0x401634`), this builds three skill/spell index
hash tables over the DBC source tables. Each index is a `TSHashTable`-shaped object (a `TSGrowableArray`
of `TSExplicitList` buckets + a node freelist + a power-of-two hash mask):

| index | base | vtable | node RTTI |
|---|---|---|---|
| RaceClassInfo | `0xceab3c` | `0x811818` | `RACECLASSINFO` |
| SkillCostsInfo | `0xceab80` | `0x811828` | `SKILLCOSTSINFO` |
| SkillLineRec | `0xceabec` (array `0xceac08`) | `0x811848` | `SKILLLINEREC` |

Index node layout: `{+0x00 key, +0x14 flag byte, +0x18 record-ptr into the DBC source, +0x1c
run-length}`. The SKILLLINEREC node widens `+0x14` into `{+0x14 class byte, +0x15 race byte, +0x16
word}`. The builder runs three passes, each scanning a pre-sorted DBC source table and de-duplicating
consecutive equal keys into `{firstRow, runLength}` index runs (the `key == prevKey` short-circuit bumps
`node[+0x1c]++`). Pass 3 filters each SkillLineRec through the player's race/class before inserting.

The availability gate `ClassRaceMaskMatch 0x6dddd0` is the canonical class/race test used by the lookups:

```
if invertClass: classMask = ~classMask
if invertRace:  raceMask  = ~raceMask
if classMask != 0 and not (classMask & (1 << (classIndex-1))): return false
if raceMask  != 0 and not (raceMask  & (1 << (raceIndex -1))): return false
return true
```

The `1 << (idx-1)` bitmask form and the two inversion flags (so one helper serves both "allowed" and
"forbidden" masks) are the durable shape. Lookups: `RaceClassInfoIndex_FindMatching 0x6ddf90`,
`0x6de040` (SkillLineRec find), `0x6de110` (SkillCostsInfo getter).

`Spell_C::SystemInitialize 0x6e7150` (called from world-enter master init `@0x6038da`) zeroes the
singleton, registers the SMSG handlers, and registers the five `Script::Spell*` Lua bindings.
`Spell_C::Destroy 0x6e99e0` is the mirror teardown.

## The cast lifecycle state machine

The casting subsystem is a request/confirm machine over SPELLCAST (the pending request) and SPELLMGR (the
targeting flag_word + the inflight/saved-cast slots). The flow:

1. `TryCast 0x6e4b60` (the begin-cast entry point, called from the action bar / Lua / object layer).
   Validates the spell can be cast *now* — passive (`SpellRec+0x18 & 0x40`), in-combat
   (`& 0x800000` exempts), reagents/totems (`0x6e4000`), power/form (`0x6e40e0`), LOS, already-casting,
   special effect dispatch on `SpellRec+0xf4` (mounted → reason `0x82`, etc.). Each failure routes to
   `HandleCastFailed 0x6e1a00` with a reason byte. On success it clears SPELLCAST (0x40 dwords), writes
   caster_guid / target_guid / spell_id, runs the target validator `CanTargetUnit 0x6e4440`, and hands to
   the cast-arm.

2. `ArmCast 0x6e5250` — the target-constraint builder, the heart of the system. It **seeds
   `flag_word = SpellRec+0x34`** (`mov ax,[esi+0x34]; mov [0xcecac0],ax`), the Spell.dbc `Targets`
   (TARGET_FLAG_*) mask, then runs a **62-case switch on `SpellRec+0x148`** (the implicit-target enum;
   jump table `0x6e5484`, index byte-table `0x6e54b0`) to set/clear individual target-flag bits. Then:
   - **`flag_word == 0`** → self/instant cast → commit immediately via `SendCast 0x6e54f0`.
   - **`flag_word != 0`** → a target is required → resolve a candidate (caster's current target / explicit
     guid / current-target global) and hand to the binder `BindTarget 0x6e5b40`; fall back to the player
     (runonce) or to a world destination (range-gated; the destination is stored at SPELLCAST+0x18, and
     the gate fails unless `GetMinMaxRange` max `> 100.0` — the const `0x806b10 = 100.0` self-cast
     sentinel).
   - **ground-target** (switch case 16) → returns false → TryCast enters targeting-cursor mode, scans the
     effect types for AoE kinds `{0x32,0x68,0x69,0x6a,0x6b,0x4c,0x51}`, and sets the SPELLMGR
     ground_target_flag.

3. `BindTarget 0x6e5b40` — the flag_word consumer. For each set bit in priority order, if the
   resolved object satisfies the required object relation, it binds the object's guid into SPELLCAST, sets
   the SPELLCAST+0x14 status bit, and **clears that bit from flag_word**. When flag_word reaches 0 (all
   constraints satisfied) it commits via SendCast. Location bits are bound by `BindLocation 0x6e60f0`
   (writing the AoE Vec3 to SPELLCAST+0x30/+0x3c), trade items by `0x6e6900`.

4. `SendCast 0x6e54f0` — frees the prior inflight handle, serializes the `SpellCastTargets` payload
   into a CDataStore, and ships **CMSG_CAST_SPELL (opcode `0x12e`)** via `ClientServices::Send 0x5ab630`.
   It then arms the GCD (`StartGlobalCooldown 0x6e2de0`), and for auto-repeat spells
   (`SpellRec+0x20 & 0x20` or `+0x18 & 0x2`) sets the auto-repeat key `0xceac30 = SpellRec+0x00` and fires
   FrameScript event `0x1b3`, and performs the nested-cast save.

5. **Server reply.** The SMSG handlers (registered at init) confirm/drive the cast — see the handler
   table below.

6. `AbortCast 0x6e4940` (stop targeting / stop casting). If targeting, it just clears flag_word; if
   casting, it ships **CMSG_CANCEL_CAST (opcode `0x12f`)**. Always: pops the nested save, frees the
   inflight handle, fires FrameScript `0x152`/`0x153`/`0x154` (stop / interrupted / failed), and (if the
   target was the player) stops the cast animation. `CancelAura 0x6e7040` ships **CMSG_CANCEL_AURA
   (opcode `0x136`)**. `CastSpellByName 0x6e5ad0` with the literal `"none"` (`0x83f2c8`) cancels the
   current cast.

Key predicates: `IsTargeting 0x6e48a0` = `flag_word != 0`; `IsCasting 0x6e3d30` = `inflight_spell_id !=
0`. `IsValidCastTarget 0x6e3d60` is the comprehensive "can the player cast this on this object right now"
gate behind the red/green cursor.

## The targeting flag_word `0xcecac0`

The flag_word **is the `SpellCastTargets` target mask** (the wire `TARGET_FLAG_*` field). It is seeded
from `SpellRec+0x34`, adjusted by the 62-case switch in ArmCast, consumed bit-by-bit by the binders, and
tested by the read-only validators/accessors. A nonzero word means "a target is still required". The
`TARGET_FLAG_*` names are the corroborated community labels; the binary behaviour is what is verified.

| bit (mask) | meaning | relation check |
|---|---|---|
| 1 (`0x0002`) | UNIT — generic unit target | unit-flag `[obj+0x110]+0xa0` bit8 |
| 2 (`0x0004`) | friendly (raid) | `0x606d20` + CanAssist `0x6066f0` |
| 3 (`0x0008`) | friendly (party) | `0x606c20` + CanAssist `0x6066f0` |
| 4 (`0x0010`) | ITEM target | writes item guid to SPELLCAST+0x20 |
| 5 (`0x0020`) | SOURCE_LOCATION | writes Vec3 to SPELLCAST+0x30 |
| 6 (`0x0040`) | DEST_LOCATION | writes Vec3 to SPELLCAST+0x3c |
| 7 (`0x0080`) | UNIT_ENEMY (hostile) | CanAttack `0x606980` |
| 8 (`0x0100`) | assist/friendly | CanAssist `0x6066f0` |
| 9 (`0x0200`) | dead/corpse unit | CanAssist + corpse predicate `0x6067d0` |
| 10 (`0x0400`) | requires explicit unit selection (gate; no own guid) | — |
| 11 (`0x0800`) | GAMEOBJECT | writes GO guid to SPELLCAST+0x18 |
| 13 (`0x2000`) | STRING (name) — copies pending name to SPELLCAST+0x70 | — |
| 14 (`0x4000`) | GAMEOBJECT_ITEM / LOCKED — part of the item mask `0x4010` and GO mask `0x4800` | — |
| 15 (`0x8000`) | CORPSE (ally) | CanAssist + corpse predicate `0x6067d0` |

The switch constraint→bit map (constraint = `SpellRec+0x148 - 1`): `1`→clear bit10, `5`→clear bit15,
`6/53`→set bit7, `16`→ground-target, `21/45`→set bit8, `23`→set bit11, `25/63`→set bit1, `26`→set bit14,
`35`→set bit3, `57/61`→set bit2; all others are no-ops. The whole-word accessor predicates (each a bare
`flag_word & mask` test): `0x878e` any-unit (`0x6e6180`), `0x850e` friendly-unit (`0x6e61a0`), `0x40c`
friendly (`0x6e61e0`), `0x500` (`0x6e6200`), `0x480` (`0x6e6210`), `0x8600` (`0x6e6230`), `0x4800`
GameObject (`0x6e62d0`), `0x60` location/AoE (`0x6e6320`), `0x4010` item (`0x6e6330`); bit13 STRING
(`0x6e68f0`). The same 62-case switch is reproduced at `CanTargetUnit 0x6e4440` over the three effect
target slots `SpellRec+0x148/+0x14c/+0x150`.

## Precondition checks

- **`CheckReagentsAndTotems 0x6e4000`** — 2 totem-category slots (`SpellRec+0xa0..`) checked via
  `0x622270`; 8 reagent slots (`SpellRec+0xa8..`) via `0x622130`, comparing the carried count. Shortage →
  reason `0x5c` (totems) / `0x78` (reagents).
- **`CheckPowerAndForm 0x6e40e0`** — power type `SpellRec+0xe8`, cost `+0xec`; out-of-power → reason
  `0x1a`/`0x19`/`0x1b` (by mana/health/rage class). The form gate scans the shapeshift DBC table
  `0xc0db90` (stride `0x74`); wrong form → reason `0x2f`.

## Server SMSG handlers

`Spell_C::SystemInitialize 0x6e7150` registers every handler via 22 calls to
`ClientServices::SetMessageHandler 0x5ab650` (handler in `edx`, opcode in `ecx`); `Spell_C::Destroy
0x6e99e0` mirrors it with 22 calls to `0x5ab670`. The handler ABI is uniform: `__fastcall(ecx=0,
edx=opcode, [ebp+8]=arg0, [ebp+0xc]=CDataStore*)`, all `ret 0x8`. Packet fields are read with net's
`CDataStore` getters (`GetUInt8 0x418cb0`, `GetUInt16 0x418db0`, `GetInt32 0x418e30`, `GetUInt32
0x418eb0`, `GetUInt32_2 0x418fb0`, `GetGuid 0x4190b0`, `GetPackGuid 0x642ed0`).

| opcode | SMSG | handler |
|---|---|---|
| `0x130` | CAST_RESULT | `0x6e7330` |
| `0x131` | SPELL_START | `0x6e7640` → `0x6e7700` |
| `0x132` | SPELL_GO | `0x6e7640` → `0x6e7a70` |
| `0x133` | SPELL_FAILURE | `0x6e8d80` |
| `0x138` | PET_CAST_FAILED | `0x6e8eb0` |
| `0x134` | SPELL_COOLDOWN | `0x6e9460` |
| `0xb0` | ITEM_COOLDOWN | `0x6e95d0` |
| `0x135` | COOLDOWN_EVENT | `0x6e9670` |
| `0x1de` | CLEAR_COOLDOWN | `0x6e9670` |
| `0x1e1` | COOLDOWN_CHEAT | `0x6e9730` |
| `0x173` | PET_TAME_FAILURE | `0x6e97e0` |
| `0x1e2` | SPELL_DELAYED | `0x6e74f0` |
| `0x139` | CHANNEL_START | `0x6e7550` |
| `0x13a` | CHANNEL_UPDATE | `0x6e75f0` |
| `0x1f3` | PLAY_SPELL_VISUAL | `0x6e98d0` |
| `0x266` | SET_FLAT_SPELL_MODIFIER | `0x6e9950` |
| `0x267` | SET_PCT_SPELL_MODIFIER | `0x6e9950` |
| `0x29c` | CANCEL_AUTO_REPEAT | `0x6e99d0` |
| `0x2a6` | SPELL_FAILED_OTHER | `0x6e8e40` |
| `0x2a7` | (unit cast/aura-state refresh) | `0x6e9790` |
| `0x2b4` | FEIGN_DEATH_RESISTED | `0x6e9800` |
| `0x330` | (spell chain-target update) | `0x6e9820` |

(Opcodes `0x2a7` and `0x330` have observed behaviour but their canonical SMSG names are not confirmed
against a 1.12.1 opcode table.)

Notable bodies:

- **`CastResultHandler 0x6e7330`** (CAST_RESULT) — on failure (`result byte == 2`) reads a reason + two
  aux ints, calls `HandleCastFailed`, and if `reason != 0x3c` and `SpellRec+0x18 & 0x2000000` reverts the
  optimistically-applied cooldown via `ClearCooldown 0x6e3050`. It pops the saved/inflight slots and, for
  an auto-repeat spell (`SpellRec+0x98`), re-arms.
- **`SpellStartHandler 0x6e7640`** (SPELL_START / SPELL_GO demux) — decodes the common header (two packed
  guids + spell_id) then dispatches START → `0x6e7700` (drives the cast bar via FrameScript event
  `0x151`, the cast visual `0x6ec220`, the cast animation `0x60c9b0`) or GO → `0x6e7a70`.
- **`HandleSpellGo 0x6e7a70`** — the client-side cast-resolution pipeline. Decodes castFlags (u16),
  hitCount + hit-target guids, missCount + per-miss `{guid, reason, extra}` (reason `0xb` =
  resist/reflect carries the extra byte), and (if `castFlags & 0x20`) the ammo displayId/inventoryType.
  Branches on the projectile-speed gate `fld [SpellRec+0x94]; fcomp [0x7ffd74]` (a DBC field vs a 0.0
  sentinel): speed present → spawn travel visuals per hit target; speed zero → instant impact. The GCD
  prune throttle runs every 2 min: `if (recvTime - [0xcee8b8]) >= 0` then `[0xcee8b8] = recvTime +
  0x1d4c0` (120000 ms) and both cooldown lists are swept.
- **`HandleSpellCooldown 0x6e9460`** (SPELL_COOLDOWN) — loops over `{spellId, cooldownMs}` pairs, picking
  the self/pet list from the caster guid. For each: duration = `cooldownMs` if nonzero, else
  `SpellRec+0x4c` with spell-mod `0xb` applied; GCD = `{SpellRec+0x274, SpellRec+0x278}` with spell-mod
  `0x15` applied unless `SpellRec+0x18` bit25 is set; then `AddCooldown`.
- **`HandleItemCooldown 0x6e95d0`** (ITEM_COOLDOWN) — a fixed **30000 ms** cooldown on the item's on-use
  spell.
- **`HandlePetCastFailed 0x6e8eb0`** — a jump-table (`0x6e9394`, 15 entries indexed through the
  142-byte byte-table at `0x6e93d0`) maps the reason to a `CGGameUI::DisplayError 0x496720` error id.
- **`HandleSetSpellModifier 0x6e9950`** (SET_FLAT `0x266` / SET_PCT `0x267`) — writes one modifier table
  entry at index `modOp*0x1d + modSlot` (stride `0x1d` = 29): opcode `0x267` → `SPELLMOD_PCT[idx]`
  (`0xcecb30`), opcode `0x266` → `SPELLMOD_FLAT[idx]` (`0xcead60`).

## Cooldown management

The write path is `AddCooldown 0x6e12c0` (the single primitive that inserts/refreshes a node). It early-
outs when duration, category-duration, gcd-duration, and flags are all zero (nothing to track); otherwise
it reuses a matching node or allocates a fresh 48-byte node (`SMemAlloc(0x30)`), and fills the fields
listed in the SPELLHISTORY table above. `StartCooldown 0x6e2c60` computes the recovery / category-recovery
durations (applying spell-mod `0xb`) before inserting; `StartGlobalCooldown 0x6e2de0` inserts a GCD-only
node (spell-mod `0x15`).

Read/query: `GetCooldownInfo 0x6e13e0` (the core per-spell getter — walks the list, accumulating remaining
time from the spell `{+0x10,+0x14}`, category `{+0x1c,+0x20}`, and gcd `{+0x28,+0x2c}` pairs, subtracting
`now = OsGetAsyncTimeMs` and clamping) and the boolean `IsSpellOnCooldown 0x6e1690`. Public wrappers
select the self/pet list by `0xcecaec + listIdx*0x18`: `GetCooldown 0x6e2ea0`, `IsSpellOnCooldown
0x6e2fa0`, `GetItemCooldown 0x6e2ed0`, `IsItemOnCooldown 0x6e2fc0`.

Expire/clear: `0x6e1910` sweeps elapsed nodes (skipping item-linked nodes with `+0x24 != 0`); `0x6e1790`
removes/resets by id; `0x6e1630` zeroes only the GCD portion then re-sweeps; `SpellHistory::ClearHistory
0x6e1880` drains a whole list to its free-list; `ClearCooldown 0x6e3050` is the public wrapper. All
cooldown time math is integer-millisecond subtraction — no floating point.

## Cast-time / duration / range / radius math

These are the floating-point fidelity formulas an implementer must reproduce exactly. The `1.0` constant
is `0x7ff9d8`; the `0.0` sentinel is `0x7ffd74`. A shared level helper feeds them:

```
GetLevel(spellId, useTarget)  [0x6e3130]:
    lvl = obj->vtbl[0xa8](spellId) / 5      # integer reciprocal-multiply (truncates)
```

**`GetCastTime 0x6e31b0`:**

```
lvl = obj->vtbl[0xa8](spellId) / 5
v   = SpellRec+0x80 + (lvl - SpellRec+0x70) * SpellRec+0x84
if SpellRec+0x270 != 0:
    v += (caster_skill(SpellRec+0x7c) * SpellRec+0x270) / 100   # integer /100
v += [obj+0x110 + school*4 + 0x29c]                              # per-school flat add
castTime = round_to_int( v * (1.0 + [obj+0x110 + school*4 + 0x2b8]) )   # x87
if castTime <= 0: castTime = 0
apply spell-mod type 0xe
```

**`GetDuration 0x6e3340`:**

```
row = SpellDuration[ SpellRec+0x48 ]                       # [0xc0d878 + idx*4]
dur = row+0x4 + (lvl - SpellRec+0x70) * row+0x8
if dur < row+0xc: dur = row+0xc                            # floor
apply spell-mod type 0xa
if (SpellRec+0x18 & 0x30) == 0 and dur > 0:
    m = [obj+0x110 + 0x22c]                                # unit duration mult
    if |m - 1.0| > 2.384e-7 (0x8029d4):                    # epsilon
        dur = round_to_int( dur * m )                      # x87
```

**`GetMinMaxRange 0x6e3480`** (out `{min, max}`):

```
if SpellRec+0x18 & 0x404:  max = 100.0 (0x8118d4); return   # self-cast
row = SpellRange[ SpellRec+0x90 ]                           # [0xc0d79c + idx*4]
if row+0xc & 1:                                             # melee
    min = 0
    max = floor_to( [caster+0x110]+0x1f0 + [target+0x110]+0x1f0 + 1.3333 (0x80b058), 5.0 (0x80a1e8) )
else:                                                       # ranged
    min = row+0x4 ;  max = row+0x8
    if target is a unit: add the combat-reach sum to both  # guarded vs 0.0
# PvP bonus: if both units flagged 0x200d and target hostile:  max += 2.6667 (0x811914)
# inspect/UI: max = otherRec+0x118 * 0.01 (0x8029d0) * max
```

**`GetMinMaxRadius 0x6e3760`** resolves the float radii (via `GetSpellRadii 0x6e3800`) then rounds
half-away-from-zero to ints. The rounding is done in fixed point to avoid `lround`:

```
round(val) = trunc(2*val ± 0.5) >> 1     # +0.5 if val>=0 else -0.5; const 0.5 = 0x86f94c
```

`GetCurrentCastRadius 0x6e6350` (the AoE radius for the cursor): `radius = SpellRadius+0x4 +
caster_level([obj+0x110]+0x70) * SpellRadius+0x8`, defaulting to `0.0` for an absent RadiusIndex, taking
the larger of the two indices `SpellRec+0x160/+0x164`, then applying the float modifier.

**Range gates** (`IsTargetInRange 0x6e47b0`, `CheckGroundPointInRange 0x6e6810`) compute a squared
distance and compare against squared bounds, with no literal constants:

```
distSq = dx² + dy² + dz²
if distSq > max²: out of range
if distSq < min²: too close
```

Note that the `/5` and `/100` divides above are **integer reciprocal-multiplies** (e.g. `imul
0x51eb851f; sar edx,5` for `/100`) — they truncate, and are distinct from the x87 scaling on the same
line.

## The spell-modifier engine

The two modifier tables are `i32[64][29]` (64 family slots × 29 modifier-op columns), indexed `slot*0x1d
+ op`: **SPELLMOD_FLAT** at `0xcead60`, **SPELLMOD_PCT** at `0xcecb30`. `GetSpellModifiers 0x6e6b30`
computes the `{flat, pct}` totals of a given modifier type that apply to a spell — gated on the spell's
modifier-family id (`SpellRec+0x280` must equal `SPELLMGR+0x4c`, the modifier class category), then
accumulating over the 64 slots whose `SpellRec+0x284` SpellFamilyFlags bit is set; `pct` baselines at 100.

```
ApplySpellMod_int   [0x6e6af0]:  *val = (*val + flat) * pct / 100        # integer
ApplySpellMod_float [0x6e6bf0]:  *val = (flat + *val) * pct * 0.01       # const 0.01 = 0x8029d0
ApplySpellMod_clamped [0x6e6c30]: float modifier with a min/max clamp against ceiling 0xcf0b0c
```

The integer helper is what the cooldown/timing math calls with modifier-type bytes `0xb`/`0x15`/`0xe`/
`0xa`; the float helper is used by the range/radius math. `SetModifierClassCategory 0x6e6ca0` sets the
family selector `SPELLMGR+0x4c = chrClasses[classIdx]+0x3c`.

## Lua / FrameScript bindings

Five `Script::Spell*` functions are registered from the table at `0x86f950`:

| Lua name | function | does |
|---|---|---|
| `SpellIsTargeting()` | `0x6e6cd0` | `IsTargeting` → bool |
| `SpellCanTargetUnit("unit")` | `0x6e6d00` | resolves the unit token → `CanTargetUnit 0x6e6460` (read-only) |
| `SpellTargetUnit("unit")` | `0x6e6d90` | binds the unit via `0x6e5b10` (or fails with reason `0x59`) |
| `SpellStopTargeting()` | `0x6e6e30` | `StopTargeting 0x6e4900` |
| `SpellStopCasting()` | `0x6e6e80` | stops auto-repeat, else `AbortCast` (CMSG_CANCEL_CAST) |

## Cast-failure error display

`HandleCastFailed 0x6e1a00` is the central failure handler (called from ~25 sites). It first reverts the
spell's optimistic cooldown if appropriate, then maps the reason byte → a localized error string. The
reason→format-string table is the jump table at `0x6e2764` (in `0x6e23e0`), spanning the `WoW\Source\Magic`
cast-fail string resources; item/reagent specifics are formatted by `0x6e29b0` / `0x6e2a90` and shown via
`CGGameUI::DisplayError 0x496720`. `DisplayTargetingError 0x6e6a20` (codes 1..0xa, table `0x6e6ac0`) shows
the targeting-cursor errors.

## FrameScript events emitted

| event | when |
|---|---|
| `0x151` | cast-bar start (SPELL_START) |
| `0x152` / `0x153` / `0x154` | cast stop / interrupted / failed |
| `0x155` | cast-bar pushback (SPELL_DELAYED) |
| `0x156` | channel-bar start |
| `0x157` | channel-bar tick |
| `0x1b3` / `0x1b4` | auto-repeat start / stop |
| `0x26d` | auto-repeat stop signal (from `StopAutoRepeat 0x6ea080`) |
| `0x210` | cancel-aura of spell `0xa18` |

## The spell-visual / effect pipeline

The visual sub-system is three object families plus an auto-repeat engine, all rooted in fixed `.data`
globals.

### The `SpellVisual` scene/particle object

A heap object (size `0x144`) from pool `0xcee928`, tracked in the active list `0xcee970`. Layout:

| off | field |
|---|---|
| `+0x08` | name buffer (`SStrCopy` len `0x104`) |
| `+0x10c` | scene/model node ptr |
| `+0x110` | world position vec3 (set at Initialize) |
| `+0x11c` | caller-supplied kit/aux id |
| `+0x120` | f32 phase accumulator (Update advances by `dt·rate`) |
| `+0x124` | f32 scale/rate (set by the quality tier) |
| `+0x138` | spawned beam-segment list |
| `+0x140` | live particle/emitter element list |

Lifecycle: **Initialize `0x6eb930`** → per-frame **Update `0x6ebad0`** → **RenderSubmit `0x6ebee0`** →
**dtor `0x6eb850`**. Particle element nodes (`0x20` bytes) come from pool `0xcee8f0`.

`SpellVisual::Initialize 0x6eb930` sets the size by the graphics-quality CVar (cached object `0xcee9a8`,
read at `+0x28`):

```
quality 0  ->  +0x124 = scale * 0.33   (0x808300)
quality 1  ->  +0x124 = scale * 0.66   (0x81199c)
else       ->  +0x124 = scale          (×1.0)
```

It then builds an identity 4×3 transform, creates the scene node, and sets the model bounding box from
the radius (corner vectors `±radius`, half-scale `0.5`).

`SpellVisual::Update 0x6ebad0` is the particle emitter:

```
phase += Δt(0xc62510) * rate
if phase < 1.0: finalize
while phase >= 1.0:                       # one element per whole unit of phase
    phase -= 1.0
    allocate + zero a 0x20-byte node, link into +0x138
    r1 = CRandom uniform in [1,2)         # rand & 0x7fffff | 0x3f800000
    frac = r1 - 1.0
    sincos(theta) -> (sin, cos)           # engine sincos 0x44fd00
    offset = (frac*sin*scale, frac*cos*scale), plus +16.6667 (0x810e20) Z lift and +1/6 (0x803568)
    terrain ray-test gates the spawn; on hit, position scaled ×0.95 (0x8119a0) with -100/3 (0x80fee4) correction
    colour idx = ((CRandom-1.0) * 255.0(0x7ffe58) + 512.0(0x8029cc)) >> 0xe  -> table 0xc7b2c0
```

The per-particle placement/colour math draws from `CRandom` over a render particle pool, a models submit,
and a terrain ray — that emit path depends on fully-loaded render/world state and is read at runtime, not
expressible as a closed formula. The deterministic parts (the phase accumulator and the "still active"
verdict — `+0x124 fcomp 0.0` plus an empty-list check) are closed.

### The visual-kit / beam / chain system

A system object at `0xcee948` (created by `SpellVisualSystem::Initialize 0x6ec0e0`, a models-owned type),
driven per frame by **`UpdateAll 0x6eca20`**. Its owned timing math:

| address | formula | constants |
|---|---|---|
| `GetAttachPointDefault 0x6ec6f0` | attach height `= modelHeight·scale·0.75`, added to base Z | `0.75` = `0x8012cc` |
| `ComputeProgressFraction 0x6ec870` | fade-in/hold/fade-out envelope over a 4-keyframe timeline | `1.0` = `0x7ff9d8` |
| `ScheduleTimeline 0x6ec910` | keyframe offset `= duration × fraction` (ftol) | fraction is a passed f32 |
| `UpdateAll 0x6eca20` | global fade clock `frac = (now − last) · 0.001` (ms→sec) | `0.001` = `0x801360` |
| `CreateChainVisual 0x6ecbd0` | segment lifetime `= effectDuration(int) · 0.001` | `0.001` = `0x801360` |

The intensity envelope (`0x6ec870`) is the durable cast/channel fade shape:

```
now < start          : 0
start <= now < mid1   : (now-start)/(mid1-start)        # rising
mid1 <= now < mid2    : 1.0
mid2 <= now < end     : 1.0 - (now-mid2)/(end-mid2)     # falling
now >= end            : reset, 0
```

`ScheduleTimeline 0x6ec910` reads `GetDuration` and sets the global cast-visual timeline keyframes
`0xcee8d8..0xcee8e4` to `now {+0, +offset, +duration, +duration+100ms}`.

### Auto-repeat / wand cadence

`GetAutoRepeatSpellKey 0x6e9fd0` reads the key `0xceac30`; `StopAutoRepeat 0x6ea080` fires events `0x26d`
then `0x1b4` and runs the cast-bar cancel sequence; `TryConfirmPendingAutoRepeat 0x6ea1e0` re-validates
and re-arms a pending recast when the target still matches.

### Init-time precomputed constants

A set of CRT dynamic-initializer trampolines (interleaved in the visual region, with no `.text` caller —
reached only via the CRT init array) precompute spell-data scaling constants into the `.data` region
`0xceca94..0xcee830`, consumed by the cast / object-layer code. They are not runtime code, but their
resolved values matter to an implementer:

| global | value | derivation |
|---|---|---|
| `0xceca94` | 0.58333 | −21.0 × (−1/36) |
| `0xcecb28` | −148.75 | 0.58333 × −255.0 |
| `0xceac24` | 0.5 | 18.0 × (1/36) |
| `0xceac40` | 128.0 | 0.5 × 256.0 |
| `0xcecabc` | 0.24 | 1.0 ÷ 4.16667 |
| `0xceac20` / `0xcecacc` / `0xcecaa0` / `0xceac18` / `0xcead48` | 100.0 | 10.0² |
| `0xceac28` | 0.05 | 0.5 ÷ 10.0 |
| `0xcecae8` | 123.457 | 11.1111² |
| `0xcead50` | 30.864 | 5.5556² |
| `0xcecae4` | 25.0 | 5.0² |
| `0xcecb1c` | 900.0 | 30.0² |
| `0xceac38` | 14400.0 | 120.0² |
| `0xcead58` | 400.0 | 20.0² |
| `0xcecab8` | 0.25 | 0.5² |
| `0xceca9c` | 9.0 | 3.0² |
| `0xcecad0` / `0xcecae0` | 10000.0 | 100.0² |
| `0xcead54` | 5.0 | immediate `0x40a00000` |
| `0xceca98` | 2500.0 | 50.0² |
| `0xcee830` | 1600.0 | 40.0² |
| `0xceac1c` | 2.25 | 1.5² |
| `0xcee8d0` | +inf | `0x7f800000` (no-max sentinel) |
| `0xcee98c` | 6.28319 | 2π |
| `0xcee90c` | 0.159155 | 1/(2π) |

The squared values are distance-squared thresholds; the `−21/36`, `18/36`, `×256/−255` chain and the
`2π` / `1/(2π)` pair are angle/period conversions.

## The DBC records the subsystem reads

These records are owned by [dbc](dbc.md), but the offsets the spell code consumes are load-bearing.

**SpellRec** = `[0xc0d788 + id*4]` (count `0xc0d78c`):

| offset | field |
|---|---|
| `+0x08` | category |
| `+0x18` | Attributes (bit25 = cooldown-on-event; `& 0x40` passive; `& 0x404` self-cast; `& 0x800000` usable-in-combat) |
| `+0x1c` / `+0x20` / `+0x24` | AttributesEx flags |
| `+0x34` | target-type word (seeds flag_word) |
| `+0x48` | DurationIndex |
| `+0x4c` / `+0x50` | RecoveryTime / CategoryRecoveryTime |
| `+0x70` | base level |
| `+0x7c` | mechanic / skill index |
| `+0x80` / `+0x84` | cast-time base / per-level |
| `+0x88` / `+0x8c` | (likely min cast-time) base / per-level |
| `+0x90` | RangeIndex |
| `+0x94` | projectile Speed (f32) |
| `+0x98` | visual / auto-repeat key |
| `+0xa0` / `+0xa8` | totem-category / reagent slots |
| `+0xe8` / `+0xec` | power type / cost |
| `+0xf4` / `+0xf8` / `+0xfc` | effect-type words (3 slots) |
| `+0x148` / `+0x14c` / `+0x150` | implicit-target enums (drive the 62-case switch) |
| `+0x160` / `+0x164` | RadiusIndex (2 slots) |
| `+0x16c` | aura-type / RadiusIndex per effect |
| `+0x1e0[locale]` | localized name array |
| `+0x270` | cast-time skill-scale factor |
| `+0x274` / `+0x278` | GCD category / GCD duration |
| `+0x280` / `+0x284` | modifier family id / SpellFamilyFlags |

**SpellRange** = `[0xc0d79c + idx*4]` (count `0xc0d7a0`): `+0x4` minRange (f32), `+0x8` maxRange (f32),
`+0xc` flags (bit0 = melee).

**SpellRadius** = `[0xc0d7b0 + idx*4]`: `+0x4` radius, `+0x8` per-level multiplier.

**SpellDuration** = `[0xc0d878 + idx*4]` (count `0xc0d87c`): `+0x4` base, `+0x8` per-level, `+0xc` floor.

**SpellVisual** = `[0xc0d74c + idx*4]` (count `0xc0d750`).
