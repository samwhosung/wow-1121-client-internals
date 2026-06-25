# Core loop — the per-frame event scheduler and event bus

The architectural spine the rest of the client plugs into. Core is not a fixed `input(); update();
render(); present();` sequence but a single **per-frame event-scheduler loop** plus a **dispatch
primitive** (an event bus) that fans each frame phase out to the handlers registered for it. Subsystems
(render, world, net, UI) register on a Context's per-category lists rather than living in the loop, so
core owns the loop, the scheduler, the dispatch, the timing/`dt`, and the Win32 message intake.

All addresses are VAs (ImageBase `0x400000`).

## Bootstrap — PE entry to the frame loop

The chain from process start to the frame loop:

| VA | What |
|---|---|
| `0x401000` `entry` | PE entry (RVA 0x1000). 10 bytes: `FUN_00645010(); FUN_004098c0();` |
| `0x645010` | CRT pre-init (security cookie / heap bootstrap) |
| `0x4098c0` `__tmainCRTStartup` | MSVC CRT startup; `GetVersion`/`GetCommandLineA`/`GetStartupInfoA`, then a 4-arg `WinMain(GetModuleHandleA(0), 0, lpCmdLine, nShowCmd)` |
| `0x404100` WinMain (wrapper) | `_controlfp(0x9001f, -1)` (FPU control word), process-token ACL hardening (`0x42a1c0`), then the real body `0x402120` |
| `0x402120` WinMain body | identified by the string `"World of WarCraft (build 5875)"`; runs init `0x402350`, then loop-entry `0x420be0`, then teardown `0x403bc0` |
| `0x420be0` → `0x420c00(1)` | loop-entry thunk; sole caller of the frame loop |

The `_controlfp(0x9001f, -1)` in the WinMain wrapper sets the x87 control word to **PC_53**
(53-bit / `FPCW = 0x027F`) for the whole process — this matters for several core computations below,
which round an f64 intermediate to f32 (double-rounding) rather than computing in pure f32. The CRT
internals below `0x645010`/`0x4098c0` are statically-linked MSVC runtime.

## The frame loop — `0x420c00`

A generic engine event-scheduler run-loop. It runs on a dedicated thread (named `Engine_%x`) and is
**not** a textbook PeekMessage idle-pump — OS messages are drained out-of-band, and the per-frame
tick fires when a scheduler timeout elapses. One loop iteration is one frame:

```
FUN_00437060()                                  // register this thread's engine record
while (true) {
  if (WaitForSingleObject(quitEvent, 0) == 0) break;   // FUN_006599c0; signaled => quit
  ctx = FUN_00421430()                          // pop the next-due Context (scheduler)
  wait = (ctx != 0) ? ctx[+0x3c] - now() : -1   // time until this Context is due
  if (DAT_00883f0c == 0) r = WaitForSingleObject(quitEvent, wait)   // sleep until due / quit
  else { FUN_0043f050(); r = 0x102 }            // forced-run path -> WAIT_TIMEOUT
  if (ctx != 0) {
     start = now()
     if (r == 0x102) {                          // *** WAIT_TIMEOUT => run the frame tick ***
        FUN_00420e20()                           // [1] frame begin
        FUN_00428510()                           // [2] run due timed callbacks
        if (ctx[+0x44] & 2) FUN_00423920()       // [3] drain OS input (conditional)
        FUN_00420ff0()                           // [4] phase dispatch
        FUN_00424ad0()                           // [5] drain deferred work
        FUN_00420f70()                           // [6] dt update  <-- the per-frame "update(dt)"
        FUN_00421030()                           // [7] post-update phase
     }
     ... reschedule ctx (FUN_00421570 / FUN_00420e60) ...
  }
}
FUN_004211d0(); FUN_004371e0()                  // teardown
```

`FUN_006599c0` @ `0x6599c0` is a thin `WaitForSingleObject((HANDLE)*p1, p2)`. So
`WaitForSingleObject(quitEvent, 0) == 0` (WAIT_OBJECT_0) means the quit event is signaled → exit the
loop; and the per-frame work runs on the **`0x102` = `WAIT_TIMEOUT`** return — i.e. the scheduled
inter-frame wait elapsed without a quit. (`0x102` here is the Win32 `WAIT_TIMEOUT` constant, not
WM_CHAR.) Because the frame fires on the Context's scheduled timeout rather than on message arrival,
frame pacing is owned by the Context's due-time, independent of OS message volume.

## The per-frame `dt` — `0x420f70`

The load-bearing update tick computes elapsed time and broadcasts it:

```
dt_seconds = (float)(now_ms - ctx[+0x40]) * 0.001f ;   ctx[+0x40] = now_ms ;   FUN_004245b0(&dt_seconds)
```

The constant `_DAT_00801360` = bytes `6f 12 83 3a` = `0x3a83126f` = **`0.001f`**, so the multiply is
**milliseconds → seconds**. The clock `now_ms` comes from `0x42b790`→`0x42b750` = `GetTickCount()`
(millisecond resolution) by default, with an alternate high-resolution path gated on `DAT_00884c80`.

Precision: the multiply is done in x87 at PC_53 — an f64 multiply of `(f32)delta` by `0.001f`, then
rounded to f32. This is **not** the same as `delta as f32 * 0.001f`; the double-rounding can differ
in the last bit. Time is therefore integer-millisecond internally, surfaced to subsystems as `float`
seconds.

## The event bus — `0x4245b0`, dispatch over 29 categories

`0x4245b0` is `__fastcall(ecx = ctx, edx = category, [stack] = payload)`. It walks the Context's
handler list for that category — base `ctx + 0x5c + category*0xc` — and invokes each registered
handler in priority order:

```
(*handler->fn)(ecx = payload, edx = handler->arg)
```

Each handler node is `0x18` bytes (RTTI `.?AUEvtHandler@@`):

| Offset | Field |
|---|---|
| `+0x00` | next |
| `+0x04` | prev |
| `+0x08` | `fn` (handler function pointer) |
| `+0x0c` | `arg` (handler argument) |
| `+0x10` | `prio` (f32 priority) |
| `+0x14` | guard |

The **category index lives in EDX**, which the decompiler drops at the call sites (it renders
`0x4245b0(payload)` and hides the register). Reading the decompile alone makes every call look like
"category 0"; the truth is read from the disassembly (`mov edx, imm` before each `call`). Internally
`0x4245b0` does `mov %edx,-8(%ebp)` then `lea 0x5c(%esi,ecx,4)` with `ecx = category*3`. The 29 lists
are an event/handler dispatch table — an event bus — not per-frame render phases. The dispatch
re-snapshots the list each iteration, so handlers may add or remove handlers during a dispatch
(re-entrant-safe). The category space is **`[0, 0x1d)` = 29**, bounds-checked by the public register
API.

Registration / dispatch / removal:

- **`0x424f00` AddHandler primitive** `(ecx=ctx, edx=category, fn, arg, prio:float)` — allocate a
  `0x18`-byte handler node and **insert sorted ascending by `prio`** into `ctx+0x5c+category*0xc`;
  ties keep registration order.
- **`0x41fca0` public AddHandler** — bounds-checks `category < 0x1d`, then forwards to `0x424f00`.
  Its callers are the subsystems outside the loop — that is the seam where each category's meaning is
  bound.
- **`0x425000` / `0x41fd90`** — RemoveHandler (sweeps all 29 lists) and its public wrapper.
- **`0x424710` payload demux** — keyed on the category, runs *before* the handler walk and updates
  the Context's input-state mirror at `+0x1c4`/`+0x1d0`: category 2 → `0x424790` (flush the key-event
  queue), 8/9 → `0x424810` (add/remove a keyed event, synthesizing key-repeat), 0xb/0xe → `0x424920`
  (set/clear mouse-button mask bits). Other categories skip the demux. The cat-2 path bounds-checks
  against `0xc` and routes through a byte-map into a 4-entry jump table.

Dispatch order in practice: registration is ascending-by-`prio` insertion, and the dispatch walks the
list **tail-first** while handlers keep returning nonzero — so the **highest priority runs first**, and
equal priorities run LIFO.

## The category map

A category is named only where core has an emitter or a default handler; the full naming of the rest
lives with the subsystem that registers for it.

| Category | Emitted by / meaning |
|---|---|
| `0` | not emitted by core |
| `1` | activate |
| `2` | char (flush) |
| `3` | teardown (dispatched by `0x420e60`) |
| `4` | teardown / end (frame loop + Context-ctor default handler, prio `1000.0f`) |
| `5` | the dt tick (payload `&dt`), emitted by `0x420f70` |
| `6` | update, emitted by `0x420ff0` |
| `7` | frame-begin (frame loop + Context-ctor default handler, prio `1000.0f`) |
| `8` / `9` | key down / up |
| `0xa` | internal only (key-down → repeat promotion) |
| `0xb` / `0xe` | mouse button down / up |
| `0xc` / `0xd` | mouse / mouse wheel |
| `0xf` / `0x10` | further input (window/mode-change, mouse position) |
| `0x11` | post-update, emitted by `0x421030` |
| `0x12 … 0x1a` | no in-core emitter |
| `0x1b` / `0x1c` | resize / locale |

The Context constructor wires default handlers into categories `7` and `4` at priority `1000.0f`. The
deferred drain `0x424ad0` replays a queued node's stored category (`node+0xc`).

## The Context object model

The loop's unit of work is a **Context** — one cooperatively-scheduled engine task (registrar
`0x422090`, string `"Context (interactive) %u idleTimeout"`).

**Context** — size **`0x214`**, vtable `0x801364` (slot 0 = destructor `0x423240`, slot 1 = the
due-time comparator `0x422970`):

| Field | Meaning |
|---|---|
| `+0x00` | vtable (`0x80136c` during construction → `0x801364` when complete) |
| `+0x2c` | **run-state**: `0` runnable → `1` shutdown → `2` terminated |
| `+0x30` | embedded **heap-node hook** (own vtable `0x801368`); `node+4` = owner back-ptr, `node+8` = heap index, `node+0xc` = `+0x3c` |
| `+0x3c` | **due-time (ms)** — the min-heap key; the loop's sleep deadline |
| `+0x40` | **last-tick timestamp (ms)** — the `dt` anchor |
| `+0x44` | **flags**: bit0 frame-begun · bit1 interactive/has-update-phase · bit2 update-pending |
| `+0x48` / `+0x4c` | idle-timeout min / max (ms) — sleep clamps |
| `+0x50` / `+0x54` | load-balance cost: previous / target (`1000` interactive, else `1`) |
| `+0x58` | rebalance-needed flag |
| `+0x5c … +0x1b4` | **29 event-category handler-lists** (intrusive list-heads, stride `0xc`) — the `0x4245b0` fan-out targets |
| `+0x1b8` | deferred-work queue (tag dword `= 4`) |
| `+0x1c4` / `+0x1d0` | input-state mirror: pending-event/button mask, and the key-event queue the `0x424710` demux maintains |
| `+0x1f4` | timed-callback list (`+0x1f8` size, `+0x1fc` head; record: `+0x10` due-ms, `+0x18`/`+0x20` fn-ptrs, `+0x28`/`+0x2c` args) |

`Context+0x28` is a reserved, unread field.

**EvtSlot** — size **`0x38`**, the per-slot scheduler object held in the `DAT_00883f18[]` table (count
`DAT_00883f10`, all linked in list `DAT_00883ee8`). It owns a **binary min-heap of Contexts** (heap
sub-object at `+0x24`: `+0x28` element count, `+0x2c` array ptr, `+0x34` element-base-offset `= 0x30`)
plus load accounting (`+0x0c` refcount, `+0x10` load, `+0x18` member count). The heap orders Contexts
by `node+0xc` (= `Context+0x3c`, due-time) via comparator `0x422970`.

The two structs must not be conflated: `+0x2c`/`+0x28` are the *run-state* / an unused word on a
**Context**, but the *heap count* / *heap array* on an **EvtSlot**. `0x420e60` writing the literal `2`
to `Context+0x2c` (run-state) is the disambiguator.

## The scheduler — a per-slot min-heap

Scheduling is a per-slot binary min-heap keyed on due-time, with load-balanced migration across slots
by a per-Context cost (`1000` for the interactive Context, `1` otherwise).

- **Pop next-due** `0x421430` — array binary-heap **delete-min** on `EvtSlot[id]`: take the root
  (soonest due), move the last element to the root, decrement count, then sift down choosing the
  smaller child via the comparator (**tie → right child**). Returns the next-due Context (`0` if the
  heap is empty).
- **Reschedule** `0x421570` — write `Context+0x3c` = next-due (re-sift in place via `0x421860` on a
  key change), recompute cost (`+0x50/+0x54/+0x58`), optionally **migrate** the Context to a
  less-loaded slot (walking the `DAT_00883eec` slot list), then sift-up insert into the chosen slot's
  heap. The migration only picks *which* slot a Context lives in; *when* it runs is owned by the
  due-time heap, so moving slots is timing-invariant.
- **Find-or-create slot** `0x421080` — pick the least-loaded / first-empty slot in `DAT_00883f18[]`;
  allocate a `0x38` EvtSlot (heap element-base `0x30`) and link it into `DAT_00883ee8` when empty;
  bump refcount.
- **Re-sift** `0x421860` — delete-at-`i` (sift-down filling with last), then re-insert via sift-up
  (stop when the parent compares `<=` the element); `ecx = node` (`+4` container, `+8` index).

The comparator `0x422970` computes `a.due <= b.due`, but on a **wrapping** subtraction:
`(i32)(a - b) <= 0`, not `a <= b`. This matters when millisecond timestamps wrap.

## The tick phases

The `WAIT_TIMEOUT` block runs these in order; each "dispatch" is `0x4245b0` on the named category:

1. `0x420e20` **frame-begin** — idempotent (`flags&1` guard); set bit0, stamp `+0x40 = now_ms`,
   dispatch category **7**.
2. `0x428510` **timed-callback runner** — walk the due-ordered list at `+0x1f8`/`+0x1fc`; stop at the
   first not-yet-due record; invoke each due callback *with the lock released* (`(*fn)()` or
   `(*fn)(arg1,arg2)`), recycling spent records.
3. `0x423920` **OS input drain** (guarded by `flags&2`) → pump `0x42c9f0` → event-switch `0x4239a0`
   (which emits the input categories 1/2/8/9/0xb–0x10/0x1b/0x1c).
4. `0x420ff0` dispatch category **6** (unless run-state `== 1`).
5. `0x424ad0` **deferred-work drain** — atomically detach the `+0x1b8` queue, dispatch each node via
   `0x4245b0(category = node+0xc, payload = node+0x10)`, splice back any nodes re-enqueued during the
   drain.
6. `0x420f70` **dt update** (above) — dispatch category **5** with `&dt`; sets `flags|4`
   (update-pending) when `flags&2`.
7. `0x421030` **post-update** — if `flags&4`, clear bit2 and dispatch category **0x11**.

**Teardown** `0x420e60` — when a Context reaches run-state `1`: frame-begin if needed, dispatch
category **3**, set run-state `2` under lock, drain deferred work (`0x424ad0`), dispatch category
**4**, run frame-end, then call the Context's own virtual destructor `(**ctx)(1)` = `0x423240`.

## The window-event switch — `0x4239a0`

`__fastcall(ecx=ctx, EDX=event-type tag, payload, *quit_out)`; 15 cases (tags `0…0xe`) via jump table
`0x423b10`. Each case reads fields off the OS event record and calls a per-type handler that
**translates the OS event into one event-bus category** (the loop's input intake). As with the
dispatch primitive, the emitted category is in EDX and is read from the disassembly:

| Tag | Handler | OS event → emitted category |
|---|---|---|
| 0 | `0x423b70` | focus-loss → replays held buttons as button-up (cat `0xe`) |
| 1 | `0x423bb0` | activate value → cat **1** |
| 2 | `0x423be0` | WM_CHAR run (per-UTF16) → cat **1** |
| 3 | `0x423c30` | locale/codepage (reads `GetACP`) → cat **0x1b** |
| 4 | `0x423c70` | locale companion → cat **0x1c** |
| 5 | `0x423b50` | close request → quit-query callback; if allowed sets `*quit_out` |
| 6 | `0x423cb0` | WM_ACTIVATE → cat **2** (+ capture release) |
| 7 | `0x423d00` | key down → cat **8** (sets modifier bit) |
| 8 | `0x423d50` | key up → cat **9** (clears modifier bit) |
| 9 | `0x423db0` | mouse button down → cat **0xb** (sets button mask) |
| 0xa | `0x423fe0` | mouse move → cat **0xc** |
| 0xb | `0x424130` | mouse position/hover → cat **0x10** |
| 0xc | `0x424050` | mouse wheel (float deltas) → cat **0xd** |
| 0xd | `0x4240b0` | mouse button up → cat **0xe** (clears button mask) |
| 0xe | `0x423b50` | WM_QUIT (GetMessageA→0) → quit-query; if allowed sets `*quit_out` |

`0x424200` (cat **0xf**, window/mode-change) is emitted outside this switch. The handlers maintain
shared input-state globals: `DAT_00883f30` active/mode, `DAT_00883f34` key-modifier mask,
`DAT_00883f40` mouse-button mask, `DAT_00883f44` mouse-capture mask, `DAT_00883f5c`/`+0x60` the
installed quit-query callback. Mouse handlers fold the current cursor position (`0x423e20`) and active
bit (`0x423fc0`) into their payloads.

## Cursor transform — `0x423e20`

This shapes the input feel, so its exact math matters. It clamps the raw point to the client rect,
warps the OS cursor to the clamped point (`0x42ce50`), and emits **normalized** coordinates:

```
u = x / width
v = 1.0 − y / height        // Y-flipped to a bottom-left origin
```

read off the cached client rect (`0x42cea0`). The normalize is x87 `fildl/fidivl/fstps` at PC_53 —
an f64 divide rounded to f32 (the same double-rounding as `dt`). The Win32 cursor calls
(`SetCursorPos`, `ClientToScreen`, `GetClientRect`) and the OS-cursor warp surround this math.

**Capture state machine**: button-down latches `DAT_00883f40`; `0x4241a0` sets the capture owner
(`DAT_00883f44`) and toggles cursor-confine mode (`0x42cc90` → center `0x42ccd0` / restore `0x42cd20`);
button-up auto-releases capture once the held buttons clear (`0x423ce0` → `0x420540`).

## Win32 message intake — `0x42c9f0` via `0x423920`

The message pump is **not** in WinMain; it is reached from the loop's input-drain step `0x423920`,
which loops `0x42c9f0` until the queue is empty, dispatching each event through the window-event
switch `0x4239a0`. `0x42c9f0` contains `PeekMessageA`/`GetMessageA`/`TranslateMessage`/
`DispatchMessageA` plus a `WM_SIZE`/iconify window-resize path (`IsIconic`/`GetWindowRect`/
`AdjustWindowRectEx`). The game window's HWND is read via `FUN_00435c30`. The WndProc is `0x42cfe0`
(mouse/key/char/button/wheel/activate/syscommand → `DefWindowProcA`), installed by the
[graphics device](graphics-device.md) init layer, not by WinMain.

## Thread model and lifecycle

The loop runs on a dedicated **engine thread**. Each OS thread that runs the engine gets a per-thread
record (RTTI `.?AUEvtThread@@`, `0x189c` bytes) held under a TLS index (`DAT_00884ef0`) and
doubly-linked into a global list (`DAT_00884ef8`, guarded by critical section `DAT_00884ecc`, live
count `DAT_00884f00`):

- `0x437060` **register** the current thread's record (allocate, `DuplicateHandle`, name `"%s [tid]"`,
  TLS-store, list-link); `0x4371e0` **teardown** (clear TLS, `CloseHandle`, unlink/free, `TlsFree`
  when the last leaves).
- Per tick the loop **binds** the current work-item to the thread record (`0x4374d0`:
  `item+0x08 = record`, `record+0x14 = item`) and **unbinds** at frame-end (`0x437560`).
- The loop **re-asserts the engine-thread priority every tick**: `if (GetThreadPriority() !=
  DAT_00883ec0) SetThreadPriority(DAT_00883ec0)` (`0x64bbf0`/`0x64bc00`). `DAT_00883ec0` is that
  target priority — a thread-priority value, not a thread id.
- `0x422030` wakes every scheduler slot (`SetEvent` on each `DAT_00883f18[]` slot's event) when work
  is posted.

## What plugs in

This is the root of the tree. The single seam every subsystem attaches at is the **event bus**: a
subsystem registers a handler on a Context's event-category list via `0x41fca0` (`category`, `fn`,
`arg`, `prio`), and is invoked by `0x4245b0` when the loop or an event producer emits that category —
the dt tick (category 5) carries `&dt` (seconds). The OS-input seam is the window-event switch
`0x4239a0` fed by the pump `0x42c9f0`. A static call graph stops at these indirections, which is what
makes them the subsystem boundaries: the set of handlers a subsystem registers (and which categories
it consumes) lives in that subsystem. The init path `0x402350` and the registered handlers reach the
separate [graphics device](graphics-device.md) + window creation, the [MPQ](mpq.md) asset VFS, the
[DBC](dbc.md) tables, and the [network](net.md) session.
