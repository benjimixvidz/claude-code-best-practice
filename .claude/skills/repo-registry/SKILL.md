---
name: repo-registry
description: "Registry of all managed repositories with their paths, tech stacks, conventions, and routing rules. Preloaded into the multi-repo orchestrator agent for task routing."
user-invocable: false
---

# Repo Registry

You are the knowledge base for the multi-repo orchestrator. This skill contains the registry of all managed repositories, their metadata, and routing rules.

## How to Use This Registry

When the orchestrator receives a user request:
1. Parse the intent (what needs to be done)
2. Match against repo `domains` and `keywords` below
3. Identify which repo(s) are involved
4. Return the routing plan with repo paths, relevant context, and execution order

## Registry Format

Each repo entry follows this structure:

```yaml
- id: unique-short-name
  path: /absolute/path/to/repo
  description: What this repo does
  stack: [tech1, tech2, tech3]
  domains: [area1, area2]         # Business domains this repo covers
  keywords: [keyword1, keyword2]  # Trigger words for routing
  branch_convention: main|develop # Default branch
  test_command: npm test          # How to run tests
  build_command: npm run build    # How to build
  has_claude_md: true|false       # Whether repo has its own CLAUDE.md
  dependencies: [other-repo-id]   # Cross-repo dependencies
```

## Registered Repositories

> **IMPORTANT**: Update this section with your actual repositories.
> Below is a template with example entries. Replace with your real repos.

```yaml
repositories:

  # ── Example: Monorepo with multiple packages ──────────────
  - id: main-app
    path: /home/user/projects/main-app
    description: "Main application monorepo (frontend + backend + shared)"
    stack: [typescript, react, node, postgresql]
    domains: [frontend, backend, api, database]
    keywords: [ui, component, page, endpoint, route, api, query, migration]
    branch_convention: main
    test_command: pnpm test
    build_command: pnpm build
    has_claude_md: true
    dependencies: []
    packages:
      - name: frontend
        path: packages/frontend
        domains: [frontend, ui]
        keywords: [component, page, form, button, modal, style, css]
      - name: backend
        path: packages/backend
        domains: [backend, api]
        keywords: [endpoint, route, controller, middleware, auth]
      - name: shared
        path: packages/shared
        domains: [shared, types]
        keywords: [type, interface, util, helper, constant]

  # ── Example: Infrastructure repo ──────────────────────────
  - id: infra
    path: /home/user/projects/infra
    description: "Infrastructure as code (Terraform, Docker, CI/CD)"
    stack: [terraform, docker, github-actions]
    domains: [devops, infrastructure, deployment]
    keywords: [deploy, ci, cd, pipeline, docker, terraform, k8s, env, secret]
    branch_convention: main
    test_command: terraform validate
    build_command: terraform plan
    has_claude_md: false
    dependencies: [main-app]

  # ── Example: Documentation site ───────────────────────────
  - id: docs
    path: /home/user/projects/docs
    description: "Public documentation site"
    stack: [markdown, docusaurus, react]
    domains: [documentation, content]
    keywords: [doc, guide, tutorial, readme, changelog, api-docs]
    branch_convention: main
    test_command: npm run build
    build_command: npm run build
    has_claude_md: false
    dependencies: [main-app]
```

## Routing Rules

### Single-repo tasks
When a task clearly maps to ONE repo, route directly:
- "Fix the login button" → `main-app` (frontend package)
- "Update the deployment pipeline" → `infra`
- "Add API docs for /users" → `docs`

### Cross-repo tasks
When a task spans MULTIPLE repos, define execution order based on `dependencies`:
- "Add a /users endpoint with frontend form and docs"
  1. `main-app/backend` — Create the API endpoint (upstream first)
  2. `main-app/frontend` — Create the form consuming the endpoint
  3. `docs` — Document the new endpoint

### Ambiguous tasks
When routing is unclear, the orchestrator MUST ask the user to clarify before dispatching.

## Cross-Repo Conventions

These conventions apply across ALL registered repos:
- **Branch naming**: `feature/<description>`, `fix/<description>`, `chore/<description>`
- **Commit style**: Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`)
- **PR format**: Title < 70 chars, body with ## Summary and ## Test Plan
- **Testing**: Always run repo's test_command after changes
- **No force push**: Never force-push to default branch
