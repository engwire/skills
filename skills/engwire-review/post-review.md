# Posting the review

One request, one review. Never post comments individually.

## Re-check, then choose the event

The pull request may have moved during the review. Read it again, immediately before posting:

```sh
gh pr view <n> -R <repo> --json state,isDraft,labels,baseRefOid,headRefOid
gh api user --jq .login
```

If the login differs from the one recorded in step 1, the account switched mid-run. Post nothing and report it: this review was gathered as one person and would be published as another.

If `state` is no longer `OPEN`, or `isDraft` has become true, stop and post nothing. The pull request was closed, merged, or taken back to draft during the review; either way it has left the reviewable state since Engwire picked it up.

Otherwise post a `COMMENT` review, unless all of these hold, in which case `APPROVE`:

- `labels` includes `AI approval allowed`, compared case-insensitively.
- `headRefOid` equals `<sha>` and `baseRefOid` equals the base you diffed against. An approval of a revision the branch has moved past can still satisfy branch protection on the new head, which is an approval of code nobody read. When either has moved, add a clause to the body saying so before building the payload — it is why the review did not approve.
- No surviving finding has `impact: high`.
- `coverage.complete` is true. "Nothing blocking in the part I read" is not an approval.
- The author is not the re-checked login. GitHub rejects an approval of your own pull request.

Medium and low findings do not prevent approval, and the new ones are still posted inline: approving says nothing blocks the merge, not that there was nothing to say. Keep the body brief.

Never `REQUEST_CHANGES`. A `high` finding posts as a `COMMENT` review that says plainly what would stop the merge; refusing the change is the reviewer's own call.

## Build the payload

Write the JSON into the scratch directory:

```json
{
  "commit_id": "9f8e7d6c5b4a39281706f5e4d3c2b1a098765432",
  "event": "COMMENT",
  "body": "Reviewed `9f8e7d6` …",
  "comments": [
    { "path": "src/billing/refund.ts", "line": 118, "side": "RIGHT", "body": "…" },
    { "path": "src/billing/refund.ts", "start_line": 114, "start_side": "RIGHT", "line": 118, "side": "RIGHT", "body": "…" }
  ]
}
```

- `commit_id` is the reviewed revision, which anchors the comments to the code you read.
- `body` is required for `COMMENT` and must not be empty.
- `path` is repository-relative, exactly as the diff spells it, with no leading `./`.
- `line` is a line in the head revision for `side: "RIGHT"`, or in the base for `side: "LEFT"`. In a range, `start_line` precedes `line` and both sides match.

## Check the anchors first

GitHub rejects the whole review — every comment in it — if one anchor is not in the diff, and the error does not reliably say which comment was at fault. Check first, against the hunk headers of the file each comment names.

A header reads `@@ -a,b +c,d @@`, and a count is omitted when it is 1, so `@@ -2 +2 @@` means `b` and `d` are both 1. Within that hunk, `RIGHT` lines run from `c` through `c + d - 1` and `LEFT` lines from `a` through `a + b - 1`. Match hunks from the comment's own path, not from anywhere in the diff. The range is necessary but not sufficient on the left: a `LEFT` anchor must land on a `-` line, since GitHub reserves that side for deletions and an unchanged context line belongs to `RIGHT`. Check both endpoints of a range.

A finding whose line falls outside every hunk for its file goes into the body, named by file and line, rather than being forced or dropped.

## Post

```sh
gh api --method POST "repos/<repo>/pulls/<n>/reviews" --input "<scratch>/review.json" --jq .html_url
```

A `422` from this endpoint is either a validation failure or abuse throttling, so read the message before reacting:

- If it names a line, an anchor, a review comment, or the `commit_id`, post once more with `comments` removed and the body rebuilt so every finding appears exactly once, each carrying its file and line where it has one. Drop `commit_id` from that retry, and send it as `COMMENT` even if the first attempt was an `APPROVE` — an approval with no fixed revision approves whatever is there now.
- Anything else — spam, rate limits, a field you cannot place — is not retried. Print the composed review and stop.

If posting fails for any other reason, print the composed review to the transcript so the reviewer can post it by hand rather than lose it.
