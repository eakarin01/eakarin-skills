---
name: jira-card
description: Write or summarize a Jira card description for engineering tickets. Handles two modes — (1) write a new description from scratch given bug/feature context, (2) summarize an existing description into a concise format. Trigger on /jira-card, when the user says "เขียน jira", "เขียน description ของ jira", "ช่วยเขียน card", "สรุป jira", "summarize jira", "write jira description", or pastes an existing Jira description and asks for a summary.
---

# Jira Card

Write or summarize Jira card descriptions for engineering tickets. Engineer-audience. Keep it concise and actionable — QA and other engineers should be able to understand the problem, reproduce it, and verify the fix without asking follow-up questions.

## When to invoke

- `/jira-card`
- "เขียน jira / เขียน description ของ jira / ช่วยเขียน card"
- "สรุป jira / summarize jira / write jira description"
- User pastes an existing Jira description and asks for a summary or rewrite

---

## Mode 1 — Write new description

Use when the user provides context about a bug or feature and wants a card written from scratch.

### Required inputs — ask if missing

- [ ] **What is the problem?** What breaks or is missing?
- [ ] **Root cause** (for bugs) — what in the code causes it?
- [ ] **How to reproduce** — steps to trigger the issue
- [ ] **Solution** — what was or will be changed to fix it?

If any are missing, list what's needed and stop. Do not fabricate.

### Output structure

```
## Problem
One paragraph. What breaks, in user/system terms. Who/what is affected.

## Root Cause
The actual bug mechanism. Name files, functions, column types, or API fields involved.
For features: explain why the current behavior falls short.

## How to Reproduce
1. Step-by-step numbered list
2. Observable symptom at the end

## Solution
- Bullet list of code changes, migrations, or config updates
- Each bullet: what changed and why it addresses the root cause

## Impact & Test Scope
What QA must verify. Group by feature area.
- Each item is a concrete test case, not a vague area
- Include happy path, edge cases, and regression checks
```

### Rules for Mode 1

- Write in **English**, plain language — no jargon that a new team member wouldn't understand
- Code identifiers (file names, function names, column names) are welcome and expected
- Do not invent root cause or solution details — ask the user if unclear
- Impact & Test Scope is mandatory — QA uses it to know what to test

---

## Mode 2 — Summarize existing description

Use when the user pastes an existing Jira description and asks for a shorter or cleaner version.

### Output structure

```
## Summary
2–3 sentences. What the ticket is about, what breaks, and what the fix is.

## Key Points
- Root cause in one line
- Solution in one line
- Who/what is affected

## Test Scope (if present in original)
Condensed bullet list of what to verify
```

### Rules for Mode 2

- Do not add information not present in the original — only compress and clarify
- If the original is missing critical information (e.g. no root cause), flag it after the summary
- Preserve technical identifiers exactly as written — do not paraphrase function names or file paths

---

## Tone & style

- **English** for all output
- Short sentences. Active voice.
- No hedging ("appears to", "may have", "we believe") — state it or don't
- No bullet padding — each bullet must carry a distinct fact
- QA-first mindset for Impact & Test Scope — write each item as a testable action, not a topic

---

## Output flow

1. **Detect mode** from context (new description vs. existing to summarize)
2. **Check required inputs** — if Mode 1 and inputs are missing, list what's needed and stop
3. **Produce draft** as a single chat block
4. After output, offer: *"Want me to adjust tone for a specific audience (e.g. management summary)?"*
