---
name: code-walkthrough
description: >-
  Walk a reviewer through a change (code an agent just wrote, a PR or branch pair, or staged/unstaged work) in the order it runs, ending with one VAO-format message for the virtual-agent-office review page. Trigger on /code-walkthrough, or when the user asks to be walked through or to have a change explained ("พาผม walkthrough", "อธิบาย code ที่แก้", "ช่วยเล่า flow", "change เยอะ review ไม่ไหว", "walk me through the diff"), or asks to review code, a PR, a branch pair or uncommitted work ("จะ review code ด้วย", "ช่วย review PR feat/x -> main", "review X against Y", "review ก่อน commit"). Modes: quick (default) / deep (ละเอียด, ไล่ทีละบรรทัด).
---

# Code walkthrough

The reviewer did not write this code and the change is too big to read cold. Make it readable in the order it runs,
and be honest about what you decided on their behalf.

## Output: always one VAO message

Every walkthrough or review, quick or deep, ends with **one message in the format below**. It is shown on the VAO review
page: files left, diff right, your notes under their lines. Use terminal markdown only when the user's current request
asks for it by name ("terminal format", "ไม่ต้องขึ้น VAO"), never because of a memory or handoff note; then say in one
line it will not reach the page. The page shows commit ranges and staged/unstaged/working-tree changes: a note saying
otherwise is out of date.

### `Compare:` line

Look first (`git status --short`, `git log --oneline -5`). "X -> Y" or "X against Y" means head X, base Y.

- PR or branch: `` `Y...X` `` (only X named: base = default branch, say so)
- Commits made: `` `HEAD~1...HEAD` ``, `` `abc123...HEAD` ``
- Staged only: `` `HEAD...@staged` ``. Unstaged plus untracked: `` `HEAD...@unstaged` ``
- All uncommitted (default for "changes on the working tree"): `` `HEAD...@worktree` ``
- Branch plus uncommitted: `` `origin/main...@worktree` `` (any merge base)

Refs must work in `git diff <base>...<head>`: check each with `git rev-parse --verify` (`git fetch` first if a branch is
only on the remote). Do not check anything out.

### Message shape

1. **Verdict**, plain text above the heading: ready / ready with changes / not ready, with the one or two reasons. For an
   explanation only: one line on what the change does.
2. **`## Walkthrough`**; its first line is ``Compare: `base...head` ``. One plain scope line may follow (files, +/-, what
   was excluded, what was not run). For uncommitted work add that notes follow the files as they are now and that
   `.env`-like files show no content.
3. **One `###` heading per step**, in reading order.
4. **One bullet per note:** ``- `path:start-end` — text``, ``- `path:12` — text``, or ``- `path` — text`` for a whole
   file. Head-version lines, ` — ` as separator; a long note continues on lines indented two spaces.
5. **Findings:** the label is the **first word after the dash**:
   ``- `src/seat.ts:12` — **Risk:** the fallback hides a missing rule (read the code)``.
   `**Issue:**` should change · `**Question:**` needs the author · `**Risk:**` fragile or unproven · `**OK:**` a decision
   worth confirming. The page files a note by this label and shows its icon, so:
   - A label before the path, after a description, or on the indented line is not read. Describing the line too?
     Finding first, or two notes on the same path.
   - Every finding gets a label; "low risk" or "suggest …" as plain text is lost.
   - Say how you know: read the code, ran a test, or reasoned.
6. Anything else (flow map, review order, offer to go deeper) is plain text or a bullet not starting with a code-span
   path. The page does not show it, so what must be seen goes in a note.

**The last message wins.** The page shows only the last message of your turn: make every tool call before it. A
follow-up ("zoom into X") is a complete message with the same `Compare:` line, not a delta. Max 16,000 characters
(280 or fewer is kept as a snippet only): annotate what matters, list the rest in one plain line.

**Check before sending:** `Compare:` is first under `## Walkthrough` and its refs resolve · no tables · every note starts
``- `path`` with the label right after the dash · under 16k chars, no tool call after · scope matches the request · you
are in the repo the change belongs to (otherwise say so and stop).

## Rules

- **Read from git and the current files, never memory.** `git diff --stat`, then the hunks of the `Compare:` range
  (`--cached`, plain, or `HEAD` for uncommitted; read untracked files directly). Take `file:line` from the current text.
- **Scope is the change set only.** Leave out merge commits, upstream code, the user's local edits and generated files;
  say what you left out.
- **Every step carries its why**, and the rejected alternative when there was one.
- **Be candid:** assumptions, ticket deviations, added scope, bugs you made and fixed.
- **Verified vs reasoned:** name the command or test that proves a claim, or say it is reasoning and what would prove it.
- **Match the user's language** for prose; identifiers, paths and code stay English.
- No praise, no hedging on what you verified, short notes (a paragraph is two notes). Re-reading is your last
  self-review: flag or fix what looks wrong. Publish nowhere unless asked.

## Modes

No mode named: **quick**, then offer deep in one plain line on the group most worth it.

- **Quick:** follow the diff. One note per hunk (what it does, why only when not obvious), grouped under `###` steps by
  area; risky or unproven hunks get `**Risk:**` / `**Question:**`. Scope line counts files and hunks. No flow map.
- **Deep** (ละเอียด, ไล่ทีละบรรทัด, or the user takes the offer): **read
  [`references/deep-mode.md`](references/deep-mode.md) first.**
