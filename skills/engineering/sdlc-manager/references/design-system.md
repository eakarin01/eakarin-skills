# Design-system integration (optional, per-project)

Part of [`sdlc-manager`](../SKILL.md). A quoted section name not found in this file, and "above"/"below" pointing at stages, refer to `SKILL.md`.

A project may keep a hosted design-system project (via the `DesignSync` tool, backing claude.ai/design) as the source of truth for its UI components and foundations (colors, typography, buttons, forms, ...), instead of stage 3 re-deriving conventions from scratch on every UI ticket. This is **opt-in per project** via `.claude/pipeline.json`, same pattern as `taskBoard`. If a project has no `designSystem` config, stage 3 falls back to reading shipped source/screenshots directly (see stage 3 in `../SKILL.md`) — don't create a design-system project unprompted mid-pipeline.

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
- **If a project already has a Figma design**, don't re-derive it into HTML previews from scratch (that's only the fallback for projects with no existing design source) — import the real Figma frames into the `designSystem` project instead, via Claude Design's Figma-import ("Start with context" → Figma) so Figma stays the actual source of truth rather than something reconstructed from code/screenshots. Ask the user, per-project, whether Figma or the synced claude.ai/design copy is the system of record going forward — this changes whether stage 3 treats the design-system project as read-only reference (re-import from Figma when it changes) or as the thing it edits directly.
- **Figma export is one-way (import only), verified 2026-08-12**: Claude Design (claude.ai/design) has no native export back to Figma (`.fig`) — checked live via the product's own Share → Export menu (PDF, Project HTML, PowerPoint, PNG, video, and a fixed list of "Connect destinations" that does not include Figma) and cross-checked with a web search, both agreeing. If a project needs Claude Design changes reflected back in Figma (e.g. to keep a read-only client-facing Figma view current), there is no automatic sync path — say so explicitly rather than assuming one exists, and the only options are manual: browser extensions like `html.to.design` to capture a Claude Design page into Figma, or exporting PNG/SVG and placing it on the Figma canvas by hand. This is a live product surface (labelled Beta) and can change — re-verify before relying on this if it's been a while.
