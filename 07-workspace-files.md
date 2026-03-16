# 7. Workspace Files: How the Claws Pattern Works

## Overview

An agent's identity and instructions are defined in plain markdown files in a workspace directory. The harness reads these files, concatenates them into a system prompt, and passes that string to pi-mono. That's it.

Memory, tools, and skills are handled by external systems (Funes, KAVAK CLIs) and connected separately. The workspace files are purely about **who the agent is and how it behaves**.

## The Files

| File | Purpose | Example |
|------|---------|---------|
| `SOUL.md` | Persona, tone, values, boundaries | "You are Claudia, a personal automotive concierge. Warm, direct, never pushy." |
| `AGENTS.md` | Operating instructions | "When a customer contacts you, check their purchase history first." |
| `IDENTITY.md` | Name, vibe | "Name: Claudia. Creature: concierge." |
| `USER.md` | Who this agent serves | "Andrea Garcia, +525512345678, owns a 2023 Honda CR-V." |
| `HEARTBEAT.md` | What to check on wake-up | "Check if Andrea's insurance renewal is approaching." |

That's 5 files. No TOOLS.md (CLIs are injected separately). No skills/ directory (tools come from the external catalog). No memory/ (Funes handles that).

## How It Works

### OpenClaw's Mechanism (What We're Simplifying)

OpenClaw reads 8+ files, applies boundary checks, caches by inode, trims to token budgets, runs hook overrides, handles front-matter stripping, and builds a complex multi-section system prompt with skill descriptions, channel routing, reaction guidance, reply tags, and more.

That's ~500 lines of workspace handling across 6 source files.

### KavakClaw's Mechanism (~40 Lines)

```typescript
// kavakclaw/src/agent-runner/workspace-loader.ts

import fs from "node:fs";
import path from "node:path";

const WORKSPACE_FILES = ["SOUL.md", "AGENTS.md", "IDENTITY.md", "USER.md", "HEARTBEAT.md"];
const MAX_FILE_CHARS = 50_000;
const MAX_TOTAL_CHARS = 150_000;

export function buildSystemPrompt(workspaceDir: string): string {
  const parts: string[] = [];
  let totalChars = 0;

  for (const name of WORKSPACE_FILES) {
    if (totalChars >= MAX_TOTAL_CHARS) break;

    const filePath = path.join(workspaceDir, name);
    if (!fs.existsSync(filePath)) continue;

    let content = fs.readFileSync(filePath, "utf8").trim();
    if (!content) continue;

    // Trim to budget
    const budget = Math.min(MAX_FILE_CHARS, MAX_TOTAL_CHARS - totalChars);
    if (content.length > budget) {
      content = content.slice(0, budget) + "\n[TRUNCATED]";
    }

    parts.push(`## ${name}\n${content}`);
    totalChars += content.length;
  }

  return parts.join("\n\n---\n\n");
}
```

### Passing to pi-mono

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, registerBuiltInApiProviders } from "@mariozechner/pi-ai";
import { buildSystemPrompt } from "./workspace-loader.js";

registerBuiltInApiProviders();

const agent = new Agent({
  getApiKey: async (provider) => getCredential(provider),
});

agent.state = {
  systemPrompt: buildSystemPrompt("/workspace"),
  model: getModel(config.provider, config.modelId),
  tools: kavakTools,  // Injected from external CLI catalog, not from workspace
  thinkingLevel: "high",
};
```

pi-mono sees one string. It doesn't know about files. It sends the string + conversation history to the LLM, executes tool calls, and returns the response.

## Different Agent = Different Files

### Concierge
```
workspaces/concierge-andrea-mx/
├── SOUL.md       → "You are Claudia, Andrea's personal concierge..."
├── AGENTS.md     → "Check purchase history. Proactively suggest insurance renewal..."
├── IDENTITY.md   → "Name: Claudia"
├── USER.md       → "Andrea García, drives 2023 Honda CR-V, family of 4..."
└── HEARTBEAT.md  → "Check insurance renewal date. Check financing status."
```

### City CEO
```
workspaces/ceo-cuernavaca/
├── SOUL.md       → "You are the GM of Kavak Cuernavaca. You own the P&L..."
├── AGENTS.md     → "Every morning check inventory, pricing, sales velocity..."
├── IDENTITY.md   → "Name: CEO-Cuernavaca"
├── USER.md       → "Reports to Regional VP. 500 cars in inventory."
└── HEARTBEAT.md  → "Check: inventory age, daily sales, reconditioning queue."
```

### Production Manager
```
workspaces/prod-mgr/
├── SOUL.md       → "You manage Kavak's operational infrastructure..."
├── AGENTS.md     → "Monitor systems. Diagnose and fix issues. Open PRs."
├── IDENTITY.md   → "Name: ProdMgr"
├── USER.md       → "Serves: Engineering team. Escalation: Slack #ops-alerts."
└── HEARTBEAT.md  → "Check error rates, queue depths, deployment status."
```

Same 40-line workspace loader. Same pi-mono agent loop. Different files = different agent.

## What Connects Externally

| Concern | Not in workspace | Where it lives |
|---------|-----------------|----------------|
| Memory | Funes graph | Exposed as a tool the agent can query |
| Tools | KAVAK CLI catalog | Injected as pi-mono `AgentTool[]` at boot |
| ATH | Escalation channel | Injected as a tool (send_to_human) |
| Evals | Eval framework | External system that tests the agent |
| Channels | Slack/WhatsApp | Harness routes messages to/from agent |

The workspace files are only about identity and behavior. Everything else plugs in from outside.
