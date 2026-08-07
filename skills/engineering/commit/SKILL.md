---
name: commit
description: Create a git commit in this user's conventional-commit style — `type(TICKET): description :whale:`, scope taken from the Jira ticket in the branch name, commitlint-conventional types, always suffixed with the `:whale:` emoji, no Co-Authored-By line. Trigger on /commit or when the user says "commit this", "commit these changes", "make a commit", "git commit".
---

# Commit

Create a git commit that matches this repo's established convention, enforced by
`commitlint.config.js` (`@commitlint/config-conventional`) via the `.husky/commit-msg` hook.

## Message format

```
<type>(<TICKET>): <description> :whale:
```

- **type** — conventional-commit type: `feat`, `fix`, `refactor`, `test`, `chore`, `docs`, `style`, `perf`, `build`, `ci`. Pick the one matching the change (`fix` for bug fixes, `feat` for new behavior, `refactor` for no-behavior-change restructuring, `test` for test-only changes).
- **(TICKET)** — the Jira ticket ID, e.g. `INS-2512`. Extract it from the current branch name (`feature/INS-2512` → `INS-2512`, `hotfix/INS-2670` → `INS-2670`). If the branch name has no ticket ID, ask the user for one instead of guessing.
- **description** — lowercase, imperative, concise (what changed, not why). No trailing period.
- **`:whale:`** — always append this literal emoji shortcode, every commit, no exceptions. It renders as 🐳 and is this user's signature suffix (confirmed consistent across the entire commit history on this branch).
- **No `Co-Authored-By` trailer.** This repo's commit history never includes one — do not add `Co-Authored-By: Claude ...` or any similar attribution line, even though that is Claude Code's usual default.

Examples from this repo's actual history:
```
feat(INS-2512): update schema validator for birthDate :whale:
fix(INS-2512): update schema transform mapping for SDB :whale:
refactor(INS-2512): extract shared broker guard base class :whale:
test(INS-2511): add unit tests for tqmVerifyPolicy and address name resolution :whale:
```

## Steps

1. **Get branch name** — `git branch --show-current` — to derive `<TICKET>`.
2. **Review changes** — run in parallel: `git status`, `git diff` (staged + unstaged), `git log -5 --oneline` for recent style confirmation.
3. **Decide scope of the commit** — if the user asked to commit "this" or specific files, stage only those. Otherwise stage all modified/untracked files relevant to the current work (never blindly `git add -A`; check `git status` output first for anything unexpected like stray files or secrets).
4. **Stage, then pause for review** — run `git add` for the decided scope only, then show `git status` and `git diff --staged` so the user can recheck exactly what's staged before anything is committed. Do not run `git commit` in the same step as `git add` — wait for the user to confirm (or give a go-ahead) before committing, even if they already said "commit this" up front.
5. **Pick type + write description** from the actual diff content — do not just reuse the branch's Jira title verbatim; describe the specific change.
6. **Commit** using a HEREDOC so formatting is exact, with just the single subject line — no body, no trailer:
   ```bash
   git commit -m "$(cat <<'EOF'
   <type>(<TICKET>): <description> :whale:
   EOF
   )"
   ```
7. **Verify** with `git status` after.

## Rules

- Only commit when the user explicitly asks — never commit proactively.
- Always split `git add` and `git commit` into separate steps, even for a single-file commit requested in the same breath — show the staged diff and let the user recheck before running `git commit`.
- One commit per logical change unless the user asks to bundle everything.
- Do not add a commit body/footer unless the user asks for one — this repo's history is consistently single-line subjects.
- **Never add a `Co-Authored-By` line** — overrides Claude Code's default commit behavior for this repo.
- Never use `--no-verify` (the commitlint hook must run) unless the user explicitly asks to skip it.
- Never amend an existing commit unless the user explicitly asks.
- If `commitlint` rejects the message (hook failure), read the error, fix the format, and retry — don't bypass with `--no-verify`.