# 5. Security Model

## Threat Model

KavakClaw agents handle customer data, have access to internal systems, and can modify their own code. The security model must account for:

1. **Prompt injection** -- An external message tricks the agent into performing unintended actions
2. **Agent escape** -- The agent's execution environment accesses resources it shouldn't
3. **Credential exposure** -- API keys or internal secrets leak to the agent or external parties
4. **Cross-agent contamination** -- Agent A accesses Agent B's data or workspace
5. **Self-improvement abuse** -- An agent modifies its own instructions maliciously

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│  TRUSTED ZONE                                                │
│  (Harness process, config, credential store)                 │
│                                                              │
│  • Has real API keys                                         │
│  • Has database access                                       │
│  • Controls agent lifecycle                                  │
│  • Routes messages                                           │
│  • Manages crons/heartbeats                                  │
└────────────────────────────┬────────────────────────────────┘
                             │ IPC (constrained interface)
                             │
┌────────────────────────────▼────────────────────────────────┐
│  SANDBOXED ZONE                                              │
│  (Agent container / subprocess)                              │
│                                                              │
│  • NO real API keys (proxy/placeholder only)                 │
│  • Only sees its own workspace (mounted/scoped)              │
│  • Can only send messages via IPC (not directly)             │
│  • Can only modify files within workspace                    │
│  • Network access to provider APIs (via proxy or direct)     │
└────────────────────────────┬────────────────────────────────┘
                             │ External input (untrusted)
                             │
┌────────────────────────────▼────────────────────────────────┐
│  UNTRUSTED ZONE                                              │
│  (Customer messages, external webhooks)                      │
│                                                              │
│  • Potential prompt injection                                │
│  • Potential social engineering                               │
│  • May contain malicious content                             │
└─────────────────────────────────────────────────────────────┘
```

## Security Controls

### 1. Credential Isolation

**Docker mode (strongest):**

Real API keys never enter agent containers. A credential proxy runs on the host and injects authentication headers transparently.

```
Agent container                    Host (credential proxy)
     │                                  │
     │  POST http://proxy:3001/v1/...   │
     │  Authorization: placeholder      │
     │ ──────────────────────────────►  │
     │                                  │  Strips placeholder auth
     │                                  │  Injects real API key
     │                                  │  Forwards to api.anthropic.com
     │                                  │
     │  ◄──────────────────────────────  │
     │  Response from provider          │
```

- API keys are stored in the host's config or env vars, never passed to containers
- The credential proxy is the only process with access to real keys
- Agents cannot discover real credentials via environment, stdin, files, or /proc
- The proxy validates that requests go to known provider endpoints (no exfiltration via proxy)

**Process mode (AWS):**

API keys are injected into the harness process via AWS Secrets Manager. Child processes inherit only whitelisted environment variables.

```typescript
// Only these env vars are passed to child processes:
const ALLOWED_ENV_VARS = [
  "KAVAKCLAW_CONFIG",
  "KAVAKCLAW_WORKSPACE",
  "KAVAKCLAW_IPC_MODE",
  "NODE_ENV",
  "TZ",
];

// API keys are fetched on-demand by the harness and passed via IPC,
// never as environment variables to the child process.
```

For maximum isolation on AWS, deploy each agent as its own ECS task with its own IAM role scoped to only the secrets it needs.

### 2. Filesystem Isolation

**Docker mode:**
- Each agent container only sees its own workspace (mounted as a volume)
- Project root is NOT mounted (unlike NanoClaw which mounts it read-only)
- .env files are shadowed with /dev/null
- No access to other agents' workspaces, host filesystem, or harness code

**Process mode:**
- Each child process has its workspace path set as cwd
- Workspace paths are validated to prevent traversal (no `../`)
- The harness validates all file operations stay within the workspace

**Mount security (adopted from NanoClaw):**

```typescript
// Blocked patterns -- these are NEVER mounted or accessible
const BLOCKED_PATTERNS = [
  ".ssh", ".gnupg", ".aws", ".azure", ".gcloud", ".kube",
  ".docker", "credentials", ".env", ".netrc", ".npmrc",
  "id_rsa", "id_ed25519", "private_key", ".secret",
];

// All paths are resolved (symlinks followed) before validation
// Container paths reject ".." and absolute paths
// Workspace boundary is enforced: all writes must resolve within workspace/
```

### 3. Cross-Agent Isolation

- Each agent has its own workspace directory (no shared filesystems between agents)
- Each agent has its own pi-mono session (no shared conversation history)
- Each agent has its own IPC channel
- Channel bindings prevent one agent from receiving another's messages
- The harness never forwards messages between agents without explicit routing rules

### 4. Self-Improvement Controls

Agents can modify their own workspace, but with guardrails:

```typescript
// Allowed modifications:
// - AGENTS.md (operating instructions)
// - skills/ (new tools and scripts)
// - memory/ (journals, topics)
// - TOOLS.md (tool-specific notes)
// - HEARTBEAT.md (heartbeat behavior)

// Restricted modifications (require human approval):
// - SOUL.md (core identity -- changes queued for review)
// - USER.md (customer profile -- changes queued for review)

// Forbidden modifications:
// - Anything outside workspace/
// - Agent runner code
// - Harness code
// - Config files

interface SelfImprovementPolicy {
  allowed: string[];         // Glob patterns for unrestricted modification
  requiresApproval: string[];  // Glob patterns for human-approved modification
  forbidden: string[];       // Glob patterns that are never modifiable
}

const defaultPolicy: SelfImprovementPolicy = {
  allowed: ["AGENTS.md", "TOOLS.md", "HEARTBEAT.md", "skills/**", "memory/**"],
  requiresApproval: ["SOUL.md", "USER.md", "IDENTITY.md"],
  forbidden: ["../**, /*.json", "agent-runner/**"],
};
```

When an agent tries to modify a `requiresApproval` file:
1. The change is saved to a staging area (not applied)
2. A notification is sent via ATH channel (Slack)
3. A human reviews and approves/rejects
4. If approved, the change is applied to the workspace

### 5. Audit Trail

Every action is logged to an append-only audit log:

```typescript
// Actions that are ALWAYS logged:
// - Tool calls (all)
// - Workspace modifications
// - Messages sent to channels
// - Heartbeat frequency changes
// - Cron modifications
// - PR creation
// - Errors

// Audit log format (JSONL, one file per agent per day):
{
  "ts": "2026-03-16T14:30:00Z",
  "agent": "concierge-andrea-mx",
  "action": "tool_call",
  "tool": "send_message",
  "params": { "channel": "whatsapp", "target": "+525512345678" },
  "result": "success",
  "tokens": 1250,
  "cost": 0.018,
  "model": "anthropic/claude-opus-4-6"
}
```

Audit logs are:
- Append-only (the agent cannot delete or modify them)
- Stored outside the agent's workspace
- Rotated daily
- Shipped to CloudWatch (AWS) or local log aggregation

### 6. Rate Limiting

```typescript
interface RateLimits {
  maxTurnsPerHour: number;      // Prevent runaway loops
  maxTokensPerHour: number;     // Cost control
  maxMessagesPerHour: number;   // Spam prevention
  maxPRsPerDay: number;         // Self-improvement rate limit
  maxWorkspaceEditsPerHour: number;  // Prevent workspace corruption
}

const defaultLimits: RateLimits = {
  maxTurnsPerHour: 120,
  maxTokensPerHour: 1_000_000,
  maxMessagesPerHour: 60,
  maxPRsPerDay: 10,
  maxWorkspaceEditsPerHour: 30,
};
```

When a rate limit is hit:
1. The action is blocked
2. An alert is sent to the monitoring system
3. The agent receives an error message explaining the limit
4. The agent can request a limit increase via ATH

### 7. Network Controls

**Docker mode:**
- Agents have network access (needed for API calls)
- Network is restricted to known provider endpoints via Docker network policy
- Internal KAVAK APIs are accessed via the credential proxy (which validates endpoints)

**Process mode (AWS):**
- Network controlled by VPC security groups
- Agents access provider APIs directly (credentials from Secrets Manager)
- Internal KAVAK APIs accessible within VPC
- No public internet access except to provider endpoints (via NAT gateway)

## Security Comparison vs Alternatives

| Control | OpenClaw | NanoClaw | KavakClaw |
|---------|----------|----------|-----------|
| Agent isolation | None (same process) | Docker containers | Docker (local) / Process (AWS) |
| Credential protection | Config file | Credential proxy | Proxy (Docker) / Secrets Mgr (AWS) |
| Filesystem scoping | Workspace dir | Container mounts | Container mounts / path validation |
| Cross-agent isolation | Session separation | Per-group containers | Per-agent workspace + process |
| Self-improvement controls | None | N/A (no self-improvement) | Policy-based with human approval |
| Audit trail | Session JSONL | Per-group logs | Structured audit log with cost tracking |
| Rate limiting | None | None | Configurable per-agent |
