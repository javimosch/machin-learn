---
name: mfl-terminal
description: "Terminal/TUI programming in MFL: raw_mode/read_key for live input, ANSI escape sequences for colored rendering, progress bars, frame loops, and terminal state management. Build interactive CLI apps."
---

# MFL Terminal Programming

Build interactive terminal applications using machin's `raw_mode`/`read_key` builtins and ANSI escape codes.

## 1. Terminal state management

```mfl
// Enable raw mode (cbreak + no echo)
raw_mode(1)

// ... app logic ...

// RESTORE raw mode before exiting — critical!
raw_mode(0)
```

**⚠️ ALWAYS restore raw mode before exit.** If the program crashes without calling `raw_mode(0)`, the terminal will be in a broken state (no echo, no line buffering). Users will need to run `reset` or `stty sane`.

## 2. ANSI escape codes

Build the ESC byte (0x1b) from hex since MFL strings don't have `\x` escapes:

```mfl
func esc() (s) { s = bytes_str(from_hex("1b")) }

func home() (s) { s = esc() + "[H" }           // cursor to 0,0
func hide() (s) { s = esc() + "[?25l" }        // hide cursor
func show() (s) { s = esc() + "[?25h" }        // show cursor
func clr() (s) { s = esc() + "[2J" }           // clear screen
func col(n) (s) { s = esc() + "[" + str(n) + "m" }  // SGR color
func rst() (s) { s = esc() + "[0m" }           // reset all attributes
func bold() (s) { s = esc() + "[1m" }          // bold
```

**Common ANSI colors:**

```mfl
col(92)  // bright green
col(93)  // bright yellow
col(94)  // bright blue
col(97)  // bright white
col(90)  // bright black (grey)
col(42)  // green background
col(100) // dark grey background
col(91)  // bright red
col(32)  // green
```

## 3. Full rendering pattern (from machin-pomo)

Build the entire frame as one string, then `print` + `flush`:

```mfl
func render(state, remaining, total, cycle, done, paused, interval) {
    out := home()

    // Header
    if state == 0 {
        out = out + bold() + col(92) + "  >>> POMODORO [" + str(cycle) + "/" + str(interval) + "]" + rst() + "\n"
    } else if state == 1 {
        out = out + bold() + col(94) + "  >>> SHORT BREAK" + rst() + "\n"
    } else {
        out = out + bold() + col(93) + "  >>> LONG BREAK" + rst() + "\n"
    }
    out = out + "  " + rep("=", 28) + "\n\n"

    // Time display (centered)
    ts := fmt_time(remaining)
    pad := (30 - len(ts)) / 2
    if pad < 0 { pad = 0 }
    out = out + "  " + rep(" ", pad) + bold() + col(97) + ts + rst() + "\n\n"

    // Progress bar
    pct := 0.0
    if total > 0 {
        pct = float(total - remaining) * 100.0 / float(total)
    }
    bw := 28
    filled := int(pct * float(bw) / 100.0)
    if filled > bw { filled = bw }
    out = out + "  ["
    out = out + col(42)  // green background
    i := 0
    while i < filled {
        out = out + " "
        i = i + 1
    }
    out = out + rst()
    i = filled
    while i < bw {
        out = out + " "
        i = i + 1
    }
    out = out + "]\n"

    // Status
    if paused {
        out = out + col(93) + "  ** PAUSED **" + rst() + "\n"
    } else {
        out = out + col(92) + "  >> RUNNING" + rst() + "\n"
    }
    out = out + col(90) + "  Controls: [p] pause [r] resume [s] skip [q] quit" + rst() + "\n"

    print(out)
    flush()
}
```

## 4. Keyboard input loop

```mfl
running := true
while running {
    // Render current state
    render(state, remaining, total, cycle, done, paused, interval)

    // Process non-blocking key input
    k := read_key()
    if k == "q" { running = false }
    if k == "p" { paused = true }
    if k == "r" { paused = false }
    if k == "s" { skip_phase() }

    // Tick (e.g., 1 second)
    if paused == false && remaining > 0 {
        remaining = remaining - 1
    }

    sleep(1000)  // 1-second tick
}
```

`read_key()` is non-blocking — returns `""` if no key is pressed. Check for keys each frame.

## 5. Frame timing

Use `sleep(ms)` for the frame rate:

```mfl
sleep(1000)   // 1 tick per second (Pomodoro timer)
sleep(110)    // ~9 FPS (Snake game)
sleep(50)     // ~20 FPS (smooth animation)
```

## 6. Helper: string repeat

```mfl
func rep(ch, n) (s) {
    s = ""
    i := 0
    while i < n {
        s = s + ch
        i = i + 1
    }
}
```

## 7. Helper: zero-padded numbers

```mfl
func pad2(n) (s) {
    if n < 10 { return "0" + str(n) }
    return str(n)
}

func fmt_time(secs) (s) {
    m := secs / 60
    sec := secs % 60     // NOTE: don't name this 's' — shadows named return!
    return pad2(m) + ":" + pad2(sec)
}
```

## 8. Full skeleton

```mfl
// ---- ANSI helpers ----
func esc() (s) { s = bytes_str(from_hex("1b")) }
func home() (s) { s = esc() + "[H" }
func hide() (s) { s = esc() + "[?25l" }
func show() (s) { s = esc() + "[?25h" }
func clr() (s) { s = esc() + "[2J" }
func col(n) (s) { s = esc() + "[" + str(n) + "m" }
func rst() (s) { s = esc() + "[0m" }
func bold() (s) { s = esc() + "[1m" }

func main() {
    raw_mode(1)
    print(hide() + clr())

    // ... app init ...

    running := true
    while running {
        // render
        // handle input
        // tick
        sleep(100)  // or whatever frame rate
    }

    raw_mode(0)
    print(show())
    print(clr() + home())
    println("Goodbye.")
}
```

## 9. Known patterns from real apps

| App | Technique | Link |
|-----|-----------|------|
| machin-pomo | Live countdown timer with progress bar, keyboard controls | [Source](https://github.com/javimosch/machin-pomo) |
| Snake game | Real-time game loop, WASD movement, colored rendering | [Source](https://github.com/javimosch/machin-game-demo-snake) |
