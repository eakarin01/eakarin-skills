# eakarin-skills

Agent skills loaded by Claude Code.

## Layout

Skills live under `skills/`, grouped into buckets:

- `engineering/` — daily code work
- `productivity/` — daily non-code workflow tools
- `misc/` — kept around but rarely used
- `personal/` — tied to my own setup, not promoted
- `in-progress/` — drafts not yet ready to ship
- `deprecated/` — no longer used

Each skill is its own directory containing a `SKILL.md` (with YAML frontmatter — `name` and `description`) and any bundled scripts or reference files.

## Install

As a Claude Code plugin (all skills under `engineering/` and `productivity/`, listed in `.claude-plugin/plugin.json`):

```bash
/plugin install eakarin-skills --marketplace eakarin01/eakarin-skills
```

The repo is private, so this needs read access to `eakarin01/eakarin-skills` on GitHub.

For my own dev loop — symlink every shippable and `personal/` skill into `~/.claude/skills/` (`in-progress/` and `deprecated/` are skipped):

```bash
./scripts/link-skills.sh
```

List every `SKILL.md` in the repo:

```bash
./scripts/list-skills.sh
```

## Install from other marketplaces

Third-party skills/plugins can be added the same way as any Claude Code marketplace, via the `/plugin` slash command:

```bash
# 1. Add the marketplace
/plugin marketplace add multica-ai/andrej-karpathy-skills

# 2. Install its plugin (bundles all skills in the repo)
/plugin install andrej-karpathy-skills@karpathy-skills
```

`mattpocock/skills` publishes its whole repo as a single plugin too (`mattpocock-skills@mattpocock`) — but since we only wanted the `handoff` skill from it, that one was vendored directly into `skills/personal/handoff/` instead of installed via the marketplace (see the source comment in that file for attribution/update tracking).

Full syntax and other source types (local dir, remote `marketplace.json`, non-GitHub git hosts) are in Claude Code's official plugin marketplace documentation.

## Reference

### Engineering

- **[bump-release](./skills/engineering/bump-release/SKILL.md)** — Create a release or hotfix branch with a bumped semantic version, handling stash, source branch, remote rc checks, and push.
- **[code-walkthrough](./skills/engineering/code-walkthrough/SKILL.md)** — Walk the user through code an agent just implemented, organised by runtime flow with file:line, so a human reviewer can understand a large change without reading every file.
- **[commit](./skills/engineering/commit/SKILL.md)** — Create a git commit in this user's conventional-commit style, scoped from the Jira ticket in the branch name.
- **[pagination-join-safety](./skills/engineering/pagination-join-safety/SKILL.md)** — Checklist for paginated list queries that join or load a one-to-many relation, catching row-multiplying join bugs.
- **[sdlc-manager](./skills/engineering/sdlc-manager/SKILL.md)** — Orchestrate a full software development pipeline from a raw requirement through tickets, design, coding, review, test, deploy, and monitoring, dispatching each stage to a specialist subagent where one exists.

### Productivity

- **[ticket-card](./skills/productivity/ticket-card/SKILL.md)** — Write, summarize, or split a ticket/issue description for any tracker (Jira, Linear, GitHub Issues, etc.).

### Misc

_(none yet)_
