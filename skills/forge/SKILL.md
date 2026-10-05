---
name: forge
description: Platform layer for GitHub and GitLab. Detects whether the current repo is hosted on GitHub or GitLab and documents the gh / glab command for each named operation (issues, PRs/MRs, review threads, inline comments, CI). Load before any command or agent step that talks to the code host.
---

# forge

Commands and agents say "run forge op `threads-unresolved`". Resolve the platform first, then look
up that heading in the matching reference file and follow it exactly. The op names and their
meanings are a fixed contract: the same 23 ops exist on both platforms, in the same order.

## Detect the platform

1. If the `FORGE` env var is set to `github` or `gitlab`, it wins. Stop detecting.
2. Otherwise take the host from the origin remote (both URL forms):

```bash
URL=$(git remote get-url origin)
case "$URL" in
  git@*) HOST=${URL#git@}; HOST=${HOST%%:*} ;;                       # git@host:path.git
  *)     HOST=$(printf '%s' "$URL" | sed -E 's#https?://([^/]+)/.*#\1#') ;;  # https://host/path.git
esac
echo "$HOST"
```

3. Map the host:
   - `github.com` → `github`
   - host containing `gitlab` (e.g. `gitlab.com`, `gitlab.example.com`) → `gitlab`
   - anything else (self-hosted, ambiguous) → run `gh auth status --hostname <host>` and
     `glab auth status --hostname <host>` and use whichever CLI is authenticated for that host.
     If neither is, or both are, ask the user which platform to use — do not guess.
4. Read the reference file for the platform, relative to this skill directory
   (`~/.claude/skills/forge/references/`):
   - `github` → `references/github.md`
   - `gitlab` → `references/gitlab.md`
5. If the required CLI is not installed, say how to install it and **stop**:
   - `gh`: https://cli.github.com
   - `glab`: https://gitlab.com/gitlab-org/cli

## Terminology

| GitHub | GitLab |
|---|---|
| pull request (PR) | merge request (MR) |
| PR number | MR IID |
| review thread | diff discussion (resolvable discussion with a `position`) |
| review | batch of draft notes |
| checks / workflow run | pipeline / job |
| `owner/repo` | project full path (`group/subgroup/project`) |
| `gh` | `glab` |

Commands say "PR/MR" or "change" when they mean either one. In the reference files, `<PR>` is used
on GitHub and `<MR>` on GitLab for the same thing: the PR number / MR IID.

## Operation index

The contract. Every op below is a `## <name>` heading in **both** `references/github.md` and
`references/gitlab.md`, with the same meaning on both platforms.

| Op | Purpose | Returns |
|---|---|---|
| `context` | Platform identifiers for the current repo | GH: `OWNER`, `REPO`, default branch, current username; GL: project full path, URL-encoded project id, host, default branch, current username |
| `issue-view` | Show one issue with body and comments, given number or URL | Issue title, state, body, comments |
| `issue-list-mine` | List open issues assigned to the current user | Issue numbers and titles |
| `change-for-branch` | PR/MR for the current branch | PR number / MR IID (empty if none) |
| `change-view` | Full state of the PR/MR | State, source/target branch, head SHA, mergeability, CI rollup, files, commits |
| `change-diff` | Diff of the PR/MR | Full unified diff, plus name-only list of changed files |
| `change-checkout` | Fetch the PR/MR head (also from forks) into a local branch | Local branch `review-<N>` at the PR/MR head |
| `change-create` | Open a PR/MR from the current branch | New PR/MR URL and number (title, body from file, target branch) |
| `change-comment` | Top-level (non-inline) comment on the PR/MR | Comment URL / id |
| `comments-list-top` | Top-level (conversation) comments on the PR/MR | List of `{id, author, body, created}`, system/bot events excluded |
| `comments-list-inline` | All existing inline comments | List of `{id, path, line, body}` — for dedupe and post-verification |
| `threads-unresolved` | Unresolved review threads / discussions | List of `{thread id, first comment id, path, line, body}` |
| `thread-reply` | Reply inside an existing thread | New reply id |
| `thread-resolve` | Mark a thread resolved | Resolved confirmation |
| `inline-comment` | One comment anchored on a file line of the diff | New comment/discussion id |
| `review-batch-post` | Post many inline comments as one review (event COMMENT) | Review id/URL, or publish result for draft notes |
| `review-batch-rollback` | Undo a failed or wrong batch | Nothing (cleanup); deletes the review or the draft/published notes |
| `comment-edit` | Edit the body of an existing comment | Updated comment |
| `reviews-list` | Review/approval state and requested reviewers | Reviews, approvals, requested reviewers |
| `ci-status` | CI status for the PR/MR | Per check/job pass/fail |
| `ci-runs` | Recent workflow runs / pipelines for a branch | Run/pipeline ids, names, statuses (GitHub: also filter by workflow name) |
| `ci-logs` | Logs of failed jobs | Log text filtered through the standard grep patterns |
| `ci-artifacts` | Download artifacts of a run/job into a directory | Files in the target directory |

## Commentable lines (both platforms)

Both platforms reject an inline comment on a line that is not in the diff. Build the set of valid
anchors once from the `change-diff` output, and validate every candidate against it before
`inline-comment` or `review-batch-post`:

```bash
cat > "$WORK/difflines.awk" <<'AWK'
# Emit "PATH<TAB>SIDE<TAB>LINE<TAB>OLD" for every commentable line in a unified diff.
# SIDE: RIGHT = new-file line (added or context), LEFT = old-file line (deleted).
# OLD: old-file line number for context lines (GitLab needs it), "-" otherwise.
/^\+\+\+ /  { p=$2; sub(/^b\//,"",p); next }
/^@@ /      { match($0,/-[0-9]+/);  old=substr($0,RSTART+1,RLENGTH-1)+0
              match($0,/\+[0-9]+/); new=substr($0,RSTART+1,RLENGTH-1)+0
              next }
p=="" || p=="/dev/null" { next }
/^-/  { print p"\tLEFT\t"old"\t-";  old++; next }
/^\+/ { print p"\tRIGHT\t"new"\t-"; new++; next }
/^ /  { print p"\tRIGHT\t"new"\t"old; old++; new++; next }
AWK

awk -f "$WORK/difflines.awk" "$WORK/pr.diff" | sort -u > "$WORK/valid.tsv"

# Check one candidate (note the trailing tab: match PATH, SIDE and LINE exactly):
grep -qP "^\Q<PATH>\E\t<RIGHT|LEFT>\t<LINE>\t" "$WORK/valid.tsv" && echo OK || echo REJECT
```

- Added and context lines → `RIGHT` with the new-file number; deleted lines → `LEFT` with the
  old-file number. Never comment `RIGHT` on a deleted line.
- Lines outside every hunk are not commentable even though they exist in the file. Do not snap a
  finding to a nearby line; drop it and report it.
- Comment payloads use the same neutral shape on both platforms:
  `{path, line, side, startLine?, startSide?, body}`. Each reference file maps it to the
  platform's API (GitHub: `side`/`line`; GitLab: `new_line`/`old_line`, using the OLD column).

## Rules

- **Always resolve the platform before the first op.** Every later step assumes it is known.
- **Never mix `gh` and `glab` in one run.** One platform per run; if a task spans both hosts,
  finish one side before switching, and say so.
- **Paginate lists**: `--paginate` with `gh api`, `glab api --paginate` with `per_page=100`.
  A first page is not the whole list.
- **Write request bodies to files in the scratch dir** rather than inlining long text in shell
  strings. Markdown with backticks, quotes and `$` corrupts inline arguments; a file avoids all
  escaping issues.
- **GitLab inline positions need the MR's `diff_refs`** (base/start/head SHA), fetched fresh with
  `change-view` right before posting. Stale SHAs make GitLab reject or misplace comments.
- **GitLab batch publishing is not atomic** like GitHub's `addPullRequestReview` mutation. If any
  step of `review-batch-post` fails on GitLab, run `review-batch-rollback` before retrying or
  publishing.
