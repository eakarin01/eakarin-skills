---
name: sdlc-manager
description: Orchestrate a full software development pipeline from a raw requirement through tickets, design, coding, review, test, deploy, and monitoring — dispatching each stage to a specialist subagent where one exists, and to a general-purpose agent otherwise. Reads a per-project config to run fully automated or pause for confirmation between stages; production deploy is always gated. DRAFT — only ticket breakdown (via ticket-card) and review (via code-review) currently have a real backing skill; the rest run on general-purpose by default. Trigger on /sdlc-manager or when the user hands you a feature/requirement and asks to run or plan the whole dev pipeline for it.
---

# SDLC Manager

Orchestrator skill. Takes a raw requirement, breaks it into tickets, and drives each ticket through the development pipeline by dispatching stages to specialist subagents via the `Agent` tool. Does not do the work of any stage itself — its job is sequencing, gating, and handing off.

**Status: draft, not yet validated on a real run.** Stages other than ticketing and review don't have a dedicated specialist skill/agent yet — see the stage/agent mapping table below. `general-purpose` is the accepted default for those; no need to ask permission per run. The one stage that still needs explicit human confirmation regardless of agent is production deploy — that's a property of the action, not of which agent executes it.

## When to invoke

- `/sdlc-manager`
- User describes a feature/requirement and asks to plan or run the full pipeline for it
- User asks to resume or check status of a pipeline already in progress for the current project

## Pipeline stages

1. **Requirement intake** — clarify scope with the user if the ask is underspecified. Do not proceed to ticketing on a vague requirement; ask.
2. **Ticket breakdown** — split the requirement into tickets. Use the `ticket-card` skill's Mode 3 (split) to produce the sub-tickets, each following its Mode 1 template. For each ticket, decide whether it needs UX/UI work (anything touching a user-facing screen/flow) and mark it — this decides whether stage 3 runs for that ticket.
3. **UX/UI design** — wireframes/mockups/flow. **Conditional**: only runs for tickets flagged as UI-touching in stage 2; skip straight to stage 4 for backend-only or infra-only tickets. Its output (screens, flows, states) feeds stage 4 as a constraint on the technical approach.
   - Must produce a real rendered artifact, not narrated text. Use the `Artifact` tool (with the `artifact-design` skill) to build an actual HTML wireframe/mockup for every new-or-changed screen and state (loading/error/empty, not just the happy path) — this is the same "don't fabricate a stage's output" principle already in Rules, applied to design instead of code/tests.
   - If `.claude/pipeline.json` has `designSystem` configured (see "Design-system integration" below), check it via `DesignSync` *before* designing anything new — reuse an existing component/pattern rather than inventing a fresh one, and only design something new for a genuine gap. After the ticket's coding stage lands, push new or changed components back into the project so it stays current for the next ticket. If no `designSystem` is configured, fall back to reading the actual shipped source (and a live screenshot via `claude-in-chrome` if it's already deployed) for whatever screens already exist, so the new screen stays visually/structurally consistent instead of guessing fresh each time.
4. **Technical design** — architecture/approach for each ticket, informed by the UX/UI output when stage 3 ran.
5. **Coding** — implementation.
6. **Review** — use the `code-review` skill against the resulting diff. If findings need fixing, dispatch a separate, scoped `Agent` call listing only those findings (don't reuse the original coding-stage agent's context) — then the orchestrator commits once, after the user confirms, rather than the sub-agent committing. This keeps the diff the user reviews and the diff that gets committed identical.
7. **Test** — verification (unit/integration, or manual test plan if no test agent exists yet). If this is the first ticket in the project needing a given kind of test (e.g. no test framework installed yet), expect this stage to cost noticeably more than design/coding — it's paying for one-time infra setup (test runner, emulator wiring, config), not just writing test cases. Say so when reporting the stage's result, so the cost isn't a surprise. Later tickets that reuse the same infra won't carry that cost.
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

### First-run bootstrap

If `.claude/pipeline.json` doesn't exist when `sdlc-manager` is invoked on a project, treat this as a one-time setup conversation before running any stage — ask once, covering both of the following together, not in separate follow-up passes:

1. **Mode:** project-wide `auto` or `manual`, and whether `deploy` needs per-environment overrides.
2. **Task board (optional):** whether this project tracks work on an external board. If the user names one, check which matching MCP tools/skills are actually connected in the current session (don't assume — verify) before configuring `taskBoard`. If none is connected yet, say so and offer to proceed without board integration rather than configuring a tool that isn't reachable. If the user has no board, omit `taskBoard` entirely — it's optional, not a default.

Write the resulting `.claude/pipeline.json` (see shape above and in "Task board integration" below), confirm it back to the user in one message, then proceed to stage 1. Re-running this bootstrap on a project that already has the file is unnecessary — just read it.

### Auto mode

Chain stages without stopping in between. Still:
- Dispatch each stage as its own `Agent` call — foreground when the next stage depends on this stage's output (e.g. design must finish before coding starts), background only for genuinely independent work (e.g. monitoring after deploy).
- Cross-ticket dependencies (a sub-ticket's `depends-on:` from Mode 3 splitting) apply per stage, not just once: if ticket B depends on ticket A, B's coding stage needs A's actual implementation output (e.g. the real API contract), not just A's design doc. Don't start B's coding off of A's design alone — wait for A's coding stage to finish.
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
    "lists": {
      "refine": "To Refine",
      "ready": "Ready",
      "inProgress": "In Progress",
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

- `tool` names which integration to use (only whatever MCP tools/skills are actually available in the session — don't assume one is connected without checking).
- `lists` maps the pipeline's generic stage names to this board's actual list names, since boards vary. Read from this map rather than hardcoding a list name.
- `labels` is optional. If present, attach `labels.epic` to the epic/tracker item and `labels.subTask` to each sub-ticket item when creating them, so epic vs. sub-ticket is visually distinguishable on the board (in addition to the naming convention below). If the board tool can't create/rename labels via the tools available in-session, don't try to work around that mid-pipeline — set up the label mapping once during the first-run bootstrap (find or create suitable labels, get the user to name them if the tool can't do it programmatically) and record the resulting identifiers in this config, same as `lists`. If `labels` is absent, skip label attachment entirely — it's a visual nice-to-have, not required for the pipeline to function.

**Entry points (generic, board-agnostic):**
- **A — requirement given as a prompt:** refine it in-chat, then, if `taskBoard` is configured, create it as a card/issue in the `ready` list (skip `refine` if already fully scoped — no open questions left unresolved).
- **B — requirement already exists on the board:** scan the `refine` and `ready` lists for items to pick up. An item in `refine` still needs intake/scoping before ticket breakdown; an item in `ready` can go straight to stage 2.

**Sub-ticket mapping (stage 2 — ticket breakdown), when a board is configured:** `ticket-card` Mode 3 output maps to **one card/issue per sub-ticket**, not a checklist on the parent — each sub-ticket needs to move independently through the stage lists (e.g. a UI sub-ticket can be in `review` while a backend sub-ticket is still `inProgress`), and `depends-on:` relationships need to be visible as links between real items, not buried in one item's checklist.
- The original requirement item becomes the **epic/tracker**: its description links to each sub-ticket item, and it moves to `inProgress` once the first sub-ticket starts, to `done` once every sub-ticket reaches `done`. It does not itself move through `review`.
- Each sub-ticket item starts in `ready` (already scoped by the split) and moves through `ready → inProgress → review → done` independently as its own stages 3–9 run.
- **Naming convention:** give each sub-ticket a title prefixed with the parent requirement's number and its own index, e.g. `[003.2] <sub-ticket title>` for the 2nd sub-ticket of requirement 003. The epic/tracker keeps its plain title, no prefix — this alone (before any label config) makes epic vs. sub-ticket distinguishable at a glance.
- Encode `depends-on:` in the dependent item's description (e.g. a link to the blocking item) so it's visible without relying on checklist-item state.

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

## Resuming a pipeline

When the user asks to resume or check status of a pipeline already in progress (see "When to invoke"), and a task board is configured, don't start a fresh breakdown — reconstruct where things actually are:

1. **Read the epic/tracker card.** Its description already links every sub-ticket that's been split off so far (see "Sub-ticket mapping" above). If some obviously-needed sub-tickets aren't listed yet, the ticket-breakdown stage is incomplete — offer to continue it before touching execution stages.
2. **For each linked sub-ticket, read its current list.** The list gives a coarse stage bucket (e.g. `inProgress` covers stages 3–5, `review` covers 6–7, `done` covers 8–9) — not the exact stage, since one list spans multiple pipeline stages.
3. **Pin down the exact stage from real evidence, not the list alone.** Check the actual repo: commits/PRs referencing the sub-ticket (e.g. `git log --grep=<ticket-id-or-Trello-URL>`), whether design/coding artifacts exist, whether tests exist and pass, whether it's been deployed. The list position is a hint for where to start looking, not the source of truth — this follows the same "never fabricate, verify against the real thing" rule as any other stage. If the list and the real evidence disagree (e.g. card says `review` but no commit exists yet), trust the evidence and correct the card's list to match.
4. **Resume from the next stage after the last one with real, verified output**, respecting `depends-on:` — don't resume a sub-ticket whose blocking dependency isn't actually finished, even if its card looks further along.
5. Report what was found per sub-ticket (list vs. actual state, any corrections made) before resuming work, so the user can catch a wrong reconstruction before more stages run on top of it.

## Rules

- Never fabricate a stage's output (e.g. don't invent a design doc or test results) — dispatch to the real subagent/skill or ask the user to provide it.
- If a requirement is ambiguous, resolve it at intake, not mid-pipeline.
- Report per-stage status as you go: what stage ran, which skill/agent handled it, and the outcome.
- Missing a specialist agent is not a blocker — dispatch to `general-purpose` and move on. Only stop and ask if `general-purpose` genuinely can't do the stage (e.g. it needs credentials/access the user hasn't granted).

## Open TODOs before this skill graduates out of `in-progress/`

- [ ] Build specialist agents/skills for UX/UI design, technical design, coding, test, deploy, monitor stages — as-needed, once general-purpose proves insufficient for a given stage on real work, not speculatively
- [ ] Decide the concrete shape of `.claude/pipeline.json` once used on a real project (per-ticket overrides? per-branch?)
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
