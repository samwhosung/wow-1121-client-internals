# UI engine — FrameScript/Lua binding and the widget/text engine

The 1.12.1 interface is a tree of C++ widget objects (frames, buttons, edit boxes, font strings, …)
driven by Lua. The default UI — and every addon — is Lua and XML *data* loaded from MPQs at runtime;
the client supplies the engine that parses that XML into frame objects, binds each frame to a Lua
table, resolves frame geometry from anchors, dispatches input and game events into Lua handlers, and
measures/draws text. This chapter covers that engine: the frame object model, anchor→rect layout, the
Lua↔C++ binding (FrameScript), the FrameXML loader, event/handler firing, and the FontString text
path.

The embedded scripting VM is **unmodified public Lua 5.0**: the banner string at `0x811b30` reads
`"Lua: Lua 5.0 Copyright..."`, `lua_gettop` at `0x6f3070` is the canonical `(top−base)>>4`, and the
standard `LUA_REGISTRYINDEX`/`LUA_GLOBALSINDEX` constants and `LUA_MAXCCALLS=200` are present. The only
client modification is an `lmemPool.cpp` allocator hook (supported by stock Lua). An implementer can
drop in a vetted Lua 5.0; everything in this chapter is the *binding* layered on top of the `lua_*` C
API.

## Memory map

The engine occupies several address bands:

| Range | Contents |
|---|---|
| `0x47f000–0x508800` | `Ui*.cpp` frame classes (WorldFrame … CGGameUI at `0x489`–`496` … Quest/ActionBar/…) |
| `0x50d000–0x537000` | `UIUtil*.cpp` (Camera/InputControl/ScriptEvents/AddOns/Cursor/Tooltip) |
| `0x6edc00–0x6f3000` | FrameXML loader + frame factory (FrameXML.cpp / XMLTree / Create_SimpleFrame) |
| `0x6f3000–0x701000` | Lua 5.0 VM (delegated, stock) |
| `0x701000–0x706400` | FrameScript binding glue |
| `0x760000–0x7a4000` | CSimpleFrame widget framework + FontString text engine |

## Initialization

The UI is built at **world-enter**, not at services boot. (`0x402350` is the services-boot top —
device/VFS/DBC; it does not create the UI.) The chain is:

```
0x46c236 → 0x401570 (Client.cpp enter-world setup)
         → 0x48f480 (CGGameUI::InitializeGame)
         → 0x48fbf0 (CGGameUI::Initialize)
```

`Initialize` builds, in order:

- the **CSimpleTop root frame** — `DAT_00b4e240 = 0x764180()` plus four root callbacks at
  `+0x1128..+0x1134`. The screen root singleton is `DAT_00cf0bd8` (the default anchor parent).
- the **FrameScript context** — `0x703b80`, then `0x490250` registering ~216 C bindings, then
  `0x703d90`.
- **RegisterFrameFactories** `0x495940` (8 factories).
- the **FrameXML.toc / Bindings.xml** data load.
- the **event-bus registration** below.

### The per-frame and input dispatch

The UI's main event handler is `0x495590`, registered on event-bus **category 5**. Inside
`Initialize`: `mov edx,0x495590 / mov ecx,5 / call 0x41fc90`; `0x41fc90` is a fastcall shim that
forwards `ecx`(=category 5)/`edx`(=handler) to the core bus-register primitive `0x41fca0` (a
29-category bound). The frame loop `0x420c00` → dispatch `0x4245b0` drives it each tick. `0x495590`
also services the deferred ReloadUI (it is the second caller of `Initialize`).

Input flows: OS input → `0x420c00` → `0x4245b0` → InputControl.cpp → ScriptEvents.cpp →
**`FrameScript_SignalEvent 0x703e50`** → Lua.

## The frame object model

Every region is built from **multiple inheritance** of two bases:

- **CScriptObject** — the Lua/type base; primary vtable at `this+0`. The type-guard `IsObjectType`
  lives at **vtable+0x10** and is called by every Lua method wrapper.
- **CLayoutFrame** — the anchorable geometry base; secondary vtable at `this+0x24`.

The inheritance chain:

```
CScriptObject + CLayoutFrame
  → CScriptRegion          (ctor 0x76c4f0)
  → CSimpleFrame           (ctor 0x769090, 0x314 B — the interactive base)
  → CSimpleButton / CSimpleModel / CSimpleEditBox / CSimpleStatusBar / CSimpleCheckbox /
    CSimpleMessageFrame / CSimpleScrollFrame / CSimpleSlider / CSimpleHTML /
    CSimpleColorSelect / CSimpleMovieFrame
```

**CSimpleTop** (`0x764180`) is *not* a CSimpleFrame — it single-inherits CLayoutFrame at `+0` and is
the singleton screen root. Texture and FontString are leaf CScriptRegions.

### Struct layout (CSimpleFrame-absolute)

The geometry subobject sits at `+0x24`. Core fields:

| Offset | Field |
|---|---|
| `+0x00` | CScriptObject vtable |
| `+0x24` | CLayoutFrame (geometry) vtable |
| `+0x28` | `anchorPoints[9]` — `CAnchor*[]` indexed by point-id |
| `+0x60` | flags (rect-cache-valid bit0 / dirty bit2 / unresolvable bit3) |
| `+0x64` | `cachedRect` — f32×4, ordered **(BOTTOM, LEFT, TOP, RIGHT)** |
| `+0x74` | width (device px; 0 = derive from anchors) |
| `+0x78` | height (device px; 0 = derive from anchors) |
| `+0x7c` | layoutScale |
| `+0x9c` | parent |
| `+0xa0` | screenRoot |
| `+0xb8` / `+0xbc` | userScale |
| `+0xc0` | frameStrata (default 3 = MEDIUM) |
| `+0xc4` | frameLevel |
| `+0xc8` | alpha (u8, 0–255) |
| `+0xd0` | shown |
| `+0xd4` | effectiveVisible |

A `CAnchor` is **0x14 bytes**: `{vtable, xOff@+4, yOff@+8, relativeTo@+0xc, relativePoint@+0x10}`
with vtable `&0x81c44c`.

## Layout resolution — anchors → screen rect

The screen rect is computed on read with a dirty-flag cache. Mutators (SetPoint / SetWidth /
SetHeight / SetScale) clear the cache-valid bit and call the invalidate/resolve dispatcher
**`0x7680e0`**; readers test dirty (`0x768310` = `flags & 4`) then resolve and read the cached rect
(`0x768320`).

### Anchor math

Each of the 9 `anchorPoints` slots is indexed by its own point-id (0=TOPLEFT … 8=BOTTOMRIGHT) and
holds a `CAnchor {xOff, yOff, relativeTo, relativePoint}`. A point's coordinate on a rect is
selected by the point-id's column/row:

```
X ∈ { left, (left+right)/2, right }      (column: 0=left, 1=center, 2=right)
Y ∈ { top,  (top+bottom)/2,  bottom }    (row:    0=top,  1=center, 2=bottom)
```

**CAnchor::ResolveX / ResolveY** (`0x7a2f90` / `0x7a3070`):

```
coord = ownOffset * layoutScale + relativeTo.<edge>
```

If `relativeTo` is itself dirty it is force-resolved first (pull dependency). Each of the 4 edges
resolves by scanning its 3 point-ids for an anchor; if the edge is unanchored it is derived from the
opposite edge/center plus the signed width/height·scale, through **combineEdge** (`0x767440`) /
**combineCenter** (`0x7672d0`).

- Two opposing anchors pin all edges directly ⇒ width/height are ignored (size = the anchor span).
- A single anchor + explicit W/H sizes the opposite edge.

Sentinels: an **unset edge = +Infinity (`0x7f800000`)**; no-explicit-size = `0.0`. The whole chain
works in **screen pixels** (it bottoms out at the CSimpleTop screen root); `layoutScale` scales only
the offsets and the W/H span.

### Cache, transform, OnSizeChanged

The rect is cached at `+0x64` in **(BOTTOM, LEFT, TOP, RIGHT)** order, lazy-resolved on read behind a
dirty/queued flag. Mutators enqueue into the per-frame deferred-layout queue at **`0xcf0bec`**
(dependency-ordered); readers force-resolve.

- **`0x767a20`** assembles the rect (with optional clamp-to-screen).
- **`0x768d20`** recomputes, ε-compares (≈`1e-5`), caches, and calls **`0x76b580` (ApplyRect)** =
  CLayoutFrame geometry vtable slot 0. ApplyRect applies per-edge insets×scale to produce the screen
  rect and fires **OnSizeChanged** (frame vtable `+0x48`) when `|Δw|` or `|Δh| ≥ 2.384e-7`.
- The inner resolver is `0x7681b0`.

The vtables that the resolver dispatches through carry these slots:

- geometry vtable **`0x81c400`**: `+0x1c` = GetWidth `0x768420`, `+0x20` = GetHeight `0x768410`,
  `+0x24` = normalize predicate `0x46ff60` (`xor eax,eax; ret` ⇒ 0).
- CAnchor vtable **`0x81c44c`**: `+0x04` = ResolveX `0x7a2f90`, `+0x08` = ResolveY `0x7a3070`,
  `+0x0c` = the `relativeTo` target getter (used by SetPoint change-detection).

### Rect getters/setters

- **GetRect `0x768320`**`(out[4])`: returns 0 if `flags & 1` (rect-cache-valid) is clear — it does
  *not* resolve; the caller force-resolves first. Otherwise copies `cachedRect` `+0x64..+0x70` and
  returns 1.
- **GetWidth `0x768420`** = `+0x74`; **GetHeight `0x768410`** = `+0x78` (raw device size; 0 = derive).
- **SetWidth `0x7683d0`**`(w)` / **SetHeight `0x7683f0`**`(h)`: store the device-space value at
  `+0x74`/`+0x78`, clear the unresolvable marker (`flags &= ~8`), and call the invalidate dispatcher
  `0x7680e0(0)`.

### SetPoint

**SetPoint `0x767c70`**`(point_id, relativeTo, relativePoint, xOff, yOff, doResolve)`:

- rejects `relativeTo` null-or-self (error `0x57`).
- change-detects against any existing `anchorPoints[point_id]` (slot `+0x28 + id*4`): same target
  (via CAnchor vtable `+0xc`), same `relativePoint`, and both offsets within `eps = 2^-22 =
  2.384e-7` ⇒ no-op.
- else detaches the old anchor (`0x767fa0`) + frees it, allocates a **CAnchor** (0x14 B: vtable
  `&0x81c44c`, `xOff@+4`, `yOff@+8`, `relativeTo@+0xc`, `relativePoint@+0x10`), stores it at the
  slot, clears `flags &= ~8`, registers the dependency (`0x767ee0`), and (if `doResolve`) invalidates
  (`0x7680e0(0)`).

SetPoint only commits the anchor; the device rect is recomputed lazily by the resolver chain
(`0x7680e0` → `0x768d20` → `0x76b580`). **SetAllPoints `0x767db0`** is the 4-edge analog.

## The FrameScript binding (Lua ↔ C++)

The binding glue lives in `0x701000–0x706400`, layered on the stock `lua_*` C API. (Within this band
the Lua bytecode compiler `0x701010–0x701910` and the Lua base/os/debug/string library
`0x702860–0x703760` are stock Lua, interleaved here; the rest is the binding.)

### Function registration

C bindings come from an 8-byte `{const char* name; lua_CFunction fn}` table at **base `0x83de68`**
(216 core entries). The driver `0x490250` calls **RegisterFunction `0x704120`**, which does
`_G[name] = lua_pushcclosure(fn)`. Per-widget-type registrars (reached via the master `0x706350`)
register the ~1142 widget methods.

### The marshalling shape

Every accessor follows the same template:

```c
int __fastcall f(lua_State* L /* = ecx */) {
    // type-guard self via CScriptObject vtable+0x10  IsObjectType
    // read args with lua_to*
    // call the C++ method / read-write the frame-model field
    // push results (object results via lua_rawgeti(REGISTRYINDEX, obj[+8]))
    return /* #results */;
}
```

Object results are pushed by `lua_rawgeti(REGISTRYINDEX, obj[+8])`; the registry ref is lazily bound
by `0x701bd0` when the wrapper refcount `obj[+4] == 0`.

### Events

- **GetContext `0x7040d0`** = `*0xceef74`.
- **SignalEvent `0x703e50`** walks the per-event listener list (registry stride `0x10` at `0xceef68`)
  → `0x704d50` (push handler + self, pcall).
- **SignalEvent2 `0x703f50`** marshals `%d/%f/%s/%u` varargs.

## The Lua frame object model and method dispatch

A frame's Lua value is a **table `T`** with:

- `T[0] = lightuserdata(CScriptObject*)`, and
- the SHARED metatable `_G["__framescript_meta"]`.

The wrapper is built lazily by **bind `0x701bd0`**:

```
newtable
T[0] = lightuserdata(this)
setmetatable(T, _G["__framescript_meta"])
this[+8] = luaL_ref(REGISTRYINDEX)     // pushed back via lua_rawgeti(REGISTRYINDEX, this+8)
                                        // wrapper refcount = this+4
```

If the frame is **named** and `_G[name]` is nil, `0x701bd0` also sets `_G[name] = T` (published as a
global, non-overwriting).

**Dispatch of `frame:Method(...)`.** The shared metatable's `__index` is the C dispatcher
**`0x7020b0`** (built once at FrameScript init: `MT["__index"] = lua_pushcclosure(0x7020b0);
_G["__framescript_meta"] = MT`). It requires arg1 a table + arg2 a string, takes `self =
touserdata(T[0])`, and calls the virtual **`self->vtable[+8](L, name)`** — the per-type method lookup,
which pushes the method's C closure if the object's runtime type (or a base) defines `name`, else nil.

Each method closure (template: CSimpleModel `SetPosition 0x76dc00`):

1. requires arg1 a table — else *"Attempt to find 'this' in non-table object (used '.' instead of
   ':' ?)"*.
2. recovers `self = touserdata(lua_rawgeti(arg1, 0))` — else *"…non-framescript object"*.
3. type-guards `self->vtable[+0x10] IsObjectType(typeToken)` — else *"Wrong object type for member
   function"*.
4. operates, pushing object results via `lua_rawgeti(REGISTRYINDEX, result[+8])`.

Per-type methods are registered by **`0x701d80`**`(methodTable, count, &hashtable)` from a
`{name, lua_CFunction}[]` table (one registrar per widget type under the master `0x706350`; e.g.
CSimpleModel's 23 methods at `0x878948`) into a per-type C++ name→closure hashtable.

**Global accessors.** `getglobal`/`setglobal` (FrameScript base `0x7027b0`/`0x702770`) are `_G[name]`
get/set via `lua_gettable`/`lua_settable` on `LUA_GLOBALSINDEX (-10001)`. So FrameXML's global
`FrameName` and `getglobal("FrameName")` both resolve to the named frame's wrapper.

## Handler firing — `this` / `event` / `arg1..argN`

A `<Scripts>` handler (compiled by `loadstring`) is fired with **`lua_pcall` nargs = 0** — it gets no
Lua arguments and reads its inputs from **globals** that the dispatcher sets and restores around the
call:

| Global | Address | Meaning |
|---|---|---|
| `this` | `0x872e64` | the firing frame's wrapper table |
| `event` | `0x84b648` | the event name |
| `arg1`..`argN` | `0x8722dc` | the args (up to 18) |

Each global is **set-then-restored**: `luaL_ref` saves the prior value, then `lua_settable` /
`luaL_unref` restore it after the pcall — so firing is nesting-safe.

- **SignalEvent `0x703e50`**(eventId) sets `event`=name around the listener walk. Each listener's
  handler lives at `frame+0xc` (the CScriptObject `OnEvent` slot, `0x702590`) → `0x702690` →
  **`0x704d50`** (sets `this`, pcall(0), no args). `0x702690` also swaps the Lua globals-table to the
  listener frame's env.
- **SignalEvent2 `0x703f50`**(eventId, fmt, …) routes through **`0x704f10`**, which marshals the
  `%d/%f/%s/%u` args into `arg1..argN` globals (then `this`, pcall(0), restore). `0x704f10` is the
  generic "fire with format args" runner.

By kind:

| Script | Globals set |
|---|---|
| OnLoad | `this` |
| OnEvent | `event` + `this` (+ `arg1..argN` if fired via SignalEvent2) |
| OnUpdate | `this` + `arg1` = elapsed (`%f`) |
| OnClick / input | `this` + `arg1` = button/key |

All fire with pcall nargs = 0. An implementer's `fire_script` must set `this`/`event`/`arg*` as
globals (not pass them as pcall args), pcall with nargs 0, and save+restore each around the call.

The C→Lua event spine (eventId→name) comes from the `.data` init table at `0xbe11d8`; there are 251
`SignalEvent` call sites in the binary.

## The FrameXML XML loader — schema → frame ops

The loader region `0x6edc00–0x6f3000` turns a parsed FrameXML node tree into frame operations. The
*XML parse* is delegated; the **schema→op map** is the engine's. Attributes are read via
`GetAttribute 0x6f2cf0`; element-name compares are case-insensitive (`0x64a4c0`). Node struct:
`name@+0x8`, `children@+0x4`, `next@+0x1c`, `body@+0xc`.

### Top-level file loader `0x6ede10` (`<Ui>`)

- `<Include file=>` → recurse-load.
- `<Script file=>` / inline `<Script>` → run Lua (`0x704bc0` / `0x704cd0`) **in document order** —
  this is how XML-referenced FrameXML Lua loads.
- `<Font>` → font definition.
- a frame element with **`virtual="true"`** → register a template (`0x6ee500`, by `name`); else
  instantiate (`0x6ee280`).

### Instantiate `0x6ee280` (CreateFrame-from-node)

Signature: `(node, defaultParent, status)`. The element tag selects a frame-type factory from the
global hashtable `DAT_00cee9d8` (registered by `0x6ef070` → `RegisterFrameType 0x6ee880`); resolves
`parent`/`name`; creates via the factory; splices any `inherits=`/template nodes (merge-before-
interpret); then runs the load chain `vtable+0x20` · CSimpleFrame::LoadXML `0x769820` · `vtable+0x24`.

The 13 frame-element types and their factory addresses:

| Element | Address | | Element | Address |
|---|---|---|---|---|
| Button | `0x6eeab0` | | Slider | `0x6eee40` |
| CheckButton | `0x6eeb30` | | SimpleHTML | `0x6eeeb0` |
| EditBox | `0x6eeba0` | | StatusBar | `0x6eef20` |
| Frame | `0x6eec10` (→ CSimpleFrame ctor `0x769090`, 0x314 B) | | ColorSelect | `0x6eef90` |
| MessageFrame | `0x6eec80` | | MovieFrame | `0x6ef000` |
| Model | `0x6eecf0` | | | |
| ScrollFrame | `0x6eed60` | | | |
| ScrollingMessageFrame | `0x6eedd0` | | | |

### CSimpleFrame::LoadXML `0x769820`

Attributes: `inherits` (template merge) → super **CLayoutFrame::LoadXML `0x767800`** →
`hidden`(Show/Hide) · `toplevel`/`movable`/`resizable` (flags `+0xb4` |0x1/0x100/0x200) ·
`frameStrata` (`0x76a470`, `+0xc0`) · `frameLevel` (`0x76a4f0`, `+0xc4`) · `alpha` (×255 → `+0xc8`) ·
`enableMouse`/`enableKeyboard`/`clampedToScreen`.

Child elements: `<TitleRegion>` · `<ResizeBounds>` · `<Backdrop>` (`0x76a5d0`) · `<HitRectInsets>`
(`0x76b1b0`) · `<Layers>` (`0x769d70`) · `<Scripts>` (`0x769ef0`).

### CLayoutFrame::LoadXML `0x767800` (geometry)

- `<Size>` AbsDimension → SetWidth/SetHeight (`vtable+0x14`/`+0x18`).
- `<Anchors><Anchor point relativeTo relativePoint><Offset>` → **SetPoint `0x767c70`**
  (`relativePoint` defaults to `point`, `relativeTo` to the layout parent).
- `setAllPoints="true"` → SetAllPoints.
- then resolve `0x7680e0`.

**Point-id enum** (`0x811a38`): TOPLEFT=0, TOP=1, TOPRIGHT=2, LEFT=3, CENTER=4, RIGHT=5,
BOTTOMLEFT=6, BOTTOM=7, BOTTOMRIGHT=8.

### Layers and Scripts

- **Layers `0x769d70`**: `<Layer level=>` (BACKGROUND/BORDER/ARTWORK/OVERLAY/HIGHLIGHT, default
  ARTWORK) → `<Texture>` (`0x6f26f0`) / `<FontString>` (`0x6f2780`) region, added to the render list
  (`0x77fd10`).
- **Scripts `0x769ef0` → SetScript `0x7025c0`**: each `<Scripts>` child (OnLoad/OnEvent/OnUpdate/
  OnShow/OnHide/OnClick/OnEnter/OnLeave/OnMouse*/OnKey*/OnChar/OnDrag*) → compile its body with
  `loadstring`, store the ref in the frame's script slot. The base name→offset map `0x76a0d0`:
  OnLoad `+0x118` · OnUpdate `+0x128` · OnShow `+0x130` · OnHide `+0x138` · OnEnter `+0x140` · … ·
  OnKeyUp `+0x190` (plus OnEvent `+0xc`). An input handler auto-enables its input.

### Dimensions

**AbsDimension `0x6f1eb0`**: `<AbsDimension x= y=>` → device px = `attr / (uiScale·K)`;
`<RelDimension>` → raw fractions.

### `$parent` name substitution

A frame/region `name` beginning (case-insensitively) with **`$parent`** is rewritten before the
`_G[name]` publish. The rule lives in **SetName `0x76c650` → `0x76c5b0`** (reached via the node-apply
`0x769770`), and runs *before* the wrapper-bind `0x701bd0`:

- if `name` starts (CI) with the literal `"$parent"` (`0x8788b0` — the only such token; no
  `$parentKey`/`$parentN` exist), replace the 7-char prefix with the nearest ancestor up the parent
  chain (`+0x9c`) whose `GetName` (vtable+4) is non-empty, else the literal default `"Top"`
  (`0x8788ac`).
- append the post-`$parent` suffix verbatim (`SStrCat`).

Applies only to `name` (`0x838090`). Example: parent `PlayerFrame` + `"$parentHealthBar"` →
`PlayerFrameHealthBar`; a top-level `$parentFoo` → `TopFoo`. It uses the parent's already-resolved
name (stored at `+0x98`) so chains compose.

### Nested `<Frames>`

`<Frames>` is *not* handled in `0x769820`'s child loop; it is a separate pass **LoadChildFrames
`0x76a060`**, driven from the **post-load finalizer `0x76a2f0`** (the `0x6ee280` chain's `vtable+0x24`
step). So child frames are created *after* the parent's own Size/Anchors/Layers/Scripts.

`0x76a060` resolves `inherits` (recursing into the template's `<Frames>` first), then for each
frame-element child calls **`0x6ee280(child, edx = this /* default parent */, status)`**. A child's
parent = its `parent=` attr (resolved by name via `0x76c760` = `_G[name]`) else the enclosing frame.
Names/templates stay file-global (no scoping). The finalizer then fires the parent's **OnLoad**
(`+0x118`) and **OnShow** (`+0x130`). Because each child is fully loaded (including its own OnLoad)
inside step (1), OnLoad is **bottom-up** — a child's OnLoad fires before its parent's.

### Two referenced globals worth noting

- `GetItemQualityColor` is a **C-registered** binding (`Script::GetItemQualityColor 0x48dfb0`).
- `TEXT` is **Lua-defined** via `<Script file=>` includes (a load-order matter, not a C
  registration).

## Per-type widget LoadXML

Each of the 13 frame types has its own `LoadXML` (run after the CSimpleFrame base `0x769820`) reading
the type's scripts, sub-elements, and attributes. The load-bearing mechanism:

**Scripts are resolved, not matched.** SetScript (`0x7025c0`) finds a script's slot via the type's
script-name→slot resolver (a vtable method), which **chains the base map** (`0x76a0d0`, above) then
adds the type's own `OnX → +0xNNN`. So a type's script set = base + per-type additions:

| Type | Resolver | Added scripts (slot) |
|---|---|---|
| Button | `0x778c50` | OnClick `+0x4cc`, OnDoubleClick `+0x4d4` |
| Slider | `0x789750` | OnValueChanged `+0x330` |
| StatusBar | `0x7830d0` | OnValueChanged `+0x32c` |
| EditBox | `0x77a310` | OnEnterPressed/OnEscapePressed/OnTabPressed/OnTextChanged/OnEditFocusGained/Lost/… (`+0x3f8…+0x440`) |
| ScrollFrame | `0x786c40` | OnHorizontalScroll `+0x32c`, OnVerticalScroll `+0x334`, OnScrollRangeChanged `+0x33c` |
| Model | — | OnUpdateModel, OnAnimFinished |
| SimpleHTML / ScrollingMessageFrame | — | OnHyperlinkClick/Enter/Leave |
| ColorSelect | — | OnColorSelect |
| MovieFrame | — | OnMovieFinished/ShowSubtitle/HideSubtitle |
| CheckButton / MessageFrame | — | (none) |

**Shared enum tables** (each a `{u32 value, char* name}` array):

| Table | Address | Values |
|---|---|---|
| alphaMode | `0x811aa8` | DISABLE=0, ALPHAKEY=1, BLEND=2, ADD=3, MOD=4 |
| justify | `0x811ad0` | justifyH (`&7`): LEFT=1, CENTER=2, RIGHT=4 · justifyV (`&0x38`): TOP=8, MIDDLE=0x10, BOTTOM=0x20 |
| orientation | `0x811b00` | HORIZONTAL=0, VERTICAL=1 |
| drawLayer | `0x811a84` | BACKGROUND=0 … HIGHLIGHT=4 |

**Regions.** `<Texture>` (`0x76fe20`) and `<FontString>` (`0x770f40`) super-call the geometry base
`0x767800`, then read their own attributes — Texture: `file`/`alphaMode`/`<TexCoords>`/`<Color>`/
`<Gradient>`; FontString: `text`/`font`/`justifyH`/`justifyV`/`bytes`/`maxLines`/`spacing`/`<Color>`/
`<Shadow>`. The shared `<Font>` object loader is `0x783c30`.

## The FontString text engine

A FontString frame owns only the **text state and cache**; the actual glyph measurement and
line-layout are done by a lower-level kernel that the frame forwards to. The width query chain:

```
GetStringWidth 0x79e510
  → 0x772890   (UI-level cache + scale)
  → 0x44d670   (CGxString shim)
  → 0x5c2050   (5-arg forwarder)
  → 0x5c6940   (the measurement kernel)
```

`0x5c6940` walks the byte string, parses newline and control codes, and accumulates a float advance
via per-glyph helpers. Text measurement, line-breaking, wrap, and justify-positioning all live in
this kernel (`0x5c6940` et al.), *not* in the FontString frame — it is a separate text-layout
subsystem (the engine that owns it is outside the UI region; the addresses do not belong to the
graphics device).

What the FontString frame itself owns:

- the **state** — text/font/color/justify/spacing/shadow fields.
- **lazy-cache bookkeeping** — dirty flags and cached width/height at `+0xfc` / `+0x100` (the
  rect-cache idiom).
- trivial px↔coord scaling.
- the `|c…|r` color-escape scan.
- a UTF-8 ellipsis truncate loop (each candidate width still comes from the kernel).

Justify is **6 stored bits** the frame stores and hands off as a flag — consumed by the text-layout
kernel above (which owns justify-positioning), **not** by the graphics device that draws the glyphs.
The accessor methods are the same binding glue as every other widget, over the FONTSTRING /
FONTINSTANCE structs.

## The item quality colour table

`GetItemQualityColor(quality)` (the binding `Script::GetItemQualityColor 0x48dfb0`) returns
`(R/255, G/255, B/255, escapeString)`.

`0x48dfb0` reads the matched entry from the BGRA dword array at `0xc0d3c8` (accessor `0x52ad70`:
`[quality*4 + 0xc0d3c8]`, with **quality ≥ 7 clamped to index 1**) — bytes `[+2]=R, [+1]=G, [+0]=B`,
each ×`1/255` — and the colour-escape string from the parallel `char*` array at `0x854124` (accessor
`0x52ad90`). Both arrays are filled by the static init `0x5291d0` from these 7 ARGB literals
(`0xAARRGGBB`):

| q | name | ARGB | R,G,B | escape string |
|---|------|------|-------|---------------|
| 0 | Poor | `0xff9d9d9d` | 9d,9d,9d | `\|cff9d9d9d` |
| 1 | Common | `0xffffffff` | ff,ff,ff | `\|cffffffff` |
| 2 | Uncommon | `0xff1eff00` | 1e,ff,00 | `\|cff1eff00` |
| 3 | Rare | `0xff0070dd` | 00,70,dd | `\|cff0070dd` |
| 4 | Epic | `0xffa335ee` | a3,35,ee | `\|cffa335ee` |
| 5 | Legendary | `0xffff8000` | ff,80,00 | `\|cffff8000` |
| 6 | Artifact | `0xffe6cc80` | e6,cc,80 | `\|cffe6cc80` |

## Movement keybind command handlers

The movement keybinds (e.g. `MoveForwardStart 0x513e20` / `MoveForwardStop 0x513e50`, …; registration
table `0x8500b8`) set/clear bits on the **input/local-mover object** `[0xbe1148]+4` via the setter
`0x515090(X, set/clear, time, 0)`, doing `[MOVE+4] |= X` / `&= ~X` (`0x514840` / `0x514b70`). The
input bits:

| Bit | Action |
|---|---|
| `0x10` | Forward |
| `0x20` | Backward |
| `0x40` | StrafeLeft |
| `0x80` | StrafeRight |
| `0x100` | TurnLeft |
| `0x200` | TurnRight |
| `0x400` | PitchUp |
| `0x800` | PitchDown |
| `0x1000` | Autorun (toggle) |
| `0x1` | mouse-turn |
| `0x2` | mouse-move |

The emitter `0x514640` translates these input bits to the wire `CMovement` MOVEMENTFLAGS (`+0x40`)
that the physics step `0x7c5360` reads, by **`wireBit = inputBit >> 4`**: Forward→`0x1`,
Backward→`0x2`, StrafeL→`0x4`, StrafeR→`0x8`, TurnL→`0x10`, TurnR→`0x20`, PitchU→`0x40`,
PitchD→`0x80`. Jump is the exception: `0x200800` via `0x60dea0` (one-shot). Specials: autorun folds
into the forward axis; held mouselook converts the turn keys to strafe at the `+0x40` level;
Jump/ToggleRun bypass the input word entirely (physics-owned).

## Cross-references

- Frame anchoring works in the same screen-pixel space the [Camera](camera.md) and
  [Rendering math](rendering-math.md) chapters use.
- Frames bind to game state through the [Object model](object-model.md) UpdateFields, driven by the
  [Net](net.md) state bridge.
- Text drawing and texture upload go through the [Graphics device](graphics-device.md); the BLP
  textures FontStrings and Texture regions reference come through [Image](image.md).
- Lua and XML UI data are loaded from [MPQ](mpq.md) archives.
