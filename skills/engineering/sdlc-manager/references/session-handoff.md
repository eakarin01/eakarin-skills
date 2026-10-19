# Handing a pipeline over between sessions

Part of [`sdlc-manager`](../SKILL.md). A quoted section name not found in this file, and "above"/"below" pointing at stages, refer to `SKILL.md`.

The orchestrator's own context also grows (many ticket texts, agent reports, tool output). Hand off at a **ticket boundary** (a PR merged and the board updated), never in the middle of a stage, and before the context is so large that every turn is expensive. No threshold has been measured; use the client's context indicator and err on the early side. A new session also picks up changes to this skill, which is another reason to hand off after editing it.

The handoff doc (one file, written by the orchestrator; the `handoff` skill can produce the shell) must hold, in this order:
1. **Where things stand:** project path and repo, board ids and links, `pipeline.json` summary, and a ticket table (state, link, one note) for the epic in progress and the next ones.
2. **Resume point:** the single next step, with the options and a recommendation.
3. **Decisions not written elsewhere:** answers the user gave in chat, with dates. Anything already on the board or in a design doc is a link, not a copy.
4. **Working rules for this project:** runtime version, push and auth commands, commit format and who confirms commits, ports the user owns, subagent pitfalls, tests and how to trust their output.
5. **State to verify:** unpushed commits, open PRs, worktrees, branches pending deletion, services started.
6. **Things the user will look at themselves** and have not been verified by any stage.
7. **Reference files** outside the repo (design docs, patches) with paths.
8. **Cost ledger:** tokens per stage so far, to fold into `references/run-notes.md` when the epic closes.
9. **Authority granted in this session** (e.g. "auto commit", "auto merge") and its scope. Say plainly that it expires with the session; the new session must ask again.

The new session must **verify, not trust**: re-read this skill, run `git worktree list`, `git status` and `git log` in the project, read the epic's board state, and compare each with the doc before acting. If they disagree, trust the evidence and tell the user (see "Resuming a pipeline"). Three handoffs in the trial (2026-09-30, 2026-10-01 and 2026-10-02) resumed this way without a correction. The 2026-10-02 session also found the next ticket's text out of date against four shipped tickets and rewrote it before any design: read the ticket against what shipped, not only against the board state.

**Draft, 2026-10-02: when to hand off, and what is known.** The owner's working rule is to open a new session at a ticket boundary once the orchestrator's own context is about **300K tokens or more**. It is a rule of thumb from use, not a measured threshold. What outside sources say (checked 2026-10-02; none gives a handoff threshold):
- Per-turn cost grows with context because every turn re-reads it. One measured multi-agent run ([touchstone issue #11](https://github.com/dimoschi/touchstone/issues/11), single run, model not stated): 85% of cost was re-read context, phase-to-phase handoffs were 0.6%, each agent's context grew from about 13–28k to 105–119k tokens over 90–142 turns, and the last 30% of turns took 42–46% of cache reads.
- One anecdote ([architecture-kit issue #85](https://github.com/shawrylk/architecture-kit/issues/85), one session, no method given): a fresh agent for a two-line follow-up cost 45k tokens, whereas the resumed agent would have re-sent 199k on every call. It proposes fixed text first and task text last in dispatch prompts, and passing the brief as a file path.
- [Anthropic, effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): compaction (summarise and reinitiate), notes kept outside the context, and sub-agents that return a 1,000–2,000 token summary; recall gets worse as context grows ("context rot", also [Chroma's study](https://trychroma.com/research/context-rot)); no thresholds or costs given. [Anthropic's multi-agent research post](https://www.anthropic.com/engineering/multi-agent-research-system): multi-agent runs use about 15 times the tokens of a chat, and token use explains most of the quality difference, so cutting tokens is not free.
- Not applicable: [The Handoff Tax](https://arxiv.org/pdf/2608.24358) is about continuing another model's trajectory, not about restarting the same model from a file.

Open questions, to measure on a real run before turning any of this into a rule: the context size where a fresh session beats continuing (the 300K figure above, and the 100–150k for a coding agent under "Reusing agents"), how much of the re-read is served from the prompt cache for a subagent in this harness, and whether the fixed-prefix brief shape helps. Until then: hand off at ticket boundaries, earlier rather than later, and when the client's context indicator passes the owner's 300K.
