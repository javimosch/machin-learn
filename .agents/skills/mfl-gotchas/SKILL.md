---
name: mfl-gotchas
description: "Every MFL gotcha, pitfall, caveat, and 'why doesn't this compile' moment discovered through real-world usage building machin-pomo and machin-serve. Read this BEFORE writing any MFL code."
---

# MFL Gotchas & Pitfalls

Compiled from real-world usage building [machin-pomo](https://github.com/javimosch/machin-pomo) and [machin-serve](https://github.com/javimosch/machin-serve), plus studying the [spec](https://github.com/javimosch/machin/blob/main/SPEC.md).

## 🚨 Critical (will crash or silently break)

### 1. `read_file` on a directory = SEGFAULT

```mfl
// WRONG — crashes with SIGSEGV
body := read_file("./mydir")  // mydir is a directory, not a file

// RIGHT — check with list_dir first
entries := list_dir("./mydir")
if len(entries) > 0 {
    // It's a directory — serve index.html instead
    body := read_file("./mydir/index.html")
} else {
    body := read_file("./mydir")  // safe: not a dir
}
```

This is the #1 runtime crash. `list_dir` returns non-empty for directories, empty for files and non-existent paths. Use it as a directory check before calling `read_file`.

### 2. Named return shadowing with `:=`

```mfl
func fmt_time(secs) (s) {
    m := secs / 60
    // WRONG — shadows the named return 's' with a local int!
    s := secs % 60
    return pad2(m) + ":" + pad2(s)  // compile error: type mismatch

    // RIGHT — use a different name for the local
    sec := secs % 60
    return pad2(m) + ":" + pad2(sec)
}
```

`:=` always creates a new local variable, even if a named return with the same name exists. Use `=` to assign to the named return, or use a different name for locals.

### 3. No string slice syntax

```mfl
s := "hello world"
// WRONG — doesn't compile
part := s[0:5]

// RIGHT — use substr builtin
part := substr(s, 0, 5)  // "hello"

// To get the last character:
last := charat(s, len(s)-1)  // "d"

// To remove trailing char:
trimmed := substr(s, 0, len(s)-1)
```

### 4. `byte_at` works on `bytes`, NOT `string`

```mfl
s := "hello"
// WRONG — type mismatch: string vs bytes
b := byte_at(s, 0)

// RIGHT — use charat or convert to bytes
c := charat(s, 0)     // "h" (string)
b := byte_at(bytes(s), 0)  // 104 (int, ASCII value)
```

### 5. No `!` logical NOT operator

```mfl
// WRONG — doesn't compile
if !ok { ... }

// RIGHT — use == false
if ok == false { ... }
```

### 6. Multi-return builtins REQUIRE multi-assign

```mfl
// WRONG — compile error: multi-assign only
body := http_get("https://example.com/")

// RIGHT
status, body, err := http_get("https://example.com/")

// Also applies to json_get
// WRONG
val := json_get(data, ".name")

// RIGHT
val, err := json_get(data, ".name")
```

### 7. `contains` is a builtin — don't name functions after it

You cannot define a function named `contains`, `len`, `str`, `flush`, `keys`, `print`, `println`, `exit`, `sleep`, or any other builtin name:

```mfl
func contains(xs, x) (b) { ... }  // COMPILE ERROR
```

## ⚠️ Common mistakes

### 8. `http_get` vs `https_get`

| Function | Returns | Use case |
|----------|---------|----------|
| `http_get(url)` | `(status, body, err)` | Full control, error handling |
| `https_get(url)` | `body` (string) | Simple GET, "" on error |
| `https_post(url, body)` | `body` (string) | Simple POST |

`http_get` works with both http:// and https:// URLs.

### 9. `regex_groups` indexing

```mfl
m := regex_groups(str, "GET ([^ ]+) HTTP")
// Index 0: full match  ("GET /index.html HTTP")
// Index 1+: captures  ("/index.html")
// So the capture group is at m[1], not m[0]
path := "/"
if len(m) > 1 { path = m[1] }
```

### 10. Structs are value types

```mfl
type Point struct { x int  y int }

func move(p) {
    p.x = p.x + 1  // modifies the COPY, not the caller's Point
}

func main() {
    pt := Point{x: 1, y: 2}
    move(pt)
    println(str(pt.x))  // 1 — unchanged!
}
```

For mutable state, use a map:

```mfl
type State struct {
    m  map[string]int   // shared mutable state
}
```

Or return the modified struct:

```mfl
func move(p) (out) {
    out = p
    out.x = out.x + 1
}
```

### 11. stdout buffering with pipes

```mfl
// When piped, stdout is fully buffered — output won't appear
// until the buffer fills or the program exits

// ALWAYS call flush() after writes when output matters:
print(some_output)
flush()
```

### 12. Map comma-ok doesn't exist

```mfl
m := make(map[string]int)
m["k"] = 42

// WRONG — doesn't compile
v, ok := m["k"]

// RIGHT — use has() to test presence
if has(m, "k") {
    v := m["k"]
}
```

### 13. Lambda functions can't have named returns

```mfl
// WRONG — doesn't parse
adder := func(x) (s) { s = x + 1 }

// RIGHT — use return expression
adder := func(x) { return x + 1 }
```

### 14. `parse_int` returns 0 on failure (no error)

```mfl
n := parse_int("not-a-number")
println(str(n))  // 0

// No way to distinguish 0 from parse failure
```

Validate inputs before parsing!

### 15. Comma-ok receive from closed channel

```mfl
ch := make(chan int)
close(ch)

// A closed channel immediately fires its receive case
// with ok==false — detect it!
v, ok := <-ch
if ok == false {
    // channel is closed — stop selecting on it
}
```

## 🔧 Build & toolchain

### 16. Workflow: .src → .mfl → binary

Always write loose `.src` files, encode to `.mfl`, then build:

```bash
machin encode app.src > app.mfl
machin build app.mfl -o app
```

Do NOT try to write `.mfl` directly — the canonical form is hard to edit.

### 17. Build with --safe during development

```bash
machin build app.mfl --safe -o app
```

Catches out-of-bounds slice access, division by zero, and integer overflow at runtime.

### 18. Get the full picture in one call

```bash
machin guide --text
```

Prints every keyword, builtin with signature, idiom, and gotcha — version-exact, guaranteed to match the implementation.

## 💡 Performance tips

### 19. Memory: per-goroutine arena

- Each goroutine gets its own arena — memory is freed on return
- Wrap hot allocation loops in `arena { ... }` to keep peak memory flat
- Scalars (int, float, bool) are not heap-allocated

### 20. Channel sends deep-copy values

Values sent over a channel are COPIED (strings fast, slices/maps/structs via JSON). Good for safety across goroutine boundaries, but be aware of the overhead for large data.
