---
name: mfl-storage
description: "Data persistence in MFL: file I/O patterns, SQLite integration, JSON serialization, and the critical caveats around read_file segfaults, list_dir usage, and parameterized queries."
---

# MFL Storage & Persistence

File I/O, SQLite, and JSON data handling in MFL.

## 1. File I/O

### Reading files

```mfl
// Text files
content := read_file("data.txt")       // "" on error (file not found or empty)
bytes := read_file_bytes("data.bin")   // NUL-safe binary read
```

**⚠️ CRITICAL: `read_file` on a DIRECTORY segfaults**

```mfl
// This crashes with SIGSEGV:
body := read_file("./mydir")  // if mydir is a directory

// Always check with list_dir first:
entries := list_dir("./mydir")
if len(entries) > 0 {
    // It's a directory — handle accordingly
} else {
    body := read_file("./mydir")  // safe
}
```

### list_dir behavior

```mfl
list_dir(".")                 // returns entries in current directory
list_dir("./subdir")          // returns entries (2 in test)
list_dir("file.txt")          // returns empty []string (not a dir)
list_dir("nonexistent")       // returns empty []string (not found)
```

`list_dir` returns:
- Non-empty `[]string` → path is a directory
- Empty `[]string` → path is a file or doesn't exist

### Writing files

```mfl
ok := write_file("output.txt", "hello world")
if ok == -1 {
    println("write failed")
}

// write_file returns -1 on error, some positive value on success
// (it's the number of bytes written? Not documented — check for -1)
```

### Directory operations

```mfl
entries := list_dir(".")             // list directory
ok := mkdir("newdir")                // create directory (0 ok, -1 error)
```

## 2. SQLite

SQLite is embedded directly — no external database needed.

### Opening a database

```mfl
db := sqlite_open("mydb.db")     // returns handle (0 on fail)
memory_db := sqlite_open(":memory:")  // in-memory database
```

### Executing statements

```mfl
// DDL
sqlite_exec(db, "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT, age INTEGER)")

// INSERT with parameterized query (ALWAYS use ? bindings!)
sqlite_exec(db, "INSERT INTO users (name, age) VALUES (?, ?)", []string{"Alice", "30"})
sqlite_exec(db, "INSERT INTO users (name, age) VALUES (?, ?)", []string{"Bob", "25"})
```

**⚠️ ALWAYS use parameterized queries** with `?` + `[]string` params. Never string-concatenate user input into SQL — there's no SQL injection protection otherwise.

### Querying data

```mfl
// Returns a JSON string: [{"id":1,"name":"Alice","age":30},{"id":2,"name":"Bob","age":25}]
rows_json := sqlite_query(db, "SELECT * FROM users WHERE age > ?", []string{"20"})
```

### Decoding query results

Use `parse()` with a struct slice witness:

```mfl
type User struct {
    id   int
    name string
    age  int
}

rows_json := sqlite_query(db, "SELECT * FROM users")
users := parse(rows_json, []User{})

// Now range over the typed slice
for _, u := range users {
    println(u.name + ": " + str(u.age))
}
```

### Decoding single values with json_get

```mfl
// For pulling one field from a result
val, err := json_get(rows_json, ".[0].name")
// val is the RAW JSON token — a string comes back QUOTED: "\"Alice\""
```

**⚠️** `json_get` returns the raw JSON token. A string value is quoted. Strip quotes or use `parse()` for whole objects.

### Closing

```mfl
// SQLite databases are closed automatically when the program exits
// There's no explicit sqlite_close builtin visible
```

## 3. JSON serialization

### Serialize

```mfl
type Point struct { x int  y int }

p := Point{x: 10, y: 20}
json_str := json(p)
// json_str = '{"x":10,"y":20}'
```

### Deserialize

```mfl
type Point struct { x int  y int }

data := `{"x":10,"y":20}`
p := parse(data, Point{})
println(str(p.x))  // 10

// Arrays
points := parse(`[{"x":1,"y":2},{"x":3,"y":4}]`, []Point{})
```

### JSON path queries

```mfl
data := `{"user":{"name":"Alice","age":30},"tags":["admin","user"]}`

// Multi-assign only!
name, err := json_get(data, ".user.name")
if len(err) == 0 {
    // name is RAW JSON — "\"Alice\"" (with quotes)
    // Strip them or use parse()
}

// Array index
first_tag, err := json_get(data, ".tags[0]")
```

**⚠️** `json_get` returns QUOTED strings. For clean values, prefer `parse()` when you have a target type:

```mfl
type Response struct { user User  tags []string }
resp := parse(data, Response{})
println(resp.user.name)  // clean, unquoted
```

## 4. Pattern: key-value store with SQLite

```mfl
type KV struct { key string  value string }

func kv_set(db, k, v) {
    sqlite_exec(db, "INSERT OR REPLACE INTO kv (key, value) VALUES (?, ?)", []string{k, v})
}

func kv_get(db, k) (v) {
    rows := sqlite_query(db, "SELECT value FROM kv WHERE key = ?", []string{k})
    entries := parse(rows, []KV{})
    if len(entries) > 0 {
        v = entries[0].value
    }
}

func main() {
    db := sqlite_open("kv.db")
    sqlite_exec(db, "CREATE TABLE IF NOT EXISTS kv (key TEXT PRIMARY KEY, value TEXT)")
    kv_set(db, "greeting", "hello")
    v := kv_get(db, "greeting")
    println(v)  // "hello"
}
```

## 5. Binary data

```mfl
// Read binary file
data := read_file_bytes("image.png")

// Convert to hex
hex := to_hex(data)

// Parse hex back to bytes
restored := from_hex(hex)

// Base64 encode/decode binary
b64 := base64_encode_bytes(data)
decoded := base64_decode_bytes(b64)
```

## 6. Reference apps

| App | Storage technique | Link |
|-----|-----------------|------|
| machin-kv | SQLite key-value store | [Source](https://github.com/javimosch/machin-kv) |
| machin-meet | SQLite + HTTP + booking system | [Source](https://github.com/javimosch/machin-meet) |
| machin-serve | File I/O (read_file, list_dir) | [Source](https://github.com/javimosch/machin-serve) |
