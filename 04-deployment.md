# 4. Deployment Modes

## Overview

KavakClaw runs in two modes from the same codebase. The `runtime` config field selects the isolation strategy.

```json
{ "runtime": "docker" }    // Mac mini, dev environments
{ "runtime": "process" }   // AWS ECS/Fargate, production
```

## Mode 1: Docker (Mac mini / Dev)

For local development, personal assistants, and small-scale deployments.

### Architecture

```
┌──────────────────────────────────────────────────┐
│  Mac mini (host)                                  │
│                                                   │
│  ┌──────────────────────────────────────────────┐ │
│  │  KavakClaw Harness (Node.js process)         │ │
│  │  - Agent Manager                             │ │
│  │  - Scheduler                                 │ │
│  │  - Channel Router                            │ │
│  │  - Credential Proxy (port 3001)              │ │
│  │  - agents.db (SQLite)                        │ │
│  └──────┬──────────┬──────────┬─────────────────┘ │
│         │          │          │                    │
│    ┌────▼───┐ ┌────▼───┐ ┌───▼────┐              │
│    │Docker  │ │Docker  │ │Docker  │              │
│    │Agent A │ │Agent B │ │Agent C │              │
│    │        │ │        │ │        │              │
│    │/workspace (mounted volume)    │              │
│    │pi-mono agent loop             │              │
│    │Credential proxy → host:3001   │              │
│    └────────┘ └────────┘ └────────┘              │
└──────────────────────────────────────────────────┘
```

### How It Works

1. Agent Manager calls `docker run` with:
   - Workspace directory mounted as volume at `/workspace`
   - Credential proxy URL as env var
   - Agent config as env var
   - Network access to host (for credential proxy + external APIs)
   - No real API keys in container environment

2. Container runs the agent-runner process, which:
   - Loads workspace files (SOUL.md, etc.)
   - Initializes pi-mono with credential proxy
   - Waits for messages via filesystem IPC

3. Credential Proxy (runs on host, port 3001):
   - Agent container sends API requests to `http://host.docker.internal:3001`
   - Proxy strips placeholder auth
   - Proxy injects real API key from local config
   - Proxy forwards to actual provider API
   - Real keys never enter the container

### Docker Container Spec

```dockerfile
FROM node:22-slim

# Non-root user
RUN useradd -m -s /bin/bash agent
USER agent
WORKDIR /home/agent

# Install pi-mono dependencies
COPY package.json package-lock.json ./
RUN npm ci --production

# Copy agent runner
COPY agent-runner/ ./agent-runner/

# Entrypoint
CMD ["node", "agent-runner/index.js"]
```

### Volume Mounts

| Host Path | Container Path | Mode | Purpose |
|-----------|---------------|------|---------|
| `workspaces/{agentId}/` | `/workspace` | rw | Agent workspace (SOUL.md, etc.) |
| `data/sessions/{agentId}/` | `/home/agent/.sessions/` | rw | pi-mono session transcripts |
| `data/ipc/{agentId}/` | `/ipc` | rw | IPC directory |
| `/dev/null` | `/workspace/.env` | ro | Shadow any .env files |

### docker-compose.yml (example)

```yaml
version: "3.8"

services:
  harness:
    build: .
    ports:
      - "18800:18800"  # Admin API
    volumes:
      - ./workspaces:/workspaces
      - ./data:/data
      - ./config.json:/config.json:ro
    environment:
      - KAVAKCLAW_CONFIG=/config.json
      - KAVAKCLAW_RUNTIME=docker
    depends_on:
      - redis  # Optional: for cross-process events

  # Agent containers are spawned dynamically by the harness
  # They are NOT defined in docker-compose
```

## Mode 2: Process (AWS ECS/Fargate)

For production deployments with hundreds or thousands of concurrent agents.

### Architecture

```
┌───────────────────────────────────────────────────────┐
│  AWS ECS Cluster                                       │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  ECS Task: KavakClaw Harness                     │  │
│  │  (Fargate, 2 vCPU, 4GB RAM)                      │  │
│  │                                                   │  │
│  │  ┌───────────────────────────────────────────┐   │  │
│  │  │  Node.js Process                          │   │  │
│  │  │  - Agent Manager                          │   │  │
│  │  │  - Scheduler                              │   │  │
│  │  │  - Channel Router                         │   │  │
│  │  │  - agents.db (SQLite on EFS)              │   │  │
│  │  └──────┬──────────┬──────────┬──────────────┘   │  │
│  │         │          │          │                   │  │
│  │    ┌────▼───┐ ┌────▼───┐ ┌───▼────┐             │  │
│  │    │Child   │ │Child   │ │Child   │             │  │
│  │    │Process │ │Process │ │Process │             │  │
│  │    │Agent A │ │Agent B │ │Agent C │             │  │
│  │    │        │ │        │ │        │             │  │
│  │    │Workspace on EFS    │                        │  │
│  │    │pi-mono agent loop  │                        │  │
│  │    │API keys from       │                        │  │
│  │    │Secrets Manager     │                        │  │
│  │    └────────┘ └────────┘ └────────┘             │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────┐     │
│  │ EFS         │  │ Secrets      │  │ CloudWatch│     │
│  │ (workspaces)│  │ Manager      │  │ (logs)    │     │
│  └─────────────┘  └──────────────┘  └──────────┘     │
└───────────────────────────────────────────────────────┘
```

### How It Works

1. Agent Manager calls `child_process.fork()` with:
   - Workspace path (on EFS)
   - Agent config as serialized env var
   - Restricted environment (only whitelisted env vars)

2. Child process runs the same agent-runner code, but:
   - Credentials come from AWS Secrets Manager (not proxy)
   - IPC via Node.js process messaging (not filesystem)
   - Workspace on EFS (shared filesystem, per-agent directories)

3. Isolation is weaker than Docker but:
   - The ECS task itself IS a container (so there's one layer of containment)
   - Child processes run with minimal env vars
   - Workspace directories are scoped per agent (no shared access)
   - For stronger isolation, each agent can be its own ECS task (higher cost)

### AWS Resources

| Resource | Purpose | Sizing |
|----------|---------|--------|
| ECS Fargate | Runs the harness + agent subprocesses | 2 vCPU, 4GB per harness instance |
| EFS | Agent workspaces, session data | Pay per GB |
| Secrets Manager | API keys for all providers | One secret per provider |
| CloudWatch | Logs, metrics, alarms | Standard |
| SQS (optional) | Cross-instance message queue | For multi-instance scaling |
| DynamoDB (optional) | Replace SQLite for multi-instance | For multi-instance scaling |

### ECS Task Definition (example)

```json
{
  "family": "kavakclaw-harness",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "2048",
  "memory": "4096",
  "executionRoleArn": "arn:aws:iam::role/kavakclaw-execution",
  "taskRoleArn": "arn:aws:iam::role/kavakclaw-task",
  "containerDefinitions": [
    {
      "name": "harness",
      "image": "ECR_REPO/kavakclaw:latest",
      "essential": true,
      "environment": [
        { "name": "KAVAKCLAW_RUNTIME", "value": "process" },
        { "name": "KAVAKCLAW_WORKSPACE_ROOT", "value": "/efs/workspaces" },
        { "name": "KAVAKCLAW_DB_PATH", "value": "/efs/data/agents.db" }
      ],
      "secrets": [
        { "name": "ANTHROPIC_API_KEY", "valueFrom": "arn:aws:secretsmanager:...:anthropic-key" },
        { "name": "OPENAI_API_KEY", "valueFrom": "arn:aws:secretsmanager:...:openai-key" },
        { "name": "GOOGLE_API_KEY", "valueFrom": "arn:aws:secretsmanager:...:google-key" }
      ],
      "mountPoints": [
        { "sourceVolume": "efs-workspaces", "containerPath": "/efs" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/kavakclaw",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "harness"
        }
      }
    }
  ],
  "volumes": [
    {
      "name": "efs-workspaces",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-XXXXX",
        "rootDirectory": "/kavakclaw"
      }
    }
  ]
}
```

### Scaling Strategy

**Vertical (more agents per harness):**
- Each harness instance handles up to ~50 agents (depends on activity level)
- Increase ECS task CPU/memory for more agents per instance

**Horizontal (more harness instances):**
- Multiple ECS tasks sharing EFS for workspaces
- SQS for message routing across instances
- DynamoDB replaces SQLite for shared state
- Each agent is assigned to one harness instance (sticky routing)

**Per-agent isolation (maximum security):**
- Each agent runs as its own ECS task
- Higher cost but complete isolation
- Use for high-value agents (City CEO, Production Manager)

## Config File

```json
{
  "runtime": "docker",

  "docker": {
    "image": "kavakclaw-agent:latest",
    "credentialProxyPort": 3001,
    "network": "kavakclaw-net",
    "memoryLimit": "512m",
    "cpuLimit": "1.0"
  },

  "process": {
    "maxMemoryMb": 512,
    "maxAgentsPerInstance": 50
  },

  "models": {
    "default": "anthropic/claude-opus-4-6",
    "costOptimized": "anthropic/claude-sonnet-4-5",
    "heartbeatModel": "anthropic/claude-haiku-3-5",
    "providers": {
      "anthropic": { "apiKey": "env:ANTHROPIC_API_KEY" },
      "openai": { "apiKey": "env:OPENAI_API_KEY" },
      "google": { "apiKey": "env:GOOGLE_API_KEY" }
    }
  },

  "scheduler": {
    "defaultHeartbeat": {
      "active": { "intervalMs": 900000, "prompt": "Check your active tasks and customer status." },
      "idle": { "intervalMs": 14400000, "prompt": "Quick check: any pending items?" },
      "dormant": { "intervalMs": 604800000, "prompt": "Long-term check-in." }
    }
  },

  "channels": {
    "slack": {
      "enabled": true,
      "token": "env:SLACK_BOT_TOKEN",
      "defaultChannel": "C0XXXXX"
    },
    "whatsapp": {
      "enabled": true,
      "provider": "twilio",
      "accountSid": "env:TWILIO_SID",
      "authToken": "env:TWILIO_TOKEN"
    }
  },

  "security": {
    "mountBlockedPatterns": [
      ".ssh", ".gnupg", ".aws", ".env", "credentials",
      "id_rsa", "id_ed25519", "private_key", ".secret"
    ],
    "auditLogPath": "logs/audit/"
  }
}
```
