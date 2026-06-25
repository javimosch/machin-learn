---
name: mfl-mcp-client
description: "Using machin-mcp as a tool server from any MCP-compatible client (Claude Code, Claude Desktop) or directly via bash from pi or other agents. Covers registration, tool discovery, and direct protocol calls."
---

# Using machin-mcp as a Tool Server

[machin-mcp](https://github.com/javimosch/machin-mcp) is a 31 KB static binary MCP server. It exposes 6 tools over stdio using JSON-RPC 2.0.

## Option 1: Claude Code (native MCP support)

```bash
# Register the server
claude mcp add machin-mcp -- /path/to/machin-mcp

# Verify it's connected
claude mcp get machin-mcp

# Now Claude Code can use all 6 tools in conversations
# It discovers them automatically via tools/list
```

Once registered, Claude Code will automatically:
1. Call `tools/list` on startup to discover available tools
2. Display them in the 🔌 MCP servers panel
3. Call `tools/call` when it decides a tool would help
4. Pass the results back to the LLM

## Option 2: Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "machin-mcp": {
      "command": "/path/to/machin-mcp"
    }
  }
}
```

## Option 3: Direct from any agent (pi, Claude Code CLI, etc.)

Call the MCP server directly via bash. Each request is one JSON line → one JSON line response.

### Initialize session

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | ./mcp
# → {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"machin-mcp","version":"1.0.0"}}}
```

### Discover tools

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | ./mcp
# → lists all 6 tools with their JSON schemas
```

### Call a tool

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"now","arguments":{}}}' | ./mcp
# → {"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"2026-06-25 12:34:56"}]}}
```

```bash
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"read_file","arguments":{"path":"/etc/hostname"}}}' | ./mcp
```

### Multi-request session (persistent)

For multiple requests, pipe them all to one server process:

```bash
{
  echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
  echo '{"jsonrpc":"2.0","method":"notifications/initialized"}'
  echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"now","arguments":{}}}'
  echo '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_dir","arguments":{"path":"."}}}'
} | ./mcp
```

## Available tools

| Tool | What it does | When an agent should use it |
|------|-------------|---------------------------|
| `echo` | Echoes back input | Testing MCP connectivity |
| `now` | Returns current date/time | When the agent needs to know the current time |
| `read_file` | Reads any file | Inspecting logs, configs, source files |
| `write_file` | Writes content to a file | Creating/modifying files with structured content |
| `list_dir` | Lists directory entries | Exploring directory structure |
| `sqlite_query` | Runs SQL queries | Querying SQLite databases, inspecting data |

## Protocol notes for agents

- **JSON must be compact** (single line) — the stdio transport reads one line per message
- **Initialize first** — `initialize` must be the first call; `notifications/initialized` is optional
- **Notifications** (no `id` field) get no response — send them but don't wait for a reply
- **String arguments** — use `json()` in MFL or `json.dumps()` in Python to properly escape strings with newlines, quotes, etc.
- **Error responses** have `"isError":true` in the result — check for this before using the content
- **Unknown methods** return error code `-32601` (MethodNotFound)

## Example: pi using machin-mcp via bash

In a pi session, the agent can call machin-mcp tools using the bash tool:

```bash
# Agent needs the current time
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | /path/to/machin-mcp | tail -1 | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['serverInfo'])"

# Agent needs to read a file
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"read_file","arguments":{"path":"/var/log/syslog"}}}' | /path/to/machin-mcp | tail -1
```
