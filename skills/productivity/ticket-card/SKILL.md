---
name: ticket-card
description: Write, summarize, or split a ticket/issue description for engineering work — bug, feature, or task/chore — for any tracker (Jira, Linear, GitHub Issues, Azure DevOps, etc.). Three modes — (1) write a new description from scratch, (2) summarize an existing one, (3) split a requirement/ticket into smaller sub-tickets, e.g. for an AI-driven task breakdown workflow. Trigger on /ticket-card, "write a ticket/issue", "summarize this ticket/issue", "split this into tickets", "break this down into tasks", or when the user pastes a ticket and asks for a summary or breakdown.
---

# Ticket Card

Write, summarize, or split engineering ticket/issue descriptions. Tool-agnostic (Jira, Linear, GitHub Issues, Azure DevOps, or anything else). Output is compact and structurally consistent — readable at a glance by a human, and parseable by downstream automation (e.g. an orchestrator consuming sub-tickets).

## When to invoke

- `/ticket-card`
- "write a ticket/issue" · "summarize this ticket/issue" · "split this into tickets" · "break this down into tasks"
- User pastes a ticket and asks for a summary or breakdown

## Ticket types

- **Bug** — something broken
- **Feature/story** — new capability
- **Task/chore** — no user-facing behavior change (refactor, upgrade, cleanup, infra)

Infer type from context; ask if ambiguous. Don't force the bug template onto a feature — there's no "root cause" for something that doesn't exist yet.

---

## Mode 1 — Write new

**Required inputs (ask if missing, never fabricate):**

| Type | Needed |
|---|---|
| Bug | problem, root cause, repro steps, solution |
| Feature | goal, current-behavior gap, proposed solution, acceptance criteria |
| Task | what, why now, definition of done |

**Templates.** Every ticket opens with a one-line `Type:`/`Title:` header — cheap for a human to skim and the one fixed anchor automation can grep for:

```
Bug:
**Type:** Bug   **Title:** <short title>
## Problem            What breaks, who/what is affected
## Root Cause         Mechanism — name files/functions/fields
## How to Reproduce   Numbered steps → observable symptom
## Solution           Bullets: what changed, why it fixes root cause
## Impact & Test Scope  Concrete QA test cases: happy path, edge cases, regressions

Feature:
**Type:** Feature   **Title:** <short title>
## Goal               What, for whom, why it matters
## Proposed Solution  Bullets: the approach, files/endpoints/components
## Acceptance Criteria  Checklist, testable, done/not-done — no vague statements
## Out of Scope (opt.)  What this deliberately excludes
## Impact & Test Scope  Concrete QA test cases: happy path, edge cases, regressions

Task/chore:
**Type:** Task   **Title:** <short title>
## Summary   What the work is
## Why       Motivation
## Definition of Done  Checklist
```

If the work covers more than one independently shippable/testable outcome, use **Mode 3** instead (INVEST: independent, negotiable, valuable, estimable, small, testable).

**Rules:** English, plain language, no jargon. Code identifiers welcome. Bug needs Impact & Test Scope; Feature needs both Acceptance Criteria *and* Impact & Test Scope — criteria define "done," test scope defines "how QA verifies it." Generic terminology only — no Jira-specific fields (story points, epic links) unless the user's context uses them.

---

## Mode 2 — Summarize existing

Compress a pasted description — no splitting.

```
**Type:** <Bug|Feature|Task>   **Title:** <short title>
## Summary       2–3 sentences: what it's about, the work, the outcome
## Key Points    Root cause/goal (1 line) · solution/approach (1 line) · who/what affected
## Test Scope    Condensed, if present in original
```

**Rules:** Don't add info not in the original. Flag missing critical info (no root cause / no acceptance criteria) after the summary. Preserve technical identifiers exactly.

---

## Mode 3 — Split into sub-tickets

Use when a requirement/ticket is too large for one shippable/testable unit, or the user asks to break it down. This is the mode a pipeline orchestrator should call for its ticket-breakdown stage — output is structured for machine parsing as much as reading.

**Split along:** layers (API vs. UI vs. migration), components/services, or dependency-safe sequence steps. Not arbitrary size — each piece must ship/test on its own.

**Required inputs:** the source requirement/ticket, and any sequencing constraints. If detail is missing for a sub-ticket, ask — don't invent it.

```
## Split Summary   Why split, how many sub-tickets
## Sub-tickets
### 1. <Title> [bug | feature | task]
depends-on: none
<full Mode 1 template body for that type — Type/Title line through Impact & Test Scope or Definition of Done>
### 2. <Title> [bug | feature | task]
depends-on: 1
<...>
```

**Rules:** No fabricated scope beyond the source. Prefer fewer well-scoped sub-tickets over max decomposition — split until independently shippable/testable, then stop. The `### N. <Title> [type]` heading and `depends-on:` line are fixed format — keep them stable so other automation can extract individual sub-tickets and their order.

---

## Tone & style

English. Short sentences, active voice. No hedging ("appears to", "may have"). No bullet padding — each bullet is a distinct fact. Test-scope items are testable actions, not topics.

## Output flow

1. Detect mode (write / summarize / split)
2. Detect ticket type (per ticket, including each sub-ticket in Mode 3)
3. Check required inputs — if missing, list them and stop
4. Produce draft as a single chat block
5. Offer a follow-up: tone adjustment (Modes 1–2) or split-boundary adjustment (Mode 3)
