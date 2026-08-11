---
name: sdlc-manager
description: Orchestrate a full software development pipeline from a raw requirement through tickets, design, coding, review, test, deploy, and monitoring — dispatching each stage to a specialist subagent. Reads a per-project config to run fully automated or pause for confirmation between stages, with deploy gated by default. DRAFT — specialist subagents for design/coding/test/deploy/monitor are not built yet; only ticket breakdown (via jira-card) and review (via code-review) currently have a real backing skill. Trigger on /sdlc-manager or when the user hands you a feature/requirement and asks to run or plan the whole dev pipeline for it.
---

# SDLC Manager

Orchestrator skill. Takes a raw requirement, breaks it into tickets, and drives each ticket through the development pipeline by dispatching stages to specialist subagents via the `Agent` tool. Does not do the work of any stage itself — its job is sequencing, gating, and handing off.

**Status: draft.** Stages other than ticketing and review don't have a dedicated specialist skill/agent yet — see the stage/agent mapping table below. Until those exist, fall back to `general-purpose` (or ask the user how they want that stage handled) and flag it as a gap rather than silently improvising.

## When to invoke

- `/sdlc-manager`
- User describes a feature/requirement and asks to plan or run the full pipeline for it
- User asks to resume or check status of a pipeline already in progress for the current project

## Pipeline stages

1. **Requirement intake** — clarify scope with the user if the ask is underspecified. Do not proceed to ticketing on a vague requirement; ask.
2. **Ticket breakdown** — split the requirement into tickets. Use the `jira-card` skill to draft each ticket's description. For each ticket, decide whether it needs UX/UI work (anything touching a user-facing screen/flow) and mark it — this decides whether stage 3 runs for that ticket.
3. **UX/UI design** — wireframes/mockups/flow. **Conditional**: only runs for tickets flagged as UI-touching in stage 2; skip straight to stage 4 for backend-only or infra-only tickets. Its output (screens, flows, states) feeds stage 4 as a constraint on the technical approach.
4. **Technical design** — architecture/approach for each ticket, informed by the UX/UI output when stage 3 ran.
5. **Coding** — implementation.
6. **Review** — use the `code-review` skill against the resulting diff.
7. **Test** — verification (unit/integration, or manual test plan if no test agent exists yet).
8. **Deploy** — ships the change, per environment:
   - **Staging/UAT** — low-risk, reversible. Can run in `auto` mode like any other stage.
   - **Production** — **always a hard gate** (see below), regardless of `mode`.
9. **Monitor** — post-deploy observation for regressions.

Stages 3–5 and 7–9 have no dedicated specialist skill in this repo yet. Treat them as open slots in the pipeline, not stages to skip (except stage 3, which is intentionally skippable per-ticket as described above).

## Stage → agent mapping

| Stage | Backing skill/agent today | Gap |
|---|---|---|
| Ticket breakdown | `jira-card` skill | none |
| UX/UI design | none | needs a specialist agent; conditional per ticket |
| Technical design | none | needs a specialist agent |
| Coding | none | needs a specialist agent |
| Review | `code-review` skill | none |
| Test | none | needs a specialist agent |
| Deploy | none | needs a specialist agent |
| Monitor | none | needs a specialist agent |

When dispatching a stage with no backing skill, say so explicitly to the user before improvising with a general-purpose agent.

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
- If the file doesn't exist, default to `"manual"` everywhere — don't assume a project wants unattended execution, including to staging. Ask the user once whether to create the config, and with what defaults.

### Auto mode

Chain stages without stopping in between. Still:
- Dispatch each stage as its own `Agent` call — foreground when the next stage depends on this stage's output (e.g. design must finish before coding starts), background only for genuinely independent work (e.g. monitoring after deploy).
- Surface each stage's result to the user as it completes; don't wait until the end of the whole pipeline to report anything.

### Manual mode

Run one stage, report the result, then stop and ask for confirmation before advancing to the next stage. Never assume approval.

### Production deploy is always gated

Staging/UAT deploys can run in `auto` mode like any other stage — treat them as reversible, low-risk. **Production deploy stays a hard confirmation point regardless of `mode`**, since it's hard-to-reverse and affects shared/real-user systems. Only skip the confirmation if `stages.deploy.production` is explicitly set to `"auto"` in the config **and** the user has previously confirmed they want that for this project. When in doubt, ask.

## Rules

- Never fabricate a stage's output (e.g. don't invent a design doc or test results) — dispatch to the real subagent/skill or ask the user to provide it.
- If a requirement is ambiguous, resolve it at intake, not mid-pipeline.
- Report per-stage status as you go: what stage ran, which skill/agent handled it, and the outcome.
- If the pipeline stalls on a missing specialist agent, tell the user which stage is blocked and ask how they want to proceed (build the specialist agent now, handle manually, or use general-purpose as a stand-in for this run only).

## Open TODOs before this skill graduates out of `in-progress/`

- [ ] Build specialist agents/skills for UX/UI design, technical design, coding, test, deploy, monitor stages
- [ ] Decide the concrete shape of `.claude/pipeline.json` once used on a real project (per-ticket overrides? per-branch?)
- [ ] Validate the auto-mode dependency chaining on an actual multi-stage run before trusting it unattended
