# machin-learn 🧠

**Master Machin (MFL)** — a knowledge base for agents learning the [Machine-First Language](https://github.com/javimosch/machin). Compiled from real-world usage building [machin-pomo](https://github.com/javimosch/machin-pomo), [machin-serve](https://github.com/javimosch/machin-serve), [machin-mcp](https://github.com/javimosch/machin-mcp), [machin-revproxy](https://github.com/javimosch/machin-revproxy), [machin-scan](https://github.com/javimosch/machin-scan), [machin-hexdump](https://github.com/javimosch/machin-hexdump), [machin-qr](https://github.com/javimosch/machin-qr), [machin-gen](https://github.com/javimosch/machin-gen), [machin-top](https://github.com/javimosch/machin-top), [machin-blob](https://github.com/javimosch/machin-blob), and studying the language spec, and building [machin-game-roguelike](https://github.com/javimosch/machin-game-roguelike) (a terminal roguelike) and [machin-game-dungeon](https://github.com/javimosch/machin-game-dungeon) (a raylib-based first-person 3D dungeon crawler prototype) with procedural generation, FOV, monster AI, turn-based combat, leveling, and JSON save/load).

## Skills

| Skill | Covers |
|-------|--------|
| [`mfl-language`](.agents/skills/mfl-language/SKILL.md) | Syntax, types, functions, structs, generics, control flow, concurrency |
| [`mfl-builtins`](.agents/skills/mfl-builtins/SKILL.md) | Complete builtin catalog organized by category, with usage examples |
| [`mfl-gotchas`](.agents/skills/mfl-gotchas/SKILL.md) | Every pitfall, caveat, and "why doesn't this compile" moment |
| [`mfl-terminal`](.agents/skills/mfl-terminal/SKILL.md) | Terminal/TUI programming: raw_mode, read_key, ANSI rendering |
| [`mfl-networking`](.agents/skills/mfl-networking/SKILL.md) | HTTP servers & clients, WebSocket, TLS, request parsing |
| [`mfl-storage`](.agents/skills/mfl-storage/SKILL.md) | File I/O, SQLite, JSON, data persistence patterns |
| [`mfl-ffi`](.agents/skills/mfl-ffi/SKILL.md) | C FFI: extern blocks, cstructs, raw memory, pointer/array ops |
| [`mfl-mcp-client`](.agents/skills/mfl-mcp-client/SKILL.md) | Using machin-mcp as tool server from Claude Code, Claude Desktop, or directly via bash |

## Quick reference

```bash
# Encode loose source → canonical .mfl
machin encode myapp.src > myapp.mfl

# Build → native binary
machin build myapp.mfl -o myapp

# Build with runtime safety checks
machin build myapp.mfl --safe -o myapp

# Run directly
machin run myapp.mfl

# Full feature catalog (one-shot agent knowledge)
machin guide --text

# Build → WebAssembly
machin build myapp.mfl --target wasm -o myapp.wasm
```

## Source

Pulling `machin-learn` as a skill:

```bash
# Agent: load the language skill
# read .agents/skills/mfl-language/SKILL.md

# Agent: learn gotchas before writing code
# read .agents/skills/mfl-gotchas/SKILL.md

# Agent: load relevant domain skill
# read .agents/skills/mfl-terminal/SKILL.md
```
