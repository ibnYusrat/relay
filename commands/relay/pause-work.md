---
name: relay:pause-work
description: Create context handoff when pausing work on a ticket
allowed-tools:
  - Read
  - Write
  - Bash
---

<objective>
Create `.continue-here.md` handoff file to preserve complete work state across sessions.

Enables seamless resumption in fresh session with full context restoration.
</objective>

<context>
@.relay/STATE.md
</context>

<process>

<step name="detect">
**Find active ticket from STATE.md.**

Read STATE.md and extract active ticket ID. If no active ticket, report:
```
No active ticket found. Nothing to pause.
```
STOP.
</step>

<step name="gather">
**Collect complete state for handoff:**

1. **Current position**: Which ticket, which stage, which task
2. **Work completed**: What got done this session
3. **Work remaining**: What's left in current plan
4. **Decisions made**: Key decisions and rationale
5. **Blockers/issues**: Anything stuck
6. **Mental context**: The approach, next steps
7. **Files modified**: What's changed but not committed

Ask user for clarifications if needed.
</step>

<step name="write">
**Write handoff to `.relay/tickets/${TICKET_ID}/.continue-here.md`:**

```markdown
---
ticket: ${TICKET_ID}
stage: ${CURRENT_STAGE}
status: paused
last_updated: [timestamp]
---

<current_state>
[Where exactly are we? Immediate context]
</current_state>

<completed_work>
- [What was accomplished]
</completed_work>

<remaining_work>
- [What's left to do]
</remaining_work>

<decisions_made>
- [Key decisions and rationale]
</decisions_made>

<blockers>
- [Any blockers or issues]
</blockers>

<context>
[Mental state, the plan, approach being taken]
</context>

<next_action>
Start with: [specific first action when resuming]
</next_action>
```

Be specific enough for a fresh Claude to understand immediately.
</step>

<step name="commit">
**Check planning config:**

```bash
COMMIT_PLANNING_DOCS=$(cat .relay/config.json 2>/dev/null | grep -o '"commit_docs"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "false")
git check-ignore -q .relay 2>/dev/null && COMMIT_PLANNING_DOCS=false
```

**If `COMMIT_PLANNING_DOCS=false` (default):** Skip git operations

**If `COMMIT_PLANNING_DOCS=true`:**

```bash
git add .relay/tickets/${TICKET_ID}/.continue-here.md
git commit -m "wip(${TICKET_ID}): paused work"
```
</step>

<step name="confirm">
```
Handoff created: .relay/tickets/${TICKET_ID}/.continue-here.md

Current state:
- Ticket: ${TICKET_ID}
- Stage: ${CURRENT_STAGE}
- Status: paused

To resume: /relay:resume-work
```
</step>

</process>

<success_criteria>
- [ ] Active ticket identified from STATE.md
- [ ] .continue-here.md created in ticket directory
- [ ] All sections filled with specific content
- [ ] Committed as WIP
- [ ] User knows how to resume
</success_criteria>
