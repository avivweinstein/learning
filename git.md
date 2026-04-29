# Git

Personal learning notes on Git. Starting point: comfortable with `add` / `commit` / `push` / `merge` at a high level. Goal: actually understand what's happening underneath, and learn the operations a solid SWE is expected to know.

This is a living doc — append new questions and answers as they come up.

---

## 1. The mental model (this is the whole game)

If you only internalize one thing about Git, make it this: **Git is a graph of commits. Everything else is a pointer or a tool for moving commits around in that graph.**

### A commit is a snapshot, not a diff

A commit is a full snapshot of every tracked file at a point in time. Internally it stores:

- A pointer to a **tree** (the root directory snapshot — which itself points to subtrees and file blobs).
- A pointer to its **parent commit(s)** (zero for the first commit, one for normal commits, two for merges).
- Author, committer, timestamp, message.
- A SHA-1 hash that uniquely identifies all of the above.

Git *displays* commits as diffs because that's what humans want to see, but the underlying data is snapshots. This is why Git is so good at things like cherry-picking — it can compute the diff between any two snapshots on demand.

### Branches are just labels on commits

A branch is a file in `.git/refs/heads/<branchname>` containing a single commit SHA. That's it. Creating a branch is essentially free — it's writing 41 bytes to disk.

`HEAD` is another pointer that usually points *to a branch* (e.g. it contains the text `ref: refs/heads/main`). When you commit on `main`, Git advances both `HEAD`'s target (`main`) and stays pointing at it.

When `HEAD` points directly at a commit instead of a branch, you're in **detached HEAD** state. Commits you make here aren't on any branch and can be lost if you check out elsewhere — though `git reflog` (see below) usually saves you.

### Tags vs branches

- **Branch**: a moving pointer. Advances when you commit.
- **Tag**: a stationary pointer. Used for releases (`v1.0.0`). Doesn't move.

---

## 2. Terminology

Definitions grouped by topic. Skim once, refer back as needed. The mental model in §1 explains *why* these things exist; this section is the crisp lookup.

### Repository structure

- **Repository (repo)** — A directory tracked by Git. Contains your project files plus a hidden `.git/` subdirectory holding all the history and metadata. Cloning a repo copies both.
- **Working tree (working directory)** — The actual files on disk that you edit. The "checked out" version of one specific commit, plus any local changes you've made on top.
- **Index (a.k.a. staging area, cache)** — A middle layer between your working tree and the repo. A snapshot of what *will* go into the next commit. `git add` puts changes here; `git commit` turns it into a commit.
- **HEAD** — A pointer to the commit you currently have checked out. Almost always points indirectly via a branch (e.g. `HEAD → main → <some commit>`). When it points directly at a commit, you're in **detached HEAD** state.

### Commits and the object graph

- **Commit** — A snapshot of every tracked file at a point in time, plus metadata (author, timestamp, message, and a pointer to the parent commit(s)). Identified by a SHA hash. Commits are immutable — "editing" a commit really creates a new one with a new SHA.
- **Diff** — The set of changes between two snapshots. Could be commit-vs-commit, working-tree-vs-index, branch-vs-branch, etc. Shown as added (`+`) and removed (`-`) lines in each affected file. Git computes diffs on demand from snapshots — they're not stored.
- **Patch** — A diff in a portable text format (a `.patch` or `.diff` file) that can be applied with `git apply` to recreate those changes elsewhere. A patch is just a serialized diff plus enough context to be reapplied.
- **Hunk** — A contiguous block of changes within a diff — a few changed lines plus surrounding context. `git add -p` lets you stage hunk-by-hunk.
- **Tree** — Internal Git object representing a directory: a list of names mapped to blobs (files) and other trees (subdirectories). A commit points to one root tree.
- **Blob** — Internal Git object representing the raw contents of a single file. No filename, no permissions — just bytes and a SHA. Two files with identical contents share one blob.
- **SHA / hash** — The 40-character hex ID of any Git object (commit, tree, blob, tag). Practically unique. You can usually abbreviate to the first 7+ characters when there's no ambiguity.
- **Parent** — The commit(s) directly before another in history. Most commits have one parent; merge commits have two; the very first commit has none.
- **Ancestor** — Any commit reachable by walking parents back from a given commit.

### Refs (named pointers to commits)

- **Ref** — A name that points to a commit. Branches and tags are both refs. Stored as plain text files under `.git/refs/`.
- **Branch** — A *movable* ref. Advances to the new commit each time you commit while it's checked out. Created with `git branch <name>` or `git checkout -b <name>`.
- **Tag** — A *stationary* ref, used for marking releases (`v1.0.0`). Lightweight tags are just a name pointing at a commit; annotated tags (`git tag -a`) carry their own metadata (message, author, date) and can be GPG-signed.
- **Remote-tracking branch** — A local ref like `origin/main` that mirrors where a remote branch was at last fetch. Updated by `git fetch`. You don't commit on it directly.
- **Upstream** — The remote branch your local branch is configured to track. Set with `git push -u origin <branch>` or `git branch --set-upstream-to`. Determines what bare `git push` and `git pull` do.

### Remotes and collaboration

- **Remote** — A named URL for another copy of the repo (typically on a server like GitHub). List them with `git remote -v`.
- **Origin** — Conventional default name for the remote you cloned from. Not magic — just a name. A repo can have many remotes (`upstream`, `fork`, etc.).
- **Clone** — A full local copy of a remote repo: every commit, every branch, every tag. `git clone <url>`.
- **Fork** — A platform-level concept (GitHub/GitLab), *not* a Git command. Creates your own server-side copy of someone else's repo so you can push to it without write access to the original.
- **Pull request (PR) / Merge request (MR)** — A platform feature for proposing that one branch be merged into another, with code review and discussion. Lives on GitHub/GitLab/Bitbucket; not a Git concept.
- **Fetch** — Download new commits and refs from a remote into your local `origin/*` tracking refs. Doesn't change your branches or working tree.
- **Pull** — `fetch` + `merge` (or `rebase`) the matching upstream branch into your current branch. A convenience that bundles two operations.
- **Push** — Send your local commits to a remote, advancing the corresponding remote branch.

### Comparing and integrating changes

- **Conflict** — When Git can't automatically combine two diverging changes to the same lines and asks you to resolve manually. Marked in the affected files with `<<<<<<<`, `=======`, `>>>>>>>` markers.
- **Merge** — Combining two branches' histories. Produces a merge commit with two parents — unless a fast-forward is possible.
- **Fast-forward** — A "merge" that's really just sliding the branch pointer forward, because the target branch hasn't moved since the source diverged. No new merge commit; history stays linear.
- **Merge commit** — A commit with two or more parents, recording the integration of branches. Has no diff of its own; its parents capture both sides.
- **Rebase** — Replay a series of commits onto a different base commit, producing *new* commits with the same changes but different SHAs. Rewrites history.
- **Cherry-pick** — Apply a single commit from one branch onto another as a new commit.
- **Squash** — Combine multiple commits into one. Often done during interactive rebase, or as a merge strategy ("Squash and merge").
- **Amend** — Replace the most recent commit (`git commit --amend`) — typically to fix the message or include a forgotten file. Creates a new commit with a new SHA; safe only if you haven't pushed.
- **Revert** — Create a *new* commit that undoes a prior one. Safe on shared history because it doesn't rewrite anything.

### State management and recovery

- **Checkout** — The classic verb for switching branches *or* restoring files. Overloaded; modern Git splits it into `switch` (branches) and `restore` (files).
- **Switch** — Change the current branch (`git switch <branch>`). Newer, narrower replacement for `git checkout <branch>`.
- **Restore** — Discard working-tree changes or unstage files (`git restore <file>`, `git restore --staged <file>`). Newer, narrower replacement for `git checkout -- <file>`.
- **Stash** — A stack of saved-but-uncommitted changes you can park and reapply later. `git stash`, `git stash pop`.
- **Reflog** — A *local* log of every time HEAD moved (commit, checkout, reset, rebase, merge). Even commits you "lost" via a bad reset are usually still reachable via reflog for ~90 days. The recovery tool of last resort.
- **Detached HEAD** — When HEAD points directly at a commit instead of a branch. Commits made here aren't on any branch and can be lost. To save them: `git switch -c <new-branch>`.

---

## 3. The three areas

Every file in a Git repo lives in (up to) three places at once:

| Area              | What it is                                       | Command to inspect          |
| ----------------- | ------------------------------------------------ | --------------------------- |
| Working directory | The actual files on disk you edit                | `ls`, your editor           |
| Index / staging   | A snapshot of what *will* go in the next commit  | `git diff --staged`         |
| HEAD / repo       | The last commit on the current branch            | `git show HEAD`             |

The flow:

```
edit files  --git add-->  index  --git commit-->  HEAD
   (working dir)         (staged)                (committed)
```

Things that confused me before, now obvious in this model:

- `git diff` (no args) → working dir vs. index (what you've edited but not staged).
- `git diff --staged` → index vs. HEAD (what you've staged but not committed).
- `git diff HEAD` → working dir vs. HEAD (everything not committed).
- `git restore <file>` → throw away working-dir changes; copy from index.
- `git restore --staged <file>` → unstage; copy from HEAD into index.

---

## 4. Refs, HEAD, and how to refer to commits

You'll see a lot of cryptic syntax in Git docs. It's all just ways to name a commit.

| Syntax           | Meaning                                                  |
| ---------------- | -------------------------------------------------------- |
| `<sha>`          | The commit with that SHA (first 7 chars usually enough)  |
| `HEAD`           | The current commit                                       |
| `HEAD~`          | Parent of HEAD (same as `HEAD~1`)                        |
| `HEAD~3`         | Three commits back, following first parent each time     |
| `HEAD^`          | First parent of HEAD (matters for merges)                |
| `HEAD^2`         | Second parent (the merged-in branch)                     |
| `main`           | The commit that branch `main` points at                  |
| `origin/main`    | The commit your local copy of remote `main` points at    |
| `main@{2.days.ago}` | Where `main` pointed 2 days ago (uses reflog)          |
| `<branch>..<branch>` | Range: commits in the second branch but not the first |

`origin/main` deserves special mention. It is **not** the live state of `main` on the remote — it's your *local cache* of where `main` was on the remote the last time you ran `git fetch` (or any operation that fetches). To update it, run `git fetch`.

---

## 5. Rebase vs merge — the big one

You have a feature branch off `main`. While you've been working, `main` has moved forward. You want your branch to incorporate the new `main` work. There are two ways.

### Setup (used in both diagrams)

```
              C---D---E   feature
             /
A---B---F---G   main
```

You branched off `B`. `main` has since advanced to `G`.

### Merge: preserves history exactly as it happened

`git checkout feature && git merge main`

```
              C---D---E---M   feature
             /           /
A---B---F---G-----------    main
```

A new **merge commit** `M` is created with two parents (`E` and `G`). The graph forks and rejoins — true history.

- ✅ Non-destructive. Existing commits keep their SHAs. Safe on shared branches.
- ✅ Records when integration actually happened.
- ❌ Lots of merge commits clutter `git log`.
- ❌ "Bubbles" in history can be hard to read.

### Rebase: rewrites your branch as if you'd started from the new main

`git checkout feature && git rebase main`

```
                          C'--D'--E'   feature
                         /
A---B---F---G   main ---'
```

Git takes your commits `C, D, E`, sets them aside, fast-forwards `feature` to `G`, and then **replays** each commit on top one at a time, producing *new* commits `C', D', E'` (different SHAs — same changes, new parents).

- ✅ Linear history. `git log` reads top-to-bottom like a story.
- ✅ Each replayed commit can be re-tested individually.
- ❌ **Rewrites history.** New SHAs. Anyone who pulled the old commits now has divergent history.
- ❌ Conflicts are resolved per-commit, not all at once. Can be tedious.

### The rule

> **Rebase commits that only exist locally (or only on your private branch). Merge when integrating shared history.**

The reason: if you rebase commits that someone else has already pulled, when they next try to merge, Git will see the original commits in their history and re-introduce them alongside your rewritten ones. Mess.

In practice, most teams settle on one of:

1. **Rebase your feature branch onto main, then merge into main** (often as a fast-forward or squash) — gives a linear main.
2. **Merge main into your feature branch periodically, then merge feature into main** — preserves all the bubbles.
3. **Squash-merge** (see §7) — collapses the whole branch into one commit on main.

### Interactive rebase: `git rebase -i`

This is the killer feature people miss. `git rebase -i HEAD~5` opens an editor showing the last 5 commits and lets you:

- `pick` — keep as is
- `reword` — keep the change, edit the message
- `edit` — pause to amend the commit (split it, add files, etc.)
- `squash` — combine into the previous commit (keeps both messages for editing)
- `fixup` — like squash but discards this commit's message
- `drop` — delete the commit entirely
- Reorder lines to reorder commits

Use it to clean up a messy local branch before pushing. Don't use it on commits other people have pulled.

---

## 6. Commands every SWE should be fluent in

Roughly in order of how often you'll reach for them.

### `git status` and `git log` (varieties)

`git log --oneline --graph --decorate --all` is the single most useful log invocation. Alias it. It draws the graph with branch labels — once you can see the graph, every other command makes sense.

```
* a3f9b2c (HEAD -> feature) Add validation
* 8e1d4f0 Wire up handler
| * 7b2c9d1 (origin/main, main) Bump version
|/
* 4f6e8a2 Initial commit
```

Other useful ones:

- `git log -p <file>` — show every commit that touched a file, with diffs.
- `git log -S "string"` — show every commit that added or removed `"string"`. Incredible for "when did this code appear?"
- `git log --author="aviv"` — filter by author.
- `git blame <file>` — who last touched each line, and in what commit.

### `git diff` variants

- `git diff` — working dir vs. index.
- `git diff --staged` (or `--cached`) — index vs. HEAD.
- `git diff HEAD` — working dir vs. HEAD.
- `git diff main..feature` — what's on `feature` that isn't on `main`.
- `git diff main...feature` (three dots) — what's on `feature` since it diverged from `main`. Subtle but common in PR review tools.

### `git stash`

Park uncommitted changes temporarily.

- `git stash` — stash working dir + index.
- `git stash -u` — also stash untracked files.
- `git stash list` — see all stashes.
- `git stash pop` — apply the most recent and remove it from the stack.
- `git stash apply stash@{1}` — apply a specific stash without removing it.

Stashes are global to the repo, not branch-specific. Don't treat the stash as long-term storage; commit instead.

### `git reset` — three flavors

`git reset` moves the current branch pointer to a different commit. The flavors differ in what they do to the index and working dir.

| Flavor              | Branch pointer | Index            | Working dir       | Use case |
| ------------------- | -------------- | ---------------- | ----------------- | -------- |
| `--soft <commit>`   | Moves          | Unchanged        | Unchanged         | Uncommit, keep changes staged. "I want to redo the last commit." |
| `--mixed` (default) | Moves          | Reset to commit  | Unchanged         | Uncommit and unstage. Keep edits in working dir. |
| `--hard <commit>`   | Moves          | Reset to commit  | Reset to commit   | Nuke everything back to `<commit>`. **Loses uncommitted work.** |

Common move: `git reset --soft HEAD~` — undo the last commit but keep its changes staged so you can recommit.

### `git revert`

Creates a *new* commit that undoes a previous one. Safe to use on shared branches because it doesn't rewrite history.

- `git revert <sha>` — make a commit that inverts that one.
- Use this instead of `reset --hard` when the commit is already pushed.

### `git cherry-pick <sha>`

Take a commit from one branch and replay it on the current branch as a new commit. Useful for hotfixes (apply a fix from `main` to a release branch, or vice versa).

### `git reflog` — your safety net

**This is the command that saves your career.** Git keeps a log of every time `HEAD` moved — every commit, checkout, reset, rebase. Even commits you "lost" via a bad reset are usually still there for ~90 days, dangling but recoverable.

```
$ git reflog
a3f9b2c HEAD@{0}: reset: moving to HEAD~3
4f6e8a2 HEAD@{1}: commit: the work I just deleted
```

`git reset --hard HEAD@{1}` and you're back. If you ever think "I lost work," `git reflog` first, panic later.

### `git bisect`

Binary search through history to find the commit that introduced a bug.

```
git bisect start
git bisect bad           # current commit is broken
git bisect good v1.2     # this old version was fine
# git checks out a commit halfway between
# you test, then:
git bisect good   # or: git bisect bad
# repeat until git names the culprit
git bisect reset         # back to where you were
```

Can be automated with `git bisect run <test-script>` if you have a script that exits 0 for good and non-zero for bad. Magical for "this regressed three weeks ago and I have no idea where."

### `git fetch` vs `git pull`

- `git fetch` — download remote changes into `origin/*` refs. **Doesn't touch your branches.**
- `git pull` — `fetch` + `merge` (or `rebase` if configured) into your current branch.

I recommend `git fetch` followed by inspecting `git log HEAD..origin/main` before merging or rebasing. `pull` is a footgun when you've forgotten what's pending.

Configure `git config --global pull.rebase true` if you prefer rebase semantics on pull.

### `git push --force-with-lease`

Sometimes you've rebased and need to push over what's on the remote. **Never** use plain `--force` on a shared branch — it can blow away a teammate's commits if they pushed while you were rebasing.

`--force-with-lease` says "force-push, but only if the remote is still where I last saw it." If a teammate pushed in the meantime, your push is rejected and you can investigate.

---

## 7. Workflows you'll meet in the wild

### Feature branch workflow

1. `git checkout -b feature/foo` off `main`.
2. Commit as you go.
3. Periodically `git fetch && git rebase origin/main` to stay current.
4. Push: `git push -u origin feature/foo`.
5. Open PR. Address review feedback in new commits.
6. Before merging, optionally interactive-rebase to clean up commits.
7. Merge into `main` (one of three ways below).

### Three ways to land a PR

- **Merge commit** — preserves the branch's commits, adds a merge commit. History shows the bubble.
- **Squash and merge** — collapses the whole PR into a single commit on `main`. Loses fine-grained history but keeps `main` tidy. Common for small PRs.
- **Rebase and merge** — replays the PR's commits onto `main` linearly, no merge commit. Preserves individual commits in a clean line.

Most teams pick one and stick to it. Know which yours uses.

### Trunk-based development

Short-lived branches (hours, not weeks). Frequent merges to `main`. Feature flags hide unfinished work. Less merge pain, more discipline required.

### Git Flow (older, heavier)

Long-lived `develop` branch, `release/*` branches, `hotfix/*` branches, etc. You'll see this in older codebases. Generally heavier than modern teams need.

---

## 8. When things go wrong — recovery cheatsheet

| Symptom                                                  | Fix                                                       |
| -------------------------------------------------------- | --------------------------------------------------------- |
| Committed to wrong branch                                | `git reset --soft HEAD~` then checkout the right branch and commit. |
| Bad commit message                                       | `git commit --amend` (only if not pushed).                |
| Forgot to add a file to the last commit                  | `git add <file> && git commit --amend --no-edit`.         |
| Pushed a bad commit; need to undo on shared branch       | `git revert <sha>` — makes an inverse commit.             |
| `git reset --hard`'d and lost work                       | `git reflog` → `git reset --hard HEAD@{N}` to the prior state. |
| Rebase went sideways midway through                      | `git rebase --abort` to undo and start over.              |
| Merge conflict during rebase, fixed it                   | `git add <files> && git rebase --continue`.               |
| Need to throw away all local changes to a file           | `git restore <file>` (or `git checkout -- <file>` on older Git). |
| Pulled and got a confusing merge commit you didn't want  | `git reset --hard origin/<branch>` (if you have nothing to keep). |
| Accidentally committed a secret                          | The commit is now in history — even after deleting the file. Use `git filter-repo` (or BFG Repo-Cleaner) to scrub history, then force-push. **Rotate the secret regardless** — assume it's compromised. |

---

## 9. The `.git` directory, briefly

Worth poking around once. It's not magic, just files.

```
.git/
├── HEAD                    # text file: "ref: refs/heads/main"
├── config                  # repo-local config
├── refs/
│   ├── heads/              # one file per branch, contents = SHA
│   ├── tags/               # one file per tag
│   └── remotes/origin/     # cached remote-tracking refs
├── objects/                # the content-addressed store: commits, trees, blobs
└── logs/                   # the reflog data
```

Run `cat .git/HEAD` and `cat .git/refs/heads/main` to see exactly what your branch and HEAD pointers are.

---

## 10. Gotchas worth knowing

- **`.gitignore` doesn't untrack already-tracked files.** Adding a file to `.gitignore` after it's been committed does nothing. You need `git rm --cached <file>`.
- **Line endings on cross-platform repos.** `core.autocrlf` is the setting; on macOS/Linux usually `input`, on Windows usually `true`. Misconfigured = the entire file looks changed.
- **Case-insensitive filesystems (macOS default).** Renaming `Foo.js` → `foo.js` won't be detected. Use `git mv -f Foo.js foo.js`.
- **Detached HEAD after checkout `<sha>`.** Commits made here aren't on a branch. To save them, `git switch -c new-branch-name` immediately.
- **Submodules.** Avoid unless you really need them. They're another repo embedded as a pointer; they don't auto-update with the parent.

---

## 11. What I want to explore next

(Add questions here as they come up.)

- How does `git worktree` actually work — multiple working dirs sharing one `.git`?
- When is `git rerere` worth turning on? (Resolves repeated merge conflicts the same way each time.)
- The plumbing commands (`git cat-file`, `git rev-parse`, `git ls-tree`) and how to read raw objects.
- `git filter-repo` for history rewriting.
- What exactly does `git gc` do, and when do I need to care?

---

## Open questions / Q&A

(Append questions and answers here over time, like networking.md.)
