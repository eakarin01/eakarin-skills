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

For my own dev loop — symlink every shippable skill into `~/.claude/skills/`:

```bash
./scripts/link-skills.sh
```

List every `SKILL.md` in the repo:

```bash
./scripts/list-skills.sh
```

## Reference

### Engineering

- **[bump-release](./skills/engineering/bump-release/SKILL.md)** — Create a release or hotfix branch with a bumped semantic version, handling stash, source branch, remote rc checks, and push.
- **[commit](./skills/engineering/commit/SKILL.md)** — Create a git commit in this user's conventional-commit style, scoped from the Jira ticket in the branch name.
- **[pagination-join-safety](./skills/engineering/pagination-join-safety/SKILL.md)** — Checklist for paginated list queries that join or load a one-to-many relation, catching row-multiplying join bugs.

### Productivity

- **[jira-card](./skills/productivity/jira-card/SKILL.md)** — Write or summarize a Jira card description for engineering tickets.

### Misc

_(none yet)_
