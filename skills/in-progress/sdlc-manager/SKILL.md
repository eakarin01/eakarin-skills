---
name: sdlc-manager
description: Orchestrate a full software development pipeline from a raw requirement through tickets, design, coding, review, test, deploy, and monitoring — dispatching each stage to a specialist subagent where one exists, and to a general-purpose agent otherwise. Reads a per-project config to run fully automated or pause for confirmation between stages; production deploy is always gated. DRAFT — only ticket breakdown (via ticket-card) and review (via code-review) currently have a real backing skill; the rest run on general-purpose by default. Trigger on /sdlc-manager or when the user hands you a feature/requirement and asks to run or plan the whole dev pipeline for it.
---

# SDLC Manager

Orchestrator skill. Takes a raw requirement, breaks it into tickets, and drives each ticket through the development pipeline by dispatching stages to specialist subagents via the `Agent` tool. Does not do the work of any stage itself — its job is sequencing, gating, and handing off.

**Status: draft, validated on two real trials** (`dormitory-management-system` on Trello + Firebase, `virtual-agent-office` on ClickUp + GitHub — see run notes at the end). Stages other than ticketing and review don't have a dedicated specialist skill/agent yet — see the stage/agent mapping table below. `general-purpose` is the accepted default for those; no need to ask permission per run. The one stage that still needs explicit human confirmation regardless of agent is production deploy — that's a property of the action, not of which agent executes it.

## When to invoke

- `/sdlc-manager`
- User describes a feature/requirement and asks to plan or run the full pipeline for it
- User asks to resume or check status of a pipeline already in progress for the current project

## Pipeline stages

1. **Requirement intake** — clarify scope with the user if the ask is underspecified. Do not proceed to ticketing on a vague requirement; ask.
2. **Ticket breakdown** — split the requirement into tickets, unless it needs no split or no ticket at all (see "Work units: with or without tickets" below). Use the `ticket-card` skill's Mode 3 (split) to produce the sub-tickets, each following its Mode 1 template. For each ticket, decide whether it needs UX/UI work (anything touching a user-facing screen/flow) and mark it — this decides whether stage 3 runs for that ticket.
   - **Design/build split**: when a ticket is UI-touching *and* is expected to involve a genuine new design decision (not pure reuse of an existing design-system pattern, not an already-approved import — see "If a project already has a Figma design" below), split it into two linked sub-tickets via the same Mode 3 `depends-on:` mechanism used for any other cross-ticket dependency: a **design sub-ticket** (stage 3 only) and a **build sub-ticket** (stages 4 onward, `depends-on:` the design sub-ticket). Skip the split entirely when there's no real design decision to make — the build ticket just runs stage 3 inline as a quick reuse-check with no separate approval step.
3. **UX/UI design** — wireframes/mockups/flow. **Conditional**: only runs for tickets flagged as UI-touching in stage 2; skip straight to stage 4 for backend-only or infra-only tickets. Its output (screens, flows, states) feeds stage 4 as a constraint on the technical approach. If the design/build split from stage 2 applies, this stage runs entirely within the design sub-ticket.
   - Before publishing any design version (including v1), take one rendered look at it (screenshot/preview) and check it loads without JS errors and its main interactions work. An unchecked v1 shipped broken interactions in the `virtual-agent-office` trial, and each revision round cost 150–190k tokens — cheap checks up front save whole rounds.
   - Questions about the underlying data model that change what the screen shows (e.g. "what is an idle agent, if agents only exist while spawned?") belong at intake, not in design review. In the `virtual-agent-office` trial they surfaced during v1 review and forced two more design rounds.
   - Must produce a real rendered artifact, not narrated text. Use the `Artifact` tool (with the `artifact-design` skill) to build an actual HTML wireframe/mockup for every new-or-changed screen and state (loading/error/empty, not just the happy path) — this is the same "don't fabricate a stage's output" principle already in Rules, applied to design instead of code/tests.
   - If `.claude/pipeline.json` has `designSystem` configured (see "Design-system integration" below), check it via `DesignSync` *before* designing anything new — reuse an existing component/pattern rather than inventing a fresh one, and only design something new for a genuine gap. After the ticket's coding stage lands, push new or changed components back into the project so it stays current for the next ticket. If no `designSystem` is configured, fall back to reading the actual shipped source (and a live screenshot via `claude-in-chrome` if it's already deployed) for whatever screens already exist, so the new screen stays visually/structurally consistent instead of guessing fresh each time.
   - **Approval, when the design/build split applies**: the design sub-ticket's own description gets a checklist line — `- [ ] Design approved — see Artifact: <url>` — plain text in the ticket body, not a board-specific mechanism, consistent with how `ticket-card` templates work across any tracker. If a task board is configured with a `designReview` list (see "Task board integration" below), move the design sub-ticket there once the Artifact is published; once the checklist line is checked, move it straight to `done` — it does not re-enter `inProgress`. The linked build sub-ticket cannot start stage 4 until the design sub-ticket reaches `done`. This two-ticket shape (rather than one ticket cycling through `inProgress` twice) is deliberate: a list a ticket revisits with a different meaning each time is genuinely ambiguous to anyone glancing at the board without opening the card, whereas a design ticket that reaches `done` and stays there is not.
4. **Technical design** — architecture/approach for each ticket, informed by the UX/UI output when stage 3 ran.
5. **Coding** — implementation.
6. **Review** — use the `code-review` skill against the resulting diff. If findings need fixing, dispatch a separate, scoped `Agent` call listing only those findings (don't reuse the original coding-stage agent's context) — then the orchestrator commits once, after the user confirms, rather than the sub-agent committing. This keeps the diff the user reviews and the diff that gets committed identical. Ask the reviewer to say whether any changed test hides a bug (with a repro if so), and file a ticket instead of widening the PR when it does. Read the code to confirm each finding before dispatching a fix: only confirmed ones go back, through `SendMessage` to the coding agent (same files, so about a tenth of a fresh agent's cost).
7. **Test** — verification (unit/integration, or manual test plan if no test agent exists yet). For any ticket with a UI, a live browser check (e.g. `claude-in-chrome` against the dev server) is part of this stage, not optional: pure, well-tested logic can still be wired wrong at runtime. In the `virtual-agent-office` trial, 77 passing unit tests covered the walker simulation, but the live check found characters never re-routing on status change — the tests had locked in the wrong behaviour. Before judging any animation or timing seen through `claude-in-chrome`, check `document.visibilityState`: a tab the extension drives can be hidden, which pauses requestAnimationFrame and makes a healthy scene look frozen (this produced a false "deadlock" report in the same trial). The same hidden tab can also drop real mouse clicks while scripts still run; if a click has no effect, trigger the control with `element.click()` through the page-script tool and say in the report that the click path itself went untested. If this is the first ticket in the project needing a given kind of test (e.g. no test framework installed yet), expect this stage to cost noticeably more than design/coding — it's paying for one-time infra setup (test runner, emulator wiring, config), not just writing test cases. Say so when reporting the stage's result, so the cost isn't a surprise. Later tickets that reuse the same infra won't carry that cost. The first live check of a UI PR is worth its cost even after a green review (003.4 PR 2: five visual or keyboard problems that no unit test or review saw), and a CSS fix made in response to a review needs a screenshot recheck, because a fix can break what it fixed (a focus ring clipped, then made near-invisible). When a stage-7 run is interrupted by a rate limit or a dropped Chrome extension, ask the agent to write results per item to a file as it goes, then resume the same agent; if the extension is down, ask the user to reconnect it.
8. **Deploy** — ships the change, per environment:
   - **Staging/UAT** — low-risk, reversible. Can run in `auto` mode like any other stage.
   - **Production** — **always a hard gate** (see below), regardless of `mode`.
   - Before deploying, check for environment/dependency staleness proactively rather than waiting for the deploy command to surface it as a warning after the fact — e.g. a runtime version nearing deprecation, an outdated core framework dependency. A pipeline run can span real time between intake and deploy; assumptions baked in at scaffold time can go stale by the time deploy actually happens.
   - Treat actionable warnings in the deploy command's own output (not just its pass/fail exit code) as part of the stage's job to resolve or explicitly report — e.g. a missing cleanup policy, a soon-to-be-decommissioned runtime. Don't just check "did it exit 0."
   - After deploy, run at least one real smoke-test call against the actually-deployed function/environment before considering the stage done — don't just trust the deploy command's exit code or emulator test results. Emulator tests never touch real cloud APIs/infra (disabled APIs, missing database instances, IAM gaps, etc. are all invisible to them), and a successful `firebase deploy` only confirms the function *deployed*, not that it works against real data. A single real call (e.g. the actual `onCall` function invoked from a real client, or `curl`/CLI equivalent) catches this class of gap immediately instead of leaving it for whichever later ticket happens to exercise the code path live first.
9. **Monitor** — post-deploy observation for regressions.

Stages 8–9 assume a deployed, running target (an app/service with environments to ship to and metrics to watch). For a repo with no such target — docs, config, a tool repo like this one — "deploy" collapses to committing/merging to the main branch, and "monitor" doesn't apply; skip it rather than forcing a fit. Detect this at intake, not partway through.

Stages 3–5 and 7–9 have no dedicated specialist skill in this repo yet. Treat them as open slots in the pipeline, not stages to skip (except stage 3, which is intentionally skippable per-ticket as described above).

## Stage → agent mapping

| Stage | Backing skill/agent today | Default when none |
|---|---|---|
| Ticket breakdown | `ticket-card` skill (Mode 3) | — |
| UX/UI design | none | `general-purpose` (conditional per ticket) |
| Technical design | none | `general-purpose` |
| Coding | none | `general-purpose` |
| Review | `code-review` skill | — |
| Test | none | `general-purpose` |
| Deploy | none | `general-purpose`, staging/UAT only auto — production always gated |
| Monitor | none | `general-purpose` |

`general-purpose` is the default backing agent for any stage without a dedicated skill — dispatch to it without stopping to ask. Report which agent handled each stage as part of the normal per-stage status update (see Rules), so the user can see where a real specialist would add value later, without that visibility requiring a stop-and-ask.

## Config: auto vs. manual mode

Read `.claude/pipeline.json` at the project root before starting a run. Example:

```json
{
  "mode": "manual",
  "stages": {
    "deploy": {
      "staging": "auto",
      "uat": "auto",
      "production": "manual"
    }
  }
}
```

- `mode` is the project-wide default: `"auto"` or `"manual"`.
- `stages` optionally overrides specific stages regardless of the project-wide mode. `deploy` is keyed by environment since staging/UAT and production carry different risk.
- If the file doesn't exist, default to `"manual"` everywhere — don't assume a project wants unattended execution, including to staging.

### Schema

Every key the skill reads. Only `mode` is required; each optional block is explained in its own section below.

| Key | Type | Default | Meaning |
|---|---|---|---|
| `mode` | `"auto"` \| `"manual"` | `"manual"` | Project-wide stage gating |
| `stages.<stage>` | `"auto"` \| `"manual"` | `mode` | Per-stage override; stage keys are `intake`, `tickets`, `uxui`, `design`, `coding`, `review`, `test`, `deploy`, `monitor` |
| `stages.deploy.<env>` | `"auto"` \| `"manual"` | `"manual"` for `production`, else `mode` | Per-environment deploy gating; `production` is gated unless explicitly `"auto"` *and* confirmed |
| `deployTarget.tool` | string | — | What "deploy" means here (`"firebase"`, `"vercel"`, `"local"`, …). `"local"` collapses stage 8 to merging to main and skips stage 9 |
| `deployTarget.note` | string | — | Free-text context for the deploy agent (e.g. why a ticket is on hold) |
| `taskBoard.tool` | string | — | Board integration to use; must match a connected MCP/skill |
| `taskBoard.boardUrl` | URL | — | Human link to the board |
| `taskBoard.workspaceId` / `spaceId` / `listId` | string | — | Board ids the tool needs (ClickUp needs all three; Trello needs none) |
| `taskBoard.moveBy` | `"list"` \| `"status"` | `"list"` | How a stage transition is shown on the board |
| `taskBoard.subTickets` | `"card"` \| `"native-subtask"` | `"card"` | Sub-ticket representation |
| `taskBoard.stageMap` | object | — | Generic stage → this board's list or status name. Required keys `refine`, `ready`, `inProgress`, `review`, `done`; optional `designReview`. Legacy name: `lists` — read it when `stageMap` is absent, and write `stageMap` when creating or editing a config |
| `taskBoard.labels.epic` / `.subTask` | string | — | Tool-specific label ids; omit to skip labelling |
| `designSystem.projectId` | string | — | DesignSync project id |
| `repos.<name>` | absolute path | — | Repos a requirement may touch, by short name. Lets the orchestrator find each checkout without asking; without it, ask once per run which repos are involved |
| `runsDir` | absolute path | single repo: `<repo>/.claude/runs`; several repos: `<parent of the repos>/.sdlc-runs` | Where run files and contracts for ticketless or multi-repo runs live (`<runsDir>/<slug>/run.md`, `contract.md`). Keep it outside every repo when more than one repo is involved |
| `git.mergeMethod` | `"merge"` \| `"squash"` \| `"rebase"` | — (ask) | How a ticket's PR lands on the main branch (`gh pr merge --merge` / `--squash` / `--rebase`). A per-project preference: ask at bootstrap instead of assuming |

Reject a config with an unknown top-level key or a `mode` outside the two values, and say which key is wrong, rather than guessing. Per-ticket or per-branch overrides are not supported; nothing in two real trials needed them.

### First-run bootstrap

If `.claude/pipeline.json` doesn't exist when `sdlc-manager` is invoked on a project, treat this as a one-time setup conversation before running any stage — ask once, covering both of the following together, not in separate follow-up passes:

1. **Mode:** project-wide `auto` or `manual`, and whether `deploy` needs per-environment overrides.
2. **Task board (optional):** whether this project tracks work on an external board. If the user names one, check which matching MCP tools/skills are actually connected in the current session (don't assume — verify) before configuring `taskBoard`. If none is connected yet, say so and offer to proceed without board integration rather than configuring a tool that isn't reachable. If the user has no board, omit `taskBoard` entirely — it's optional, not a default.

3. **Code host access (verify, don't assume):** if the pipeline will push branches and open PRs, check the CLI's auth scopes now (e.g. `gh auth status`). Pushing `.github/workflows/*` needs the `workflow` scope, and git may use a credential cached in the OS keychain rather than the CLI's token — so a refreshed CLI token can still be rejected on push. Fix both at bootstrap (e.g. `gh auth refresh -h github.com -s workflow`, and push via `git -c credential.helper= -c 'credential.helper=!gh auth git-credential' push …` or `gh auth setup-git`), not when the first push fails.
4. **Board structure the tools can't create:** some board MCPs can't create statuses or labels (ClickUp: no status creation; Trello: no label creation/renaming). List exactly what the user needs to add in the board UI, wait for it, then re-read the board to verify names match the config before writing it.

Write the resulting `.claude/pipeline.json` (see shape above and in "Task board integration" below), confirm it back to the user in one message, then proceed to stage 1. Re-running this bootstrap on a project that already has the file is unnecessary — just read it.

### Auto mode

Chain stages without stopping in between. Still:
- Dispatch each stage as its own `Agent` call — foreground when the next stage depends on this stage's output (e.g. design must finish before coding starts), background only for genuinely independent work (e.g. monitoring after deploy).
- Cross-ticket dependencies (a sub-ticket's `depends-on:` from Mode 3 splitting) apply per stage, not just once: if ticket B depends on ticket A, B's coding stage needs A's actual implementation output (e.g. the real API contract), not just A's design doc. Don't start B's coding off of A's design alone — wait for A's coding stage to finish, unless the two share an approved contract (see "Running sub-tickets in parallel (contract-first)"), in which case B codes against the contract's stubs and both integrate at the end.
- Surface each stage's result to the user as it completes; don't wait until the end of the whole pipeline to report anything.

### Manual mode

Run one stage, report the result, then stop and ask for confirmation before advancing to the next stage. Never assume approval.

### Production deploy is always gated

Staging/UAT deploys can run in `auto` mode like any other stage — treat them as reversible, low-risk. **Production deploy stays a hard confirmation point regardless of `mode`**, since it's hard-to-reverse and affects shared/real-user systems. Only skip the confirmation if `stages.deploy.production` is explicitly set to `"auto"` in the config **and** the user has previously confirmed they want that for this project. When in doubt, ask.

## Task board integration (optional, per-project)

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

## Design-system integration (optional, per-project)

A project may keep a hosted design-system project (via the `DesignSync` tool, backing claude.ai/design) as the source of truth for its UI components and foundations (colors, typography, buttons, forms, ...), instead of stage 3 re-deriving conventions from scratch on every UI ticket. This is **opt-in per project** via `.claude/pipeline.json`, same pattern as `taskBoard`. If a project has no `designSystem` config, stage 3 falls back to reading shipped source/screenshots directly (see stage 3's description above) — don't create a design-system project unprompted mid-pipeline.

Config shape (`.claude/pipeline.json`):

```json
{
  "designSystem": {
    "projectId": "<DesignSync project id>"
  }
}
```

- **Bootstrapping** (one-time, not a per-ticket cost): when a project first opts in, extract the UI already shipped (components, states, foundational tokens) into preview HTML files and push them via `DesignSync` (`create_project` → `finalize_plan` → `write_files`). Each preview's first line should be an `<!-- @dsCard group="..." -->` marker so it shows up in the Design System pane without needing `register_assets`. Group by rough category (e.g. `Foundations`, `Components`) — match the project's own natural categories rather than forcing a fixed taxonomy.
- **Every stage-3 run** (per UI-touching ticket): read the project via `list_files`/`get_file` first to see what already exists, design new screens by composing/extending those patterns, then after the ticket's coding stage lands, push whatever's new or changed back (new component, new state, a foundational token that changed) so the project stays current. Treat "check before designing, update after shipping" as the steady-state loop — bootstrapping only happens once.
- **Note real inconsistencies found while bootstrapping or updating** (e.g. a color hardcoded in one place but tokenized elsewhere, two components solving the same problem slightly differently) as commentary in the preview file itself, not just in chat — so the next stage-3 run sees it without having to rediscover it.
- **If a project already has a Figma design**, don't re-derive it into HTML previews from scratch (that's only the fallback for projects with no existing design source) — import the real Figma frames into the `designSystem` project instead, via Claude Design's Figma-import ("Start with context" → Figma) so Figma stays the actual source of truth rather than something reconstructed from code/screenshots. Ask the user, per-project, whether Figma or the synced claude.ai/design copy is the system of record going forward — this changes whether stage 3 treats the design-system project as read-only reference (re-import from Figma when it changes) or as the thing it edits directly.
- **Figma export is one-way (import only), verified 2026-08-12**: Claude Design (claude.ai/design) has no native export back to Figma (`.fig`) — checked live via the product's own Share → Export menu (PDF, Project HTML, PowerPoint, PNG, video, and a fixed list of "Connect destinations" that does not include Figma) and cross-checked with a web search, both agreeing. If a project needs Claude Design changes reflected back in Figma (e.g. to keep a read-only client-facing Figma view current), there is no automatic sync path — say so explicitly rather than assuming one exists, and the only options are manual: browser extensions like `html.to.design` to capture a Claude Design page into Figma, or exporting PNG/SVG and placing it on the Figma canvas by hand. This is a live product surface (labelled Beta) and can change — re-verify before relying on this if it's been a while.

## Running sub-tickets in parallel (contract-first)

Work is split the way a team splits it: one party writes down the **contract** (the seam between the pieces), then each ticket builds against the contract alone, so nobody waits for another's code. Integration happens once, at the end. This replaces the sequential rule under Auto mode ("wait for A's coding stage") for every pair of tickets whose seam is covered by an approved contract; tickets with no contract still follow that rule.

Applies to sub-tickets in one repo (one worktree each) and to tickets in different repos (each repo's own checkout). The steps are the same.

### Work units: with or without tickets

The unit that runs in parallel is a **work unit**: one repo (or one area of a repo) plus what must be done there. A unit does not have to be a sub-ticket. Pick the lightest shape that keeps the work trackable:

| Shape | When | Where the units live |
|---|---|---|
| **Sub-tickets** (default) | The work is large, spans days, or the team tracks it on a board | One ticket per unit, from `ticket-card` Mode 3 |
| **One ticket, units inside** | One piece of work touches several repos but needs no separate tracking per repo | One ticket whose description has a `Scope by repo` section: per repo, what to change, files or areas if known, and `produces`/`consumes` for each contract. `ticket-card` Mode 1 writes it. The ticket moves through the stage lists as one item |
| **No ticket** | The user gives a requirement in chat and wants it built, with no board entry | A run file, `<runsDir>/<slug>/run.md` (see "Where run files live"), written by the orchestrator: the requirement, the repos involved, the contract path, a units table (repo, branch, worktree, state, PR) and decisions made. It is the only state, so keep it current at every stage transition |

Rules for all three:
- **Resolve the repos at intake.** Take them from the requirement, then `repos` in `pipeline.json`; if the user named a repo not in `repos`, ask for its path once. Read each repo's entry points enough to say which units exist and which seams they share. Do not ask the user to list units; propose them and let the user correct.
- **Splitting is the orchestrator's job, but the user approves the split** in manual mode: present the units table and the seams (one line each) before writing any contract.
- Do not create a ticket just to satisfy the pipeline. With `taskBoard` configured and no ticket wanted, say so and run on the run file; offer to create the ticket at the end as a record if the user wants one.
- A single-repo requirement with no real seam is not a parallel run: use one unit and the normal stages.
- Everything below says "ticket" for brevity; read it as "unit" in the second and third shapes. Briefs, worktrees, contracts, review and the integration check work the same. The only difference is where status is written: board, ticket description, or the run file.

**Where run files live.** The run file and the contract files are the only state of a ticketless run, so every session and every worktree must find them from the same place:
- **Several repos:** `runsDir`, outside all repos (default `<parent of the repos>/.sdlc-runs/`). The contract goes in `<runsDir>/<slug>/contract.md`. Each unit's brief gives it as an absolute path. Nothing is added to any repo's tree, so there is nothing to ignore.
- **One repo:** `<repo>/.claude/runs/<slug>/` is fine. Do not edit the repo's `.gitignore` for it. Instead, never stage it: add files by name, read the staged file list before every commit, and say in the run file's header that the directory is untracked on purpose.
- The run file records the absolute path of every repo and worktree, so a session started from any directory can resume. On resume, look in `runsDir` (or the project's `.claude/runs/`) first; do not guess from the current directory.
- A contract that the user wants to keep in a repo's history (for example an API spec) is copied there by a normal commit at the end, as a deliberate step. The working copy stays in `runsDir`.

### 1. Decide whether to parallelise

Run in parallel only when all hold: the tickets are independent *given a contract* (the dependency is on an interface, not on behaviour one ticket must discover by building), the seam can be written down in a page or less, and each ticket's file set is mostly its own. If the seam keeps moving (the consumer will not know what it needs until the producer exists), stay sequential and say why. Do this at stage 2 when `ticket-card` Mode 3 emits `depends-on:`: for each dependency, ask "could a contract replace this?" and mark the ticket `contract: <name>` instead of `depends-on:` when yes.

### 2. Stage 4a: write the contract (orchestrator, before any coding)

One contract per seam, in one file, at a path every party can read (a `contracts/<epic>-<seam>.md` in the repo that owns it, or a shared path outside all repos for cross-repo work; record the path in the epic). It holds only what the parties must agree on:

- **Interface:** the exact shapes: endpoints with request/response and error codes, event names and payloads, shared types, DB columns/migrations, config keys, file formats. Real types or JSON examples, not prose.
- **Ownership:** which ticket **produces** each item and which **consume** it.
- **Behaviour rules the other side relies on:** ordering, idempotency, nullability, auth, limits, defaults.
- **Fixtures:** one or two sample payloads (happy path and a typical error) that both sides test against.
- **Version and change rule:** a version number, and that a party who needs a change stops and asks the orchestrator rather than editing its copy.

Then:
- **Get it approved** in manual mode (the user confirms the contract before fan-out; it is the cheapest place to catch a wrong split). In auto mode, a review agent with fresh eyes checks it against every ticket's text.
- **Commit the contract first** and branch every ticket from that commit, so the contract is byte-identical everywhere. If it includes code (types, a composition root), run the formatter and linter on it before fan-out: a long line in the contract commit failed `format:check` in two PRs of the trial.
- **Put the contract path and version in the epic and in each ticket**, with `contract: <path>@<version>`, produces/consumes noted. On a native-dependency board, do not add a blocking dependency between them; the contract is the link.

### 3. Fan out: build against the contract, never against each other

- One worktree (or one repo checkout) per ticket, own branch from the contract commit. Tell each agent which directory is its own, that the contract file is read-only, and not to kill processes it did not start.
- **The producer** implements the contract. **The consumer** codes against a **stub or mock that comes from the contract's fixtures** (a mock server, a fake client, a typed interface), never against the producer's unfinished branch. Each side adds a **contract test**: the producer proves its output matches the fixtures and types; the consumer proves it handles the fixtures, including the error sample.
- Dispatch all coding agents in one message (see "Dispatching a stage"). Put the contract path in every brief under "Read first", and under "Decisions already made" say that the contract is settled.
- **Contract change requests:** an agent that finds the contract wrong or incomplete stops that part, finishes what it can, and reports the exact change it needs. The orchestrator decides, bumps the version, commits, and sends the new version to **every** affected party (`SendMessage` to a continued agent, or into the brief of a new one). Never let one side patch its copy. Count change requests in the status line: more than one or two per seam means the seam was wrong; stop and re-split.
- Keep the other shared-file rules from the trial: give each ticket its own i18n key namespace, its own CSS file, and a one-line change to the app entry file. Expect conflicts anyway in `package.json`/lockfile, locale files and the app entry.

### 4. Review per ticket, then integrate once

- **Per-ticket review and unit tests** run in parallel, as usual, each against its own diff and the contract.
- **Merge order:** merge the producer first when one exists, then rebase each consumer onto main, resolve conflicts by keeping both sides, regenerate the lockfile from main's version, rerun every check before pushing. Remove each worktree before `gh pr merge --delete-branch`: git cannot delete a branch a worktree has checked out, and `gh` then skips the remote branch too. A shared type widened by one ticket can break another's fixtures; plan a separate `test(...)` commit after the rebase.
- **Integration check (the point of the whole design):** replace the stubs with the real thing and run stage 7 on a scratch integration worktree (or, across repos, with both services running locally) that applies all diffs, before the first merge. For UI that renders what another ticket makes clickable, this includes a live browser check. Stubs prove each side matches the contract; only this check proves the contract matched reality. Mismatches go back as a contract change (step 3), not as a quiet patch on one side.
- For cross-repo work, merge order across repos follows the deploy order (producer deployed before consumer calls it, or the consumer tolerant of the old shape); state that order in the epic before merging.

### 5. Board and status

With a board, each ticket moves independently. Without one, update the run file's units table instead. 
Each ticket still moves through its own stages independently. Add `contract-ready` to the epic when step 2 is approved, and `integrated` when the integration check passes; the epic moves to `done` only after that check, not when the last ticket merges. Report in the status line: contract version, tickets running, change requests so far.

### Cost note

Not measured yet. Expect the contract stage to cost one design-sized agent round or an inline orchestrator pass, and each fan-out ticket to cost what it would alone, plus a stub and a contract test each; the saving is wall-clock time, not tokens. Record real numbers in the first run notes.

## Resuming a pipeline

For a ticketless run, start from `<runsDir>/<slug>/run.md` (or the single repo's `.claude/runs/<slug>/run.md`) instead of an epic card, then verify each unit against git (branches, worktrees, PRs) exactly as in step 3 below.

When the user asks to resume or check status of a pipeline already in progress (see "When to invoke"), and a task board is configured, don't start a fresh breakdown — reconstruct where things actually are:

1. **Read the epic/tracker card.** Its description already links every sub-ticket that's been split off so far (see "Sub-ticket mapping" above). If some obviously-needed sub-tickets aren't listed yet, the ticket-breakdown stage is incomplete — offer to continue it before touching execution stages.
2. **For each linked sub-ticket, read its current list.** The list gives a coarse stage bucket (e.g. `inProgress` covers stages 3–5, `review` covers 6–7, `done` covers 8–9) — not the exact stage, since one list spans multiple pipeline stages.
3. **Pin down the exact stage from real evidence, not the list alone.** Check the actual repo: commits/PRs referencing the sub-ticket (e.g. `git log --grep=<ticket-id-or-Trello-URL>`), whether design/coding artifacts exist, whether tests exist and pass, whether it's been deployed. The list position is a hint for where to start looking, not the source of truth — this follows the same "never fabricate, verify against the real thing" rule as any other stage. If the list and the real evidence disagree (e.g. card says `review` but no commit exists yet), trust the evidence and correct the card's list to match.
4. **Resume from the next stage after the last one with real, verified output**, respecting `depends-on:` — don't resume a sub-ticket whose blocking dependency isn't actually finished, even if its card looks further along.
5. Report what was found per sub-ticket (list vs. actual state, any corrections made) before resuming work, so the user can catch a wrong reconstruction before more stages run on top of it.

## Dispatching a stage: when to spawn, and how to brief

A subagent starts with no context and re-reads what it needs, so every spawn costs tokens the orchestrator would not have spent doing the work itself. Measured on `virtual-agent-office`: about 1.1M subagent tokens for three mid-size PRs (design ~180k, coding 90–340k each, review 65–110k each, a fix pass 55–170k, a live browser check ~115k). Spawn when it pays, not by default.

**Do the stage inline (no spawn) when** the change is small and already decided: roughly three files or fewer, no new design decision, and the orchestrator already holds the needed context. Design-only reuse checks, one-line fixes from a review finding, and README/config edits belong here.

**Spawn when** at least one of these holds: two or more independent tickets can run in parallel; the stage is read-heavy exploration whose raw output would bloat the orchestrator's context; a review or live check benefits from fresh eyes; or the coding stage is large. State in the status line why a spawn was worth it.

**Every brief must let an agent with zero context finish without asking.** Put these in the prompt, in this order, and keep each to a line or two:

1. **Goal and ticket**: ticket id/link and the outcome wanted, in one or two sentences.
2. **Scope**: what to change, and what not to touch (other worktrees, files owned by a sibling PR, the main checkout).
3. **Where**: the absolute path of its worktree and branch. Say "use absolute paths, or `cd` inside every Bash call; your shell cwd is another directory". Subagents do not inherit the orchestrator's cwd, and a skill that reviews "the current diff" then reviews the wrong repo (this cost a full review round). For reviews, hand over a patch file (`git add -N . && git diff > file`, then `git reset -q`) and tell the agent not to invoke cwd-based skills.
4. **Read first**: the exact files/sections that hold the spec (design doc path and section numbers, ticket text). Point to files and line ranges; do not paste whole documents and do not say "explore the repo". Say which claims to verify against real code and to report disagreements.
5. **Decisions already made**: a short list, so the agent does not reopen them (merge method, rejected options, approved limits).
6. **Constraints**: runtime version (e.g. `nvm use 24`), no commit/push, no new dependencies, comment density and naming like the surrounding code, never kill processes it did not start.
7. **How to verify**: the exact commands, and how to trust them (JSON reporter over text summary; count test files against disk; write a failing test first where feasible).
8. **Report format**: ask for a short fixed-field report (files changed, test counts, deviations from the design with reasons, what needs a live check, decisions for the orchestrator). No code dumps. Put long findings in a file and return a summary of about ten lines.

**Keep the cost down:**
- Pass the previous stage's output as a file path plus a short summary of its decisions, not as pasted text.
- Give a fix agent only the confirmed findings, each with file:line and the intended fix. Read the code yourself first and drop findings you cannot confirm.
- Continue a related agent with `SendMessage` (it keeps its context) instead of starting a fresh one when the follow-up touches the same files.
- Read-only stages (design, review) must not edit files; say so, so they do not spend tokens on scratch work.
- Launch independent agents in one message so they run in parallel, and do not poll them; wait for their completion notices.

## Reusing agents and keeping context small

Every continued agent re-reads its whole context on each turn, so a long-lived agent gets more expensive per follow-up as it grows, while a fresh agent pays once to re-read what it needs. Decide per follow-up; do not always reuse and do not always restart. Numbers from the `virtual-agent-office` trial (the agents' own token counters, cumulative):

| Follow-up | Cost of the follow-up |
|---|---|
| Scoped code fixes on a coding agent (5–8 confirmed findings) | +8k to +13k |
| Design revision on a design agent that renders and screenshots (three rounds) | +73k, +31k, +30k |
| A fresh review agent reading a patch and the code | 79k to 113k |
| A fresh coding agent for a PR | 80k to 150k |

**Continue the same agent (`SendMessage`) when** the follow-up touches the files it already read, its context is still modest (a coding agent under roughly 100–150k counted tokens was cheap to continue; no upper limit has been measured), and the work is fixes, not new exploration. A scoped fix pass was about ten times cheaper than a fresh agent.

**Start a fresh agent with a file-based brief when** the follow-up is unrelated to what the agent read, the agent's counter is already large (a design agent at 280k+ cost 30–70k even for small edits, because each round re-rendered and re-screenshotted), or the stage should have fresh eyes (review, live check). The heuristic threshold above is unvalidated: when you have the numbers for a fresh agent doing the same follow-up, record both and adjust.

**Make a handoff file the unit of reuse, not the agent's memory.** Ask every stage agent to end with a short report (about ten lines) and to write long output (a design doc, review findings, a decision list) to a named file. The next stage gets the path, and a continued agent is optional rather than required. This is what let a PR B agent be replaced by a review agent and a fix agent without losing anything.

**Orchestrator habits that kept its own context down:**
- Do not paste tool output into prompts; pass file paths. Read design docs and review files by section, or not at all, when an agent will act on them.
- Updating a ClickUp description means sending the whole description again (about 2.5k tokens each time; six round trips on one design ticket). Put progress in a single description edit at the end of a stage and use a comment for interim notes.
- Keep agent reports short and fixed-field (see the brief checklist). Reports longer than about twenty lines go to a file.
- Route read-only, mechanical stages (a live check against a scripted scenario, a patch review) to a cheaper model with the `Agent` tool's `model` setting (`"sonnet"`). Tried on five live checks in epic 003 (003.2a, 003.2b, 003.4b, 003.5, 003.6): the reports were as useful as before and cost about the same tokens (110k–210k each), so the saving is in price per token, not in count. Keep the brief as explicit as for any model.

## Handing a pipeline over between sessions

The orchestrator's own context also grows (many ticket texts, agent reports, tool output). Hand off at a **ticket boundary** (a PR merged and the board updated), never in the middle of a stage, and before the context is so large that every turn is expensive. No threshold has been measured; use the client's context indicator and err on the early side. A new session also picks up changes to this skill, which is another reason to hand off after editing it.

The handoff doc (one file, written by the orchestrator; the `handoff` skill can produce the shell) must hold, in this order:
1. **Where things stand:** project path and repo, board ids and links, `pipeline.json` summary, and a ticket table (state, link, one note) for the epic in progress and the next ones.
2. **Resume point:** the single next step, with the options and a recommendation.
3. **Decisions not written elsewhere:** answers the user gave in chat, with dates. Anything already on the board or in a design doc is a link, not a copy.
4. **Working rules for this project:** runtime version, push and auth commands, commit format and who confirms commits, ports the user owns, subagent pitfalls, tests and how to trust their output.
5. **State to verify:** unpushed commits, open PRs, worktrees, branches pending deletion, services started.
6. **Things the user will look at themselves** and have not been verified by any stage.
7. **Reference files** outside the repo (design docs, patches) with paths.
8. **Cost ledger:** tokens per stage so far, to fold into the run notes when the epic closes.
9. **Authority granted in this session** (e.g. "auto commit", "auto merge") and its scope. Say plainly that it expires with the session; the new session must ask again.

The new session must **verify, not trust**: re-read this skill, run `git worktree list`, `git status` and `git log` in the project, read the epic's board state, and compare each with the doc before acting. If they disagree, trust the evidence and tell the user (see "Resuming a pipeline"). Three handoffs in the trial (2026-09-30, 2026-10-01 and 2026-10-02) resumed this way without a correction. The 2026-10-02 session also found the next ticket's text out of date against four shipped tickets and rewrote it before any design: read the ticket against what shipped, not only against the board state.

**Draft, 2026-10-02: when to hand off, and what is known.** The owner's working rule is to open a new session at a ticket boundary once the orchestrator's own context is about **300K tokens or more**. It is a rule of thumb from use, not a measured threshold. What outside sources say (checked 2026-10-02; none gives a handoff threshold):
- Per-turn cost grows with context because every turn re-reads it. One measured multi-agent run ([touchstone issue #11](https://github.com/dimoschi/touchstone/issues/11), single run, model not stated): 85% of cost was re-read context, phase-to-phase handoffs were 0.6%, each agent's context grew from about 13–28k to 105–119k tokens over 90–142 turns, and the last 30% of turns took 42–46% of cache reads.
- One anecdote ([architecture-kit issue #85](https://github.com/shawrylk/architecture-kit/issues/85), one session, no method given): a fresh agent for a two-line follow-up cost 45k tokens, whereas the resumed agent would have re-sent 199k on every call. It proposes fixed text first and task text last in dispatch prompts, and passing the brief as a file path.
- [Anthropic, effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): compaction (summarise and reinitiate), notes kept outside the context, and sub-agents that return a 1,000–2,000 token summary; recall gets worse as context grows ("context rot", also [Chroma's study](https://trychroma.com/research/context-rot)); no thresholds or costs given. [Anthropic's multi-agent research post](https://www.anthropic.com/engineering/multi-agent-research-system): multi-agent runs use about 15 times the tokens of a chat, and token use explains most of the quality difference, so cutting tokens is not free.
- Not applicable: [The Handoff Tax](https://arxiv.org/pdf/2608.24358) is about continuing another model's trajectory, not about restarting the same model from a file.

Open questions, to measure on a real run before turning any of this into a rule: the context size where a fresh session beats continuing (the 300K figure above, and the 100–150k for a coding agent under "Reusing agents"), how much of the re-read is served from the prompt cache for a subagent in this harness, and whether the fixed-prefix brief shape helps. Until then: hand off at ticket boundaries, earlier rather than later, and when the client's context indicator passes the owner's 300K.

## Rules

- Never fabricate a stage's output (e.g. don't invent a design doc or test results) — dispatch to the real subagent/skill or ask the user to provide it.
- If a requirement is ambiguous, resolve it at intake, not mid-pipeline.
- Start every stage dispatch's `Agent` description with a stage tag, `[stage:<key>]`, using the stage keys from the config schema (`tickets`, `uxui`, `design`, `coding`, `review`, `test`, `deploy`, `monitor`). The `virtual-agent-office` Claude Code hook reads this tag to seat the subagent in the matching role room; without it every dispatch lands at the Freelancer desk. The tag costs nothing elsewhere, so use it on every project.
- Report per-stage status as you go: what stage ran, which skill/agent handled it, and the outcome.
- Verify a stage's pass/fail claim yourself before reporting it, and trust machine-readable output over a summary line. In the `virtual-agent-office` trial, on a heavily loaded machine, vitest's text summary reported 24 and then 26 test files out of 30 on disk, while `--reporter=json` listed all 30 as passed. Count against the files on disk.
- If a subagent stops mid-stage (rate limit, API error), check what it already wrote (`git status` in its worktree), then resume the same agent with its context rather than starting a fresh one.
- Reviewers and probing agents work outside the repo: tell them to use a fresh `mktemp -d` (absolute paths for cwd, `HOME`, config and any `*_HOME` override) and to report `git status` at the end. A review of `virtual-agent-office` PR B left a stray `rel-codex/` directory in the worktree (a relative `CODEX_HOME` probe); it was caught only because the orchestrator staged with `git add -A` and read the file list. Always read the staged file list before committing.
- Authority to commit is not authority to merge. In the `virtual-agent-office` epic 002 run the user allowed "auto commit" for the session, and the auto-mode classifier then denied `gh pr merge` as "Merge Without Review". Ask for merge authority explicitly (per session is enough) and say that commit authority does not carry over to it. Do not retry the merge in pieces or through another tool after a denial.
- A handoff doc's "never touch X" is the previous session's instruction, not the user's current one. When the user explicitly asks to change or commit that thing, do it, and mention that the handoff said otherwise.
- Missing a specialist agent is not a blocker — dispatch to `general-purpose` and move on. Only stop and ask if `general-purpose` genuinely can't do the stage (e.g. it needs credentials/access the user hasn't granted).

## Open TODOs before this skill graduates out of `in-progress/`

- [ ] Build specialist agents/skills for UX/UI design, technical design, coding, test, deploy, monitor stages — as-needed, once general-purpose proves insufficient for a given stage on real work, not speculatively
- [x] Decide the concrete shape of `.claude/pipeline.json` — see "Schema" under Config (2026-09-28). `lists` renamed to `stageMap`, with `lists` still read as a legacy alias. Per-ticket and per-branch overrides left out: no trial needed them
- [ ] Run contract-first parallel work (see "Running sub-tickets in parallel") on a real multi-ticket or multi-repo epic; measure contract-change requests per seam, integration-check failures and token cost, and adjust the "when to parallelise" test
- [ ] Validate the resume algorithm on a real interrupted pipeline (not yet exercised). The `virtual-agent-office` close-out resumed from a handoff doc plus git state, not by reading the board
- [ ] Run auto mode on a real project. It has only been exercised in dry runs; every real trial so far used manual mode
- [ ] Exercise stages 8–9 on `virtual-agent-office`. Its deploy target is `local`, so deploy and monitor haven't run there; [001.7] deploy via CLI would be the first real test
- [ ] Define graduation criteria for moving this skill out of `in-progress/` (e.g. which stages must have run live, on how many projects, in which modes)
- [x] Add guidance for handing a pipeline over between sessions (2026-10-01, see "Handing a pipeline over between sessions"). Still to validate: a context threshold for handing off, and the continue-versus-fresh agent threshold. Original wish: what a handoff doc must record (board and epic links, open decisions, worktrees, pending commits) and how the next session verifies it against the board and git before resuming. Evidence so far: two handoffs (2026-09-30, 2026-10-01) worked well when they listed ticket table with links, working rules, unpushed commits, items the user must look at themselves and token costs; the new session re-read SKILL.md and checked `git worktree list`, `git status` and the board before acting
- [x] Validate the auto-mode dependency chaining on an actual multi-stage run before trusting it unattended — see dry run notes below (2026-08-11)
- [x] Validate manual-mode stop-and-confirm behavior and real `.claude/pipeline.json` config parsing — see dry run notes below (2026-08-11)
- [x] Design task board integration once a real trial actually needed one — see "Task board integration" section above (2026-08-12, `dormitory-management-system` trial)
- [x] Run a full 9-stage pipeline end to end on a real ticket, not just a dry run — see Run 5 below (2026-08-12, `dormitory-management-system` trial)

## Dry run notes (2026-08-11)

**Run 1 — auto mode, happy path.** Traced a fabricated 2-ticket requirement ("avatar upload") through the full 9 stages: ticketing (real `ticket-card` Mode 3 call), UX/UI (conditional skip on the backend ticket, ran on the UI ticket), technical design, coding, review (real `code-review` reference), test, staged deploy, production-deploy gate, monitor.

**Finding, already folded into this file:** cross-ticket `depends-on:` isn't a one-time design-stage check — it recurs at the coding stage too (a downstream ticket's coding needs the upstream ticket's actual implementation output, not just its design). See the note under Auto mode.

**Run 2 — manual mode.** Traced ticket #1 with `{"mode": "manual"}`: every stage transition (design → coding → review → test → deploy staging → deploy production → monitor) stopped for confirmation before advancing. No branch found where two stages could chain silently — manual mode needs no separate dispatch logic, just a different stopping condition than auto. No fix needed.

**Run 3 — real config edge case.** Config `{"mode": "auto", "stages": {"deploy": {"staging": "auto", "uat": "auto"}}}` (production deliberately omitted, everything else auto): production deploy still required confirmation, per the "only skip if `stages.deploy.production` is explicitly `\"auto\"`" rule — the omission-defaults-to-gated behavior held up as intended. No fix needed.

**Run 4 — live run (2026-08-12), real `Agent` dispatches.** Task: add missing `README.md` for the `skills/personal/` and `skills/in-progress/` buckets in this repo. Ran for real: ticket breakdown (`ticket-card` Mode 1), technical design (`general-purpose` `Agent` call, read-only), coding (`general-purpose` `Agent` call, wrote the files), review (`code-review` skill, background).

**Findings:**
- Design and coding stages worked correctly on real dispatches. The coding-stage agent independently caught that the design stage's `sdlc-manager` description was stale relative to the actual frontmatter and re-verified against the real file rather than trusting the handoff — a good sign for the "don't fabricate, verify against the real thing" rule already in this file.
- `code-review` caught a real issue (missing trailing newline on both new files, inconsistent with every other bucket README in the repo) — fixed directly rather than dispatching another agent, since it was a one-line fix.
- **Deploy and Monitor don't map onto every target.** This repo has no staging/production/running service — "deploy" is just a commit to `main`, "monitor" doesn't apply. Folded a note into stage 8–9's description above: detect at intake whether the target is a deployed service or not, and collapse/skip those stages accordingly rather than forcing the template to fit.

This closes out the "no live run yet" gap from Run 1–3. The stage-dispatch mechanics (real `Agent` calls, real skill calls, background review) all worked as designed.

## Live run notes (2026-08-12) — first full 9-stage pipeline

**Run 5 — `dormitory-management-system` trial, ticket 001.1 (Room data model & CRUD API), manual mode, Trello task board configured.** First time every stage ran for real against an actual product codebase with a real deploy target, not this skills repo. Full chain: intake → `ticket-card` Mode 3 split into 4 sub-tickets (with the epic/sub-ticket Trello structure this file now documents) → technical design (`general-purpose`, read-only) → coding (`general-purpose`, wrote real Cloud Functions) → `code-review` (background, found 4 real fixable issues + 3 correctly-expected-and-ignored auth gaps) → fix dispatch (separate scoped `Agent` call) → orchestrator commit → test (`general-purpose`, had to bootstrap jest/ts-jest/Firestore-emulator wiring from scratch since none existed — 12/12 passing) → deploy (staging/production collapsed to the same single real Firebase project, since the trial has no separate staging project; hit a real Blaze-plan billing gate, stopped and asked rather than working around it) → monitor (skipped — no live traffic yet for a backend-only ticket with no frontend caller).

**Findings, already folded into this file:**
- Review-fix-commit as a named pattern (stage 6 description): fix dispatches should be a separate `Agent` call scoped to just the findings, and the orchestrator — not the sub-agent — commits once after user confirmation, so the reviewed diff and the committed diff are provably the same thing.
- Test-stage cost asymmetry when bootstrapping infra (stage 7 description): the first ticket needing a test framework pays a one-time setup cost far above normal per-stage cost (this run: ~78k tokens / 72 tool calls / ~10.5 min for test stage vs. ~27k tokens / 1 call for design, ~31k tokens / 12 calls for coding) — say so up front so it's not read as the stage misbehaving.
- Deploy-time environment staleness (stage 8 description): `functions/package.json`'s `engines.node: "20"` was set at project scaffold time, weeks before this pipeline run; by deploy time Node 20 had already been deprecated for 3+ months. Only surfaced via the deploy command's own warning output, not proactively. Added guidance to check for this kind of drift before deploying, and to treat actionable deploy-output warnings (e.g. a missing Artifact Registry cleanup policy) as part of the stage's job, not an aside.
- Real, hard-to-reverse blocker handled correctly without being told how: Cloud Functions deploy requires the Blaze billing plan; the project was still on the free Spark plan. Stopped, explained exactly what was needed and linked the upgrade page, did not attempt any workaround (e.g. silently switching to a different deploy target) — this is the "ask when a stage genuinely can't proceed" rule from Rules working as intended on a real billing/access gate, not just a hypothetical.
- Task-board label-marker idea validated end-to-end, including its real limitation: the connected Trello MCP tools can attach/detach existing labels but cannot create or rename them — worked around by using the board's existing unnamed default-color labels and asking the user to name them once via the Trello UI. Folded into the `labels` config guidance above: set this up once during first-run bootstrap, record the resulting identifiers, don't improvise mid-pipeline if the tool can't do it programmatically.
- Single-environment reality: many small/trial projects won't have separate staging and production Firebase/cloud projects. When that's the case, "staging/UAT" and "production" in stage 8 point at the same real infrastructure — the production hard-gate rule still applies (this run asked before deploying, since `pipeline.json` had `production: "manual"`), but call this out explicitly if a project's config implies environments that don't actually exist as separate infra, rather than assuming a staging sandbox is always available.

## Live run notes (2026-08-12) — `dormitory-management-system` trial, epic 001 completion

**Runs 6–8 — sub-tickets `[001.2]`, `[001.3]`, `[001.4]` of the same epic, manual mode, Trello task board.** Completed the "Room & Resident Management (Core CRUD)" epic started in Run 5. All 4 sub-tickets and the epic card reached Trello's Done list.

**Finding, now folded into stage 8 above:** `[001.3]`'s deploy stage exited clean and its emulator tests passed, but the deployed function still failed in production — the Cloud Firestore API was disabled at the GCP project level, and separately, no Firestore database instance existed in the project at all. Neither gap was visible to emulator-based tests (they never touch real GCP APIs) or to the deploy command's own exit code (it only confirms the function artifact deployed, not that it works against real data). The gap was only caught two tickets late, via live `claude-in-chrome` browser testing during `[001.3]`'s own manual verification — not by any pipeline stage designed to catch it. Root-caused and added the smoke-test requirement to stage 8's description so the next project catches this at deploy time instead.
- Both infra gaps were fixed with the user's explicit confirmation before acting (enabled the API, created the default Firestore database) — consistent with "ask before hard-to-reverse/production-affecting actions," even though this was arguably closer to a deploy-stage bug fix than a new feature.

**Other pattern reinforced, no new fix needed:** `claude-in-chrome` live testing caught a second real bug this run that no earlier stage caught — `[001.4]`'s room-picker "Refresh & Retry" button was silently closing instead of staying open with fresh data after a real concurrent-assignment conflict. Confirms the Run 5-era decision to lean on live browser testing for frontend tickets rather than trusting type-check/unit-test-only verification; no skill change needed here beyond what's already documented, just evidence it keeps paying off.

## Live run notes (2026-09-24/25) — `virtual-agent-office` trial, epic 001

**Setup.** New project, manual mode, ClickUp task board (status-based, native subtasks and dependencies), private GitHub repo, local run only (the deploy ticket is on hold). The project is a pixel-art office that shows AI agents as employees, built with React, PixiJS and Zustand on mock data. Epic 001 was split into 9 sub-tickets. Seven of them (001.1–001.5, 001.8, plus the design ticket) went through the full design → code → review → test → PR → merge loop. 001.2/001.8 and 001.4/001.5 ran in parallel.

**Findings, already folded into this file:**
- **Status-based boards:** ClickUp moves stages by status, has native subtasks and dependency links, and its MCP can't create statuses. This led to `moveBy`, `subTickets` and bootstrap step 4.
- **Wrong workspace:** the ClickUp connector first reached the wrong workspace. See the note on `tool`.
- **Push credentials:** pushing a workflow file failed for lack of `workflow` scope. After the token was refreshed, it still failed because git used the old keychain credential. This led to bootstrap step 3.
- **Design cost:** three rounds cost about 460k tokens (v1 ~120k, v2 ~187k, v3 ~150k). v1 wasn't checked before publishing, and a data-model question (idle vs. spawned-only agents) came up at design review. This led to the two stage 3 bullets.
- **Parallel tickets:** both pairs needed worktrees, identical shared glue and area-split i18n keys. The second PR of each pair conflicted exactly where predicted. This led to "Running sub-tickets in parallel".
- **Live browser test:** in the combined browser test (001.4 + 001.5), characters never re-routed when their status changed. Unit tests passed because an existing test locked in the wrong behaviour. This led to the stage 7 note.
- **Rate limit:** a coding agent hit a rate limit halfway through. Resuming it with its context worked cleanly. This led to the rule in Rules.

**Other patterns reinforced, no new fix needed:**
- **Review:** `code-review` found real bugs in 4 of 5 code tickets, including a StrictMode subscription loss and a render-crash. Scoped fix agents with fail-then-pass tests handled all of them.
- **Commit pause:** the user's `commit` skill always pauses between staging and committing, so every commit needs its own confirmation even after a stage was approved. Stage, show and ask once per commit, and batch several tickets into one question where possible.
- **Stage cost:** the most expensive stages were UX/UI design (~120–190k per round) and the Pixi coding stage (~220k). Technical design stages cost 45–85k, and fix agents 55–130k.

**Left open:** walker polish follow-up ([001.9]), the list view ([001.6]) and deploy via CLI ([001.7], on hold).

## Live run notes (2026-09-26/28) — `virtual-agent-office` trial, epic 001 close-out

**Runs — [001.6], [001.9] (resumed from a handoff), then two new stories [001.10] (Artifact pixel sprites as the default theme, Kenney switchable) and [001.11] (the theme covers the whole office).** Manual mode, ClickUp, one worktree per ticket, PRs #6–#9. Stage 3 was skipped for 001.10/001.11 because the approved design v3 already existed; stage 4 read the design's own drawing code as its spec.

**Findings, already folded into this file:**
- **Schema:** the config grew `deployTarget`, board ids, `moveBy` and `subTickets` without a written schema. See "Schema"; `lists` is now `stageMap`.
- **Hidden tab clicks:** the Chrome tab stayed `hidden`, and in one run real clicks never reached the page. See the stage 7 note.
- **Unreliable test summary:** under heavy machine load the vitest text summary under-counted test files. See Rules.

**Other patterns reinforced, no new fix needed:**
- **Port, don't redraw:** for "make it look like the design", stage 5 copied the design's drawing functions verbatim into an asset generator and pixel-diffed the output against the design (105/105 frames identical). For a refactor that must not change the look, it recorded a baseline of the old draw calls *before* touching code and proved 0 differing pixels after. Both are cheap, strong evidence; ask for them in the stage 5 prompt.
- **Reviewer claims need checking:** one of two `code-review` findings on 001.11 was wrong (it said a transparent canvas would show the page background; the container already had the dark background). Read the code before dispatching a fix.
- **Stage cost:** 001.10 ≈ 420k tokens (design 108k, coding 186k, review 82k, fix 46k); 001.11 ≈ 500k (design 120k, coding 250k, review 83k, fix 47k). Coding stays the most expensive stage when it includes asset generation.

## Live run notes (2026-09-28/30) — `virtual-agent-office` trial, epic 002

**Runs — [002.1] schema + collector, [002.2] Claude Code hook adapter, [002.3] design v4, [002.4] RealFeed (PR A) and Lead desk (PR B), plus a new epic 003 split.** Manual mode, ClickUp, PRs #10–#13. The session resumed from a handoff doc; the board and git agreed, so nothing needed correcting.

**Findings, already folded into this file:**
- **Stage tags:** the project's own Claude Code hook seats a subagent by `[stage:<key>]` in the Agent `description` (not the prompt). See Rules.
- **Worktree before merge:** `gh pr merge --delete-branch` couldn't delete a branch still checked out in a worktree, and then skipped the remote branch too. See "Running sub-tickets in parallel".
- **Merge method is a preference:** the user switched from rebase to merge commits mid-epic ("rebased history looks unfinished"). See `git.mergeMethod`.

**Other patterns reinforced or new, no skill change needed yet:**
- **Shared contract file worked:** the orchestrator wrote `stages.ts` once into both worktrees (002.2 and 002.4 in parallel). Only README conflicted. But the second PR also had to regenerate the first PR's replay fixtures, because it widened a shared type; plan a separate `test(...)` commit for that after the rebase.
- **Stage 4 can overturn the ticket:** the 002.2 design agent ran real `claude` sessions and found that `http` hooks can't meet two acceptance criteria. It switched to command hooks. Update the ticket text at approval so the board matches what is built.
- **Stage 7 must look, not only read:** the first live check of 002.4 PR A read the drawer through `innerText` and missed that the Lead list had no CSS at all. PR B's coding agent found it from a screenshot. Screenshot every new panel and state.
- **Real clicks can work:** in PR B's check a real mouse click on the canvas worked while `visibilityState` was `hidden`. Try the real click first and fall back to `element.click()`.
- **Environment hiccups:**
  - Auto-mode safety-classifier outages happened twice. Stop after 3–4 consecutive "no verdict" errors and tell the user; the turn ends at 10.
  - After `/login`, the Chrome extension lost its page connection until the user fixed it.
  - One `code-review` fork stalled (600 s watchdog); a rerun worked.
  - The ClickUp connector saw two workspaces, so every call needed `workspace_id`.
- **Scope changes at design review:** the user dropped per-vendor visuals entirely and added a Lead desk and a learned-roles model, which became epic 003. Cheap questions at intake ("what should differ per vendor?", "who defines roles for someone else's agents?") would have saved a design round.
- **Stage cost:**
  - 002.1 ≈ 440k tokens: design 105k, coding 162k, review 88k, fixes 106k + 68k.
  - 002.3 ≈ 390k: two rounds, 309k + 78k.
  - 002.2 ≈ 670k: design 183k, coding 274k, review 97k + a stalled run, fix 113k.
  - 002.4 PR A ≈ 740k: design 261k, coding 337k, review 110k, fixes 89k + 63k.
  - 002.4 PR B ≈ 520k: coding 257k, review 95k, fix 169k.

## Live run notes (2026-09-30) — `virtual-agent-office` trial, epic 002 round 2

**Runs — [002.5] liveness/dedupe/idle/capability flags and [002.7] connection troubleshooting (raised from user feedback after first real use), one tech design, three parallel PRs (#14 collector, #15 app idle, #16 connection UI), manual mode.**

**Findings, already folded into this file ("Dispatching a stage"):**
- **Review ran on the wrong repo:** the `code-review` skill reviews the cwd diff, and a subagent's cwd is not the worktree. The first review round reported findings for a different repository. Hand reviewers a patch file and absolute paths.
- **Decide spawning by size:** the user saw the Lead handle nearly everything itself in real use; forcing spawns to fill the office would only add tokens. Spawn for parallelism, read-heavy work and fresh-eyes review.

**Other patterns, no skill change needed yet:**
- **A contract commit first worked:** the orchestrator wrote the shared type change once (`RunStatus += 'idle'`, capabilities type) and branched three worktrees from it. Keep the contract commit's formatting clean: its one over-long line failed `format:check` in two PRs. Run prettier on it before the branches fan out.
- **Merge order follows CI:** with a shared commit failing CI in two PRs, merge the PR that carries the fix first, then merge main into the others. No force-push needed.
- **Reviewers earned their cost:** the collector review found a late-`session.end` hole (reaped run not remembered), unclamped producer timestamps and a dropped stage on resume; the connection-UI review found a port-change button that did nothing with a `#hash` in the URL. The design doc was wrong in two places (8 of 13 fixtures change, not 12; stage carry-over only within one main generation), so ask design agents to verify claims against code.
- **Stage cost:** design 181k; coding 164k (collector), 93k (app idle), 129k (connection UI); reviews 107k, 66k, 92k plus ~36k wasted; fix passes 68k and 59k; live browser check 115k. Total about 1.1M.

## Live run notes (2026-09-30/10-01) — `virtual-agent-office` trial, epic 002 round 3

**Runs — [001.12] landscape layout (PR #19), [001.14] back-link style (#17), [001.15] zoom ladder (#20), [002.6] Codex adapter in three PRs (A #18, B #22, C #23), plus [001.16] "Fit prefers Campus" (#21), raised by the user while the agents ran.** Manual mode until the user granted per-session commit and merge authority, ClickUp, merge commits.

**Findings, already folded into this file (Rules):**
- **Probe leftovers:** a reviewer's probe left `rel-codex/` in the worktree. See Rules.
- **Commit is not merge:** the classifier denied a merge after "auto commit" was granted. See Rules.
- **Stale handoff instructions:** see Rules.

**Other patterns, no skill change needed yet:**
- **Pin behaviour before a refactor:** for PR B (refactor of the shared Claude installer plus a new vendor) the brief asked the coding agent to write a byte-identity test over the existing output *before* touching code and make it pass on the old code. The reviewer then ran that test against a `git archive` of main to prove its hashes were main's output. Cheap, strong evidence; ask for it whenever a refactor must stay behaviour-neutral.
- **Keep wrappers instead of editing old tests:** the coding agent kept the old `planInstall(text, spec)` signatures as thin wrappers over the new dialect-based functions, so every existing test stayed untouched. Good default for "behaviour-neutral" briefs.
- **Reviews of installers earned their cost:** PR B's review found a symlinked `hooks.json` being replaced by a regular file (a pre-existing hole in the shared `writeAtomic`, fixed for both vendors) and a relative `CODEX_HOME`; PR C's review found that the `notify` refusal missed quoted/dotted/table forms (a second key would make `config.toml` invalid and Codex would drop the whole file), a trailing-space edit that made uninstall lose our line, a regression in hooks mode (unreadable `config.toml` threw) and a shared hook script deleted while the other mode still used it. Read the code to confirm each finding, dispatch only the confirmed ones, and continue the original coding agent with `SendMessage` (same files, so its context pays off).
- **Skip a finding when it is cosmetic and sits in shared code:** a notify turn without text shows "Working" in `runTracker.ts`; left out of the PR and listed as a live check instead.
- **Look at the real thing, then compute the rest:** the "Strip overflows" report was reproduced in the user's own dev server. The window could not be made as wide as the user's, so an in-page iframe at the target size plus `await import('/src/office/fit.ts')` (Vite serves source modules) evaluated `chooseFit` at any canvas size without editing code. Ask for the user's canvas size, or measure it, before judging a layout bug.
- **Small decided change stays inline:** 001.16 (one constant, four tests) was done by the orchestrator in its own worktree without spawning; PR, CI and merge took less than one agent round.
- **Design says Strip, user prefers Campus:** a layout the algorithm scores best can still feel wrong. The fix was to restrict the candidate list (`FIT_PLANS`) rather than tune the score, which kept every other test meaningful.
- **Worktree before merge, again:** removing both worktrees first and then `gh pr merge --merge --delete-branch` worked for all three PRs. Merging with `--delete-branch` while the worktree exists would have skipped the remote branch delete.
- **Old branches after rebase merges:** `git branch --merged` shows nothing for branches whose PRs were rebase-merged. Check each branch's PR (`gh pr list --head <branch> --state all`) and `git cherry origin/main origin/<branch>` (no `+` lines) before deleting; seven epic 001 branches were removed this way.
- **Stage cost:**
  - 001.12 ≈ 650k: design 148k, coding 131k + 160k + 217k (including the Real-mode reload debug), review 90k, live check 117k.
  - 002.6 PR A ≈ 470k: design 166k, coding 149k + 154k, review 68k.
  - 002.6 PR B ≈ 245k: coding 132k, review 113k; fixes done inline by the orchestrator.
  - 002.6 PR C ≈ 400k: coding 139k, review 102k, fix pass about 10k more through `SendMessage`.
  - 001.16: no spawn.
  - Epic 002 round 2 (002.5 + 002.7) is in the previous section: about 1.1M.

## Live run notes (2026-10-01) — `virtual-agent-office` trial, epic 003 start

**Runs — [003.3] design v5 (hire flow, Staff page, empty slots), [003.1] persisted roster in two PRs (#25 model and storage, #26 wiring and Recent freelancers), [001.13] (earlier the same day).** Manual mode until the user granted per-session commit and merge authority; ClickUp; merge commits.

**Findings, already folded into this file:**
- **Agent reuse and cost:** see "Reusing agents and keeping context small".
- **Session handoff:** see "Handing a pipeline over between sessions".

**Other patterns, no skill change needed yet:**
- **Settle capacity and geometry before the design starts:** [003.3] took four rounds (413k tokens counted) because the user's answers changed the design between rounds: staff per role 3 to 6, then a re-layout of rooms, then the Co-workspace to 6 desks. Each change was cheap for the user to decide and cost 30–70k to redraw. Put "what is the capacity, and does the floor plan have to change?" into the intake questions of any ticket that adds rooms or limits.
- **A design agent can also size the build:** asked to, it simulated `chooseFit` at ten canvas sizes against old and new world sizes and wrote the `layout.ts` change list and the list of tests pinning old numbers into the ticket. That turned "does the layout change need its own ticket?" into a number-based decision.
- **Tech design versus code:** the 003.1 design agent found the brief was wrong in two places (the store does not build `profiles`/`personas`: the feed does, through an existing seam; Mock has 8 personas, not 9) and used the existing seam instead of the brief's idea. Keep asking design agents to verify the brief against code.
- **Reviews caught real design gaps again:** PR A: a role's second agentType never matched (seating takes one per profile), role id `@lead` collided with the Lead pseudo-profile; PR B: an unreadable stored roster (newer version or corrupt) was overwritten by the first automatic recording, a reconnect or feed switch double-counted runs. Confirm each finding by reading the code, send only confirmed ones to the original coding agent.
- **A live check earned its cost on a data-only change:** all seven checks passed, and it showed the visible effect the unit tests could not (the empty Real office packs four rooms to the left and leaves the right half of the floor blank), which became a design question for the next ticket.
- **Commit authority is not merge authority** (see Rules); granting both for one session worked smoothly afterwards.
- **Stage cost:**
  - 003.3 design ≈ 413k over four rounds (279k, +73k, +31k, +30k).
  - 003.1 tech design 97k; PR A: coding 82k, review 79k, fixes +13k; PR B: coding 80k, review 88k, fixes +8k, live check 69k. About 680k in total.
  - [001.13]: no spawn.

## Live run notes (2026-10-01/02) — `virtual-agent-office` trial, epic 003 (build)

**Runs — [003.2] seating rules (PRs #27, #28), [003.4] hire from the drawer (#29, #30), [003.5] Staff page (#31), [003.6] walker routing (#32), plus [003.7] raised by the 003.6 live check.** Manual mode with per-session commit, merge and pull authority (granted by the user at the start of the 2026-10-02 session; it expired with the 2026-10-01 one), ClickUp, merge commits. Two handoffs in the epic, both resumed without correction.

**Findings, already folded into this file:**
- **Spawn only when it pays; live checks on `sonnet`:** see "Reusing agents and keeping context small".
- **Review, fix and live-check habits:** see stage 6 and stage 7.
- **Ticket text goes stale:** see "Handing a pipeline over between sessions".

**Other patterns, no skill change needed yet:**
- **Reviews found real design gaps, again:** names not stable across replays; a storage listener bumping `change` on recents-only writes; a hire-then-throw leaving memory and storage inconsistent (003.2/003.4); an imported file with `lastSeenAt: 1e300` crashing the Staff page on every reload because only `isFinite` was checked (003.5, the only high finding). Any UI that makes a file import reachable deserves a "hand-edited file" probe in the review brief.
- **Design agents catch brief errors:** the 003.5 design agent found Export/Import had no UI at all (only pure functions), `unreadable` private to the store, and Mock recents not always empty. Keep asking design agents to verify the brief against code and to give questions answerable in one line each with a recommended default; the user answered ten of them with one sentence.
- **Tests before source:** agents often write source first despite "tests first". Ask for the report to say so, and for a failing-first run for review fixes (the 003.5 fix round did: 13 failures before the fixes). A reviewer can check it from file mtimes.
- **A weakened test hid a bug twice:** a skip added for a sim overlap in 003.2 hid the routing bug that became 003.6. Removing that skip was the ticket's acceptance criterion and exposed a second cause (walkers planned in one update are not resting yet, so claimed seats must include seats others are walking to). A wider test then showed a neighbouring, pre-existing case (leavers); it became 003.7 instead of widening the PR. Scope the new test to the ticket's case and file the rest.
- **Small decided change inline:** 003.6 (three source files) was done by the orchestrator without a coding spawn, with a failing-first test and a `sonnet` live check; PR, CI and merge in one sitting.
- **Hidden tab:** every live check ran in a `hidden` tab (no animation judged). For routing, the 003.6 agent drove the sim with a virtual clock through the page and sampled paths and distances, which is a better evidence than watching.
- **Commit trailers differ by project:** the project's `commit` skill forbids any trailer, while the session reminder asks for one; the project's own rule won. `eakarin-skills` commits keep the `Co-Authored-By` trailer.
- **PR footer URL:** write it exactly as the project's rule says (`https://claude.com/claude-code`); a hand-typed variant had to be corrected with `gh pr edit`.
- **Stage cost (agents' counters):**
  - 003.2 ≈ 1.05M: tech design 208k; PR A coding 95k + 15k fixes, review 114k, live check 109k; PR B coding 199k + 13k, review 151k, live check 148k.
  - 003.4 ≈ 2.3M: tech design 230k; PR 1 coding 137k + 12k, review 142k; PR 2 coding 218k + 27k, review 175k, live checks 149k + 211k (interrupted twice) + 119k recheck, last fix pass 25k.
  - 003.5 ≈ 0.6M: tech design 127k, coding 134k + 26k fixes, review 133k, live check 177k.
  - 003.6 ≈ 0.12M: live check 118k; everything else inline.
