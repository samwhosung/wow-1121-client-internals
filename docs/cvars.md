# CVars — the console-variable configuration registry

The client's configuration registry: a name→value store that every subsystem reads its tunables from.
The client owns the registry *mechanism* — registration, the value-coercion / flag / latch / dirty /
change-callback state machine, and the `Config.wtf` load — while the container and all string/number
primitives are delegated to the Storm library. It is one of the first subsystems wired at startup
(`0x402350` saturates with `CVar::Register`/`Set`/`LoadFile` calls). The translation unit is
`Source\Console\ConsoleVar.cpp`.

## The `CVar` record — 0xc4 (196) bytes

Each registered variable is a 196-byte record. Storm's intrusive hash-table and list links are embedded
directly in the record (offsets `0x04`–`0x10`).

| Offset | Width | Field | Meaning |
|---|---|---|---|
| `0x00` | u32 | hashKey | cached `SStrHashHT(name)`, the bucket-chain match key |
| `0x04`–`0x10` | 4×u32 | Storm TSHashTable/TSLink links | intrusive bucket-chain + global all-CVars list anchors |
| `0x14` | char* | name | CVar name (`SStrDupA`) |
| `0x18` | u32 | category | registration group id |
| `0x1c` | u32 | flags | bit0 registered · bit1 (0x2) latched · bit2 (0x4) read-only · bit31 hidden |
| `0x20` | char* | value | current value string (canonical form) |
| `0x24` | f32 | valueAsFloat | value parsed → f32 (stored result of `SStrToFloat`) |
| `0x28` | i32 | valueAsInt | value parsed → i32 (`SStrToInt`) |
| `0x2c` | u32 | changeCount | incremented on every accepted Set |
| `0x30` | char* | defaultValue | factory default (lazily dup'd) |
| `0x34` | char* | resetValue | reset target (lazily dup'd) |
| `0x38` | char* | latchedValue | pending value applied at the latch boundary (flag bit1) |
| `0x3c` | char[0x80] | description | inline help-string buffer (`[0x3c, 0xbc)`) |
| `0xbc` | code* | callback | change-callback `bool(*)(newVal, arg)`, invoked BEFORE apply |
| `0xc0` | u32 | callbackArg | opaque arg passed to the callback |

The change-callback at `0xbc` is invoked **before** the new value is applied; it returns a bool that
decides whether the Set is accepted.

## The registry container — a Storm `TSHashTable<CVar>`

The registry is a single Storm intrusive hash table plus a `TSExplicitList`, over a fixed record pool.
Its constructor (`0x63d020`) is a static-init constructor; its effect is the initial empty registry.

Global locations:

| Global | Address | Role |
|---|---|---|
| pool object | `0xc4ed94` | vtable `0x80e1dc`; slot `+4` = alloc |
| global-list link / sentinel / head | `0xc4ed98` / `0xc4ed9c` / `0xc4eda0` | all-CVars list anchors |
| bucket base | `0xc4edb0` | bucket stride `0xc`, chain head at `+8` |
| hash mask | `0xc4edb8` | `0xffffffff` means empty |
| config-dirty flag | `0xc4edd8` | set on any value change |

The in-memory shape of the table object at `0xc4ed94` is
`{m_alloc.vtable@0, linkOffset@4, listLink@8/c, loadCount@10, bucketVec{cap,count,data,grow}@14..20, mask@24}`.
Each bucket is `0xc` bytes: `{linkOffset, TSLink head{fwd, back}}`, where the `back` pointer's LSB is
tagged to mark the head sentinel.

**Lookup.** Compute `h = SStrHashHT(name)`; the bucket is `base + (h & mask)*0xc`; walk the chain
matching `rec.hashKey == h && SStrCmpI(rec.name, name) == 0`.

**Insert** (Register): same probe; on a miss, grow if needed, pool-allocate a record, link it, populate
it, and Set its default.

## The state machine — CVar control flow

These functions implement the registry's control flow. The only floating-point operation in the entire
translation unit is the `fstp` that stores Storm `SStrToFloat`'s result into `valueAsFloat`; everything
else is string and integer manipulation delegated to Storm.

| Function | VA | Behaviour |
|---|---|---|
| Register | `0x63db90` | hash + lookup; on hit, merge flags/callback then Set; on miss, allocate, populate, copy name + description, then Set the default |
| Set | `0x63df50` | if change is allowed, run `callback(newVal, arg)`; if it returns true, bump `changeCount`; if the var is **latched** (flag bit1) store to `latchedValue` (deferred), else `InternalSet` |
| InternalSet | `0x63e0b0` | `SStrCmpI` vs current; on change, free+dup the value string, parse `SStrToInt`→i32 and `SStrToFloat`→f32; lazily initialise default/reset strings; set the global config-dirty flag |
| Reset | `0x63dff0` | `InternalSet` to the reset string |
| Default | `0x63e010` | `InternalSet` to the default string |
| Update | `0x63e060` | apply a pending `latchedValue` via `InternalSet`, then free the latched slot |
| LoadFile | `0x63d820` | `SFile::Load` the `WTF\<file>` config, strip a UTF-8 BOM, `SStrTokenize` lines, and `ConsoleCommandExecute` each (the `SET name value` directives) |

The latch mechanism (flag bit1) lets a Set be deferred: the new value is parked in `latchedValue` and
only applied to the live value when `Update` runs at the latch boundary.

## The TSHashTable container helpers

The container is a generic Storm `TSHashTable<CVar>`; its helpers are `this`-relative methods with no
absolute global references.

**Pool vtable at `0x80e1dc` (4 slots):**

| Slot | VA | Role |
|---|---|---|
| `[0]` | `0x63e2b0` | free-node (CVar destructor + `SMemFree`) |
| `[1]` | `0x63e2e0` | alloc+link-node (`SMemAlloc`, the record allocator) |
| `[2]` | `0x63e460` | container deleting-destructor |
| `[3]` | `0x63e3a0` | clear-to-empty (`mask ← 0xffffffff`) |

**Link primitives:** `0x63e520` `TSLink::Unlink` (circular doubly-linked splice-out, the one inlined
idiom) · `0x63e6f0` `LinkNode` (insert-after / head-insert) · `0x63e6d0` `TSGetLink` · `0x63ed80`
prev-slot tagged-pointer resolver · `0x63e9c0` / `0x63ec30` sentinel init.

**Teardown loops** (composed from Unlink): `0x63e560` (drain bucket+head) · `0x63e5f0` (free array) ·
`0x63ebe0` (drain chain) · `0x63e910` (drain + optional delete).

**Resize / rehash:** `0x63e7b0` (grow to 4 buckets + rehash) · `0x63e9f0` (chain > `0xd` ⇒ double
capacity + migrate) · `0x63ec40` `TSGrowableArray::resize` · `0x63ee10` (realloc via `SMemReAlloc`). The
sizing math is integer-only: `0x63edb0` (highest power-of-2 ≤ n, clamped at `0x15`) and `0x63edf0`
(round-up-to-multiple). A chain longer than `0xd` (13) entries triggers a capacity doubling.

## Persistence and lifecycle

| Function | VA | Behaviour |
|---|---|---|
| SaveConfig | `0x63d980` | dirty-gated write of every non-default *registered* cvar as `SET %s "%s"\n` to `WTF\<config>` (`CreateFileW`/`WriteFile`/`CloseHandle`) — the persistence half of LoadFile |
| DeleteConfigFile | `0x63d910` | `DeleteFileW` `<name>`, falling back to `WTF\<name>` |
| Shutdown | `0x63daf0` | SaveConfig, unregister the 4 console commands, drain the all-CVars list, free every record via pool vtable slot 0 |
| CreatePathDirectories | `0x63d440` | split a path on `\`, `CreateDirectory` each prefix (tolerating already-exists) |
| ~CVar | `0x63d280` | unregister + free the 5 owned strings + unlink both TSLinks |
| member zero-init | `0x63d260` | |

## Console-command handlers

Registered by Initialize:

| Command / handler | VA | Behaviour |
|---|---|---|
| `set` | `0x63d500` | tokenize name+value → Lookup/Set; otherwise Register a category-5 cvar |
| `reset` | `0x63d590` | |
| `default` | `0x63d640` | |
| `cvarlist` | `0x63d6f0` | |
| per-CVar callback | `0x63dde0` | installed under each cvar's name by Register — with an empty arg prints `CVar "%s" is "%s"`, otherwise Sets |
| Lookup-hidden-only | `0x63dec0` | returns a record only when flag bit31 (hidden) is set; the `Script::` binding path |

`DumpToFile` (`0x63e1a0`) is a real `ConsoleVar.cpp` function — the unregistered file-dump sibling of
`cvarlist` — but it has **zero references anywhere in the image**: dead code.

## What is delegated to Storm

The client owns the `CVar` record layout and the registration / lookup-probe / set-coerce-latch-dirty /
change-callback / reset-default / `Config.wtf`-load control flow and state transitions. Everything below
that is delegated:

- **Storm library:** the hash-table container; `SStrHashHT` (hash); `SStrCmpI` / `SStrCmp` (compare);
  `SStrDupA` / `SStrCopy` / `SMemFree` (strings); `SStrToFloat` / `SStrToInt` (the numeric coercion that
  fills `valueAsFloat` and `valueAsInt`); `SStrTokenize` / `SStrPrintf`; `SFile::Load` (config IO).
- **CRT:** the static-init constructor / atexit.
- **[Console](console.md):** command registration and line output.
