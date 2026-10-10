# Lab 0: Set up your Windows lab environment

| | |
|---|---|
| **Course** | Claude Certified Architect – Foundations |
| **Duration** | 20–25 minutes (do this once, before Lab 1) |
| **Level** | Beginner |
| **Shell** | Windows PowerShell (or Windows Terminal with PowerShell) |
| **Cost** | A few cents for the smoke test |

## What you will set up

A Python virtual environment with the Anthropic SDK, the Claude Code CLI, and an API key that both of them can use. Every later lab starts from this setup.

```
  Your PowerShell
     |-- python + .venv  --> anthropic SDK, mcp, jsonschema   (Labs 1-4, 6-8)
     |-- claude (CLI)    --> Claude Code                      (Labs 4-6)
     `-- ANTHROPIC_API_KEY (environment variable) used by both
```

## Steps

### 1. Open PowerShell and check the version

Press **Win**, type `PowerShell`, press Enter.

```powershell
$PSVersionTable.PSVersion
```

*Why:* Windows PowerShell 5.1 and PowerShell 7 both work. This only confirms the shell runs.

### 2. Install Python 3.12 or newer

```powershell
python --version
```

If this prints a version below 3.11 or an error, install Python, then **close and reopen PowerShell**:

```powershell
winget install Python.Python.3.12
```

*Why:* the labs use modern type hints (`list[str]`, `str | None`). Reopening the shell refreshes `PATH`.

### 3. Install Claude Code

```powershell
irm https://claude.ai/install.ps1 | iex
claude --version
```

*Why:* this is the native Windows installer. If `claude` is not found, reopen PowerShell. Alternatives: `winget install Anthropic.ClaudeCode`, or `npm install -g @anthropic-ai/claude-code` if you have Node.js 22 or newer.

### 4. Create the course folder and a virtual environment

```powershell
mkdir C:\cca-labs
cd C:\cca-labs
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

*Why:* a virtual environment keeps lab packages away from your system Python. Your prompt should now start with `(.venv)`.

If PowerShell says *running scripts is disabled*, allow local scripts for your user and activate again:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
.\.venv\Scripts\Activate.ps1
```

### 5. Install the Python packages

```powershell
python -m pip install --upgrade pip
pip install anthropic jsonschema httpx uvicorn pytest "mcp>=2"
```

*Why:* `anthropic` is the SDK. `jsonschema` is used in Lab 7. `mcp>=2` and `uvicorn` are used in Lab 4. The `>=2` matters because version 2 of the MCP package renamed its server class; the lab code is written for it.

### 6. Set your API key

Use the key from the Anthropic Console. Set it for this window:

```powershell
$env:ANTHROPIC_API_KEY = Read-Host "Paste your API key"
```

To keep it for future windows (takes effect in new windows only):

```powershell
setx ANTHROPIC_API_KEY "paste-your-key-here"
```

*Why:* both the Python SDK and Claude Code read `ANTHROPIC_API_KEY` automatically, so you never put the key in code. Never paste the key into a file you commit, and never share it in chat. In the Console, set a monthly spend limit for the training workspace.

### 7. Smoke-test the Python SDK

Create the file and paste the code:

```powershell
notepad hello.py
```

```python
import os

import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY
resp = client.messages.create(
    model=os.environ.get("CLAUDE_MODEL", "claude-sonnet-5-5"),
    max_tokens=50,
    messages=[{"role": "user", "content": "Reply with exactly: lab ready"}],
)
print(resp.content[0].text)
```

```powershell
python hello.py
```

Expected output: `lab ready` (wording may vary slightly).

*Why:* proves the key, network, SDK and model name all work. Labs read the model from the `CLAUDE_MODEL` environment variable and default to `claude-sonnet-5-5`. To use another model: `$env:CLAUDE_MODEL = "model-name"`.

### 8. Smoke-test Claude Code

```powershell
claude -p "Reply with the single word: ready"
claude auth status
```

*Why:* `-p` is headless (print) mode, the same mode you use in CI in Lab 6. If Claude Code asks how to authenticate, choose the API key / Console option. `claude auth status` shows which method is active. If it reports a subscription login but you want API billing, run `claude auth login --console`.

### 9. Know your lab folder layout

Each lab uses its own folder under `C:\cca-labs`:

```
C:\cca-labs\
  .venv\          <- shared virtual environment
  lab01\ ... lab08\
```

At the start of every lab session, run:

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
```

## Verify

- `python --version` shows 3.11 or newer.
- `claude --version` prints a version.
- `python hello.py` prints a reply from Claude.
- `claude -p "Reply with the single word: ready"` prints `ready`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `python` opens the Microsoft Store | Install with `winget`, or disable the Python app alias in *Settings > Apps > Advanced app settings > App execution aliases*. |
| `claude` is not recognized | Close and reopen PowerShell. Check `$env:PATH` contains the Claude Code install folder, or use the winget/npm method. |
| `401 authentication_error` | The key is wrong, expired, or not set in this window. Run step 6 again. |
| `404 not_found_error` on the model | The model name is wrong for your account. Set `$env:CLAUDE_MODEL` to a model you can use. |
| `Activate.ps1 cannot be loaded` | Run the `Set-ExecutionPolicy` command in step 4. |
| `pip` cannot find `mcp>=2` | Upgrade Python to 3.11+ and pip, then retry. |
| Corporate proxy blocks calls | Ask IT for the proxy address and set `$env:HTTPS_PROXY`. |

## Cleanup

Nothing to clean up. Keep the environment for all eight labs.
