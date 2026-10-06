---
name: code-walkthrough
description: Walk the user through a change (code an agent just wrote, a PR or branch pair, or work that is still staged or unstaged), so a human reviewer can understand and review it without reading every file. Every walkthrough or review ends with ONE message in the VAO format (verdict, `## Walkthrough`, a `Compare:` line, notes pinned to file:line) that the virtual-agent-office review page opens as a file list plus diff; plain terminal markdown only when the user asks for it by name. Two modes: quick (one line per hunk saying what it does) and deep (flow-ordered steps, then line-by-line through a chosen group of code, with findings). Deep follows the runtime flow (entry → decision → side effects → failure path), not the file list; every note points to file:line, says why the logic is shaped that way, and separates what was verified (commands run) from what was only reasoned. Trigger on /code-walkthrough, or when the user asks to be walked through or to have a change explained ("พาผม walkthrough", "อธิบาย code ที่แก้", "ช่วยเล่า flow", "change เยอะ review ไม่ไหว", "walk me through the diff", "walkthrough change บน working tree"), or asks to review code, a PR, a branch pair or uncommitted work ("จะ review code ด้วย", "ช่วย review PR feat/x -> main", "review X against Y", "review ก่อน commit"). Mode words: quick / สั้น / อธิบาย diff → quick; deep / ละเอียด / ไล่ทีละบรรทัด → deep.
---

# Code walkthrough

The reviewer did not write this code and the change is too big to read cold. Your job is to make the code
*readable in the order it runs*, and to be honest about what you decided on their behalf.

## The output is always the VAO format

Every walkthrough and every review, quick or deep, ends with **one message in the VAO format** described below. The
user reads it on the virtual-agent-office review page: the changed files on the left, the diff on the right, your notes
under the lines they refer to. This is the default whatever the words of the request are ("walkthrough", "review", "ขอแบบ
ละเอียด"): do not wait for "review", "PR" or "VAO" to appear.

- **Terminal markdown only on request.** Write terminal-style markdown (headings of your choice, tables) only when the
  user asks for it by name ("in the terminal", "terminal format", "ไม่ต้องขึ้น VAO"). The user's current request is the
  only thing that can ask for it: a handoff note, a memory or an earlier session saying "terminal format" does not. When
  you do write terminal markdown, say in one line that it will not appear on the review page.
- **This skill wins over old notes about what the page can show.** The review page opens commit ranges (since
  2026-10-05) and uncommitted work, staged, unstaged and the whole working tree (since 2026-10-06). A memory or handoff
  that says it cannot show uncommitted work is out of date. If you doubt it, try: the page, not a note, is the test.
- **Tables are for this file, not for your message.** The page cannot render them.

### Choose the `Compare:` line from the scope of the work

Look first (`git status --short`, `git log --oneline -5`), then pick the one that matches what the user wants reviewed.
`X -> Y` as in "review PR X -> Y" reads like a PR: X is the head, Y the base.

| What is reviewed | `Compare:` |
| --- | --- |
| A PR or branch, "X -> Y" or "X against Y" | `` `Y...X` `` (base first; if only X is named, the base is the default branch, and say so) |
| Commits already made, "the last commit", "since abc123" | `` `HEAD~1...HEAD` ``, `` `abc123...HEAD` `` |
| Only what is staged ("ที่ staged") | `` `HEAD...@staged` `` |
| Only edits not yet added, plus new untracked files ("ที่ยัง unstaged") | `` `HEAD...@unstaged` `` |
| Everything not committed, staged and unstaged, plus new files (the default for "the changes on the working tree" when the user does not say which) | `` `HEAD...@worktree` `` |
| The whole branch plus uncommitted edits | `` `origin/main...@worktree` `` (any base the work will merge into) |

Refs must work exactly as written in `git diff <base>...<head>` (a branch, `origin/<branch>`, a commit hash, `HEAD~3`):
check each with `git rev-parse --verify`, and `git fetch` first when the network is allowed and a branch is only on the
remote. Do not check a branch out unless asked.

### Shape of the message

1. **Verdict (above the heading).** For a review: ready, ready with changes, or not ready, with the one or two reasons.
   For an explanation with no review asked: one line saying what the change does. Plain text, short.
2. **`## Walkthrough`**, and the **first line under it** is ``Compare: `base...head` `` (three dots, backticks, no spaces
   inside). One plain scope line may follow it (files and +/-, what was excluded, what was not run). For uncommitted work
   say in one line that notes refer to the files as they are now and move if the files are edited afterwards, and that
   `.env`-like files are shown without their content.
3. **One `###` heading per step**, in the order a reviewer should read.
4. **Under each heading, one bullet per note:** ``- `path:start-end` — text``. One line is ``- `path:12` — text``; a whole
   file (a deleted one too) is ``- `path` — text``. Lines are those of the head version. The separator is a dash with a
   space on each side; a long note may continue on following lines indented by two spaces. Use only this shape for
   notes.
5. **Findings are notes.** The bullet still starts with the path; the label opens the text after the dash:
   ``- `src/seat.ts:12` — **Risk:** the fallback hides a missing rule (read the code, not run)``. A bullet that starts
   with the label (``- **Risk:** `src/seat.ts:12` …``) is not a note: the page ignores it. Labels: `**Issue:**` (should
   change), `**Question:**` (needs the author's answer), `**Risk:**` (works but fragile or unproven), `**OK:**` (a
   decision worth confirming). Say how you know: read the code, ran a test, or reasoned. A finding that continues on
   its own line is indented by two spaces, so it stays part of the same note.
6. **Everything else** (the flow map, the suggested review order, the offer to go deeper) is a plain sentence or a
   bullet that does not start with a code-span path (that would be read as a note). The page does not show such lines
   yet, so anything the reader must see on the page belongs in a note.

### The last message wins, and it must be this one

The page shows the **last message of your turn** and replaces anything shown before it.
- Do every tool call (running tests, editing files or memory) **before** you write the walkthrough. If something has to
  happen after it, post the whole walkthrough again as the very last message.
- A later walkthrough replaces an earlier one, so a follow-up ("zoom into group X") is a **complete** message with the
  same `Compare:` line, not a delta: expand the group asked for and keep the other notes short.
- The page keeps at most **16,000 characters** (a message of 280 characters or fewer is kept as a snippet only). Stay well
  under that: annotate the files that matter (risky logic, public contracts, anything unproven) and list the rest in one
  plain line, rather than one note per hunk.

### Check before you send

- [ ] `## Walkthrough` is there and the ``Compare:`` line is the first line under it, with refs that resolve.
- [ ] No table anywhere in the message.
- [ ] Every note starts with ``- `path`` or ``- `path:line` — `` (a finding's label comes after the dash, never before the path), and the paths and lines come from the current files.
- [ ] Under 16,000 characters; this is the last message of the turn (no tool call after it).
- [ ] The scope matches the request (a PR is not reviewed as a working tree, or the other way round).
- [ ] You are in the repository the change belongs to (the page asks for the diff in the folder where this session
      started). If you are somewhere else, say so and stop rather than guess.

## Rules

- **Deep mode: flow, not file list.** (Quick mode is the exception: it follows the diff, file by file.) Tell it the way a
  request travels: entry point → decision → side effects → failure/rollback. A file appears where the flow reaches it,
  never as its own step.
- **Re-read the code before explaining it.** Never narrate from memory of what you wrote; the file may have changed
  (reviews, merges, formatter). Open each file and take `file:line` from the current text.
- **Read the change from git, not from memory.** `git diff --stat`, then the hunks, of exactly the range in the
  `Compare:` line; for uncommitted work `git diff --cached`, `git diff` or `git diff HEAD`, and `git status --short` for
  new files (untracked files are not in `git diff`: read their content directly).
- **Scope = the change set only.** Exclude merge commits, upstream code and the user's own local edits. Say what you
  excluded; if the head contains merge commits, say so in one line (the three-dot diff already leaves out what the base
  brought in).
- **Every step carries its "why".** Not "this calls X" but "this calls X because Y; the alternative was Z and was rejected
  because …".
- **Be candid about your own judgement calls.** Call out assumptions, ticket deviations, scope you added, and bugs you
  introduced and fixed on the way. The reviewer should not discover these themselves.
- **Verified vs reasoned.** For each claim of correctness say which command/test proves it, or that it is only reasoning
  and what would prove it (e.g. "needs Postgres").
- **Match the user's language** for prose (Thai if they write Thai); keep identifiers, paths and code in English.

## Modes

Pick the mode from the request; if it names none, use **quick** and offer deep in one plain line at the end.

| Mode | Answers | In the VAO message |
| --- | --- | --- |
| **quick** | "What does each part of this diff do?" | One note per hunk, grouped under a `###` step per area, in the order a reviewer should read them. |
| **deep** | "How does this group of code work, line by line, and is it right?" | Steps follow the flow; line-by-line notes for the groups the reviewer picks; findings pinned to their lines. |

Both modes share the Rules and the output above. Fix the change set (step 1 of the deep workflow) first in either.

### Quick mode

1. Fix the change set, then read the diff hunk by hunk from the current files, not from memory.
2. For each hunk one note: ``- `file:line-range` — what it does``, plus a short why only when the intent is not obvious
   from the code. Group the notes under `###` steps by area.
3. Mark what deserves a human eye (risky logic, a deviation from the ticket, anything unproven) with `**Risk:**` or
   `**Question:**` in the note itself, not a paragraph.
4. The scope line says how many files and hunks and what was excluded. End with one plain line offering deep mode on the
   group most worth it.

No flow map, no ledger. Keep it short; a hunk that needs more than a line is a candidate for deep mode.

### Deep mode

Follow the workflow below, but write it in the VAO format:

- The **flow map** is one or two plain sentences under the scope line (entry → decision → side effects → failure path).
- Every **chapter** is a `###` step in execution order; what happens and why are notes pinned to the lines that do it.
- The **line-by-line pass** for each group the reviewer picks (or the 1-3 riskiest): a `###` step named for the group,
  then notes of the form ``- `file:line` — what this line does — why it is there``, by the smallest run of lines that
  does one thing. Follow the data: where each variable or branch comes from and what it can be, including the empty,
  null and error cases. Boilerplate (imports, trivial getters) is one note for the lot.
- The **decisions table** becomes `**OK:**` or `**Risk:**` notes on the lines where the decision lives (who decided,
  how to change it). The **verification ledger** becomes the "how you know" part of each note, and one `**Risk:**` note
  per thing that could not be run ("not run: needs Postgres").
- Close the group with what a reader should now believe about it: a note on its last line.

## Workflow (deep mode)

1. **Fix the change set.** Use git, not memory: `git status --short`, then the diff of the range you chose above.
   Separate: feature changes / merge commits / the user's local edits / generated files. Report the count (files, +/-) in
   the scope line.
2. **Find the entry points** the change hangs off (controller, consumer, cron). Read them and follow the calls until the
   change set is exhausted. Collect `file:line` as you go.
3. **Map first.** The flow in your head (and in one plain sentence in the message) before any note.
4. **Write the steps, one per flow or decision,** in execution order. In each: what happens, why it is shaped this way and
   the rejected alternative if there was one, the edge and failure behaviour (what is rolled back, what is left, what the
   caller sees), and the tests that pin it (spec file and case names), all as notes on the lines involved.
5. **Data and contract changes together,** in one step: migrations, entity columns, API request/response shape, events,
   config. These are what other teams feel first.
6. **Suggested review order:** one plain sentence naming 5-8 files, and the 2-3 places where a human eye is most valuable.
7. **Offer to go deeper:** large changes: finish the steps, then say in one plain line which part you would zoom into
   rather than dumping every file. The follow-up is a complete message (see above).

## Style

- No restating the diff line by line, except in the line-by-line pass of deep mode, where it is the point; no praise; no
  hedging about things you verified.
- Short notes. A note that needs a paragraph is two notes, or a step.
- If something looks wrong while re-reading, say so and fix or flag it: the walkthrough is also a last self-review.
- Do not publish the walkthrough anywhere unless the user asks; it is for the person who asked for it.
