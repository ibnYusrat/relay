---
name: relay:setup
description: Configure Relay for your project — detect integrations, map codebase, initialize .relay/
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

<objective>
Initialize Relay for the current project. Detects available MCP integrations (Jira, GitHub Issues, Azure DevOps), configures the project, maps the codebase, and creates `.relay/` directory structure.

This replaces the old `/relay:new-project` command with a ticket-driven setup flow.
</objective>

<process>

## 1. Check Existing Setup

```bash
test -f .relay/config.json && echo "exists" || echo "missing"
```

**If `.relay/config.json` exists:**

Read current config and offer options:

```
AskUserQuestion(
  header: "Setup",
  question: "Relay is already configured for this project. What would you like to do?",
  options: [
    { label: "Reconfigure", description: "Update integration settings and refresh config" },
    { label: "Cancel", description: "Keep current configuration" }
  ]
)
```

If "Cancel", STOP.

## 2. Detect Available MCP Integrations

Probe for available MCP tools by checking what's available:

```bash
# Check for Jira MCP tools
echo "Checking integrations..."
```

Check for these MCP tool patterns (use tool availability, not execution):
- `mcp__jira__*` → Jira integration available
- `mcp__github__*` → GitHub Issues integration available
- `mcp__azure_devops__*` → Azure DevOps integration available

Report what was detected.

## 3. Configure Integration

Based on detected MCP tools, ask user to confirm:

```
AskUserQuestion(
  header: "Integration",
  question: "Which ticket system should Relay connect to?",
  options: [
    { label: "Jira", description: "Connect via Jira MCP server" },
    { label: "GitHub Issues", description: "Connect via GitHub MCP server" },
    { label: "Azure DevOps", description: "Connect via Azure DevOps MCP server" },
    { label: "None", description: "Use Relay without external ticket system" }
  ]
)
```

If an integration is selected, ask for the project key:

```
AskUserQuestion(
  header: "Project",
  question: "What is your project key? (e.g., PROJ for Jira, owner/repo for GitHub)",
  options: [
    { label: "Enter key", description: "Provide your project identifier" }
  ]
)
```

## 4. Detect Existing Code

Check if this is a brownfield project:

```bash
# Count source files (exclude node_modules, .git, etc.)
find . -type f \( -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.go" -o -name "*.rs" -o -name "*.java" -o -name "*.rb" -o -name "*.php" -o -name "*.cs" -o -name "*.swift" -o -name "*.kt" \) ! -path "*/node_modules/*" ! -path "*/.git/*" ! -path "*/vendor/*" ! -path "*/dist/*" ! -path "*/build/*" 2>/dev/null | wc -l
```

If source files > 0, offer codebase mapping:

```
AskUserQuestion(
  header: "Codebase",
  question: "Existing code detected. Map the codebase for better analysis?",
  options: [
    { label: "Yes (Recommended)", description: "Run parallel codebase mappers to understand the project" },
    { label: "Skip", description: "Set up without codebase mapping" }
  ]
)
```

## 5. Create `.relay/` Structure

```bash
mkdir -p .relay/tickets
mkdir -p .relay/codebase
mkdir -p .relay/debug
mkdir -p .relay/quick
mkdir -p .relay/todos/pending
```

## 6. Write Config

Write `.relay/config.json` with detected settings:

```json
{
  "mode": "interactive",
  "depth": "standard",
  "integration": {
    "type": "{jira|github|azure_devops|null}",
    "project_key": "{project_key|null}",
    "mcp_server": "{mcp_server_name|null}"
  },
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  },
  "planning": {
    "commit_docs": true,
    "search_gitignored": false
  },
  "parallelization": {
    "enabled": true,
    "plan_level": true,
    "task_level": false,
    "skip_checkpoints": true,
    "max_concurrent_agents": 3,
    "min_plans_for_parallel": 2
  },
  "gates": {
    "confirm_analysis": true,
    "confirm_plan": true,
    "confirm_execution": true,
    "confirm_sync": true,
    "issues_review": true
  },
  "git": {
    "ticket_branch_template": "{ticket_id}/{slug}"
  },
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  }
}
```

## 7. Write STATE.md

Write `.relay/STATE.md` from template:

```markdown
# Relay State

## Active Ticket

**Ticket:** None
**Branch:** —
**Stage:** —
**Started:** —

## Recent Tickets

| Ticket | Title | Status | Branch | Completed |
|--------|-------|--------|--------|-----------|

## Velocity Metrics

**Completed:** 0 tickets
**Average duration:** —
**Recent trend:** —

## Session Continuity

Last session: {current date}
Stopped at: Initial setup
Resume file: None

## Pending Todos

None yet.

## Blockers/Concerns

None yet.
```

## 8. Run Codebase Mapping (if selected)

If user opted for codebase mapping:

```
SlashCommand("relay:map-codebase")
```

## 9. Display Completion

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► SETUP COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Setting            | Value                          |
|--------------------|--------------------------------|
| Integration        | {type or "None"}               |
| Project Key        | {key or "—"}                   |
| Codebase Mapped    | {Yes/No}                       |

## Next Steps

- `/relay:tickets` — browse available tickets
- `/relay:work <ticket-id>` — start working on a ticket
- `/relay:settings` — adjust workflow settings
```

</process>

<success_criteria>
- [ ] MCP integrations detected
- [ ] Integration type and project key configured
- [ ] `.relay/` directory structure created
- [ ] `config.json` written with integration settings
- [ ] `STATE.md` initialized
- [ ] Codebase mapping offered for brownfield projects
- [ ] User knows next steps (`/relay:tickets` or `/relay:work`)
</success_criteria>
