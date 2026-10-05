# forge — GitHub reference (`gh`)

One section per op, in contract order. Each section: what it does, the command, the output shape,
and the gotchas that have bitten before. Placeholders: `<PR>` (PR number), `<OWNER>`, `<REPO>`,
`<RUN_ID>`, `<THREAD_NODE_ID>`, `<COMMENT_DATABASE_ID>`, `<BRANCH_NAME>`, `<WORKFLOW_NAME>`,
`<ARTIFACT_NAME>`. Build every JSON payload with `jq`, never string interpolation. Write long
bodies to files in the scratch dir and pass them with uppercase `-F body=@file` (lowercase `-f`
posts the literal string `@file`).

## context

Resolve the platform identifiers for the current repo: `OWNER`, `REPO`, default branch, and the
authenticated username.

```bash
OWNER=$(gh repo view --json owner --jq .owner.login)
REPO=$(gh repo view --json name  --jq .name)
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
ME=$(gh api user -q .login)
```

Output: four shell variables. Gotchas:
- `gh repo view --json | jq keys` lists every available field when you need one not shown here.
- `Resource not accessible by integration` on any call → the token lacks the `repo` scope; fix with
  `gh auth refresh -s repo`.
- To work against another repo without `cd`, most commands take `-R <OWNER>/<REPO>`.

## issue-view

Show one issue with its body and comments, given a number or a full URL.

```bash
gh issue view <URL-OR-NUMBER> --comments
```

Output: rendered title, state, labels, body, then all comments. Gotchas:
- Accepts `123` or `https://github.com/<OWNER>/<REPO>/issues/123`; it errors loudly if the issue
  does not exist or is inaccessible.
- For machine-readable output add `--json number,title,state,body,comments,labels,assignees`.

## issue-list-mine

List the open issues assigned to the current user.

```bash
gh issue list --assignee @me
```

Output: one line per issue — number, title, labels, updated date. Gotchas:
- Defaults to open issues only; add `--state all` if the user asks for closed ones.
- `-R <OWNER>/<REPO>` scopes it to one repo; without it, it uses the current repo.

## change-for-branch

Find the PR number for the current branch; empty output means there is no PR.

```bash
PR=$(gh pr view --json number --jq .number 2>/dev/null)
[ -z "$PR" ] && echo "no PR for the current branch"
```

Output: the PR number, or nothing. Gotchas:
- `gh pr view` with no argument resolves the current branch and exits non-zero when there is no
  PR — always redirect stderr and treat empty as "none", not as an error.

## change-view

Fetch the full state of a PR: state, branches, head SHA, mergeability, CI rollup, files, commits.

```bash
# Identifiers needed for review mutations (pin these before posting anything)
gh pr view <PR> -R <OWNER>/<REPO> \
  --json id,number,headRefOid,headRefName,baseRefName,state,isDraft,author \
  --jq '{id, number, headRefOid, headRefName, baseRefName, state, isDraft, author: .author.login}'

# Status and mergeability
gh pr view <PR> --json state,mergeable,mergeStateStatus,reviews,statusCheckRollup

# Files and commits
gh pr view <PR> --json files,commits,headRefName,baseRefName
```

Output shape: `id` is the GraphQL node id (`PR_…`), `headRefOid` the head commit SHA,
`statusCheckRollup` an array of per-check results. Gotchas:
- Some JSON fields do not exist: use `statusCheckRollup` instead of `checks`, `headRefName`
  instead of `head.ref`, `pullRequest` instead of `pull_request`. Check with
  `gh pr view <PR> --json | jq keys`.
- `headRefOid` is the `commitOID` to pin in review mutations — without it GitHub anchors comments
  to an inferred commit and they can land pre-marked as outdated.
- If new commits landed since you fetched, re-fetch: the SHA and all line numbers have moved.

## change-diff

Get the PR's full unified diff, and the name-only list of changed files.

```bash
gh pr diff <PR> -R <OWNER>/<REPO>            # full unified diff
gh pr diff <PR> -R <OWNER>/<REPO> --name-only # changed file names only
```

Output: raw unified diff on stdout; one path per line with `--name-only`. Gotchas:
- The full diff is the input for building the set of commentable lines (see `inline-comment`);
  redirect it to a scratch file rather than piping through the model when it is large.

## change-checkout

Fetch the PR head into a local branch — works for forks too, because GitHub publishes every PR
head on the base repo.

```bash
git fetch origin pull/<PR>/head:review-<PR>
git worktree add ../worktrees/<REPO>-review-<PR> review-<PR>
```

Output: local branch `review-<PR>` at the PR head. Gotchas:
- The branch is read-only in practice: pushing to it does not update the PR. To push fixes to
  your own PR, check out its real head branch (`headRefName` from `change-view`) instead.
- Clean up with `git worktree remove ../worktrees/<REPO>-review-<PR> && git branch -D review-<PR>`.

## change-create

Open a PR from the current branch against a base branch, with the body read from a file.

```bash
gh pr create --base main --title "<TITLE>" --body-file body.md
```

Output: the new PR URL. Gotchas:
- Write the description to `body.md` in the scratch dir first; long markdown with backticks and
  quotes breaks inline `--body` strings.
- Include the sections the repo's `.github/pull_request_template.md` expects (typically Summary,
  Problem, Solution, Test Plan), and no "Generated with Claude Code" footer.
- Replace `main` with the default branch from `context` if the repo does not use `main`.

## change-comment

Post a top-level (non-inline) comment on the PR conversation timeline.

```bash
gh pr comment <PR> --body-file body.md

# Equivalent raw REST call (PR comments are issue comments in the API):
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/<OWNER>/<REPO>/issues/<PR>/comments \
  -F body=@body.md
```

Output: the comment URL. Gotchas:
- Use the PR number as the issue number — PR-level comments live under `/issues/<PR>/comments`.
- Top-level comments appear in the general conversation, **never** inline with code; if the
  comment is about a specific line, use `inline-comment` instead. Never confuse the two.
- To answer a specific person's top-level comment, mention them with `@<login>` in the body.

## comments-list-top

List the top-level (conversation) comments on the PR — not inline review comments.

```bash
gh pr view <PR> --json comments \
  --jq '[.comments[] | select(.isMinimized | not) | {id, author: .author.login, body, created: .createdAt}]'
```

Output shape: one object per comment. Gotchas:
- Minimized (hidden) comments are filtered out.
- Inline review comments are **not** included; use `comments-list-inline` or
  `threads-unresolved` for those.
- Reply with `change-comment`, mentioning the author with `@<login>`; top-level comments cannot be
  resolved.

## comments-list-inline

List all existing inline review comments with id, path, line and body — for dedupe before posting
and verification after.

```bash
gh api "/repos/<OWNER>/<REPO>/pulls/<PR>/comments?per_page=100" --paginate \
  | jq -r '.[] | {id, path, line, body}'
```

Output shape: one object per inline comment; `id` is the REST integer id used by `comment-edit`
and `thread-reply`. Gotchas:
- Always `--paginate` **and** `per_page=100`; the first page is not the whole list.
- This endpoint returns only inline (review) comments, not top-level ones.
- Post-verification: a body that starts with `@` or has lost its markdown means a broken
  `-f body=@file` post — it still returned HTTP 201. Check with:

```bash
gh api "/repos/<OWNER>/<REPO>/pulls/<PR>/comments?per_page=100" --paginate |
  jq -r '.[] | select(.body | startswith("@")) | "BROKEN \(.id) \(.path):\(.line)"'
```

  Repair broken bodies in place with `comment-edit` — do not delete and repost.

## threads-unresolved

Fetch the unresolved review threads with thread id, first comment id, path, line and body.

```bash
gh api graphql -f query='
  query {
    repository(owner: "<OWNER>", name: "<REPO>") {
      pullRequest(number: <PR>) {
        reviewThreads(first: 50) {
          nodes {
            id
            isResolved
            isOutdated
            comments(first: 10) {
              nodes {
                id
                databaseId
                author {
                  login
                }
                body
                path
                line
                createdAt
              }
            }
          }
        }
      }
    }
  }
' | jq -r '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | {
  threadId: .id,
  commentDatabaseId: .comments.nodes[0].databaseId,
  path: .comments.nodes[0].path,
  line: .comments.nodes[0].line,
  author: .comments.nodes[0].author.login,
  isOutdated: .isOutdated,
  body: .comments.nodes[0].body,
  replies: [.comments.nodes[] | {author: .author.login, body: .body}]
}'
```

Output shape: one object per unresolved thread. Gotchas:
- `isResolved: false` filters to unresolved threads; `isOutdated` tells you the comment is on
  superseded code.
- `threadId` is the GraphQL node id (starts with `PRRT_`) — used by `thread-resolve`.
- `databaseId` is the integer id needed for REST replies (`thread-reply`) and edits
  (`comment-edit`). Never pass the node id to a REST endpoint or the integer to a GraphQL
  mutation.
- Consumers read `comments.nodes[0]`: a thread opened with several comments hides all but the
  first, so post one comment per thread.

## thread-reply

Reply inside an existing review thread, so the reply appears inline with the code.

```bash
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/<OWNER>/<REPO>/pulls/<PR>/comments/<COMMENT_DATABASE_ID>/replies \
  -F body=@reply.md
```

Output: the created reply object (its `id`, `html_url`). Gotchas:
- Use the comment's `databaseId` (integer) from `threads-unresolved`, **not** the GraphQL `id`.
- Endpoint/ID quick reference:

| Comment type | Method | Endpoint | ID needed |
|---|---|---|---|
| Inline (review thread) | REST POST | `/repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies` | `databaseId` (integer) |
| PR-level (general) | REST POST | `/repos/{owner}/{repo}/issues/{pr}/comments` | none (creates new comment) |

- Review thread replies appear inline attached to the thread's line; PR-level comments appear in
  the conversation timeline. They are different endpoints and different places — never mix them.

## thread-resolve

Mark a review thread resolved.

```bash
gh api graphql -f query='mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}' -F id=<THREAD_NODE_ID>
```

Output: `{"data":{"resolveReviewThread":{"thread":{"isResolved":true}}}}`. Gotchas:
- `<THREAD_NODE_ID>` is the GraphQL node id (`PRRT_…`) from `threads-unresolved`, passed with
  uppercase `-F` so it binds to the `$id` mutation variable.
- Resolved threads disappear from `threads-unresolved` output — resolve only after the fix is
  actually pushed, or the finding is lost.

## inline-comment

Post a single comment anchored to a file line of the diff.

First validate the line against `valid.tsv`, built from `change-diff` with the script in the
forge skill's **Commentable lines** section — **not optional**: GitHub rejects any comment whose
line is not part of the diff with `Line could not be resolved`.

Then post, with the body in a file and the commit SHA pinned:

```bash
COMMIT_SHA=$(gh pr view <PR> --json headRefOid --jq .headRefOid)   # never guess or reuse an old SHA

gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  /repos/<OWNER>/<REPO>/pulls/<PR>/comments \
  -F body=@comment.md \
  -f commit_id="$COMMIT_SHA" \
  -f path="<FILE_PATH>" \
  -F line=<LINE_NUMBER> \
  -f side="RIGHT"
```

Output: the created comment object (`id` = integer `databaseId`, `html_url`). Gotchas:
- Side/line rules: added and context lines → `side: RIGHT` with the new-file line number; deleted
  lines → `side: LEFT` with the old-file line number. Never comment `RIGHT` on a deleted line —
  that number belongs to a different line or does not exist. Lines outside every hunk are **not**
  commentable even though they exist in the file. Do not silently snap to a nearby line; drop the
  finding and report it.
- `path` must be relative to the repo root, never absolute.
- Uppercase `-F body=@file` expands the file; lowercase `-f body=@file` posts the literal string
  `@file` and still returns HTTP 201. Always verify afterwards with `comments-list-inline`.
- "Line number not found" errors usually mean the PR got new commits — re-fetch the SHA, the diff
  and the line numbers.

## review-batch-post

Post many inline comments as a single review with event `COMMENT` (one notification, atomic).

Assemble the threads array (`{path, line, side, startLine?, startSide?, body}` per comment —
validate every line against `valid.tsv` from the forge skill's Commentable lines section first), then build the payload with
`jq` so bodies are escaped correctly:

```bash
jq -n \
  --arg pid "$PR_NODE_ID" \
  --arg oid "$HEAD_OID" \
  --arg summary "$SUMMARY_BODY" \
  --slurpfile threads "$WORK/threads.json" '
{
  query: "mutation($input: AddPullRequestReviewInput!) {
            addPullRequestReview(input: $input) {
              pullRequestReview {
                id url state
                comments(first: 100) { totalCount nodes { databaseId path line } }
              }
            }
          }",
  variables: { input: {
    pullRequestId: $pid,
    commitOID:     $oid,
    body:          $summary,
    event:         "COMMENT",
    threads:       $threads[0]
  }}
}' > "$WORK/payload.json"

gh api graphql --input "$WORK/payload.json" > "$WORK/resp.json"

jq -r 'if .errors then "FAILED: " + ([.errors[].message] | join("; "))
       else .data.addPullRequestReview.pullRequestReview
            | "posted \(.state) \(.url) with \(.comments.totalCount) inline comments"
       end' "$WORK/resp.json"
```

Output: the review `id` (`PRR_…`), `url`, `state`, and the created comments. Gotchas:
- `pullRequestId` is the `PR_…` node id and `commitOID` the head SHA from `change-view` — always
  pin `commitOID` or comments can land pre-marked outdated.
- Use `event: "COMMENT"`: GitHub rejects `APPROVE` and `REQUEST_CHANGES` on a PR you authored,
  and `COMMENT` leaves threads unresolved, which `threads-unresolved` consumers expect. Omitting
  `event` creates a `PENDING` review, invisible until submitted — a safe rehearsal; remove it
  with `review-batch-rollback`.
- The mutation is **atomic**: on error nothing at all is created, so there is no partial state to
  clean up and retrying is always safe.
- Post **one** review, not one review per finding — N reviews send N notifications.
- Omit unused `startLine`/`startSide` keys entirely rather than sending `null`.

Error table:

| Error | Cause | Fix |
|---|---|---|
| `Line could not be resolved` | A line is not in the diff, or `RIGHT` was used for a deleted line | Re-check the commentable-lines set; the message does not say which thread, so bisect the array |
| `Can not request changes on your own pull request` | `event` was not `COMMENT` | Use `COMMENT` |
| `Resource not accessible by integration` | Token lacks `repo` scope | `gh auth refresh -s repo` |
| `Variable $input of type … was provided invalid value` | Usually `null` in an omitted `startLine` | Drop the key rather than sending null |

## review-batch-rollback

Undo a pending or wrongly posted review by deleting it.

```bash
gh api graphql -f query='mutation($id:ID!){ deletePullRequestReview(input:{pullRequestReviewId:$id}){ pullRequestReview { id state } } }' -F id="$REVIEW_ID"
```

Output: the deleted review's `id` and `state`. Gotchas:
- Needed only for `PENDING` (rehearsal) reviews or a review posted with the wrong content; a
  failed `review-batch-post` mutation created nothing, so there is nothing to roll back.
- `$REVIEW_ID` is the `PRR_…` node id returned by the review mutation.
- Deleting a review deletes its inline comments with it. For a single broken comment, prefer
  `comment-edit` in place — it keeps the thread and URL.

## comment-edit

Edit the body of an existing comment in place (same thread, same URL).

```bash
# Inline review comment:
gh api --method PATCH /repos/<OWNER>/<REPO>/pulls/comments/<COMMENT_ID> -F body=@correct-body.md
```

Output: the updated comment object. Gotchas:
- `<COMMENT_ID>` is the integer `databaseId` (from `comments-list-inline` or
  `threads-unresolved`), not a node id.
- Uppercase `-F` expands `@file`; lowercase `-f` posts the literal path.
- Repair mangled bodies in place — do not delete and repost. A body that reads as a file path or
  raw shell string means the original post was mangled.

## reviews-list

Show the review/approval state and the requested reviewers for a PR.

```bash
gh pr view <PR> --json reviews,reviewRequests
```

Output shape: `reviews` = submitted reviews with `author.login`, `state`
(`APPROVED` / `CHANGES_REQUESTED` / `COMMENTED`), `body`; `reviewRequests` = who has been asked
but not yet answered. Gotchas:
- For a quick human-readable view, plain `gh pr view <PR>` includes a reviews section.
- A `CHANGES_REQUESTED` review blocks merging even when checks pass — check
  `mergeStateStatus` from `change-view` alongside it.

## ci-status

Check the CI status of a PR, per check.

```bash
gh pr checks <PR>

# Machine-readable:
gh pr view <PR> --json statusCheckRollup
```

Output: one line per check — name, status (`pass`/`fail`/`pending`/`skipping`), link. Gotchas:
- Use `statusCheckRollup`, **not** `checks` — the `checks` JSON field does not exist.
- `gh pr checks` exits non-zero when any check fails, which is useful in scripts.

## ci-runs

Find recent workflow runs for a branch, or for one named workflow.

```bash
# Discover exact workflow names first (case-sensitive, vary by project):
gh workflow list | head -10

# Latest run of a named workflow:
gh run list --workflow="<WORKFLOW_NAME>" --limit 1

# Runs for the branch, via the API:
gh api "repos/<OWNER>/<REPO>/actions/runs?per_page=5" \
  --jq '.workflow_runs[] | select(.head_branch == "<BRANCH_NAME>") | {id: .id, name: .name, status: .status}'

# Summary of one run (shows which steps failed):
gh run view <RUN_ID>
```

Output: run ids, workflow names, statuses/conclusions. Gotchas:
- Workflow names are case-sensitive and vary by project; always take them from
  `gh workflow list`.
- Run IDs come from the workflow URL or `gh run list`.
- Field names: `headRefName` not `head.ref`; the API's field is `head_branch`.
- To monitor after pushing fixes: `sleep 60 && gh run list --workflow="<WORKFLOW_NAME>" --limit 1`.

## ci-logs

Pull the logs of a run and grep for the failure.

```bash
# Find specific errors in logs
gh run view <RUN_ID> --log | grep -E "(FAILED|ERROR|ruff check|Found.*errors|mypy)" | head -20

# Get error context (shows lines before and after)
gh run view <RUN_ID> --log | grep -B10 -A10 "<error_pattern>" | head -30
```

Common error patterns to search for:
- Ruff: `E501|W291|W293|Found.*errors`
- Pytest: `FAILED|assert|TypeError`
- Mypy: `mypy.*error|type.*error`

Gotchas:
- The `--log-failed` option often misses important context; prefer `--log` piped through grep.
- Logs are long — always grep and `head`, never dump a full log into context.

## ci-artifacts

Download a run's artifacts into a directory for local analysis.

```bash
gh run download <RUN_ID> --name <ARTIFACT_NAME> --dir <DIR>
```

Output: the artifact's files extracted into `<DIR>`. Gotchas:
- `<ARTIFACT_NAME>` comes from the run summary (`gh run view <RUN_ID>`) or the workflow's
  `upload-artifact` step; omit `--name` to download every artifact of the run.
- Use a dedicated directory (e.g. `./checks/<repo>-<issue>/`) and delete it when done.
