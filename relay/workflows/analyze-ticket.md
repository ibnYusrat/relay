# Analyze Ticket Workflow

How to produce ANALYSIS.md from a ticket + codebase context.

## Trigger

Called during Stage 2 (ANALYZE) of `/relay:work`.

## Input

- Ticket data from external system (ID, title, description, type, priority, acceptance criteria)
- Codebase context from `.relay/codebase/` (if available)

## Process

1. **Parse ticket** — extract requirements, acceptance criteria, type
2. **Explore codebase** — find related files, existing patterns, test structure
3. **Map impact** — which files need changes, what tests need updates
4. **Assess risks** — breaking changes, dependencies, performance
5. **Propose approach** — high-level technical strategy
6. **Surface questions** — ambiguities needing answers
7. **Write ANALYSIS.md** — to `.relay/tickets/{TICKET_ID}/ANALYSIS.md`

## Output

`ANALYSIS.md` in the ticket directory, consumed by `relay-planner` for plan creation.

## Agent

Spawns `relay-ticket-researcher` agent which:
- Has full codebase access (Read, Grep, Glob, Bash)
- Can research via WebSearch/WebFetch/Context7
- Writes ANALYSIS.md directly to disk
- Returns structured result to orchestrator

## Quality Criteria

- Acceptance criteria are explicit and verifiable
- Impact table references actual files in the codebase
- Technical approach follows existing codebase patterns
- Risks have concrete mitigations
- Questions are specific and actionable
