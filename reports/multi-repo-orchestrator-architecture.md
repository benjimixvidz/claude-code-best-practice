# Multi-Repo Orchestrator Architecture

> A "Grand Orchestrator" pattern for Claude Code — a single entry point that routes tasks to the right repository, dispatches parallel workers, and aggregates results.

## The Problem

When working across multiple repositories (e.g., a backend, frontend, infra, and docs repo), you typically need to:
- Manually switch between repos
- Keep context about each repo's conventions
- Coordinate cross-repo changes (e.g., API contracts)
- Run tests in each repo separately

**The Grand Orchestrator solves this** by giving you a single conversational interface that handles all of this automatically.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  USER                                                           │
│  "Add a /users endpoint, a React form, and update the docs"    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  /multi-repo-orchestrator  (Command)                            │
│  Entry point — receives user request, launches orchestrator     │
│  Model: opus (needs strong reasoning for task decomposition)    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Agent tool
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  multi-repo-orchestrator  (Agent)                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  repo-registry (Preloaded Skill)                        │    │
│  │  - Repo paths, stacks, domains, keywords                │    │
│  │  - Routing rules and dependency graph                   │    │
│  │  - Cross-repo conventions                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Phase 1: Analyze → Route → Plan                                │
│  Phase 2: Dispatch workers (parallel when possible)             │
│  Phase 3: Aggregate results → Report                            │
└────────┬──────────────┬──────────────┬──────────────────────────┘
         │              │              │
         ▼              ▼              ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Worker A │  │ Worker B │  │ Worker C │
   │ (worktree)│  │ (worktree)│  │ (worktree)│
   │ backend  │  │ frontend │  │ docs     │
   │          │  │          │  │          │
   │ 1. cd    │  │ 1. cd    │  │ 1. cd    │
   │ 2. read  │  │ 2. read  │  │ 2. read  │
   │ 3. edit  │  │ 3. edit  │  │ 3. edit  │
   │ 4. test  │  │ 4. test  │  │ 4. test  │
   │ 5. commit│  │ 5. commit│  │ 5. commit│
   └──────────┘  └──────────┘  └──────────┘
```

## Component Breakdown

### 1. Command: `/multi-repo-orchestrator`

**File**: `.claude/commands/multi-repo-orchestrator.md`

The user-facing entry point. Invoked via `/multi-repo-orchestrator [request]`.
- Receives natural language request
- Dispatches to the orchestrator agent
- Reports results back to user

**Why a Command?** Commands are the recommended entry point for workflows (per CLAUDE.md best practices). They provide argument handling, model selection, and tool allowlists.

### 2. Agent: `multi-repo-orchestrator`

**File**: `.claude/agents/multi-repo-orchestrator.md`

The brain of the system. Has the `repo-registry` skill preloaded for instant routing decisions.
- **Model**: inherits from command (opus for reasoning)
- **Preloaded skill**: `repo-registry` (always in context)
- **Key capability**: Can launch other agents via the Agent tool

**Why an Agent, not just a Command?** The Agent pattern provides:
- Preloaded skills (repo-registry always available)
- Isolated context (doesn't pollute main conversation)
- Color-coded output (cyan for visual distinction)
- Configurable model and tool restrictions

### 3. Skill: `repo-registry`

**File**: `.claude/skills/repo-registry/SKILL.md`

The knowledge base — contains all repo metadata and routing rules.
- **Not user-invocable** (`user-invocable: false`)
- **Preloaded into agent** via `skills:` frontmatter
- **User-maintained** — update when repos are added/removed

**Why a Skill, not hardcoded in the Agent?** Skills follow the progressive disclosure pattern:
- Descriptions loaded at startup (lightweight)
- Full content loaded on-demand (or preloaded for agents)
- Can be updated independently of the agent definition
- Reusable by other agents if needed

### 4. Workers: General-purpose Agents with Worktree Isolation

Workers are ephemeral — launched per-task via the Agent tool with `isolation: "worktree"`.
- Each worker gets an isolated copy of the repo
- No file conflicts between parallel workers
- Automatic cleanup if no changes are made

## Key Constraints & Design Decisions

### Constraint: No Agent Nesting
Claude Code subagents **cannot spawn other subagents**. This is why:
- The **Command** launches the **Orchestrator Agent**
- The **Orchestrator Agent** launches **Workers** (general-purpose agents)
- **Workers** do the actual work — they cannot delegate further

This means the hierarchy is exactly 2 levels deep:
```
Command → Orchestrator Agent → Worker Agents (leaf nodes)
```

### Constraint: Single CWD per Session
Each Claude Code session has one working directory. Workers overcome this by:
- Using `cd` to navigate to the target repo
- Running in worktree isolation for safe parallel execution

### Constraint: Context Window
Multi-repo work is context-heavy. Mitigations:
- Workers run in **isolated contexts** (don't pollute the orchestrator)
- The repo-registry uses **structured metadata** (not full repo contents)
- Workers report **summaries**, not full diffs
- Orchestrator compacts at 80% (per settings.json)

## Usage Examples

### Single-repo task
```
/multi-repo-orchestrator Fix the auth middleware to validate JWT expiry
```
→ Routes to `main-app/backend`, single worker

### Cross-repo task
```
/multi-repo-orchestrator Add a /users CRUD endpoint with a React admin form and API docs
```
→ Sequential: backend worker (creates API) → parallel: frontend worker + docs worker

### Infrastructure task
```
/multi-repo-orchestrator Update the CI pipeline to run on Node 22 and fix any breaking changes in all repos
```
→ Parallel workers across all repos with Node dependency

## Customization

### Adding a New Repo
Edit `.claude/skills/repo-registry/SKILL.md` and add an entry to the `repositories` list:
```yaml
  - id: my-new-repo
    path: /home/user/projects/my-new-repo
    description: "What this repo does"
    stack: [python, fastapi, postgresql]
    domains: [backend, ml]
    keywords: [model, predict, train, inference]
    branch_convention: main
    test_command: pytest
    build_command: docker build .
    has_claude_md: true
    dependencies: []
```

### Changing Routing Behavior
Modify the "Routing Rules" section in the repo-registry skill to adjust how ambiguous requests are handled.

### Cross-Repo Conventions
Update the "Cross-Repo Conventions" section to enforce team standards (commit style, branch naming, etc.).

## Comparison with Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **This Orchestrator** | Single interface, parallel dispatch, structured routing | 2-level nesting limit, context overhead |
| **Agent Teams** (experimental) | True multi-session, inter-agent messaging | Experimental, higher token cost, complex setup |
| **Agent SDK** (programmatic) | Full control, CI/CD integration, no nesting limits | Requires code, not interactive |
| **Manual switching** | Simple, no setup | Tedious, error-prone, no coordination |

## Relationship to Anthropic's Skill Guidelines

This architecture follows Anthropic's recommended patterns:
1. **Command as entry point** — not standalone agents
2. **Progressive disclosure** — repo knowledge in a skill, loaded on demand
3. **Feature-specific agents** — orchestrator has one job
4. **Worktree isolation** — safe parallel execution
5. **Human-in-the-loop** — orchestrator asks for plan confirmation before dispatching
