# Claude Code Fundamentals + GitHub Actions CI Lab

## Complete Hands-On Lab Series

This lab series teaches:

1. `CLAUDE.md`
2. Claude Code permissions
3. Claude Code hooks
4. Project-level configuration
5. GitHub repository setup
6. GitHub Actions CI
7. Claude Code in GitHub Actions
8. Security and least privilege
9. Human approval boundaries

## Prerequisites

- GitHub account
- Git
- Node.js/npm
- Claude Code
- Anthropic API key
- Ubuntu/macOS/WSL recommended

Check:

```bash
git --version
node --version
npm --version
claude --version
```

> Claude Pro and Anthropic API access are separate. GitHub Actions can authenticate Claude Code using an Anthropic API key, or supported Claude Code OAuth authentication. Never commit credentials.

---

# Lab 1 — Create a New GitHub Repository

## Objective

Create a clean repository for the labs.

Create a GitHub repository named:

```text
claude-code-ci-lab
```

Clone it:

```bash
git clone https://github.com/YOUR_USERNAME/claude-code-ci-lab.git
cd claude-code-ci-lab
```

Configure Git:

```bash
git config user.name "YOUR NAME"
git config user.email "YOUR EMAIL"
```

Create the structure:

```bash
mkdir -p src tests .claude/hooks .github/workflows
```

Final structure:

```text
claude-code-ci-lab/
├── src/
├── tests/
├── .claude/
│   └── hooks/
└── .github/
    └── workflows/
```

---

# Lab 2 — What Is CLAUDE.md?

## Objective

Understand how `CLAUDE.md` provides project-specific instructions to Claude Code.

Think of it as:

```text
CLAUDE.md
    |
    v
Project instruction manual
```

It can contain:

- Project overview
- Coding conventions
- Important commands
- Testing instructions
- Architecture information
- Security rules
- Files Claude should avoid changing

Create:

```text
CLAUDE.md
```

Add:

```markdown
# Project Instructions

## Project

This is a simple Python calculator application.

## Code Style

- Use Python 3.
- Keep functions small and readable.
- Use meaningful variable names.
- Use type hints where practical.
- Do not add unnecessary dependencies.

## Testing

Run:

```bash
pytest -q
```

## Important Rules

- Do not modify `.github/workflows/` unless explicitly requested.
- Do not commit secrets.
- Do not hardcode API keys or passwords.
- Run tests after modifying application code.
```

Start Claude Code:

```bash
claude
```

Ask:

```text
Read CLAUDE.md and explain the rules you will follow.
```

Then:

```text
Create a simple Python calculator according to CLAUDE.md.
```

Check:

```bash
find . -maxdepth 3 -type f
```

## Key idea

Without `CLAUDE.md`:

```text
Claude → generic coding behavior
```

With `CLAUDE.md`:

```text
Claude → project-specific instructions → more consistent work
```

---

# Lab 3 — Directory-Level CLAUDE.md

## Objective

Understand how more specific instructions can be added for directories.

Create:

```text
src/CLAUDE.md
```

```markdown
# Source Code Instructions

- Keep business logic inside Python functions.
- Do not put test code in this directory.
- Prefer standard library functionality when practical.
```

Create:

```text
tests/CLAUDE.md
```

```markdown
# Test Instructions

- Use pytest.
- Tests should be independent.
- Test normal cases and invalid inputs.
```

Structure:

```text
claude-code-ci-lab/
├── CLAUDE.md
├── src/
│   ├── CLAUDE.md
│   └── calculator.py
└── tests/
    ├── CLAUDE.md
    └── test_calculator.py
```

Ask Claude:

```text
Explain which instructions apply when modifying src/calculator.py.
```

Then:

```text
Explain which instructions apply when modifying tests/test_calculator.py.
```

---

# Lab 4 — What Are Claude Code Permissions?

## Objective

Understand permissions as access control.

Think:

```text
Claude wants to use a tool
        |
        v
Permission system
   /      |      Allow    Ask     Deny
```

Claude Code uses permissions to control tool usage. Project settings can contain allow/deny rules.

Start:

```bash
claude
```

Run:

```text
/permissions
```

Try:

```text
Run pwd
```

Then:

```text
Run git status
```

Then:

```text
Create demo.txt containing "Hello Claude".
```

Observe when Claude asks for approval.

## Important lesson

Do not solve every permission prompt with:

```text
Allow everything
```

Prefer:

```text
Least privilege
```

---

# Lab 5 — Configure Project Permissions

## Objective

Create a project-level permission policy.

Create:

```text
.claude/settings.json
```

Example:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git status)",
      "Bash(git diff)",
      "Bash(pytest:*)"
    ]
  }
}
```

The intent is:

```text
Read       → allowed
Glob       → allowed
Grep       → allowed
git status → allowed
git diff   → allowed
pytest     → allowed
```

But unrelated destructive commands should not be automatically approved.

Test:

```text
Run git status.
```

Then:

```text
Run pytest -q.
```

Then carefully test the permission boundary with a harmless command that is not in the allowlist.

> Permission matching can change between Claude Code versions. Use `/permissions` and the current Claude Code documentation to verify syntax supported by your installed version.

---

# Lab 6 — What Are Hooks?

## Objective

Understand hooks and how they differ from permissions.

### Permissions

Answer:

```text
Can Claude perform this action?
```

### Hooks

Answer:

```text
What deterministic logic should run around an agent action?
```

Common lifecycle events include:

```text
SessionStart
UserPromptSubmit
PreToolUse
PostToolUse
PermissionRequest
Stop
PostCompact
```

Examples:

```text
PreToolUse
    ↓
Log/check a command

PostToolUse
    ↓
Run formatter

SessionStart
    ↓
Load project context
```

---

# Lab 7 — Create a PreToolUse Hook

## Objective

Log when Claude is about to use Bash.

Update `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git status)",
      "Bash(git diff)",
      "Bash(pytest:*)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Claude is about to execute Bash'"
          }
        ]
      }
    ]
  }
}
```

Start:

```bash
claude
```

Ask:

```text
Run git status.
```

Observe the hook.

---

# Lab 8 — Create a PostToolUse Formatting Hook

## Objective

Automatically format Python code after Claude writes or edits it.

Install Black:

```bash
pip install black
```

Create:

```text
.claude/hooks/format.sh
```

```bash
#!/bin/bash

black src tests
```

Make executable:

```bash
chmod +x .claude/hooks/format.sh
```

Configure:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "./.claude/hooks/format.sh"
          }
        ]
      }
    ]
  }
}
```

Ask Claude:

```text
Create a poorly formatted Python function in src/calculator.py.
```

The hook should execute Black after the file edit.

---

# Lab 9 — Build the Calculator

## Objective

Create a small application for CI.

Create:

```text
src/calculator.py
```

```python
def add(a: float, b: float) -> float:
    return a + b


def subtract(a: float, b: float) -> float:
    return a - b


def multiply(a: float, b: float) -> float:
    return a * b


def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")

    return a / b
```

Create:

```text
tests/test_calculator.py
```

```python
from src.calculator import add, subtract, multiply, divide


def test_add():
    assert add(2, 3) == 5


def test_subtract():
    assert subtract(5, 3) == 2


def test_multiply():
    assert multiply(4, 3) == 12


def test_divide():
    assert divide(10, 2) == 5


def test_divide_by_zero():
    try:
        divide(10, 0)
        assert False
    except ValueError:
        assert True
```

Install:

```bash
pip install pytest black
```

Run:

```bash
pytest -q
```

Format:

```bash
black src tests
```

---

# Lab 10 — Push the Project to GitHub

Create `.gitignore`:

```text
venv/
__pycache__/
.pytest_cache/
.env
*.pyc
```

Then:

```bash
git status
git add .
git commit -m "Add Claude Code training project"
git push -u origin main
```

---

# Lab 11 — Create Normal GitHub Actions CI

## Objective

First build deterministic CI. Claude will be added afterward.

Create:

```text
.github/workflows/ci.yml
```

```yaml
name: Python CI

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pytest black

      - name: Format check
        run: |
          black --check src tests

      - name: Run tests
        run: |
          pytest -q
```

Commit:

```bash
git add .github/workflows/ci.yml
git commit -m "Add Python CI"
git push
```

Open:

```text
GitHub → Actions
```

Verify the workflow.

---

# Lab 12 — Add the Anthropic API Secret

## Objective

Securely provide Claude authentication to GitHub Actions.

Go to:

```text
Repository
→ Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

Name:

```text
ANTHROPIC_API_KEY
```

Value:

```text
YOUR_ANTHROPIC_API_KEY
```

Never write this:

```yaml
anthropic_api_key: "sk-ant-..."
```

Use:

```yaml
anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

The official Anthropic setup recommends repository secrets for API-key authentication. citeturn1search4

---

# Lab 13 — Integrate Claude Code into GitHub Actions

## Objective

Run Claude automatically as part of CI.

Create:

```text
.github/workflows/claude-review.yml
```

```yaml
name: Claude Code Review

on:
  pull_request:
    types:
      - opened
      - synchronize

jobs:
  claude-review:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Run Claude Code
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Review this pull request.

            Focus on:
            1. Bugs
            2. Security problems
            3. Error handling
            4. Missing tests
            5. Obvious performance issues

            Do not modify code.
            Provide concise findings.
```

The official Anthropic action supports prompt-driven workflows and the `anthropic_api_key` input. citeturn1search0turn1search6

---

# Lab 14 — Create a Pull Request

Create a branch:

```bash
git checkout -b test-claude-review
```

Introduce an intentional bug:

```python
def divide(a: float, b: float) -> float:
    return a / b
```

Commit:

```bash
git add .
git commit -m "Test Claude review"
git push -u origin test-claude-review
```

Create a Pull Request.

Observe:

```text
Pull Request
      |
      v
GitHub Actions
      |
      v
Claude Code
      |
      v
Review
```

---

# Lab 15 — Make Claude Follow CLAUDE.md in CI

Update `CLAUDE.md`:

```markdown
# Project Instructions

## Project

This is a Python calculator application.

## Code Style

- Use Python 3.
- Use type hints.
- Keep functions small.
- Prefer readable code.

## Testing

Always run:

```bash
pytest -q
```

## Review Rules

When reviewing code:

- Identify bugs first.
- Check error handling.
- Check test coverage.
- Check security issues.
- Do not modify code during review.
- Do not expose secrets.
```

Because the action runs Claude Code in the repository working directory, project configuration such as `CLAUDE.md` and `.claude/` can be read by Claude Code. citeturn1search0

---

# Lab 16 — Use Permissions in CI

## Objective

Understand why a CI review agent should have limited capabilities.

For a review-only job:

```text
Read       → needed
Glob       → needed
Grep       → needed
git diff   → needed
pytest     → possibly needed

Write      → not needed
Edit       → not needed
Deploy     → not needed
Production → not needed
```

The Anthropic base action supports settings and tool restrictions. citeturn1search0

Example `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git status)",
      "Bash(git diff)",
      "Bash(pytest:*)"
    ],
    "deny": [
      "WebFetch"
    ]
  }
}
```

For a custom workflow, the base action can receive a settings file:

```yaml
- name: Run Claude Code
  uses: anthropics/claude-code-base-action@beta
  with:
    prompt: "Review this repository."
    settings: ".claude/settings.json"
    claude_args: '--allowedTools "Read,Glob,Grep,Bash(git:*)"'
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

Use the current Anthropic action documentation to verify the exact action/version and supported permission syntax in your environment. citeturn1search0turn1search10

---

# Lab 17 — Add a Claude Security Review

Create:

```text
.github/workflows/claude-security.yml
```

```yaml
name: Claude Security Review

on:
  pull_request:

jobs:
  security-review:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Claude Security Review
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Perform a security-focused code review.

            Check for:
            - Hardcoded secrets
            - Unsafe shell execution
            - Injection risks
            - Insecure file handling
            - Authentication problems
            - Authorization problems
            - Sensitive information exposure

            Do not modify files.
            Report findings with severity:
            HIGH, MEDIUM, or LOW.
```

---

# Lab 18 — Claude + Deterministic CI

The final CI architecture should be:

```text
                    Pull Request
                         |
             +-----------+-----------+
             |                       |
             v                       v
        Normal CI                Claude Code
             |                       |
      +------+------+          +-----+------+
      |             |          |            |
     Build         Test     CLAUDE.md   Permissions
                                  |
                                  v
                                Hooks
                                  |
                                  v
                              AI Review
```

Use normal CI for:

```text
Unit tests
Build
Lint
Type checking
Security scanners
Deployment checks
```

Use Claude for:

```text
Code review
Failure explanation
Test suggestions
Architecture review
Security analysis
Change summaries
```

Claude should complement deterministic CI rather than replace it.

---

# Lab 19 — Security Threat Model

An AI agent running in CI can potentially:

```text
Read repository files
Run commands
Modify files
Access environment variables
Access credentials
Interact with external services
```

Important threats:

### Threat 1 — Prompt injection

A malicious PR could contain instructions intended to manipulate the agent.

### Threat 2 — Secret exposure

An agent might be tricked into revealing sensitive environment data.

### Threat 3 — Unsafe commands

An agent may attempt commands that should not run in CI.

### Threat 4 — Excessive GitHub permissions

A workflow with unnecessary write permissions increases blast radius.

### Threat 5 — Production access

A coding agent should not automatically receive production credentials.

### Threat 6 — Untrusted PRs

Public repositories and fork PRs require special care because their content may be untrusted.

The official Anthropic Action documentation explicitly discusses trust boundaries and recommends the main Claude Code Action for workflows processing untrusted inputs such as public issues or fork pull requests. citeturn1search0turn1search9

---

# Lab 20 — Human Approval Boundary

Avoid:

```text
Claude
  |
  v
Production
```

Prefer:

```text
Claude
  |
  v
Create / modify PR
  |
  v
CI
  |
  v
Human review
  |
  v
Approval
  |
  v
Production
```

This separates:

```text
AI assistance
```

from:

```text
Production authorization
```

---

# Lab 21 — Final Repository

Your repository should look approximately like:

```text
claude-code-ci-lab/
│
├── CLAUDE.md
│
├── src/
│   ├── CLAUDE.md
│   └── calculator.py
│
├── tests/
│   ├── CLAUDE.md
│   └── test_calculator.py
│
├── .claude/
│   ├── settings.json
│   └── hooks/
│       └── format.sh
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── claude-review.yml
│       └── claude-security.yml
│
└── .gitignore
```

---

# Lab 22 — Complete End-to-End Exercise

Create:

```bash
git checkout -b feature-square-root
```

Ask Claude Code:

```text
Read CLAUDE.md.

Implement a square_root function in src/calculator.py.

Add appropriate unit tests.

Run the tests.

Do not modify .github/workflows/.
```

Review:

```bash
git diff
```

Run:

```bash
pytest -q
```

Commit:

```bash
git add .
git commit -m "Add square root calculation"
git push -u origin feature-square-root
```

Create a Pull Request.

Now observe:

```text
Developer
    |
    v
Pull Request
    |
    +----------------------+
    |                      |
    v                      v
Python CI             Claude Review
    |                      |
    +-- Black              +-- CLAUDE.md
    +-- pytest             +-- permissions
                           +-- hooks
                           +-- code review
                                  |
                                  v
                            Human decision
```

---

# Lab 23 — Troubleshooting

## `claude: command not found`

Check:

```bash
claude --version
```

Install/update Claude Code using the current official installation instructions.

## API authentication failure

Check locally:

```bash
test -n "$ANTHROPIC_API_KEY" && echo "API key exists"
```

Do not print the actual key.

For GitHub Actions check:

```text
Repository
→ Settings
→ Secrets and variables
→ Actions
→ ANTHROPIC_API_KEY
```

## Workflow permission problems

Check:

```yaml
permissions:
  contents: read
  pull-requests: write
```

Only grant permissions actually required.

## Hook does not execute

Check:

```bash
cat .claude/settings.json
```

Verify:

- Event name
- Matcher
- Tool name
- JSON syntax
- Script executable permission

Run:

```bash
chmod +x .claude/hooks/format.sh
```

---

# Quick Revision

## CLAUDE.md

```text
Project instructions
```

It tells Claude:

```text
How the project works
What conventions to follow
What commands to use
What rules to respect
```

## Permissions

```text
Access control
```

They answer:

```text
Can Claude perform this action?
```

## Hooks

```text
Deterministic automation around agent events
```

Examples:

```text
PreToolUse
PostToolUse
SessionStart
Stop
```

## GitHub Actions

```text
Automation environment
```

It allows Claude Code to participate in CI/CD workflows.

## Claude Code Action

```text
GitHub Actions
      |
      v
Claude Code
      |
      v
Repository
```

---

# Production Checklist

## Claude Code

- [ ] `CLAUDE.md`
- [ ] Project-specific instructions
- [ ] Directory-specific instructions where useful
- [ ] Minimal permissions
- [ ] Hooks for deterministic checks
- [ ] Bounded agent behavior

## GitHub Actions

- [ ] API key stored in GitHub Secrets
- [ ] Minimal GitHub permissions
- [ ] No credentials in YAML
- [ ] Separate review and deployment responsibilities
- [ ] Pull request validation
- [ ] Human approval for sensitive operations

## Security

- [ ] No secrets committed
- [ ] No unrestricted shell access
- [ ] No unnecessary write permissions
- [ ] Untrusted PR inputs considered
- [ ] Production credentials isolated
- [ ] Agent actions observable/auditable

---

# Final Mental Model

```text
CLAUDE.md
    =
"What should Claude know about this project?"

Permissions
    =
"What is Claude allowed to do?"

Hooks
    =
"What automatic logic should run around Claude's actions?"

GitHub Actions
    =
"Where and when should Claude run automatically?"

Secrets
    =
"How do we securely provide credentials?"

Human approval
    =
"Who authorizes sensitive changes?"
```

The goal is not simply:

```text
"Put Claude in GitHub Actions."
```

The real production goal is:

```text
                Safe Agentic CI

       +-----------------------------+
       |       GitHub Actions        |
       |                             |
       |  CLAUDE.md                  |
       |      ↓                      |
       |  Claude Code                |
       |      ↓                      |
       |  Permissions                |
       |      ↓                      |
       |  Hooks                      |
       |      ↓                      |
       |  Code / Tests / Review      |
       |      ↓                      |
       |  Human Approval             |
       +-------------+---------------+
                     |
                     v
                 Deployment
```
