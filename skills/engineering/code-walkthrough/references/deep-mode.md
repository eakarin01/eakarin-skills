# Deep mode

Part of [`code-walkthrough`](../SKILL.md). Read this when the request asks for deep mode (deep / ละเอียด / ไล่ทีละบรรทัด), or
when the user takes up the offer to go deeper after a quick walkthrough. The Rules, the `Compare:` line, the shape of the
message and the checklist in `SKILL.md` all still apply; this file only adds what deep mode does differently.

**Flow, not file list.** Tell it the way a request travels: entry point → decision → side effects → failure/rollback. A
file appears where the flow reaches it, never as its own step.

## Writing it in the VAO format

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

## Workflow

1. **Fix the change set.** Use git, not memory: `git status --short`, then the diff of the range in the `Compare:` line.
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
   rather than dumping every file. The follow-up is a complete message (see "The last message wins" in `SKILL.md`).
