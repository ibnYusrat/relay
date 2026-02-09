---
name: relay:settings
description: Configure Relay workflow toggles and model profile
allowed-tools:
  - Read
  - Write
  - AskUserQuestion
---

<objective>
Allow users to toggle workflow agents on/off and select model profile via interactive settings.

Updates `.relay/config.json` with workflow preferences and model profile selection.
</objective>

<process>

## 1. Ensure config exists

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

## 2. Read Current Config

```bash
cat .relay/config.json
```

Parse current values (default to `true` if not present):
- `workflow.research` — spawn researcher during plan-phase
- `workflow.plan_check` — spawn plan checker during plan-phase
- `workflow.verifier` — spawn verifier during execute-phase
- `model_profile` — which model each agent uses (default: `balanced`)
- `git.branching_strategy` — branching approach (default: `"none"`)
- `integration.type` — connected ticket system (jira, github, azure_devops, or null)
- `integration.project_key` — project identifier for ticket system
- `code_hosting.type` — code hosting provider (github, gitlab, bitbucket, or null)
- `code_hosting.cli` — hosting CLI binary (gh, glab, or null)

## 3. Present Settings

Use AskUserQuestion with current values shown:

```
AskUserQuestion([
  {
    question: "Which model profile for agents?",
    header: "Model",
    multiSelect: false,
    options: [
      { label: "Quality", description: "Opus everywhere except verification (highest cost)" },
      { label: "Balanced (Recommended)", description: "Opus for planning, Sonnet for execution/verification" },
      { label: "Budget", description: "Sonnet for writing, Haiku for research/verification (lowest cost)" }
    ]
  },
  {
    question: "Spawn Plan Researcher? (researches domain before planning)",
    header: "Research",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Research phase goals before planning" },
      { label: "No", description: "Skip research, plan directly" }
    ]
  },
  {
    question: "Spawn Plan Checker? (verifies plans before execution)",
    header: "Plan Check",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify plans meet phase goals" },
      { label: "No", description: "Skip plan verification" }
    ]
  },
  {
    question: "Spawn Execution Verifier? (verifies phase completion)",
    header: "Verifier",
    multiSelect: false,
    options: [
      { label: "Yes", description: "Verify must-haves after execution" },
      { label: "No", description: "Skip post-execution verification" }
    ]
  },
  {
    question: "Git branching strategy?",
    header: "Branching",
    multiSelect: false,
    options: [
      { label: "None (Recommended)", description: "Commit directly to current branch" },
      { label: "Per Phase", description: "Create branch for each phase (relay/phase-{N}-{name})" },
      { label: "Per Ticket", description: "Create branch for each ticket ({ticket_id}/{slug})" }
    ]
  }
])
```

**Pre-select based on current config values.**

## 4. Update Config

Merge new settings into existing config.json:

```json
{
  ...existing_config,
  "model_profile": "quality" | "balanced" | "budget",
  "workflow": {
    "research": true/false,
    "plan_check": true/false,
    "verifier": true/false
  },
  "git": {
    "branching_strategy": "none" | "phase" | "ticket"
  }
}
```

Write updated config to `.relay/config.json`.

## 5. Confirm Changes

Display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Relay ► SETTINGS UPDATED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| Setting              | Value |
|----------------------|-------|
| Model Profile        | {quality/balanced/budget} |
| Plan Researcher      | {On/Off} |
| Plan Checker         | {On/Off} |
| Execution Verifier   | {On/Off} |
| Git Branching        | {None/Per Phase/Per Ticket} |
| Code Hosting         | {github/gitlab/bitbucket or "None"} |

These settings apply to future `/relay:work` runs.

Quick commands:
- /relay:set-profile <profile> — switch model profile
- /relay:setup — reconfigure integration or code hosting
```

</process>

<success_criteria>
- [ ] Current config read
- [ ] User presented with 5 settings (profile + 3 workflow toggles + git branching)
- [ ] Config updated with model_profile, workflow, and git sections
- [ ] Changes confirmed to user
</success_criteria>
