---
name: relay:update
description: Update Relay to latest version with changelog display
---

<objective>
Check for Relay updates, install if available, and display what changed.

Provides a better update experience than raw `npx relay-cc` by showing version diff and changelog entries.
</objective>

<process>

<step name="get_installed_version">
Detect whether Relay is installed locally or globally by checking both locations:

```bash
# Check local first (takes priority)
if [ -f "./.claude/relay/VERSION" ]; then
  cat "./.claude/relay/VERSION"
  echo "LOCAL"
elif [ -f ~/.claude/relay/VERSION ]; then
  cat ~/.claude/relay/VERSION
  echo "GLOBAL"
else
  echo "UNKNOWN"
fi
```

Parse output:
- If last line is "LOCAL": installed version is first line, use `--local` flag for update
- If last line is "GLOBAL": installed version is first line, use `--global` flag for update
- If "UNKNOWN": proceed to install step (treat as version 0.0.0)

**If VERSION file missing:**
```
## Relay Update

**Installed version:** Unknown

Your installation doesn't include version tracking.

Running fresh install...
```

Proceed to install step (treat as version 0.0.0 for comparison).
</step>

<step name="check_latest_version">
Check npm for latest version:

```bash
npm view relay-cc version 2>/dev/null
```

**If npm check fails:**
```
Couldn't check for updates (offline or npm unavailable).

To update manually: `npx relay-cc --global`
```

STOP here if npm unavailable.
</step>

<step name="compare_versions">
Compare installed vs latest:

**If installed == latest:**
```
## Relay Update

**Installed:** X.Y.Z
**Latest:** X.Y.Z

You're already on the latest version.
```

STOP here if already up to date.

**If installed > latest:**
```
## Relay Update

**Installed:** X.Y.Z
**Latest:** A.B.C

You're ahead of the latest release (development version?).
```

STOP here if ahead.
</step>

<step name="show_changes_and_confirm">
**If update available**, fetch and show what's new BEFORE updating:

1. Fetch the changelog from `https://raw.githubusercontent.com/ibnYusrat/relay/main/CHANGELOG.md`
2. Extract entries between installed and latest versions
3. Display preview and ask for confirmation:

```
## Relay Update Available

**Installed:** 1.5.10
**Latest:** 1.5.15

### What's New
────────────────────────────────────────────────────────────

## [1.5.15] - 2026-01-20

### Added
- Feature X

## [1.5.14] - 2026-01-18

### Fixed
- Bug fix Y

────────────────────────────────────────────────────────────

⚠️  **Note:** The installer performs a clean install of Relay folders:
- `commands/relay/` will be wiped and replaced
- `relay/` will be wiped and replaced
- `agents/relay-*` files will be replaced

(Paths are relative to your install location: `~/.claude/` for global, `./.claude/` for local)

Your custom files in other locations are preserved:
- Custom commands not in `commands/relay/` ✓
- Custom agents not prefixed with `relay-` ✓
- Custom hooks ✓
- Your CLAUDE.md files ✓

If you've modified any Relay files directly, back them up first.
```

Use AskUserQuestion:
- Question: "Proceed with update?"
- Options:
  - "Yes, update now"
  - "No, cancel"

**If user cancels:** STOP here.
</step>

<step name="run_update">
Run the update using the install type detected in step 1:

**If LOCAL install:**
```bash
npx relay-cc --local
```

**If GLOBAL install (or unknown):**
```bash
npx relay-cc --global
```

Capture output. If install fails, show error and STOP.

Clear the update cache so statusline indicator disappears:

**If LOCAL install:**
```bash
rm -f ./.claude/cache/relay-update-check.json
```

**If GLOBAL install:**
```bash
rm -f ~/.claude/cache/relay-update-check.json
```
</step>

<step name="display_result">
Format completion message (changelog was already shown in confirmation step):

```
╔═══════════════════════════════════════════════════════════╗
║  Relay Updated: v1.5.10 → v1.5.15                           ║
╚═══════════════════════════════════════════════════════════╝

⚠️  Restart Claude Code to pick up the new commands.

[View full changelog](https://github.com/ibnyusrat/relay/blob/main/CHANGELOG.md)
```
</step>

</process>

<success_criteria>
- [ ] Installed version read correctly
- [ ] Latest version checked via npm
- [ ] Update skipped if already current
- [ ] Changelog fetched and displayed BEFORE update
- [ ] Clean install warning shown
- [ ] User confirmation obtained
- [ ] Update executed successfully
- [ ] Restart reminder shown
</success_criteria>
