---
name: review-inline
description: Review a PR/MR and post findings as one batch of inline comments, consumable by change-iteration
argument-hint: <pr-mr-number-or-url>
---

## Platform

Load the `forge` skill first. It detects whether this repo is on GitHub or GitLab and gives the exact command for each forge op named below. "PR/MR" means a GitHub pull request or a GitLab merge request.

Review a PR/MR and publish each finding as an **inline comment** anchored to a specific file and
line, posted as a single review (GitHub) / batch of draft notes (GitLab).

The comments this command posts are the input that `/change-iteration` consumes. The two commands
are two halves of one loop:

```
/review-inline 8              →  posts unresolved review threads on PR/MR #8
         ↓
/change-iteration <issue-url> →  reads those threads, fixes the code, replies, resolves
```

**Do not include the "Generated with Claude Code" footer** in any comment body.
**NEVER commit or push anything from the notes directory.**

---

## Contract with `/change-iteration`

`/change-iteration` reads unresolved review threads (GitHub) / diff discussions (GitLab) with
forge op `threads-unresolved`, which returns per thread: its id, the anchored `path` and `line`,
and the `body` of the first comment. Everything this command posts must therefore satisfy:

| Requirement | Why |
|---|---|
| Must be an **inline review thread**, not a top-level comment | Top-level comments live in the general conversation, and `/change-iteration` handles them separately (forge op `comments-list-top` to read them, forge op `change-comment` to reply) |
| Must have non-null `path` and `line` | `/change-iteration` reports them per-file; null path means the thread is file-level and unaddressable |
| Must be left **unresolved** | Resolved threads are filtered out and never seen |
| **One comment per thread** | `/change-iteration` reads the first comment only; a thread opened with multiple comments would hide all but the first |
| Body carries a hidden marker | Enables idempotent re-runs (Phase 2) |

---

## Phase 0: Resolve the PR/MR

`{{ARGS}}` is a PR/MR number (`8`), a URL (`https://github.com/owner/repo/pull/8` or
`https://gitlab.com/group/project/-/merge_requests/8`), or empty (use the current branch's
PR/MR).

Normalise the argument into a project path and a PR/MR number:

- **URL** → parse the project path and the number out of it (shapes above; a trailing `/files`
  or similar also works)
- **Number or empty** → take the project path from forge op `context`; when the argument is
  empty, take the number for the current branch from forge op `change-for-branch`. If no number
  can be resolved, stop with an error asking for one.

Then fetch the change details with forge op `change-view`: state, source and target branches, and
the head SHA.

- **Always pin comments to the head SHA.** Without it the platform anchors comments to an
  inferred commit and they can land pre-marked as outdated.

Stop and tell the user if the change is not open.

---

## Phase 1: Build the set of commentable lines

**This phase is not optional.** Both platforms reject a comment whose line is not part of the
diff. On GitHub one bad coordinate discards the *entire* batch; on GitLab the notes are posted
one by one, so a bad coordinate loses that finding mid-batch and forces a rollback. Either way:
validate every line before posting.

```bash
WORK="$(mktemp -d)"
```

Save the full diff from forge op `change-diff` to `$WORK/pr.diff`, then build
`$WORK/valid.tsv` with the script in the forge skill's **Commentable lines** section. Each row is
`PATH<TAB>SIDE<TAB>LINE<TAB>OLD`:

- **Context lines are commentable.** Any line inside a hunk works, not just `+` lines.
- **Added/context lines → `RIGHT`** with the new-file line number.
- **Deleted lines → `LEFT`** with the old-file line number. Never comment `RIGHT` on a
  deleted line; that number belongs to a different line or does not exist.
- Lines outside every hunk are **not** commentable, even though they exist in the file.
- `OLD` is only needed on GitLab (context lines carry both numbers); the forge op uses it.

Check any candidate before using it (the trailing tab makes it an exact match on PATH, SIDE, LINE):

```bash
grep -qP "^\Qsrc/app/metrics.py\E\tRIGHT\t42\t" "$WORK/valid.tsv" && echo OK || echo REJECT
```

---

## Phase 2: Read existing comments (idempotency)

Re-running must not duplicate comments. Fetch the existing inline comments with forge op
`comments-list-inline` and save them to `$WORK/existing.jsonl` — one JSON object per comment,
including path, line, body, and resolved state. Read **all** of them, resolved or not.

Every body this command posts ends with a hidden marker:

```
<!-- review-inline:<severity>:<stable-key> -->
```

It renders as nothing on both platforms. Build the set of already-posted keys and **skip any
finding whose key is present**, resolved or not — re-raising a finding the user already dismissed
is worse than staying quiet.

```bash
grep -oP '(?<=<!-- review-inline:)[^>]+(?= -->)' "$WORK/existing.jsonl" | sort -u > "$WORK/seen.txt"
```

Use `<severity>:<path>:<short-slug-of-the-issue>` as the key — stable across re-runs even when
line numbers shift, which they will after the author pushes a fix.

---

## Phase 3: Generate findings (code-reviewer agent)

Launch the **`code-reviewer`** agent via the Agent tool. Give it:

- the diff at `$WORK/pr.diff`
- the PR/MR title and body for intent (from forge op `change-view`)
- the repo's `CLAUDE.md` if one exists
- the commentable-line list at `$WORK/valid.tsv`, and an instruction to anchor every finding to a
  line that appears in it

Require the agent to return **JSON only**, in this shape:

```json
{
  "findings": [
    {
      "path": "src/app/metrics.py",
      "line": 42,
      "side": "RIGHT",
      "startLine": null,
      "severity": "blocking",
      "key": "unguarded-division",
      "title": "Division by zero when no features are proposed",
      "body": "`total` is zero whenever the proposer returns an empty list, so this raises `ZeroDivisionError` before the `is_valid_*` check runs.\n\n```suggestion\nreturn 0.0 if total == 0 else matched / total\n```"
    }
  ]
}
```

Field notes:

- `severity` — `blocking` | `important` | `minor` | `nit`
- `startLine` — optional; set for a multi-line range, must be `< line`, same `side`
- `body` — markdown. ```` ```suggestion ```` blocks are strongly encouraged: both platforms render
  them as an applicable patch, which gives `/change-iteration` a directly applicable fix instead
  of prose to interpret
- `key` — the stable slug for the marker

**Quality bar.** Prefer few, high-confidence findings. Every comment posted is work
`/change-iteration` must later triage and reply to, so a false positive costs real time on both
ends. Do not post style nits a linter already enforces.

---

## Phase 4: Validate and assemble

For each finding, in order:

1. **Drop** if `key` is in `$WORK/seen.txt`.
2. **Drop** if no row of `$WORK/valid.tsv` starts with `path\tside\tline\t`.
   Do not silently snap to a nearby line — a comment on the wrong line is misleading. Report it as
   dropped instead.
3. If `startLine` is set, require it to also be valid, share `side`, and be `< line`.
4. Append the marker to the body.

Report the drop counts to the user before posting. Never let a finding vanish silently.

Assemble the batch as a JSON array at `$WORK/threads.json` — one object per finding:
`{path, line, side, startLine?, startSide?, body}`; omit the `startLine`/`startSide` keys entirely
when unused rather than sending `null`. Build every entry with `jq` so bodies are escaped
correctly — backticks, quotes, and newlines in markdown will corrupt any hand-built JSON string:

```bash
jq -n --arg path "$P" --argjson line "$L" --arg side "$S" --arg body "$BODY" \
  '{path: $path, line: $line, side: $side, body: $body}' >> "$WORK/entries.jsonl"
# ...one entry per finding, then collect them into the array:
jq -s '.' "$WORK/entries.jsonl" > "$WORK/threads.json"
```

---

## Phase 5: Post the batch

Post everything as **one** review (GitHub) / batch of draft notes (GitLab) with forge op
`review-batch-post`, passing the threads file, the pinned head SHA, and a short summary body.
One batch means one notification to reviewers, and every thread starts unresolved — exactly what
`/change-iteration` looks for.

Post it as a plain comment-style review, never as an approval or a change request: this command
normally runs against your own PR/MR, which both platforms restrict for formal verdicts.

### If it fails

- **GitHub**: the review is atomic — on failure nothing at all is created, there is no partial
  state to clean up, and it is always safe to retry.
- **GitLab**: the batch is **not atomic** — a failure can leave some notes already posted.
  Rollback is mandatory: after **any** failure run forge op `review-batch-rollback` to undo what
  was posted, then fix the problem and retry.

If the failure says a line could not be anchored, re-check Phase 1 — the platform does not say
which finding failed, so bisect the array.

---

## Phase 6: Verify and hand off

Confirm the threads are visible the way `/change-iteration` will see them: re-fetch with forge op
`threads-unresolved` (ids, path, line, body) and check that every thread this run posted is
unresolved, anchored to the right path and line, and carries an intact body ending with the
`<!-- review-inline:… -->` marker. Save the output as a JSON array of `{id, path, line, body}` in
`$WORK/posted.json` (wrap it with `jq -s` if the op prints one object per line, and rename the
platform's id field to `id`); a check like this flags problems:

```bash
jq -r '.[] | "\(.path):\(.line)"
       + (if (.body | test("<!-- review-inline:")) then ""
          else "  ⚠️ no marker (id \(.id))" end)' "$WORK/posted.json"
```

A body without the marker, or one that reads as a file path or raw shell string, means it was
posted mangled — repair it **in place** with forge op `comment-edit` (same thread, same URL); do
not delete and repost. Threads written by humans or other tools never carry the marker, so a flag
on a thread you did not create is noise — leave those alone.

Then clean up and report:

```bash
rm -rf "$WORK"
```

Summarise for the user: counts by severity, anything dropped in Phase 4 and why, the review URL,
and the exact `/change-iteration` invocation to run next.

---

## Important Rules

- **Get explicit user approval before Phase 5.** Posting a review is outward-facing and notifies
  reviewers. Show the findings first and let the user cut any of them.
- Post **one batch**, not one review per finding — N separate reviews send N notifications.
- Never post a finding you cannot anchor to a real diff line.
- Prefer ```` ```suggestion ```` blocks over prose; they are directly applicable by
  `/change-iteration`.
- Use forge ops for all platform work; build every payload with `jq`, never string interpolation.
- On GitLab the batch is not atomic: run forge op `review-batch-rollback` after any failure,
  before retrying.
- If the PR/MR has new commits since the diff was fetched, re-run Phase 1 — the pinned head SHA
  and the line numbers will both have moved.

## Workflow Summary

```
PR/MR number/URL → resolve project + number (forge op `context`, forge op `change-for-branch`)
                 → state + head SHA (forge op `change-view`)
                         ↓
               Phase 1: diff (forge op `change-diff`) → valid (path, side, line) set
                         ↓
               Phase 2: existing comments (forge op `comments-list-inline`) → seen keys
                         ↓
               Phase 3: code-reviewer agent → findings JSON
                         ↓
               Phase 4: drop seen / drop unanchorable → threads.json
                         ↓
               Phase 5: forge op `review-batch-post`   ← user approval
                        (on any failure: forge op `review-batch-rollback`, then retry)
                         ↓
               Phase 6: verify (forge op `threads-unresolved`, repair with forge op `comment-edit`)
                         ↓
                   /change-iteration <issue-url>
```
