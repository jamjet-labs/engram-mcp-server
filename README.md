<div align="center">

<h1>Engram MCP Server</h1>

**Durable memory for AI agents — temporal knowledge graph, hybrid retrieval, SQLite or PostgreSQL.**

[![crates.io](https://img.shields.io/crates/v/jamjet-engram-server?label=jamjet-engram-server&style=flat-square&color=f5c518)](https://crates.io/crates/jamjet-engram-server)
[![Docker](https://img.shields.io/badge/ghcr.io-engram--server-f5c518?style=flat-square&logo=docker)](https://github.com/jamjet-labs/jamjet/pkgs/container/engram-server)
[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-engram--server-5865F2?style=flat-square)](https://registry.modelcontextprotocol.io/servers/io.github.jamjet-labs/engram-server)
[![License](https://img.shields.io/badge/license-Apache%202.0-f5c518?style=flat-square)](LICENSE)

[java-ai-memory.dev](https://java-ai-memory.dev) · [Source code](https://github.com/jamjet-labs/jamjet/tree/main/runtime/engram-server) · [JamJet docs](https://docs.jamjet.dev) · [Discord](https://discord.gg/SAYnEj86fr)

</div>

---

Engram is a **durable memory layer for AI agents**. It extracts facts from conversations, stores them in a temporal knowledge graph, and retrieves them with hybrid semantic + keyword search — backed by a single **SQLite** file or a **PostgreSQL** database.

This repo hosts the Glama registry listing. Source code lives in the [main JamJet repo](https://github.com/jamjet-labs/jamjet/tree/main/runtime/engram-server).

## Quickstart — 30 seconds

```bash
# Docker — uses local Ollama by default
docker run --rm -i \
  -v engram-data:/data \
  ghcr.io/jamjet-labs/engram-server:0.5.0
```

Or install from crates.io:

```bash
cargo install jamjet-engram-server
engram serve
```

### Claude Desktop configuration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "engram": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "-v", "engram-data:/data",
        "ghcr.io/jamjet-labs/engram-server:0.5.0"
      ]
    }
  }
}
```

After restart, 11 MCP tools are available to the model.

## MCP Tools (11)

### Memory tools (7)

| Tool | Description |
|------|-------------|
| `memory_add` | Extract and store facts from conversation messages using LLM-powered fact extraction. Side effects: calls the configured LLM to parse facts, then writes them to the knowledge graph. Returns extracted fact IDs. Requires `messages` array and `user_id`. |
| `memory_recall` | Semantic search over stored facts using vector similarity. Read-only, no side effects. Returns ranked facts matching the query, scoped by `user_id` and optional `org_id`. Use this to retrieve relevant context before generating a response. |
| `memory_context` | Assemble a token-budgeted context block for LLM prompts with tier-aware fact selection. Read-only. Returns a formatted string of the most relevant facts, capped at the specified token budget. Use this instead of memory_recall when you need a ready-to-use prompt snippet. |
| `memory_search` | Keyword search over facts using full-text search (SQLite FTS5 / Postgres). Read-only, no side effects. Returns facts matching exact keywords. Use this when you need precise term matching rather than semantic similarity from memory_recall. |
| `memory_forget` | Soft-delete a fact by ID with an optional reason. Side effect: marks the fact as deleted in the knowledge graph (does not physically remove it). Irreversible via this tool. Use when a user asks to remove specific information. |
| `memory_stats` | Get aggregate statistics: total facts, valid (non-deleted) facts, entity count, and relationship count. Read-only, no side effects. Use this to understand the size and health of the memory store. |
| `memory_consolidate` | Run a maintenance cycle over the knowledge graph — decay stale facts, promote high-confidence ones, deduplicate near-duplicates, and summarize clusters. Side effects: modifies fact scores and may merge or archive facts. Run periodically to keep memory accurate. |

### Message store tools (4)

| Tool | Description |
|------|-------------|
| `messages_save` | Save chat messages for a conversation by ID. Side effects: writes messages to the store and optionally triggers fact extraction (controlled by `--extract-on-save`). Use this to persist full conversation history alongside extracted facts. |
| `messages_get` | Retrieve all messages for a conversation by ID. Read-only, no side effects. Returns the ordered message array. Use this to replay or inspect a past conversation. |
| `messages_list` | List all conversation IDs in the message store. Read-only, no side effects. Returns an array of conversation ID strings. Use this to discover what conversations are stored before retrieving with messages_get. |
| `messages_delete` | Delete all messages for a conversation by ID. Side effect: permanently removes the conversation's messages from the store. Irreversible. Does not affect extracted facts — use memory_forget for that. |

All memory tools are scoped by `(org_id, user_id, session_id)` — org is the coarsest, session the finest.

## LLM Providers

**Provider-agnostic.** One binary, set `ENGRAM_LLM_PROVIDER=...` and go:

| Provider | Env value | Notes |
|----------|-----------|-------|
| Ollama | `ollama` (default) | Local, free, no API keys |
| OpenAI-compatible | `openai-compatible` | OpenAI, Azure, Groq, Together, Mistral, DeepSeek, vLLM, LM Studio, ... |
| Anthropic | `anthropic` | Claude via Messages API |
| Google | `google` | Gemini via generateContent |
| Shell command | `command` | Pipe to any external script |
| Mock | `mock` | Deterministic, for tests only |

```bash
# Example: use Groq instead of Ollama
docker run --rm -i \
  -e ENGRAM_LLM_PROVIDER=openai-compatible \
  -e ENGRAM_OPENAI_BASE_URL=https://api.groq.com/openai/v1 \
  -e OPENAI_API_KEY=gsk_... \
  -v engram-data:/data \
  ghcr.io/jamjet-labs/engram-server:0.5.0
```

## Why Engram?

| Problem | Engram's answer |
|---------|-----------------|
| Every agent memory library is Python-first | **Rust core** with native Python, Java, and MCP clients |
| Needs Postgres + Qdrant + Neo4j just to try | **Single SQLite file** (zero infra) or **Postgres** when you need it |
| Conversation history is not knowledge memory | **Fact extraction pipeline** — structured facts from messages |
| Old facts drift and contradict | **Conflict detection + consolidation** — decay, promote, dedup, summarize |
| Memory recall is either semantic OR keyword | **Hybrid retrieval** — vector search + FTS5 in one query |
| MCP support is an afterthought | **MCP-native** — 11 tools exposed by a single binary |
| Can't isolate memory per user or tenant | **First-class scopes** — org / user / session built into every query |

## Client SDKs

| Language | Package | Install |
|----------|---------|---------|
| Python | `jamjet` (includes `EngramClient`) | `pip install jamjet` |
| Java | `dev.jamjet:jamjet-sdk` (includes `EngramClient`) | Maven Central |
| Spring Boot | `dev.jamjet:engram-spring-boot-starter` | Maven Central |
| Rust | `jamjet-engram` (embed directly) | `cargo add jamjet-engram` |

## Related

- [JamJet](https://github.com/jamjet-labs/jamjet) — the full agent-native runtime (parent project)
- [java-ai-memory.dev](https://java-ai-memory.dev) — comparison with Mem0, Zep, LangChain4j, Spring AI, and others
- [Full Engram docs](https://github.com/jamjet-labs/jamjet/tree/main/runtime/engram-server)

## License

Apache 2.0 — see [LICENSE](LICENSE).

---

<div align="center">
  <sub>Part of <a href="https://jamjet.dev">JamJet</a> · Built by <a href="https://github.com/sunilp">Sunil Prakash</a> · © 2026 JamJet Labs</sub>
</div>
