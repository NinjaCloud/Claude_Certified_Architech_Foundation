# Lab: Apply Prompt Caching and Compaction to a Long-Running Conversation

### Objective

In this lab, you will learn how to:

- Understand why long conversations become expensive.
- Understand **prompt caching**.
- Understand **context compaction**.
- Use caching for information that is reused many times.
- Compact an old conversation into a short summary.
- Combine caching + compaction in an AI application.

### Simple Scenario

Imagine a customer is chatting with an AI support agent:

```text
Customer → AI Agent

Customer: I purchased a laptop.
AI: What is the problem?

Customer: The laptop is overheating.
AI: When did this start?

Customer: Yesterday...
...
```

After 50–100 messages, the conversation becomes very long.

The agent needs to send a lot of previous information to Claude again and again.

That means:

```text
More context
     ↓
More input tokens
     ↓
Higher cost
     ↓
Slower requests
```

We will solve this using **two techniques**:

```text
Prompt Caching
      +
Compaction
      ↓
Lower repeated processing
      +
Smaller conversation context
```

---

# Part 1 — Understand the Problem

Suppose our application has this system prompt:

```text
You are a customer support assistant.

Company: ABC Electronics

Return Policy:
- Customers can return products within 30 days.
- Products must be in original condition.
- Refunds are processed within 5 business days.

Warranty Policy:
- Laptops have a 2-year warranty.
- Batteries have a 1-year warranty.
- Physical damage is not covered.

Support Rules:
- Never promise a refund immediately.
- Always verify the order number.
- Never ask for a customer's password.
```

This information rarely changes.

But imagine sending this information with **every request**.

### Request 1

```text
System instructions
+
Customer question
```

### Request 2

```text
System instructions
+
Previous conversation
+
Customer question
```

### Request 3

```text
System instructions
+
Previous conversation
+
Customer question
```

The application repeatedly sends the same information.

This is where **prompt caching** can help.

---

# Part 2 — What Is Prompt Caching?

Think about a teacher explaining something to a student.

The teacher says:

> "Our company return policy is..."

The student understands it.

Five minutes later the teacher doesn't need to explain the entire policy again.

Instead:

```text
Already explained information
        ↓
Keep it available
        ↓
Reuse it
```

Prompt caching works on a similar idea.

You identify a portion of the prompt that is **reused across requests** and allow the model/API to reuse that cached prompt content.

### Good candidates for caching

```text
System instructions
Company policies
Product documentation
Large reference documents
Tool definitions
Large static instructions
```

### Poor candidates

```text
Current user question
Frequently changing data
One-time information
Temporary conversation messages
```

---

# Part 3 — Simple Example

Imagine our application sends:

```text
SYSTEM PROMPT
--------------------------------
You are a customer support agent.

Company policy:
1. Returns within 30 days.
2. Laptop warranty = 2 years.
3. Battery warranty = 1 year.
4. Physical damage isn't covered.

Support rules:
1. Verify order number.
2. Never request passwords.
3. Don't promise refunds.
--------------------------------

USER
--------------------------------
My laptop is overheating.
```

The company policy is relatively static.

So we can structure our application conceptually like:

```text
┌─────────────────────────────┐
│      CACHE THIS             │
│                             │
│ Company policies            │
│ Support instructions        │
│ Product documentation       │
└──────────────┬──────────────┘
               │
               ↓
        Claude request
               ↑
               │
┌──────────────┴──────────────┐
│       DON'T CACHE           │
│                             │
│ Current user question       │
│ Current conversation        │
└─────────────────────────────┘
```

---

# Part 4 — Create a Simple Python Application

We will use the Anthropic Python SDK.

### Step 1 — Create a folder

```bash
mkdir prompt-lab
cd prompt-lab
```

### Step 2 — Create a virtual environment

Ubuntu/macOS:

```bash
python3 -m venv venv
source venv/bin/activate
```

Windows PowerShell:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

### Step 3 — Install the SDK

```bash
pip install anthropic
```

### Step 4 — Set API key

Ubuntu/macOS:

```bash
export ANTHROPIC_API_KEY="your-api-key"
```

Windows PowerShell:

```powershell
$env:ANTHROPIC_API_KEY="your-api-key"
```

---

# Part 5 — First Simple Claude Request

Create:

```text
cache_demo.py
```

Example:

```python
import anthropic
import os

client = anthropic.Anthropic(
    api_key=os.environ["ANTHROPIC_API_KEY"]
)

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=500,

    system=[
        {
            "type": "text",
            "text": """
You are a customer support assistant.

Company Policy:
- Returns are allowed within 30 days.
- Laptops have a 2-year warranty.
- Batteries have a 1-year warranty.
- Physical damage is not covered.

Support Rules:
- Always verify the order number.
- Never ask for passwords.
- Never promise a refund immediately.
""",
            "cache_control": {
                "type": "ephemeral"
            }
        }
    ],

    messages=[
        {
            "role": "user",
            "content": "My laptop is overheating. What should I do?"
        }
    ]
)

print(response.content[0].text)
```

The important part is:

```python
"cache_control": {
    "type": "ephemeral"
}
```

This tells the API that this part of the prompt is a candidate for prompt caching.

---

# Part 6 — Run the Lab

Run:

```bash
python cache_demo.py
```

You should receive a response similar to:

```text
I'm sorry you're experiencing overheating.

Please provide your order number so we can check
whether the laptop is still under warranty.

You can also try:
- Checking whether the air vents are blocked.
- Using the laptop on a hard surface.
- Closing unnecessary applications.
```

---

# Part 7 — Understanding Cache Hits

Imagine we make multiple requests.

### Request 1

```text
Company Policy
       +
Question 1
```

### Request 2

```text
Company Policy
       +
Question 2
```

### Request 3

```text
Company Policy
       +
Question 3
```

The policy is the same.

Conceptually:

```text
                 ┌── Question 1
                 │
Cached Policy ───┼── Question 2
                 │
                 └── Question 3
```

Instead of treating the repeated static content as entirely new input each time, prompt caching can reduce the cost/latency associated with repeatedly processing that cached portion.

**Important teaching point:** caching doesn't mean Claude permanently "remembers" the information. It is an API-level optimization for reusable prompt content.

---

# Part 8 — Now Understand the Long Conversation Problem

Let's simulate a conversation.

```text
User:
I purchased a laptop.

Assistant:
Okay. What is the problem?

User:
It is overheating.

Assistant:
When did this start?

User:
Yesterday.

Assistant:
Does it shut down?

User:
Yes.

Assistant:
Can you provide your order number?

User:
ORD12345

Assistant:
Thank you. Let me check the warranty.

User:
The laptop is 8 months old.

Assistant:
Your laptop should still be within the 2-year warranty period.

User:
I also dropped it last week.

Assistant:
Was there any physical damage?

User:
Yes, there is a crack.
```

Now imagine **100 more messages**.

The conversation might look like:

```text
Message 1
Message 2
Message 3
...
Message 100
Message 101
```

If we keep sending the complete conversation, the context becomes increasingly large.

---

# Part 9 — What Is Compaction?

Compaction means:

> **Take an old, long conversation and turn it into a much smaller summary that preserves the important information.**

For example, instead of sending:

```text
Message 1
Message 2
Message 3
...
Message 100
```

we can create:

```text
Conversation Summary:

Customer purchased laptop ORD12345.
Laptop is 8 months old.
Laptop is overheating and sometimes shuts down.
Customer dropped the laptop last week.
There is visible physical damage.
Customer wants to know whether warranty applies.
```

Now our context becomes:

```text
Original conversation
       ↓
    Compact
       ↓
Short summary
```

---

# Part 10 — Simple Compaction Example

Create:

```text
compaction_demo.py
```

Use Claude to summarize the old conversation.

```python
import anthropic
import os

client = anthropic.Anthropic(
    api_key=os.environ["ANTHROPIC_API_KEY"]
)

conversation = """
Customer: I purchased a laptop.
Assistant: What is the problem?
Customer: The laptop is overheating.
Assistant: When did this start?
Customer: Yesterday.
Assistant: Does it shut down?
Customer: Yes.
Assistant: Please provide your order number.
Customer: ORD12345.
Assistant: How old is the laptop?
Customer: 8 months old.
Customer: I dropped it last week.
Assistant: Is there physical damage?
Customer: Yes, there is a crack.
"""

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=300,

    system="""
Summarize the conversation for another support agent.

Keep only information that is important for continuing
the conversation.

Include:
- Order number
- Product
- Problem
- Important events
- Customer's current question

Do not include unnecessary conversation.
""",

    messages=[
        {
            "role": "user",
            "content": conversation
        }
    ]
)

print(response.content[0].text)
```

Run:

```bash
python compaction_demo.py
```

You might get:

```text
Conversation Summary:

- Product: Laptop
- Order: ORD12345
- Age: 8 months
- Problem: Laptop overheats and sometimes shuts down
- Incident: Customer dropped the laptop last week
- Damage: Laptop has a visible crack
- Customer wants to know if the laptop is covered by warranty
```

---

# Part 11 — Continue the Conversation Using the Summary

Now instead of sending all previous messages, send the compacted summary.

```python
summary = """
Customer purchased a laptop, order ORD12345.

The laptop is 8 months old and overheats.
It sometimes shuts down.

The customer dropped the laptop last week
and the laptop now has a visible crack.

The customer wants to know whether the laptop
is covered by warranty.
"""

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=300,

    system="""
You are a customer support assistant.

Warranty Policy:
- Laptops have a 2-year warranty.
- Physical damage is not covered.

Answer based on the conversation summary.
""",

    messages=[
        {
            "role": "user",
            "content": summary + """

Customer:
Is my laptop covered by warranty?
"""
        }
    ]
)

print(response.content[0].text)
```

Claude can now continue the conversation without receiving every historical message.

---

# Part 12 — Combine Caching + Compaction

This is the important **Architect Foundation** concept.

A production application can have:

```text
                  Claude Application
                         │
              ┌──────────┴──────────┐
              │                     │
       Static Information       Conversation
              │                     │
              ↓                     ↓
       Prompt Caching           Compaction
              │                     │
              ↓                     ↓
       Company Policies        Conversation
       Product Docs            Summary
       Instructions
              │                     │
              └──────────┬──────────┘
                         ↓
                       Claude
```

### Example

```text
CACHE
────────────────────────────
Company policies
Product documentation
Agent instructions
Tool definitions
────────────────────────────

COMPACTED CONTEXT
────────────────────────────
Customer information
Previous conversation
Important decisions
Current issue
────────────────────────────

NEW MESSAGE
────────────────────────────
"Can I get a replacement?"
────────────────────────────
```

This gives us a much more efficient context design.

---

# Part 13 — When Should I Use Which?

| Situation | Technique |
|---|---|
| Same company policy used repeatedly | Prompt caching |
| Same product documentation used repeatedly | Prompt caching |
| Large system instructions | Prompt caching |
| Conversation has become very long | Compaction |
| Agent has hundreds of previous messages | Compaction |
| Static documents + long conversation | **Both** |
| Current user question | Don't cache |
| Temporary information | Usually don't cache |

---

# Part 14 — Simple Real-World Example

Imagine an **HR assistant**.

Every request uses the same:

```text
Employee Handbook
Leave Policy
Work From Home Policy
Company Rules
```

These are good candidates for caching.

The employee conversation might be:

```text
Employee:
I want leave.

AI:
What type?

Employee:
Annual leave.

AI:
How many days?

Employee:
5.

...
```

After a long conversation:

```text
100 messages
       ↓
Compaction
       ↓
"Employee wants 5 days annual leave.
They have discussed dates X and Y.
Manager approval is required."
```

The next request can use:

```text
Cached:
Employee Handbook
Leave Policy
Company Rules

+

Compacted:
Previous conversation

+

New:
"What documents do I need?"
```

---

# Part 15 — Architecture Exercise

Ask learners to classify these items.

### Question

Which should be **cached**, and which should be **compacted**?

```text
1. Company refund policy
2. 100 previous chat messages
3. Product documentation
4. Current customer question
5. Customer's previous decisions
6. Agent instructions
7. Tool definitions
8. Old conversation history
```

### Expected answer

```text
CACHE
─────
1. Company refund policy
3. Product documentation
6. Agent instructions
7. Tool definitions


COMPACTION
──────────
2. 100 previous chat messages
5. Customer's previous decisions
8. Old conversation history


CURRENT REQUEST
───────────────
4. Current customer question
```

---

# Part 16 — Key Architect Takeaways

Remember these five points for **Claude Architect Foundation**:

### 1. Prompt caching ≠ memory

Caching is mainly about **reusing prompt content efficiently**.

```text
Cache = reuse prompt content
```

### 2. Compaction ≠ caching

Compaction is about **reducing conversation history**.

```text
Compaction = shrink conversation context
```

### 3. Cache stable information

Good examples:

```text
Policies
Documentation
Instructions
Tool definitions
```

### 4. Compact old conversations

When conversations become large:

```text
Long conversation
       ↓
Summary
       ↓
Continue conversation
```

### 5. Production systems can use both

The ideal pattern is often:

```text
       Static Context
            ↓
        CACHE IT
            +
       Old History
            ↓
       COMPACT IT
            +
       New Request
            ↓
          Claude
```

## Final Lab Challenge

Modify the lab to create a **travel assistant**.

Create a static prompt containing:

```text
Hotel cancellation policy
Flight change policy
Luggage policy
Travel agent instructions
```

Then simulate a conversation with **20+ messages**.

Your application should:

1. Cache the static travel policies.
2. Maintain the conversation.
3. Create a compact summary after the conversation becomes long.
4. Continue the conversation using the summary.
5. Ask Claude:

```text
Can I change my flight and keep my hotel booking?
```

The learner should be able to explain **why the static policy is cached and why the old conversation is compacted**.
