# sdlc-manager run notes

Dry runs and live trials behind the rules in [`../SKILL.md`](../SKILL.md), oldest first. Lessons that became rules are already folded into SKILL.md; these notes keep the evidence and the per-stage token costs.

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

## Live run notes (2026-10-02/04) — `virtual-agent-office` trial, epic 002 round 4 (answering a question from the office)

**Runs — [002.9] design, [002.10] capture, [002.11] spike, [002.12] UI design, [002.13] collector registry (#38), [002.14] hook wait mode (#39), [002.15] app card (#40), [002.16] docs and headers (#41), [002.17] live check by the user, [002.19] hook-gone hardening (#42).** Manual mode, merge commits, per-session commit/push/merge authority that expired with each session. Three handoffs, all resumed by verifying git, board and doc against each other. [002.18] (approve or deny permissions, v1b) was deliberately not started: the user decided to leave it; it needs spikes only the user can run and a pairing step, so it is a decision for the owner, not a pipeline step.

**Findings (kept here until a second project confirms them):**
- **A spike that needs the user's login:** the auto-mode classifier refuses to let the assistant create hook-registering settings. Write the throwaway scripts, give the user the commands to create the settings, and read the results from the capture files, not from the user's description. The first spike prompt must force the shape under test (a "two questions" prompt gave one question).
- **An outsider review of a design pays for itself:** `scrutinize` on the 002.9 design (109k) removed an auth layer v1a did not need, found a real flaw in the planned pairing (a hook-token holder could forge an Origin) and several wrong claims about current code.
- **The live check found what no review saw, twice:** a preflight 403 without CORS made the app's own error message unreachable (002.15), and a killed hook left a card that accepted answers nobody received (002.17, T7). The second was already a ticket from review; the live check confirmed it and fixed its priority. Keep stage 7 for UI and protocol tickets after a clean review, and ask the user to kill processes mid-wait, not only to run the happy path.
- **Agents above ~200k tokens stalled** three times ("no progress for 600s") on one fix pass. Ask each agent to append a progress line to a named file as its first action and after each piece, keep tool calls small and send test output to a file. A fresh smaller-model agent with a file brief finished the same pass.
- **Contract-first worked in practice:** a contract file plus "decisions already made" let collector, hook and app be built in separate worktrees without conflicts. The contract file was also the source for the docs ticket.
- **Docs written against the design drift from the code:** 002.16 first said a clipped option list made a card read-only; the code only does that for a redacted label. Check every behavioural claim in docs against the shipped code before committing (one grep each).
- **Small and decided stays inline:** 002.16 and 002.19 were done by the orchestrator with no spawn (tests first for 002.19: 5 failures before the change, two older tests updated and said so in the PR). 002.19 had no outside review; say so in the report rather than implying one.
- **A handoff's "ask before pulling the user's checkout" is easy to break:** a `git pull` in the user's main checkout reloads their running app. The session grant said "pull"; the handoff said ask again. Ask once per session before the first pull in the user's checkout, even when a grant exists, unless the grant names that checkout.
- **Stage cost (agents' counters):** 002.10 ≈ 0.27M (coding 97k, review 92k, live check 82k); design ≈ 0.49M (design 207k with revision, outsider review 109k, UI design 172k); 002.13 ≈ 0.43M plus stalls; 002.14 ≈ 0.31M; 002.15 ≈ 0.54M (coding and fix 263k, review 113k, live check and recheck 162k); 002.16 and 002.19 inline. About 2.0M subagent tokens for 002.10 to 002.15, six PRs, roughly 400 new tests before 002.19 (2,078 to 2,090 tests, 104 files).

## Live run notes (2026-10-05) — `virtual-agent-office` trial, [001.7] deploy via CLI

**Run:** first real stages 8–9. Manual mode, inline (no subagents), one PR (#43, merge commit), about 20 tool calls of deploy work. Handoff doc, git and board agreed, so resuming needed no repair.

**Findings (kept here until a second project confirms them):**
- **The first "preview" deploy was production.** The assistant ran `deploy:preview` as the first deploy of a new Vercel project; Vercel promoted it to production. No harm (Authentication was on), but the production gate was bypassed by a host rule nobody had read. Now in stage 8.
- **A config never tested on a real deploy was fine:** `vercel.json` headers (written in an earlier PR) were all served. Check them with `vercel curl`, because a plain `curl` hits the SSO redirect and shows the wrong headers.
- **Declared a script that was not installed:** `package.json` scripts called `vercel`, which only existed through `npx`; the first run failed with "command not found". Run every documented command once before committing it.
- **A permission denial can leave a false note behind:** the merge was blocked by the auto-mode classifier ("Merge Without Review") in the same turn as a ClickUp comment saying "merged". Post board notes after the action succeeds, not in parallel with it, and fix any note that got ahead of the facts. The user's explicit "merge #43" in chat cleared it.
- **A decision the user questioned:** read-only versus answering from the hosted page. Explain the real risk (the hosted code runs in the user's browser and could approve for them; visitors cannot reach the user's collector) instead of repeating "accepted risk"; the user then chose read-only, changeable later on the collector without a redeploy.

## Live run notes (2026-10-05) — `virtual-agent-office` trial, epic 004-style feature (phase 3: review page)

**Run:** ClickUp epic with 5 subtasks, manual mode with per-session commit, merge and push authority (UI PRs wait for the user), contract-first. One orchestrator session from intake to a production deploy; PRs #44–#51, merge commits. Units: A collector endpoint, B pure parser plus a skill edit in another repo, C review page, D links between them. A, B and C ran in parallel from one contract commit; D followed C.

**Findings (kept here until a second project confirms them):**
- **The contract paid off, and its one gap came from the consumer:** unit C found a type missing from the error list the contract itself described (403 codes). One change request in four units, applied once (v1.1) and merged into the other branches. Write the types in code (a shared package) and fixtures next to the text, and add a tiny test that keeps the fixtures consistent with the types.
- **The integration check found nothing wrong, and was still worth doing:** a scratch worktree merging all three branches, a real collector, a real git repo and Chrome took about 15 tool calls. It also exposed that the collector did not remember `cwd` at all (found earlier, when reading the code to write the contract).
- **Review of a security-relevant endpoint needs two rounds:** round 1 found a high (a user's `diff.noprefix` config cut two characters off every path and evaded the secret-file rule) that every test missed because the tests had cleared the global git config. Round 2 found that the fix for multi-line secrets blanked ordinary code that merely contained the marker text. Ask the reviewer to try again on the new code, and say in the brief that tests must not hide user config.
- **A reviewer found a crash in old code (`GET //[` took the collector down).** It went into the same PR as its own small change, and the PR says so.
- **A fix agent that reports "all pass" is not a pass:** counting test files on disk and rerunning the checks myself caught nothing this time, but a `--reporter=json` run wrote a report into the tree and it was committed once. Check `git status` before every commit, not only the staged list at the end.
- **Auto-mode classifier denied an amend plus force-push of the agent's own unmerged branch.** A follow-up commit was enough; do not ask for the rewrite.
- **GitHub closes a stacked PR when its base branch is deleted by the merge.** Retarget the stacked PR to the main branch before `--delete-branch`, or open a replacement PR from the same branch (what happened).
- **A leftover dev server and a shared port cost two detours.** The user's dev server on 5173 (in another worktree) served old code, and a broad `pkill -f "port 47912"` killed the user's own demo collector. Never `pkill` by a pattern that can match the user's processes; look at `lsof` first, use a different port (4173 preview is allowed by the collector's defaults), and say which processes you started.
- **A jump that moves focus by script after a mouse click shows no focus ring**, so "nothing happens" (found by the user, not by tests or the live check). Mark the target for a moment as well.
- **A hidden Chrome tab drops real clicks and key presses again;** `element.click()` through the page script worked, and key presses did not move focus. Say so in the report and ask the user for a real-input pass.
- **A `vercel link` is per checkout and `.vercel/` is git-ignored:** the main checkout had none, so a bare `vercel deploy --prod` would have created a new project. Check `vercel project ls`, link with `--project`, then deploy; the link also writes `.env.local` (ignored).
- **Stage cost (agents' counters, cumulative per agent):** A 188k (code 135k, two fix rounds), B 65k, C 146k (code 130k, one fix round), D 109k, security reviewer 172k over two rounds, UI reviewer 105k. About 0.8M subagent tokens for four PRs plus a contract, a parser, an endpoint and a page; orchestrator reading and checks come on top.
