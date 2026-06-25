# Graphics device — the render device and state submission

The client's rendering device is `CGxDevice` and its OpenGL implementation `CGxDeviceOpenGl`
(Blizzard's `ENGINE\Source\gx\CGxDevice\…`). It is the foundation everything visual plugs into: the
glue screens (login / char-select), the 3D world, models, and the UI all draw through the `Gx*` device
API. WoW 1.12.1 is a **primarily fixed-function GL 1.x** renderer. The shipped binary is a dual-backend
build (`CGxDeviceOpenGl` + `CGxDeviceD3d`) selected at startup by the `gxApi` CVar; the OpenGL `WoW.exe`
runs the OpenGL backend, and the Direct3D backend is dead code on that build.

## Module layout

The device module is `0x58c000`–`0x5a3000`, fronted by a C-API facade at `0x589xxx`–`0x58bxxx` that
owns the global device singleton at `ds:0xc0ed38`. The live OpenGL backend's GL-issuing methods live in
`0x59c000`–`0x5a0000`. Interior code is grouped (by RTTI: `CGxDevice` / `CGxDeviceOpenGl` /
`CGxStateBom` / `CGxTexCache` / `CGxPool`) into render-state machines, transform/matrix loaders, the
texture cache + GL upload path, and the buffer/pool allocator.

## Device creation and lifecycle

| Address | Role |
|---|---|
| `0x63a230` | App glue device-init layer: reads the `gxApi` CVar, drives create |
| `0x589ac0` | `GxDevCreate` factory — switches on the device-type arg in `ecx` |
| `0x58dd70` | **live** `CGxDeviceOpenGl` constructor (type `0`) |
| `0x58ba70` | dead `CGxDeviceD3d` constructor (type `1`) |
| `0x589a00` | device-init method (`ecx` = device) |
| `0x58d2b0` | window + GL-context init |

`GxDevCreate` (`0x589ac0`) selects the constructor on its device-type argument (`ecx == 0` →
`0x58dd70`, `ecx == 1` → `0x58ba70`) and stores the result in the global device pointer
(`mov ds:0xc0ed38, eax` at `0x589ad3` / `0x589adf`). The live OpenGL constructor `0x58dd70` installs
the OpenGL vtable: `mov [esi], 0x809af8` at `0x58dee7` / `0x58dfeb`.

The init method `0x589a00` runs `0x58b640` / `0x58b6a0` / `0x58b740`, then `0x58d2b0`, which creates the
OS window and GL context: `CreateWindowExA` (window class `FGxWindowClassOpenGl`),
`ChoosePixelFormat` / `SetPixelFormat`, `wglCreateContext` / `wglMakeCurrent`, and the
`wglGetProcAddress` GL function loader. The window's message handler is **not** in this module — the
WndProc is core's `0x42cfe0`; the device creates the window, core owns the message pump. See [Core](core.md).

The live OpenGL device object's vtable is `0x809af8` (62 virtual methods).

## Device identity — the two vtables

The build carries both backends, distinguished by their vtables:

- **`0x809af8` = `CGxDeviceOpenGl` (live)** — its methods issue `gl*` / `wgl*` calls via the opengl32
  import table.
- **`0x809ef8` = `CGxDeviceD3d` (dead)** — its methods dispatch `IDirect3DDevice9` by COM
  (`call [edx+0xNN]`) and contain zero GL (e.g. at `0x5a2f00`).

Because of this split, the functions `0x5a11d0` / `0x5a17d0` / `0x5a1100` / `0x5a2eb0` and the
dispatchers `0x5a2570` / `0x5a2f00` belong to the dead D3d backend; their live OpenGL twins are in
`0x59c000`–`0x5a0000`. An implementer studying the device should follow the OpenGL vtable `0x809af8`
and ignore the D3d subtree.

## Per-frame present

`CGxDeviceOpenGl::ScenePresent` is `0x59ba10`: it flushes through the vtable (`call [eax+0x14]`), then
presents via the import-table `wglSwapLayerBuffers` (`ds:0x7ff46c`) and `glFinish` (`ds:0x7ff468`).
Scene-clear is the neighbouring function around `0x59b9c0`. The 3D scene and glue-screen *content* is
driven by higher-level render code (WorldFrame, CSimpleRender); the device only provides the
draw/state/present primitives those callers invoke.

## Shaders — not purely fixed-function

On ARB-capable hardware the device uses `GL_ARB_fragment_program` programs rather than pure
fixed-function. The fog program is an inline `!!ARBfp1.0` program assembled at `0x59e530` from `.data`
fragments (`OPTION ARB_fog_exp2` / `exp` / `linear` plus a body) and submitted via `glProgramStringARB`
(resolved at `0x59aff0`). There is also an FFX post-process `.bls` program path, and an NV
register-combiner / `*_program` path (`glGenProgramsNV` / `glBindProgramNV`).

The CPU side only assembles the program text and submits it; the per-pixel shading runs GPU-side
(driver-compiled, and therefore GPU/driver-dependent rather than a fixed computed result). The shader
*source* — the inline `.data` ARB fog program and the on-disk `.bls` files — is fixed asset-like data.

## What the device computes vs what it hands to GL

The device itself computes the feel-defining transform, state, and format logic; it delegates raw GL
plumbing to the host:

- **Computed by the device**: the transform / matrix setup including the right-handed → GL **Z-flip**,
  viewport / scissor / depth-range, the render-state machine (`CGxStateBom` push/pop plus the GL
  state-setter dispatcher across `~0x59c000`–`0x59e000`: blend / alpha / depth / fog / material /
  light), the `GxTex_*` texture-format + sampler model, the `GxVertexDecl` vertex/index format, the
  buffer/pool model, and draw-order / flush.
- **Delegated to GL / Win32**: issuing the `gl*` / `wgl*` calls, GL function-pointer loading, the OS
  window, GPU upload/readback, and gamma.

### Render-state machines

There are three redundant-set-eliding render-state machines (`0x59baa0` live, plus `0x5a2570` /
`0x5a2f00` on the dead D3d side). Each does a bounds-check, maps the state id through a byte-map into a
jump table, consults a per-state cache to skip redundant sets, and runs an inline GL-setter arm.

### Transform / matrix loaders

| Address | Loader |
|---|---|
| `0x59c730` | viewport |
| `0x59c820` | light |
| `0x59ca60` | texture matrix |
| `0x59fe20` | projection |
| `0x5a0000` | shared Z-flip |

The 4×4 matrix/vector library these call is shared with the rest of the client and lives at
`0x7bc040`–`0x7bdf50` (the [Math primitives](math-primitives.md) library); from the device it is an
external dependency, not part of the device's own code.

### Texture cache + GL upload

The real `glTexImage2D` uploader lives in the `0x58c`–`0x58f` facade. `0x58da30` is an alignment-round
helper, **not** the uploader — a gotcha when tracing the upload path. BLP texture *decode* is a separate
subsystem; the device consumes already-decoded pixels and owns only the GL upload plus the format/sampler
enums. See [Image](image.md).

### Buffer / pool

`CGxPool` / `CGxBuf` manage vertex/index buffers with Storm-heap bookkeeping.

### Gamma

Gamma is handled outside the device's owned math: `0x58cbb0` (live) / `0x599a60` (dead D3d GDI ramp I/O)
do the ramp read/write, but the curve math is computed elsewhere.

## The transform and colour math

Five small pieces of arithmetic define exact pixel-affecting results. They run with the x87 control word
at `FPCW = 0x027F` (PC_53, i.e. 64-bit double-extended precision rounding to 53-bit mantissa where noted).

### `z_flip` — `0x5a0000`

The right-handed → GL handedness flip. Copies a 4×4 matrix and `fchs`-negates the Z elements (indices
2 / 6 / 10 / 14). Signature is `__stdcall(src, dst)`. This is the `fchs` applied to the Z column before
`glLoadMatrixf`.

### `color_to_byte` — `0x5933c0`

Colour float → byte:

```
byte = __ftol(c · 255 + 0.5)
```

i.e. round-half-up then truncate (`__ftol`).

### `color_norm` — `0x59c100`

Byte → float, multiply by `1/255`:

```
f = (f32)byte · [0x809ffc]      ; fild; fmul [0x809ffc]   ([0x809ffc] = 1/255)
```

### `color_modulate` — `0x593040`

Scalar-modulated packed-colour unpack:

```
out = byte · (scalar · 1/255)
```

where `scalar/255` is kept at PC_53 f64 using the constant `[0x8026c8]`. This is **distinct** from
`color_norm · scalar`: the rounding point differs, so the two produce different results — do not
substitute one for the other.

### `viewport` — `0x59c730`

NDC → pixel. Each edge is computed `scale · (PC_53 f64)` then truncated with `__ftol`, and the integer
pixel extents are formed by **differencing the already-truncated edges**. The independent per-edge
rounding (truncate first, subtract after) is the exact rule — computing width/height before rounding
would diverge.
