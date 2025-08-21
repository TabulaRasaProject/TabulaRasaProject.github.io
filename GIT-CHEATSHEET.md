# Git Cheatsheet — `roomba-gpt` + `gpt-oss` Submodule

> Quick, safe Git workflows for this repo. Keep it simple, prevent 404s, avoid force-push pain.

---

## Repo Layout (mental model)
- **Parent repo:** `roomba-gpt` (this repo)
- **Submodule:** `mind/gpt-oss` → a separate Git repo pinned at an exact commit SHA
- The parent repo stores **only the submodule’s commit SHA**, not its files.

---

## One-time Setup (SSH everywhere)
```bash
# Parent repo (roomba-gpt)
git remote -v
# If not SSH, switch it:
git remote set-url origin git@github.com:TabulaRasaProject/roomba-gpt.git

# Submodule (gpt-oss)
git -C mind/gpt-oss remote -v
# If not SSH, switch it:
git -C mind/gpt-oss remote set-url origin git@github.com:TabulaRasaProject/gpt-oss.git

# Ensure future clones use SSH for the submodule too
git config -f .gitmodules submodule.mind/gpt-oss.url git@github.com:TabulaRasaProject/gpt-oss.git
git submodule sync --recursive
```

**SSH key check**
```bash
ssh -T git@github.com
# Expect: "Hi <you>! You've successfully authenticated, but GitHub does not provide shell access."
```

---

## Daily Flow (parent repo)
```bash
git status                                  # what changed
git pull --rebase origin main                # update your local branch
# ...edit files in the parent repo...
git add -A
git commit -m "<message>"
git push
```

**Feature branches (recommended)**
```bash
git checkout -b feat/<short-name>
# work, commit, push
git push -u origin feat/<short-name>
```

---

## Submodule Workflow (safe pattern)
> Golden rule: **push the submodule first, then bump the pointer in the parent**.

```bash
# 1) Work INSIDE the submodule
cd mind/gpt-oss
# ...make changes...
git add -A
git commit -m "<gpt-oss change>"
git push

# 2) Record the new submodule commit in the parent
cd ../..
git add mind/gpt-oss
git commit -m "chore: bump gpt-oss submodule pointer"
git push
```

**Why?** The parent’s blue link on GitHub needs that exact submodule commit to exist on the remote. If it doesn’t, users see a 404.

---

## Handling "modified: mind/gpt-oss (modified content)"
This means there are uncommitted or untracked files **inside** the submodule.

**Option A — Keep & publish**
```bash
git -C mind/gpt-oss add -A
git -C mind/gpt-oss commit -m "chore: ignore artifacts / docs update"
git -C mind/gpt-oss push

git add mind/gpt-oss
git commit -m "chore: bump gpt-oss submodule pointer"
git push
```

**Option B — Revert to recorded state (⚠️ deletes untracked stuff like .venv)**
```bash
git -C mind/gpt-oss reset --hard HEAD
git -C mind/gpt-oss clean -fdx
git submodule update --init --recursive
```

**Option C — Silence local noise in parent (use sparingly)**
```bash
git config -f .gitmodules submodule.mind/gpt-oss.ignore dirty
git submodule sync --recursive
git add .gitmodules && git commit -m "chore: ignore local dirty state for gpt-oss" && git push
```

---

## Preventing 404s on Submodule Links (what bit us before)
404 happens when the parent points at a submodule commit that **GitHub can’t see**.

**Causes**
1) Submodule commit isn’t pushed yet (or history was force-updated).
2) Submodule repo is private (public will 404).
3) URL mismatch / dead path.

**Fixes**
- Push the submodule first → then bump parent.
- Avoid force-pushing `main`. If history must change, keep old SHAs reachable:

```bash
# Inside submodule, preserve older SHAs so the parent’s old pointers still work
old=e662dbf
git branch archive/$old $old
git tag keep-$old $old
git push origin archive/$old keep-$old
```

- Prefer **tags** for stable public pointers: e.g., `gpt-oss@v0.1.0`.

---

## Pre-push Guard (local hook)
Block pushing the parent if it points at a submodule SHA that the remote doesn’t know.

**.git/hooks/pre-push** (make executable with `chmod +x .git/hooks/pre-push`)
```bash
#!/usr/bin/env bash
set -euo pipefail
sub="mind/gpt-oss"
url=$(git config -f .gitmodules submodule.$sub.url || true)
[ -z "${url:-}" ] && exit 0
sha=$(git rev-parse :$sub 2>/dev/null || true)
[ -z "${sha:-}" ] && exit 0
if ! git ls-remote "$url" | grep -qi "^$sha"; then
  echo "❌ Submodule $sub points at $sha which is NOT on remote $url" >&2
  echo "Push the submodule commit first (or preserve it with a branch/tag), then bump the pointer." >&2
  exit 1
fi
```

---

## CI Gate (GitHub Actions)
Fail PRs that reference unknown submodule SHAs.

**.github/workflows/check-submodules.yml**
```yaml
name: Check submodule SHAs
on: [pull_request]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: false
      - name: Verify submodule SHAs exist on remotes
        run: |
          set -euo pipefail
          git config -f .gitmodules -l | grep '\.url=' | while read -r line; do
            path=$(echo "$line" | sed -E 's/^submodule\.([^=]+)\.url=.*/\1/')
            url=$(echo "$line" | cut -d= -f2-)
            sha=$(git rev-parse ":$path")
            echo "Checking $path ($url) @ $sha"
            git ls-remote "$url" | grep -qi "^$sha" || { echo "Missing $sha in $url"; exit 1; }
          done
```

---

## Undo & Recovery
```bash
# discard unstaged changes in a file
git restore <file>

# unstage a file
git restore --staged <file>

# drop the last commit (destructive)
git reset --hard HEAD~1

# see a nice graph of branches and commits
git log --oneline --graph --decorate --all
```

---

## Quick Glossary
- **Commit:** Snapshot + message.
- **Branch:** Movable pointer to a commit (e.g., `main`).
- **Remote:** Another copy of the repo (GitHub).
- **Submodule:** A repo embedded by reference; parent tracks only the **SHA**.

---

## Contributor TL;DR
1. **SSH only** (no passwords/tokens).
2. **Push submodule first** → then `git add mind/gpt-oss && git commit && git push` in parent.
3. **No force-push to `main`**.
4. If submodule looks dirty: commit/push inside it or reset/ignore per above.

— TabulaRasaRobotics | Athera Roomba
