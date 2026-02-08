---
name: relay:resume
description: Resume work on a specific ticket by ID
argument-hint: <ticket-id>
allowed-tools:
  - Read
  - Bash
  - Glob
  - AskUserQuestion
  - SlashCommand
---

<objective>
Resume work on a specific ticket by ID. Unlike `/relay:resume-work` (which restores the last session), this command takes a ticket ID, detects the last completed stage, and routes to `/relay:work` for continuation.

Thin routing layer — no agents spawned.
</objective>

<context>
@.relay/STATE.md
Ticket ID: $ARGUMENTS
</context>

<process>

## 0. Pre-flight

```bash
test -f .relay/config.json && echo "config exists" || echo "no config"
```

If no config: suggest `/relay:setup` and STOP.

## 1. Resolve Ticket ID

**If $ARGUMENTS provided:** Use as ticket ID.

**If no arguments:** List tickets with existing work and let user pick.

```bash
ls -d .relay/tickets/*/ 2>/dev/null | sed 's|.relay/tickets/||;s|/||'
```

If no ticket directories exist:
```
No ticket work found. Start with:
- `/relay:tickets` — browse available tickets
- `/relay:work <id>` — start working on a ticket
```
STOP.

For each ticket directory, detect status:

```bash
for dir in .relay/tickets/*/; do
  ID=$(basename "$dir")
  STATUS="unknown"
  test -f "$dir/VERIFICATION.md" && STATUS="verified"
  test -f "$dir/SUMMARY.md" -a ! -f "$dir/VERIFICATION.md" && STATUS="executed"
  test -f "$dir/PLAN.md" -a ! -f "$dir/SUMMARY.md" && STATUS="planned"
  test -f "$dir/ANALYSIS.md" -a ! -f "$dir/PLAN.md" && STATUS="analyzed"
  test -f "$dir/ESTIMATE.md" -a ! -f "$dir/ANALYSIS.md" && STATUS="estimated"
  test -f "$dir/.continue-here.md" && STATUS="paused"
  echo "$ID — $STATUS"
done
```

Display as selectable list:

```
AskUserQuestion(
  header: "Resume",
  question: "Which ticket would you like to resume?",
  options: [
    { label: "${ID_1}", description: "${STATUS_1}" },
    { label: "${ID_2}", description: "${STATUS_2}" },
    { label: "${ID_3}", description: "${STATUS_3}" }
  ]
)
```

## 2. Validate Ticket

```bash
test -d .relay/tickets/${TICKET_ID} && echo "exists" || echo "missing"
```

If ticket directory doesn't exist:
```
No work found for ${TICKET_ID}.

- `/relay:work ${TICKET_ID}` — start fresh work on this ticket
- `/relay:estimate ${TICKET_ID}` — estimate before committing
```
STOP.

## 3. Detect Last Stage

Check artifacts to determine progress:

```bash
echo "--- Artifacts for ${TICKET_ID} ---"
test -f .relay/tickets/${TICKET_ID}/VERIFICATION.md && echo "VERIFICATION: exists"
test -f .relay/tickets/${TICKET_ID}/SUMMARY.md && echo "SUMMARY: exists"
test -f .relay/tickets/${TICKET_ID}/PLAN.md && echo "PLAN: exists"
test -f .relay/tickets/${TICKET_ID}/ANALYSIS.md && echo "ANALYSIS: exists"
test -f .relay/tickets/${TICKET_ID}/ESTIMATE.md && echo "ESTIMATE: exists"
test -f .relay/tickets/${TICKET_ID}/.continue-here.md && echo "CONTINUE-HERE: exists"
```

Determine resume point:
- Has VERIFICATION.md → Already complete. Show results and ask if user wants to re-verify or start fresh.
- Has SUMMARY.md (no VERIFICATION) → Resume at VERIFY stage
- Has PLAN.md (no SUMMARY) → Resume at CONFIRM stage
- Has ANALYSIS.md (no PLAN) → Resume at PLAN stage
- Has only ESTIMATE.md → Resume at FETCH stage
- Empty directory → Resume at FETCH stage

## 4. Check Branch

```bash
TICKET_BRANCH=$(git branch --list "*${TICKET_ID}*" 2>/dev/null | head -1 | sed 's/^[* ]*//')
CURRENT_BRANCH=$(git branch --show-current)
```

If a ticket branch exists and we're not on it:
```
AskUserQuestion(
  header: "Branch",
  question: "Switch to ticket branch '${TICKET_BRANCH}'?",
  options: [
    { label: "Switch", description: "Checkout ${TICKET_BRANCH} to continue work" },
    { label: "Stay", description: "Stay on ${CURRENT_BRANCH}" }
  ]
)
```

If "Switch":
```bash
git checkout "${TICKET_BRANCH}"
```

## 5. Load Handoff Context

If `.continue-here.md` exists, read it for context:

```bash
cat .relay/tickets/${TICKET_ID}/.continue-here.md 2>/dev/null
```

Display handoff summary to user:
```
Loaded handoff from previous session:
- Stage: ${STAGE}
- Status: ${STATUS}
- Next action: ${NEXT_ACTION}
```

## 6. Route to Work

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► RESUMING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Last completed: ${STAGE}
Resuming from: ${NEXT_STAGE}
```

Route to `/relay:work ${TICKET_ID}` which handles stage detection and resumption internally.

</process>

<success_criteria>
- [ ] Ticket ID resolved (from argument or user selection)
- [ ] Ticket directory validated
- [ ] Last completed stage detected from artifacts
- [ ] Branch check and optional switch
- [ ] Handoff context loaded if available
- [ ] Routed to `/relay:work` for continuation
</success_criteria>
