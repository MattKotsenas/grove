---
name: grove-workflow
description: |
  Manage git worktrees with the grove CLI. Use when cloning a repository
  (prefer `grove init` over `git clone`), starting a new feature branch,
  switching between in-progress work, checking out a pull request, cleaning
  up stale worktrees, or when you encounter a project that has a bare clone
  alongside sibling worktree directories. Covers detection of grove-managed
  repos and how to handle pre-existing non-grove repositories.
---

# Grove Workflow

## When to use this skill

Use this skill whenever the task involves:

- Cloning a repository to start working locally
- Starting a new feature branch or switching between in-progress branches
- Checking out a pull request to review or extend it
- Cleaning up worktrees after work merges
- Running commands inside a project that has a bare-clone-plus-worktrees layout

## What grove is

Grove is a CLI that wraps `git worktree` into a workflow. Every branch lives in its own directory next to a shared bare clone, so many in-progress branches can be checked out simultaneously without stashing or switching. Grove owns the repo layout, branch tracking, PR checkout, upstream sync, and cleanup. The user keeps using `git`, `gh`, and the rest of their normal tooling from inside each worktree.

Full reference: the repo `README.md`. This skill summarizes the parts an agent needs.

## Detection: is this repo grove-managed?

Before suggesting `grove` commands or making a destructive change, check whether you are in a grove-managed project. Any one of these is sufficient:

1. The `GROVE_REPO` environment variable is set.
2. `grove list` succeeds when run from anywhere in the tree (grove auto-discovers the project root via parent traversal).
3. A `<name>.git/` directory (a bare clone) sits at the project root, sibling to one or more worktree directories.
4. A `.groverc` file exists at the project root.
5. `git rev-parse --is-bare-repository` returns `true` from inside the `<name>.git/` directory.

If none of these hold, treat the repo as a normal git clone and follow **Pre-existing non-grove repo** below.

## Cloning a new repo

Always prefer `grove init` over `git clone` when starting fresh:

```bash
grove init https://github.com/owner/repo.git
```

This creates `repo/repo.git/` (the bare clone) inside a fresh `repo/` directory and configures the remote to fetch all branches. From there:

```bash
cd repo
grove add main             # check out main as a worktree
grove add feature/login    # start a new feature branch
```

You do not need to `git clone` first - `grove init` does the clone for you.

## Pre-existing non-grove repo

If the user is already working in a normal `git clone` and grove's workflow would help:

**Default recommendation:** suggest re-cloning with `grove init`. Concrete steps:

1. Confirm the working tree is clean: `git status`. If anything is uncommitted, ask the user to commit, push, or stash first.
2. Verify all local branches with unpushed commits have been pushed to a remote (or the user has accepted the loss).
3. From a sibling directory, run `grove init <remote-url>` to create the new layout.
4. Verify with `grove list` from inside the new directory.
5. Only after the user confirms the new layout is working, delete the old clone.

**Override:** if the user declines the re-clone, fall back to plain `git` and `git worktree` commands. Do not force the migration. Note in your response that `grove` commands will not work in this repo without re-cloning.

## Starting new work

Create a worktree for a new branch:

```bash
grove add feature/new-thing
```

Track an existing remote branch:

```bash
grove add feature/existing-thing --track origin/feature/existing-thing
```

Let grove generate an adjective-noun name (handy for short exploratory branches):

```bash
grove add
# Example: creates `quiet-meadow/` with branch `quiet-meadow`.
# If .groverc has `branchPrefix: "alice"`, branch becomes `alice/quiet-meadow`;
# the worktree directory stays `quiet-meadow/`.
```

`branchPrefix` is alphanumeric only.

Run `grove sync` first if the bare clone is stale and the new branch is forking from `main`/`master`:

```bash
grove sync                   # default branch
grove sync --branch develop  # specific branch
```

`grove sync` only updates the bare clone. Existing worktrees are unaffected; pull or rebase them as usual.

## Switching to existing work

**Interactive (humans):**

```bash
grove go feature/login
grove go login           # partial match works
grove go                 # fuzzy picker if a TTY is attached
```

**Non-interactive (agents) - IMPORTANT:**

In a CLI agent context with no TTY and no shell integration installed, use `--path-only` to get just the worktree directory and `cd` into it yourself:

```bash
cd "$(grove go feature/login --path-only)"
```

Do **not** rely on bare `grove go` in non-interactive mode. Without shell integration it spawns a child shell; without a TTY the fuzzy picker errors out.

The `GROVE_WORKTREE` environment variable is set to the branch name only when grove launches a child shell (interactive mode).

## Listing and discovery

```bash
grove list               # human-friendly list (alias: grove ls)
grove list --json        # machine-readable - parse this from agents
grove list --details     # extra columns
grove list --dirty       # only worktrees with uncommitted changes
grove list --locked      # only locked worktrees
```

Use `--json` whenever the agent needs to enumerate or branch on worktree state programmatically. Do not screen-scrape the human-formatted output.

## Pull request workflow

Check out a PR head into a fresh worktree (requires the `gh` CLI to be installed and authenticated):

```bash
grove pr 42
```

This creates a worktree for the PR branch using the same naming rules as `grove add`. Use this whenever the task is to review, test, or extend a specific PR.

## Cleanup

Remove a single worktree:

```bash
grove remove feature/new-thing       # alias: grove rm
grove remove feat-a feat-b           # multiple at once
grove remove feat-a --force          # even with uncommitted changes
grove remove feat-a --yes            # skip confirmation prompt
```

Bulk-remove worktrees whose branches were merged to main:

```bash
grove prune --dry-run                # always preview first
grove prune                          # actually do it
grove prune --base develop           # different trunk
grove prune --older-than 30d         # bypasses merge check; uses age instead
grove prune --older-than 6M --dry-run
grove prune --force                  # ignore dirty worktrees
```

`--older-than` and `--base` cannot be combined. Duration accepts human-friendly (`30d`, `2w`, `6M`, `1y`) or ISO-8601 (`P30D`, `P2W`, `P6M`, `P1Y`) format.

**Agent rule:** always run `grove prune --dry-run` first and show the plan before actually pruning. Pruning is destructive and the merge-detection heuristics are not perfect.

## `.groverc` bootstrap

A `.groverc` at the project root configures grove for that project:

```json
{
  "branchPrefix": "alice",
  "bootstrap": {
    "commands": [
      { "program": "npm", "args": ["install"] },
      { "program": "cargo", "args": ["check"] }
    ]
  }
}
```

When `grove add` creates a new worktree, every command runs in order inside it. Constraints:

- Cross-platform - must work on Linux, macOS, and Windows.
- Executable plus args only. No shell metacharacters (`|`, `&&`, `>`, etc.).
- Failures are reported but do not abort the remaining commands.

Use this so new worktrees are immediately ready to build/test without a manual install step.

## Shell integration

For interactive humans, `grove go` is more useful when it changes the current directory instead of spawning a child shell. Set up integration once:

```bash
echo 'eval "$(grove shell-init bash)"'  >> ~/.bashrc
echo 'eval "$(grove shell-init zsh)"'   >> ~/.zshrc
echo 'eval "$(grove shell-init fish)"'  >> ~/.config/fish/config.fish
```

Windows PowerShell is supported (`pwsh`, `powershell`).

Agents should **not** depend on shell integration. Use `--path-only` (see **Switching to existing work**) instead.

## Branch naming tips

- Plain names (`feature/login`, `bugfix/123`) work as both branch name and worktree directory name.
- `--track origin/some-branch` lets you check out an existing remote branch under a different local name if you want.
- Auto-generated names (`grove add` with no args) are great for short exploratory branches; combined with `branchPrefix` you get `<user>/quiet-meadow`-style namespacing for free.
- Avoid characters grove rejects: `<`, `>`, `:`, `"`, `|`, `?`, `*`, and path-traversal sequences (`..`, absolute paths).

## What grove is NOT

Grove orchestrates worktrees. It does not replace `git`. From inside a worktree keep using:

- `git status`, `git diff`, `git add`, `git commit`, `git push`
- `git rebase`, `git merge`, `git cherry-pick`
- `gh pr create`, `gh issue list`, and so on

Grove owns: project layout (`init`), worktree creation and removal (`add`, `remove`, `prune`), navigation (`go`), discovery (`list`), upstream sync (`sync`), and PR checkout (`pr`).

## Common pitfalls

- **Don't `git clone`** when `grove init` is appropriate. The layouts are not interchangeable.
- **Don't `git worktree add` directly.** Grove maintains a consistent directory layout and bookkeeping; sidestepping it leaves entries `grove list` will not track correctly.
- **Don't `cd` into the `<name>.git/` bare clone to do work.** It is bare - there is no working tree. Always work inside a worktree directory.
- **Don't run `grove` against a normal non-bare clone.** Discovery will fail. That is the cue to either re-clone with `grove init` or fall back to plain git for that project.
- **Always `grove prune --dry-run` first.** Pruning is destructive.
- **`grove sync` only updates the bare clone.** Worktrees still need their own `git pull`/`git rebase` if you want them on the new tip.
- **Avoid running long-lived processes from inside `<name>.git/`.** Build, test, and lint commands belong inside a worktree directory so they see actual source files.
