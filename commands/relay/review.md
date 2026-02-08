---
name: relay:review
description: Review a PR or address review comments on your own PR
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
---

<objective>
Two modes:
- **Mode A — Review a PR**: Analyze someone else's PR with a code review agent
- **Mode B — Address comments**: Fix review comments on your own PR

Selected upfront via user choice.
</objective>

<context>
@.relay/config.json
@.relay/STATE.md
</context>

<process>

## 0. Pre-flight

```bash
test -f .relay/config.json && echo "config exists" || echo "no config"
command -v gh >/dev/null 2>&1 && echo "gh available" || echo "gh missing"
```

If no config: suggest `/relay:setup` and STOP.
If `gh` CLI not available: report that `gh` is required and STOP.

Read model profile:
```bash
MODEL_PROFILE=$(cat .relay/config.json 2>/dev/null | grep -o '"model_profile"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "balanced")
```

**Model lookup table:**

| Agent | quality | balanced | budget |
|-------|---------|----------|--------|
| relay-reviewer | opus | sonnet | sonnet |

## 1. Choose Mode

```
AskUserQuestion(
  header: "Review",
  question: "What would you like to do?",
  options: [
    { label: "Review a PR", description: "Analyze someone else's PR and provide code review" },
    { label: "Address comments", description: "Fix review comments on your own PR" }
  ]
)
```

---

## Mode A — Review a PR

### A1. List Open PRs

```bash
gh pr list --state open --limit 20
```

If no open PRs: report and STOP.

```
AskUserQuestion(
  header: "PR",
  question: "Which PR would you like to review?",
  options: [
    { label: "#${PR_1_NUM}", description: "${PR_1_TITLE}" },
    { label: "#${PR_2_NUM}", description: "${PR_2_TITLE}" },
    { label: "#${PR_3_NUM}", description: "${PR_3_TITLE}" }
  ]
)
```

### A2. Save Current Branch and Checkout PR

```bash
ORIGINAL_BRANCH=$(git branch --show-current)
git fetch origin
gh pr checkout ${PR_NUMBER}
```

### A3. Get Diff

```bash
BASE_BRANCH=$(gh pr view ${PR_NUMBER} --json baseRefName --jq '.baseRefName')
git diff ${BASE_BRANCH}...HEAD --stat
git diff ${BASE_BRANCH}...HEAD
```

### A4. Spawn Reviewer Agent

```bash
mkdir -p .relay/reviews
```

```
Task(
  prompt="
<pr>
Number: ${PR_NUMBER}
Title: ${PR_TITLE}
Author: ${PR_AUTHOR}
Base: ${BASE_BRANCH}
Description: ${PR_DESCRIPTION}

<diff>
${DIFF_CONTENT}
</diff>

<files_changed>
${DIFF_STAT}
</files_changed>
</pr>

<codebase_context>
@.relay/codebase/ARCHITECTURE.md (if exists)
@.relay/codebase/CONVENTIONS.md (if exists)
@.relay/codebase/TESTING.md (if exists)
</codebase_context>

<output>
Write review to: .relay/reviews/PR-${PR_NUMBER}-review.md
</output>
",
  subagent_type="relay-reviewer",
  model="{reviewer_model}",
  description="Review PR #${PR_NUMBER}"
)
```

### A5. Display Review and Offer Actions

Read `.relay/reviews/PR-${PR_NUMBER}-review.md` and display summary.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► REVIEW COMPLETE — PR #${PR_NUMBER}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Recommendation: ${APPROVE | COMMENT | REQUEST_CHANGES}
Critical: ${COUNT}
Suggestions: ${COUNT}

Full review: .relay/reviews/PR-${PR_NUMBER}-review.md
```

```
AskUserQuestion(
  header: "Action",
  question: "What would you like to do with this review?",
  options: [
    { label: "Post as PR comment", description: "Submit review via gh pr review" },
    { label: "View full review", description: "Display the complete review document" },
    { label: "Done", description: "Return to original branch" }
  ]
)
```

**If "Post as PR comment":**
```bash
gh pr review ${PR_NUMBER} --body "$(cat .relay/reviews/PR-${PR_NUMBER}-review.md)" --comment
```

### A6. Restore Original Branch

```bash
git checkout "${ORIGINAL_BRANCH}"
```

Always restore the original branch, regardless of action taken.

---

## Mode B — Address PR Comments

### B1. List Your Open PRs

```bash
gh pr list --state open --author @me --limit 20
```

If no open PRs: report and STOP.

```
AskUserQuestion(
  header: "PR",
  question: "Which PR has comments to address?",
  options: [
    { label: "#${PR_1_NUM}", description: "${PR_1_TITLE}" },
    { label: "#${PR_2_NUM}", description: "${PR_2_TITLE}" },
    { label: "#${PR_3_NUM}", description: "${PR_3_TITLE}" }
  ]
)
```

### B2. Fetch Review Comments

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh api repos/${REPO}/pulls/${PR_NUMBER}/comments --jq '.[] | {path: .path, line: .line, body: .body, user: .user.login}'
gh api repos/${REPO}/pulls/${PR_NUMBER}/reviews --jq '.[] | select(.state != "APPROVED") | {body: .body, user: .user.login, state: .state}'
```

### B3. Display Comments Grouped by File

Group comments by file path and display:

```
## Review Comments for PR #${PR_NUMBER}

### ${FILE_1}
- Line ${LINE}: ${COMMENT_BODY} (by ${USER})

### ${FILE_2}
- Line ${LINE}: ${COMMENT_BODY} (by ${USER})
```

### B4. Choose What to Address

```
AskUserQuestion(
  header: "Comments",
  question: "Which comments would you like to address?",
  options: [
    { label: "All comments", description: "Address every review comment" },
    { label: "Select specific", description: "Choose which comments to fix" },
    { label: "View only", description: "Just review the comments, don't change code" }
  ]
)
```

If "View only": display and STOP.

If "Select specific": let user pick which comments to address.

### B5. Checkout PR Branch and Fix

```bash
gh pr checkout ${PR_NUMBER}
```

For each comment to address:
1. Read the file at the mentioned line
2. Understand the feedback
3. Make the code change using Edit
4. Verify the fix

### B6. Commit Fixes

For each file changed, commit with descriptive message:

```bash
# Detect ticket ID from branch name or PR title
TICKET_ID=$(git branch --show-current | grep -oE '[A-Z]+-[0-9]+' | head -1)
TICKET_ID=${TICKET_ID:-review}

git add ${CHANGED_FILE}
git commit -m "fix(${TICKET_ID}): address review — ${DESCRIPTION}"
```

### B7. Offer to Push

```
AskUserQuestion(
  header: "Push",
  question: "Push fixes to update the PR?",
  options: [
    { label: "Push", description: "Push commits to update PR #${PR_NUMBER}" },
    { label: "Skip", description: "Keep changes local for now" }
  ]
)
```

If "Push":
```bash
git push
```

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► COMMENTS ADDRESSED — PR #${PR_NUMBER}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Comments addressed: ${COUNT}
Commits: ${COMMIT_COUNT}
Pushed: ${YES/NO}
```

</process>

<success_criteria>
**Mode A:**
- [ ] PR selected from open PRs
- [ ] Original branch saved
- [ ] PR branch checked out
- [ ] Diff captured
- [ ] Reviewer agent produced structured review
- [ ] Review displayed with recommendation
- [ ] Action taken (post, view, or skip)
- [ ] Original branch restored

**Mode B:**
- [ ] User's PR selected
- [ ] Review comments fetched and displayed
- [ ] Selected comments addressed with code changes
- [ ] Each fix committed with descriptive message
- [ ] Pushed if requested
</success_criteria>
