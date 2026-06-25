---
name: mfl-language
description: "Master the MFL (Machine-First Language) syntax, type system, functions, structs, generics, control flow, and concurrency. Covers the canonical form, inference rules, and compilation model."
---

# MFL Language — Complete Reference

MFL (Machine-First Language) is a Go-flavored, statically typed, type-inferred language that compiles through C to a single native binary. Unboxed values, no interpreter, no VM.

## 1. Source formats

| Form | Extension | Usage |
|------|-----------|-------|
| **Loose** (agent-friendly) | `.src` | Write multi-line Go-like source; can have comments, whitespace |
| **Canonical** (tight) | `.mfl` | One declaration per line, no extra whitespace; what the compiler reads |
| **Packed** (distribution) | `.mfl` | Base64-encoded single line per decl; `machin pack` produces it |

```bash
# Write loose source, then encode to canonical
machin encode myapp.src > myapp.mfl

# Build
machin build myapp.mfl -o myapp

# Or run directly
machin run myapp.mfl
```

## 2. Program structure

A `.mfl` file is a sequence of **declarations**, one per non-blank line, separated by blank lines.

### Function declaration

```mfl
func add(a, b) { return a + b }
```

- Parameters are untyped; types are **inferred** from usage
- Return types are inferred too - no annotations anywhere
- One declaration per line in canonical form

### Named returns

```mfl
func divmod(a, b) (q, r) {
    q = a / b
    r = a % b
    return
}
```

Named returns:
- Declare return variable names in `(name1, name2)` after params
- Assign to them in the body
- `return` by itself returns their current values
- **WARNING:** `:=` creates a new LOCAL variable that shadows the named return (see pitfall below)

### Struct type

```mfl
type User struct {
    name   string
    age    int
    tags   []string
}
```

- Struct fields have explicit types
- Structs are **value types**: copying/passing copies the whole struct
- Use maps (`map[K]V`) for shared mutable state

### Package globals

```mfl
var count = 0
var cache = make(map[string]int)
```

- Mutable, shared by all functions
- Persist across calls (useful for wasm exports holding state)
- `=` assigns the global; `:=` makes a local (may shadow!)

## 3. Types

| Type | Description | Zero value |
|------|-------------|------------|
| `int` | 64-bit signed | `0` |
| `float` | IEEE-754 double | `0.0` |
| `bool` | true/false | `false` (`0`) |
| `string` | immutable UTF-8 | `""` |
| `bytes` | NUL-safe binary buffer | empty bytes |
| `[]T` | slice of T | `nil` |
| `map[K]V` | hash map (K=int or string) | `nil` |
| `chan T` | channel | `nil` |
| `func` | function value / closure | `nil` |
| struct | named record | zero-valued fields |

### Key inference rules

```mfl
n := 42        // int
x := 3.14      // float
s := "hello"   // string
b := true      // bool
xs := []int{}  // []int — must specify element type in literal
m := make(map[string]int)  // map[string]int
ch := make(chan int)       // chan int
```

### int ↔ float conversion

**NO implicit int→float.** Only a numeric LITERAL promotes against float:

```mfl
func main() {
    n := 42
    // x := n / 2.0     ← COMPILE ERROR: int vs float
    x := float(n) / 2.0  // OK — explicit conversion
    
    y := 5 / 2.0  // OK — 5 is a literal, promotes to float
}
```

### Zero values

```mfl
var s string     // ""
var n int        // 0
var b bool       // false (0)
var xs []int     // nil (empty slice, len 0)
```

A map read of an absent key returns the value type's zero value. **There is no map comma-ok** — use `has(m, k)` to test presence.

## 4. Control flow

### Conditionals

```mfl
if x < 10 {
    println("small")
} else if x < 100 {
    println("medium")
} else {
    println("large")
}
```

**No `!` operator** — use `== false`:

```mfl
if ok == false {
    println("not ok")
}
```

### Loops

```mfl
// while loop
i := 0
while i < 10 {
    println(str(i))
    i = i + 1
}

// for-range over slice
for _, v := range xs {
    println(str(v))
}

// for-range over map
for k, v := range m {
    println(k + ": " + str(v))
}

// for-range over channel
for v := range ch {
    println(str(v))
}

// infinite loop (like for{} in Go)
for {
    conn := accept(server)
    go handle(conn)
}
```

## 5. Functions

### Multiple return values

```mfl
func stats(xs) (min, max, sum) {
    min = xs[0]
    max = xs[0]
    sum = xs[0]
    i := 1
    while i < len(xs) {
        if xs[i] < min { min = xs[i] }
        if xs[i] > max { max = xs[i] }
        sum = sum + xs[i]
        i = i + 1
    }
    return
}

// Call with multi-assign
mn, mx, sm := stats(nums)
```

### Variadic parameters

```mfl
func sum(nums...) (t) {
    t = 0
    for _, n := range nums {
        t = t + n
    }
}
```

### Closures (by-reference capture)

```mfl
func counter() (f) {
    n := 0
    f = func() (v) {
        n = n + 1
        return n
    }
}

c := counter()
println(str(c()))  // 1
println(str(c()))  // 2
```

### Implicit generics (monomorphized)

```mfl
func id(x) (v) { v = x }

// Each call compiles a separate native function
id(42)          // int version
id("hello")     // string version
id(3.14)        // float version
```

## 6. Concurrency

### Goroutines

```mfl
go myfunc(arg1, arg2)
```

### Channels

```mfl
ch := make(chan int)
ch <- 42          // send
v := <-ch         // receive
v, ok := <-ch     // comma-ok receive (ok=false after close)
close(ch)         // close
```

### Select

```mfl
select {
case v := <-ch1:
    println(str(v))
case v := <-ch2:
    println(str(v))
case <-timeout:
    println("timeout")
default:
    println("no one ready")
}
```

### Worker pool pattern (bounded concurrency)

Run N goroutines competing on a shared jobs channel, collecting results via `select`:

```mfl
func worker(host, jobs, results, done) {
    for {
        port, ok := <-jobs
        if ok == false { break }    // jobs channel closed
        if try_port(host, port) {
            results <- port         // send result back
        }
    }
    done <- true                     // signal completion
}

func main() {
    jobs := make(chan int)
    results := make(chan int)
    done := make(chan bool)

    // Start N workers
    w := 0
    while w < 100 {
        go worker(host, jobs, results, done)
        w = w + 1
    }

    // Dispatch jobs
    j := 0
    while j < 1000 {
        jobs <- port_list[j]
        j = j + 1
    }
    close(jobs)   // workers exit after draining

    // Collect results via select
    workers_done := 0
    collecting := true
    while collecting {
        select {
        case port := <-results:
            println(str(port))       // process result
        case <-done:
            workers_done = workers_done + 1
            if workers_done >= concurrency {
                collecting = false   // all workers finished
            }
        }
    }
}
```

**Key points:**
- Close `jobs` to signal workers that no more work is coming
- Workers detect close via `port, ok := <-jobs` / `ok == false`
- Use `done` channel per worker to track completion
- `select` reads from either results or done channels
- Bound concurrency by fixing the number of workers

### Timeout via goroutine + channel

```mfl
func after(ms, ch) {
    sleep(ms)
    ch <- true
}

timeout_ch := make(chan bool)
go after(5000, timeout_ch)

select {
case result := <-results:
    // got a result in time
case <-timeout_ch:
    println("timed out")
}
```

### Arena GC

Each goroutine has its own arena — memory is reclaimed when the goroutine returns.

```mfl
// Scoped arena for hot loops
arena {
    // Allocations here are freed when the block ends
    tmp := expensiveComputation()
    result = extractResult(tmp)
}
// tmp is freed, result survives (scalars are safe)
```

## 7. Error handling idiom

Some builtins return `(value, err)` pairs:

```mfl
status, body, err := http_get("https://example.com/")
if len(err) > 0 {
    println("error: " + err)
    exit(1)
}
```

**MULTI-ASSIGN ONLY** — calling them as a single value is a compile error.

## 8. Compilation model

```
.mfl → parse → lambda-lift → infer+monomorphize → emit C → cc -O2 → native binary
                                                         └→ zig cc → .wasm
```

- **Inference:** Unification over union-find — deferred resolution for `x[i]`, `x.f`, `range`
- **Monomorphization:** One native function per concrete call-site signature (deduplicated)
- **--safe:** Inserts bounds/div-zero/overflow checks (opt-in, zero overhead by default)

## 9. Lambda functions

```mfl
// Lambda — NO named returns allowed
adder := func(x) { return x + 1 }

// WRONG: func(x) (s) { s = x + 1 } — COMPILE ERROR
```

## 10. Builtin naming restriction

You cannot name a user function after a builtin:

```mfl
func flush() { ... }     // COMPILE ERROR — flush is a builtin
func len() { ... }       // COMPILE ERROR — len is a builtin
func str(x) { ... }      // COMPILE ERROR — str is a builtin
```
