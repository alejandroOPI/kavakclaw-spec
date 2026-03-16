# 3. Technical Specification

## Overview

KavakClaw is ~2,000 lines of TypeScript that connects pi-mono (the agent engine) to an isolation runtime, a scheduler, and a channel router. Everything else -- the agent's behavior, knowledge, and tools -- lives in markdown files and CLIs outside the harness.

## Components

### 3.1 Agent Manager

Manages the lifecycle of long-running agent instances.

```typescript
interface AgentManager {
  /**
   * Boot a new agent from a workspace directory.
   * The workspace must contain at minimum SOUL.md and AGENTS.md.
   * Returns a handle used for all subsequent operations.
   */
  spawn(config: AgentConfig): Promise<AgentHandle>;

  /**
   * Pause an agent (stops heartbeats, holds messages in queue).
   * The agent process stays alive but stops receiving input.
   */
  pause(agentId: string): Promise<void>;

  /**
   * Resume a paused agent (restarts heartbeats, drains message queue).
   */
  resume(agentId: string): Promise<void>;

  /**
   * Permanently stop and clean up an agent.
   * Workspace files are preserved on disk; only the process is killed.
   */
  kill(agentId: string): Promise<void>;

  /**
   * List all agents with their current status.
   */
  list(): Promise<AgentStatus[]>;

  /**
   * Health check for a specific agent.
   */
  health(agentId: string): Promise<HealthCheck>;

  /**
   * Spawn N agents from a template workspace.
   * Used for scaling concierge agents: one template, many instances.
   * Each instance gets a unique workspace cloned from the template.
   */
  scale(template: string, count: number, customizer?: (workspace: string, index: number) => Promise<void>): Promise<AgentHandle[]>;
}

interface AgentConfig {
  /** Unique identifier for this agent instance */
  id: string;

  /** Path to workspace directory containing SOUL.md, AGENTS.md, etc. */
  workspace: string;

  /** Model to use, in provider/model format (e.g., "anthropic/claude-opus-4-6") */
  model: string;

  /** Heartbeat configuration */
  heartbeat: HeartbeatConfig;

  /** Which runtime to use for isolation */
  runtime: "docker" | "process";

  /** Which channels this agent can send/receive on */
  channels: ChannelBinding[];

  /** Maximum concurrent turns (prevents runaway loops) */
  maxConcurrentTurns: number;

  /** Maximum tokens per turn (cost control) */
  maxTokensPerTurn?: number;

  /** Thinking level for the model */
  thinkingLevel?: "off" | "minimal" | "low" | "medium" | "high";
}

interface AgentHandle {
  id: string;
  status: "booting" | "idle" | "active" | "paused" | "error" | "dead";
  runtime: string;
  model: string;
  uptime: number;
  lastActivity: Date;
  messageCount: number;
  totalTokens: number;
  totalCost: number;
}

interface AgentStatus extends AgentHandle {
  heartbeat: HeartbeatConfig;
  channels: ChannelBinding[];
  workspace: string;
  errors: string[];
}

interface HealthCheck {
  healthy: boolean;
  checks: {
    processAlive: boolean;
    workspaceReadable: boolean;
    modelReachable: boolean;
    lastHeartbeatAge: number;
    memoryUsageMb: number;
    errorRate: number;  // errors per hour
  };
}
```

**Implementation notes:**

- Agents are registered in a SQLite database (`agents.db`) with their config and current status.
- On harness restart, all agents marked as "active" or "idle" are automatically re-spawned.
- The `scale()` method clones a template workspace, optionally runs a customizer function (to inject customer-specific data), and spawns each instance.
- Agent IDs are human-readable (e.g., `concierge-andrea-mx`, `ceo-cuernavaca`, `prod-mgr-main`).

### 3.2 Model Gateway

pi-mono serves as the model gateway. No new code needed for model routing. The harness configures it.

```typescript
// Agent runner (inside container/subprocess) initializes pi-mono:

import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, registerBuiltInApiProviders } from "@mariozechner/pi-ai";

// Register all built-in providers (Anthropic, OpenAI, Google, etc.)
registerBuiltInApiProviders();

// Create agent with the configured model
const agent = new Agent({
  getApiKey: async (provider: string) => {
    // In Docker mode: calls credential proxy
    // In Process mode: reads from injected env var
    return getCredentialForProvider(provider);
  },
});

// Configure from workspace files
agent.state = {
  systemPrompt: buildSystemPrompt(workspace),  // Reads SOUL.md, AGENTS.md, etc.
  model: getModel(config.provider, config.modelId),
  tools: loadToolsFromSkills(workspace),
  thinkingLevel: config.thinkingLevel || "high",
};
```

**Supported providers (via pi-mono, no additional code):**

| Provider | API | Auth |
|----------|-----|------|
| Anthropic | anthropic-messages | API key |
| OpenAI | openai-responses | API key |
| OpenAI Codex | openai-codex-responses | OAuth (ChatGPT) |
| Google | google-generative-ai | API key |
| Google Vertex | google-vertex | Service account |
| Google Gemini CLI | google-gemini-cli | OAuth |
| Azure OpenAI | azure-openai-responses | API key |
| Amazon Bedrock | bedrock-converse-stream | IAM role |
| Mistral | mistral-conversations | API key |
| GitHub Copilot | openai-responses (custom headers) | OAuth |
| OpenRouter | openai-responses (custom base URL) | API key |
| Any OpenAI-compatible | openai-completions | API key |

**Model hot-swap:** The agent's model can be changed at runtime by updating the config. The Agent Manager restarts the agent's model configuration without killing the process. Useful for cost optimization: use Opus for complex tasks, Sonnet for routine, Haiku for heartbeat checks.

### 3.3 Scheduler

Variable-frequency heartbeats and cron tasks. The scheduler runs in the harness process (not inside agent containers).

```typescript
interface Scheduler {
  /**
   * Set or update an agent's heartbeat configuration.
   * The heartbeat injects a message into the agent at the configured interval.
   */
  setHeartbeat(agentId: string, config: HeartbeatConfig): void;

  /**
   * Dynamically adjust heartbeat frequency.
   * Called by the agent itself via the set_heartbeat tool.
   */
  adjustFrequency(agentId: string, level: "active" | "idle" | "dormant"): void;

  /**
   * Add a cron task for an agent.
   * The cron injects a message into the agent at the scheduled time.
   */
  addCron(agentId: string, cron: CronConfig): string;  // returns cronId

  /**
   * Remove a cron task.
   */
  removeCron(cronId: string): void;

  /**
   * List all crons for an agent.
   */
  listCrons(agentId: string): CronConfig[];

  /**
   * Get next scheduled events for an agent.
   */
  nextEvents(agentId: string, count: number): ScheduledEvent[];
}

interface HeartbeatConfig {
  /** Base interval in milliseconds */
  intervalMs: number;

  /** What to inject when the heartbeat fires */
  prompt: string;

  /** Named frequency levels the agent can switch between */
  levels: {
    /** Agent is actively engaged (e.g., customer in purchase flow) */
    active: { intervalMs: number; prompt: string };
    /** Agent is waiting but attentive (e.g., customer said "I'll think about it") */
    idle: { intervalMs: number; prompt: string };
    /** Agent is in long-term mode (e.g., customer not expected back for months) */
    dormant: { intervalMs: number; prompt: string };
  };

  /** Current level */
  currentLevel: "active" | "idle" | "dormant";
}

interface CronConfig {
  id?: string;
  schedule: string;        // Cron expression (e.g., "0 9 * * 1-5")
  timezone: string;        // IANA timezone
  prompt: string;          // Message to inject
  enabled: boolean;
}

interface ScheduledEvent {
  type: "heartbeat" | "cron";
  agentId: string;
  scheduledFor: Date;
  prompt: string;
}
```

**Cost model integration:**

```
Cost per agent = heartbeat_frequency * activity_cost * harness_efficiency

Where:
- heartbeat_frequency = 1 / heartbeat.intervalMs (how often the agent wakes)
- activity_cost = avg_tokens_per_turn * model_price_per_token
- harness_efficiency = ratio of useful work to total tokens (compaction quality)
```

The scheduler enables the first lever. An agent that self-adjusts from "active" (every 15 min) to "dormant" (every week) reduces its cost by ~672x.

**Agent-facing tools:**

The agent can modify its own heartbeat via a tool:

```typescript
// Available to every agent automatically
const heartbeatTool: AgentTool = {
  name: "set_heartbeat",
  label: "Adjust heartbeat frequency",
  description: "Change how often I wake up. Use 'active' for engaged work, 'idle' for waiting, 'dormant' for long-term.",
  schema: Type.Object({
    level: Type.Union([Type.Literal("active"), Type.Literal("idle"), Type.Literal("dormant")]),
    reason: Type.String({ description: "Why this change (logged for audit)" }),
  }),
  execute: async (id, params) => {
    scheduler.adjustFrequency(agentId, params.level);
    auditLog.append({ agent: agentId, action: "heartbeat_change", level: params.level, reason: params.reason });
    return {
      content: [{ type: "text", text: `Heartbeat set to ${params.level}: ${heartbeatConfig.levels[params.level].intervalMs / 1000}s interval` }],
      details: { level: params.level },
    };
  },
};

const cronTool: AgentTool = {
  name: "schedule_task",
  label: "Schedule a recurring task",
  description: "Create a cron job that will send me a prompt at the specified schedule.",
  schema: Type.Object({
    schedule: Type.String({ description: "Cron expression" }),
    timezone: Type.String({ description: "IANA timezone" }),
    prompt: Type.String({ description: "Message to inject when the cron fires" }),
  }),
  execute: async (id, params) => {
    const cronId = scheduler.addCron(agentId, { ...params, enabled: true });
    return {
      content: [{ type: "text", text: `Cron ${cronId} created: "${params.schedule}" (${params.timezone})` }],
      details: { cronId },
    };
  },
};
```

### 3.4 Channel Router

Routes messages between external channels and agents.

```typescript
interface ChannelRouter {
  /**
   * Register a channel implementation.
   * Channels self-register at startup (NanoClaw pattern).
   */
  registerChannel(channel: Channel): void;

  /**
   * Route an inbound message to the correct agent.
   * Uses the routing table to find the agent that handles this sender/channel.
   */
  route(message: InboundMessage): Promise<AgentHandle | null>;

  /**
   * Send a message from an agent to a channel.
   */
  send(agentId: string, channel: string, target: string, message: OutboundMessage): Promise<void>;

  /**
   * Bind an agent to a channel + target.
   * Example: bind concierge-andrea to WhatsApp +525512345678
   */
  bind(agentId: string, channel: string, target: string): void;

  /**
   * Unbind an agent from a channel.
   */
  unbind(agentId: string, channel: string, target: string): void;
}

interface Channel {
  name: string;
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  sendMessage(target: string, text: string, attachments?: Attachment[]): Promise<void>;
  onMessage(callback: (message: InboundMessage) => void): void;
  isConnected(): boolean;
}

interface ChannelBinding {
  channel: string;     // "slack", "whatsapp", "api"
  target: string;      // Channel-specific target (Slack channel ID, phone number, etc.)
  direction: "inbound" | "outbound" | "both";
}

interface InboundMessage {
  channel: string;
  sender: string;
  target: string;
  text: string;
  attachments?: Attachment[];
  metadata?: Record<string, unknown>;
  timestamp: Date;
}

interface OutboundMessage {
  text: string;
  attachments?: Attachment[];
  metadata?: Record<string, unknown>;
}
```

**Routing logic:**

1. Message arrives on channel X from sender Y
2. Router looks up bindings: which agent is bound to channel X + sender Y?
3. If found: deliver message to that agent via AgentRuntime.send()
4. If not found: check default routing rules (e.g., "all unknown WhatsApp senders go to a triage agent")
5. If no match: queue for human review

**Agent-facing tool:**

```typescript
const sendMessageTool: AgentTool = {
  name: "send_message",
  label: "Send a message to a channel",
  description: "Send a message to a customer or internal channel.",
  schema: Type.Object({
    channel: Type.String({ description: "Channel name (slack, whatsapp, etc.)" }),
    target: Type.String({ description: "Target (phone number, channel ID, etc.)" }),
    message: Type.String({ description: "Message text" }),
  }),
  execute: async (id, params) => {
    await channelRouter.send(agentId, params.channel, params.target, { text: params.message });
    return {
      content: [{ type: "text", text: `Message sent to ${params.channel}:${params.target}` }],
      details: {},
    };
  },
};
```

### 3.5 Self-Improvement Tools

Tools that let the agent modify its own workspace and improve adjacent systems.

```typescript
// Modify own workspace files
const editWorkspaceTool: AgentTool = {
  name: "edit_workspace",
  label: "Modify my own instructions or skills",
  description: "Update AGENTS.md, create skills, adjust SOUL.md. Changes take effect on next turn.",
  schema: Type.Object({
    file: Type.String({ description: "Relative path within workspace (e.g., 'AGENTS.md', 'skills/new-skill/SKILL.md')" }),
    content: Type.String({ description: "New file content" }),
    reason: Type.String({ description: "Why this change (logged for audit)" }),
  }),
  execute: async (id, params) => {
    const fullPath = path.resolve(workspace, params.file);
    // Security: ensure path doesn't escape workspace
    if (!fullPath.startsWith(workspace)) {
      return { content: [{ type: "text", text: "Error: path escapes workspace" }], details: { error: true } };
    }
    fs.mkdirSync(path.dirname(fullPath), { recursive: true });
    fs.writeFileSync(fullPath, params.content);
    auditLog.append({
      agent: agentId,
      action: "workspace_edit",
      file: params.file,
      reason: params.reason,
      contentLength: params.content.length,
    });
    return {
      content: [{ type: "text", text: `Updated ${params.file} (${params.content.length} bytes)` }],
      details: { file: params.file },
    };
  },
};

// Open a PR in an external repo
const openPrTool: AgentTool = {
  name: "open_pr",
  label: "Open a pull request in an external repository",
  description: "Fix a bug or improve a system I interact with by opening a PR.",
  schema: Type.Object({
    repo: Type.String({ description: "Repository (e.g., 'kavak/pricing-api')" }),
    branch: Type.String({ description: "Branch name" }),
    title: Type.String({ description: "PR title" }),
    description: Type.String({ description: "PR description" }),
    changes: Type.Array(Type.Object({
      file: Type.String(),
      content: Type.String(),
    })),
  }),
  execute: async (id, params) => {
    // Uses git CLI (available in container/subprocess)
    // 1. Clone repo to temp dir
    // 2. Create branch
    // 3. Apply changes
    // 4. Commit + push
    // 5. Open PR via gh CLI
    const result = await gitOperations.openPR(params);
    auditLog.append({ agent: agentId, action: "open_pr", repo: params.repo, pr: result.url });
    return {
      content: [{ type: "text", text: `PR opened: ${result.url}` }],
      details: { prUrl: result.url },
    };
  },
};
```

### 3.6 Agent Runner (Inside Container/Subprocess)

This is the code that runs inside each agent's isolated environment. It's the bridge between the harness and pi-mono.

```typescript
// agent-runner/index.ts
// This runs INSIDE the Docker container or subprocess

import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, registerBuiltInApiProviders } from "@mariozechner/pi-ai";
import { loadWorkspace, loadTools, buildSystemPrompt } from "./workspace-loader.js";
import { createIpcTransport } from "./ipc.js";

// Initialize providers
registerBuiltInApiProviders();

// Load config from environment
const config = JSON.parse(process.env.KAVAKCLAW_CONFIG || "{}");
const workspace = process.env.KAVAKCLAW_WORKSPACE || "/workspace";

// Create IPC transport (communicates with harness)
const ipc = createIpcTransport(config.ipcMode);  // "filesystem" for Docker, "stdio" for Process

// Create the pi-mono agent
const agent = new Agent({
  getApiKey: async (provider: string) => {
    if (config.runtime === "docker") {
      // Credential proxy: fetch from host proxy
      const res = await fetch(`http://${config.credentialProxyHost}:${config.credentialProxyPort}/credential/${provider}`);
      return (await res.json()).apiKey;
    } else {
      // Process runtime: read from env
      return process.env[`${provider.toUpperCase().replace(/-/g, "_")}_API_KEY`];
    }
  },
  thinkingBudgets: config.thinkingBudgets,
  transport: config.transport,
});

// Load workspace into agent state
const ws = loadWorkspace(workspace);
const tools = [
  ...loadTools(path.join(workspace, "skills")),  // Skills from workspace
  ...getBuiltinTools(workspace, config),           // Harness tools (heartbeat, send_message, etc.)
];

agent.state = {
  systemPrompt: buildSystemPrompt(ws),
  model: getModel(config.provider, config.modelId),
  tools,
  thinkingLevel: config.thinkingLevel || "high",
};

// Main loop: wait for messages from harness via IPC
ipc.on("message", async (message) => {
  try {
    // Send message to agent and collect response
    const response = await agent.prompt(message.text);
    ipc.send("response", { id: message.id, text: response, tokens: agent.lastTurnTokens });
  } catch (error) {
    ipc.send("error", { id: message.id, error: error.message });
  }
});

ipc.on("heartbeat", async (heartbeat) => {
  try {
    const response = await agent.prompt(heartbeat.prompt);
    ipc.send("heartbeat_response", { text: response, tokens: agent.lastTurnTokens });
  } catch (error) {
    ipc.send("error", { error: error.message });
  }
});

// Signal ready
ipc.send("ready", { agentId: config.agentId });
```

**System prompt construction:**

```typescript
function buildSystemPrompt(ws: Workspace): string {
  const parts: string[] = [];

  // Core identity
  if (ws.soul) parts.push(`## Soul\n${ws.soul}`);
  if (ws.identity) parts.push(`## Identity\n${ws.identity}`);

  // Operating instructions
  if (ws.agents) parts.push(`## Instructions\n${ws.agents}`);

  // User/customer profile
  if (ws.user) parts.push(`## User\n${ws.user}`);

  // Tool notes
  if (ws.tools) parts.push(`## Tools\n${ws.tools}`);

  // Heartbeat instructions
  if (ws.heartbeat) parts.push(`## Heartbeat\n${ws.heartbeat}`);

  // Inject skill descriptions (name + description, not full content)
  const skills = listSkills(ws.skillsDir);
  if (skills.length > 0) {
    parts.push(`## Available Skills\n${skills.map(s => `- ${s.name}: ${s.description}`).join("\n")}`);
  }

  return parts.join("\n\n---\n\n");
}
```

## Data Flow

### Message Flow (inbound)

```
1. Customer sends WhatsApp message
2. Channel (WhatsApp) receives message
3. Channel Router looks up binding: customer phone -> agent ID
4. Agent Manager finds running agent
5. Agent Runtime delivers message to agent process via IPC
6. Agent runner feeds message to pi-mono Agent
7. Agent processes (may call tools, make API calls)
8. Agent produces response
9. Agent runner sends response back via IPC
10. Agent Runtime delivers response to Agent Manager
11. Agent Manager routes response to Channel Router
12. Channel Router sends via WhatsApp channel
13. Customer receives response
```

### Heartbeat Flow

```
1. Scheduler timer fires for agent X
2. Scheduler calls AgentManager.heartbeat(agentId)
3. Agent Manager sends heartbeat message via runtime IPC
4. Agent runner injects heartbeat prompt into pi-mono Agent
5. Agent checks HEARTBEAT.md instructions (e.g., "check pending tasks, review customer status")
6. Agent may:
   a. Do nothing (respond with HEARTBEAT_OK)
   b. Send a proactive message (via send_message tool)
   c. Adjust its own heartbeat (via set_heartbeat tool)
   d. Create/modify cron tasks
7. Response flows back through IPC -> Agent Manager
8. If agent sent messages, Channel Router delivers them
```

### Self-Improvement Flow

```
1. Agent encounters a recurring issue (e.g., "I keep forgetting the pricing API format")
2. Agent calls edit_workspace tool to update AGENTS.md with a note about the API format
3. Harness writes the file to the agent's workspace
4. On next turn, the system prompt is rebuilt with the updated AGENTS.md
5. Agent's behavior improves
```

```
1. Agent discovers a bug in a KAVAK API it consumes
2. Agent calls open_pr tool with the fix
3. Harness clones repo, applies changes, opens PR via gh CLI
4. Human reviewer approves or rejects
5. If approved, the API improves for all agents
```

## State Management

### What's Persisted

| Data | Where | Why |
|------|-------|-----|
| Agent configs | `agents.db` (SQLite) | Survive harness restarts |
| Agent workspaces | Filesystem (EBS on AWS, local on Mac) | Agent identity and state |
| Message history | Agent's pi-mono session (JSONL) | Conversation continuity |
| Heartbeat/cron configs | `agents.db` | Survive restarts |
| Channel bindings | `agents.db` | Customer -> agent mapping |
| Audit log | Append-only file per agent | Compliance and debugging |
| Cost tracking | `agents.db` | Usage monitoring |

### What's Not Persisted (Ephemeral)

| Data | Why |
|------|-----|
| In-flight tool execution | Retried on restart |
| IPC messages in transit | Re-queued on restart |
| Agent process state (heap) | Reconstructed from workspace + session |

## Audit Trail

Every significant action is logged:

```typescript
interface AuditEntry {
  timestamp: Date;
  agentId: string;
  action: string;          // "workspace_edit", "open_pr", "heartbeat_change", "message_sent", "tool_call"
  details: Record<string, unknown>;
  tokenCount?: number;
  cost?: number;
}
```

Audit logs are append-only, one file per agent per day:
```
logs/
├── concierge-andrea-mx/
│   ├── 2026-03-16.jsonl
│   └── 2026-03-17.jsonl
├── ceo-cuernavaca/
│   └── 2026-03-16.jsonl
```
