# Sync Ticket Workflow

How to post results back to the external ticket system with user approval.

## Trigger

Called during Stage 6 (VERIFY & SYNC) of `/relay:work`, after verification completes.

## Input

- Ticket ID
- Integration config from `.relay/config.json`
- SUMMARY.md — execution results
- VERIFICATION.md — acceptance criteria verification results
- Branch name and commit references

## Process

1. **Read results** — load SUMMARY.md and VERIFICATION.md
2. **Format comment** — create a structured comment for the external system
3. **Present to user** — show what will be posted, get approval (gate: `confirm_sync`)
4. **Post comment** — via MCP tool for the configured integration
5. **Update status** — optionally transition ticket status (e.g., "In Review", "Done")

## Comment Format

```markdown
## Relay — Work Summary

**Branch:** {branch_name}
**Commits:** {commit_count}

### What was done

{summary from SUMMARY.md}

### Verification

{pass/fail status for each acceptance criterion from VERIFICATION.md}

### Files Changed

{list of modified files}
```

## MCP Tools

| Integration | Post Comment | Update Status |
|------------|-------------|---------------|
| Jira | `mcp__jira__add_comment` | `mcp__jira__update_issue` |
| GitHub | `mcp__github__create_comment` | `mcp__github__update_issue` |
| Azure DevOps | `mcp__azure_devops__update_work_item` | `mcp__azure_devops__update_work_item` |

## Safety

- **Always requires user confirmation** before posting to external systems
- User can choose: post comment + update status, post comment only, or skip sync entirely
- Never auto-transitions tickets without explicit user approval
