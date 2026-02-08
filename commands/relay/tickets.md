---
name: relay:tickets
description: Browse and select tickets from your connected ticket system
allowed-tools:
  - Read
  - Bash
  - Grep
  - AskUserQuestion
  - mcp__jira__search
  - mcp__jira__get_issue
  - mcp__github__search_issues
  - mcp__github__get_issue
  - mcp__azure_devops__get_work_items
---

<objective>
Fetch and display tickets from the configured external ticket system (Jira, GitHub Issues, or Azure DevOps) via MCP tools.

Presents a formatted table of available tickets and suggests `/relay:work <id>` to start working.
</objective>

<context>
@.relay/config.json
@.relay/STATE.md
</context>

<process>

## 1. Read Configuration

```bash
cat .relay/config.json 2>/dev/null || echo "missing"
```

If config missing:
```
No Relay configuration found. Run `/relay:setup` first.
```
STOP.

Extract integration settings:
- `integration.type` — jira, github, azure_devops, or null
- `integration.project_key` — project identifier
- `integration.mcp_server` — MCP server name

If `integration.type` is null:
```
No ticket system configured. Run `/relay:setup` to connect one,
or use `/relay:work` directly with a ticket description.
```
STOP.

## 2. Fetch Tickets

Based on integration type, fetch open tickets:

**Jira:**
Use `mcp__jira__search` with JQL:
```
project = {project_key} AND status NOT IN (Done, Closed) ORDER BY priority DESC, updated DESC
```

**GitHub Issues:**
Use `mcp__github__search_issues` with:
```
repo:{project_key} state:open sort:updated-desc
```

**Azure DevOps:**
Use `mcp__azure_devops__get_work_items` with query for open items in the project.

## 3. Display Tickets

Format as a table:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► TICKETS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| ID | Title | Type | Priority | Assignee |
|----|-------|------|----------|----------|
| {id} | {title} | {type} | {priority} | {assignee} |
...

Showing {N} open tickets from {project_key}
```

## 4. Suggest Next Action

```
## Next Steps

`/relay:work <ticket-id>` — start working on a ticket

Example: `/relay:work PROJ-123`
```

</process>

<success_criteria>
- [ ] Config read and integration type detected
- [ ] Tickets fetched from external system via MCP
- [ ] Formatted table displayed
- [ ] User directed to `/relay:work <id>`
</success_criteria>
