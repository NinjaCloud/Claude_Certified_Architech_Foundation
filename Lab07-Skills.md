# Lab: Create and Use a Custom Skill in Claude

## 1. Lab Objective

In this lab, you will learn:

- What a **Claude Skill** is
- Why Skills are useful
- How to create a custom Skill
- How to define instructions for a Skill
- How Claude uses a Skill
- How to call a Skill from **Claude Code**
- How Skills differ from prompts and tools

---

# 2. What is a Skill?

A **Skill** is a reusable set of instructions that teaches Claude how to perform a particular type of task.

Think about it like this:

```text
Normal Claude
      ↓
"Do this task"

Skill
      ↓
"Here are the instructions, rules and knowledge
you should follow whenever you perform this task."
```

For example, suppose your company frequently asks Claude to create meeting summaries.

Instead of writing this every time:

```text
Summarize this meeting.

Include:
- Decisions
- Action items
- Owners
- Deadlines

Use simple language.
Do not include unnecessary discussion.
```

You can create a reusable:

```text
Meeting Summary Skill
```

Then Claude can use those instructions whenever that task is required.

---

# 3. Skill vs Prompt vs Tool

This is an important Architect Foundation concept.

| Concept | Simple meaning |
|---|---|
| Prompt | Tells Claude what to do now |
| Skill | Reusable instructions for a type of task |
| Tool | Allows Claude to interact with something external |
| MCP | Standard way to connect Claude with external tools/data |

Example:

```text
Prompt:
"Summarize this meeting."

Skill:
"Always summarize meetings using Decisions,
Actions, Owners and Deadlines."

Tool:
"Create a Jira ticket."

MCP:
"Connect Claude to Jira."
```

A Skill primarily provides **reusable expertise/instructions**. It doesn't itself give Claude access to an external system.

---

# 4. Lab Scenario

We will create a very simple custom Skill:

> **Meeting Summary Skill**

Our Skill will tell Claude to convert raw meeting notes into:

```text
Meeting Summary

1. Key Decisions
2. Action Items
3. Owner
4. Deadline
5. Open Questions
```

---

# 5. Prerequisites

You need:

- Claude Code
- Claude API access
- Terminal
- Python is optional

Check Claude Code:

```bash
claude --version
```

If Claude Code is already installed, start it:

```bash
claude
```

---

# 6. Create a Skills Directory

For this lab, create a project:

```bash
mkdir claude-skills-lab
cd claude-skills-lab
```

Create the Skills directory:

```bash
mkdir -p .claude/skills/meeting-summary
```

Your structure should look like:

```text
claude-skills-lab/
└── .claude/
    └── skills/
        └── meeting-summary/
```

---

# 7. Create SKILL.md

Create:

```text
.claude/skills/meeting-summary/SKILL.md
```

Put this inside:

```markdown
---
name: meeting-summary
description: Summarizes meeting notes into decisions, action items, owners, deadlines, and open questions.
---

# Meeting Summary Skill

When this skill is used, convert raw meeting notes into a clear
and structured meeting summary.

## Output Format

Use the following sections:

### Key Decisions
List the important decisions made during the meeting.

### Action Items
List tasks that need to be completed.

For each action item include:
- Task
- Owner
- Deadline

### Open Questions
List questions that were not resolved.

## Rules

- Use simple business language.
- Do not invent information.
- If the owner is unknown, write "Not specified".
- If the deadline is unknown, write "Not specified".
- Remove unnecessary conversation.
- Keep the summary concise.
```

---

# 8. Understand SKILL.md

The most important part is the front matter:

```yaml
---
name: meeting-summary
description: Summarizes meeting notes into decisions, action items, owners, deadlines, and open questions.
---
```

Think of it as the Skill's metadata.

### `name`

```yaml
name: meeting-summary
```

This identifies the Skill.

### `description`

```yaml
description: Summarizes meeting notes...
```

This tells Claude what the Skill is useful for.

The remaining Markdown contains the actual instructions.

---

# 9. Test the Skill

Start Claude Code from your project directory:

```bash
claude
```

Give Claude a meeting note such as:

```text
We had a project meeting today.

The team decided to deploy the application on Azure.
Rahul will create the Azure resources by Friday.
Priya will prepare the deployment pipeline.
The team will review the deployment on Monday.

We still need to decide whether to use PostgreSQL or MySQL.
```

Ask:

```text
Use the meeting summary skill to summarize these notes.
```

Claude should produce something similar to:

```text
### Key Decisions

- The application will be deployed on Azure.

### Action Items

| Task | Owner | Deadline |
|---|---|---|
| Create Azure resources | Rahul | Friday |
| Prepare deployment pipeline | Priya | Not specified |

### Open Questions

- Should the application use PostgreSQL or MySQL?
```

---

# 10. How Does Claude Know About the Skill?

Conceptually:

```text
Claude Code
    |
    ↓
Looks at project
    |
    ↓
.claude/skills/
    |
    ↓
meeting-summary/
    |
    ↓
SKILL.md
    |
    ↓
Reads Skill instructions
    |
    ↓
Performs task
```

The important idea is that the Skill is **available as reusable project-level knowledge/instructions**.

You don't need to paste the complete instructions into every prompt.

---

# 11. Calling the Skill

There are two useful ways to think about calling a Skill.

### Natural-language invocation

You can ask Claude:

```text
Use the meeting summary skill.
```

or:

```text
Apply the meeting-summary skill to these notes.
```

Claude can determine that the Skill is relevant.

### Explicit invocation

In Claude Code, Skills can also be invoked explicitly using the Skill command mechanism.

For example:

```text
/meeting-summary
```

If your Claude Code version exposes the Skill as a slash command, Claude will load the corresponding Skill instructions and execute them.

> The exact Skill discovery/invocation behavior can vary with the Claude Code version, so use `claude --help` and the current Claude Code documentation if your environment doesn't expose the Skill as expected.

---

# 12. Add a Real Input File

Now let's make the lab more realistic.

Create:

```text
meeting.txt
```

Put this inside:

```text
Project: Banking Application

The development team discussed the production deployment.

Decision:
The application will be deployed to Azure Kubernetes Service.

Rahul:
Create the AKS cluster.

Priya:
Create the CI/CD pipeline.

Amit:
Prepare the production database.

Deadline:
All tasks should be completed by Friday.

Open question:
The team has not decided whether to use Azure Database
for PostgreSQL or Azure SQL Database.
```

Now ask Claude:

```text
Use the meeting-summary skill and summarize meeting.txt.
```

Claude can read the file and apply the Skill's instructions.

---

# 13. Improve the Skill

Now modify `SKILL.md`.

Add:

```markdown
## Quality Rules

Before producing the final summary:

1. Identify every decision.
2. Identify every action item.
3. Identify the owner of each action.
4. Identify deadlines.
5. Identify unresolved questions.
6. Never create missing information.
```

Now your Skill contains reusable business rules.

This is one of the major advantages of Skills.

---

# 14. Create Another Skill

Now create a second Skill:

```bash
mkdir -p .claude/skills/email-writer
```

Create:

```text
.claude/skills/email-writer/SKILL.md
```

Use:

```markdown
---
name: email-writer
description: Creates concise professional business emails from rough notes.
---

# Professional Email Skill

Convert rough notes into a professional business email.

## Rules

- Keep the email concise.
- Use a professional tone.
- Add a clear subject.
- Use short paragraphs.
- Do not invent facts.
- Do not use unnecessary corporate buzzwords.

## Structure

Subject

Greeting

Main message

Required action

Closing
```

Now your project has:

```text
.claude/
└── skills/
    ├── meeting-summary/
    │   └── SKILL.md
    │
    └── email-writer/
        └── SKILL.md
```

---

# 15. Test Multiple Skills

Start Claude:

```bash
claude
```

Ask:

```text
Use the meeting-summary skill on meeting.txt.
```

Then:

```text
Use the email-writer skill to create an email
informing the team about the decisions.
```

Now you have:

```text
             Claude Code
                 |
        ┌────────┴────────┐
        ↓                 ↓
Meeting Summary       Email Writer
     Skill                Skill
        ↓                 ↓
Structured            Professional
summary                  email
```

---

# 16. Skill + Tool

This is where Skills become interesting for agent architecture.

Suppose we have:

```text
Meeting Summary Skill
```

and a Jira tool.

The Skill could instruct Claude:

```text
After creating the meeting summary,
identify action items.

For each action item:
- identify the owner
- identify the deadline
- create a Jira ticket
```

The architecture becomes:

```text
                  Claude Agent
                      |
             ┌────────┴────────┐
             ↓                 ↓
          Skill              Tool
             |                 |
             ↓                 ↓
      Meeting rules          Jira
             |                 |
             └────────┬────────┘
                      ↓
                 Final result
```

**Skill tells Claude how to perform the task.**

**Tool allows Claude to actually do something outside Claude.**

That distinction is very important.

---

# 17. Skill + MCP

You can take this one step further.

For example:

```text
Meeting Summary Skill
          +
       MCP Server
          |
          ↓
        Jira
```

The Skill might contain instructions such as:

```text
When an action item is identified,
create a Jira ticket using the available Jira tool.
```

The MCP server provides the Jira capability.

So:

```text
Skill
  =
How should I perform this task?

MCP/Tool
  =
What external capability can I use?
```

---

# 18. Mini Architecture Exercise

Ask learners:

### Scenario

Your company wants an AI assistant that processes support tickets.

It must:

1. Read a support ticket.
2. Classify the issue.
3. Generate a response.
4. Create a Jira ticket for engineering issues.

What would you use?

### Answer

```text
Support Classification Skill
        +
Response Generation Skill
        +
Jira Tool/MCP
```

Architecture:

```text
             Support Agent
                  |
       ┌──────────┼──────────┐
       ↓          ↓          ↓
 Classification Response   Jira
    Skill        Skill      Tool
       |           |          |
       └───────────┴──────────┘
                  |
                  ↓
              Final result
```

---

# 19. Important Takeaways

For the exam, remember:

```text
Skill
 ↓
Reusable instructions
```

```text
Tool
 ↓
External capability
```

```text
MCP
 ↓
Standard connection to external tools/data
```

And:

```text
Prompt
 ↓
One request

Skill
 ↓
Reusable expertise/instructions

Tool
 ↓
Perform an external action

MCP
 ↓
Connect Claude to external capabilities
```

### Final challenge

Create your own:

```text
.claude/skills/incident-summary/SKILL.md
```

The Skill should take a technical incident such as:

```text
Application is down.
CPU is 95%.
Database connection errors started at 10:30.
Restarting the application did not help.
Database team has been contacted.
```

and produce:

```text
Incident Summary
Root Cause
Impact
Actions Taken
Next Actions
Owner
Status
```
