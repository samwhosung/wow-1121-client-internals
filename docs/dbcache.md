# DBCache — the server-streamed record cache

The client cannot ship every creature, item, quest, and NPC definition in its data files — most of that data lives on the server and is streamed on demand. `DBCache` (`WoW\Source\DB\DBClient\DBCache.cpp`) is the client-side cache for those records: when the client needs a record it lacks (a creature/item/quest/etc. id), it sends a `CMSG_*_QUERY`, queues a callback, and the server streams back a `SMSG_*_QUERY_RESPONSE`; the registered handler deserializes the record into a per-type cache (`DBCache<T>`) and persists it to a `.wdb` file on disk. This is distinct from the static [DBC](dbc.md) tables shipped in the MPQs — `DBCache` is for the server-authoritative data that varies per realm.

## The query/response flow

At startup, `0x402b16 call 0x554ff0` (RegisterHandlers) installs **14 SMSG query-response handlers** through the net layer's `SetMessageHandler` (`0x5ab650`). `DBCacheInitialize` (`0x554f40`/`0x554f50`) constructs the **13 `DBCache<T>` singletons** and loads their backing `.wdb` files.

The runtime cycle per record:

1. A consumer misses on `GetRecord`. The cache fires the type's `CMSG_*_QUERY` (via net) and queues a `DBCACHECALLBACK` on the pending node.
2. The server replies; the response handler calls `DBCache<T>::Receive`.
3. `Receive` reads the cache key from the response (net `CDataStore`), does a hash lookup/insert (Storm `TSHashTable` + a `TSExplicitList` LRU), then calls **`T::Read(CDataStore*)`** to decode the record fields into the cache node.
4. The node is marked ready, the queued callbacks fire and drain, and the record is appended to the `.wdb` file.

The cache state machine itself does no record-format arithmetic. The hash bucket is a generic `index = key & mask` (`0x556e7c`); the `.wdb` header is a constant emit plus an equality check on read (there is no checksum/CRC, no timestamp, no TTL). Wire reads/writes delegate to net (`CDataStore::Get*`/`Put*`, `Set`/`Clear`/`SendMessageHandler`); hash and list node memory comes from Storm; allocation, strings, and `.wdb` file I/O go to host (`SMemAlloc`/`SMemFree`, `SStr*`, `OsCreateFile`/`Directory`, `SFile::Read`/`Close`). The format-defining logic is the per-record `T::Read` decoders below.

## The per-record decoders

Each cached type has its own decoder with a uniform ABI:

```
void __thiscall T::Read(this = ecx = record-base, CDataStore* ds = [esp+4])    ; ret 4
```

The cache key is consumed by `Receive`, not by the decoder (the one exception is `ItemText`, which reads its own id inline). There are two string-storage modes:

- **`char*` (heap)** — strings are duplicated onto the heap via `SStrDupA` (`0x64a620`) and the struct holds a pointer: Creature, GameObject, ItemName, ItemStats, NPCText.
- **inline `char[]`** — `GetCString` copies straight into a fixed buffer inside the struct: Guild, PageText, Petition, Quest, Name, PetName, ItemText.

| Store type | `Receive` | Decoder | Record size | Notes |
|---|---|---|---|---|
| CreatureStats_C | 0x556e20 | 0x7c8120 | ~0x34 | name[4] char* (empty-reuse), subname, int×7, bool×2 |
| GameObjectStats_C | 0x5588e0 | 0x7c87c0 | 0x7c | int×2, 5×char* (reuse), int×24 |
| ItemName_C | 0x55a310 | 0x7c8dc0 | 4 | single char* |
| ItemStats_C | 0x55bdb0 | 0x7c9640 | ~0x1d4 | int/char*/float[5]×2/u32[5]/spell-slots/desc — the large one |
| NPCText | 0x55d8b0 | 0x7c9fa0 | 0x140 | 8-loop: float prob, 2×char*, lang, 3×emote |
| GuildStats_C | 0x5611b0 | 0x62f260 | 0x2f8 | u32, name[0x60], 10×rank[0x40] inline, int×5 |
| QuestCache | 0x562c80 | 0x6dbf60 | ~0x18e0 | int×15, float×2, 4×big inline text, 4×4 objectives |
| PageTextCache_C | 0x564ab0 | 0x7cabd0 | 0x404 | char[0x400] inline, next-id |
| PetNameCache | 0x566470 | 0x6dc290 | 0x58 | u32, char[0x50] inline, u32 |
| CGPetition | 0x567fa0 | 0x7c7a30 | ~0x13c4 | **variable-length**: count@+0x13bc bounds a signer-name loop |
| NameCache | 0x55f310 | 0x6dc1f0 | 0x144 | guid-keyed; guid, name[0x30], char[0x100], race/gender/class |
| ItemTextCache_C | 0x569d00 | inline | — | int id (high-bit evict), char[8000] inline |

`WardenCachedModule` (the anti-cheat module blob cache, `0x6ca5c0`) is **not** in this set: it has no SMSG query-response handler — it is fed by the Warden path, not a field decoder — so it is decoded by its own code, not a `T::Read`.

## State structures

**`DBCache<T>` instance** — 0x3c (60) bytes; vtable `0x80912c`. The 13 singletons sit in `.data` at `0xc0e0c0..0xc0e354`, stride `0x3c`.

| Offset | Field |
|---|---|
| `+0x00` | vtable |
| `+0x08`/`+0x0c` | LRU `TSExplicitList` head |
| `+0x10` | load counter |
| `+0x18` | bucket count |
| `+0x1c` | bucket-array base |
| `+0x24` | hash mask (= count − 1) |
| `+0x28` | `.wdb` FourCC |
| `+0x2c` | filename |
| `+0x30` | CMSG opcode |
| `+0x38` | query-includes-GUID flag |
| `+0x39` | enable flag |

**`DBCACHEHASH` node** — the per-record cache entry:

| Offset | Field |
|---|---|
| `+0x00` | key |
| `+0x04`/`+0x08` | hash-chain link |
| `+0x0c`/`+0x10` | LRU link |
| `+0x18` | record payload (inline) |
| `+0x4c` | key copy |
| `+0x50` | valid flag |
| `+0x5c` | `DBCACHECALLBACK` list head |
| `+0x61` | dispatching byte |

**`DBCACHECALLBACK`** — 0x20 bytes:

| Offset | Field |
|---|---|
| `+0x08` | invoke fn |
| `+0x10` | owner-id (64-bit) |
| `+0x18` | data |

Callbacks fire on record arrival, keyed by the 64-bit owner id (cancellation logs `DBCache::CancelCallback ignored for id %016I64X.`).

## Contracts

**Hash.** Open-chained. `index = key & mask`; the bucket head is `base + index*12 + 8`. Growth is the generic Storm container grow.

**LRU.** Access reorders the node to most-recently-used. There is **no TTL and no size-cap auto-eviction** — eviction is explicit: a response carrying a key with the high bit set means evict, or `SMSG_INVALIDATE_PLAYER` (`0x31c`) triggers a remove-by-key (`0x556ff0`).

**Callback.** A `GetRecord` miss queues the callback on node `+0x5c` and sends the CMSG. On the response: set `+0x61 = 1` (dispatching), run `T::Read`, fire and drain the callback list, then clear `+0x61 = 0`.

**`.wdb` persistence.** Files live at `WDB/<name>.wdb` with a **20-byte header**: `[FourCC | build 0x16f3 (=5875) | locale | recordSize | version 1]`. Records are capped at `0x4000` bytes. Byte-level I/O goes through `CDataStore` (net) for serialization and `SFile` (host) for the file; the record body is decoded by the same `T::Read` used on the wire.
