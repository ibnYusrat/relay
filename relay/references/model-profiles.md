# Model Profiles

Model profiles control which Claude model each Relay agent uses. This allows balancing quality vs token spend.

## Profile Definitions

| Agent | `quality` | `balanced` | `budget` |
|-------|-----------|------------|----------|
| relay-planner | opus | opus | sonnet |
| relay-ticket-researcher | opus | sonnet | haiku |
| relay-executor | opus | sonnet | sonnet |
| relay-debugger | opus | sonnet | sonnet |
| relay-codebase-mapper | sonnet | haiku | haiku |
| relay-verifier | sonnet | sonnet | haiku |
| relay-plan-checker | sonnet | sonnet | haiku |
| relay-estimator | opus | sonnet | sonnet |
| relay-reviewer | opus | sonnet | sonnet |

## Profile Philosophy

**quality** - Maximum reasoning power
- Opus for all decision-making agents
- Sonnet for read-only verification
- Use when: quota available, critical architecture work

**balanced** (default) - Smart allocation
- Opus only for planning (where architecture decisions happen)
- Sonnet for execution and research (follows explicit instructions)
- Sonnet for verification (needs reasoning, not just pattern matching)
- Use when: normal development, good balance of quality and cost

**budget** - Minimal Opus usage
- Sonnet for anything that writes code
- Haiku for research and verification
- Use when: conserving quota, high-volume work, less critical tasks

## Resolution Logic

Orchestrators resolve model before spawning:

```
1. Read .relay/config.json
2. Get model_profile (default: "balanced")
3. Look up agent in table above
4. Pass model parameter to Task call
```

## Switching Profiles

Runtime: `/relay:set-profile <profile>`

Per-project default: Set in `.relay/config.json`:
```json
{
  "model_profile": "balanced"
}
```
