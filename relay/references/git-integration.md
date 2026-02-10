<overview>
Git integration for Relay framework.
</overview>

<core_principle>

**Commit outcomes, not process.**

The git log should read like a changelog of what shipped, not a diary of planning activity.
</core_principle>

<default_branch_protection>

**NEVER commit to the default branch.**

Before any git commit, verify the current branch is NOT the default branch (main, master, or whatever `git remote show origin | grep 'HEAD branch'` returns). If you are on the default branch, STOP and create a working branch first. There are no exceptions to this rule — no hotfix, no "just one small change", no override. All work is committed on a non-default branch and merged via PR.

```bash
DEFAULT_BRANCH=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | sed 's/.*: //')
DEFAULT_BRANCH=${DEFAULT_BRANCH:-main}
CURRENT_BRANCH=$(git branch --show-current)
if [ "${CURRENT_BRANCH}" = "${DEFAULT_BRANCH}" ]; then
  echo "ERROR: On default branch (${DEFAULT_BRANCH}). Create a working branch before committing."
  exit 1
fi
```

</default_branch_protection>

<commit_points>

| Event                   | Commit? | Why                                              |
| ----------------------- | ------- | ------------------------------------------------ |
| ANALYSIS.md created     | NO      | Intermediate - analysis of ticket                |
| PLAN.md created         | NO      | Intermediate - commit with plan completion       |
| **Task completed**      | YES     | Atomic unit of work (1 commit per task)         |
| **Ticket completed**    | YES     | Metadata commit (SUMMARY + STATE)               |
| Handoff created         | YES     | WIP state preserved                              |

</commit_points>

<git_check>

```bash
[ -d .git ] && echo "GIT_EXISTS" || echo "NO_GIT"
```

If NO_GIT: Run `git init` silently.
</git_check>

<branching>

## Ticket Branches

When branching is enabled, each ticket gets its own branch:

```bash
# Branch template from config.git.ticket_branch_template
# Default: "{ticket_id}/{slug}"
# Example: PROJ-123/add-user-validation

SLUG=$(echo "${TITLE}" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//' | cut -c1-40)
BRANCH="${TICKET_ID}/${SLUG}"
git checkout -b "${BRANCH}"
```

</branching>

<commit_formats>

<format name="task-completion">
## Task Completion (During Plan Execution)

Each task gets its own commit immediately after completion. Include ticket ID in commit message.

```
{type}({ticket_id}): {task-name}

- [Key change 1]
- [Key change 2]
- [Key change 3]
```

**Commit types:**
- `feat` - New feature/functionality
- `fix` - Bug fix
- `test` - Test-only (TDD RED phase)
- `refactor` - Code cleanup (TDD REFACTOR phase)
- `perf` - Performance improvement
- `chore` - Dependencies, config, tooling

**Examples:**

```bash
# Standard task
git add src/api/auth.ts src/types/user.ts
git commit -m "feat(PROJ-123): create user registration endpoint

- POST /auth/register validates email and password
- Checks for duplicate users
- Returns JWT token on success
"

# Bug fix task
git add src/utils/validation.ts
git commit -m "fix(BUG-456): prevent null pointer in email validation

- Added null check before regex match
- Returns early with error for empty input
"
```

</format>

<format name="ticket-completion">
## Ticket Completion (After All Tasks Done)

After all tasks committed, one final metadata commit captures ticket completion.

```
docs({ticket_id}): complete {ticket-title}

Tasks completed: [N]/[N]
- [Task 1 name]
- [Task 2 name]
- [Task 3 name]

SUMMARY: .relay/tickets/{TICKET_ID}/SUMMARY.md
```

What to commit:

```bash
git add .relay/tickets/{TICKET_ID}/PLAN.md
git add .relay/tickets/{TICKET_ID}/SUMMARY.md
git add .relay/tickets/{TICKET_ID}/VERIFICATION.md
git add .relay/STATE.md
git commit
```

**Note:** Code files NOT included - already committed per-task.

</format>

<format name="handoff">
## Handoff (WIP)

```
wip({ticket_id}): paused at task [X]/[Y]

Current: [task name]
[If blocked:] Blocked: [reason]
```

What to commit:

```bash
git add .relay/
git commit
```

</format>
</commit_formats>

<example_log>

```
# Ticket PROJ-125 - Checkout Flow
1a2b3c docs(PROJ-125): complete checkout flow
4d5e6f feat(PROJ-125): add webhook signature verification
7g8h9i feat(PROJ-125): implement payment session creation
0j1k2l feat(PROJ-125): create checkout page component

# Ticket PROJ-124 - Product Catalog
3m4n5o docs(PROJ-124): complete product catalog
6p7q8r feat(PROJ-124): add pagination controls
9s0t1u feat(PROJ-124): implement search and filters

# Ticket BUG-200 - Login Validation
5y6z7a fix(BUG-200): prevent crash on empty email input
```

Each ticket produces 2-5 commits (tasks + metadata). Clear, granular, bisectable.

</example_log>

<anti_patterns>

**Still don't commit (intermediate artifacts):**
- ANALYSIS.md creation (intermediate)
- PLAN.md creation (commit with ticket completion)
- Minor planning tweaks

**Do commit (outcomes):**
- Each task completion (feat/fix/test/refactor)
- Ticket completion metadata (docs)
- Handoff state (wip)

**Key principle:** Commit working code and shipped outcomes, not planning process.

**Absolute rule:** Never commit directly to the default branch. Always work on a non-default branch.

</anti_patterns>

<commit_strategy_rationale>

## Why Per-Task Commits?

**Context engineering for AI:**
- Git history becomes primary context source for future Claude sessions
- `git log --grep="{ticket_id}"` shows all work for a ticket
- `git diff <hash>^..<hash>` shows exact changes per task
- Less reliance on parsing SUMMARY.md = more context for actual work

**Failure recovery:**
- Task 1 committed, Task 2 failed
- Claude in next session: sees task 1 complete, can retry task 2
- Can `git reset --hard` to last successful task

**Debugging:**
- `git bisect` finds exact failing task
- `git blame` traces line to specific task context
- Each commit is independently revertable

**Observability:**
- Ticket ID in every commit links back to external system
- Atomic commits are git best practice

</commit_strategy_rationale>
