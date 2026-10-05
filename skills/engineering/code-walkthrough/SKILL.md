---
name: code-walkthrough
description: Walk the user through code that an agent (Claude) just implemented, so a human reviewer can understand a large change without reading every file. Two modes: quick (annotate the diff, one line per hunk saying what it does) and deep (flow-ordered chapters, then line-by-line through a chosen group of code). Deep organises the explanation by runtime flow (entry → decision → side effects → failure path), not by file; every step points to file:line, says why the logic is shaped that way, and separates what was verified (commands run) from what was only reasoned. Trigger on /code-walkthrough, or when the user asks to be walked through / explained the changes the agent made ("พาผม walkthrough", "อธิบาย code ที่แก้", "ช่วยเล่า flow", "change เยอะ review ไม่ไหว", "walk me through the diff"). Mode words: quick / สั้น / อธิบาย diff → quick; deep / ละเอียด / ไล่ทีละบรรทัด → deep.
---

# Code walkthrough

The reviewer did not write this code and the change is too big to read cold. Your job is to make the code
*readable in the order it runs*, and to be honest about what you decided on their behalf.

## Rules

- **Deep mode: flow, not file list.** (Quick mode is the exception: it follows the diff, file by file.) Tell it the way a request travels: entry point → decision → side effects → failure/rollback.
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

## Modes

Pick the mode from the request; if it names none, use **quick** and offer deep at the end.

| Mode | Answers | Output |
| --- | --- | --- |
| **quick** | "What does each part of this diff do?" | The diff annotated: per file, per hunk, one line of what it does. |
| **deep** | "How does this group of code work, line by line?" | Flow map and chapters (the Workflow below), then a line-by-line pass over each group the reviewer picks. |

Both modes share the Rules. Fix the change set (Workflow step 1) first in either.

### Quick mode

1. Fix the change set (step 1), then read the diff hunk by hunk from the current files (`git diff`), not from memory.
2. Group hunks by file, in the order a reviewer should read them. For each hunk write one line: `file:line-range — what it does`, plus a short why only when the intent is not obvious from the code.
3. Mark what deserves a human eye (risky logic, a deviation from the ticket, anything unproven) with a single flag, not a paragraph.
4. Close with 2-3 lines: how many files / hunks, what was excluded, and an offer: "deep on <group>?" naming the group most worth it.

No flow map, no chapters, no ledger. Keep it to what fits on a screen or two; a hunk that needs more than a line is a candidate for deep mode.

### Deep mode

Run the Workflow below (flow map, chapters, decisions, verification). Then, for each group of code the reviewer picks (or the 1-3 riskiest if they name none):

- Name the group (a function, a module, a handler) and say what it is for in one sentence.
- Go **line by line, or by the smallest run of lines that does one thing**: `file:line` — what this line does — why it is there. Quote the line or a short fragment; do not paste the whole block first.
- Follow the data: for each variable or branch say where it comes from and what it can be, including the empty / null / error cases.
- End the group with what a reader should now believe about it, and which test or command shows it (or that nothing does).

Lines that are plain boilerplate (imports, trivial getters) are grouped in one line, not narrated.

## Workflow (deep mode)

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
- No restating the diff line by line, except in the line-by-line pass of deep mode, where it is the point; no praise; no hedging about things you verified.
- If something looks wrong while re-reading, say so and fix or flag it — the walkthrough is also a last self-review.
- Do not publish the walkthrough anywhere unless the user asks; it is for the person who asked for it.
