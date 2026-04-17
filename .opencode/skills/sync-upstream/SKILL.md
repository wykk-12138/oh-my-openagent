---
name: sync-upstream
description: "Sync this local customized oh-my-openagent repo with the latest official upstream code while preserving local modifications. Triggers: 'sync upstream', 'update from upstream', 'merge upstream', 'pull latest upstream'."
---

# Sync Upstream — Preserve Local Modifications

<role>
Safely update this local customized oh-my-openagent checkout from the official upstream repository while preserving local modifications.
</role>

## Goal

Bring the current folder up to date with the latest official upstream code from `code-yeongyu/oh-my-openagent` and keep existing local customizations intact.

## Scope Rules

- Update the current repository only.
- Preserve local modifications instead of overwriting them.
- Prefer the safest path: stash tracked local changes, fast-forward or merge upstream, then re-apply local changes.
- Do not commit unless the user explicitly asks.
- Do not push unless the user explicitly asks.

## Workflow

### 1. Verify repository state

Run these checks first:

```bash
GIT_MASTER=1 git remote -v
GIT_MASTER=1 git branch --show-current
GIT_MASTER=1 git status --short
```

Confirm that this repo points at the official upstream repository. In this project, `origin` is usually the official upstream.

### 2. Preserve local modifications

If tracked local changes exist, stash them before syncing upstream:

```bash
GIT_MASTER=1 git stash push -m "pre-upstream-sync"
```

Leave untracked runtime artifacts alone unless they interfere.

### 3. Fetch latest official upstream code

```bash
GIT_MASTER=1 git fetch origin --prune
```

### 4. Update the local base branch

This project normally tracks `dev`.

```bash
GIT_MASTER=1 git checkout dev
GIT_MASTER=1 git merge --ff-only origin/dev
```

If fast-forward is not possible, stop and inspect why before continuing.

### 5. Re-apply local modifications

```bash
GIT_MASTER=1 git stash pop
```

If conflicts appear, resolve them by keeping the upstream update and restoring the intended local customizations.

### 6. Verify the merged result

Run the project validations that matter for local changes:

```bash
bun run typecheck
bun run build
```

If tests are needed, run them, but distinguish pre-existing upstream failures from new failures introduced by local customizations.

## Safety Rules

- Never use `git reset --hard` unless the user explicitly asks.
- Never drop a stash automatically if re-apply fails.
- Never overwrite local customizations silently.
- Never commit the result unless the user explicitly asks.

## Expected Outcome

After the workflow:

- local `dev` includes the latest official upstream code
- local customizations are still present in the working tree
- upstream changes and local changes coexist in the current folder

## Notes for This Repository

- Official upstream is `code-yeongyu/oh-my-openagent`
- Base branch is typically `dev`
- This workflow is specifically for updating the local customized checkout in place
