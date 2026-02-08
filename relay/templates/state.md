# State Template

Template for `.relay/STATE.md` — the project's living memory.

---

## File Template

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
| — | — | — | — | — |

## Velocity Metrics

**Completed:** 0 tickets
**Average duration:** —
**Recent trend:** —

## Session Continuity

Last session: [YYYY-MM-DD HH:MM]
Stopped at: [Description of last completed action]
Resume file: [Path to .continue-here*.md if exists, otherwise "None"]

## Pending Todos

[From .relay/todos/pending/ — ideas captured during sessions]

None yet.

## Blockers/Concerns

[Issues that affect current work]

None yet.
```

<purpose>

STATE.md is the project's short-term memory spanning all tickets and sessions.

**Problem it solves:** Information is captured in summaries and verifications but not systematically consumed. Sessions start without context.

**Solution:** A single, small file that's:
- Read first in every workflow
- Updated after every significant action
- Contains the active ticket and recent history
- Enables instant session restoration

</purpose>

<lifecycle>

**Creation:** During `/relay:setup`
- Initialize empty state
- Set "No active ticket"

**Reading:** First step of every workflow
- status: Present current state to user
- work: Know if a ticket is in progress
- resume-work: Know what to resume

**Writing:** After every significant action
- work (fetch): Set active ticket, branch, stage
- work (execute): Update stage progress
- work (verify): Record completion, add to recent tickets table
- pause-work: Note paused state

</lifecycle>

<sections>

### Active Ticket
The currently in-progress ticket:
- Ticket ID and title
- Git branch name
- Current stage (FETCHING, ANALYZING, PLANNING, EXECUTING, VERIFYING, SYNCING)
- When work started

### Recent Tickets
History of recently completed tickets for context and velocity tracking.

### Velocity Metrics
Track throughput to understand execution patterns:
- Total tickets completed
- Average duration per ticket
- Recent trend (improving/stable/degrading)

### Session Continuity
Enables instant resumption:
- When was last session
- What was last completed
- Is there a .continue-here file to resume from

### Pending Todos
Ideas captured via /relay:add-todo
- Count of pending todos
- Reference to .relay/todos/pending/

### Blockers/Concerns
Issues that affect current or future work.

</sections>

<size_constraint>

Keep STATE.md under 80 lines.

It's a DIGEST, not an archive. If recent tickets grows too long:
- Keep only last 10 tickets
- Archive older entries

The goal is "read once, know where we are" — if it's too long, that fails.

</size_constraint>
