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

### With the skills CLI (pick individual skills)

The [skills CLI](https://github.com/vercel-labs/skills) works with Claude Code and other agents.

```bash
# See which skills the repo offers
npx skills add eakarin01/eakarin-skills --list

# Interactive: tick the skills and agents you want (run in a normal terminal)
npx skills add eakarin01/eakarin-skills

# Non-interactive: name the skills you want
npx skills add eakarin01/eakarin-skills --skill ticket-card --skill commit

# Install everything without prompts
npx skills add eakarin01/eakarin-skills -y
```

Run from inside an agent session (e.g. `!` in Claude Code), the CLI skips the picker and installs every skill, so pass `--skill` there.

Installed skills are copied to your machine and do not update on their own:

```bash
npx skills check              # list skills with a newer version on GitHub
npx skills update             # pull the latest version
npx skills remove ticket-card # uninstall one
```

### As a Claude Code plugin (all shipped skills)

Installs every skill under `engineering/` and `productivity/`, as listed in `.claude-plugin/plugin.json`:

```bash
/plugin marketplace add eakarin01/eakarin-skills
/plugin install eakarin-skills@eakarin-skills
```

Update with `/plugin marketplace update eakarin-skills`, or turn on auto-update for the marketplace in `/plugin`. Bump `version` in `.claude-plugin/plugin.json` on every release so installs pick up the change.

### Notes for this repo

The skills CLI scans the whole repo and parses frontmatter with strict YAML. Wrap a `description` that contains `: ` in a `>-` block scalar, or the skill is skipped.

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
