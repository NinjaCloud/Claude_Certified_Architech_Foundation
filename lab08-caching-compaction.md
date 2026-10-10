# Lab 8: Apply prompt caching and compaction to a long-running conversation

| | |
|---|---|
| **Module** | 5 – Context Management & Reliability (Domain 5) |
| **Duration** | 35–40 minutes |
| **Level** | Intermediate |
| **Prerequisite** | Lab 0 completed |
| **Cost** | About 15–30 cents for all runs |

## What you will build

Two experiments on a support agent that carries a large policy handbook (about 5,500 tokens) in its system prompt:

1. **Prompt caching:** measure what happens when the handbook is a stable cached prefix, and what happens when a timestamp ahead of it changes the prefix on every call.
2. **Compaction:** run a 12-turn conversation, cache the stable prefix and the conversation so far, and summarize old turns when the history passes a limit.

```
 Request layout (order matters for caching):

 [ system: policy handbook  <cache breakpoint> ]  stable, big   --> cached
 [ messages: old turns ...                      ]  grows        --> cached up to the last breakpoint
 [ messages: newest user turn <breakpoint>      ]  new each turn --> written, then read next turn

 When history > limit:   old turns --> summary (one extra call) --> history shrinks
```

You will learn: where to place stable and dynamic content, how to read the usage fields, what cache hits and misses cost, and the trade-off of compaction.

## Steps

### 1. Create the lab folder

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
mkdir lab08
cd lab08
```

### 2. Generate the long, stable document

```powershell
notepad policy_doc.py
```

```python
"""Builds a long, stable support-policy document (~5-6k tokens) to cache."""

TOPICS = [
    ("Refunds", "Refund requests are accepted within {d} days of delivery when the item is unused. Refunds go to the original payment method and normally post within {p} business days."),
    ("Shipping", "Standard shipping takes {d} to {p} business days. Orders over ${m} ship free. Expedited options are priced at checkout and cannot be changed after dispatch."),
    ("Returns", "Returns need an RMA number issued by support. Items must arrive within {d} days of the RMA date, in original packaging, or a {p} percent restocking fee applies."),
    ("Warranty", "Hardware carries a {d}-month limited warranty covering manufacturing defects. Accidental damage and liquid damage are excluded. Claims require proof of purchase."),
    ("Privacy", "Customer data is retained for {d} months after the last order. Deletion requests are completed within {p} days. Support agents never request full card numbers."),
    ("Escalation", "Cases open longer than {d} hours, or involving amounts above ${m}, are escalated to a senior agent. Legal threats go straight to the compliance team."),
    ("Loyalty", "Members earn {p} points per dollar. Points expire after {d} months of inactivity. Points cannot be exchanged for cash or transferred between accounts."),
    ("Billing", "Duplicate charges are reversed within {p} business days after verification. Disputed charges over ${m} require a written statement from the account holder."),
]


def build_policy(sections: int = 48, salt: str = "") -> str:
    parts = [f"ACME SUPPORT POLICY HANDBOOK {salt}".strip(),
             "Agents must answer only from this handbook and cite the section number."]
    for n in range(1, sections + 1):
        topic, tmpl = TOPICS[(n - 1) % len(TOPICS)]
        body = tmpl.format(d=7 + n % 24, p=2 + n % 9, m=50 + 25 * (n % 12))
        extra = (f" Applies to region group {n % 6 + 1}. Exceptions need written approval from a "
                 f"team lead and must be logged in the case record with reference POL-{1000 + n}. "
                 f"Agents should confirm the customer's identity before sharing any account detail, "
                 f"and should never promise outcomes that depend on a later review.")
        parts.append(f"Section {n}: {topic}. {body}{extra}")
    return "\n\n".join(parts)


if __name__ == "__main__":
    text = build_policy()
    print(f"{len(text)} characters, about {len(text) // 4} tokens")
```

```powershell
python policy_doc.py
```

Expected: something like `22380 characters, about 5595 tokens`.

*Why:* caching only works above a minimum prompt size that depends on the model, so you need a realistic large block. The text is generated and identical every time, which is exactly what a cache needs.

### 3. Add shared helpers

```powershell
notepad common8.py
```

```python
"""Shared helpers: client, model, usage accounting."""
import os

import anthropic

MODEL = os.getenv("CLAUDE_MODEL", "claude-sonnet-5-5")
client = anthropic.Anthropic()

# Billing multipliers relative to the base input price.
# Cache write: 1.25x (5-minute TTL). Cache read: 0.1x on most models, 0.05x on
# some newer ones. Check the pricing page and edit READ_MULT for your model.
WRITE_MULT = 1.25
READ_MULT = 0.05


def usage_tuple(u):
    """(uncached_input, cache_write, cache_read) token counts from a response.usage."""
    return (u.input_tokens,
            getattr(u, "cache_creation_input_tokens", 0) or 0,
            getattr(u, "cache_read_input_tokens", 0) or 0)


def billed_equivalent(uncached, write, read):
    """Input cost expressed as 'plain input tokens'."""
    return uncached + WRITE_MULT * write + READ_MULT * read
```

*Why:* `usage_tuple` extracts the three numbers that matter from every response: normal input tokens, **cache write** tokens (`cache_creation_input_tokens`) and **cache read** tokens (`cache_read_input_tokens`). `billed_equivalent` converts them into "plain input tokens" using the price multipliers: a write costs more than normal input, a read costs far less. Check the current multipliers on the Anthropic pricing page and edit `WRITE_MULT` and `READ_MULT` for your model.

### 4. Measure the real token count

```powershell
notepad count_policy.py
```

```python
from common8 import MODEL, client
from policy_doc import build_policy

n = client.messages.count_tokens(
    model=MODEL, system=build_policy(),
    messages=[{"role": "user", "content": "hi"}],
).input_tokens
print("prompt tokens:", n)
```

```powershell
python count_policy.py
```

*Why:* `count_tokens` is a free call that tells you the exact size before you pay for a request. Use it to check that your prefix is above the cache minimum.

### 5. Write the caching experiment

```powershell
notepad cache_demo.py
```

```python
"""Compare a stable cached prefix against a prefix that changes on every call."""
import random
import time
from datetime import datetime

from common8 import MODEL, billed_equivalent, client, usage_tuple
from policy_doc import build_policy

QUESTIONS = [
    "How many days do I have to request a refund?",
    "Who handles legal threats?",
    "Do loyalty points expire?",
]
SALT = f"run-{random.randint(1000, 9999)}"  # makes call 1 a fresh cache write each run


def ask(system_blocks, question, dynamic_suffix=""):
    resp = client.messages.create(
        model=MODEL, max_tokens=120, system=system_blocks,
        messages=[{"role": "user", "content": question + dynamic_suffix + " Answer in one sentence."}],
    )
    return usage_tuple(resp.usage)


def scenario(name, make_system, make_suffix=lambda: ""):
    print(f"\n== {name}")
    print(f"{'call':4} {'uncached':>9} {'cache_write':>11} {'cache_read':>10} {'cost(eq. tokens)':>17}")
    total = 0.0
    for i, q in enumerate(QUESTIONS, 1):
        u, w, r = ask(make_system(), q, make_suffix())
        cost = billed_equivalent(u, w, r)
        total += cost
        print(f"{i:<4} {u:>9} {w:>11} {r:>10} {cost:>17.0f}")
        if i == 1 and w + r == 0:
            print("   !! nothing cached: prompt is below the minimum cacheable size for this model")
        time.sleep(1)
    print(f"{'total':>37} {total:>17.0f}")
    return total


def stable_system():
    return [{"type": "text", "text": build_policy(salt=SALT),
             "cache_control": {"type": "ephemeral"}}]          # breakpoint after the policy


def stable_system_v2():
    return [{"type": "text", "text": build_policy(salt=SALT + "-c"),
             "cache_control": {"type": "ephemeral"}}]


def volatile_system():
    # Anti-pattern: a timestamp BEFORE the big block changes the prefix every call.
    stamp = f"Current time: {datetime.now().isoformat()}\n"
    return [{"type": "text", "text": stamp + build_policy(salt=SALT + "-v"),
             "cache_control": {"type": "ephemeral"}}]


def timestamp_suffix():
    return f" (time now: {datetime.now().isoformat()})"


if __name__ == "__main__":
    a = scenario("A: stable prefix (policy first, question last)", stable_system)
    b = scenario("B: volatile prefix (timestamp at the top)", volatile_system)
    c = scenario("C: same timestamp, moved after the cached block", stable_system_v2, timestamp_suffix)
    print(f"\nA costs {a / b:.0%} of B.  C costs {c / b:.0%} of B (timestamp moved to the end).")
```

*Why, piece by piece:*
- `cache_control: {"type": "ephemeral"}` on the system block is a **cache breakpoint**: everything up to and including that block can be cached for about 5 minutes (a longer TTL is optional).
- Scenario A puts the handbook first and the changing question last, in the user message. The prefix is identical on each call.
- Scenario B puts `Current time: ...` **before** the handbook. The prefix is different every call, so nothing ever matches.
- Scenario C keeps the same handbook as A but puts the timestamp **after** the cached block, inside the user message.
- `SALT` makes each script run start with a fresh cache write so your results are repeatable.

### 6. Run it

```powershell
python cache_demo.py
```

Expected output (numbers will differ slightly):

```
== A: stable prefix (policy first, question last)
call  uncached cache_write cache_read  cost(eq. tokens)
1          17        5597          0              7013
2          12           0       5597               292
3          12           0       5597               292
                                total              7597

== B: volatile prefix (timestamp at the top)
call  uncached cache_write cache_read  cost(eq. tokens)
1          17        5608          0              7027
2          12        5608          0              7022
3          12        5608          0              7022
                                total             21071

== C: same timestamp, moved after the cached block
call  uncached cache_write cache_read  cost(eq. tokens)
1          26        5597          0              7022
2          22           0       5597               302
3          22           0       5597               302
                                total              7626

A costs 36% of B.  C costs 36% of B (timestamp moved to the end).
```

*Why, read it:*
- Call 1 of A **writes** the cache (about 25 percent surcharge).
- Calls 2 and 3 of A **read** it: almost all tokens are read at a fraction of the normal price, and only the new question is "uncached".
- In B every call writes again and never reads. You pay the write surcharge each time, and get no benefit.
- In C the time is still sent, but after the cached block, so the cache works again.
- If A shows zeros in both `cache_write` and `cache_read`, the prompt is below the model's minimum cacheable size.

### 7. Draw the rule from the results

Compare the B and C tables. Both send the current time on every call. Only the position differs.

*Why:* the rule is **static content first, dynamic content last.** Anything that changes (time, user name, retrieved snippets) belongs after the cached block, in the user message. A single changed character before a breakpoint invalidates the cache for everything after it.

### 8. Write the long-conversation script

```powershell
notepad compaction.py
```

```python
"""Long conversation: cache the stable prefix, compact old turns when history grows."""
import argparse

from common8 import MODEL, billed_equivalent, client, usage_tuple
from policy_doc import build_policy

HISTORY_LIMIT = 1200   # compact when the message history exceeds this many tokens
KEEP_LAST = 4          # keep the last 4 messages (2 exchanges) verbatim

SYSTEM = [{"type": "text", "text": build_policy(salt="compaction"),
           "cache_control": {"type": "ephemeral"}}]

NAMES = ["Asha", "Ben", "Chen", "Dana", "Eli", "Farah", "Gus", "Hana", "Ivan", "Jo", "Kira", "Leo"]
ISSUES = ["a late parcel", "a duplicate charge", "a cracked screen", "points that vanished",
          "a data deletion request", "a refund that has not arrived"]


def user_turns(n=12):
    for i in range(n):
        detail = (f"Customer {NAMES[i % len(NAMES)]} (case C-{2000 + i}) wrote in about "
                  f"{ISSUES[i % len(ISSUES)]}. They explain at length that they ordered "
                  f"{3 + i} items, contacted us {i % 3 + 1} times already, and want to know what "
                  f"the handbook says. Please name the single most relevant section number. ") * 3
        yield detail + "Answer in one sentence."


def with_breakpoint(messages):
    """Copy messages and mark the last content block as a cache breakpoint."""
    msgs = [dict(m) for m in messages]
    last = msgs[-1]
    content = last["content"]
    if isinstance(content, str):
        content = [{"type": "text", "text": content}]
    else:
        content = [dict(b) for b in content]
    content[-1]["cache_control"] = {"type": "ephemeral"}
    last["content"] = content
    return msgs


def history_tokens(messages):
    return client.messages.count_tokens(model=MODEL, messages=messages).input_tokens


def compact(messages):
    old, recent = messages[:-KEEP_LAST], messages[-KEEP_LAST:]
    transcript = "\n".join(
        f"{m['role']}: {m['content'] if isinstance(m['content'], str) else m['content'][0]['text']}"
        for m in old)
    resp = client.messages.create(
        model=MODEL, max_tokens=300,
        system="Summarize this support conversation. Keep case IDs, customer names, "
               "decisions and cited section numbers. Be brief.",
        messages=[{"role": "user", "content": transcript}],
    )
    summary = resp.content[0].text
    return ([{"role": "user", "content": f"Summary of the earlier conversation:\n{summary}"},
             {"role": "assistant", "content": "Understood. I will use this summary as context."}]
            + recent)


def run(turns=12, use_compaction=True):
    messages, rows = [], []
    for t, text in enumerate(user_turns(turns), 1):
        compacted = False
        if use_compaction and messages and history_tokens(messages) > HISTORY_LIMIT:
            messages = compact(messages)
            compacted = True
        messages.append({"role": "user", "content": text})
        resp = client.messages.create(model=MODEL, max_tokens=100, system=SYSTEM,
                                      messages=with_breakpoint(messages))
        messages.append({"role": "assistant", "content": resp.content[0].text})
        u, w, r = usage_tuple(resp.usage)
        rows.append((t, u + w + r, u, w, r, compacted, billed_equivalent(u, w, r)))
    return rows


def report(title, rows):
    print(f"\n== {title}")
    print(f"{'turn':4} {'prompt':>7} {'uncached':>9} {'write':>6} {'read':>6} {'compacted':>9} {'cost':>7}")
    for t, prompt, u, w, r, c, cost in rows:
        print(f"{t:<4} {prompt:>7} {u:>9} {w:>6} {r:>6} {'yes' if c else '':>9} {cost:>7.0f}")
    total_prompt = sum(r[1] for r in rows)
    total_cost = sum(r[6] for r in rows)
    print(f"total prompt tokens {total_prompt}, billed-equivalent {total_cost:.0f} "
          f"({total_cost / total_prompt:.0%} of uncached)")
    return total_cost


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("turns", nargs="?", type=int, default=12)
    ap.add_argument("--no-compaction", action="store_true", help="keep the full history")
    args = ap.parse_args()
    report("no compaction" if args.no_compaction else "cache + compaction",
           run(args.turns, not args.no_compaction))
```

*Why, piece by piece:*
- `SYSTEM` is the cached handbook, as before.
- `with_breakpoint()` adds a second cache breakpoint on the **latest user message** each turn, so the whole conversation so far is read from cache next turn instead of re-sent at full price.
- `history_tokens()` uses `count_tokens` on the messages only. When it passes `HISTORY_LIMIT`, `compact()` runs.
- `compact()` asks Claude to summarize all but the last four messages, keeping case IDs, names and cited sections, then rebuilds the history as: summary, short acknowledgement, last two exchanges.
- Compaction changes the start of the history, so the next call must **write** a new cache for the conversation part. The handbook (system) stays a cache **read**.

### 9. Run the long conversation

```powershell
python compaction.py
```

Expected output (shortened, numbers will differ):

```
turn  prompt  uncached  write   read compacted    cost
1       5776        17   5777      0              7220
2       5962        12    367   5597               737
...
7       6906        14   1315   5597              1918
8       6184        12    591   5597       yes    1015
...
total prompt tokens 77186, billed-equivalent 22615 (29% of uncached)
```

*Why, read it:*
- `read` stays near 5,600 every turn: the handbook is never re-billed at full price.
- `prompt` grows each turn, then **drops** on the turn marked `compacted`: the history shrank.
- `write` is the new conversation tail added each turn. After compaction it is a fresh write of the shorter history.

### 10. Compare with no compaction

```powershell
python compaction.py --no-compaction
```

*Why:* compare the `prompt` column and the totals. Caching already makes the repeated history cheap, so over 12 turns compaction may not win on cost: it adds a summarization call and one cache re-write. Its job is to **keep the context bounded** so a very long session never hits the context limit and the model keeps its focus. Try `python compaction.py 30` and `python compaction.py 30 --no-compaction` to see the gap grow with length.

### 11. Check what the summary kept

Add `print(summary)` after the `summary = ...` line in `compact()` and run again.

*Why:* compaction is lossy. Check that case IDs, customer names and section numbers survived. In production, keep facts that must not be lost (IDs, decisions, open tasks) in a structured "case facts" block that you never summarize.

## Verify

- Scenarios A and C show one write, then reads. Scenario B shows writes only.
- The totals for A and C are clearly lower than the total for B.
- The long run shows a `compacted` turn where `prompt` drops, with `read` still near the handbook size.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `cache_write` and `cache_read` both 0 | The prefix is below the model's minimum cacheable size, or `cache_control` is missing. Check `count_policy.py` and the model docs. |
| Call 2 of A shows a write, not a read | More than about 5 minutes passed, or something before the breakpoint changed. Re-run quickly. |
| `AttributeError: ... count_tokens` | Upgrade the SDK: `pip install -U anthropic`. |
| No compaction happens | Lower `HISTORY_LIMIT` in `compaction.py` (for example to 800) or run more turns. |
| Numbers differ from the examples | Normal. Token counts depend on the model's tokenizer. |

## Stretch goals

1. Use a one-hour cache TTL (`"cache_control": {"type": "ephemeral", "ttl": "1h"}`) for the handbook and compare cost patterns for a pause longer than five minutes.
2. Store must-keep facts in a separate `case_facts` string that is placed in the system prompt (after the handbook) and updated, instead of relying on the summary.
3. Trim long tool outputs before they enter the history, and measure the effect on `prompt`.

## Exam takeaways

- Put **stable content first**, dynamic content last. Any change before a breakpoint breaks the cache for what follows it.
- Read `cache_creation_input_tokens` and `cache_read_input_tokens` to confirm caching works; a minimum prompt size applies.
- A cache write costs a bit more than normal input; a cache read costs a small fraction. Caching pays off when a prefix is **reused**.
- **Compact** long histories by summarizing older turns, but keep critical facts (IDs, decisions) outside the summary.
- Plan for the 5-minute default TTL and use `count_tokens` and token budgets to control cost.

## Cleanup

Nothing to remove. You can delete the `C:\cca-labs` folder when the course ends.
