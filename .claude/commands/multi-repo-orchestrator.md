---
name: multi-repo-orchestrator
description: "Grand Orchestrator — single entry point for multi-repo task management. Analyzes your request, routes to the right repos, dispatches parallel workers, and aggregates results."
model: opus
argument-hint: "[describe what you want done across your repos]"
allowed-tools: Agent, Read, Bash, Grep, Glob
---

# Multi-Repo Orchestrator Command

You are the entry point for the Grand Orchestrator system. The user describes what they want done in natural language, and you handle everything.

## Your Flow

### Step 1: Understand the Request
Read the user's request: $ARGUMENTS

If no arguments were provided, ask the user what they'd like done.

### Step 2: Dispatch to the Orchestrator Agent
Use the Agent tool to launch the `multi-repo-orchestrator` agent:

```
Agent(
  subagent_type="multi-repo-orchestrator",
  description="Orchestrate: [brief summary of request]",
  prompt="The user wants: [full request]

  Analyze this request, consult your repo-registry, present an execution plan,
  and after user confirmation, dispatch workers to the appropriate repos.

  User's exact words: $ARGUMENTS"
)
```

### Step 3: Report Results
Once the orchestrator agent completes, present the results to the user in a clear summary:
- What was done in each repo
- Any branches created
- Any issues or manual steps needed
- Links to relevant files changed

## Usage Examples

```
/multi-repo-orchestrator Add a /users REST endpoint with CRUD operations, a React form to manage users, and update the API docs
```

```
/multi-repo-orchestrator Fix the auth token expiry bug — it affects both the backend token generation and the frontend token refresh logic
```

```
/multi-repo-orchestrator Update all repos to Node 22 and fix any breaking changes
```
