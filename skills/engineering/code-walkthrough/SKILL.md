---
name: code-walkthrough
description: Walk the user through code that an agent (Claude) just implemented, so a human reviewer can understand a large change without reading every file. Organises the explanation by runtime flow (entry → decision → side effects → failure path), not by file; every step points to file:line, says why the logic is shaped that way, and separates what was verified (commands run) from what was only reasoned. Trigger on /code-walkthrough, or when the user asks to be walked through / explained the changes the agent made ("พาผม walkthrough", "อธิบาย code ที่แก้", "ช่วยเล่า flow", "change เยอะ review ไม่ไหว", "walk me through the diff").
---

# Code walkthrough

The reviewer did not write this code and the change is too big to read cold. Your job is to make the code
*readable in the order it runs*, and to be honest about what you decided on their behalf.

## Rules

- **Flow, not file list.** Tell it the way a request travels: entry point → decision → side effects → failure/rollback.
  A file appears where the flow reaches it, never as its own chapter.
- **Re-read the code before explaining it.** Never narrate from memory of what you wrote; the file may have changed
  (reviews, merges, formatter). Open each file and take `file:line` from the current text.
- **Scope = the change set only.** Exclude merge commits, upstream code and the user's own local edits. Say what you excluded.
- **Every step carries its "why".** Not "this calls X" but "this calls X because Y; the alternative was Z and was rejected because …".
- **Be candid about your own judgement calls.** Call out assumptions, ticket deviations, scope you added, and bugs you
  introduced and fixed on the way. The reviewer should not discover these themselves.
- **Verified vs reasoned.** For each claim of correctness say which command/test proves it, or that it is only reasoning
  and what would prove it (e.g. "needs Postgres").
- **Match the user's language** for prose (Thai if they write Thai); keep identifiers, paths and code in English.

## Workflow

1. **Fix the change set.** Use git, not memory: `git status --short`, `git diff HEAD --stat` (staged + unstaged), untracked
   files. Separate: feature changes / merge commits / user's local edits / generated files. Report the count (files, +/-).
2. **Find the entry points** the change hangs off (controller, consumer, cron). Read them and follow the calls until the
   change set is exhausted. Collect `file:line` as you go.
3. **Draw the map first** (one screen). A short ASCII flow of the main path with the decision points, plus a table
   `step → file:line → what it does`. This is what the reviewer keeps in their head.
4. **Write chapters, one per flow or decision,** in execution order. Per chapter:
   - *What happens* in 2-4 sentences, then the smallest code excerpt that shows it (5-15 lines, with `file:line`).
   - *Why it is shaped this way* and the rejected alternative, if there was one.
   - *Edge cases / failure behaviour* (what is rolled back, what is left, what the caller sees).
   - *Tests* that pin this behaviour (spec file + case names).
5. **Data/contract changes together,** in one short chapter: migrations, entity columns, API request/response shape,
   events, config. These are what other teams feel first.
6. **Decisions & assumptions table.** Each row: decision, why, who decided (user/agent), and how to change it.
7. **Verification ledger.** Table of claim → evidence (command + result) → status (verified / reasoned / not run).
   List what could not be run and why.
8. **Suggested review order.** 5-8 files in the order that makes the rest easy, and the 2-3 places where a human eye is
   most valuable (risky logic, assumptions, anything unproven).
9. **Offer to go deeper.** Large changes: finish the map + chapters, then ask which part to zoom into rather than
   dumping every file.

## Style

- Terminal markdown. Short paragraphs, tables for mappings, code fences only for excerpts you actually need.
- No restating the diff line by line; no praise; no hedging about things you verified.
- If something looks wrong while re-reading, say so and fix or flag it — the walkthrough is also a last self-review.
- Do not publish the walkthrough anywhere unless the user asks; it is for the person who asked for it.
