# Engwire Skills

Skills for [Engwire Runner](https://github.com/engwire/engwire).

Engwire Runner decides **when** a pull request should be reviewed and prepares the exact revision to review. A skill decides **how** the review is done: what context to gather, which problems matter, how to explain them, and what to post back to GitHub.

```text
Review requested from you
          ↓
     Engwire Runner
          ↓
   pinned PR worktree
          ↓
   /engwire-review
          ↓
  review posted as you
```

Runner deliberately does not bundle a review skill: your review policy stays visible, editable, and separate from the machinery that runs it. This repository is where that policy starts, so nobody has to begin with an empty `SKILL.md`.

## Skills

| Skill | What it does |
| --- | --- |
| [engwire&#8209;review](skills/engwire-review/SKILL.md) | Reviews the requested pull request, gathers the relevant code and GitHub context, produces structured findings, and posts them as a pull request review. |

## Install

Install `engwire-review` for Claude Code at user scope:

```sh
npx skills add engwire/skills --skill engwire-review -g -a claude-code -y
```

That is the [skills.sh](https://skills.sh) CLI, which needs Node.js 22.20 or newer — Engwire itself does not. Without Node, copy `skills/engwire-review` from this repository into your Claude Code user skills directory instead — `~/.claude/skills/`, or `$CLAUDE_CONFIG_DIR/skills/` when that is set. Nothing about the skill depends on how it got there.

Point an Engwire rule at it in `~/.config/engwire/config.toml`:

```toml
[[review]]
repos = ["acme/*"]
skill = "engwire-review"
```

Then check the setup:

```sh
engwire doctor
```

User scope is not a preference. Engwire starts Claude Code with user settings only, so a skill, `CLAUDE.md`, `.claude/` directory or `.mcp.json` carried by the pull request cannot configure the reviewer — and a skill installed anywhere else is not found either.

## Make it yours

`engwire-review` works as installed, but it is a starting point rather than a policy you are stuck with.

Read [`skills/engwire-review/SKILL.md`](skills/engwire-review/SKILL.md), then copy the skill directory under a name of your own, change the `name` in its `SKILL.md` to match the new directory, and edit from there: what earns a finding, what context is worth gathering, how much evidence a claim needs, what to stay quiet about, and how findings should read. Your own name is also what keeps the file yours — updating `engwire-review` with the skills CLI reinstalls it from this repository, over whatever you changed.

Give the repositories that need it their own rule — first match wins, so the specific rule goes first:

```toml
[[review]]
repos = ["acme/payments"]
skill = "review-payments"

[[review]]
repos = ["acme/*"]
skill = "engwire-review"
```

Changing a review means editing Markdown. No Engwire release, no redeploy.

## Runner contract

Any skill Engwire runs — these or your own — works within a small execution contract.

- **Input.** Engwire invokes `/<skill> owner/repo#123 at <sha>`. That invocation identifies the pull request and the revision the skill was asked to review.
- **Checkout.** The working directory is Engwire's own detached worktree at that revision. Read the code from disk and treat the given SHA as authoritative; the pull request can move on GitHub while the review runs.
- **GitHub.** The skill posts its own review with `gh`. Engwire pins the host and repository for the run, and the review is posted with your authenticated GitHub account.
- **No interaction.** Nobody is waiting at a terminal. A skill that asks a question hangs until the review times out.
- **Security.** The pull request is contributor-controlled code and text. Keep `allowed-tools` narrow, and do not build, install or execute anything from the branch. Engwire is not a sandbox — see [Engwire's security model](https://github.com/engwire/engwire/blob/main/SECURITY.md).

`engwire doctor` checks what it can before a review is accepted: that the skill is present at user scope, that its front matter leaves it invocable by name, and that the `gh`, `git` and `claude` it found are the ones the review will use.

## Try a change

The fastest feedback loop is a real pull request, but a new installation has to start watching before it can see one. Engwire fixes that point the first time a runner starts with a rule configured, and never looks further back — so run it once first, with nothing to find:

```sh
engwire run --once
```

Then request your own review on a pull request, and run it again:

```sh
engwire run --once    # one poll, at most one review, then exit
engwire status        # what it last did, and where the logs are
```

Each finished review names the transcript it wrote; read that to see what the skill actually did. That first step is needed only once for an installation.

Review behavior lives in the skill, so improving a review is mostly prompt engineering: edit the skill, request another review, compare.
