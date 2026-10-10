# Lab 7: Build a schema-enforced extraction pipeline with retries

| | |
|---|---|
| **Module** | 4 – Prompt Engineering & Structured Output (Domain 4) |
| **Duration** | 35–40 minutes |
| **Level** | Intermediate |
| **Prerequisite** | Lab 0 completed |
| **Cost** | About 10–20 cents per full pipeline run |

## What you will build

A pipeline that turns messy invoice text into validated JSON. Claude is forced to answer through a tool with a schema. Your code then checks the schema and the business rules, sends specific errors back for a limited number of retries, and routes anything still wrong to a human review queue.

```
 document --> [Extract: forced tool + few-shot] --> [Validate: schema + rules]
                    ^                                    |
                    |  invalid, retryable                | valid --> accepted
                    +---- feed exact errors back --------+
                                                         | invalid, retries used up
                                                         | or info missing in source
                                                         v
                                                   review queue
```

You will learn: schema design with nullable fields, forcing structure with `tool_choice`, few-shot examples, two kinds of validation, the error-feedback retry pattern, and why retries cannot fix missing information.

## Steps

### 1. Create the lab folder

```powershell
cd C:\cca-labs
.\.venv\Scripts\Activate.ps1
mkdir lab07
cd lab07
```

### 2. Define the schema and the validators

```powershell
notepad schema.py
```

```python
"""Extraction schema (as a tool) plus schema and business-rule validation."""
import re
from datetime import date

from jsonschema import Draft202012Validator

EXTRACT_TOOL = {
    "name": "record_invoice",
    "description": (
        "Record the fields extracted from an invoice. Use null for anything the "
        "document does not state. Never guess or calculate missing values."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "vendor": {"type": ["string", "null"], "description": "Seller name, or null if the document is not an invoice."},
            "invoice_number": {"type": ["string", "null"]},
            "invoice_date": {"type": ["string", "null"], "description": "ISO format YYYY-MM-DD, or null."},
            "currency": {"enum": ["USD", "EUR", "INR", "other"]},
            "currency_other": {"type": ["string", "null"], "description": "ISO code when currency is 'other', else null."},
            "line_items": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "description": {"type": "string"},
                        "quantity": {"type": "number"},
                        "unit_price": {"type": "number"},
                    },
                    "required": ["description", "quantity", "unit_price"],
                },
            },
            "subtotal": {"type": ["number", "null"]},
            "tax": {"type": ["number", "null"]},
            "total": {"type": ["number", "null"], "description": "Total exactly as printed, or null if not printed."},
            "confidence": {"enum": ["high", "medium", "low"]},
        },
        "required": ["vendor", "invoice_number", "invoice_date", "currency", "currency_other",
                     "line_items", "subtotal", "tax", "total", "confidence"],
    },
}

_schema_validator = Draft202012Validator(EXTRACT_TOOL["input_schema"])

# Rule codes that retrying cannot fix: the information is simply not in the source.
NON_RETRYABLE = {"no_vendor"}


def validate(data: dict) -> list[tuple[str, str]]:
    """Return a list of (code, message). Empty list means valid."""
    errors = [("schema", f"{'/'.join(map(str, e.path)) or 'root'}: {e.message}")
              for e in _schema_validator.iter_errors(data)]
    if errors:
        return errors  # do not run business rules on a malformed object

    if data["vendor"] is None:
        errors.append(("no_vendor", "vendor is null: document does not look like an invoice"))
    if data["currency"] == "other" and not data["currency_other"]:
        errors.append(("currency", "currency is 'other' but currency_other is empty"))
    if data["invoice_date"] is not None:
        if not re.fullmatch(r"\d{4}-\d{2}-\d{2}", data["invoice_date"]):
            errors.append(("date", f"invoice_date '{data['invoice_date']}' is not YYYY-MM-DD"))
        else:
            try:
                date.fromisoformat(data["invoice_date"])
            except ValueError:
                errors.append(("date", f"invoice_date '{data['invoice_date']}' is not a real date"))

    items_sum = round(sum(i["quantity"] * i["unit_price"] for i in data["line_items"]), 2)
    if data["subtotal"] is not None and data["line_items"] and abs(items_sum - data["subtotal"]) > 0.01:
        errors.append(("subtotal", f"line items add up to {items_sum} but subtotal is {data['subtotal']}"))

    base = data["subtotal"] if data["subtotal"] is not None else (items_sum if data["line_items"] else None)
    if base is not None and data["total"] is not None:
        expected = round(base + (data["tax"] or 0), 2)
        if abs(expected - data["total"]) > 0.01:
            errors.append(("total", f"total is {data['total']} but base {base} + tax {data['tax'] or 0} = {expected}"))
    return errors
```

*Why, piece by piece:*
- The schema is a **tool** called `record_invoice`. Claude "calls" it to give us structured data.
- Fields that may be absent are **nullable** (`["number", "null"]`). This is the best defence against invented values: Claude is allowed to say "not stated".
- `currency` is an **enum with an `other` escape hatch** plus `currency_other`, so unusual currencies do not force a wrong answer.
- `confidence` lets you route uncertain extractions for review.
- `validate()` runs in two layers. The **schema check** guarantees shape. The **business rules** check meaning: line items add up to the subtotal, subtotal plus tax equals the total, dates are real ISO dates.
- Rule codes in `NON_RETRYABLE` mark problems that retrying cannot fix.

### 3. Add labelled sample documents

```powershell
notepad samples.py
```

```python
"""Six labelled sample documents. `expected` holds ground truth for scoring."""

SAMPLES = [
    {
        "id": "01-clean-usd",
        "text": """INVOICE INV-1001
Acme Supplies Inc.
Date: 2026-03-14
2 x Widget @ $10.00 = $20.00
1 x Gadget @ $5.50 = $5.50
Subtotal: $25.50
Tax: $2.04
Total due: $27.54""",
        "expected": {"vendor": "Acme Supplies Inc.", "invoice_number": "INV-1001", "currency": "USD", "total": 27.54},
    },
    {
        "id": "02-euro-german-date",
        "text": """Rechnung Nr. R-2207
Mueller GmbH, Berlin
Datum: 05.04.2026
3 Stueck Dichtung a 12,00 EUR = 36,00 EUR
MwSt 19%: 6,84 EUR
Gesamtbetrag: 42,84 EUR""",
        "expected": {"vendor": "Mueller GmbH", "invoice_number": "R-2207", "currency": "EUR", "total": 42.84},
    },
    {
        "id": "03-inr-informal",
        "text": """Sharma Traders - Tax Invoice #ST/889
10 pcs Steel bolts at Rs. 150 each = Rs. 1500
GST (18%) Rs. 270
Grand total: Rs. 1770
Dated: 2nd Feb 2026""",
        "expected": {"vendor": "Sharma Traders", "invoice_number": "ST/889", "currency": "INR", "total": 1770.0},
    },
    {
        "id": "04-gbp-other-currency",
        "text": """Bristol Parts Ltd
Invoice: BP-5521    Date: 19 May 2026
4 x Brake pad set @ GBP 25.00 = GBP 100.00
VAT 20%: GBP 20.00
Total: GBP 120.00""",
        "expected": {"vendor": "Bristol Parts Ltd", "invoice_number": "BP-5521", "currency": "other", "total": 120.0},
    },
    {
        "id": "05-total-not-printed",
        "text": """Northwind Consulting
Invoice NC-77, 2026-06-01
1 x Strategy workshop @ $500.00
Total: (to be confirmed after scope review)""",
        "expected": {"vendor": "Northwind Consulting", "invoice_number": "NC-77", "currency": "USD", "total": None},
    },
    {
        "id": "06-source-arithmetic-error",
        "text": """Delta Freight Co.
Invoice DF-310, 2026-07-09
2 x Pallet shipping @ $50.00 = $100.00
Subtotal: $100.00
Tax: $10.00
Total due: $150.00""",
        "expected": {"vendor": "Delta Freight Co.", "invoice_number": "DF-310", "currency": "USD", "total": 150.0},
    },
    {
        "id": "07-not-an-invoice",
        "text": """Hi team, lunch on Friday at 1pm? Let me know if you have dietary needs. Thanks, Priya""",
        "expected": {"vendor": None, "invoice_number": None, "currency": "USD", "total": None},
    },
]
```

*Why:* seven documents with known correct answers (`expected`) so you can measure accuracy. They include traps: a German-format date, a non-listed currency (GBP), a missing total, a total that does not match its line items (document 06), and a message that is not an invoice at all (document 07).

### 4. Write the extraction and retry logic

```powershell
notepad extract.py
```

```python
"""Forced-tool extraction with few-shot examples and a capped retry loop."""
import json
import os

import anthropic

from schema import EXTRACT_TOOL, NON_RETRYABLE, validate

MODEL = os.getenv("CLAUDE_MODEL", "claude-sonnet-5-5")

RULES = """You extract invoice data. Rules:
- Call the record_invoice tool exactly once.
- Copy values as printed. Never calculate or guess a missing value; use null.
- Dates must be ISO YYYY-MM-DD. Use currency 'other' plus currency_other for anything but USD, EUR, INR.
- If the document is not an invoice, set vendor to null, line_items to [] and confidence to low."""

EXAMPLES = """<examples>
<example>
<document>Invoice A-9 from Orion Labs dated 2026-01-02. 1 x Sensor @ $40.00. Total $40.00</document>
<output>{"vendor":"Orion Labs","invoice_number":"A-9","invoice_date":"2026-01-02","currency":"USD","currency_other":null,"line_items":[{"description":"Sensor","quantity":1,"unit_price":40.0}],"subtotal":null,"tax":null,"total":40.0,"confidence":"high"}</output>
</example>
<example>
<document>Reminder: the quarterly review is on Thursday.</document>
<output>{"vendor":null,"invoice_number":null,"invoice_date":null,"currency":"USD","currency_other":null,"line_items":[],"subtotal":null,"tax":null,"total":null,"confidence":"low"}</output>
</example>
</examples>"""


def system_prompt(use_examples: bool = True) -> str:
    return RULES + ("\n\n" + EXAMPLES if use_examples else "")


def _call(client, messages, use_examples):
    return client.messages.create(
        model=MODEL, max_tokens=1024,
        system=system_prompt(use_examples),
        tools=[EXTRACT_TOOL],
        tool_choice={"type": "tool", "name": EXTRACT_TOOL["name"]},  # force the tool
        messages=messages,
    )


def extract_with_retry(text: str, client, max_retries: int = 2, use_examples: bool = True) -> dict:
    """Return {status, data, attempts, first_try_valid, errors}. status: accepted | review."""
    messages = [{"role": "user", "content": f"<document>\n{text}\n</document>"}]
    first_try_valid = None
    errors: list = []
    data = None
    for attempt in range(1, max_retries + 2):          # 1 try + max_retries retries
        resp = _call(client, messages, use_examples)
        block = next(b for b in resp.content if b.type == "tool_use")
        data = block.input
        errors = validate(data)
        if first_try_valid is None:
            first_try_valid = not errors
        if not errors:
            return {"status": "accepted", "data": data, "attempts": attempt,
                    "first_try_valid": first_try_valid, "errors": []}
        if any(code in NON_RETRYABLE for code, _ in errors):
            break                                       # information is missing; retry is waste
        # Feed the exact errors back as a tool_result error on the failed call.
        messages.append({"role": "assistant", "content": resp.content})
        messages.append({"role": "user", "content": [{
            "type": "tool_result", "tool_use_id": block.id, "is_error": True,
            "content": "Validation failed:\n" + "\n".join(f"- {m}" for _, m in errors)
                       + "\nFix only what is wrong. Do not invent values; use null if the source lacks them.",
        }]})
    return {"status": "review", "data": data, "attempts": attempt,
            "first_try_valid": first_try_valid, "errors": [m for _, m in errors]}


if __name__ == "__main__":
    from samples import SAMPLES
    client = anthropic.Anthropic()
    result = extract_with_retry(SAMPLES[0]["text"], client)
    print(json.dumps(result, indent=2))
```

*Why, piece by piece:*
- `tool_choice={"type": "tool", "name": "record_invoice"}` **forces** Claude to use the tool, so there is no stray prose to parse.
- The system prompt holds the rules and two **few-shot examples** in `<examples>` tags. One example is a "nothing found" case so Claude learns that null and low confidence are acceptable.
- `extract_with_retry` validates every answer. On failure it appends the failed assistant turn and a `tool_result` with `is_error: True` that lists the **exact errors**. Claude then corrects only what is wrong.
- Retries are **capped** (`max_retries`). Errors in `NON_RETRYABLE` skip the retry loop and go straight to review.
- `first_try_valid` is recorded before any retry so you can measure how much the retries are helping.

### 5. Try one document

```powershell
python extract.py
```

Expected: a JSON result with `"status": "accepted"`, `"attempts": 1`, and `vendor` equal to `Acme Supplies Inc.`.

*Why:* a quick check that the key, model, schema and validators all work together before running everything.

### 6. Write the pipeline runner

```powershell
notepad pipeline.py
```

```python
"""Run all samples and print the three metrics: valid rate, accuracy, review rate."""
import argparse
import json

import anthropic

from extract import extract_with_retry
from samples import SAMPLES

FIELDS = ["vendor", "invoice_number", "currency", "total"]


def run(client, max_retries: int, use_examples: bool) -> dict:
    rows = []
    for s in SAMPLES:
        r = extract_with_retry(s["text"], client, max_retries, use_examples)
        got = r["data"] or {}
        correct = sum(got.get(f) == s["expected"][f] for f in FIELDS)
        rows.append({"id": s["id"], **r, "correct": correct})
    n = len(rows)
    summary = {
        "first_try_valid_rate": sum(r["first_try_valid"] for r in rows) / n,
        "accepted_rate": sum(r["status"] == "accepted" for r in rows) / n,
        "review_rate": sum(r["status"] == "review" for r in rows) / n,
        "field_accuracy": sum(r["correct"] for r in rows) / (n * len(FIELDS)),
        "total_attempts": sum(r["attempts"] for r in rows),
    }
    return {"rows": rows, "summary": summary}


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--max-retries", type=int, default=2)
    ap.add_argument("--no-examples", action="store_true", help="drop few-shot examples")
    args = ap.parse_args()

    out = run(anthropic.Anthropic(), args.max_retries, not args.no_examples)
    print(f"{'document':28} {'status':9} {'tries':5} {'fields ok':9}")
    for r in out["rows"]:
        print(f"{r['id']:28} {r['status']:9} {r['attempts']:<5} {r['correct']}/{len(FIELDS)}")
        for e in r["errors"]:
            print(f"    -> {e}")
    print()
    for k, v in out["summary"].items():
        print(f"{k:22} {v:.2f}" if isinstance(v, float) else f"{k:22} {v}")
    with open("results.json", "w", encoding="utf-8") as f:
        json.dump(out, f, indent=2)


if __name__ == "__main__":
    main()
```

*Why:* it runs all seven documents and reports three numbers: the **first-try valid rate**, the **accepted and review rates**, and **field accuracy** against the labels. It also saves `results.json`.

### 7. Run the pipeline

```powershell
python pipeline.py
```

Expected output (details will vary):

```
document                     status    tries fields ok
01-clean-usd                 accepted  1     4/4
02-euro-german-date          accepted  1     4/4
03-inr-informal              accepted  1     4/4
04-gbp-other-currency        accepted  1     4/4
05-total-not-printed         accepted  1     4/4
06-source-arithmetic-error   review    3     4/4
    -> total is 150.0 but base 100.0 + tax 10.0 = 110.0
07-not-an-invoice            review    1     4/4
    -> vendor is null: document does not look like an invoice

first_try_valid_rate   0.71
accepted_rate          0.71
review_rate            0.29
field_accuracy         1.00
total_attempts         9
```

*Why, read it row by row:*
- **05**: the total is not printed, so the correct result has `total: null`. It is accepted. Nullable fields prevented an invented number.
- **07**: not an invoice. The vendor is null, so the rule `no_vendor` sends it to review at once with **1 attempt**. Retrying could never create a vendor.
- **06**: the source document is internally inconsistent. Look carefully at what your run did. Two outcomes are possible:
  1. Claude keeps `150.0`, the retries are used up, and the document goes to review. This is the correct behavior.
  2. Claude "fixes" the total to `110.0` to satisfy the validator. The document is accepted, but the data no longer matches the source, and `fields ok` drops to `3/4`. This is **silent data corruption**, and it is the reason retry messages say "do not invent values".

### 8. Break it: remove the few-shot examples

```powershell
python pipeline.py --no-examples
```

*Why:* compare `first_try_valid_rate` and `total_attempts` with step 7. Examples teach format (ISO dates, `other` for GBP, null for missing data). The difference may be small with a strong model; the point is to measure it rather than assume.

### 9. Break it: no retries

```powershell
python pipeline.py --max-retries 0
```

*Why:* every first-try failure now goes to the review queue. Compare `accepted_rate` and `review_rate`. This is the value of the retry loop, and its cost is the extra attempts.

### 10. Tighten a rule and watch the effect

Open `schema.py`. In `validate()`, change the tolerance `0.01` in the total rule to `0.0001` and add a new rule after the date check:

```python
    if data["confidence"] == "low" and data["vendor"] is not None:
        errors.append(("low_conf", "confidence is low: re-read the document or use null for unclear fields"))
```

Run `python pipeline.py` again.

*Why:* business rules are code you can change without touching the prompt. Check what changed in the table. Remove the rule when you are done if you want to compare with the earlier results.

## Verify

- Step 7 accepts documents 01 to 05 and sends 07 to review after a single attempt.
- You can explain what your run did with document 06, and why it matters.
- You have compared `first_try_valid_rate` across steps 7, 8 and 9.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `StopIteration` in `extract.py` | The response had no tool call. Check `tool_choice` is exactly as shown and that you did not enable extended thinking, which is not compatible with a forced tool. |
| `ModuleNotFoundError: jsonschema` | Run `pip install jsonschema` in the activated environment. |
| `400 ... tool_use ids were found without tool_result` | In a retry, the `tool_result` must follow the assistant message and use the same `tool_use_id`. Compare with `extract.py`. |
| Every document goes to review | Print the errors (they are listed under each row). A typo in a rule is the usual cause. |
| `UnicodeEncodeError` in the console | Run `$env:PYTHONUTF8 = "1"` and retry. |

## Stretch goals

1. Add a `evidence` field with the exact text snippet that supports `total`, and validate that the snippet appears in the document.
2. Process the seven documents in parallel with a thread pool and compare run time.
3. Add a tenth sample of your own that your pipeline gets wrong, then fix it with a rule, an example, or a schema change.

## Exam takeaways

- Force structure with a **tool schema** (`tool_choice`) or structured outputs, not with "please return JSON".
- Make possibly-missing fields **nullable**; use **enum plus other**; add **confidence**.
- A schema guarantees **shape, not truth**. Validate business rules too.
- On failure, **feed back the specific errors**, cap retries, and route leftovers to review.
- **Retries fix slips, not missing source data.** Do not retry when the information is not there.
- Use few-shot examples (including a "nothing found" case) inside XML tags.

## Cleanup

```powershell
Remove-Item results.json
```
