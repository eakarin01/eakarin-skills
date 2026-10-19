# Running sub-tickets in parallel (contract-first)

Part of [`sdlc-manager`](../SKILL.md). A quoted section name not found in this file, and "above"/"below" pointing at stages, refer to `SKILL.md`.

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

One real run so far (`virtual-agent-office` phase 3: 3 units in parallel, about 0.8M subagent tokens, see `references/run-notes.md`); no sequential comparison yet. Expect the contract stage to cost one design-sized agent round or an inline orchestrator pass, and each fan-out ticket to cost what it would alone, plus a stub and a contract test each; the saving is wall-clock time, not tokens. Record real numbers in the run notes.
