# Planner Subagent Prompt Template

Template for spawning relay-planner agent. The agent contains all planning expertise - this template provides planning context only.

---

## Template

```markdown
<planning_context>

**Phase:** {phase_number}
**Mode:** {standard | gap_closure}

**Project State:**
@.relay/STATE.md

**Roadmap:**
@.relay/ROADMAP.md

**Requirements (if exists):**
@.relay/REQUIREMENTS.md

**Phase Context (if exists):**
@.relay/tickets/{phase_dir}/{phase}-CONTEXT.md

**Research (if exists):**
@.relay/tickets/{phase_dir}/{phase}-RESEARCH.md

**Gap Closure (if --gaps mode):**
@.relay/tickets/{phase_dir}/{phase}-VERIFICATION.md
@.relay/tickets/{phase_dir}/{phase}-UAT.md

</planning_context>

<downstream_consumer>
Output consumed by /relay:execute-phase
Plans must be executable prompts with:
- Frontmatter (wave, depends_on, files_modified, autonomous)
- Tasks in XML format
- Verification criteria
- must_haves for goal-backward verification
</downstream_consumer>

<quality_gate>
Before returning PLANNING COMPLETE:
- [ ] PLAN.md files created in phase directory
- [ ] Each plan has valid frontmatter
- [ ] Tasks are specific and actionable
- [ ] Dependencies correctly identified
- [ ] Waves assigned for parallel execution
- [ ] must_haves derived from phase goal
</quality_gate>
```

---

## Placeholders

| Placeholder | Source | Example |
|-------------|--------|---------|
| `{phase_number}` | From roadmap/arguments | `5` or `2.1` |
| `{phase_dir}` | Phase directory name | `05-user-profiles` |
| `{phase}` | Phase prefix | `05` |
| `{standard \| gap_closure}` | Mode flag | `standard` |

---

## Usage

**From /relay:plan-phase (standard mode):**
```python
Task(
  prompt=filled_template,
  subagent_type="relay-planner",
  description="Plan Phase {phase}"
)
```

**From /relay:plan-phase --gaps (gap closure mode):**
```python
Task(
  prompt=filled_template,  # with mode: gap_closure
  subagent_type="relay-planner",
  description="Plan gaps for Phase {phase}"
)
```

---

## Continuation

For checkpoints, spawn fresh agent with:

```markdown
<objective>
Continue planning for Phase {phase_number}: {phase_name}
</objective>

<prior_state>
Phase directory: @.relay/tickets/{phase_dir}/
Existing plans: @.relay/tickets/{phase_dir}/*-PLAN.md
</prior_state>

<checkpoint_response>
**Type:** {checkpoint_type}
**Response:** {user_response}
</checkpoint_response>

<mode>
Continue: {standard | gap_closure}
</mode>
```

---

**Note:** Planning methodology, task breakdown, dependency analysis, wave assignment, TDD detection, and goal-backward derivation are baked into the relay-planner agent. This template only passes context.
