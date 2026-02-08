---
name: relay:status
description: Show current Relay status — active ticket, recent work, and next actions
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
  - SlashCommand
---

<objective>
Show the current state of Relay for this project: active ticket progress, recent tickets, and routing to the next action.

Provides situational awareness before continuing work.
</objective>

<process>

<step name="verify">
**Verify Relay structure exists:**

```bash
test -d .relay && echo "exists" || echo "missing"
```

If no `.relay/` directory:

```
No Relay setup found.

Run /relay:setup to initialize Relay for this project.
```

STOP.

If missing STATE.md: suggest `/relay:setup`.
</step>

<step name="load">
**Load project context:**

- Read `.relay/STATE.md` for current state
- Read `.relay/config.json` for settings
</step>

<step name="active_ticket">
**Check active ticket:**

From STATE.md, extract:
- Active ticket ID and title
- Current stage (FETCHING, ANALYZING, PLANNING, EXECUTING, VERIFYING, SYNCING)
- Branch name
- When started

If active ticket exists, read its artifacts:
```bash
ls .relay/tickets/${TICKET_ID}/ 2>/dev/null
```
</step>

<step name="recent">
**Gather recent work:**

- Read recent tickets table from STATE.md
- Count pending todos: `ls .relay/todos/pending/*.md 2>/dev/null | wc -l`
- Check for active debug sessions: `ls .relay/debug/*.md 2>/dev/null | grep -v resolved | wc -l`
</step>

<step name="report">
**Present status report:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► STATUS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Active Ticket

{TICKET_ID}: {title}
Stage: {ANALYZING | PLANNING | EXECUTING | VERIFYING}
Branch: {branch_name}
Started: {date}

## Recent Tickets

| Ticket | Title | Status | Completed |
|--------|-------|--------|-----------|
| {id} | {title} | {done/verified} | {date} |

## Velocity

Completed: {N} tickets
Average: {duration}

## Pending Todos

{count} pending — /relay:check-todos to review

## Active Debug Sessions

{count} active — /relay:debug to continue
(Only show this section if count > 0)
```
</step>

<step name="route">
**Route to next action:**

| State | Suggestion |
|-------|-----------|
| Active ticket in progress | `/relay:work {ID}` to resume |
| No active ticket | `/relay:tickets` to browse, `/relay:work <id>` to start |
| Paused ticket (.continue-here exists) | `/relay:resume-work` to continue |

```
## Next Steps

{context-appropriate suggestion}
```
</step>

</process>

<success_criteria>
- [ ] Current state loaded from STATE.md
- [ ] Active ticket shown with stage
- [ ] Recent tickets displayed
- [ ] Smart routing to next action
- [ ] User knows what to do next
</success_criteria>
