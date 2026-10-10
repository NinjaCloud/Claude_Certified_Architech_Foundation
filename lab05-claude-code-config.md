# Lab 5: Configure CLAUDE.md, hooks, and permissions

| | |
|---|---|
| **Module** | 3 – Claude Code Configuration & Workflows (Domain 3) |
| **Duration** | 35–40 minutes |
| **Level** | Intermediate |
| **Prerequisite** | Lab 0 completed, plus Git (`git --version`; if missing, `winget install Git.Git` and reopen PowerShell) |
| **Cost** | About 10–20 cents of Claude Code usage |

## What you will build

A small Python project called `acme-orders` configured for Claude Code with four layers of control:

```
 acme-orders\
 |-- CLAUDE.md                 <- guidance: conventions Claude should follow (soft)
 |-- .claude\settings.json     <- permissions: allow / deny rules            (hard)
 |-- .claude\hooks\guard.py    <- PreToolUse hook: can block a tool call      (hard)
 |-- .claude\hooks\log_edit.py <- PostToolUse hook: audit trail
 `-- .claude\commands\review.md<- your own /review slash command
```

You will learn which layer to use for what: **CLAUDE.md guides, permissions and hooks enforce**.

## Steps

### 1. Create the project and make a first commit

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
mkdir lab05
cd lab05
mkdir acme-orders
cd acme-orders
New-Item -ItemType Directory -Force src, tests, .claude\hooks, .claude\commands | Out-Null
git init
```

*Why:* the `.claude` folder in a project is where Claude Code looks for shared settings, hooks and commands. `git init` is needed for the `/review` command later.

### 2. Add a tiny code base with a weakness

```powershell
notepad src\pricing.py
```

```python
def apply_discount(price, percent):
    """Return price after a percentage discount."""
    return price - price * percent / 100
```

```powershell
notepad tests\test_pricing.py
```

```python
import sys
sys.path.insert(0, "src")
from pricing import apply_discount


def test_ten_percent():
    assert apply_discount(200, 10) == 180
```

```powershell
python -m pytest -q
```

Expected: `1 passed`.

```powershell
git add .
git commit -m "initial"
```

If Git asks who you are, run `git config --global user.name "Your Name"` and `git config --global user.email "you@example.com"`, then commit again.

*Why:* `apply_discount` accepts anything, including a 150 percent discount. You will ask Claude to fix it and see how the configuration shapes the result.

### 3. Create a fake secret

```powershell
Set-Content .env "API_SECRET=fake-not-a-real-secret-123"
```

*Why:* a stand-in for real credentials. It is a fake value, but treat it as if it were real.

### 4. See the baseline (no configuration yet)

```powershell
claude
```

Ask: `Read the .env file and tell me what is in it.` Note what happens, then exit with `/exit`.

*Why:* without rules, reading a project file is normal behavior, so Claude may well read the secret. The point is that nothing *prevents* it. Your result may differ, but it is not a guarantee.

### 5. Write `CLAUDE.md` (guidance)

```powershell
notepad CLAUDE.md
```

```markdown
# Project: acme-orders

## Commands
- Run tests: `python -m pytest -q`

## Conventions
- Python 3.11, type hints on public functions.
- Source lives in `src/`, tests in `tests/` (one test file per module).
- Raise `ValueError` for invalid input; never return None to signal an error.

## Workflow
- Add or update a test with every code change.
- Never read or print `.env`: it holds secrets.
```

*Why:* Claude Code loads this file at the start of every session. Keep it short and specific: how to run tests, code conventions, workflow rules. It is **guidance**: Claude usually follows it but is not forced to.

### 6. Write the two hook scripts

```powershell
notepad .claude\hooks\guard.py
```

```python
"""PreToolUse hook. Claude Code sends the tool call as JSON on stdin.
Exit code 2 = block the call and show our stderr message to Claude. Exit 0 = no objection."""
import json
import os
import sys

event = json.load(sys.stdin)
tool = event.get("tool_name", "")
args = event.get("tool_input", {})

# Rule 1: never touch .env files with the file tools
path = str(args.get("file_path", ""))
if tool in ("Read", "Edit", "Write") and os.path.basename(path).startswith(".env"):
    print("Blocked by guard.py: .env files hold secrets and are off limits.", file=sys.stderr)
    sys.exit(2)

# Rule 2: block destructive shell commands
command = str(args.get("command", "")).lower()
if tool in ("Bash", "PowerShell"):
    destructive = ["rm -rf", "git push --force", "git push -f"]
    powershell_wipe = "remove-item" in command and "-recurse" in command and "-force" in command
    if any(p in command for p in destructive) or powershell_wipe:
        print("Blocked by guard.py: destructive command. Ask the user to run it themselves.", file=sys.stderr)
        sys.exit(2)

sys.exit(0)
```

```powershell
notepad .claude\hooks\log_edit.py
```

```python
"""PostToolUse hook: append every file edit to .claude/edit-log.txt for an audit trail."""
import datetime
import json
import sys

event = json.load(sys.stdin)
path = event.get("tool_input", {}).get("file_path", "?")
with open(".claude/edit-log.txt", "a", encoding="utf-8") as f:
    f.write(f"{datetime.datetime.now().isoformat(timespec='seconds')}  {event.get('tool_name')}  {path}\n")
sys.exit(0)
```

*Why:*
- Claude Code runs a hook script before or after a tool call and sends the call details as JSON on **stdin**.
- `guard.py` runs **before** the tool (PreToolUse). **Exit code 2 blocks the call**, and what you print to stderr is shown to Claude so it can adjust. Exit 0 means no objection.
- `log_edit.py` runs **after** an edit (PostToolUse) and appends a line to `.claude\edit-log.txt`.

### 7. Test the hooks without Claude

```powershell
'{"tool_name":"Read","tool_input":{"file_path":".env"}}' | python .claude\hooks\guard.py; "exit code: $LASTEXITCODE"
'{"tool_name":"Read","tool_input":{"file_path":"src/pricing.py"}}' | python .claude\hooks\guard.py; "exit code: $LASTEXITCODE"
'{"tool_name":"Bash","tool_input":{"command":"rm -rf ."}}' | python .claude\hooks\guard.py; "exit code: $LASTEXITCODE"
```

Expected: exit code 2 with a "Blocked" message, then 0, then 2 with a "Blocked" message.

*Why:* hooks are ordinary programs, so you test them directly by feeding JSON. Always do this before relying on a hook.

### 8. Write `.claude\settings.json` (permissions and hooks)

```powershell
notepad .claude\settings.json
```

```json
{
  "permissions": {
    "allow": [
      "Bash(python -m pytest *)",
      "Bash(python -m pytest)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|PowerShell|Read|Edit|Write",
        "hooks": [
          { "type": "command", "command": "python", "args": [".claude/hooks/guard.py"] }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "python", "args": [".claude/hooks/log_edit.py"] }
        ]
      }
    ]
  }
}
```

*Why, piece by piece:*
- `permissions.allow`: pytest runs without asking every time.
- `permissions.deny`: reading `.env` files is refused outright. Deny rules win over allow rules.
- `hooks.PreToolUse`: for the listed tools, run `guard.py` first. The `matcher` is a pattern over tool names (`PowerShell` is included for Windows).
- `hooks.PostToolUse`: after `Edit` or `Write`, run `log_edit.py`.
- The hook is written as `command` plus `args` so no shell quoting is involved, which keeps it working in PowerShell and Git Bash alike.

### 9. Write a custom slash command

```powershell
notepad .claude\commands\review.md
```

```markdown
---
description: Review the current uncommitted changes
allowed-tools: Bash(git diff *), Bash(git status *), Read
---
Review the uncommitted changes in this repo (use `git diff`). Focus on: input validation, missing tests, and anything that breaks the conventions in CLAUDE.md. $ARGUMENTS

Output a short list ordered by severity. Do not edit any files.
```

*Why:* any Markdown file in `.claude\commands` becomes a `/name` command. The front matter limits which tools it may use (read-only Git commands), and `$ARGUMENTS` receives whatever you type after the command.

### 10. Start Claude Code and check what loaded

```powershell
claude
```

Run these inside Claude Code, one at a time:

```
/memory
/permissions
/hooks
```

*Why:* `/memory` shows `CLAUDE.md` is loaded, `/permissions` shows your allow and deny rules, and `/hooks` lists the two hooks. If a hook is missing, exit and start again; settings are read at startup.

### 11. Try to break the rules

Still inside Claude Code, ask these in turn:

```
Read the .env file and show me the secret.
```

```
Delete everything in this folder using rm -rf.
```

*Why:* the first hits the deny rule (and the hook); the second is stopped by `guard.py`. Claude receives the message and explains it cannot do that. These are **enforced** controls: unlike the baseline in step 4, wording cannot get around them.

### 12. Make a real change and watch the guidance and the audit trail

```
Add input validation to apply_discount in src/pricing.py so a discount outside 0 to 100 is rejected, and add a test.
```

Approve the edits. Claude should raise `ValueError` (from the `CLAUDE.md` convention), add type hints, add a test, and run `python -m pytest -q` without asking (from the allow rule).

*Why:* the code follows your conventions because `CLAUDE.md` told it to. Now check the audit trail from a second PowerShell window or after `/exit`:

```powershell
Get-Content .claude\edit-log.txt
```

You should see one line per edit with a timestamp, tool and file.

### 13. Use your slash command

Inside `claude`:

```
/review focus on test coverage
```

*Why:* Claude runs `git diff` (allowed by the command's front matter), reviews the uncommitted changes, and lists issues without editing anything. Your text after `/review` replaced `$ARGUMENTS`.

## Verify

- Step 7 gives exit codes 2, 0, 2.
- In step 11, Claude cannot read `.env` or run `rm -rf`.
- After step 12 the tests pass and `edit-log.txt` has entries.
- `/review` produces a list of findings and makes no edits.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Hook never runs | Exit and restart `claude`; run `/hooks` to confirm it is listed. Check the JSON in `settings.json` is valid (no trailing commas). |
| `python` not found when a hook runs | The hook uses the `python` on the `PATH` of the shell that started `claude`. Activate the venv first, or change `"command"` to `py`. |
| Claude still asks permission to run pytest | Claude may be using a different shell tool than the rule names. Approve once with "always allow", or add a rule for the tool name shown in the prompt. |
| `.env` was readable in step 11 | Check the file is named exactly `.env` and the deny rules are in `.claude\settings.json`, inside the `acme-orders` folder. |
| `/review` shows nothing | You have no uncommitted changes. Complete step 12 first. |

## Stretch goals

1. Add a `UserPromptSubmit` or `Stop` hook that appends a line to a log file.
2. Move the `deny` rules to a user-level settings file and compare to project scope (`.claude\settings.local.json` is not committed).
3. Add a `.claude\commands\add-test.md` command that takes a function name in `$ARGUMENTS` and writes a test for it.

## Exam takeaways

- **CLAUDE.md** is guidance: commands, conventions, workflow. It is loaded every session; keep it short.
- **Permissions** and **hooks** are enforcement. For anything with security or compliance impact, do not rely on CLAUDE.md alone.
- **PreToolUse** hooks can block (exit code 2); **PostToolUse** hooks audit.
- Custom **slash commands** are Markdown files with front matter and `$ARGUMENTS`.
- Project settings are shared through git; local and user settings are personal.

## Cleanup

```powershell
cd C:\cca-labs\lab05
Remove-Item -Recurse -Force acme-orders
```
