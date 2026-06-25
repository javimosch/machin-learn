---
name: mfl-builtins
description: "Complete MFL builtin catalog organized by category with signatures, usage examples, and common patterns. Covers I/O, CLI, time, convert, math, string, bytes, crypto, SQLite, regex, JSON, HTTP, networking, and terminal builtins."
---

# MFL Builtins — Complete Catalog

Organized by category. Run `machin guide --text` for the version-exact catalog.

## I/O

```mfl
print(...)              // write args, no trailing newline
println(...)            // write args + trailing newline
flush()                 // flush stdout buffer (CRITICAL for pipes!)
input() -> string       // read one stdin line ("" at EOF)
read_stdin() -> string  // read ALL of stdin verbatim
```

**Pattern:** Always call `flush()` after `print()` when output might be piped.

## File I/O

```mfl
read_file(path) -> string           // read file ("" on error)
read_file_bytes(path) -> bytes      // read raw bytes, NUL-safe
write_file(path, content) -> int    // write file (-1 on error)
list_dir(path) -> []string          // list directory entries
mkdir(path) -> int                  // create directory (0 ok, -1 error)
```

**⚠️ `read_file` on a directory = SEGFAULT.** Always check with `list_dir` first:

```mfl
fpath := "./somepath"
entries := list_dir(fpath)
if len(entries) > 0 {
    // It's a directory — serve index.html
    body := read_file(fpath + "/index.html")
} else {
    body := read_file(fpath)  // safe
}
```

**`list_dir` returns:**
- Non-empty `[]string` for directories (entries count)
- Empty `[]string` for files and non-existent paths

## CLI

```mfl
args() -> []string       // CLI args (index 0 = program path)
env(name) -> string      // env var ("" if unset)
exit(code)               // terminate process
```

**Pattern:** `a[0]` is the program path; actual args start at `a[1]`.

## Time

```mfl
now() -> int                  // Unix seconds
now_ms() -> int               // Unix milliseconds
sleep(ms) ->                  // pause for N ms
time_fields(ts) -> []int      // [year,month,day,hour,min,sec,weekday,yearday]
time_format(ts, pattern) -> string  // strftime formatting (local time)
time_format_utc(ts, pattern) -> string  // strftime formatting (UTC)
time_make(year,mon,day,hour,min,sec) -> int  // calendar → Unix timestamp
```

**Notes:**
- `time_fields` weekday: 0=Sunday
- `time_format` uses strftime patterns: `%Y %m %d %H %M %S %A %B %z %Z %F %T`
- `time_make` is the inverse of `time_fields` — normalizes overflow

## Convert

```mfl
str(val) -> string        // format int|float|bool|string
int(n) -> int             // truncate to int (from float)
float(n) -> float         // int → float conversion
parse_int(s) -> int       // parse string → int (0 on failure)
```

**⚠️** No implicit int→float. Use `float(n)` explicitly:

```mfl
n := 42
// x := n / 2.0       ← compile error
x := float(n) / 2.0    // OK
```

**⚠️** `parse_int` returns 0 for non-numeric input — no error detection.

## Math

```mfl
sqrt(x), cbrt(x), pow(x,y), exp(x), log(x)
sin(x), cos(x), tan(x), asin(x), acos(x), atan(x), atan2(y,x)
hypot(x,y), floor(x), ceil(x), round(x), trunc(x)
abs(x), fmod(x,y)
noise2(x,y), noise3(x,y,z)
```

All take `int|float` (number), return `float`.

## String operations

```mfl
len(s) -> int                         // byte length
substr(s, start, end) -> string       // substring [start, end)
charat(s, i) -> string                // 1-char string at index
contains(s, sub) -> bool              // substring test
has_prefix(s, prefix) -> bool
has_suffix(s, suffix) -> bool
index(s, sub) -> int                  // first index, or -1
replace(s, old, new) -> string        // replace all
split(s, sep) -> []string             // split on separator
join(xs, sep) -> string               // join with separator
to_upper(s) -> string
to_lower(s) -> string
trim(s) -> string                     // trim surrounding whitespace
```

**No slice syntax on strings** — use `substr`:

```mfl
s := "hello world"
part := substr(s, 0, 5)         // "hello"
last := charat(s, len(s)-1)     // "d"
trimmed := substr(s, 0, len(s)-1)  // remove last char
```

## URL encoding

```mfl
url_encode(s) -> string    // RFC 3986 percent-encoding
url_decode(s) -> string    // percent-decode (+ → space)
```

## Bytes & crypto

```mfl
bytes(s) -> bytes              // string → bytes
bytes_str(b) -> string         // bytes → string (NUL-terminated)
bytes_concat(a, b) -> bytes    // concatenate two byte values
to_hex(b) -> string            // lowercase hex
from_hex(s) -> bytes           // parse hex (skips non-hex)
base64_encode(s) -> string
base64_decode(s) -> string
base64_encode_bytes(b) -> string
base64_decode_bytes(s) -> bytes
sha256(s) -> string            // lowercase hex
hmac_sha256(key, msg) -> string  // lowercase hex
```

## Regex

```mfl
regex_match(s, pattern) -> bool          // ERE match anywhere?
regex_find(s, pattern) -> string         // first match
regex_groups(s, pattern) -> []string     // [0]=full match, [1..]=captures
regex_replace(s, pattern, repl) -> string  // replace all
```

**Capture groups indexing:**

```mfl
m := regex_groups("GET /path HTTP/1.1", "GET ([^ ]+) HTTP")
// m[0] = "GET /path HTTP"
// m[1] = "/path"             ← THIS is the capture
if len(m) > 1 { path = m[1] }
```

## JSON

```mfl
json(val) -> string                    // serialize to JSON
parse(json_str, T{}) -> T              // parse JSON → struct
parse(json_arr, []T{}) -> []T          // parse JSON array → slice
json_get(s, path) -> (string, string)  // jq-style: (value, err)
```

**`json_get` returns QUOTED strings** — a string field comes back as `"\"Ada\""`. Prefer `parse()` for whole objects.

**Multi-assign only:**

```mfl
val, err := json_get(data, ".name")
// NOT: val := json_get(data, ".name")
```

## SQLite

```mfl
sqlite_open(path) -> int                  // handle (0 on fail)
sqlite_exec(h, sql, [params]) -> int       // 0 ok, -1 error
sqlite_query(h, sql, [params]) -> string   // JSON array of rows
```

**Always use parameterized queries (`?` + []string params):**

```mfl
h := sqlite_open("mydb.db")
sqlite_exec(h, "CREATE TABLE IF NOT EXISTS users (id INTEGER, name TEXT)")
sqlite_exec(h, "INSERT INTO users VALUES (?, ?)", []string{"1", "Alice"})
rows := sqlite_query(h, "SELECT * FROM users WHERE id = ?", []string{"1"})
// rows is a JSON string — decode with parse():
users := parse(rows, []User{})
```

## HTTP & networking

```mfl
// HTTP client
http_get(url) -> (status, body, err)       // multi-assign only!
https_get(url) -> string                    // body ("" on error)
https_post(url, body) -> string             // POST JSON ("" on error)
http_request(method, url, headers, body) -> (status, body, err)

// Raw TCP
dial(host, port) -> int                     // fd (-1 on fail)
listen(port) -> int                         // listener fd
accept(server) -> int                       // connection fd
read(fd) -> string                          // read from fd (blocks)
write(fd, data) -> int                      // write to fd
close(fd)                                   // close fd

// WebSocket
wss_open(url) -> int                        // handle (0 on fail)
wss_send(handle, msg) -> int
wss_recv(handle) -> string                  // blocks; "" on close
```

**Server pattern:**

```mfl
server := listen(8080)
for {
    conn := accept(server)
    go handle(conn)
}

func handle(conn) {
    req := read(conn)
    write(conn, "HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nOK")
    close(conn)
}
```

## Terminal input

```mfl
raw_mode(mode) -> int    // 1=cbreak/no-echo, 0=restore
read_key() -> string     // non-blocking single-key read ("" if none)
```

**Pattern:**

```mfl
func main() {
    raw_mode(1)
    println("Press q to quit")
    running := true
    while running {
        k := read_key()
        if k == "q" { running = false }
        if k == "p" { println("paused") }
        sleep(100)
    }
    raw_mode(0)
}
```

**⚠️ Always restore raw mode before exit** — if the program crashes without `raw_mode(0)`, the terminal will be in a broken state.

## Other

```mfl
len(x) -> int        // string bytes, slice/map length
has(m, k) -> bool    // map key existence
keys(m) -> []string  // map keys
delete(m, k)         // delete map key
rand_bytes(n) -> bytes  // cryptographically random bytes
byte_at(b, i) -> int    // byte at index in bytes value
alloc(n) -> int         // allocate n zeroed bytes (returns pointer)
free(p)                 // free allocated memory
ptr_str(p) -> string    // read NUL-terminated string from pointer
```

## ERROR: builtin name conflicts

```mfl
// WRONG — these are ALL builtin names, can't be used as user functions:
func len(x) { ... }       // ✗
func str(x) { ... }       // ✗
func flush() { ... }      // ✗
func contains(x) { ... }  // ✗
func keys(m) { ... }      // ✗
func print(x) { ... }     // ✗
func exit(n) { ... }      // ✗
func sleep(n) { ... }     // ✗
func split(s) { ... }     // ✗
func join(xs) { ... }     // ✗
```

Use different names: `my_len`, `to_string`, `do_flush`, etc.
