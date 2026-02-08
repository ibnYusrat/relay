---
name: set-profile
description: Switch model profile for Relay agents (quality/balanced/budget)
arguments:
  - name: profile
    description: "Profile name: quality, balanced, or budget"
    required: true
---

<objective>
Switch the model profile used by Relay agents. This controls which Claude model each agent uses, balancing quality vs token spend.
</objective>

<profiles>
| Profile | Description |
|---------|-------------|
| **quality** | Opus everywhere except read-only verification |
| **balanced** | Opus for planning, Sonnet for execution/verification (default) |
| **budget** | Sonnet for writing, Haiku for research/verification |
</profiles>

<process>

## 1. Validate argument

```
if $ARGUMENTS.profile not in ["quality", "balanced", "budget"]:
  Error: Invalid profile "$ARGUMENTS.profile"
  Valid profiles: quality, balanced, budget
  STOP
```

## 2. Ensure config exists

```bash
ls .relay/config.json 2>/dev/null
```

If `.relay/config.json` missing, create it with defaults:
```bash
mkdir -p .relay
```
```json
{
  "model_profile": "balanced",
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  }
}
```
Write this to `.relay/config.json`, then continue.

## 3. Update config.json

Read current config:
```bash
cat .relay/config.json
```

Update `model_profile` field:
```json
{
  "model_profile": "$ARGUMENTS.profile"
}
```

Write updated config back to `.relay/config.json`.

## 4. Confirm

```
✓ Model profile set to: $ARGUMENTS.profile

Agents will now use:
[Show table from model-profiles.md for selected profile]

Next spawned agents will use the new profile.
```

</process>

<examples>

**Switch to budget mode:**
```
/relay:set-profile budget

✓ Model profile set to: budget

Agents will now use:
| Agent | Model |
|-------|-------|
| relay-planner | sonnet |
| relay-executor | sonnet |
| relay-verifier | haiku |
| ... | ... |
```

**Switch to quality mode:**
```
/relay:set-profile quality

✓ Model profile set to: quality

Agents will now use:
| Agent | Model |
|-------|-------|
| relay-planner | opus |
| relay-executor | opus |
| relay-verifier | sonnet |
| ... | ... |
```

</examples>
