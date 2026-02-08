# Estimate Template

Template for `.relay/tickets/{TICKET-ID}/ESTIMATE.md` — codebase-aware effort estimation.

---

## File Template

```markdown
# Estimate: {TICKET-ID}

## Ticket Summary

**Title:** {title}
**Type:** {bug | feature | task | improvement}
**Priority:** {priority}

## Complexity Score

**Score:** {1 | 2 | 3 | 5 | 8 | 13} (Fibonacci)

### Rationale

{why this score — based on codebase analysis}

### Factors

| Factor | Assessment | Notes |
|--------|-----------|-------|
| Files to modify | {count} | {details} |
| New files needed | {count} | {details} |
| Cross-cutting concerns | {low/med/high} | {what systems are touched} |
| Test coverage needed | {low/med/high} | {new tests, modified tests} |
| Risk level | {low/med/high} | {unknowns, dependencies} |

## Time Estimate

**Range:** {min} — {max} (in working hours or days)

### Breakdown

| Phase | Estimate | Notes |
|-------|---------|-------|
| Analysis & planning | {time} | {details} |
| Implementation | {time} | {details} |
| Testing | {time} | {details} |
| Review & polish | {time} | {details} |

## Impact Analysis

### Files Affected

| File | Change Type | Scope |
|------|------------|-------|
| {path} | {modify/create/delete} | {minor/moderate/major} |

### Dependencies

{packages, modules, or services that this touches}

### Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| {risk} | {low/med/high} | {low/med/high} | {mitigation} |

### Downstream Effects

{what other parts of the system could be affected}

## Recommendation

{should this be worked on now? any prerequisites? suggested approach?}

---

_Estimated: {timestamp}_
_Estimator: Claude (relay-estimator)_
```

<purpose>

ESTIMATE.md provides data-driven effort estimation before committing to ticket work. It queries the actual codebase to ground estimates in reality rather than guesswork.

**Problem it solves:** Teams need to estimate tickets for sprint planning, but estimates without codebase context are unreliable. A "simple" ticket might touch 20 files across 5 modules.

**Solution:** A structured estimate that:
- Scores complexity using Fibonacci scale with codebase evidence
- Provides time ranges based on actual scope
- Analyzes impact across the codebase
- Surfaces risks before work begins

</purpose>

<lifecycle>

**Creation:** During `/relay:estimate`
- Spawned by `relay-estimator` agent
- Input: ticket data from MCP + codebase exploration
- Output: ESTIMATE.md in `.relay/tickets/{ID}/`

**Reading:**
- By user to inform sprint planning
- By `/relay:work` if estimate exists (informational context)

**Does NOT create branches or set active ticket state.**

</lifecycle>
