---
name: mfl-ffi
description: "MFL C FFI: extern blocks, cstructs, opaque handles, raw memory allocation, pointer/array operations. Call C libraries (raylib, math, etc.) from MFL with zero overhead."
---

# MFL C FFI

Call C functions directly from MFL using `extern` blocks. No CGo, no wrappers — machin compiles the C call directly.

## 1. Extern declaration syntax

```mfl
extern "libname" {
    header "header.h"      // #include <header.h>
    link "lib"             // -llib to linker
    cflags "-I/path"       // extra compiler flags
    fn name(params) ret    // C function declaration
    cstruct Name { fields } // C struct layout
}
```

## 2. Scalar FFI (Phase 1)

Call simple C functions that take and return scalar types:

```mfl
extern "m" {
    header "math.h"
    link "m"
    fn sqrt(float) float
    fn pow(float, float) float
    fn sin(float) float
    fn cos(float) float
}

func main() {
    println(str(sqrt(2.0)))  // 1.4142135623730951
    println(str(pow(2.0, 3.0)))  // 8
}
```

## 2b. OpenSSL EVP via FFI — you MUST declare the header

The crypto builtins (`aes_gcm_*`, `sha256_bytes`, …) compile OpenSSL headers into every binary that uses them. If your `extern` block calls EVP/other OpenSSL functions **without** a `header`, machin emits its own `extern int32_t fn(void*,...)` prototypes that **conflict** with the real prototypes already pulled in by the prelude → `error: conflicting types`. Always write:

```mfl
extern "crypto" {
    header "openssl/evp.h"     // lets machin skip emitting prototypes
    link "crypto"
    fn EVP_CIPHER_CTX_new() ptr
    fn EVP_DecryptUpdate(ptr, ptr, ptr, ptr, i32) i32   // int* outl -> pass alloc(4), read with peek_i32
    ...
}
```

`ptr` params map to `void*` and C converts them to the real pointer types (`EVP_CIPHER_CTX*`, `unsigned char*`, …). Proof-of-concept AES-CTR decrypt loop: [machin-suno/src/crypto.src](https://github.com/javimosch/machin-suno/blob/master/src/crypto.src) — verified against the NIST AES-128-CTR vector.

## 3. FFI type mapping

| MFL | C |
|-----|---|
| `int` / `i64` | `int64_t` |
| `i32` | `int32_t` |
| `i16` | `int16_t` |
| `i8` | `int8_t` |
| `u64` | `uint64_t` |
| `u32` | `uint32_t` |
| `u16` | `uint16_t` |
| `u8` | `uint8_t` |
| `float` / `f64` | `double` |
| `f32` | `float` |
| `bool` | `int` |
| `string` | `const char*` |
| `ptr` | `void*` (opaque handle) |

## 4. By-value struct FFI (Phase 2)

Pass C structs by value. Declare the struct layout with `cstruct`:

```mfl
extern "raylib" {
    header "raylib.h"
    link "raylib"
    link "GL"
    link "m"

    cstruct Color {
        r u8
        g u8
        b u8
        a u8
    }
    cstruct Vector2 { x f32  y f32 }
    cstruct Rectangle { x f32  y f32  width f32  height f32 }

    fn DrawRectangle(int32, int32, int32, int32, Color)
    fn DrawRectangleRec(Rectangle, Color)
    fn DrawTextureRec(Texture2D, Rectangle, Vector2, Color)

    // ... more functions
}
```

**Nested cstructs:**

```mfl
cstruct Vector3 { x f32  y f32  z f32 }
cstruct Camera3D {
    position   Vector3
    target     Vector3
    up         Vector3
    fovy       f32
    projection i32
}

// Construct and pass
cam := Camera3D{
    Vector3{0, 10, 10},
    Vector3{0, 0, 0},
    Vector3{0, 1, 0},
    45.0,
    0,
}
BeginMode3D(cam)
```

**User structs can hold cstructs** (`type Theme struct { bg Color }`) — the
compiler emits the cstruct's `mfl_` typedef before its users automatically
(#650 fix). Caveat: a **headerless** extern block (no `header "..."`) emits raw
C typedefs itself, and a cstruct field whose type is another cstruct currently
emits `void` there — use a real header for nested cstructs.

## 5. Opaque handles (Phase 3)

For C structs that contain pointers (raylib's `Sound`, `Music`, `Font`, `Texture2D`), declare an empty `cstruct`:

```mfl
cstruct Sound {}     // opaque: hold and pass, but don't construct
cstruct Texture2D {} // opaque handle

fn LoadSound(string) -> Sound
fn PlaySound(Sound)
```

MFL holds the whole C struct by value but can't inspect its fields. It can receive one from a C function, store it, and pass it back.

## 6. Raw pointers and memory

```mfl
// Allocate zeroed memory
ptr := alloc(1024)  // returns int (pointer value)

// Poke values into memory
poke_i32(ptr, 0, 42)           // offset 0: int32 = 42
poke_f32(ptr, 4, 3.14)         // offset 4: float = 3.14
poke_u8(ptr, 8, 255)           // offset 8: byte = 255
poke_ptr(ptr, 16, other_ptr)   // offset 16: pointer

// Peek values from memory
val := peek_i32(ptr, 0)        // read int32 at offset 0
fval := peek_f32(ptr, 4)       // read float at offset 4

// Free memory
free(ptr)
```

**Peek/poke width coverage is asymmetric** (v0.140.0): peek has `i8`/`u8`/`i32`/`f32`; poke adds `u16`/`ptr`. There is **no `peek_i16`, `peek_u16`, `peek_i64`, `peek_f64`** — reading an `int16_t` array (e.g. a sim's height layer) needs either two `peek_u8` + shift/or, or a shim getter. Filed as machin issue #675 — check `machin guide` on newer builds before hand-rolling.

**Pointer param patterns for C functions:**

```mfl
// *T — MFL passes a pointer (int), C dereferences it
// Example: fn LoadModelFromMesh(*Mesh) — "mesh" is a ptr (int)

// T* — INOUT: MFL passes a cstruct VARIABLE by pointer,
// C can modify it and changes are written back
// Example: fn UploadMesh(Mesh*, bool)
```

## 7. cstruct limitations: can't be MFL struct fields

A `cstruct` declared in an `extern` block (`Model`, `Color`, `Sound`, `Texture2D`, `Shader`, …)
**cannot** be used as a field in an MFL `type` struct:

```mfl
type Planet struct {
    model Model      // COMPILE ERROR — C type not available at typedef site
    orbit float
}
```

The C typedef for the cstruct isn't generated until after the MFL type's C typedef,
so `mfl_Model` doesn't exist yet when `mfl_Planet` is compiled.

**Workaround:** use parallel slices indexed together:

```mfl
type Body struct { orbit float  speed float  angle float }
bodies := []Body{}
models := []Model{}              // separate slice for opaque handles
```

`cstruct` types CAN be local variables, function parameters, return values, and
stored in **slices** (`[]Model`, `[]Color`). The limitation is only on MFL `type`
struct fields.

## 8. Non-empty []struct slice literals

`[]S{a, b, c}` fails for struct types (works for scalars like `[]int{1,2,3}`):

```mfl
// WRONG — "non-empty []struct literals are not supported"
bodies := []Body{b1, b2, b3}

// RIGHT — build with append
bodies := []Body{}
bodies = append(bodies, b1)
bodies = append(bodies, b2)
bodies = append(bodies, b3)
```

(Empty `[]Body{}` is fine.)

## 9. Complete raylib example pattern

```mfl
extern "raylib" {
    header "raylib.h"
    link "raylib"
    link "GL"
    link "m"
    cflags "-I/usr/local/include"

    cstruct Color { r u8  g u8  b u8  a u8 }
    cstruct Vector2 { x f32  y f32 }

    fn InitWindow(i32, i32, string)
    fn WindowShouldClose() -> bool
    fn BeginDrawing()
    fn EndDrawing()
    fn CloseWindow()
    fn DrawFPS(i32, i32)
    fn ClearBackground(Color)
    fn DrawText(string, i32, i32, i32, Color)
    fn GetFrameTime() -> f32
    fn IsKeyDown(i32) -> bool
}

func main() {
    InitWindow(800, 600, "machin + raylib")
    BLACK := Color{r: 0, g: 0, b: 0, a: 255}
    WHITE := Color{r: 255, g: 255, b: 255, a: 255}

    for WindowShouldClose() == false {
        BeginDrawing()
        ClearBackground(BLACK)
        DrawText("Hello from MFL!", 10, 10, 20, WHITE)
        DrawFPS(10, 40)
        EndDrawing()
    }
    CloseWindow()
}
```

## 10. Shim-owned resources: one owner, one releaser

When a C shim manages a handle stored in an `alloc()` scratch block (GPU mesh/texture,
audio stream), pick exactly one side that frees it:

```mfl
// WRONG — double-free, heap corruption on the SECOND use (Windows: 0xc0000374)
UnloadMesh(wb + W_MESH())              // MFL frees, leaves stale vaoId/vboId in wb
rl_rebuild_mesh(wb + W_MESH())         // shim sees vaoId != 0, UnloadMesh() AGAIN
```

```mfl
// RIGHT — the shim owns it: it unloads the old handle internally on rebuild
rl_rebuild_mesh(wb + W_MESH())
```

This is the nastiest crash shape: the first lifecycle "works", the corrupt heap only
detonates at the next allocation — often a whole phase later (battle 1 fine, battle 2
CTD). ASan on a two-cycle repro (`--loop2`-style flag) catches it instantly. Same rule
for `alloc()` scratch that outlives a phase: free it on teardown even if exit leaks
don't crash — a re-entry path turns "leak" into growth.

## 11. raylib `DrawMeshInstanced`: the `MATRIX_MODEL` attribute-slot contract

`DrawMeshInstanced` does NOT look up an attribute named `instanceTransform` at
draw time. In raylib 5.0 (`rmodels.c`) it sends the instance `Matrix` array to
vertex attribute slots `shader.locs[SHADER_LOC_MATRIX_MODEL] + 0..3` — the slot
`LoadShader*` filled with the `matModel` *uniform* location. The shader must
declare `in mat4 instanceTransform;` **and** the app must patch that locs slot
to the attribute's location (this is what `lighting_instancing` does):

```c
// after LoadShaderFromMemory - patch in place via the shim
static void bind_inst(Shader *s) {
    s->locs[SHADER_LOC_MATRIX_MODEL] = rlGetLocationAttrib(s->id, "instanceTransform");
}
```

Other contracts that differ from a normal `DrawMesh` call:

- `mvp` carries **view×proj only** (model is forced to identity) — the shader
  must compute `mvp * instanceTransform * pos`; `matModel`/`matNormal` uniforms
  are never uploaded in an instanced draw, so don't read them.
- If the locs slot isn't patched, instances draw with a zero/garbage matrix —
  silently invisible, not an error.
- The instanced attribute binding is **persistent VAO state**: a mesh must not
  mix instanced and immediate `DrawMesh` calls in a run (divisor + enabled
  attrib survive on `mesh.vaoId`).
- Materials are per-mesh shared: if the same material serves both instanced and
  immediate draws, swap `material.shader` for the instanced call and restore it.

Same-frame numbers from the defil renderer (llvmpipe): 27524 `DrawMesh` calls →
1078 instanced draws, 53 ms → 16 ms.

## 11b. C arrays of by-value structs: `ptr` return + `*T` deref + byte offsets

When a C API returns `StructName *` (an array) and its consumers take `StructName`
**by value** — raylib's `LoadModelAnimations`/`UpdateModelAnimation` is the
canonical case — the whole dance is raw-memory indexing, no new FFI feature:

```mfl
cstruct ModelAnimation {}                      // opaque: never constructed, only *T-deref'd
fn LoadModelAnimations(string, ptr) ptr        // ModelAnimation* -> ptr; int* out-param via alloc(4)
cnt := alloc(8)
anims := LoadModelAnimations("clips.glb", cnt) // cnt receives animCount
n := peek_i32(cnt, 0)
free(cnt)

// element i lives at anims + i*sizeof(ModelAnimation) — pin the C lib version!
// raylib 5.0 LP64: {i32,i32,ptr,ptr,char[32]} = 56 bytes
MA_SIZE := 56                                  // func or literal; sizeof is layout-specific
frames := peek_i32(anims + i * MA_SIZE, 4)     // read a field straight out of the C struct
UpdateModelAnimation(model, anims + i * MA_SIZE, frame)  // *ModelAnimation param: MFL arg is the byte-address int, C derefs it
```

- `*T` params take **any int expression** as the address (`anims + i*56`), so
  array indexing is just byte arithmetic — get `sizeof` right or you read/write
  the wrong struct.
- Reading struct fields (`frameCount`, out-params like `int *animCount`) is
  `peek_*` on the raw address; writing out-params is `alloc` + pass `ptr`.
- Proven end-to-end in **painted-valley-machin** (`src/figure.src`): 33 glTF
  clips loaded from a skin-patched GLB and played on a skinned `Model`.

**raylib gotcha that forced an asset patch:** `LoadModelAnimations(file)`
requires the *animation file itself* to contain **exactly one skin** — it builds
its bone table from `skin.joints`. A clips-only GLB (no meshes/skins, which is
what three.js-style pipelines produce) fails with "expected exactly one skin".
Fix: inject `skins:[{joints:[...]}]` into the GLB JSON, joints = the model's
joint nodes **in the model's skin-joint order by name** (raylib also wants
parents before children or it warns per-frame and skips bones). See
`painted-valley-machin/tools/patch-clips-glb.py`.

## 12. Real-world FFI apps in the ecosystem

| App | What it demonstrates | Link |
|-----|---------------------|------|
| machin-game-demo-2048 | By-value Color struct, raylib GUI | [Source](https://github.com/javimosch/machin-game-demo-2048) |
| machin-game-demo-simon | Opaque Sound handle, audio | [Source](https://github.com/javimosch/machin-game-demo-simon) |
| machin-game-demo-3d | Nested cstruct (Camera3D), 3D rendering | [Source](https://github.com/javimosch/machin-game-demo-3d) |
| machin-game-demo-planet | Raw memory (alloc/poke), GPU mesh | [Source](https://github.com/javimosch/machin-game-demo-planet) |
| machin-game-demo-cyberpunk | Instancing, shaders, procedural worlds | [Source](https://github.com/javimosch/machin-game-demo-cyberpunk) |

## 13. Software framebuffer → GPU texture (`alloc` + `poke` + `UpdateTexture`)

Observed in machin-game-demo-nanite (CPU rasterizer presented through raylib).
Pattern for blitting a computed pixel buffer every frame:

```mfl
extern "raylib" {
    cstruct Image { data ptr  width i32  height i32  mipmaps i32  format i32 }
    cstruct Texture2D { id i32  w i32  h i32  mipmaps i32  fmt i32 }
    fn LoadTextureFromImage(Image) -> Texture2D
    fn UpdateTexture(Texture2D, ptr)
    fn LoadImageColors... // not needed — Image wraps raw pixels directly
}

g_fb = alloc(W * H * 4)                       // RGBA8, once
// per pixel: poke_i32(g_fb, (y*W+x)*4, r + g*256 + b*65536 + 0xFF000000)
img := Image{g_fb, W, H, 1, 7}                // format 7 = PIXELFORMAT_UNCOMPRESSED_R8G8B8A8
gl := LoadTextureFromImage(img)               // uploads ONCE
SetTextureFilter(gl, 0)                       // point filter for crisp pixels
// per frame after rasterizing:
UpdateTexture(gl, g_fb)                       // REQUIRED — the texture does not track the buffer
DrawTexturePro(gl, srcRect, dstRect, ...)
```

Gotchas: `LoadTextureFromImage` snapshots pixels at load — without
`UpdateTexture` per frame you display the initial (usually black) frame.
`TakeScreenshot` must run **after `EndDrawing`** — raylib batches draw calls
and flushes at end-of-frame, so capturing earlier yields a stale buffer.

## 14. Earth-scale raylib scenes: separate world, render and material frames

Dogfooded in pixel-town-world (Earth radius 6,371,000m, metre-scale buildings):

- MFL `float` is double precision, but raylib `Vector3`/GPU vertex positions
  are `f32`. Subtract a shared focus origin in MFL BEFORE crossing the FFI;
  also store mesh vertices relative to nearby anchors. Subtracting absolute
  Earth-radius positions in GLSL is already too late to preserve centimetres.
- Camera, draw transforms, projected labels and fog must use the SAME scaled,
  focus-relative frame. Keep procedural material coordinates in an independent
  stable tangent frame so textures do not rotate on southern latitudes or swim
  when chunk origins move. Engine APIs take basis vectors/offsets, not city data.
- raylib's default perspective clip range is not suitable for a planet atlas.
  Set the custom projection AFTER `BeginMode3D`, which otherwise replaces it.
  A street camera needs a different depth range (e.g. near=0.08m/far=6000m),
  with the SAME FOV in Camera3D and the projection, not planet-wide planes.
- Use `SetExitKey(0)` when Esc should leave a mode rather than close the window.
  Pair `DisableCursor` with `EnableCursor` on mode exit, focus loss and shutdown;
  pause while unfocused and discard the first mouse delta after capture/resume.
- Deterministic capture flags must ignore live input and avoid cursor capture.
  Capture all destinations plus the return path; one attractive dry city can
  hide invalid spawn assumptions in wet regions. Headless movement tests and
  framebuffer screenshots verify different things; keep both.

See pixel-town-world/docs/world-atlas-approach.md and its --verify/--street-tour
entry points for the maintained implementation, not a separate surface-world swap.

For procedural billboard crowds sharing a grass shader, use raylib's default
`vertexTexCoord` attribute name or explicitly bind a custom one. Sprite UV.x
must not also drive grass bending; disable wind for the sprite material, and
discard transparent atlas texels before depth write. Keep dynamic mesh buffers
separate from the static mesh builder's scratch, with one resource owner.
Palette quantization is part of an era profile: increasing emission cannot
recover cyan if the palette only offers cream. Flat window trim/panes need
quads, not five-sided boxes; check mesh capacity as procedural detail grows.
An apparent FALS002 false positive in the same spike was actually a smoothstep
helper lacking a zero-width-range guard. Guarding it removed the warning.

For native Windows raylib builds, emit Windows-target C, not native Linux C.
The runtime/winpthreads can pull in Win32 headers before the FFI header;
`Rectangle`, `CloseWindow` and `ShowCursor` then collide with raylib names.
`-DNOGDI -DNOUSER` fixes this app when it does not itself use those Win32 APIs;
other consumers needing them should isolate/rename the conflicting declarations.
Explicit Zig linking lets the app supply its third-party include/library flags.
A successful SSH headless run does not verify the interactive OpenGL session:
pixel-town-world also ran its bounded, cursor-free demo in the logged-in
Windows console via a temporary interactive-token task (principal = current
identity SID). Verify its exit status and screenshots; remove your test task.
