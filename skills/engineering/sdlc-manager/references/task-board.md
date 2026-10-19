# Task board integration (optional, per-project)

Part of [`sdlc-manager`](../SKILL.md). A quoted section name not found in this file, and "above"/"below" pointing at stages, refer to `SKILL.md`.

A project may track work on an external task board (Trello, Jira, Linear, GitHub Projects, ...) instead of purely in-chat. This is **opt-in per project** via `.claude/pipeline.json` — the skill stays board-agnostic; nothing here names a specific tool. If a project has no `taskBoard` config, run purely conversationally as described in the stages above — don't go looking for a board.

Config shape (`.claude/pipeline.json`):

```json
{
  "taskBoard": {
    "tool": "trello",
    "boardUrl": "https://trello.com/b/XXXXXXXX/board-name",
    "stageMap": {
      "refine": "To Refine",
      "ready": "Ready",
      "inProgress": "In Progress",
      "designReview": "Design Review",
      "review": "Review",
      "done": "Done"
    },
    "labels": {
      "epic": "<tool-specific label identifier>",
      "subTask": "<tool-specific label identifier>"
    }
  }
}
```

- `tool` names which integration to use (only whatever MCP tools/skills are actually available in the session — don't assume one is connected without checking). Also check the connection can reach the right workspace: in the `virtual-agent-office` trial the ClickUp connector was authorized for a different workspace than the user's space and had to be reconnected.
- `moveBy` (`"list"` default, or `"status"`) says how a stage transition is shown. Trello moves cards between lists; ClickUp (and Jira, Linear) keep items in one list and change their status. With `"status"`, the `stageMap` map holds status names instead of list names — same generic keys, same rule of reading from the map.
- `subTickets` (`"card"` default, or `"native-subtask"`): when the board has real subtasks and dependency links (ClickUp, Jira, Linear), create sub-tickets as native subtasks of the epic and encode `depends-on:` as native dependencies as well as in the description. The board then shows what blocks what without opening cards.
- `stageMap` (legacy name `lists`) maps the pipeline's generic stage names to this board's actual list names, since boards vary. Read from this map rather than hardcoding a list name. `designReview` is optional — only needed for projects using the design/build sub-ticket split (see stage 3's "Approval" bullet above); omit it if the project never splits design out, or if it's fine falling back to no board-visible approval step (the checklist line in the design sub-ticket's body still works as a paper trail either way).
- `labels` is optional. If present, attach `labels.epic` to the epic/tracker item and `labels.subTask` to each sub-ticket item when creating them, so epic vs. sub-ticket is visually distinguishable on the board (in addition to the naming convention below). If the board tool can't create/rename labels via the tools available in-session, don't try to work around that mid-pipeline — set up the label mapping once during the first-run bootstrap (find or create suitable labels, get the user to name them if the tool can't do it programmatically) and record the resulting identifiers in this config, same as `stageMap`. If `labels` is absent, skip label attachment entirely — it's a visual nice-to-have, not required for the pipeline to function.

**Entry points (generic, board-agnostic):**
- **A — requirement given as a prompt:** refine it in-chat, then, if `taskBoard` is configured, create it as a card/issue in the `ready` list (skip `refine` if already fully scoped — no open questions left unresolved).
- **B — requirement already exists on the board:** scan the `refine` and `ready` lists for items to pick up. An item in `refine` still needs intake/scoping before ticket breakdown; an item in `ready` can go straight to stage 2.

**Sub-ticket mapping (stage 2 — ticket breakdown), when a board is configured:** `ticket-card` Mode 3 output maps to **one card/issue per sub-ticket**, not a checklist on the parent — each sub-ticket needs to move independently through the stage lists (e.g. a UI sub-ticket can be in `review` while a backend sub-ticket is still `inProgress`), and `depends-on:` relationships need to be visible as links between real items, not buried in one item's checklist.
- The original requirement item becomes the **epic/tracker**: its description links to each sub-ticket item, and it moves to `inProgress` once the first sub-ticket starts, to `done` once every sub-ticket reaches `done`. It does not itself move through `review`.
- Each sub-ticket item starts in `ready` (already scoped by the split) and moves through `ready → inProgress → review → done` independently as its own stages 3–9 run.
- **Naming convention:** give each sub-ticket a title prefixed with the parent requirement's number and its own index, e.g. `[003.2] <sub-ticket title>` for the 2nd sub-ticket of requirement 003. The epic/tracker keeps its plain title, no prefix — this alone (before any label config) makes epic vs. sub-ticket distinguishable at a glance.
- Encode `depends-on:` in the dependent item's description (e.g. a link to the blocking item) so it's visible without relying on checklist-item state.
- **Design/build split is a special case of this same pattern** (see stage 2/3 above): the design sub-ticket's flow is `ready → inProgress → designReview → done` — it does not loop back through `inProgress` a second time. The build sub-ticket carries `depends-on: <design sub-ticket>` and follows the normal `ready → inProgress → review → done` flow, only entering `inProgress` once the design sub-ticket has reached `done`.

**Card/issue movement is a side effect of stage transitions, not a separate step:** when a sub-ticket's stage advances, move it to the matching list in the same turn — don't batch board updates for later.
