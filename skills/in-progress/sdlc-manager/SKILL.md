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
4. **Technical design** — architecture/approach for each ticket, informed by the UX/UI output when stage 3 ran.
5. **Coding** — implementation.
6. **Review** — use the `code-review` skill against the resulting diff.
7. **Test** — verification (unit/integration, or manual test plan if no test agent exists yet).
8. **Deploy** — ships the change, per environment:
   - **Staging/UAT** — low-risk, reversible. Can run in `auto` mode like any other stage.
   - **Production** — **always a hard gate** (see below), regardless of `mode`.
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
    }
  }
}
```

- `tool` names which integration to use (only whatever MCP tools/skills are actually available in the session — don't assume one is connected without checking).
- `lists` maps the pipeline's generic stage names to this board's actual list names, since boards vary. Read from this map rather than hardcoding a list name.

**Entry points (generic, board-agnostic):**
- **A — requirement given as a prompt:** refine it in-chat, then, if `taskBoard` is configured, create it as a card/issue in the `ready` list (skip `refine` if already fully scoped — no open questions left unresolved).
- **B — requirement already exists on the board:** scan the `refine` and `ready` lists for items to pick up. An item in `refine` still needs intake/scoping before ticket breakdown; an item in `ready` can go straight to stage 2.

**Sub-ticket mapping (stage 2 — ticket breakdown), when a board is configured:** `ticket-card` Mode 3 output maps to **one card/issue per sub-ticket**, not a checklist on the parent — each sub-ticket needs to move independently through the stage lists (e.g. a UI sub-ticket can be in `review` while a backend sub-ticket is still `inProgress`), and `depends-on:` relationships need to be visible as links between real items, not buried in one item's checklist.
- The original requirement item becomes the **epic/tracker**: its description links to each sub-ticket item, and it moves to `inProgress` once the first sub-ticket starts, to `done` once every sub-ticket reaches `done`. It does not itself move through `review`.
- Each sub-ticket item starts in `ready` (already scoped by the split) and moves through `ready → inProgress → review → done` independently as its own stages 3–9 run.
- Encode `depends-on:` in the dependent item's description (e.g. a link to the blocking item) so it's visible without relying on checklist-item state.

**Card/issue movement is a side effect of stage transitions, not a separate step:** when a sub-ticket's stage advances, move it to the matching list in the same turn — don't batch board updates for later.

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
