---
name: sdlc-manager
description: Orchestrate a full software development pipeline from a raw requirement through tickets, design, coding, review, test, deploy, and monitoring — dispatching each stage to a specialist subagent where one exists, and to a general-purpose agent otherwise. Reads a per-project config to run fully automated or pause for confirmation between stages; production deploy is always gated. Only ticket breakdown (via ticket-card) and review (via code-review) have a dedicated backing skill; the rest run on general-purpose by default. Trigger on /sdlc-manager or when the user hands you a feature/requirement and asks to run or plan the whole dev pipeline for it.
---

# SDLC Manager

Orchestrator skill. Takes a raw requirement, breaks it into tickets, and drives each ticket through the development pipeline by dispatching stages to specialist subagents via the `Agent` tool. Does not do the work of any stage itself — its job is sequencing, gating, and handing off.

**Status: validated in manual mode on two real trials** (`dormitory-management-system` on Trello + Firebase, `virtual-agent-office` on ClickUp + GitHub — see [`references/run-notes.md`](references/run-notes.md)). Auto mode and resuming from the board are still experimental — see "Status and open work" at the end. Stages other than ticketing and review don't have a dedicated specialist skill/agent yet — see the stage/agent mapping table below. `general-purpose` is the accepted default for those; no need to ask permission per run. The one stage that still needs explicit human confirmation regardless of agent is production deploy — that's a property of the action, not of which agent executes it.

## When to invoke

- `/sdlc-manager`
- User describes a feature/requirement and asks to plan or run the full pipeline for it
- User asks to resume or check status of a pipeline already in progress for the current project

## Pipeline stages

1. **Requirement intake** — clarify scope with the user if the ask is underspecified. Do not proceed to ticketing on a vague requirement; ask.
2. **Ticket breakdown** — split the requirement into tickets, unless it needs no split or no ticket at all (see "Running sub-tickets in parallel" below). Use the `ticket-card` skill's Mode 3 (split) to produce the sub-tickets, each following its Mode 1 template. For each ticket, decide whether it needs UX/UI work (anything touching a user-facing screen/flow) and mark it — this decides whether stage 3 runs for that ticket.
   - **Design/build split**: when a ticket is UI-touching *and* is expected to involve a genuine new design decision (not pure reuse of an existing design-system pattern, not an already-approved import — see "Design-system integration" below), split it into two linked sub-tickets via the same Mode 3 `depends-on:` mechanism used for any other cross-ticket dependency: a **design sub-ticket** (stage 3 only) and a **build sub-ticket** (stages 4 onward, `depends-on:` the design sub-ticket). Skip the split entirely when there's no real design decision to make — the build ticket just runs stage 3 inline as a quick reuse-check with no separate approval step.
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
   - **A new project's first deploy is production:** some hosts (Vercel) promote the first deployment of a new project to production even when asked for a preview. Treat the first deploy of a project as a production deploy and get the gate confirmation first, or create the project and deploy nothing until it is confirmed. Protected deployments (SSO) redirect `curl`; read headers through the host's own tool (`vercel curl`) or a browser the user is signed in to.
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

**Experimental** (dry runs only so far — see "Status and open work"). Chain stages without stopping in between. Still:
- Dispatch each stage as its own `Agent` call — foreground when the next stage depends on this stage's output (e.g. design must finish before coding starts), background only for genuinely independent work (e.g. monitoring after deploy).
- Cross-ticket dependencies (a sub-ticket's `depends-on:` from Mode 3 splitting) apply per stage, not just once: if ticket B depends on ticket A, B's coding stage needs A's actual implementation output (e.g. the real API contract), not just A's design doc. Don't start B's coding off of A's design alone — wait for A's coding stage to finish, unless the two share an approved contract (see "Running sub-tickets in parallel (contract-first)"), in which case B codes against the contract's stubs and both integrate at the end.
- Surface each stage's result to the user as it completes; don't wait until the end of the whole pipeline to report anything.

### Manual mode

Run one stage, report the result, then stop and ask for confirmation before advancing to the next stage. Never assume approval.

### Production deploy is always gated

Staging/UAT deploys can run in `auto` mode like any other stage — treat them as reversible, low-risk. **Production deploy stays a hard confirmation point regardless of `mode`**, since it's hard-to-reverse and affects shared/real-user systems. Only skip the confirmation if `stages.deploy.production` is explicitly set to `"auto"` in the config **and** the user has previously confirmed they want that for this project. When in doubt, ask.

## Task board integration (optional, per-project)

Opt-in via `taskBoard` in `.claude/pipeline.json`. With no `taskBoard`, run purely in chat and don't go looking for a board. **When `taskBoard` is set, read [`references/task-board.md`](references/task-board.md) before stage 2**: config keys (`tool`, `moveBy`, `subTickets`, `stageMap`, `labels`), entry points, one card per sub-ticket with `[003.2]`-style titles, the epic as tracker, and the design/build flow on the board. Move a card in the same turn its stage advances; never batch board updates.

## Design-system integration (optional, per-project)

Opt-in via `designSystem` in `.claude/pipeline.json`. With no `designSystem`, stage 3 reads shipped source and screenshots instead; don't create a design-system project unprompted. **When `designSystem` is set, or the project has a Figma design, read [`references/design-system.md`](references/design-system.md) before stage 3**: bootstrapping, the check-before-design / push-after-ship loop, Figma import, and the fact that there is no export back to Figma.

## Running sub-tickets in parallel (contract-first)

Write the seam between tickets down as a **contract**, build every ticket against the contract (producer implements it, consumer codes against stubs from its fixtures), then integrate once with a live integration check before the first merge. This replaces the sequential "wait for A's coding stage" rule for pairs covered by an approved contract.

The unit of work is a **work unit**, not necessarily a ticket: sub-tickets (default), one ticket with a `Scope by repo` section, or no ticket at all with a run file in `runsDir`. Don't create a ticket just to satisfy the pipeline.

**Read [`references/parallel-runs.md`](references/parallel-runs.md) at stage 2 whenever** a split emits `depends-on:`, the work spans more than one repo, or the user wants it built without a ticket. It covers when to parallelise, the contract's contents and approval, fan-out and change requests, merge order and the integration check, run-file locations, and the one measured cost.

## Resuming a pipeline

**Experimental** (not yet run on a real interrupted pipeline — see "Status and open work").

For a ticketless run, start from `<runsDir>/<slug>/run.md` (or the single repo's `.claude/runs/<slug>/run.md`) instead of an epic card, then verify each unit against git (branches, worktrees, PRs) exactly as in step 3 below.

When the user asks to resume or check status of a pipeline already in progress (see "When to invoke"), and a task board is configured, don't start a fresh breakdown — reconstruct where things actually are:

1. **Read the epic/tracker card.** Its description already links every sub-ticket that's been split off so far (see "Sub-ticket mapping" in `references/task-board.md`). If some obviously-needed sub-tickets aren't listed yet, the ticket-breakdown stage is incomplete — offer to continue it before touching execution stages.
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

Hand off at a **ticket boundary** (PR merged, board updated), never mid-stage, and early rather than late: the owner's rule of thumb is when the orchestrator's context passes about 300K tokens. The new session verifies the handoff doc against git and the board before acting, and asks again for any authority (commit, merge) the old session had. **Read [`references/session-handoff.md`](references/session-handoff.md) before writing or resuming from a handoff doc**: the nine things it must hold, in order, and what is and isn't known about the threshold.

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

## Status and open work

Graduated out of `in-progress/` on 2026-10-19. Graduation criteria: all 9 stages run live in **manual mode** on 2 real projects (`dormitory-management-system`, `virtual-agent-office`), including a real deploy with a smoke test and a contract-first parallel run. Both conditions hold; the evidence is in [`references/run-notes.md`](references/run-notes.md).

**Experimental — not yet proven on real work:**
- **Auto mode.** Only exercised in dry runs; every real trial used manual mode. Before running it unattended on a project, tell the user it is experimental and offer manual mode instead. Fold the first real auto run's findings into this file.
- **Resuming a pipeline from the board.** The resume algorithm has never run on a real interrupted pipeline; the trial close-outs resumed from a handoff doc plus git state. Follow it, but report the reconstruction to the user before resuming (its step 5) without exception.

**Open work, none of it blocking:**
- Specialist agents/skills for stages 3–5 and 7–9: build one only when `general-purpose` proves insufficient for a stage on real work, not speculatively.
- Contract-first parallel work on a multi-repo epic, and a token cost comparison with a sequential run.
- Measure the handoff thresholds (the owner's 300K orchestrator context, the 100–150k continue-versus-fresh agent figure) on a real run.
