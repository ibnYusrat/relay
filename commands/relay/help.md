---
name: relay:help
description: Show available Relay commands and usage guide
---

<objective>
Display the complete Relay command reference.

Output ONLY the reference content below. Do NOT add:

- Project-specific analysis
- Git status or file context
- Next-step suggestions
- Any commentary beyond the reference
</objective>

<reference>
# Relay Command Reference

**Relay** — your AI dev agent, connected to your team's workflow.

## Quick Start

1. `/relay:setup` — Connect to Jira/GitHub Issues/Azure DevOps
2. `/relay:tickets` — Browse available tickets
3. `/relay:work PROJ-123` — Fetch, analyze, plan, execute, verify, sync

## Staying Updated

Relay evolves fast. Update periodically:

```bash
npx relay-cc@latest
```

## Core Workflow

```
/relay:setup → /relay:tickets → /relay:work <id> → repeat
```

### Setup & Configuration

**`/relay:setup`**
Initialize Relay for this project.

- Detects available MCP integrations (Jira, GitHub, Azure DevOps)
- Configures project key and integration type
- Maps existing codebase (optional)
- Creates `.relay/` directory structure

Usage: `/relay:setup`

**`/relay:map-codebase`**
Map an existing codebase for better ticket analysis.

- Analyzes codebase with parallel Explore agents
- Creates `.relay/codebase/` with 7 focused documents
- Covers stack, architecture, structure, conventions, testing, integrations, concerns

Usage: `/relay:map-codebase`

### Ticket Workflow

**`/relay:tickets`**
Browse tickets from your connected ticket system.

- Fetches open tickets via MCP tools
- Displays formatted table with ID, title, type, priority
- Suggests `/relay:work <id>` to start working

Usage: `/relay:tickets`

**`/relay:work <ticket-id>`**
The main orchestrator — 6 stages:

1. **FETCH** — Get ticket details, create git branch
2. **ANALYZE** — Spawn ticket researcher, produce ANALYSIS.md
3. **PLAN** — Spawn planner, produce PLAN.md
4. **CONFIRM** — Present analysis + plan for user approval
5. **EXECUTE** — Spawn executor, atomic commits with ticket ID
6. **VERIFY & SYNC** — Check acceptance criteria, post results back

Supports resumption — detects last completed stage and continues.

Usage: `/relay:work PROJ-123`

**`/relay:estimate <ticket-id>`**
Estimate effort before committing to work.

- Analyzes ticket against actual codebase
- Produces complexity score (Fibonacci 1/2/3/5/8/13), time estimate, impact analysis
- Does NOT create a branch or set active ticket
- Creates `.relay/tickets/{ID}/ESTIMATE.md`

Usage: `/relay:estimate PROJ-123`

**`/relay:resume <ticket-id>`**
Resume work on a specific ticket by ID.

- Detects last completed stage from artifacts
- Checks for ticket branch and offers to switch
- Loads `.continue-here.md` handoff if present
- Routes to `/relay:work` for continuation

Usage: `/relay:resume PROJ-123`

**`/relay:pr [ticket-id]`**
Create a pull request from completed ticket work.

- Auto-generates title and description from ticket artifacts
- Shows preview before creating
- Pushes branch and creates PR via `gh pr create`
- Optionally posts PR link back to ticket system

Usage: `/relay:pr PROJ-123`

### Quick Mode

**`/relay:quick`**
Execute small, ad-hoc tasks with Relay guarantees but skip analysis.

- Spawns planner + executor (skips researcher, checker, verifier)
- Quick tasks live in `.relay/quick/` separate from ticket work
- Updates STATE.md tracking

Usage: `/relay:quick`

### Status & Progress

**`/relay:status`**
Check current state and route to next action.

- Shows active ticket with stage progress
- Displays recent tickets and velocity
- Smart routing to next action

Usage: `/relay:status`

### Code Review & PRs

**`/relay:review`**
Review a PR or address review comments on your own PR.

- **Mode A — Review a PR:** Spawns reviewer agent, produces structured review with approval recommendation
- **Mode B — Address comments:** Fetches review comments, makes code fixes, commits

Usage: `/relay:review`

**`/relay:rollback <ticket-id>`**
Safely revert work done on a specific ticket.

- Uses `git revert` (creates new commits, no history rewriting)
- Supports full revert, selective revert, or soft rollback
- Handles artifacts (archive, delete, or keep)
- Requires explicit confirmation

Usage: `/relay:rollback PROJ-123`

### Session Management

**`/relay:resume-work`**
Resume work from previous session with full context restoration.

Usage: `/relay:resume-work`

**`/relay:pause-work`**
Create context handoff when pausing work on a ticket.

Usage: `/relay:pause-work`

### Debugging

**`/relay:debug [issue description]`**
Systematic debugging with persistent state across context resets.

- Gathers symptoms through adaptive questioning
- Investigates using scientific method
- Survives `/clear` — run `/relay:debug` with no args to resume

Usage: `/relay:debug "login button doesn't work"`

### History

**`/relay:history [filter]`**
Show completed ticket history with outcomes.

- Scans all ticket directories for artifacts
- Displays table with ticket, title, status, commits, date
- Shows summary stats
- Drill-down into individual tickets for full details

Usage: `/relay:history`

### Todo Management

**`/relay:add-todo [description]`**
Capture idea or task as todo from current conversation.

Usage: `/relay:add-todo Add auth token refresh`

**`/relay:check-todos [area]`**
List pending todos and select one to work on.

Usage: `/relay:check-todos`

### Configuration

**`/relay:settings`**
Configure workflow toggles, model profile, and integration settings.

Usage: `/relay:settings`

**`/relay:set-profile <profile>`**
Quick switch model profile.

- `quality` — Opus everywhere
- `balanced` — Opus for planning, Sonnet for execution (default)
- `budget` — Sonnet for writing, Haiku for research

Usage: `/relay:set-profile budget`

**`/relay:update`**
Update Relay to latest version with changelog preview.

Usage: `/relay:update`

## Files & Structure

```
.relay/
├── config.json           # Integration config, workflow settings
├── STATE.md              # Active ticket, recent history, velocity
├── codebase/             # Codebase map (7 documents)
├── tickets/
│   └── PROJ-123/
│       ├── ESTIMATE.md   # Pre-work effort estimate
│       ├── ANALYSIS.md   # Codebase-aware ticket analysis
│       ├── PLAN.md       # Executable plan
│       ├── SUMMARY.md    # Execution results
│       └── VERIFICATION.md  # Acceptance criteria check
├── reviews/              # PR review documents
├── debug/                # Debug sessions
├── quick/                # Quick task artifacts
└── todos/                # Captured ideas
    ├── pending/
    └── done/
```

## Common Workflows

**Starting with Relay:**

```
/relay:setup              # Connect integration, map codebase
/relay:tickets            # Browse available tickets
/relay:work PROJ-123      # Work on a ticket
```

**Estimating before committing:**

```
/relay:estimate PROJ-123  # Get complexity, time, impact analysis
/relay:work PROJ-123      # Start if estimate looks good
```

**Resuming a specific ticket:**

```
/relay:resume PROJ-123    # Detect stage, switch branch, continue
```

**Resuming work after a break:**

```
/relay:status             # See where you left off
/relay:work PROJ-123      # Resume (auto-detects stage)
```

**Quick ad-hoc task:**

```
/relay:quick              # Plan + execute without ticket overhead
```

**Creating a PR after work is done:**

```
/relay:pr PROJ-123        # Auto-generate title/body, push, create PR
```

**Reviewing code:**

```
/relay:review             # Review a PR or address comments on yours
```

**Checking history:**

```
/relay:history            # See all ticket work with outcomes
```

**Debugging an issue:**

```
/relay:debug "form fails"   # Start debug session
/clear
/relay:debug                # Resume investigation
```

## Planning Configuration

Configure in `.relay/config.json`:

**`planning.commit_docs`** (default: `false`)
- `true`: Planning artifacts committed to git
- `false`: Planning artifacts kept local-only

**`planning.search_gitignored`** (default: `false`)
- `true`: Include `.relay/` in project-wide searches

## Getting Help

- Read `.relay/STATE.md` for current context
- Read `.relay/config.json` for settings
- Run `/relay:status` to check where you're at
</reference>
