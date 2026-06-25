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

**Pointer param patterns for C functions:**

```mfl
// *T — MFL passes a pointer (int), C dereferences it
// Example: fn LoadModelFromMesh(*Mesh) — "mesh" is a ptr (int)

// T* — INOUT: MFL passes a cstruct VARIABLE by pointer,
// C can modify it and changes are written back
// Example: fn UploadMesh(Mesh*, bool)
```

## 7. Complete raylib example pattern

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

## 8. real-world FFI apps in the ecosystem

| App | What it demonstrates | Link |
|-----|---------------------|------|
| machin-game-demo-2048 | By-value Color struct, raylib GUI | [Source](https://github.com/javimosch/machin-game-demo-2048) |
| machin-game-demo-simon | Opaque Sound handle, audio | [Source](https://github.com/javimosch/machin-game-demo-simon) |
| machin-game-demo-3d | Nested cstruct (Camera3D), 3D rendering | [Source](https://github.com/javimosch/machin-game-demo-3d) |
| machin-game-demo-planet | Raw memory (alloc/poke), GPU mesh | [Source](https://github.com/javimosch/machin-game-demo-planet) |
| machin-game-demo-cyberpunk | Instancing, shaders, procedural worlds | [Source](https://github.com/javimosch/machin-game-demo-cyberpunk) |
