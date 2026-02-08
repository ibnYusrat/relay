---
name: relay:work
description: The main orchestrator — fetch a ticket, analyze, plan, execute, and sync
argument-hint: <ticket-id>
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
  - SlashCommand
  - mcp__jira__get_issue
  - mcp__jira__add_comment
  - mcp__jira__update_issue
  - mcp__github__get_issue
  - mcp__github__create_comment
  - mcp__github__update_issue
  - mcp__azure_devops__get_work_item
  - mcp__azure_devops__update_work_item
---

<objective>
The main Relay orchestrator. Takes a ticket ID, fetches details from the external system, analyzes the codebase impact, creates a plan, gets user approval, executes the work, verifies results, and syncs back to the ticket system.

6 stages: FETCH → ANALYZE → PLAN → CONFIRM → EXECUTE → VERIFY & SYNC

Supports resumption — if `.relay/tickets/{ID}/` already exists, detects the last completed stage and resumes from there.
</objective>

<context>
@.relay/config.json
@.relay/STATE.md
Ticket ID: $ARGUMENTS
</context>

<process>

## 0. Pre-flight

```bash
test -f .relay/config.json && echo "config exists" || echo "no config"
```

If no config: suggest `/relay:setup` and STOP.

Read config for integration settings and model profile:

```bash
MODEL_PROFILE=$(cat .relay/config.json 2>/dev/null | grep -o '"model_profile"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "balanced")
```

**Model lookup table:**

| Agent | quality | balanced | budget |
|-------|---------|----------|--------|
| relay-ticket-researcher | opus | sonnet | sonnet |
| relay-planner | opus | opus | sonnet |
| relay-plan-checker | opus | sonnet | sonnet |
| relay-executor | opus | sonnet | sonnet |
| relay-verifier | opus | sonnet | haiku |

If no $ARGUMENTS (ticket ID):
```
AskUserQuestion(
  header: "Ticket",
  question: "Which ticket would you like to work on?",
  options: [
    { label: "Browse tickets", description: "Run /relay:tickets to see available tickets" },
    { label: "Enter ID", description: "Type a ticket ID (e.g., PROJ-123)" }
  ]
)
```

## 1. Resume Detection

Check if ticket directory already exists:

```bash
test -d .relay/tickets/${TICKET_ID} && echo "resuming" || echo "new"
```

**If resuming**, detect last completed stage:
- Has VERIFICATION.md → already complete, show results
- Has SUMMARY.md → resume at VERIFY stage
- Has PLAN.md → resume at CONFIRM stage
- Has ANALYSIS.md → resume at PLAN stage
- Directory exists but empty → resume at FETCH stage

Report:
```
Resuming ticket ${TICKET_ID} from ${stage} stage.
```

Skip to the appropriate stage below.

## 2. Stage 1: FETCH

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► FETCHING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**2a. Create ticket directory:**
```bash
mkdir -p .relay/tickets/${TICKET_ID}
```

**2b. Fetch ticket details via MCP:**

Read `integration.type` from config and call appropriate MCP tool:

- **Jira:** `mcp__jira__get_issue` with issue key
- **GitHub:** `mcp__github__get_issue` with issue number and repo
- **Azure DevOps:** `mcp__azure_devops__get_work_item` with work item ID

If no integration configured, ask user to describe the ticket manually.

Store ticket data: title, description, type, priority, acceptance criteria, labels, assignee.

**2c. Create git branch:**

Read branch template from config (`git.ticket_branch_template`).
Generate branch name:

```bash
SLUG=$(echo "${TICKET_TITLE}" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//' | cut -c1-40)
BRANCH="${TICKET_ID}/${SLUG}"
git checkout -b "${BRANCH}"
```

**2d. Update STATE.md:**

Set active ticket, branch, stage = FETCHING.

## 3. Stage 2: ANALYZE

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► ANALYZING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Spawn `relay-ticket-researcher` agent to analyze the ticket in context of the codebase:

```
Task(
  prompt="
<ticket>
ID: ${TICKET_ID}
Title: ${TICKET_TITLE}
Type: ${TICKET_TYPE}
Priority: ${TICKET_PRIORITY}
Description: ${TICKET_DESCRIPTION}
Acceptance Criteria: ${ACCEPTANCE_CRITERIA}
Labels: ${LABELS}
</ticket>

<codebase_context>
@.relay/codebase/ARCHITECTURE.md (if exists)
@.relay/codebase/STRUCTURE.md (if exists)
@.relay/codebase/STACK.md (if exists)
</codebase_context>

<output>
Write analysis to: .relay/tickets/${TICKET_ID}/ANALYSIS.md
Follow template: @~/.claude/relay/templates/analysis.md
</output>
",
  subagent_type="relay-ticket-researcher",
  model="{researcher_model}",
  description="Analyze ${TICKET_ID}"
)
```

Verify ANALYSIS.md was created.

## 4. Stage 3: PLAN

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► PLANNING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Spawn `relay-planner` with ANALYSIS.md as input:

```
Task(
  prompt="
<planning_context>
Mode: ticket
Ticket: ${TICKET_ID}
Directory: .relay/tickets/${TICKET_ID}

Analysis: @.relay/tickets/${TICKET_ID}/ANALYSIS.md
Project State: @.relay/STATE.md
</planning_context>

<output>
Write plan to: .relay/tickets/${TICKET_ID}/PLAN.md
</output>
",
  subagent_type="relay-planner",
  model="{planner_model}",
  description="Plan ${TICKET_ID}"
)
```

**Optional: spawn plan checker** (if `workflow.plan_check` is true in config):

```
Task(
  prompt="
Check plan quality for ticket ${TICKET_ID}.

Plan: @.relay/tickets/${TICKET_ID}/PLAN.md
Analysis: @.relay/tickets/${TICKET_ID}/ANALYSIS.md

Verify the plan addresses all acceptance criteria from the analysis.
",
  subagent_type="relay-plan-checker",
  model="{checker_model}",
  description="Check plan ${TICKET_ID}"
)
```

## 5. Stage 4: CONFIRM

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► CONFIRMING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Present the analysis and plan to the user:

**Show summary of ANALYSIS.md:**
- Ticket title and type
- Acceptance criteria
- Files to modify
- Technical approach
- Risk assessment

**Show summary of PLAN.md:**
- Number of tasks
- Estimated scope
- Key decisions

```
AskUserQuestion(
  header: "Confirm",
  question: "Review the analysis and plan above. How would you like to proceed?",
  options: [
    { label: "Execute", description: "Start implementing the plan" },
    { label: "Modify plan", description: "Adjust the approach before executing" },
    { label: "Re-analyze", description: "Gather more context and re-analyze" },
    { label: "Cancel", description: "Stop working on this ticket" }
  ]
)
```

- **Execute** → proceed to Stage 5
- **Modify plan** → ask what to change, re-run planner with constraints
- **Re-analyze** → go back to Stage 2
- **Cancel** → clean up branch, update STATE.md, STOP

## 6. Stage 5: EXECUTE

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► EXECUTING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Spawn `relay-executor`:

```
Task(
  prompt="
Execute plan for ticket ${TICKET_ID}.

Plan: @.relay/tickets/${TICKET_ID}/PLAN.md
Analysis: @.relay/tickets/${TICKET_ID}/ANALYSIS.md
Project state: @.relay/STATE.md

<constraints>
- Execute all tasks in the plan
- Commit each task atomically
- Include ${TICKET_ID} in commit messages
- Create summary at: .relay/tickets/${TICKET_ID}/SUMMARY.md
</constraints>
",
  subagent_type="relay-executor",
  model="{executor_model}",
  description="Execute ${TICKET_ID}"
)
```

Verify SUMMARY.md was created.

## 7. Stage 6: VERIFY & SYNC

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► VERIFYING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**7a. Spawn verifier** (if `workflow.verifier` is true):

```
Task(
  prompt="
Verify work for ticket ${TICKET_ID}.

Analysis: @.relay/tickets/${TICKET_ID}/ANALYSIS.md
Summary: @.relay/tickets/${TICKET_ID}/SUMMARY.md

Verify against acceptance criteria from ANALYSIS.md.
Write report to: .relay/tickets/${TICKET_ID}/VERIFICATION.md
",
  subagent_type="relay-verifier",
  model="{verifier_model}",
  description="Verify ${TICKET_ID}"
)
```

**7b. Present results:**

Show verification summary — pass/fail for each acceptance criterion.

**7c. Ask about syncing:**

```
AskUserQuestion(
  header: "Sync",
  question: "Post results back to ${integration_type}?",
  options: [
    { label: "Post comment & update status", description: "Add summary comment and transition ticket" },
    { label: "Post comment only", description: "Add summary comment, don't change status" },
    { label: "Skip sync", description: "Don't update the external ticket" }
  ]
)
```

**7d. Sync via MCP** (if selected):

Post a comment with:
- Work summary from SUMMARY.md
- Verification results from VERIFICATION.md
- Branch name and commit references

Optionally update ticket status (e.g., move to "In Review" or "Done").

**7e. Update STATE.md:**

- Clear active ticket
- Add to recent tickets table
- Update velocity metrics
- Update session continuity

## 8. Completion

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► COMPLETE ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ticket: ${TICKET_ID} — ${TICKET_TITLE}
Branch: ${BRANCH}
Commits: ${COMMIT_COUNT}
Verification: ${PASS/FAIL}
Synced: ${YES/NO}

## Next Steps

- `/relay:tickets` — pick another ticket
- `/relay:work <id>` — start next ticket
- `/relay:status` — see overall progress
```

</process>

<success_criteria>
- [ ] Ticket fetched from external system (or described manually)
- [ ] Git branch created with ticket ID
- [ ] ANALYSIS.md created by ticket researcher
- [ ] PLAN.md created by planner
- [ ] User confirmed plan before execution
- [ ] Code implemented by executor with atomic commits
- [ ] VERIFICATION.md confirms acceptance criteria met
- [ ] Results synced back to ticket system (if configured)
- [ ] STATE.md updated throughout
</success_criteria>
