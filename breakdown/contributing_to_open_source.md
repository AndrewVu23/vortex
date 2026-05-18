# Contributing to an Open Source Project — a Practical Guide

## Why isn't it just `git add / commit / push`?

When you work on a personal repo, `git push` writes directly to the project's only copy. That works because *you own it*.

Open source projects are different in two ways:

1. **You don't have write access.** The project lives on someone else's GitHub account (or org). They can't hand commit rights to every drive-by contributor — code would land unreviewed, untested, and sometimes malicious.
2. **Maintainers want to review changes before they land.** Even trusted contributors push to a *proposal* first, not the real branch, so others can look at it.

The whole fork/branch/PR dance exists to solve both problems at once. Once you've done it twice it stops feeling complex — it's the same four commands every time.

## The mental model

There are **three copies of the repo** to keep straight:

```
        upstream                       origin                      working
   (the real project)              (your fork)              (your laptop)
github.com/vortexgpgpu/         github.com/<you>/             ~/Projects/
       vortex                       vortex                       vortex
        │                             ▲   │                        ▲
        │ you can read it             │   │                        │
        │ but never write             │   │ you push here          │ you edit here
        │                             │   │                        │
        └──────  fetch  ──────────────┼───┴─────  pull  ───────────┘
                                      │
                                      │  Pull Request
                                      └────────────────► back to upstream
```

- **upstream** — the canonical project. Read-only for you.
- **origin** — your personal copy on GitHub (a "fork"). You own it; you can push freely.
- **working copy** — the clone on your laptop. This is what you edit.

A Pull Request is a request that says: *"Hey upstream maintainers, please pull these commits from my fork into your master branch."* They review, maybe ask for changes, then click Merge.

## One-time setup

Do this once per project. After that, every contribution skips straight to the "per-contribution" section.

**1. Fork on GitHub.** Open the project page and click **Fork** (top right). GitHub copies the repo to your account. Takes about three seconds.

**2. Point your local clone at both remotes.** If you already cloned upstream (which is what most people do first), rename your remotes so `origin` is your fork and `upstream` is the project:

```bash
git remote rename origin upstream
git remote add origin https://github.com/<your-username>/vortex.git
git remote -v   # sanity check: origin → your fork, upstream → vortexgpgpu
```

If you haven't cloned yet, the cleanest order is: fork first, then `git clone https://github.com/<you>/vortex.git`, then `git remote add upstream https://github.com/vortexgpgpu/vortex.git`.

That's it for setup.

## Per-contribution flow

Every time you want to contribute something — a typo fix, a feature, a bug fix — you do these five steps:

**1. Sync with upstream.** Get the latest from the real project before starting:

```bash
git checkout master
git fetch upstream
git merge upstream/master
```

This makes sure you're branching off the current state, not a stale one. If you skip this, you'll eventually hit merge conflicts.

**2. Make a branch.** One branch per logical change. Name it after what it does:

```bash
git checkout -b fix-cpp-guidelines-typo
```

Good names: `fix-cache-coalescing-bug`, `add-bf16-support`, `docs-clarify-mshr`. Bad names: `patch-1`, `changes`, `mywork`.

**3. Edit, commit, repeat.** Normal Git:

```bash
# edit files…
git add docs/coding_guidelines_cpp.md
git commit -m "docs: fix indentation in C++ ifdef example"
```

Make as many commits as you need. They all roll into one PR.

**4. Push to your fork.**

```bash
git push -u origin fix-cpp-guidelines-typo
```

`-u` only matters the first time on a branch — it sets up tracking so future `git push` needs no arguments.

This pushes to **your fork** (origin), not upstream. You'd get a permission error if you tried upstream — that's by design.

**5. Open the PR on GitHub.**

After pushing, GitHub prints a URL in the terminal output:

```
remote: Create a pull request for 'fix-cpp-guidelines-typo' on GitHub by visiting:
remote:   https://github.com/<you>/vortex/pull/new/fix-cpp-guidelines-typo
```

Click that link. Or manually: go to the upstream repo on GitHub → **Pull requests** → **New pull request** → choose:

- **base repository**: `vortexgpgpu/vortex`  **base branch**: `master`
- **head repository**: `<your-username>/vortex`  **compare branch**: `fix-cpp-guidelines-typo`

Write a title and a short description (what changed, why), then click **Create pull request**.

That's the whole flow. Five steps, takes a minute once you've memorized it.

## After the PR is open

- **CI runs automatically.** Look at the **Checks** tab on the PR. Green = good. Red = click in and read the log.
- **Reviewers leave comments.** You don't argue and you don't take it personally. If they want changes, you make the changes in your local branch, commit, and `git push` — the PR auto-updates with the new commits.
- **Merging is up to the maintainers.** When everything's green and reviewers approve, an admin clicks Merge. Your code is now part of the project.

## Common gotchas

**"Permission denied" on `git push`.** You're pushing to upstream instead of origin. Check `git remote -v`. Origin should be your fork.

**"Updates were rejected because the remote contains work that you do not have locally."** Someone (probably you, on GitHub) pushed to that branch from elsewhere. Run `git pull --rebase` then push again.

**Merge conflicts in the PR.** Upstream master moved while your PR was open. Sync and rebase:

```bash
git checkout fix-cpp-guidelines-typo
git fetch upstream
git rebase upstream/master
# resolve conflicts if Git asks
git push --force-with-lease
```

`--force-with-lease` is safer than `--force` — it refuses to overwrite if someone else pushed to your branch since you last fetched.

**You committed to master instead of a branch.** Easy fix before pushing:

```bash
git branch fix-cpp-guidelines-typo   # save your commits on a new branch
git reset --hard upstream/master     # rewind master back to clean state
git checkout fix-cpp-guidelines-typo # continue on the branch
```

**Your PR has 47 commits and they're all messy.** Squash before merging — most projects appreciate it. `git rebase -i upstream/master`, mark the small commits as `squash` or `fixup`, save. Force-push the result. Most maintainers will also offer "Squash and merge" on their end, so you don't *have* to do this.

## Why the convention is universal

Every major open source project on GitHub — Linux, Node, React, anything — uses some version of this flow. Once you've done it on one project, you've done it on all of them. The commands don't change; only the upstream URL does.

The reason it feels like a lot the first time is that you're learning the social model of open source (forks, reviews, gatekeepers) at the same time you're learning the mechanical steps. After your second PR, the steps will feel automatic and you'll be thinking about the *code*, not the workflow.

## Vortex-specific notes

- Direct push to `vortexgpgpu/vortex` is blocked for non-admins — see [docs/contributing.md](../docs/contributing.md). The fork-and-PR flow is mandatory.
- CI runs Verilator builds and the simulator. Doc-only PRs almost always pass instantly; RTL changes can take a while.
- Style is enforced by the guidelines in [docs/coding_guidelines_verilog.md](../docs/coding_guidelines_verilog.md) and [docs/coding_guidelines_cpp.md](../docs/coding_guidelines_cpp.md). Match the surrounding code.
- A good low-stakes first PR: fix the indentation in the C++ `#ifdef` example at [coding_guidelines_cpp.md:77-83](../docs/coding_guidelines_cpp.md#L77-L83) (it uses 4 spaces in a doc that mandates 2, plus has a stray `endfunction` left over from the Verilog version).
