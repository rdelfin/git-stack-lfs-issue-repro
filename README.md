# git-stack corrupts Git LFS files during `amend`/rewrite operations

This repo is a minimal, self-contained reproduction of a bug in
[git-stack](https://github.com/gitext-rs/git-stack). When `git stack amend`
(or `reword`/`sync` — anything that goes through git-stack's internal stash)
runs on a branch where a Git LFS-tracked file was renamed, it corrupts the
LFS file: the real tracked file is deleted, and two stray sibling files show
up (`<file>~Stashed changes`, `<file>~Updated upstream`).

Confirmed with:

- `git-stack 0.10.19` (also present in the latest release, `0.10.20`,
  2025-08-05 — its changelog only covers a libgit2 version bump)
- `git-lfs 3.7.0`
- `git 2.54.0`

No existing issue for this was found in `gitext-rs/git-stack` or
`gitext-rs/git-branch-stash`.

## Root cause

git-stack's `amend` does not shell out to `git stash`. It calls libgit2
directly via `git2-rs` ([`src/git/repo.rs`](https://github.com/gitext-rs/git-stack/blob/main/src/git/repo.rs)):

```rust
pub fn stash_push(&mut self, message: Option<&str>) -> Result<git2::Oid> {
    let signature = self.repo.signature()?;
    self.repo.stash_save2(&signature, message, None)
}
```

libgit2 does not run external clean/smudge filters (like Git LFS's) unless
the embedding application explicitly registers one via `git_filter_register`.
A libgit2 maintainer confirms this directly on
[libgit2/libgit2#3373](https://github.com/libgit2/libgit2/issues/3373):

> We do not run external filters for you - you should be able to hook up an
> lfs filter via `git_filter_register`... We don't have a lot of people using
> this API quite yet.

git-stack never registers an LFS filter. So when its libgit2-based stash
snapshots a path that was renamed, it sees:

- the **index**: the small (~130 byte) LFS pointer text
- the **working tree**: the real, smudged binary content

and treats them as two genuinely different, unmergeable (binary) versions of
the same path. On stash-pop it can't reconcile them, and it materializes the
conflict as sibling files using git's own default stash-conflict labels
(`Updated upstream` = current side, `Stashed changes` = popped side), leaving
the real tracked file gone.

Plain `git stack amend` on an *unmodified* LFS file does **not** trigger
this — the bug specifically needs a rename staged for the LFS-tracked path
in the same operation (see reproduction steps below).

## Prerequisites

- `git` (tested with 2.54.0)
- `git-lfs` (tested with 3.7.0), `git lfs install --local` already run in
  this repo
- `git-stack` (tested with 0.10.19 and 0.10.20) — install with
  `cargo install git-stack`, then either run `git-stack amend` directly, or
  set up the usual alias:
  ```
  git config --global alias.amend "stack amend"
  ```

## Repo contents

- `file1.bin`, `file2.bin` — two LFS-tracked files (tracked via
  `.gitattributes`, `filter=lfs diff=lfs merge=lfs -text`). Despite the
  `.bin` extension, their content is plain ASCII text — see the note above
  on why that's enough to reproduce the bug.
- `README.md` — this file

## Reproduction steps

Run these from the root of this repo:

```sh
# 1. Create a feature branch and give it a commit, like any normal stack.
git checkout -b feature
echo "Some feature work" >> README.md
git add README.md
git commit -m "Feature: start README notes"

# 2. Rename one of the LFS-tracked files as part of ongoing work
#    (e.g. a directory/file rename, same as a `git mv`-based refactor).
git mv file1.bin file1-renamed.bin

# 3. Stage a small, unrelated real change so there's something to amend.
echo "One more line" >> README.md
git add README.md

# 4. Fold the staged changes into the feature commit.
git stack amend
```

## Expected result

`file1-renamed.bin` exists, tracked normally by LFS, containing the
original 313 bytes of text content. `git status` is clean.

## Actual result

```
$ git status
On branch feature
Unmerged paths:
  (use "git restore --staged <file>..." to unstage)
  (use "git add/rm <file>..." as appropriate to mark resolution)
	both added:      file1-renamed.bin
	both deleted:    file1.bin

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	file1-renamed.bin~Stashed changes
	file1-renamed.bin~Updated upstream
```

- `file1-renamed.bin` (the real, correctly tracked file) is gone.
- `file1-renamed.bin~Stashed changes` (313 bytes) holds the real text
  content that should have ended up at `file1-renamed.bin`.
- `file1-renamed.bin~Updated upstream` (128 bytes) is just the LFS pointer
  text.
- `git lfs status` starts failing outright:
  ```
  Failed to run `git update-index`: error running /usr/lib/git-core/git 'update-index' '-q' '--refresh': 'file1-renamed.bin: needs merge
  file1.bin: needs merge' 'exit status 1'
  ```

## Cleanup after reproducing

```sh
git checkout main
git branch -D feature
rm -rf .git/branch-stash   # internal git-stack backup, safe to remove
```
