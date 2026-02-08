---
name: relay:estimate
description: Estimate effort for a ticket against the actual codebase
argument-hint: <ticket-id>
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
  - mcp__jira__get_issue
  - mcp__github__get_issue
  - mcp__azure_devops__get_work_item
---

<objective>
Estimate effort for a ticket before committing to work. Analyzes the ticket against the actual codebase to produce a grounded estimate with complexity score, time range, and impact analysis.

Does NOT create a branch or set active ticket — estimation is non-committal.
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
| relay-estimator | opus | sonnet | sonnet |

If no $ARGUMENTS (ticket ID):
```
AskUserQuestion(
  header: "Ticket",
  question: "Which ticket would you like to estimate?",
  options: [
    { label: "Browse tickets", description: "Run /relay:tickets to see available tickets" },
    { label: "Enter ID", description: "Type a ticket ID (e.g., PROJ-123)" }
  ]
)
```

## 1. Fetch Ticket

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► ESTIMATING ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**1a. Create ticket directory (if it doesn't exist):**
```bash
mkdir -p .relay/tickets/${TICKET_ID}
```

**1b. Fetch ticket details via MCP:**

Read `integration.type` from config and call appropriate MCP tool:

- **Jira:** `mcp__jira__get_issue` with issue key
- **GitHub:** `mcp__github__get_issue` with issue number and repo
- **Azure DevOps:** `mcp__azure_devops__get_work_item` with work item ID

If no integration configured, ask user to describe the ticket manually.

Store ticket data: title, description, type, priority, acceptance criteria, labels.

## 2. Choose Estimate Type

```
AskUserQuestion(
  header: "Estimate",
  question: "What type of estimate do you need?",
  options: [
    { label: "All (Recommended)", description: "Complexity score + time estimate + impact analysis" },
    { label: "Complexity", description: "Fibonacci score (1/2/3/5/8/13) with rationale" },
    { label: "Time", description: "Time range with phase breakdown" },
    { label: "Impact", description: "Files affected, dependencies, and risks" }
  ]
)
```

## 3. Spawn Estimator

Spawn `relay-estimator` agent with ticket details and codebase context:

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

<estimate_type>${ESTIMATE_TYPE}</estimate_type>

<codebase_context>
@.relay/codebase/ARCHITECTURE.md (if exists)
@.relay/codebase/STRUCTURE.md (if exists)
@.relay/codebase/STACK.md (if exists)
@.relay/codebase/CONVENTIONS.md (if exists)
@.relay/codebase/TESTING.md (if exists)
</codebase_context>

<output>
Write estimate to: .relay/tickets/${TICKET_ID}/ESTIMATE.md
Follow template: @~/.claude/relay/templates/estimate.md
</output>
",
  subagent_type="relay-estimator",
  model="{estimator_model}",
  description="Estimate ${TICKET_ID}"
)
```

Verify ESTIMATE.md was created.

## 4. Display Results

Read `.relay/tickets/${TICKET_ID}/ESTIMATE.md` and display key findings:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► ESTIMATE COMPLETE — ${TICKET_ID}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ticket: ${TICKET_ID} — ${TICKET_TITLE}

Complexity: ${SCORE}/13 — ${RATIONALE}
Time: ${MIN} — ${MAX}
Impact: ${FILES_MODIFY} files to modify, ${FILES_CREATE} to create

Key Risks:
- ${RISK_1}
- ${RISK_2}

Full estimate: .relay/tickets/${TICKET_ID}/ESTIMATE.md

## Next Steps

- `/relay:work ${TICKET_ID}` — start working on this ticket
- `/relay:estimate <other-id>` — estimate another ticket
- `/relay:tickets` — browse available tickets
```

</process>

<success_criteria>
- [ ] Ticket fetched from external system (or described manually)
- [ ] Estimate type selected by user
- [ ] Estimator agent explored actual codebase
- [ ] ESTIMATE.md created in `.relay/tickets/{ID}/`
- [ ] Results displayed with key metrics
- [ ] No branch created, no active ticket set
- [ ] Next steps suggested
</success_criteria>
