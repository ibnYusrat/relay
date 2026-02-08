---
name: relay:history
description: Show completed ticket history with outcomes
argument-hint: [filter]
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Display completed ticket history with status, commit counts, and outcomes.

Read-only — no modifications to any files.
</objective>

<context>
@.relay/STATE.md
Filter: $ARGUMENTS
</context>

<process>

## 0. Pre-flight

```bash
test -d .relay/tickets && echo "tickets exist" || echo "no tickets"
```

If no `.relay/tickets/` directory:
```
No ticket history found.

Run /relay:setup to initialize Relay, then /relay:work <id> to start.
```
STOP.

## 1. Scan Ticket Directories

```bash
ls -d .relay/tickets/*/ 2>/dev/null | sed 's|.relay/tickets/||;s|/||'
```

If no ticket directories:
```
No ticket work found yet.

- `/relay:tickets` — browse available tickets
- `/relay:work <id>` — start working on a ticket
```
STOP.

## 2. Gather Ticket Data

For each ticket directory, collect:

```bash
for dir in .relay/tickets/*/; do
  ID=$(basename "$dir")

  # Determine status from artifacts
  STATUS="unknown"
  test -f "$dir/VERIFICATION.md" && STATUS="verified"
  test -f "$dir/SUMMARY.md" -a ! -f "$dir/VERIFICATION.md" && STATUS="completed"
  test -f "$dir/PLAN.md" -a ! -f "$dir/SUMMARY.md" && STATUS="in-progress"
  test -f "$dir/ANALYSIS.md" -a ! -f "$dir/PLAN.md" && STATUS="analyzed"
  test -f "$dir/ESTIMATE.md" -a ! -f "$dir/ANALYSIS.md" && STATUS="estimated"

  # Get title from ANALYSIS.md if available
  TITLE=""
  if [ -f "$dir/ANALYSIS.md" ]; then
    TITLE=$(grep -m1 "^##\? .*Title" "$dir/ANALYSIS.md" | sed 's/.*Title[:**]* *//')
  fi

  # Get commit count
  COMMITS=$(git log --oneline --all --grep="$ID" 2>/dev/null | wc -l | tr -d ' ')

  # Get branch
  BRANCH=$(git branch --list "*${ID}*" 2>/dev/null | head -1 | sed 's/^[* ]*//')

  # Get date from most recent artifact
  DATE=""
  LATEST=$(ls -t "$dir"/*.md 2>/dev/null | head -1)
  if [ -n "$LATEST" ]; then
    DATE=$(date -r "$LATEST" "+%Y-%m-%d" 2>/dev/null || stat -c "%y" "$LATEST" 2>/dev/null | cut -d' ' -f1)
  fi

  echo "${ID}|${TITLE}|${STATUS}|${COMMITS}|${BRANCH}|${DATE}"
done
```

## 3. Apply Filters

If $ARGUMENTS provided, filter results:

- **Status filter:** `verified`, `completed`, `in-progress`, `analyzed`, `estimated`
- **Keyword filter:** Match against ticket ID or title
- **Date filter:** `--since YYYY-MM-DD` or `--last N` (last N tickets)

```bash
# Example: filter by status
echo "$RESULTS" | grep "|${FILTER_STATUS}|"

# Example: filter by keyword
echo "$RESULTS" | grep -i "${KEYWORD}"
```

## 4. Display History Table

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► TICKET HISTORY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Ticket | Title | Status | Commits | Date |
|--------|-------|--------|---------|------|
| ${ID} | ${TITLE} | ${STATUS} | ${COMMITS} | ${DATE} |
| ... | ... | ... | ... | ... |
```

## 5. Show Summary Stats

```
## Summary

Total tickets: ${TOTAL}
Verified: ${VERIFIED_COUNT}
Completed: ${COMPLETED_COUNT}
In progress: ${IN_PROGRESS_COUNT}
Analyzed: ${ANALYZED_COUNT}
Estimated: ${ESTIMATED_COUNT}
```

## 6. Offer Drill-Down

```
AskUserQuestion(
  header: "Details",
  question: "Want to see details for a specific ticket?",
  options: [
    { label: "${ID_1}", description: "${STATUS_1} — ${TITLE_1}" },
    { label: "${ID_2}", description: "${STATUS_2} — ${TITLE_2}" },
    { label: "Done", description: "Exit history view" }
  ]
)
```

If "Done": STOP.

If a ticket is selected, show detailed view:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► TICKET DETAILS — ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## ${TICKET_ID}: ${TITLE}

Status: ${STATUS}
Branch: ${BRANCH}
Commits: ${COMMIT_COUNT}
Date: ${DATE}

## Artifacts

| File | Exists |
|------|--------|
| ESTIMATE.md | ${yes/no} |
| ANALYSIS.md | ${yes/no} |
| PLAN.md | ${yes/no} |
| SUMMARY.md | ${yes/no} |
| VERIFICATION.md | ${yes/no} |

## Commits

$(git log --oneline --all --grep="${TICKET_ID}" --format="%h %s (%ar)")

## Files Changed

$(git log --all --grep="${TICKET_ID}" --name-only --format="" | sort -u)
```

If VERIFICATION.md exists, also show:
```
## Verification Result

$(grep -A 5 "Status\|Score\|status\|score" .relay/tickets/${TICKET_ID}/VERIFICATION.md | head -10)
```

</process>

<success_criteria>
- [ ] All ticket directories scanned
- [ ] Status determined from artifacts for each ticket
- [ ] Commit counts gathered from git log
- [ ] History table displayed with all tickets
- [ ] Summary stats calculated
- [ ] Filters applied if provided
- [ ] Drill-down available for individual tickets
- [ ] No files modified (read-only operation)
</success_criteria>
