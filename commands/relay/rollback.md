---
name: relay:rollback
description: Safely revert work done on a specific ticket
argument-hint: <ticket-id>
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Safely revert ticket work using `git revert` (creates new commits — no history rewriting).

Destructive action — requires explicit user confirmation before executing.
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

If no $ARGUMENTS:
```
AskUserQuestion(
  header: "Ticket",
  question: "Which ticket's work would you like to rollback?",
  options: [
    { label: "Enter ID", description: "Type a ticket ID (e.g., PROJ-123)" },
    { label: "Cancel", description: "Don't rollback anything" }
  ]
)
```

## 1. Find Ticket Commits

```bash
git log --oneline --all --grep="${TICKET_ID}"
```

If no commits found:
```
No commits found for ${TICKET_ID}.
```
STOP.

## 2. Display Commits and Files

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► ROLLBACK — ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Show commits:
```bash
git log --oneline --all --grep="${TICKET_ID}" --format="%h %s (%ai)"
```

Show files affected:
```bash
git log --all --grep="${TICKET_ID}" --name-only --format="" | sort -u
```

Count total:
```bash
COMMIT_COUNT=$(git log --oneline --all --grep="${TICKET_ID}" | wc -l | tr -d ' ')
FILE_COUNT=$(git log --all --grep="${TICKET_ID}" --name-only --format="" | sort -u | wc -l | tr -d ' ')
```

```
Found ${COMMIT_COUNT} commits affecting ${FILE_COUNT} files.
```

## 3. Check for Interleaved Commits

Check if other work is interleaved with ticket commits:

```bash
# Get the range of ticket commits
FIRST_COMMIT=$(git log --oneline --all --grep="${TICKET_ID}" --format="%H" | tail -1)
LAST_COMMIT=$(git log --oneline --all --grep="${TICKET_ID}" --format="%H" | head -1)

# Check for non-ticket commits in that range
git log --oneline ${FIRST_COMMIT}..${LAST_COMMIT} | grep -v "${TICKET_ID}"
```

If interleaved commits exist, warn:
```
WARNING: Other commits exist between ${TICKET_ID} commits.
Reverting may cause conflicts. Consider selective revert.
```

## 4. Choose Rollback Mode

```
AskUserQuestion(
  header: "Mode",
  question: "How would you like to rollback ${TICKET_ID}? (${COMMIT_COUNT} commits, ${FILE_COUNT} files)",
  options: [
    { label: "Full revert", description: "Revert all ${COMMIT_COUNT} commits for ${TICKET_ID}" },
    { label: "Selective revert", description: "Choose which commits to revert" },
    { label: "Soft rollback", description: "Just switch to main branch (no revert commits)" },
    { label: "Cancel", description: "Don't rollback" }
  ]
)
```

If "Cancel": STOP.

### Soft Rollback

```bash
BASE_BRANCH=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | sed 's/.*: //')
BASE_BRANCH=${BASE_BRANCH:-main}
git checkout "${BASE_BRANCH}"
```

Skip to step 6.

### Selective Revert

Display commits numbered for selection:
```bash
git log --oneline --all --grep="${TICKET_ID}" --format="%h %s"
```

Ask user which commits to revert (by hash or number).

### Full Revert / Selective Revert

Continue to step 5.

## 5. Confirm and Execute

**Explicit confirmation required:**

```
AskUserQuestion(
  header: "Confirm",
  question: "This will create revert commits for ${REVERT_COUNT} commits. This cannot be easily undone. Proceed?",
  options: [
    { label: "Revert", description: "Execute git revert for selected commits" },
    { label: "Cancel", description: "Abort rollback" }
  ]
)
```

If "Cancel": STOP.

**Execute reverts (newest first):**

```bash
# Get commits in reverse chronological order (newest first)
COMMITS=$(git log --oneline --all --grep="${TICKET_ID}" --format="%H")

for COMMIT in ${COMMITS}; do
  git revert --no-edit ${COMMIT} || {
    echo "Conflict during revert of ${COMMIT}"
    echo "Resolve conflicts and run: git revert --continue"
    break
  }
done
```

If conflicts occur, display:
```
Conflict encountered while reverting ${COMMIT_HASH}.

Conflicting files:
$(git diff --name-only --diff-filter=U)

Options:
1. Resolve conflicts manually, then run: git revert --continue
2. Abort the rollback: git revert --abort
```

## 6. Handle Artifacts

```
AskUserQuestion(
  header: "Artifacts",
  question: "What should happen to the ticket artifacts in .relay/tickets/${TICKET_ID}/?",
  options: [
    { label: "Archive", description: "Move to .relay/tickets/${TICKET_ID}/.archived/" },
    { label: "Delete", description: "Remove all ticket artifacts" },
    { label: "Keep", description: "Leave artifacts as-is" }
  ]
)
```

**Archive:**
```bash
mkdir -p .relay/tickets/${TICKET_ID}/.archived
mv .relay/tickets/${TICKET_ID}/*.md .relay/tickets/${TICKET_ID}/.archived/ 2>/dev/null
```

**Delete:**
```bash
rm -rf .relay/tickets/${TICKET_ID}
```

**Keep:** No action.

## 7. Update State

Update STATE.md:
- Clear active ticket if it was the rolled-back ticket
- Add note to recent tickets: "reverted"

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► ROLLBACK COMPLETE — ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reverted: ${REVERT_COUNT} commits
Artifacts: ${archived/deleted/kept}

## Next Steps

- `/relay:tickets` — browse available tickets
- `/relay:status` — check current state
```

</process>

<success_criteria>
- [ ] Ticket commits found and displayed
- [ ] Interleaved commits warned about
- [ ] User chose rollback mode
- [ ] Explicit confirmation obtained before revert
- [ ] `git revert --no-edit` executed for selected commits (newest first)
- [ ] Conflicts handled gracefully if they occur
- [ ] Artifacts archived, deleted, or kept per user choice
- [ ] STATE.md updated (active ticket cleared, marked as reverted)
</success_criteria>
