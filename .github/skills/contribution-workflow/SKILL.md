---
name: contribution-workflow
description: "Contribution workflow for Mantella. Use when: creating feature branches, syncing with upstream, preparing PRs, managing git history. Covers: branch strategy, WIP commits, interactive rebase, squashing, pushing to fork, opening PRs to upstream. Trigger words: PR, pull request, upstream, fork, branch, rebase, squash, sync, contribute, merge request."
---

# Mantella Contribution Workflow

## Remotes

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `https://github.com/art-from-the-machine/Mantella.git` | Upstream (read-only) |
| `fork` | `https://github.com/felipe-rojo/Mantella.git` | Your fork (read/write) |

## Branch Strategy

```
main (tracks origin/main — always clean)
  │
  ├── feature/tts-piper-retry-fix     ← bite-sized PR
  ├── feature/stt-vad-tuning          ← bite-sized PR
  ├── feature/llm-new-parser          ← bite-sized PR
  └── feature/ui-settings-tab         ← bite-sized PR
```

**Rules:**
- One feature per branch
- Branch from `main`, never from another feature branch
- Keep `main` synced with `origin/main`
- Delete feature branches after PR merges

## Workflow

### 1. Sync main with upstream

```bash
git fetch origin
git checkout main
git rebase origin/main
git push fork main --force
```

### 2. Create a feature branch

```bash
git checkout main
git checkout -b feature/<descriptive-name>
```

### 3. Work — commit freely with WIP messages

```bash
git add -A
git commit -m "wip: trying thing"
git commit -m "wip: fix that"
```

### 4. Before PR: clean up history

**Option A — Interactive rebase (recommended):**
```bash
git rebase -i main
# squash/fixup WIP commits into logical units
```

**Option B — Soft reset (simplest):**
```bash
git reset main
git add -A
git commit -m "feat: concise description of the change"
```

### 5. Push and open PR

```bash
git push fork feature/<name>
# Open PR: fork/feature-branch → origin/main
```

### 6. After PR merges

```bash
git checkout main
git fetch origin
git rebase origin/main
git push fork main --force
git branch -d feature/<name>
```

## Commit Message Format

Use Conventional Commits for the final squashed commit:

```
feat: add VAD tuning to STT config
fix: handle Piper subprocess crash on empty input
refactor: extract sentence parsing to dedicated module
docs: update TTS provider setup guide
```

## Key Rules

| Rule | Why |
|------|-----|
| One feature per branch | Bite-sized PRs get reviewed faster |
| Branch from `main` only | Avoids dependency chains |
| Rebase, don't merge | Clean linear history |
| Squash WIP before PR | Upstream doesn't need debugging steps |
| Keep `main` synced | Avoids merge conflicts |
