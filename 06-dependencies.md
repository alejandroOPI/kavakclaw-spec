# 6. Dependencies

## Build vs Reuse

| Component | Decision | Source | Rationale |
|-----------|----------|--------|-----------|
| Agent loop | **REUSE** | `@mariozechner/pi-agent-core` | Battle-tested in OpenClaw, 2k lines, MIT. Handles tool calling, streaming, abort, retries. |
| LLM providers | **REUSE** | `@mariozechner/pi-ai` | 10+ providers already implemented. Adding a new provider is one file. |
| Agent Manager | **BUILD** | New | Nothing exists for long-running agent lifecycle at this scale. |
| Scheduler | **BUILD** | New | Variable-frequency heartbeats with agent self-adjustment don't exist elsewhere. |
| Channel Router | **BUILD** | New | Customer-to-agent routing is KAVAK-specific. |
| DockerRuntime | **BUILD** (informed by NanoClaw) | New, using NanoClaw patterns | NanoClaw's is ephemeral; ours is persistent. Same isolation concepts. |
| ProcessRuntime | **BUILD** | New | AWS-specific, nothing to reuse. |
| Credential Proxy | **BUILD** (informed by NanoClaw) | New, using NanoClaw patterns | Extend to multi-provider (NanoClaw is Anthropic-only). |
| Mount Security | **REUSE concepts** | Informed by NanoClaw | Blocked patterns, symlink resolution, path validation. |
| Self-improvement tools | **BUILD** | New | Novel capability. |
| Filesystem convention | **REUSE** | OpenClaw's Claws pattern | SOUL.md, AGENTS.md, skills/ -- proven, well-documented. |
| SQLite | **REUSE** | `better-sqlite3` | Local state storage. Standard. |
| Docker API | **REUSE** | `dockerode` | Node.js Docker client. |

## npm Dependencies

### Production

```json
{
  "dependencies": {
    "@mariozechner/pi-agent-core": "^0.58.0",
    "@mariozechner/pi-ai": "^0.58.0",
    "@sinclair/typebox": "^0.34.0",
    "better-sqlite3": "^11.0.0",
    "dockerode": "^4.0.0",
    "pino": "^9.0.0",
    "croner": "^9.0.0"
  }
}
```

### Why Each Dependency

| Package | Why | Size | Alternative Considered |
|---------|-----|------|----------------------|
| `pi-agent-core` | The agent loop. Tool calling, streaming, state management. | ~2k lines | Writing our own (rejected: pi-mono is battle-tested) |
| `pi-ai` | Multi-vendor LLM abstraction. All providers. | ~5k lines | Vercel AI SDK (rejected: pi-mono already integrates with our agent loop) |
| `typebox` | JSON Schema for tool parameter validation. Used by pi-mono. | Tiny | Zod (rejected: pi-mono uses typebox) |
| `better-sqlite3` | Agent configs, bindings, crons, audit. Fast, embedded. | Native addon | PostgreSQL (overkill for single-instance), DynamoDB (for multi-instance later) |
| `dockerode` | Docker API for DockerRuntime. Spawn/manage containers. | Small | Shell exec (rejected: dockerode is cleaner) |
| `pino` | Structured logging. | Tiny | winston (rejected: pino is faster) |
| `croner` | Cron expression parsing and scheduling. | Tiny | node-cron (rejected: croner is more modern) |

**Total production dependencies: 7**

Compare: OpenClaw has 70+ dependencies. NanoClaw has ~30.

## pi-mono Deep Dive

### What We Use

**`@mariozechner/pi-agent-core`** (packages/agent/):

```
agent-loop.ts (682 lines)
  - runAgentLoop(): The main loop. Takes messages, tools, model. Returns response.
  - Handles: streaming, tool calls (parallel/sequential), abort signals, errors
  - Provider-agnostic: works with any model from pi-ai

agent.ts (612 lines)
  - Agent class: Wraps agent-loop with state management
  - agent.prompt(text): Send a message, get a response
  - agent.steer(text): Inject a message during a running turn
  - agent.state: System prompt, model, tools, messages

types.ts (310 lines)
  - AgentTool: { name, label, description, schema, execute }
  - AgentContext: { systemPrompt, messages, tools }
  - AgentEvent: Lifecycle events (turn_start, tool_call, turn_end, etc.)
  - StreamFn: Provider-agnostic streaming function type

proxy.ts (340 lines)
  - Transport proxy for remote agent execution
```

**`@mariozechner/pi-ai`** (packages/ai/):

```
api-registry.ts
  - registerApiProvider(): Register a new provider
  - getApiProvider(): Look up a provider by API type

providers/
  - anthropic.ts: Claude models via Anthropic API
  - openai-responses.ts: GPT models via OpenAI Responses API
  - openai-completions.ts: GPT models via OpenAI Completions API
  - openai-codex-responses.ts: Codex models via ChatGPT subscription
  - google.ts: Gemini models via Google AI
  - google-vertex.ts: Gemini via Vertex AI
  - google-gemini-cli.ts: Gemini via CLI OAuth
  - azure-openai-responses.ts: Azure OpenAI
  - amazon-bedrock.ts: Bedrock (lazy-loaded)
  - mistral.ts: Mistral models
  - register-builtins.ts: Registers all providers at startup

types.ts (336 lines)
  - Model: { provider, id, api, baseUrl, ... }
  - StreamOptions: { temperature, maxTokens, apiKey, transport, ... }
  - Provider/Api type unions for all known providers

stream.ts
  - streamSimple(): Unified streaming function used by agent-loop
  - Handles provider-specific quirks transparently

models.ts
  - getModel(provider, id): Look up a model
  - Model catalog with pricing, context sizes, capabilities
```

### What We DON'T Use

- `@mariozechner/pi-coding-agent` -- Coding-specific tools (file read/write/edit, exec, git). We build our own tools.
- `packages/tui` -- Terminal UI. Not needed.
- `packages/web-ui` -- Web interface. Not needed.
- `packages/mom` -- Agent management UI. Not needed.
- `packages/pods` -- vLLM deployment. Not needed.

### Version Pinning Strategy

- Pin to specific minor version (e.g., `0.58.x`)
- Monitor pi-mono releases monthly
- Test upgrades in staging before production
- pi-mono is MIT -- we can fork if upstream breaks compatibility

## What We Don't Depend On

| NOT a dependency | Why |
|-----------------|-----|
| OpenClaw | 500k LOC, too coupled. We use its conventions, not its code. |
| NanoClaw | Anthropic-locked, ephemeral model. We use its security patterns, not its code. |
| ZeroClaw / PicoClaw | Nothing we need. |
| LangChain / LlamaIndex | Over-abstraction. pi-mono is simpler and more direct. |
| Vercel AI SDK | Redundant with pi-mono. Would add another provider abstraction layer. |
| Express / Fastify | No HTTP server needed (channels handle their own connections). |
| Redis / RabbitMQ | Not needed for single-instance. Add for multi-instance scaling later. |

## Estimated Codebase Size

| File | Lines (est.) | Purpose |
|------|-------------|---------|
| `src/agent-manager.ts` | ~400 | Lifecycle management |
| `src/scheduler.ts` | ~300 | Heartbeats + crons |
| `src/channel-router.ts` | ~200 | Message routing |
| `src/runtime/types.ts` | ~50 | Runtime interface |
| `src/runtime/docker.ts` | ~250 | Docker container management |
| `src/runtime/process.ts` | ~200 | Subprocess management |
| `src/security/credential-proxy.ts` | ~200 | Multi-provider credential injection |
| `src/security/mount-security.ts` | ~100 | Filesystem allowlists |
| `src/tools/self-improve.ts` | ~150 | Workspace modification tools |
| `src/tools/heartbeat.ts` | ~50 | Heartbeat self-adjustment tool |
| `src/tools/channel.ts` | ~50 | Message sending tool |
| `src/agent-runner/index.ts` | ~150 | Agent process entry point |
| `src/agent-runner/workspace-loader.ts` | ~100 | Load workspace files into system prompt |
| `src/agent-runner/ipc.ts` | ~100 | IPC transport (filesystem + stdio) |
| `src/index.ts` | ~100 | Harness entry point |
| **Total** | **~2,400** | |

Plus ~200 lines of config types, ~200 lines of tests infrastructure. Under 3,000 lines total.

For comparison:
- OpenClaw: ~500,000 lines
- NanoClaw: ~5,000 lines
- pi-mono (what we depend on): ~10,000 lines

KavakClaw: **~3,000 lines of new code + ~10,000 lines of pi-mono dependency = ~13,000 lines total.** Fully auditable by a security team in a day.
