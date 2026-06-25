---
name: mfl-networking
description: "MFL networking builtins: building HTTP servers and clients, WebSocket, request parsing, Basic auth, MIME types, concurrent request handling with goroutines. Patterns from machin-serve."
---

# MFL Networking

Build network services using machin's `listen`/`accept`/`read`/`write`/`close` builtins, HTTP clients, and WebSocket.

## 1. Server socket model

```mfl
server := listen(8080)    // create listener on 0.0.0.0:8080
for {
    conn := accept(server)  // block until connection arrives
    go handle(conn)         // handle concurrently in a goroutine
}

func handle(conn) {
    defer close(conn)  // always close the connection
    req := read(conn)  // read HTTP request
    // ... process ...
    write(conn, response)
    close(conn)
}
```

**Key points:**
- `listen(port)` binds to `0.0.0.0` by default
- `accept(server)` returns a connection file descriptor (int)
- `read(conn)` blocks until data arrives, returns the request as string
- `write(conn, data)` sends data
- `close(conn)` closes the connection
- Use `go handle(conn)` to handle requests concurrently

## 2. HTTP request parsing

Parse the request line to extract the method and path:

```mfl
req := read(conn)

// "GET /path HTTP/1.1\r\nHeader: value\r\n\r\n"

path := "/"
m := regex_groups(req, "^GET ([^ ]+) HTTP")
if len(m) > 1 { path = m[1] }   // m[0]=full match, m[1]=capture

// URL decode
path = url_decode(path)
```

**⚠️ `regex_groups` indexing:** index 0 is the full match, index 1+ are capture groups.

## 3. HTTP response building

```mfl
func ok(body, ctype) (s) {
    s = "HTTP/1.1 200 OK\r\nContent-Type: " + ctype +
        "\r\nContent-Length: " + str(len(body)) +
        "\r\nConnection: close\r\n\r\n" + body
}

func not_found() (s) {
    body := "404 Not Found"
    s = "HTTP/1.1 404 Not Found\r\nContent-Type: text/plain" +
        "\r\nContent-Length: " + str(len(body)) +
        "\r\nConnection: close\r\n\r\n" + body
}

func unauthorized() (s) {
    body := "401 Unauthorized"
    s = "HTTP/1.1 401 Unauthorized\r\nWWW-Authenticate: Basic realm=\"machin-serve\"" +
        "\r\nContent-Type: text/plain" +
        "\r\nContent-Length: " + str(len(body)) +
        "\r\nConnection: close\r\n\r\n" + body
}
```

## 4. Handling GET requests

```mfl
func handle(conn) {
    req := read(conn)

    // Parse path
    path := "/"
    m := regex_groups(req, "^GET ([^ ]+) HTTP")
    if len(m) > 1 { path = m[1] }

    // Prevent path traversal
    if contains(path, "..") {
        write(conn, not_found())
        close(conn)
        return
    }

    // URL decode
    path = url_decode(path)

    // Determine content type from file extension
    ctype := mime_type(path)

    // Read and serve
    body := read_file("." + path)
    if len(body) == 0 {
        write(conn, not_found())
    } else {
        write(conn, ok(body, ctype))
    }
    close(conn)
}
```

## 5. Basic auth pattern

```mfl
func handle(conn, auth) {
    req := read(conn)

    if len(auth) > 0 {
        am := regex_groups(req, "Authorization: Basic ([^\r\n]+)")
        authed := false
        if len(am) > 1 {
            decoded := base64_decode(am[1])  // decodes "user:pass"
            if decoded == auth { authed = true }
        }
        if authed == false {
            write(conn, unauthorized())
            close(conn)
            return
        }
    }

    // ... serve file ...
}
```

## 6. MIME type detection

```mfl
func mime_type(path) (t) {
    if has_suffix(path, ".html") || has_suffix(path, ".htm") { return "text/html; charset=utf-8" }
    if has_suffix(path, ".css") { return "text/css; charset=utf-8" }
    if has_suffix(path, ".js") { return "application/javascript" }
    if has_suffix(path, ".json") { return "application/json" }
    if has_suffix(path, ".png") { return "image/png" }
    if has_suffix(path, ".jpg") || has_suffix(path, ".jpeg") { return "image/jpeg" }
    if has_suffix(path, ".gif") { return "image/gif" }
    if has_suffix(path, ".webp") { return "image/webp" }
    if has_suffix(path, ".svg") { return "image/svg+xml" }
    if has_suffix(path, ".ico") { return "image/x-icon" }
    if has_suffix(path, ".txt") { return "text/plain; charset=utf-8" }
    if has_suffix(path, ".pdf") { return "application/pdf" }
    if has_suffix(path, ".zip") { return "application/zip" }
    if has_suffix(path, ".md") { return "text/markdown; charset=utf-8" }
    if has_suffix(path, ".wasm") { return "application/wasm" }
    return "application/octet-stream"
}
```

## 7. Directory index pattern

```mfl
// Check if path is a directory using list_dir
entries := list_dir(fpath)
if len(entries) > 0 {
    // It's a directory — serve index.html
    if charat(fpath, len(fpath)-1) != "/" {
        fpath = fpath + "/"
    }
    fpath = fpath + "index.html"
    body := read_file(fpath)
    if len(body) > 0 {
        write(conn, ok(body, mime_type(fpath)))
    } else {
        write(conn, not_found())
    }
    close(conn)
    return
}

// Otherwise serve the file
body := read_file(fpath)
```

**⚠️ Always check directories with `list_dir` BEFORE `read_file`** — `read_file` on a directory causes SEGFAULT.

## 8. HTTP client

```mfl
// Simple GET
body := https_get("https://example.com")

// GET with error handling
status, body, err := http_get("https://api.github.com/repos/javimosch/machin")
if len(err) > 0 {
    println("unreachable: " + err)
    exit(1)
}
println("status: " + str(status))
println("body length: " + str(len(body)))

// POST JSON
resp := https_post("https://httpbin.org/post", json(some_data))

// Auth'd request (http_request is multi-assign)
headers := []string{"Authorization: Bearer " + token}
status, body, err := http_request("GET", "https://api.example.com/data", headers, "")
```

**⚠️** `http_get` and `http_request` return multiple values — must use multi-assign:

```mfl
// WRONG
body := http_get("https://example.com")

// RIGHT
status, body, err := http_get("https://example.com")
```

## 9. WebSocket client

```mfl
ws := wss_open("wss://echo.websocket.org")
if ws > 0 {
    wss_send(ws, "hello")
    reply := wss_recv(ws)  // blocks
    println("got: " + reply)
    wss_send(ws, "close")
    close(ws)  // or use wss_close?
}
```

## 10. Full server skeleton

```mfl
func ok(body, ctype) (s) {
    s = "HTTP/1.1 200 OK\r\nContent-Type: " + ctype +
        "\r\nContent-Length: " + str(len(body)) +
        "\r\nConnection: close\r\n\r\n" + body
}

func handle(conn) {
    req := read(conn)
    println("got request")  // will be buffered — call flush()!
    // parse, serve, respond
    write(conn, ok("<h1>hello</h1>", "text/html; charset=utf-8"))
    close(conn)
}

func main() {
    server := listen(8080)
    println("listening on http://0.0.0.0:8080/")
    for {
        conn := accept(server)
        go handle(conn)
    }
}
```

## 11. JSON-RPC / MCP server pattern (stdio transport)

The Model Context Protocol uses JSON-RPC 2.0 over stdio — one JSON message per line on stdin/stdout.

### Core loop

```mfl
func main() {
    running := true
    while running {
        line := input()                 // read one line from stdin
        if len(line) == 0 { break }    // EOF

        method_raw, merr := json_get(line, ".method")
        if len(merr) > 0 { continue }  // invalid JSON
        method := strip_quotes(method_raw)

        id_raw, ierr := json_get(line, ".id")
        is_notification := len(ierr) > 0

        if method == "initialize" && is_notification == false {
            println(handle_initialize(id_raw))
        } else if method == "tools/list" && is_notification == false {
            println(handle_list_tools(id_raw))
        } else if method == "tools/call" && is_notification == false {
            println(handle_call_tool(line, id_raw))
        } else if is_notification == false {
            // MethodNotFound error
            println("{\"jsonrpc\":\"2.0\",\"id\":" + id_raw + ",\"error\":{\"code\":-32601,\"message\":\"Method not found\"}}")
        }
        flush()
    }
}
```

### Key JSON-RPC patterns

| Pattern | Code |
|---------|------|
| Extract method | `method_raw, _ := json_get(line, \".method\")` → `strip_quotes(method_raw)` |
| Detect notification | `id_raw, ierr := json_get(line, \".id\")` → `len(ierr) > 0` |
| Safe string encoding | `escaped := json(text)` — handles quotes, newlines, etc. |
| JSON unescape | `replace(s, \"\\\\\", \"\\\\\")`, then `\"\\n\" → \"\\n\"` etc. |

### ⚠️ `json_get` vs `parse` for escape handling

`json_get` returns RAW JSON tokens **preserving escape sequences** — `\\n` stays as literal backslash-n. `parse()` (with struct types) interprets escapes correctly. Use a `json_unescape` helper when extracting string values with `json_get`:

```mfl
func json_unescape(s) (out) {
    out = replace(s, "\\\\", "\\")
    out = replace(out, "\\\"", "\"")
    out = replace(out, "\\n", "\n")
    out = replace(out, "\\r", "\r")
    out = replace(out, "\\t", "\t")
}
```

### Real app reference

| App | Features | Link |
|-----|----------|------|
| machin-mcp | Full MCP server, JSON-RPC 2.0, 6 tools, stdio transport | [Source](https://github.com/javimosch/machin-mcp) |
| machin-serve | Static file server, Basic auth, MIME types, directory index, traversal protection | [Source](https://github.com/javimosch/machin-serve) |
| machin-fetch | HTTPS CLI client | [Source](https://github.com/javimosch/machin-fetch) |
| machin-http | Multi-command HTTPS client with flags | [Source](https://github.com/javimosch/machin-http) |

| App | Features | Link |
|-----|----------|------|
| machin-serve | Static file server, Basic auth, MIME types, directory index, traversal protection | [Source](https://github.com/javimosch/machin-serve) |
| machin-fetch | HTTPS CLI client | [Source](https://github.com/javimosch/machin-fetch) |
| machin-http | Multi-command HTTPS client with flags | [Source](https://github.com/javimosch/machin-http) |
