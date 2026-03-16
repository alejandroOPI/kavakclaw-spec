# 2. Competitive Landscape

## Why Not Use an Existing Project?

We evaluated every major open-source agentic assistant framework. None satisfies our requirements: multi-vendor model support, container isolation, long-running agents, dual deployment (AWS + Mac mini), and a codebase small enough to audit.

## The Four "Claws"

### OpenClaw
- **Repo:** github.com/openclaw/openclaw (MIT, 317k stars)
- **Language:** TypeScript
- **Architecture:** Single Node.js Gateway process. Embedded "Pi agent runtime" derived from pi-mono. Plugin system for channels (28+ supported), tools, and providers. Skill system with bundled/managed/workspace skills. Multi-agent routing via config.
- **Agent Harness:** Agent runs inside the Gateway process. Tools (exec, browser, canvas, cron, sessions) are first-class. Sub-agent isolation via `sessions_spawn`. JSONL session transcripts. The "Claws" filesystem pattern (SOUL.md, AGENTS.md, skills/) originated here.
- **Provider Support:** 30+ providers via pi-mono + plugin system. OpenAI, Anthropic, Google, Mistral, Bedrock, Azure, OpenRouter, Codex, Copilot, ZAI, and more. Provider plugins own auth, catalog, and runtime behavior.
- **Security:** Application-level only. Allowlists, pairing codes, DM policies. Single trusted operator boundary (personal assistant model, NOT multi-tenant). Everything runs in one Node process with shared memory. No OS-level sandboxing by default. Docker sandboxing is opt-in.
- **Deployment:** macOS (launchd), Linux (systemd), Docker (optional). Single-user model.
- **Strengths:** Most mature ecosystem. Best channel coverage. Battle-tested provider layer. Excellent filesystem convention we adopt.
- **Weaknesses:** 500k+ LOC -- unauditable. No native container isolation. Personal assistant model, not designed for thousands of concurrent agents. Pi agent runtime is deeply embedded, not extractable.
- **Verdict:** Too large, too coupled, wrong isolation model. But we adopt its filesystem convention and its underlying LLM engine (pi-mono).

### NanoClaw
- **Repo:** github.com/qwibitai/nanoclaw (MIT, 23.4k stars)
- **Language:** TypeScript
- **Architecture:** Single Node.js orchestrator + containerized agent execution. SQLite for message storage. Channels self-register at startup via factory pattern. Per-group message queues with concurrency control. IPC via filesystem.
- **Agent Harness:** Anthropic Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`). Each message spawns a fresh Linux container (Docker or Apple Container). Agent gets Bash, file ops, web search, browser automation, MCP tools. Per-group isolated CLAUDE.md memory.
- **Provider Support:** Anthropic only. Locked to Claude Agent SDK. Supports API-compatible endpoints but fundamentally single-vendor.
- **Security:** Best of all four. Hypervisor-level container isolation. Credential proxy (real API keys never enter containers). Mount allowlist stored outside project root. Default blocked patterns (.ssh, .gnupg, .aws, etc.). Symlink resolution prevents traversal. Non-root execution. Read-only project root mount. Per-group session isolation.
- **Deployment:** macOS launchd only. No cloud deployment story.
- **Strengths:** Best security model. Small auditable codebase (~10 files). Clean container architecture. Credential proxy pattern is excellent.
- **Weaknesses:** Single-vendor (Anthropic only). Ephemeral containers per message (not long-running). No cloud deployment. No heartbeats or crons. Personal assistant model only.
- **Verdict:** Wrong agent model (ephemeral vs long-running), wrong vendor model (Anthropic-only). But we adopt its security patterns: credential proxy, mount security, container isolation concept.

### PicoClaw
- **Repo:** github.com/sipeed/picoclaw (MIT, 25k stars)
- **Language:** Go
- **Architecture:** Single Go binary, ultra-lightweight (<10MB RAM, 1s startup). Config-file driven (JSON). Gateway serves all webhook channels on one HTTP port. Targets $10 hardware (RISC-V, ARM, MIPS, x86).
- **Agent Harness:** LiteLLM-compatible agent loop. Any OpenAI-compatible API. model_list config for zero-code provider addition. Tools: web search, file ops.
- **Provider Support:** Any OpenAI-compatible endpoint via LiteLLM pattern. OpenRouter, Anthropic, OpenAI, local models via Ollama.
- **Security:** Early development. allow_from per channel. Web console has NO authentication. No sandboxing. Explicitly warns "may have unresolved network security issues."
- **Deployment:** Docker Compose, bare metal, Termux on old phones. Targets IoT devices.
- **Strengths:** Tiny footprint. Multi-vendor. Great for hardware/IoT.
- **Weaknesses:** Explicitly not production-ready. No sandboxing. No container isolation. Immature security.
- **Verdict:** Wrong maturity level. Interesting for IoT use cases but not enterprise.

### ZeroClaw
- **Repo:** github.com/zeroclaw-labs/zeroclaw (MIT, 27.4k stars)
- **Language:** Rust
- **Architecture:** Single Rust binary (<5MB RAM, <10ms startup). Trait-driven architecture -- everything is swappable (providers, channels, tools, memory, tunnels). Config via TOML. Daemon mode for autonomous runtime.
- **Agent Harness:** Provider-agnostic agent loop with trait-based tool execution. Supports Claude Code OAuth and OpenAI Codex OAuth. Auth profiles with encrypted-at-rest credential storage.
- **Provider Support:** Multi-vendor via trait system. OpenAI, Anthropic, OpenRouter, local models. OpenAI-compatible endpoints.
- **Security:** Application-layer only (allowlists, path blocking). Sandboxing is PROPOSED/ROADMAP only. Proposal docs exist for Firejail, Bubblewrap, Docker, Landlock -- none are implemented. Auth profile encryption at rest is a plus.
- **Deployment:** Bare metal, Raspberry Pi, systemd/OpenRC. Docker support basic.
- **Strengths:** Excellent trait-based architecture. Smallest runtime footprint. Best-designed extension model. Good documentation.
- **Weaknesses:** Sandboxing is roadmap-only. Rust makes rapid iteration harder. Impersonation/scam issues around the project.
- **Verdict:** Beautiful architecture but missing security implementation. Rust adds development friction for a team that needs to move fast.

## The Underlying Engine: pi-mono

All four projects reference or depend on work from the same source: **pi-mono** (github.com/badlogic/pi-mono, MIT, 24.5k stars).

pi-mono is the open-source monorepo that powers OpenClaw's agent runtime. It contains:

| Package | What It Is | Lines |
|---------|-----------|-------|
| `@mariozechner/pi-agent-core` | Agent loop with tool calling, state management, streaming | ~2,000 |
| `@mariozechner/pi-ai` | Multi-provider LLM abstraction (Anthropic, OpenAI, Google, Mistral, Bedrock, Azure, Codex, Copilot, etc.) | ~5,000 |
| `@mariozechner/pi-coding-agent` | Coding-specific agent with file operations, exec, git | ~3,000 |

The agent loop (`agent-loop.ts`, 682 lines) is provider-agnostic. It takes:
- A model (from any provider)
- A system prompt
- A set of tools
- Message history

And runs the observe-act-verify loop until the model produces a final response. Tool calls are executed in parallel or sequentially. The loop handles streaming, error recovery, abort signals, and context management.

The provider layer (`pi-ai`) uses a registry pattern:

```typescript
registerApiProvider({ api: "anthropic-messages", stream: streamAnthropic, ... });
registerApiProvider({ api: "openai-responses", stream: streamOpenAI, ... });
registerApiProvider({ api: "google-generative-ai", stream: streamGoogle, ... });
// 10+ providers registered at startup
```

Adding a new provider means implementing a `stream` and `streamSimple` function and registering it. No changes to the agent loop.

**This is what we use as our model gateway.** We don't fork it. We depend on it.

## Summary Matrix

| Requirement | OpenClaw | NanoClaw | PicoClaw | ZeroClaw | KavakClaw |
|---|---|---|---|---|---|
| Multi-vendor LLM | Yes (30+) | No (Anthropic) | Yes (LiteLLM) | Yes (traits) | **Yes (pi-mono)** |
| Container isolation | No | Yes (best) | No | No (roadmap) | **Yes** |
| Long-running agents | No (session-based) | No (ephemeral) | No | No | **Yes** |
| Dual deploy (AWS + local) | Docker optional | macOS only | Docker/bare | Bare metal | **Yes** |
| Auditable codebase | No (500k LOC) | Yes (~10 files) | Yes but immature | Medium (Rust) | **Yes (~2k lines)** |
| Self-improving agents | No | No | No | No | **Yes** |
| Variable heartbeats | Fixed interval | None | None | None | **Yes** |
| Millions of concurrent agents | No | No | No | No | **Designed for it** |

## What We Take From Each

| Source | What We Adopt |
|--------|-------------|
| **OpenClaw** | Filesystem convention (SOUL.md, AGENTS.md, skills/), heartbeat/cron concept, pi-mono dependency |
| **NanoClaw** | Container isolation pattern, credential proxy concept, mount security concept |
| **PicoClaw** | Nothing (too immature) |
| **ZeroClaw** | Nothing directly (trait architecture is inspiring but we use TypeScript) |
| **pi-mono** | Agent loop (`pi-agent-core`), all LLM providers (`pi-ai`) -- used as npm dependency |
