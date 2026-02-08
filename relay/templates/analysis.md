# Ticket Analysis Template

Template for `.relay/tickets/{TICKET-ID}/ANALYSIS.md` — codebase-aware ticket analysis.

---

## File Template

```markdown
# Analysis: {TICKET-ID}

## Ticket Summary

**Title:** {title from external system}
**Type:** {bug | feature | task | improvement}
**Priority:** {priority from external system}
**Reporter:** {reporter}
**Assignee:** {assignee}

**Description:**
{description from external system}

## Acceptance Criteria

{extracted or inferred from ticket}

- [ ] {criterion 1}
- [ ] {criterion 2}
- [ ] {criterion 3}

## Codebase Impact

### Files to Modify

| File | Change Type | Reason |
|------|------------|--------|
| {path} | {modify/create/delete} | {why} |

### Dependencies Affected

{list of modules, packages, or systems that this change touches}

### Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| {risk} | {low/med/high} | {low/med/high} | {mitigation} |

## Technical Approach

{high-level approach to implementing this ticket}

1. {step 1}
2. {step 2}
3. {step 3}

## Questions / Clarifications

{questions that need answers before implementation}

- [ ] {question 1}
- [ ] {question 2}

## External Context

**Source:** {Jira | GitHub Issues | Azure DevOps}
**URL:** {link to ticket}
**Labels:** {labels/tags}
**Sprint:** {sprint if applicable}
```

<purpose>

ANALYSIS.md bridges external ticket systems and the codebase. It translates a ticket description into an actionable, codebase-aware analysis.

**Problem it solves:** Tickets describe what the user wants. But planning requires knowing which files to change, what patterns to follow, and what risks exist. Without codebase analysis, plans are disconnected from reality.

**Solution:** A structured document that:
- Captures the ticket's requirements and acceptance criteria
- Maps them to specific codebase locations
- Identifies risks and dependencies
- Surfaces questions before planning begins

</purpose>

<lifecycle>

**Creation:** During the ANALYZE stage of `/relay:work`
- Spawned by `relay-ticket-researcher` agent
- Input: ticket data from MCP + codebase knowledge
- Output: ANALYSIS.md in `.relay/tickets/{ID}/`

**Reading:**
- By `relay-planner` as primary input for creating PLAN.md
- By `relay-plan-checker` to validate plan against requirements
- By `relay-verifier` to check acceptance criteria
- By user during the CONFIRM stage

**Never modified after creation** — if analysis needs updating, re-run the analyze stage.

</lifecycle>
