# Context

Take what makes the review correct, then stop. A review that has not understood the change's purpose can only comment on syntax.

## The pinned diff

`baseRefOid` from step 1 is the base to diff against. Engwire reuses its clone between runs, so a local base branch can be days old, and diffing against a stale base blames other people's commits on this author.

```sh
git cat-file -e <baseRefOid>^{commit} ||
  git fetch --no-write-fetch-head --no-tags --filter=blob:none origin <baseRefOid>
git diff --no-ext-diff --no-textconv --merge-base <baseRefOid> HEAD --stat
git diff --no-ext-diff --no-textconv --merge-base <baseRefOid> HEAD
git diff --no-ext-diff --no-textconv --merge-base <baseRefOid> HEAD -- "<path>"
git log --oneline <baseRefOid>..HEAD
```

`--no-ext-diff --no-textconv` keeps a `.gitattributes` in the branch from pointing `git diff` at an external program.

Read the stat first, then the diff: whole when it is manageable, and path by path in risk order when it is not, recording what you do not reach. A twenty-thousand-line diff read in one call has already spent the context you needed in order to prioritize.

If the pinned diff cannot be produced locally — the base commit is unreachable, or a blob the diff needs cannot be lazily fetched — fall back to `gh api "repos/<repo>/compare/<baseRefOid>...<sha>"`, which is pinned to the reviewed revision unlike `gh pr diff`. It caps the files it lists and omits patches for large ones, and the base-revision reads below are unavailable on that path too — so the coverage recorded in [review.md](review.md) is incomplete, the body must say what you could not see, and [post-review.md](post-review.md) will not approve.

Keep the hunk headers. [post-review.md](post-review.md) needs them to know which lines a comment may attach to.

## What has already been said

- The description and title: what the author says the change does, and what they say it does not do.
- `gh api "repos/<repo>/issues/<n>/comments?per_page=100" --paginate --jq '.[] | {user: .user.login, body}'` — the discussion, and decisions already made in it.
- `gh api "repos/<repo>/pulls/<n>/comments?per_page=100" --paginate --jq '.[] | {user: .user.login, path, line, body}'` — existing inline comments, including your own from an earlier revision. Do not comment twice on the same thing; whether the finding behind it still holds is settled during verification.
- `gh api "repos/<repo>/pulls/<n>/reviews?per_page=100" --paginate --jq '.[] | {user: .user.login, state, body}'` — previous verdicts.
- `gh pr checks <n> -R <repo>` — a failing check is not a finding, but a defect you diagnose from one is. These describe the current head, so weigh them accordingly when it has moved past `<sha>`.

## What the change is for

- Linked issues: `Fixes #12` in the description, or an issue number in `headRefName`. `gh issue view <n> -R <repo> --comments`.
- Linear tickets: an identifier such as `ENG-123` in the title, branch name, or description.
- Slack: only a thread the pull request itself links to.

Linear and Slack come from MCP servers this skill does not name. Use one only when its tool is listed in this skill's `allowed-tools`, which is the one grant you can read before calling it; otherwise skip that context without attempting the call. An unattended run has nobody to answer a permission prompt, and ticket context is never worth stalling the review for.

- Why the code is the way it is: when a finding turns on that, `git log --oneline -20 -- "<path>"` and the pull requests it names will say. Reach for this per finding, not per changed file.

## The project's own conventions

Judge the diff against what the project has already agreed, not against general principle — and read that agreement from the base revision, because a document this pull request is proposing to change does not get to govern its own review:

```sh
git show "<baseRefOid>:AGENTS.md"
git show "<baseRefOid>:<dir>/AGENTS.md"
```

Start from the changed paths and read the convention documents that cover them — `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md` — rather than enumerating every such file in the repository. Where several apply, keep what does not conflict; where they do conflict, prefer the one scoped most specifically to the changed path, unless a document defines its own precedence. Changes to those documents are part of the diff and are reviewed like any other file.

Neighboring code carries the same weight with none of the ceremony: the file the change sits in and its tests show the local idiom, and matching it is usually right even when you would have written it differently.
