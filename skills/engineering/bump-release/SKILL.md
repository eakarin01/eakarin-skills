---
name: bump-release
description: Create a release or hotfix branch with a bumped semantic version. Reads package.json, stashes WIP, pulls from the correct source branch (master for hotfix, develop for regular release), bumps major/minor/patch, checks remote for existing rc branches, creates the branch, commits, pushes, then returns to the original branch and restores the stash. Trigger on /bump-release or when the user says "bump version", "create release branch", "bump release", "bump hotfix", "สร้าง release branch", or any similar phrasing.
---

# Bump Release

## Step 1 — Gather intent

Determine from the user's request:

1. **Release type**: `hotfix` or `release` (regular feature release)
   - If the user says "hotfix" or "bugfix" or "fix" → `hotfix`
   - Otherwise → `release`

2. **Version bump type**: `major`, `minor`, or `patch`
   - If user explicitly states `major`, `minor`, or `patch` → use that
   - Else if type is `hotfix` or `bugfix` → `patch`
   - Else if type is `release` (feature) → `minor`

If either is still ambiguous after reading the request, ask the user before proceeding. Do not guess.

---

## Step 2 — Record current branch (WIP location)

Before touching anything, record the current branch so we can return to it later:

```bash
git branch --show-current
```

Save this as `<original-branch>`.

---

## Step 3 — Stash all changes including untracked

Stash everything — tracked, untracked, and staged files — so the working tree is clean:

```bash
git stash push -u -m "WIP before bump-release"
```

Note whether anything was actually stashed (git will say "No local changes to save" if nothing was stashed). Remember this so we know whether to pop at the end.

---

## Step 4 — Pull latest from source branch

Source branch depends on release type:
- `hotfix` → pull from `master`
- `release` → pull from `develop`

Checkout the source branch first, then pull:

```bash
git checkout <source-branch>
git pull origin <source-branch>
```

If the pull fails (conflict, diverged), stop, return to `<original-branch>`, pop the stash, and report the error to the user.

---

## Step 5 — Read and compare versions from source branch and master

Read the version from the source branch (already checked out) and also from `origin/master` without switching branches:

```bash
# Version on source branch
node -e "const p=require('./package.json'); console.log(p.version);"

# Version on master (without checkout)
git show origin/master:package.json | node -e "const chunks=[]; process.stdin.on('data',c=>chunks.push(c)); process.stdin.on('end',()=>console.log(JSON.parse(chunks.join('')).version));"
```

Compare the two versions using semver ordering. Use the **higher** of the two as the base for the bump. If master has a higher version than the source branch, warn the user that master is ahead and report both versions before proceeding.

---

## Step 6 — Calculate new version

Apply semantic versioning to the current version:
- `major`: increment major, reset minor and patch to 0 (e.g. `1.2.3` → `2.0.0`)
- `minor`: increment minor, reset patch to 0 (e.g. `1.2.3` → `1.3.0`)
- `patch`: increment patch only (e.g. `1.2.3` → `1.2.4`)

Do NOT use `npm version` command — compute the new version string manually to avoid unwanted commits or tags by npm.

---

## Step 7 — Determine release candidate number

Check remote branches to find if a branch for this version already exists:

```bash
git fetch --prune
git branch -r | grep '<prefix>/v<new-version>-rc'
```

Where `<prefix>` is `release` or `hotfix`.

- If no matching branch exists → use `rc1`
- If branches exist (e.g. `release/v1.3.0-rc1`, `release/v1.3.0-rc2`) → use the next number (e.g. `rc3`)

---

## Step 8 — Create the new branch

Branch name format: `<prefix>/v<new-version>-rc<n>`

Examples:
- `release/v1.3.0-rc1`
- `hotfix/v1.2.4-rc2`

```bash
git checkout -b <branch-name>
```

---

## Step 9 — Bump version in package.json

Use `npm pkg set` to update the version field cleanly:

```bash
npm pkg set version=<new-version>
```

Verify the change:
```bash
node -e "const p=require('./package.json'); console.log(p.version);"
```

---

## Step 10 — Commit

Stage only `package.json`:

```bash
git add package.json
git commit -m "chore: bump version to <new-version>"
```

---

## Step 11 — Push to remote

**NEVER use `--force` or `--force-with-lease`.**

```bash
git push -u origin <branch-name>
```

If push fails for any reason, report the error and stop. Do not retry with force.

---

## Step 12 — Return to original branch and restore stash

Go back to where the user was before:

```bash
git checkout <original-branch>
```

If something was stashed in Step 3, pop it:

```bash
git stash pop
```

If `stash pop` results in a conflict, report it clearly to the user and do not resolve it automatically.

---

## Step 13 — Report result

After completing all steps, report a summary to the user:

```
Release branch created successfully.

  Type        : <release|hotfix>
  Bump        : <major|minor|patch>
  Version     : <old-version> → <new-version>
  Branch      : <branch-name>
  Source      : <source-branch>
  Candidate   : rc<n>
  Returned to : <original-branch>
```

---

## Error handling

- If `package.json` is not found in the current directory → stop and tell the user
- If pull fails → checkout `<original-branch>`, pop stash, report error
- If push fails → report error, do not attempt force operations
- If `stash pop` has conflicts → report clearly, leave resolution to the user
