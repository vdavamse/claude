# forge — GitLab reference (`glab` + REST API v4)

> **Version warning:** `glab` flags change between versions — when a flag errors, check
> `glab <cmd> --help` and fall back to the `glab api` form, which is stable.

One section per op, in contract order. `glab api` expands `:id` / `:fullpath` / `:branch`
placeholders in paths to the current project, so `projects/:id/...` works without looking up the
numeric project id first. Placeholders: `<MR>` (MR IID), `<BRANCH>`, `<BASE>`, `<IID>` (issue),
`<THREAD_ID>` (discussion id), `<DID>` (discussion id), `<NOTE_ID>`, `<PIPELINE_ID>`, `<JOB_ID>`,
`<DIR>`. Write long bodies to files in the scratch dir and pass them with `$(cat file.md)` or
`--input file.json`; never inline long markdown in shell strings.

## context

Resolve the platform identifiers: project full path, URL-encoded project id, host, default branch,
and the authenticated username.

```bash
glab repo view -F json | jq '{path_with_namespace, default_branch, web_url}'
PROJECT_PATH=$(glab repo view -F json | jq -r .path_with_namespace)
PROJECT_ID_ENC=$(printf '%s' "$PROJECT_PATH" | jq -sRr @uri)   # URL-encoded, e.g. group%2Fsub%2Fproject
DEFAULT_BRANCH=$(glab repo view -F json | jq -r .default_branch)
ME=$(glab api user | jq -r .username)
HOST=$(git remote get-url origin | sed -E 's#.*@([^:/]+).*#\1#; s#https?://([^/]+)/.*#\1#')
```

Output: full path (`group/subgroup/project`), its URL-encoded form (needed for raw REST calls that
don't go through `glab api`'s `:id` expansion), default branch, username, host. Gotchas:
- Self-hosted instances need `glab auth login --hostname <host>` first, or `GITLAB_HOST=<host>`
  exported so glab targets the right instance.
- Most `glab api projects/:id/...` calls below let glab fill in `:id`; use the URL-encoded path
  only when calling the REST API directly (e.g. with curl).

## issue-view

Show one issue with its body and comments, given an IID or a full URL.

```bash
glab issue view <IID> --comments
```

Output: rendered title, state, labels, body, then all comments. Gotchas:
- Also accepts a full URL (`https://gitlab.example.com/group/project/-/issues/42`).
- Machine-readable: `glab api projects/:id/issues/<IID> | jq '{title, state, description}'` plus
  `glab api "projects/:id/issues/<IID>/notes?per_page=100"` for comments.

## issue-list-mine

List the open issues assigned to the current user.

```bash
glab issue list --assignee=@me
```

Output: one line per issue — IID, title, labels, updated date. Gotchas:
- Open issues only by default; add `--all` for other states.
- Via API: `glab api "projects/:id/issues?assignee_username=$ME&state=opened&per_page=100"`.

## change-for-branch

Find the MR IID for the current branch; empty output means there is no MR.

```bash
MR=$(glab mr view -F json 2>/dev/null | jq -r .iid)
[ -z "$MR" ] || [ "$MR" = "null" ] && echo "no MR for the current branch"
```

Output: the MR IID, or nothing. Gotchas:
- `glab mr view` with no argument resolves the current branch and errors when there is no MR —
  redirect stderr and treat empty/`null` as "none".
- Via API: `glab api "projects/:id/merge_requests?source_branch=:branch&state=opened" | jq -r '.[0].iid'`
  (glab expands `:branch` to the current branch name).

## change-view

Fetch the full state of an MR: state, branches, head SHA, mergeability, CI rollup, diff refs,
files, commits.

```bash
glab mr view <MR> -F json | jq '{
  state, source_branch, target_branch, sha,
  detailed_merge_status, has_conflicts,
  head_pipeline: .head_pipeline.status,
  diff_refs
}'
# Changed files: use change-diff (the MR JSON carries no reliable file list).

# Commits on the MR:
glab api projects/:id/merge_requests/<MR>/commits | jq -r '.[] | "\(.short_id)\t\(.title)"'
```

Output shape: `state` (`opened`/`merged`/`closed`), `sha` = head SHA,
`detailed_merge_status` (`mergeable`, `ci_still_running`, `conflict`, `not_approved`, …),
`head_pipeline.status` = CI rollup, `diff_refs` = `{base_sha, start_sha, head_sha}`. Gotchas:
- `diff_refs` is what `inline-comment` and `review-batch-post` must pin — fetch it **fresh** right
  before posting; stale SHAs make GitLab reject or misplace comments.
- If new commits landed since you fetched, everything derived from `sha`/`diff_refs` is stale.

## change-diff

Get the MR's full unified diff, and the name-only list of changed files.

```bash
glab mr diff <MR> --raw    # full unified diff

glab api --paginate "projects/:id/merge_requests/<MR>/diffs?per_page=100" \
  | jq -r '.[].new_path'   # changed file names
```

Output: raw unified diff on stdout; one path per line. Gotchas:
- The `/diffs` endpoint needs GitLab ≥ 15.7; on older instances use
  `glab api projects/:id/merge_requests/<MR>/changes | jq -r '.changes[].new_path'`.
- `--raw` avoids glab's color/pager decoration, which corrupts diffs meant for parsing.
- The diff is the input for validating commentable lines before any inline post (see
  `inline-comment`).

## change-checkout

Fetch the MR head into a local branch — works for forks too, because GitLab publishes every MR
head on the target project.

```bash
git fetch origin merge-requests/<MR>/head:review-<MR>
git worktree add ../worktrees/<PROJECT>-review-<MR> review-<MR>
```

Output: local branch `review-<MR>` at the MR head. Gotchas:
- Alternative: `glab mr checkout <MR> --branch review-<MR>` (checks out in the current worktree,
  so prefer the fetch form when you want a separate worktree).
- Pushing to `review-<MR>` does not update the MR; to push fixes to your own MR, check out its
  real `source_branch` from `change-view`.
- Clean up with `git worktree remove ../worktrees/<PROJECT>-review-<MR> && git branch -D review-<MR>`.

## change-create

Open an MR from the current branch against a base branch, with the body read from a file.

```bash
glab mr create \
  --source-branch "$(git branch --show-current)" \
  --target-branch <BASE> \
  --title "<T>" \
  --description "$(cat body.md)" \
  --yes
```

Output: the new MR URL and IID. Gotchas:
- `--yes` skips the interactive prompts — required in agent runs.
- Write the description to `body.md` in the scratch dir first; `$(cat body.md)` survives backticks
  and quotes far better than an inline string.
- Replace `<BASE>` with the default branch from `context` unless told otherwise.

## change-comment

Post a top-level (non-inline) note on the MR conversation.

```bash
glab mr note <MR> -m "$(cat body.md)"
```

Output: confirmation / the note URL. Gotchas:
- Top-level notes appear in the conversation, never inline; for a line-anchored comment use
  `inline-comment`.
- Via API: `glab api -X POST projects/:id/merge_requests/<MR>/notes -f body="$(cat body.md)"`.

## comments-list-top

List the top-level (conversation) notes on the MR — not inline diff notes, not system events.

```bash
glab api --paginate "projects/:id/merge_requests/<MR>/notes?per_page=100&sort=asc" \
  | jq '[.[] | select((.system | not) and .type != "DiffNote" and .type != "DraftNote")
        | {id, author: .author.username, body, created: .created_at}]'
```

Output shape: one object per note. Gotchas:
- `system: true` notes are GitLab's automatic events ("added 2 commits", "changed the
  description") — always filter them out.
- `type == "DiffNote"` notes are inline; use `comments-list-inline` / `threads-unresolved` for those.
- Reply with `change-comment`, mentioning the author with `@<username>`.

## comments-list-inline

List all existing inline (positioned) comments with discussion id, note id, path, line and body —
for dedupe before posting and verification after.

```bash
glab api --paginate "projects/:id/merge_requests/<MR>/discussions?per_page=100" \
  | jq '[.[] as $d | $d.notes[] | select(.position != null) | {
      discussion_id: $d.id,
      note_id: .id,
      path: .position.new_path,
      line: (.position.new_line // .position.old_line),
      body: .body
    }]'
```

Output shape: one object per inline note. Gotchas:
- `discussion_id` must be the **parent discussion's** id — iterate with `.[] as $d | $d.notes[]`,
  not `.[] | .notes[]`, or you lose it.
- `select(.position != null)` filters out non-positioned (top-level) notes that live in the same
  discussions feed.
- Always `--paginate` with `per_page=100`; the first page is not the whole list.
- You need both ids to edit (`comment-edit`) and the discussion id to reply (`thread-reply`).

## threads-unresolved

Fetch the unresolved diff discussions with thread id, first note id, path, line and body.

```bash
glab api --paginate "projects/:id/merge_requests/<MR>/discussions?per_page=100" \
  | jq '[.[] | select(.notes[0].resolvable and (.notes[0].resolved | not)) | {
      thread_id: .id,
      note_id: .notes[0].id,
      path: .notes[0].position.new_path,
      line: (.notes[0].position.new_line // .notes[0].position.old_line),
      body: .notes[0].body
    }]'
```

Output shape: one object per unresolved thread; `thread_id` is the discussion id (a SHA hex
string) used by `thread-reply` and `thread-resolve`. Gotchas:
- Only diff discussions are `resolvable`; `select(.notes[0].resolvable …)` filters out plain
  conversation notes.
- Consumers read `.notes[0]` — post one finding per discussion, or later notes hide behind the
  first.
- `position` is null for non-diff threads; those still pass the resolvable filter but report
  `path: null` — treat them as file-level and unaddressable by line.

## thread-reply

Reply inside an existing discussion.

```bash
glab api -X POST projects/:id/merge_requests/<MR>/discussions/<THREAD_ID>/notes \
  -f body="$(cat reply.md)"
```

Output: the created note object (`id`, `body`). Gotchas:
- `<THREAD_ID>` is the discussion id from `threads-unresolved` / `comments-list-inline`.
- Replying never resolves the thread; run `thread-resolve` separately once the fix is pushed.

## thread-resolve

Mark a discussion resolved.

```bash
glab api -X PUT projects/:id/merge_requests/<MR>/discussions/<THREAD_ID> -f resolved=true
```

Output: the updated discussion object with `notes[].resolved == true`. Gotchas:
- Only resolvable (diff) discussions can be resolved; a 400/403 here usually means
  `<THREAD_ID>` is a plain note thread.
- Resolve only after the fix is actually pushed — resolved threads disappear from
  `threads-unresolved`.

## inline-comment

Post one comment anchored on a file line of the MR diff, as a new diff discussion.

First fetch the MR's `diff_refs` **fresh**:

```bash
glab mr view <MR> -F json | jq .diff_refs
# → {"base_sha": "...", "head_sha": "...", "start_sha": "..."}
```

Validate the line against `valid.tsv` (forge skill, **Commentable lines** section) and read its
OLD column. Map the neutral `{path, line, side}` to GitLab's position:
- `side: LEFT` (deleted line) → `old_line: line` only.
- `side: RIGHT`, OLD column `-` (added line) → `new_line: line` only.
- `side: RIGHT`, OLD column a number (context line) → `new_line: line` **and** `old_line: OLD`.

Build the payload as a JSON file in the scratch dir (use `jq` so bodies are escaped correctly).
The example below is an added line; add `old_line` / drop `new_line` per the mapping above:

```bash
jq -n \
  --arg body "$(cat comment.md)" \
  --arg base  "$BASE_SHA" \
  --arg start "$START_SHA" \
  --arg head  "$HEAD_SHA" \
  --arg path  "<FILE_PATH>" \
  --argjson line <LINE_NUMBER> '
{
  body: $body,
  position: {
    position_type: "text",
    base_sha: $base, start_sha: $start, head_sha: $head,
    old_path: $path, new_path: $path,
    new_line: $line
  }
}' > payload.json

glab api -X POST projects/:id/merge_requests/<MR>/discussions \
  -H "Content-Type: application/json" --input payload.json
```

Output: the created discussion object (`id` = discussion/thread id, `notes[0].id` = note id).

Line rules (mirror GitHub's side semantics):
- **Added line** → `new_line` only.
- **Removed line** → `old_line` only.
- **Unchanged context line** → both `old_line` and `new_line` (the OLD column of `valid.tsv`).

Gotchas:
- A line outside the diff is rejected (400) or lands as a non-positioned note — validate every
  candidate against the `change-diff` output first, exactly as the GitHub flow does: a comment on
  the wrong line is misleading, so drop unanchorable findings and report them rather than
  snapping to a nearby line.
- Stale `diff_refs` SHAs (new commits pushed since fetch) make GitLab reject or misplace the
  comment — re-fetch `change-view` if the branch moved.
- `old_path` and `new_path` differ only for renamed files; set both.

## review-batch-post

Post many inline comments as one batch, using GitLab draft notes + bulk publish (the closest
equivalent of a GitHub review).

Translate the neutral `threads.json` (`[{path, line, side, startLine?, body}]`) into one draft
payload per comment, using the OLD column of `valid.tsv` for context lines:

```bash
DR=$(glab mr view <MR> -F json | jq -c .diff_refs)        # fresh, right before posting
i=0
jq -c '.[]' "$WORK/threads.json" | while read -r t; do
  i=$((i+1))
  path=$(jq -r .path <<<"$t"); line=$(jq -r .line <<<"$t"); side=$(jq -r .side <<<"$t")
  old=$(awk -F'\t' -v p="$path" -v s="$side" -v l="$line" \
        '$1==p && $2==s && $3==l {print $4; exit}' "$WORK/valid.tsv")
  jq -n --argjson t "$t" --argjson dr "$DR" --arg old "${old:--}" '
    {note: $t.body,
     position: ({position_type: "text",
                 base_sha: $dr.base_sha, start_sha: $dr.start_sha, head_sha: $dr.head_sha,
                 old_path: $t.path, new_path: $t.path}
                + (if $t.side == "LEFT" then {old_line: $t.line}
                   elif $old != "-" then {new_line: $t.line, old_line: ($old | tonumber)}
                   else {new_line: $t.line} end))}' > "$WORK/draft-$i.json"
done
```

Then POST each payload to the draft_notes endpoint, collecting the returned ids:

```bash
# per comment — payload.json like inline-comment's, but {"note": ..., "position": {...}}
glab api -X POST projects/:id/merge_requests/<MR>/draft_notes \
  -H "Content-Type: application/json" --input draft-N.json | jq -r .id >> draft-ids.txt

# after ALL drafts succeeded — publish them as one batch:
glab api -X POST projects/:id/merge_requests/<MR>/draft_notes/bulk_publish
```

Output: per draft, its id; bulk_publish returns 204 (no content) — verify with
`comments-list-inline` that the notes appeared. Gotchas:
- **Not atomic**, unlike GitHub's `addPullRequestReview`: if any draft POST fails, run
  `review-batch-rollback` (delete every draft you created) **before** publishing, or you publish
  a half batch.
- Draft notes are invisible to everyone else until bulk_publish.
- The same fresh `diff_refs` and line-validation rules as `inline-comment` apply to every draft.
- Multi-line ranges (`startLine`) are anchored to `line` only on GitLab; mention the range in the
  body if it matters.
- Renamed files: the neutral shape has one `path`; for a renamed file set `old_path` to the
  pre-rename path from `change-diff`.
- `bulk_publish` needs a recent GitLab. If it returns 404, delete the drafts
  (`review-batch-rollback`) and post each comment as its own discussion with `inline-comment`,
  telling the user the batch was posted one by one.

## review-batch-rollback

Undo a failed or wrong batch — delete draft notes before publishing, or delete published notes
after.

```bash
# BEFORE publishing: list the drafts on the MR, delete each one you created
glab api projects/:id/merge_requests/<MR>/draft_notes | jq '.[].id'
glab api -X DELETE projects/:id/merge_requests/<MR>/draft_notes/<ID>

# AFTER publishing: delete each created note (needs discussion id + note id)
glab api -X DELETE projects/:id/merge_requests/<MR>/discussions/<DID>/notes/<NOTE_ID>
```

Output: nothing on success (204). Gotchas:
- The drafts list may contain **other users'** drafts — delete only the ids you collected in
  `review-batch-post` (`draft-ids.txt`), never blanket-delete everything listed.
- Deleting a published note leaves its discussion; if the discussion becomes empty GitLab removes
  it, but replies from others keep it alive — deleting a note others replied to destroys context,
  so prefer `comment-edit` to fix a wrong body.

## comment-edit

Edit the body of an existing note in place (same discussion, same URL).

```bash
# Inline note (inside a discussion):
glab api -X PUT projects/:id/merge_requests/<MR>/discussions/<DID>/notes/<NOTE_ID> \
  -f body="$(cat fixed.md)"

# Top-level note:
glab api -X PUT projects/:id/merge_requests/<MR>/notes/<NOTE_ID> \
  -f body="$(cat fixed.md)"
```

Output: the updated note object. Gotchas:
- You need both `<DID>` and `<NOTE_ID>` for inline notes — `comments-list-inline` returns both.
- Only the note's author (or a maintainer) can edit it.
- Repair mangled bodies in place — do not delete and repost.

## reviews-list

Show the approval state and the requested reviewers for an MR.

```bash
# Approvals:
glab api projects/:id/merge_requests/<MR>/approvals \
  | jq '{approved, approved_by: [.approved_by[].user.username], approvals_left}'

# Reviewers:
glab mr view <MR> -F json | jq '.reviewers[].username'
```

Output: `approved` (bool), who approved, how many approvals are still required, and the assigned
reviewers. Gotchas:
- GitLab has no per-user "changes requested" state like GitHub — disapproval is a thumbs-down or
  a comment; `approvals_left > 0` is the blocking signal.
- On some instances the approvals API is premium-only; if it 403s, fall back to
  `glab mr view <MR> -F json | jq '{reviewers: [.reviewers[].username]}'` and report that approval
  counts are unavailable.

## ci-status

Check the CI status for the MR: pipeline, then per-job pass/fail.

```bash
# Branch-scoped quick view:
glab ci status --branch <BRANCH>

# For the MR: latest pipeline, then its jobs
PIPELINE_ID=$(glab api "projects/:id/merge_requests/<MR>/pipelines" | jq -r '.[0].id')
glab api "projects/:id/merge_requests/<MR>/pipelines" | jq '.[0]'   # id, status, ref, sha

glab api "projects/:id/pipelines/<PIPELINE_ID>/jobs?per_page=100" \
  | jq -r '.[] | "\(.status)\t\(.stage)\t\(.name)\t\(.id)"'
```

Output: pipeline status (`success`/`failed`/`running`/`pending`), then one line per job with
status, stage, name and job id. Gotchas:
- `.[0]` of the MR pipelines list is the **latest** pipeline — confirm its `sha` matches the MR's
  head `sha` from `change-view`, or you are reading CI for an old commit.
- Keep the job ids: `ci-logs` needs them.

## ci-runs

List recent pipelines for a branch.

```bash
glab ci list --branch <BRANCH>

# Via API:
glab api "projects/:id/pipelines?ref=<BRANCH>&per_page=5" \
  | jq -r '.[] | "\(.id)\t\(.status)\t\(.sha[0:8])\t\(.updated_at)"'
```

Output: pipeline ids, statuses, SHAs, timestamps — newest first. Gotchas:
- GitLab has **no per-workflow name filter** like GitHub's `--workflow=`; a pipeline is one unit.
  Filter the pipeline's *jobs* by name or stage instead (see `ci-status`).

## ci-logs

Pull the trace of failed jobs and grep for the failure.

```bash
# Failed jobs of a pipeline:
glab api "projects/:id/pipelines/<PIPELINE_ID>/jobs?scope[]=failed" \
  | jq -r '.[] | "\(.id)\t\(.stage)\t\(.name)"'

# Trace of one job, through the same grep patterns used on GitHub:
glab ci trace <JOB_ID> | grep -E "(FAILED|ERROR|ruff check|Found.*errors|mypy)" | head -20
glab ci trace <JOB_ID> | grep -B10 -A10 "<error_pattern>" | head -30
# equivalent raw call: glab api projects/:id/jobs/<JOB_ID>/trace
```

Common error patterns to search for:
- Ruff: `E501|W291|W293|Found.*errors`
- Pytest: `FAILED|assert|TypeError`
- Mypy: `mypy.*error|type.*error`

Gotchas:
- Job traces include ANSI color codes; strip them if a pattern won't match:
  `glab ci trace <JOB_ID> | sed -E 's/\x1b\[[0-9;]*m//g' | grep …`.
- Traces are long — always grep and `head`, never dump a full trace into context.

## ci-artifacts

Download the artifacts of a job into a directory.

```bash
glab job artifact <BRANCH> <JOB_NAME> --path <DIR>
```

Output: the artifact files extracted into `<DIR>`. Gotchas:
- Older glab versions name this command `glab ci artifact` — when the command errors, check
  `glab job artifact --help` / `glab ci artifact --help` for the name and flags your version uses.
- Artifacts belong to a *job*, addressed by branch + job name (latest pipeline for that branch);
  for an older pipeline, download via the API:
  `glab api "projects/:id/jobs/<JOB_ID>/artifacts" > artifacts.zip` and unzip into `<DIR>`.
- Use a dedicated directory (e.g. `./checks/<project>-<iid>/`) and delete it when done.
