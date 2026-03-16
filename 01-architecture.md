# 1. Architecture Overview

## What We're Building

KavakClaw is a minimal agentic harness -- the execution infrastructure that turns a model + filesystem into a long-running, self-improving agent. It's not a framework, not a chatbot, not a copilot. It's the runtime that makes the v4 vision deployable: personal concierges, city CEOs, and production managers running concurrently on the same infrastructure.

## What a Harness Is

> An AI harness is the system around a model that turns it into a reliable agent. It defines the loop -- observe, act, verify, repeat -- keeps state and progress across sessions, and records what happened so results are auditable. When you evaluate "an agent," you're evaluating the model and the harness together. Never the model alone.

The harness is NOT the agent's intelligence (that's the model), NOT its knowledge (that's the memory system), NOT its capabilities (that's the CLI/tool catalog), and NOT its escalation path (that's ATH). The harness is the infrastructure that connects these pieces and keeps the agent alive.

## Design Principles

### 1. The Model Eats the Architecture

Every six months, a new model generation makes a layer of infrastructure obsolete. The harness must be thin enough that the obsolete parts can be replaced without rewriting the whole system. This means:

- No business logic in the harness. All agent behavior lives in markdown files.
- No hardcoded workflows. The agent decides what to do.
- No model-specific code in the core. Provider abstraction handles all models.

### 2. Filesystem as Interface

The agent's identity, instructions, knowledge, and tools are files in a directory:

```
workspace/
├── SOUL.md          # Who the agent is (persona, boundaries, tone)
├── AGENTS.md        # Operating instructions (what to do, how to do it)
├── TOOLS.md         # Tool-specific notes and configuration
├── IDENTITY.md      # Name, vibe, emoji
├── USER.md          # Who the agent serves (customer profile, preferences)
├── HEARTBEAT.md     # What to check on each heartbeat
├── skills/          # Tool definitions and scripts
│   ├── pricing/
│   │   └── SKILL.md
│   ├── inventory/
│   │   └── SKILL.md
│   └── ...
└── memory/          # Local memory (journals, topics, bridge files)
    ├── journal/
    ├── topics/
    └── ...
```

To create a new type of agent (concierge, city CEO, production manager), you create a new workspace directory with different markdown files. No code changes. The harness doesn't know or care what kind of agent it's running.

### 3. Long-Running, Not Ephemeral

Agents boot once and stay alive. They receive messages, wake up on heartbeats, run cron tasks, and persist state. They are not spun up per request and torn down after. This is critical for:

- **State continuity** -- The agent remembers the conversation without re-reading everything
- **Cost efficiency** -- No boot overhead per message
- **Self-improvement** -- The agent accumulates context over time
- **Relationship building** -- Andrea's concierge knows her history

### 4. Multi-Vendor, No Lock-In

The harness works with any LLM provider. Switching from Claude to GPT to Gemini is a config change, not a code change. This is achieved through pi-mono's provider abstraction (see [Dependencies](./06-dependencies.md)).

### 5. Dual Deployment

Same codebase runs on a Mac mini (Docker containers for agent isolation) and on AWS (ECS/Fargate with process isolation). The runtime abstraction handles the difference (see [Deployment Modes](./04-deployment.md)).

## What's In Scope vs Out of Scope

| In Scope (The Harness) | Out of Scope (Already Exists) |
|------------------------|------------------------------|
| Agent lifecycle management (spawn/pause/resume/kill) | Memory system (Funes) |
| Model gateway (multi-vendor LLM routing) | CLI/tool catalog (KAVAK internal CLIs) |
| Scheduler (heartbeats, crons) | ATH / human escalation (Claudia) |
| Runtime abstraction (Docker + Process) | Agent behavior / prompts (markdown files) |
| Credential injection (multi-provider) | Customer data / APIs (existing KAVAK systems) |
| Self-improvement tools (workspace modification) | Eval framework (Phase 2) |
| Channel routing (message -> agent mapping) | |
| Security (isolation, mount control) | |

## Component Map

```
┌─────────────────────────────────────────────────────────────────┐
│                       KAVAKCLAW HARNESS                         │
│                                                                 │
│  ┌──────────────┐  ┌────────────┐  ┌───────────┐  ┌─────────┐ │
│  │ Agent        │  │ Model      │  │ Scheduler │  │ Channel │ │
│  │ Manager      │  │ Gateway    │  │           │  │ Router  │ │
│  │              │  │ (pi-mono)  │  │           │  │         │ │
│  │ spawn()      │  │            │  │ heartbeat │  │ route() │ │
│  │ pause()      │  │ Anthropic  │  │ cron      │  │ send()  │ │
│  │ resume()     │  │ OpenAI     │  │ adjust()  │  │         │ │
│  │ kill()       │  │ Google     │  │           │  │ Slack   │ │
│  │ scale()      │  │ Mistral    │  │           │  │ WA      │ │
│  │              │  │ Bedrock    │  │           │  │ API     │ │
│  └──────┬───────┘  │ +20 more   │  └─────┬─────┘  └────┬────┘ │
│         │          └──────┬─────┘        │              │      │
│         │                 │              │              │      │
│  ┌──────▼─────────────────▼──────────────▼──────────────▼────┐ │
│  │                 Agent Runtime Abstraction                  │ │
│  │                                                            │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌───────────┐ │ │
│  │  │  DockerRuntime   │  │  ProcessRuntime  │  │ Lambda    │ │ │
│  │  │  (Mac mini)      │  │  (AWS ECS)       │  │ (future)  │ │ │
│  │  │                  │  │                  │  │           │ │ │
│  │  │  • Container per │  │  • Subprocess    │  │ • Function│ │ │
│  │  │    agent         │  │    per agent     │  │   per msg │ │ │
│  │  │  • Volume mounts │  │  • Temp dirs     │  │           │ │ │
│  │  │  • Credential    │  │  • Secrets Mgr   │  │           │ │ │
│  │  │    proxy         │  │  • IAM roles     │  │           │ │ │
│  │  └─────────────────┘  └─────────────────┘  └───────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└───────────────────┬───────────────┬───────────────┬─────────────┘
                    │               │               │
              ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
              │  Agent A   │  │  Agent B   │  │  Agent C   │
              │  Concierge │  │  City CEO  │  │  Prod Mgr  │
              │            │  │            │  │            │
              │  SOUL.md   │  │  SOUL.md   │  │  SOUL.md   │
              │  AGENTS.md │  │  AGENTS.md │  │  AGENTS.md │
              │  skills/   │  │  skills/   │  │  skills/   │
              │  memory/   │  │  memory/   │  │  memory/   │
              └────────────┘  └────────────┘  └────────────┘
```

## The Three Deployments (from v4)

The harness supports all three concurrently:

### Personal Concierge (Claudia)
- **Scale:** Potentially millions (one per customer)
- **Heartbeat:** Variable. Active purchase = every 15 min. Dormant = every week.
- **Workspace:** Customer-specific SOUL.md, interaction history, preferences
- **Tools:** Pricing CLI, inventory CLI, financing CLI, scheduling CLI
- **Channel:** WhatsApp, SMS, voice (via Claudia)

### City CEO
- **Scale:** Tens (one per city/region)
- **Heartbeat:** Every 15 min (always active)
- **Workspace:** City-specific SOUL.md, P&L data, inventory data
- **Tools:** Analytics CLI, pricing CLI, acquisition CLI, HR CLI
- **Channel:** Slack, dashboards, email

### Production Manager
- **Scale:** One (or one per facility)
- **Heartbeat:** Continuous
- **Workspace:** Operational SOUL.md, system access
- **Tools:** All operational CLIs, monitoring, deployment
- **Channel:** Slack, PagerDuty, internal APIs
