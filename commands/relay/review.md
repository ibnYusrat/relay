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
```

If no config: suggest `/relay:setup` and STOP.

Read code hosting config from `.relay/config.json`:
- `code_hosting.type`: `"github"` | `"gitlab"` | `"azure_devops"` | `"bitbucket"` | `null`
- `code_hosting.cli`: `"gh"` | `"glab"` | `"az"` | `null`

If `code_hosting.type` is null: suggest running `/relay:setup` to configure code hosting and STOP.

Check CLI availability for the configured provider:

```bash
# For github:
command -v gh >/dev/null 2>&1 && echo "gh available" || echo "gh missing"
# For gitlab:
command -v glab >/dev/null 2>&1 && echo "glab available" || echo "glab missing"
# For azure_devops:
command -v az >/dev/null 2>&1 && az repos pr list --top 1 >/dev/null 2>&1 && echo "az repos available" || echo "az repos missing"
```

If `code_hosting.cli` is null or the CLI binary is not found:
- For **Bitbucket** or **Other/None**: warn that automated PR listing and posting are not available; the review will use plain git for checkout/diff and the user will provide branch names manually.
- For **GitHub/GitLab/Azure DevOps**: report that the required CLI (`gh`/`glab`/`az`) is missing and STOP.

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

**If code_hosting.type == "github":**
```bash
gh pr list --state open --limit 20
```

**Else if code_hosting.type == "gitlab":**
```bash
glab mr list --state opened --per-page 20
```

**Else if code_hosting.type == "azure_devops":**
```bash
az repos pr list --status active --top 20 --output table
```

**Else:**
Automated PR listing is not available. Ask the user for the branch name to review:

```
AskUserQuestion(
  header: "Branch",
  question: "Enter the branch name to review (e.g., feature/my-branch):",
  options: [
    { label: "Enter branch", description: "Provide the remote branch name" }
  ]
)
```

Skip to A2 with the user-provided branch name (set `PR_BRANCH` directly).

If using GitHub/GitLab and no open PRs: report and STOP.

If using GitHub/GitLab, present the list for selection:

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

### A2. Save Current Branch, Stash Changes, and Checkout PR

```bash
ORIGINAL_BRANCH=$(git branch --show-current)
```

Check for uncommitted changes and stash if present:

```bash
if [ -n "$(git status --porcelain)" ]; then
  git stash push -m "relay-review-stash"
  echo "STASHED=true"
else
  echo "STASHED=false"
fi
```

Fetch latest and checkout the PR branch:

```bash
git fetch origin
```

**If code_hosting.type == "github":**
```bash
gh pr checkout ${PR_NUMBER}
```

**Else if code_hosting.type == "gitlab":**
```bash
glab mr checkout ${PR_NUMBER}
```

**Else if code_hosting.type == "azure_devops":**
```bash
az repos pr checkout --id ${PR_NUMBER}
```

**Else (git fallback):**
```bash
git checkout ${PR_BRANCH}
```

### A3. Get Diff

**If code_hosting.type == "github":**
```bash
BASE_BRANCH=$(gh pr view ${PR_NUMBER} --json baseRefName --jq '.baseRefName')
```

**Else if code_hosting.type == "gitlab":**
```bash
BASE_BRANCH=$(glab mr view ${PR_NUMBER} --output json | jq -r '.target_branch')
```

**Else if code_hosting.type == "azure_devops":**
```bash
BASE_BRANCH=$(az repos pr show --id ${PR_NUMBER} --query targetRefName --output tsv | sed 's|refs/heads/||')
```

**Else (git fallback):**
```bash
BASE_BRANCH=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | sed 's/.*: //')
BASE_BRANCH=${BASE_BRANCH:-main}
```
Ask the user to confirm the base branch (default pre-selected):
```
AskUserQuestion(
  header: "Base",
  question: "Which branch is this PR targeting? (detected: ${BASE_BRANCH})",
  options: [
    { label: "${BASE_BRANCH}", description: "Use detected default branch" },
    { label: "main", description: "Use main" },
    { label: "master", description: "Use master" },
    { label: "develop", description: "Use develop" }
  ]
)
```

```bash
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
    { label: "Post as PR comment", description: "Submit review via hosting CLI" },
    { label: "View full review", description: "Display the complete review document" },
    { label: "Done", description: "Return to original branch" }
  ]
)
```

**If "Post as PR comment":**

**If code_hosting.type == "github":**
```bash
gh pr review ${PR_NUMBER} --body "$(cat .relay/reviews/PR-${PR_NUMBER}-review.md)" --comment
```

**Else if code_hosting.type == "gitlab":**
```bash
glab mr note ${PR_NUMBER} --message "$(cat .relay/reviews/PR-${PR_NUMBER}-review.md)"
```

**Else if code_hosting.type == "azure_devops":**
```bash
az repos pr create-comment --id ${PR_NUMBER} --content "$(cat .relay/reviews/PR-${PR_NUMBER}-review.md)"
```

**Else:**
Report that posting reviews requires GitHub, GitLab, or Azure DevOps CLI. Offer to display the review locally instead.

### A6. Restore Original Branch and Unstash

```bash
git checkout "${ORIGINAL_BRANCH}"
```

If changes were stashed in A2, restore them:

```bash
if [ "${STASHED}" = "true" ]; then
  git stash pop
fi
```

Always restore the original branch and unstash, regardless of action taken.

---

## Mode B — Address PR Comments

### B1. List Your Open PRs

**If code_hosting.type == "github":**
```bash
gh pr list --state open --author @me --limit 20
```

**Else if code_hosting.type == "gitlab":**
```bash
glab mr list --state opened --author @me --per-page 20
```

**Else if code_hosting.type == "azure_devops":**
```bash
az repos pr list --status active --creator $(az account show --query user.name --output tsv) --top 20 --output table
```

**Else:**
Automated PR listing is not available. Ask the user for the branch name:

```
AskUserQuestion(
  header: "Branch",
  question: "Enter the branch name of your PR:",
  options: [
    { label: "Enter branch", description: "Provide the remote branch name" }
  ]
)
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

**If code_hosting.type == "github":**
```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh api repos/${REPO}/pulls/${PR_NUMBER}/comments --jq '.[] | {path: .path, line: .line, body: .body, user: .user.login}'
gh api repos/${REPO}/pulls/${PR_NUMBER}/reviews --jq '.[] | select(.state != "APPROVED") | {body: .body, user: .user.login, state: .state}'
```

**Else if code_hosting.type == "gitlab":**
```bash
REPO_ENCODED=$(glab repo view --output json | jq -r '.full_path' | sed 's/\//%2F/g')
glab api projects/${REPO_ENCODED}/merge_requests/${PR_NUMBER}/notes --jq '.[] | select(.system == false) | {body: .body, author: .author.username}'
```

**Else if code_hosting.type == "azure_devops":**
```bash
az repos pr list-comments --id ${PR_NUMBER} --output json | jq '.[] | select(.commentType == "text") | {content: .content, author: .author.displayName}'
```

**Else:**
Report that fetching review comments requires GitHub, GitLab, or Azure DevOps CLI. STOP.

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

### B5. Stash Changes, Checkout PR Branch, and Fix

Save current branch and stash uncommitted changes:

```bash
ORIGINAL_BRANCH=$(git branch --show-current)
if [ -n "$(git status --porcelain)" ]; then
  git stash push -m "relay-review-stash"
  echo "STASHED=true"
else
  echo "STASHED=false"
fi
```

**If code_hosting.type == "github":**
```bash
gh pr checkout ${PR_NUMBER}
```

**Else if code_hosting.type == "gitlab":**
```bash
glab mr checkout ${PR_NUMBER}
```

**Else if code_hosting.type == "azure_devops":**
```bash
az repos pr checkout --id ${PR_NUMBER}
```

**Else (git fallback):**
```bash
git fetch origin
git checkout ${PR_BRANCH}
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

### B8. Restore Original Branch and Unstash

```bash
git checkout "${ORIGINAL_BRANCH}"
```

If changes were stashed in B5, restore them:

```bash
if [ "${STASHED}" = "true" ]; then
  git stash pop
fi
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
- [ ] Code hosting provider read from config
- [ ] PR/MR listed via provider-appropriate CLI (or branch name requested for unsupported providers)
- [ ] PR selected from open PRs/MRs
- [ ] Original branch saved
- [ ] Uncommitted changes stashed if present
- [ ] PR/MR branch checked out (via CLI or plain git)
- [ ] Diff captured
- [ ] Reviewer agent produced structured review
- [ ] Review displayed with recommendation
- [ ] Action taken (post, view, or skip) via provider-appropriate CLI
- [ ] Original branch restored
- [ ] Stashed changes restored

**Mode B:**
- [ ] Code hosting provider read from config
- [ ] User's PR/MR listed and selected (or branch name requested for unsupported providers)
- [ ] Review comments fetched via provider-appropriate API
- [ ] Uncommitted changes stashed if present
- [ ] Selected comments addressed with code changes
- [ ] Each fix committed with descriptive message
- [ ] Pushed if requested
- [ ] Original branch restored and stashed changes restored
</success_criteria>
