# Lab 1: Design an orchestrated multi-step agent

| | |
|---|---|
| **Module** | 1 – Agentic Architecture & Orchestration (Domain 1) |
| **Duration** | 35–40 minutes |
| **Level** | Intermediate |
| **Prerequisite** | Lab 0 completed |
| **Cost** | About 5–10 cents |

## What you will build

A **coordinator** agent that breaks a request into subtasks and hands each one to a **specialist subagent**. Each subagent has its own small toolset and a fresh context. You will write one reusable agent loop and then use it for all three agents.

```
                    +----------------+
   user request --> |  Coordinator   | --> final brief
                    +--------+-------+
              delegate_researcher | delegate_analyst
                 +---------------+---------------+
                 v                               v
        +----------------+              +----------------+
        |  Researcher    |              |   Analyst      |
        | get_product_info              | get_sales      |
        | list_competitors              | compute_growth |
        +----------------+              +----------------+
        (fresh context, 5 turns)        (fresh context, 5 turns)
```

You will learn: the agent loop, `stop_reason`, tool results, why subagents get isolated context, and why every loop needs a turn cap.

## Steps

### 1. Create the lab folder

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
mkdir lab01
cd lab01
```

*Why:* each lab lives in its own folder so files do not mix.

### 2. Write the reusable agent loop

```powershell
notepad common.py
```

```python
"""Shared agent loop for Lab 1. One function runs any agent."""
import json
import os

import anthropic

MODEL = os.environ.get("CLAUDE_MODEL", "claude-sonnet-5-5")
client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment


def run_agent(name, system, tools, tool_impls, task, max_turns=8, verbose=True):
    """Run one agent loop and return its final text answer."""
    messages = [{"role": "user", "content": task}]
    for turn in range(1, max_turns + 1):
        resp = client.messages.create(
            model=MODEL, max_tokens=1500, system=system, tools=tools, messages=messages
        )
        messages.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason != "tool_use":  # end_turn: the agent is done
            return "".join(b.text for b in resp.content if b.type == "text")

        results = []
        for block in resp.content:
            if block.type != "tool_use":
                continue
            if verbose:
                print(f"  [{name}] turn {turn}: {block.name}({json.dumps(block.input)})")
            try:
                output, is_error = tool_impls[block.name](**block.input), False
            except Exception as exc:  # return errors to the model instead of crashing
                output, is_error = f"{type(exc).__name__}: {exc}", True
            if not isinstance(output, str):
                output = json.dumps(output)
            results.append({"type": "tool_result", "tool_use_id": block.id,
                            "content": output, "is_error": is_error})
        messages.append({"role": "user", "content": results})
    return f"[{name}] stopped: reached max_turns={max_turns}"
```

*Why, piece by piece:*
- The loop calls the model, then checks `stop_reason`. `"tool_use"` means "run my tools and call me again". Anything else (`"end_turn"`) means the agent is finished.
- The assistant reply is appended **as is**, so the `tool_use` blocks stay in the history.
- Each tool result goes back as a `tool_result` that carries the matching `tool_use_id`.
- If a tool crashes, the error is returned with `is_error: True` so the model can react instead of the program dying.
- `max_turns` is the safety stop. Without it, a confused agent could loop forever.

### 3. Add the mock data, tools and tool schemas

```powershell
notepad data_tools.py
```

```python
"""Mock business data, tool functions and tool schemas for Lab 1."""

PRODUCTS = {
    "P-100": {"name": "AeroBottle", "category": "drinkware", "price_usd": 24.0, "launched": "2025-03"},
    "P-200": {"name": "TrailPack", "category": "bags", "price_usd": 89.0, "launched": "2024-09"},
}
SALES = {("P-100", "Q2"): 1200, ("P-100", "Q3"): 1800, ("P-200", "Q2"): 900, ("P-200", "Q3"): 840}
COMPETITORS = {"drinkware": ["HydroPeak", "SipCo"], "bags": ["RidgeGear", "PackWell"]}


def get_product_info(product_id):
    return PRODUCTS[product_id]  # KeyError becomes a tool error


def list_competitors(category):
    return COMPETITORS[category]


def get_sales(product_id, quarter):
    return {"product_id": product_id, "quarter": quarter, "units": SALES[(product_id, quarter)]}


def compute_growth(old, new):
    return {"growth_percent": round((new - old) / old * 100, 1)}


def _tool(name, description, props, required):
    return {"name": name, "description": description,
            "input_schema": {"type": "object", "properties": props, "required": required}}


RESEARCHER_TOOLS = [
    _tool("get_product_info", "Get name, category, price and launch month for ONE product ID such as P-100.",
          {"product_id": {"type": "string", "description": "Product ID, e.g. P-100"}}, ["product_id"]),
    _tool("list_competitors", "List competitor brands for ONE product category such as drinkware or bags.",
          {"category": {"type": "string", "description": "Category name, lowercase"}}, ["category"]),
]

ANALYST_TOOLS = [
    _tool("get_sales", "Get units sold for ONE product in ONE quarter (Q2 or Q3).",
          {"product_id": {"type": "string"}, "quarter": {"type": "string", "enum": ["Q2", "Q3"]}},
          ["product_id", "quarter"]),
    _tool("compute_growth", "Compute percent growth from an old number to a new number.",
          {"old": {"type": "number"}, "new": {"type": "number"}}, ["old", "new"]),
]
```

*Why:* the data is fake so the lab runs the same for everyone. Notice that each agent gets only the tools it needs (`RESEARCHER_TOOLS`, `ANALYST_TOOLS`). Descriptions say what the tool does **and** its limits, for example "ONE product ID".

### 4. Write the coordinator and the two subagents

```powershell
notepad orchestrator.py
```

```python
"""Lab 1: a coordinator that delegates to two subagents."""
import sys

import data_tools as d
from common import run_agent

RESEARCHER_IMPLS = {"get_product_info": d.get_product_info, "list_competitors": d.list_competitors}
ANALYST_IMPLS = {"get_sales": d.get_sales, "compute_growth": d.compute_growth}

RESEARCHER_SYSTEM = (
    "You are a product researcher. Use your tools to gather facts. "
    "Reply with a short bullet list of facts only. Do not guess."
)
ANALYST_SYSTEM = (
    "You are a sales analyst. Use your tools to get sales numbers and compute growth. "
    "Reply with the numbers and one sentence of interpretation. Do not guess."
)


def researcher(task, context):
    """Subagent 1. It starts with a fresh context: it only knows what we pass in."""
    prompt = f"Context from the coordinator:\n{context}\n\nYour task:\n{task}"
    return run_agent("researcher", RESEARCHER_SYSTEM, d.RESEARCHER_TOOLS, RESEARCHER_IMPLS, prompt, max_turns=5)


def analyst(task, context):
    """Subagent 2. Same pattern, different tools."""
    prompt = f"Context from the coordinator:\n{context}\n\nYour task:\n{task}"
    return run_agent("analyst", ANALYST_SYSTEM, d.ANALYST_TOOLS, ANALYST_IMPLS, prompt, max_turns=5)


DELEGATE_PROPS = {
    "task": {"type": "string", "description": "One specific subtask for the specialist."},
    "context": {"type": "string", "description": "Everything the specialist needs, such as product IDs. It cannot see this conversation."},
}
COORDINATOR_TOOLS = [
    {"name": "delegate_researcher",
     "description": "Ask the researcher for product facts and competitor names. It cannot read sales data.",
     "input_schema": {"type": "object", "properties": DELEGATE_PROPS, "required": ["task", "context"]}},
    {"name": "delegate_analyst",
     "description": "Ask the analyst for sales numbers and growth calculations. It cannot read product facts.",
     "input_schema": {"type": "object", "properties": DELEGATE_PROPS, "required": ["task", "context"]}},
]
COORDINATOR_IMPLS = {"delegate_researcher": researcher, "delegate_analyst": analyst}

COORDINATOR_SYSTEM = (
    "You are a coordinator. Break the request into subtasks, delegate each one to the right specialist, "
    "pass them all the context they need, then merge their answers into a short brief with three headings: "
    "Product, Sales, Competitors. Never invent facts that no specialist returned."
)


def run_coordinator(request):
    return run_agent("coordinator", COORDINATOR_SYSTEM, COORDINATOR_TOOLS, COORDINATOR_IMPLS, request, max_turns=8)


if __name__ == "__main__":
    request = sys.argv[1] if len(sys.argv) > 1 else "Write a brief on product P-100: what it is, how Q3 sales compare with Q2, and who competes with it."
    print(f"Request: {request}\n")
    print("\n=== FINAL BRIEF ===\n" + run_coordinator(request))
```

*Why, piece by piece:*
- `researcher()` and `analyst()` each call `run_agent` with a brand new `messages` list. They cannot see the coordinator's conversation. Everything they need must be passed in the `context` argument.
- From the coordinator's point of view, delegating is just another tool call (`delegate_researcher`, `delegate_analyst`). The tool implementation runs a whole sub-agent.
- The coordinator system prompt says to merge results and never invent facts. Subagent output is the only source of facts.

### 5. Run the default request

```powershell
python orchestrator.py
```

Expected output (wording and order of calls may vary):

```
Request: Write a brief on product P-100: what it is, how Q3 sales compare with Q2, and who competes with it.

  [coordinator] turn 1: delegate_researcher({"task": "...", "context": "Product ID P-100"})
  [researcher] turn 1: get_product_info({"product_id": "P-100"})
  [researcher] turn 2: list_competitors({"category": "drinkware"})
  [coordinator] turn 2: delegate_analyst({"task": "...", "context": "Product ID P-100, Q2 vs Q3"})
  [analyst] turn 1: get_sales({"product_id": "P-100", "quarter": "Q2"})
  [analyst] turn 2: get_sales({"product_id": "P-100", "quarter": "Q3"})
  [analyst] turn 3: compute_growth({"old": 1200, "new": 1800})

=== FINAL BRIEF ===
Product: AeroBottle, drinkware, $24, launched 2025-03 ...
Sales: Q2 1,200 units, Q3 1,800 units, growth 50.0% ...
Competitors: HydroPeak, SipCo
```

*Why:* the indented lines show each agent's tool calls, so you can see the plan the coordinator chose and which specialist did what. The growth figure must be **50.0%**. That number comes from a tool, not from the model's arithmetic.

### 6. Try a different request

```powershell
python orchestrator.py "Which grew more from Q2 to Q3, P-100 or P-200? Include competitor names for the faster grower."
```

*Why:* the coordinator must call the analyst for both products, compare, and only then ask the researcher about the winner's category. P-200 sales fell from 900 to 840, so P-100 wins. Check that the coordinator planned this without you telling it the steps.

### 7. Break context isolation on purpose

Open `orchestrator.py` and change the first line inside `researcher()`:

```python
    prompt = f"Your task:\n{task}"      # context removed
```

Run the default request again.

*Why:* with the `context` dropped, the researcher only sees what the coordinator wrote in `task`. If the coordinator put the product ID only in `context`, the researcher will ask for it or call a tool with a bad ID, and you will see an error come back. Subagents know nothing you do not pass them. Restore the original line when you are done.

### 8. See the turn cap work

In `orchestrator.py`, change `max_turns=8` to `max_turns=1` inside `run_coordinator`, then run the default request.

*Why:* after one turn the coordinator has only issued delegation calls and has not merged anything, so you get `[coordinator] stopped: reached max_turns=1`. A cap turns an infinite loop into a clean, visible failure. Restore `max_turns=8`.

## Verify

- Step 5 shows at least one `delegate_researcher` call and one `delegate_analyst` call.
- The final brief has the three headings and the growth value `50.0%`.
- Step 8 prints a "stopped" message and does not hang.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError: anthropic` | Activate the virtual environment: `.\.venv\Scripts\Activate.ps1` from `C:\cca-labs`. |
| `KeyError: 'Q4'` shown as a tool error | Expected. The data only has Q2 and Q3. The model sees the error and should explain it. |
| Coordinator answers without delegating | Make the request ask for facts that need both specialists; check the tool descriptions were pasted correctly. |
| `anthropic.NotFoundError` | The model name is not available. Set `$env:CLAUDE_MODEL`. |

## Stretch goals

1. Add a third specialist, a `writer`, that formats the final brief. Decide which tools it needs (probably none).
2. Run the researcher and analyst in parallel with `concurrent.futures.ThreadPoolExecutor` when the coordinator returns two `tool_use` blocks in one turn.
3. Return a structured summary (JSON) from each subagent instead of free text.

## Exam takeaways

- The loop ends on `stop_reason`, not on text matching.
- Subagents get **isolated context**. Pass what they need explicitly.
- Give each agent a **small, focused toolset**.
- Always cap the number of turns.
- Use tools for facts and arithmetic; do not trust the model to compute or remember them.

## Cleanup

Nothing to remove. Files stay in `C:\cca-labs\lab01`.
