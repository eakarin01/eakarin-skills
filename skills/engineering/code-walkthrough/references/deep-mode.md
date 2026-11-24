# Deep mode

Adds to [`SKILL.md`](../SKILL.md); its format, rules and checklist still apply.

Follow the flow, not the file list: entry point → decision → side effects → failure/rollback. A file appears where the
flow reaches it.

1. **Entry points.** Find what the change hangs off (controller, consumer, cron), follow the calls until the change set
   is used up, collecting `file:line`.
2. **Flow map:** one or two plain sentences under the scope line.
3. **One `###` step per flow or decision, in execution order.** Notes on the lines that do it: what happens, why, the
   rejected alternative, edge and failure behaviour (what is rolled back, what is left, what the caller sees), and the
   specs that pin it.
4. **Data and contract changes in one step:** migrations, columns, API shapes, events, config.
5. **Line-by-line pass** for the groups the reviewer picks (else the 1-3 riskiest): a `###` step per group, notes
   ``- `file:line` — what it does — why``, by the smallest run of lines doing one thing. Follow the data, including
   empty, null and error cases; boilerplate is one note. End with a note on the last line saying what to believe about
   the group.
6. **Decisions** are `**OK:**` / `**Risk:**` notes where they live (who decided, how to change it). Each thing not run is
   a `**Risk:**` note ("not run: needs Postgres").
7. **Close** with one plain sentence naming 5-8 files to review first and the 2-3 places worth a human eye, then one
   line on what you would zoom into next.
