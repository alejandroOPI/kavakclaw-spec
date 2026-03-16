# 7. Workspace Files: How the Claws Pattern Works

## Overview

The "Claws" pattern is a filesystem convention where an agent's identity, instructions, knowledge, and behavior are defined in plain markdown files. The harness reads these files, concatenates them into a system prompt, and passes them to the LLM. The agent can also modify these files at runtime (self-improvement).

This document traces the exact mechanism from OpenClaw's codebase and shows how KavakClaw implements the same pattern in ~100 lines.

## The File Inventory

| File | Purpose | Editable by Agent? |
|------|---------|-------------------|
| `SOUL.md` | Persona, tone, boundaries, values. Who the agent IS. | With approval |
| `AGENTS.md` | Operating instructions. What to do on startup, how to behave, memory conventions, safety rules. | Yes |
| `IDENTITY.md` | Name, creature type, vibe. Short identity card. | With approval |
| `USER.md` | Profile of the human/customer the agent serves. Preferences, contact info, context. | With approval |
| `TOOLS.md` | Tool-specific notes. Camera names, SSH hosts, API keys locations, CLI usage notes. | Yes |
| `HEARTBEAT.md` | What to check on each heartbeat wake-up. Email? Calendar? Tasks? | Yes |
| `BOOTSTRAP.md` | One-time first-run ritual. Deleted after completion. | N/A (deleted) |
| `MEMORY.md` | Long-term facts, preferences, open loops. Durable knowledge. | Yes |
| `skills/` | Directory of tool definitions. Each skill has a `SKILL.md` describing when/how to use it. | Yes (can create new skills) |
| `memory/` | Journals, topics, bridge files. Structured memory. | Yes |

## How OpenClaw Processes These Files

### Step 1: Load Files from Disk

```typescript
// OpenClaw: src/agents/workspace.ts
export async function loadWorkspaceBootstrapFiles(dir: string): Promise<WorkspaceBootstrapFile[]> {
  const entries = [
    { name: "AGENTS.md",    filePath: path.join(dir, "AGENTS.md") },
    { name: "SOUL.md",      filePath: path.join(dir, "SOUL.md") },
    { name: "TOOLS.md",     filePath: path.join(dir, "TOOLS.md") },
    { name: "IDENTITY.md",  filePath: path.join(dir, "IDENTITY.md") },
    { name: "USER.md",      filePath: path.join(dir, "USER.md") },
    { name: "HEARTBEAT.md", filePath: path.join(dir, "HEARTBEAT.md") },
    { name: "BOOTSTRAP.md", filePath: path.join(dir, "BOOTSTRAP.md") },
    // MEMORY.md or memory.md (auto-detected)
  ];

  // Each file is read with boundary checks (no symlink escapes, size limits)
  // Missing files get { missing: true } instead of failing
  // Result: WorkspaceBootstrapFile[] with { name, path, content, missing }
}
```

Key details:
- Files are read with symlink resolution (prevents path traversal attacks)
- Maximum file size: 2MB per file
- Missing files are noted but don't cause errors
- Files are cached by inode/mtime to avoid re-reading unchanged files

### Step 2: Trim to Token Budget

```typescript
// OpenClaw: src/agents/pi-embedded-helpers/bootstrap.ts
export function buildBootstrapContextFiles(
  files: WorkspaceBootstrapFile[],
  opts?: { maxChars?: number; totalMaxChars?: number }
): EmbeddedContextFile[] {
  // Default: 50,000 chars per file, 200,000 chars total
  // Files are processed in order (AGENTS.md first, MEMORY.md last)
  // If a file exceeds the budget, it's truncated with a warning marker
  // If total budget is exhausted, remaining files are skipped
}
```

This is important for KavakClaw: the concierge agent for a customer with 5 years of interaction history might have a huge MEMORY.md. The budget system prevents context overflow.

### Step 3: Build the System Prompt

```typescript
// OpenClaw: src/agents/system-prompt.ts
export function buildAgentSystemPrompt(params) {
  const lines = [];

  // 1. Hardcoded harness instructions
  lines.push("You are Claude Code, Anthropic's official CLI for Claude.");
  lines.push("## Tooling", ...toolingSection);
  lines.push("## Skills (mandatory)", ...skillsSection);

  // 2. Workspace files section
  lines.push("## Workspace Files (injected)");
  lines.push("These user-editable files are loaded and included below.");

  // 3. Runtime metadata
  lines.push("## Runtime", runtimeLine);  // model, channel, OS, etc.

  // 4. Project Context (THE ACTUAL FILE CONTENTS)
  lines.push("# Project Context");
  lines.push("If SOUL.md is present, embody its persona and tone.");
  for (const file of contextFiles) {
    lines.push(`## ${file.path}`);  // e.g., "## /Users/clawdius/clawd/SOUL.md"
    lines.push(file.content);        // The actual file content
  }

  // 5. Behavioral rules
  lines.push("## Silent Replies", ...silentReplyRules);
  lines.push("## Heartbeats", ...heartbeatRules);

  return lines.join("\n");
}
```

The resulting system prompt is a single massive string (~10-50KB) that contains everything the agent needs to know about itself.

### Step 4: Pass to pi-mono

```typescript
// OpenClaw: src/agents/pi-embedded-runner/run/attempt.ts
// The system prompt string goes directly to pi-mono's Agent class

const systemPromptOverride = createSystemPromptOverride(builtSystemPrompt);

// pi-mono receives it as agent.state.systemPrompt
// pi-mono then prepends it to every LLM call as the system message
// The LLM (Claude, GPT, Gemini, etc.) reads it and follows the instructions
```

### What pi-mono Sees

pi-mono has NO knowledge of workspace files. It receives:

```typescript
agent.state = {
  systemPrompt: "You are Claude Code...\n## SOUL.md\n...\n## AGENTS.md\n...",
  model: getModel("anthropic", "claude-opus-4-6"),
  tools: [readTool, writeTool, execTool, editTool, ...],
  thinkingLevel: "high",
};
```

pi-mono's `agentLoop()` function:

```
1. Prepends systemPrompt as the system message
2. Appends conversation history (user/assistant/toolResult messages)
3. Sends to LLM via the registered provider (Anthropic, OpenAI, etc.)
4. If LLM returns tool calls:
   a. Validates arguments against tool schema
   b. Executes each tool (parallel or sequential)
   c. Appends tool results to conversation
   d. Loops back to step 3
5. If LLM returns text response: returns it
```

The agent loop is 682 lines. The provider registry is ~100 lines. Everything else is the harness above it.

## KavakClaw Implementation

### The Equivalent in ~100 Lines

```typescript
// kavakclaw/src/agent-runner/workspace-loader.ts

import fs from "node:fs";
import path from "node:path";

const WORKSPACE_FILES = [
  "SOUL.md",
  "AGENTS.md", 
  "IDENTITY.md",
  "USER.md",
  "TOOLS.md",
  "HEARTBEAT.md",
  "MEMORY.md",
];

const MAX_FILE_CHARS = 50_000;
const MAX_TOTAL_CHARS = 200_000;

interface WorkspaceFile {
  name: string;
  content: string;
  truncated: boolean;
}

export function loadWorkspace(workspaceDir: string): WorkspaceFile[] {
  const files: WorkspaceFile[] = [];
  let totalChars = 0;

  for (const name of WORKSPACE_FILES) {
    if (totalChars >= MAX_TOTAL_CHARS) break;
    
    const filePath = path.join(workspaceDir, name);
    if (!fs.existsSync(filePath)) continue;
    
    // Security: resolve symlinks and verify path stays within workspace
    const resolved = fs.realpathSync(filePath);
    const resolvedWorkspace = fs.realpathSync(workspaceDir);
    if (!resolved.startsWith(resolvedWorkspace)) {
      console.warn(`Skipping ${name}: resolves outside workspace`);
      continue;
    }

    let content = fs.readFileSync(filePath, "utf8");
    const originalLength = content.length;
    
    // Trim to budget
    const budget = Math.min(MAX_FILE_CHARS, MAX_TOTAL_CHARS - totalChars);
    let truncated = false;
    if (content.length > budget) {
      content = content.slice(0, budget) + "\n\n[TRUNCATED — read full file for complete content]";
      truncated = true;
    }

    totalChars += content.length;
    files.push({ name, content, truncated });
  }

  return files;
}

export function buildSystemPrompt(
  workspaceDir: string,
  extraInstructions?: string
): string {
  const files = loadWorkspace(workspaceDir);
  const parts: string[] = [];

  // Harness identity
  parts.push("You are a KavakClaw agent. Follow your SOUL.md persona strictly.");
  parts.push("Your workspace files define who you are and what you do.\n");

  // Inject workspace files
  if (files.length > 0) {
    parts.push("# Workspace Context\n");
    
    const soulFile = files.find(f => f.name === "SOUL.md");
    if (soulFile) {
      parts.push("Embody the persona defined in SOUL.md. Follow its guidance for tone, boundaries, and behavior.\n");
    }

    for (const file of files) {
      parts.push(`## ${file.name}\n`);
      parts.push(file.content);
      parts.push("");
    }
  }

  // Extra instructions (heartbeat prompts, channel context, etc.)
  if (extraInstructions) {
    parts.push("## Additional Context\n");
    parts.push(extraInstructions);
  }

  return parts.join("\n");
}
```

### Using It with pi-mono

```typescript
// kavakclaw/src/agent-runner/index.ts

import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, registerBuiltInApiProviders } from "@mariozechner/pi-ai";
import { buildSystemPrompt } from "./workspace-loader.js";
import { loadToolsFromSkills } from "./tools-loader.js";

// Register all LLM providers
registerBuiltInApiProviders();

const config = JSON.parse(process.env.KAVAKCLAW_CONFIG!);
const workspace = process.env.KAVAKCLAW_WORKSPACE!;

// Build system prompt from workspace files
const systemPrompt = buildSystemPrompt(workspace);

// Load tools from skills/ directory
const tools = loadToolsFromSkills(path.join(workspace, "skills"));

// Create pi-mono agent
const agent = new Agent({
  getApiKey: async (provider) => getCredential(provider, config),
});

agent.state = {
  systemPrompt,
  model: getModel(config.provider, config.modelId),
  tools,
  thinkingLevel: config.thinkingLevel || "high",
};

// Ready to receive messages
// agent.prompt("Hello") → sends to LLM with system prompt + tools
```

## What OpenClaw Adds (That We May or May Not Need)

| OpenClaw Feature | Lines | KavakClaw Decision |
|-----------------|-------|-------------------|
| File caching by inode/mtime | ~50 | Skip for v1 (re-read on each turn is fine) |
| Boundary-safe file open (prevents race conditions) | ~100 | Skip for v1 (single-writer model) |
| Bootstrap budget analysis + warnings | ~200 | Implement (important for large MEMORY.md files) |
| Per-session file filtering | ~50 | Skip (all agents see all their files) |
| Hook overrides for bootstrap files | ~100 | Skip for v1 (no plugin system yet) |
| Front-matter stripping | ~20 | Implement (skills use YAML front-matter) |
| SOUL.md persona enforcement instruction | 2 lines | Implement (critical for agent identity) |
| Missing file markers | ~10 | Implement (useful for debugging) |
| Truncation warnings in prompt | ~50 | Implement (agents should know when context is cut) |

## The Three Deployments with Workspace Files

### Concierge (Claudia)
```
workspaces/concierge-andrea-mx/
├── SOUL.md           → "You are Claudia, a personal automotive concierge..."
├── AGENTS.md         → "When Andrea contacts you, check her purchase history..."
├── USER.md           → "Andrea García, +525512345678, drives a 2023 Honda CR-V..."
├── TOOLS.md          → "pricing-cli: check current market value..."
├── HEARTBEAT.md      → "Check if Andrea's insurance renewal is coming up..."
├── MEMORY.md         → "Andrea mentioned wanting an SUV for her family. Last contact: Feb 2026."
└── skills/
    ├── pricing/SKILL.md
    ├── financing/SKILL.md
    └── scheduling/SKILL.md
```

### City CEO
```
workspaces/ceo-cuernavaca/
├── SOUL.md           → "You are the GM of Kavak Cuernavaca. You own the P&L..."
├── AGENTS.md         → "Every morning, check inventory levels, pricing trends..."
├── USER.md           → "Reports to: Regional VP. Direct reports: 12 team members..."
├── TOOLS.md          → "analytics-cli: query sales data. hr-cli: check schedules..."
├── HEARTBEAT.md      → "Check: inventory age, daily sales, reconditioning queue..."
├── MEMORY.md         → "SUVs moving 40% faster than sedans. Adjusted acquisition Feb 2026."
└── skills/
    ├── analytics/SKILL.md
    ├── pricing/SKILL.md
    ├── acquisition/SKILL.md
    └── hr/SKILL.md
```

### Production Manager
```
workspaces/prod-mgr-main/
├── SOUL.md           → "You manage Kavak's operational machinery..."
├── AGENTS.md         → "Monitor all systems. When issues arise, diagnose and fix..."
├── TOOLS.md          → "kubectl: manage deployments. datadog-cli: check metrics..."
├── HEARTBEAT.md      → "Check: error rates, queue depths, deployment status..."
├── MEMORY.md         → "Reconditioning API had latency issue Feb 12, fixed by..."
└── skills/
    ├── monitoring/SKILL.md
    ├── deployment/SKILL.md
    ├── incident/SKILL.md
    └── reporting/SKILL.md
```

Same harness, same pi-mono agent loop, same workspace loader. Different files = different agent.
