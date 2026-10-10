# Lab 6: Integrate Claude Code into a CI pipeline

| | |
|---|---|
| **Module** | 3 – Claude Code Configuration & Workflows (Domain 3) |
| **Duration** | 35–40 minutes |
| **Level** | Intermediate |
| **Prerequisite** | Lab 0 completed, Git installed (`git --version`) |
| **Cost** | About 10–30 cents (each review is capped at 50 cents) |

## What you will build

A **security-review gate** for pull requests. A Python script runs Claude Code in headless mode, forces a JSON result that matches a schema, and exits non-zero when it finds a serious problem. You run it on your laptop, then read the GitHub Actions workflow that runs the same script in CI.

```
 PR opened
    |
    v
 +--------------------------+      claude -p "review ..."
 |  GitHub Actions job      |      --output-format json
 |  python ci_review.py     | ---> --json-schema review_schema.json
 +------------+-------------+      --tools Read,Grep,Glob   (read-only)
              |                    --permission-mode dontAsk
              v                    --max-turns / --max-budget-usd
     review.json + review.md
              |
   high or critical finding? --> exit 1 (job fails, PR blocked)
   otherwise                 --> exit 0
```

You will learn: headless mode (`-p`), machine-readable output, schema-enforced results, locking down tools and permissions for unattended runs, and limiting turns and spend.

## Steps

### 1. Create the lab folder and sample code

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
mkdir lab06
cd lab06
mkdir sample_app
```

```powershell
notepad sample_app\db.py
```

```python
"""Tiny user-lookup module with deliberate security bugs for Lab 6."""
import sqlite3

# BUG 1: hard-coded credential
ADMIN_API_KEY = "sk-live-9f8e7d6c5b4a39281716"


def get_user(conn: sqlite3.Connection, username: str):
    # BUG 2: SQL built with string formatting -> SQL injection
    query = f"SELECT id, username, email FROM users WHERE username = '{username}'"
    return conn.execute(query).fetchall()


def get_user_safe(conn: sqlite3.Connection, username: str):
    return conn.execute(
        "SELECT id, username, email FROM users WHERE username = ?", (username,)
    ).fetchall()
```

```powershell
notepad sample_app\utils.py
```

```python
"""Clean helper module: a good review should report nothing here."""


def clamp(value: int, low: int, high: int) -> int:
    return max(low, min(high, value))
```

*Why:* `db.py` has two deliberate bugs: a hard-coded API key and an SQL query built with an f-string (SQL injection). `utils.py` is clean, so a good review should say nothing about it.

### 2. Run Claude Code headless

```powershell
claude -p "In one sentence, what does sample_app\db.py do?"
```

*Why:* `-p` (print mode) runs one prompt, prints the answer and exits, with no interactive session. This is the form CI uses. Nobody is there to answer a question, so anything that would need a permission prompt will fail or be denied.

### 3. Ask for JSON output

```powershell
claude -p "Say hello" --output-format json
```

*Why:* the output is a JSON envelope. Look for these fields: `result` (the text), `is_error`, `num_turns`, and `total_cost_usd`. Scripts read these instead of scraping text.

### 4. Define the result schema

```powershell
notepad review_schema.json
```

```json
{
  "type": "object",
  "properties": {
    "summary": {"type": "string", "description": "One or two sentences on the overall security posture of the reviewed code."},
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": {"type": "string"},
          "line": {"type": ["integer", "null"]},
          "severity": {"enum": ["low", "medium", "high", "critical"]},
          "category": {"enum": ["injection", "secrets", "auth", "crypto", "input-validation", "other"]},
          "description": {"type": "string"},
          "fix": {"type": "string"}
        },
        "required": ["file", "line", "severity", "category", "description", "fix"],
        "additionalProperties": false
      }
    }
  },
  "required": ["summary", "findings"],
  "additionalProperties": false
}
```

*Why:* this is the contract for the review. Severity and category are enums, `line` may be null, and `additionalProperties: false` rejects stray fields. With `--json-schema`, Claude Code returns a validated object in the `structured_output` field of the envelope. Your pipeline can trust the shape.

### 5. Write the CI script

```powershell
notepad ci_review.py
```

```python
"""Run Claude Code headless (claude -p) as a CI security-review gate.

Exit codes: 0 = pass, 1 = blocking findings, 2 = tooling/model error.
"""
import argparse
import json
import shutil
import subprocess
import sys
from pathlib import Path

HERE = Path(__file__).parent
SCHEMA = (HERE / "review_schema.json").read_text(encoding="utf-8")
BLOCKING = {"high", "critical"}

PROMPT = (
    "You are a security reviewer running in CI. Review {target} for security "
    "problems only: injection, hard-coded secrets, broken auth, weak crypto, "
    "missing input validation. Read the files you need. Report only real issues "
    "you can point to by file and line. If the code is clean, return an empty "
    "findings list."
)


def build_command(claude: str, prompt: str) -> list[str]:
    return [
        claude, "-p", prompt,
        "--output-format", "json",
        "--json-schema", SCHEMA,
        "--tools", "Read,Grep,Glob",      # read-only toolset
        "--permission-mode", "dontAsk",   # never wait for a human
        "--max-turns", "10",
        "--max-budget-usd", "0.50",
        "--no-session-persistence",
        "--bare",                         # skip local config; CI is reproducible
    ]


def run_review(target: str, diff_text: str | None) -> dict:
    claude = shutil.which("claude")
    if not claude:
        sys.exit("claude CLI not found on PATH. Run lab00 setup first.")
    what = "the code changes provided on stdin" if diff_text else f"the code under {target}"
    cmd = build_command(claude, PROMPT.format(target=what))
    # List form + shell=False: no PowerShell quoting problems with the JSON schema.
    proc = subprocess.run(cmd, input=diff_text, capture_output=True, text=True,
                          encoding="utf-8", timeout=600)
    if proc.returncode != 0:
        print(proc.stderr.strip() or proc.stdout.strip(), file=sys.stderr)
        sys.exit(2)
    try:
        envelope = json.loads(proc.stdout)
    except json.JSONDecodeError:
        print("Claude did not return JSON:\n" + proc.stdout[:500], file=sys.stderr)
        sys.exit(2)
    if envelope.get("is_error") or "structured_output" not in envelope:
        print("Review failed:", envelope.get("subtype") or envelope.get("result"), file=sys.stderr)
        sys.exit(2)
    return envelope


def render_markdown(review: dict) -> str:
    lines = ["# Claude security review", "", review["summary"], ""]
    if not review["findings"]:
        lines.append("No findings.")
    for f in review["findings"]:
        loc = f"{f['file']}:{f['line']}" if f["line"] else f["file"]
        lines += [f"- **{f['severity'].upper()}** `{loc}` ({f['category']}): "
                  f"{f['description']} Fix: {f['fix']}"]
    return "\n".join(lines) + "\n"


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("target", nargs="?", default="sample_app")
    ap.add_argument("--diff-file", help="review this diff file instead of a path")
    ap.add_argument("--git-diff", action="store_true", help="review `git diff HEAD` instead of a path")
    args = ap.parse_args()

    diff = None
    if args.diff_file:
        diff = Path(args.diff_file).read_text(encoding="utf-8-sig")
    elif args.git_diff:
        diff = subprocess.run(["git", "diff", "HEAD"], capture_output=True, text=True,
                              encoding="utf-8").stdout
        if not diff.strip():
            print("No changes to review.")
            return 0
    envelope = run_review(args.target, diff)
    review = envelope["structured_output"]

    Path("review.json").write_text(json.dumps(review, indent=2), encoding="utf-8")
    Path("review.md").write_text(render_markdown(review), encoding="utf-8")
    print(render_markdown(review))
    print(f"cost: ${envelope.get('total_cost_usd', 0):.4f}  turns: {envelope.get('num_turns')}")

    blocking = [f for f in review["findings"] if f["severity"] in BLOCKING]
    if blocking:
        print(f"GATE: FAIL ({len(blocking)} blocking finding(s))")
        return 1
    print("GATE: PASS")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

*Why, piece by piece:*
- The command is built as a **Python list** and run with `subprocess.run`. There is no shell, so the JSON schema needs no PowerShell quoting.
- `--tools "Read,Grep,Glob"`: the reviewer can only read. It cannot edit files or run commands.
- `--permission-mode dontAsk`: it never waits for a human; anything not allowed is denied.
- `--max-turns 10` and `--max-budget-usd 0.50`: a runaway review stops after ten turns or about fifty cents.
- `--no-session-persistence` and `--bare`: nothing is saved to disk, and local configuration is skipped so CI behaves the same everywhere. In CI the run authenticates with `ANTHROPIC_API_KEY`.
- **Exit codes** are the gate: `0` pass, `1` blocking findings (high or critical), `2` tooling or model error. A broken review must not look like a clean review.

### 6. Run the review on the sample code

```powershell
python ci_review.py
"exit code: $LASTEXITCODE"
```

Expected output (wording will differ):

```
# Claude security review

The module has a hard-coded credential and an injectable SQL query.

- **CRITICAL** `sample_app/db.py:5` (secrets): API key committed in source. Fix: read it from an environment variable.
- **HIGH** `sample_app/db.py:11` (injection): query built with an f-string. Fix: use a parameterized query.

cost: $0.0xxx  turns: 3
GATE: FAIL (2 blocking finding(s))
exit code: 1
```

*Why:* both planted bugs should be found with file and line, nothing is reported for `utils.py`, and the exit code is 1. Open `review.json` and `review.md` to see the saved output.

### 7. Fix the bugs and pass the gate

Edit `sample_app\db.py`. Delete the `ADMIN_API_KEY` line, and replace the body of `get_user` with the parameterized query that `get_user_safe` already uses:

```python
def get_user(conn: sqlite3.Connection, username: str):
    return conn.execute(
        "SELECT id, username, email FROM users WHERE username = ?", (username,)
    ).fetchall()
```

Run again:

```powershell
python ci_review.py
"exit code: $LASTEXITCODE"
```

Expected: `No findings.` (or only low or medium notes), `GATE: PASS`, `exit code: 0`.

*Why:* the gate flips from fail to pass because the code changed, which is what a pull request check should do.

### 8. Review only what changed (diff mode)

CI reviews a pull request's diff, not the whole repo. Simulate it:

```powershell
git init
git add .
git commit -m "base"
Add-Content sample_app\utils.py "`ndef calc(expr):`n    return eval(expr)`n"
python ci_review.py --git-diff
"exit code: $LASTEXITCODE"
```

If Git asks who you are, run `git config --global user.name "Your Name"` and `git config --global user.email "you@example.com"` first.

*Why:* `--git-diff` sends `git diff HEAD` to Claude on standard input. Claude should flag `eval` on user-supplied text (injection or unsafe execution) in the changed lines. Smaller input means a faster and cheaper review.

### 9. Break the budget on purpose

In `ci_review.py`, change `"--max-turns", "10"` to `"--max-turns", "1"` and run `python ci_review.py sample_app`.

*Why:* the review cannot finish in one turn, so Claude Code reports an error result such as `error_max_turns`. The script prints `Review failed: ...` and exits with **2**, not 0. Restore `"10"`.

### 10. Read the workflow that runs this in CI

Save the file as `.github\workflows\claude-review.yml` in your repository (you do not need to push it for this lab):

```powershell
mkdir .github\workflows
notepad .github\workflows\claude-review.yml
```

```yaml
name: claude-security-review
on:
  pull_request:

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Build PR diff
        run: git diff origin/${{ github.base_ref }}...HEAD > pr.diff
      - name: Run Claude review gate
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: python ci_review.py --diff-file pr.diff
      - name: Upload review
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: claude-review
          path: |
            review.json
            review.md
```

*Why, step by step:*
- `on: pull_request` triggers on every PR.
- `fetch-depth: 0` fetches full history so the diff against the base branch works.
- Node 22 and `npm install -g @anthropic-ai/claude-code` install the CLI on the runner.
- `git diff origin/<base>...HEAD > pr.diff` builds the diff of the PR only.
- `ANTHROPIC_API_KEY` comes from a **repository secret** (Settings > Secrets and variables > Actions). It is never in the file.
- `permissions: contents: read` gives the job the least access it needs.
- The final step uploads `review.json` and `review.md` even when the gate fails (`if: always()`).

If you want to run it for real, push the repo to GitHub, add the `ANTHROPIC_API_KEY` secret, and open a pull request that introduces a vulnerability.

## Verify

- Step 6 exits with code 1 and lists the SQL injection and the hard-coded key.
- Step 7 exits with code 0.
- Step 8 reports the `eval` change.
- Step 9 exits with code 2.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `claude CLI not found on PATH` | Reopen PowerShell, or reinstall Claude Code (Lab 0, step 3). |
| `Review failed: ...` with no obvious cause | Run the printed `claude -p` command parts by hand; check `claude auth status` and that `ANTHROPIC_API_KEY` is set in this window. |
| `unknown option` for a flag | Your Claude Code is older. Update it with `claude update`. |
| Gate passes but you expected a failure | Models vary. Make the prompt stricter or the sample bug more obvious, then rerun. |
| Spend limit reached | Raise `--max-budget-usd` slightly, or reduce the amount of code reviewed. |

## Stretch goals

1. Post the markdown from `review.md` as a pull request comment using the GitHub CLI (`gh pr comment`).
2. Add a `confidence` field to the schema and make the gate ignore low-confidence "medium" findings.
3. Store the schema-valid `review.json` as a build artifact and chart finding counts over time.

## Exam takeaways

- In CI always use **headless mode** (`claude -p`), never an interactive session.
- Use `--output-format json` and `--json-schema` so downstream steps parse **validated** data.
- Give an unattended run the **minimum tools** and a non-interactive permission mode.
- Cap **turns and budget**; treat an errored review as a failure (exit 2), not as a pass.
- Keep the key in a **CI secret**, and review the **diff**, not the whole repository.

## Cleanup

```powershell
cd C:\cca-labs
Remove-Item -Recurse -Force lab06\.git
```
