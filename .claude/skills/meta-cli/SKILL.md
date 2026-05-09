---
name: meta-cli
description: Use when managing multiple git repositories with the meta CLI, running commands across sub-repos, cloning a meta workspace, branching or committing across all repos, or troubleshooting meta exec expressions. Covers meta exec, meta git, meta project, meta npm/yarn, and filtering options.
---

# meta-cli

## Overview

`meta` manages multiple git repos as if they were one monorepo. A `.meta` file at the root maps local folder paths to remote URLs. Sub-repos are gitignored from the meta repo itself.

```json
{
  "projects": {
    "services/api":  "git@github.com:org/api.git",
    "services/auth": "git@github.com:org/auth.git"
  }
}
```

Install: `npm i -g meta`

---

## Quick Reference

| Goal | Command |
|------|---------|
| Init meta repo | `meta init` |
| Clone everything | `meta git clone <meta-repo-url>` |
| Pull missing repos | `meta git update` |
| Run arbitrary command | `meta exec "<cmd>"` |
| Status all repos | `meta git status` |
| Checkout branch | `meta git checkout <branch>` |
| New branch everywhere | `meta git checkout -b <branch>` |
| Pull all | `meta git pull` |
| Push all | `meta git push` |
| Add sub-repo | `meta project import <folder> <url>` |

---

## meta exec

Run any shell command across all sub-repos.

```bash
meta exec "git log --oneline -10"
meta exec "npm ci" --parallel
meta exec "git status" --include-only services/api,services/auth
meta exec "npm test" --exclude legacy-app
```

**Expression evaluation gotcha:** wrap `$()` and backticks with `\` so they evaluate inside each sub-repo, not in the meta root shell.

```bash
# Wrong — evaluates pwd in meta root
meta exec "echo `pwd`"

# Correct — evaluates pwd in each sub-repo
meta exec "echo \$(pwd)"
meta exec "echo \`pwd\`"
```

---

## meta git

Wraps git commands across all sub-repos.

```bash
meta git status
meta git branch
meta git checkout main
meta git checkout -b feature/my-feature
meta git pull origin main
meta git push origin main
meta git add .
meta git commit -m "feat: message"
meta git checkout .                              # revert modified files
meta git clean -fd                               # delete untracked files
meta git tag v1.0.0
meta git tag -l                                  # list tags
meta git remote -v
meta git remote add upstream git@github.com:org/upstream.git
```

**Clone a meta repo (including all sub-repos)** — already-present repos are skipped, so `meta git update` is idempotent:

**Sync after teammate adds a new sub-repo:**
```bash
git pull origin main    # get updated .meta
meta git update         # clone newly added repos (skips already-cloned ones)
```

---

## meta project

```bash
# Create new empty sub-repo
meta project create <folder> <repo-url>

# Import existing repo
meta project import services/api git@github.com:org/api.git

# Migrate sub-directory from monorepo (preserves git history)
meta project migrate <folder> <repo-url>
```

---

## meta npm / meta yarn

```bash
meta npm install
meta npm clean           # delete node_modules in all repos
meta npm outdated        # check outdated dependencies
meta npm run build
meta npm run test
meta npm publish
meta npm link --all      # symlink packages for local dev (via global link)
meta npm symlink         # direct symlink without global link

meta yarn install
meta yarn link --all
```

---

## Filtering Options

All commands support these flags:

| Flag | Effect |
|------|--------|
| `--include-only dir1,dir2` | Run only on specified dirs |
| `--exclude dir1,dir2` | Skip specified dirs |
| `--parallel` | Run in parallel (output grouped at end); if any sub-repo fails, others still complete — check exit code |

---

## Common Workflows

**Team onboarding:**
```bash
meta git clone git@github.com:org/meta-repo.git
cd meta-repo
meta npm install
```

**Feature branch across all repos:**
```bash
meta git checkout -b feature/new-api
# ... work ...
meta git add .        # stages within each sub-repo; does NOT stage the meta root repo itself
meta git commit -m "feat: add new API"
meta git push origin feature/new-api
```
> `meta git add .` runs `git add .` inside each sub-repo. To stage changes in the meta root repo (e.g., updated `.meta`), run plain `git add .` from the meta root separately.

**Full parallel build:**
```bash
meta exec "npm run build" --parallel
```

**Local library dev with symlinks:**
```bash
meta npm install
meta npm link --all
```

---

## Removing a Sub-repo

`meta project` has no remove command. To remove a sub-repo manually:

```bash
# 1. Remove from .meta file (edit JSON, delete the entry)
# 2. Delete the local directory
rm -rf <folder>
# 3. Commit the updated .meta
git add .meta && git commit -m "chore: remove <folder> from meta"
```

---

## Plugins

`meta` is extended via `meta-*` npm packages. Install as devDependency (recommended) or globally:

```bash
npm install --save-dev meta-npm
npm i -g meta-npm
```

| Plugin | Adds |
|--------|------|
| `meta-git` | `meta git` subcommands |
| `meta-project` | `meta project` subcommands |
| `meta-exec` | `meta exec` |
| `meta-npm` | `meta npm` subcommands |
| `meta-yarn` | `meta yarn` subcommands |
| `meta-gh` | GitHub CLI wrapping |
| `meta-loop` | loop helper functions |

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `meta exec "echo $(pwd)"` prints meta root | Escape: `"echo \$(pwd)"` |
| Sub-repos missing after clone | Run `meta git update` |
| Changes in one repo not showing in `meta git status` | Check that folder is listed in `.meta` |
| `meta git clone` only clones root | Use `meta git clone` (not plain `git clone`) to pull all sub-repos |
| `.meta` update not committed after adding/removing sub-repo | `git add .meta && git commit` from meta root |
