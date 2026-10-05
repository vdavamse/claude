---
name: change-iteration
description: Comprehensive PR/MR workflow - resolve comments, fix CI checks, and iterate on feedback
argument-hint: <issue-url-or-number>
---

## Platform

Load the `forge` skill first. It detects whether this repo is on GitHub or GitLab and gives the exact command for each forge op named below. "PR/MR" means a GitHub pull request or a GitLab merge request.

This command provides an end-to-end workflow for iterating on PRs/MRs, including resolving comments and fixing CI checks. You'll be prompted to choose your workflow scope at the beginning.

"Review threads" below means review threads (GitHub) / diff discussions (GitLab).

## Setup

### Extract Issue Information

1. **Parse the issue URL from {{ARGS}}:**
   - GitHub example: `https://github.com/owner/repo-name/issues/123`
   - GitLab example: `https://gitlab.com/group/repo-name/-/issues/123`
   - Extract: owner/group, project name, issue number
   - If {{ARGS}} is a bare number, resolve the project from the current directory with forge op `context`

2. **Calculate note file path:**
   - Format: `./notes/<repo-name>-<issue-number>.md`
   - Example: `./notes/repo-name-123.md` (relative to repository root)

3. **Read the note file:**
   - The note file MUST exist (created during issue workflow)
   - Search for the PR/MR URL in the notes (typically in Phase 5 section)
   - Extract the PR/MR number from the PR/MR URL; if the notes have none, resolve it from the working branch with forge op `change-for-branch`
   - Remember the worktree directory location from the notes

4. **Load project context:**
   - Load the CLAUDE.md from the specific project if it exists
   - Example: if working on repo `/path/to/repo-name`, load `/path/to/repo-name/CLAUDE.md`

**NEVER commit and push any file from the notes directory** - these are for local tracking only.
**NEVER commit and push CLAUDE.md file** - this is local file only.
**Do not include the "Generated with Claude Code" footer**

## Phase 0: Choose Workflow Scope

Use the **AskUserQuestion** tool to ask the user which workflow they want:

```json
{
  "questions": [
    {
      "question": "Which PR/MR iteration workflow would you like to run?",
      "header": "Workflow",
      "multiSelect": false,
      "options": [
        {
          "label": "Comments Only",
          "description": "Resolve PR/MR comments and review feedback only"
        },
        {
          "label": "Full Iteration",
          "description": "Fix CI checks AND resolve PR/MR comments"
        }
      ]
    }
  ]
}
```

Based on the user's selection, proceed to the appropriate workflow:
- **Comments Only** → Go to **Workflow A**
- **Full Iteration** → Go to **Workflow B**

---

## Workflow A: Comments Only

### Phase 1: Fetch All Comments

Retrieve both types of comments from the PR/MR:

#### 1.1 Get Review Threads (Inline Comments)

Fetch the unresolved review threads (GitHub) / diff discussions (GitLab) with forge op `threads-unresolved`.

**Key fields per thread (exact shape comes from the forge skill):**
- thread id — needed to reply and to resolve
- `path` and `line` the thread is anchored to
- author and body of the first comment
- whether the thread is outdated (anchored to code that has since changed)
- existing replies

Only unresolved threads are returned; resolved ones have already been handled.

#### 1.2 Get Top-Level Comments (General Conversation)

Fetch the PR/MR's top-level conversation comments with forge op `comments-list-top` (hidden comments and system events are already filtered out). Note each comment's author, body, and creation date.

#### 1.3 Check Review Status

Fetch approvals and review state with forge op `reviews-list`.

### Phase 2: Address All Comments

For each unresolved comment:

1. **Read the comment and understand the feedback**
2. **Navigate to the worktree directory** (found in notes file)
3. **Make necessary code changes to address the feedback**
4. **After addressing, reply to the comment using the appropriate method**

#### Replying to Review Threads (Inline Comments)

Reply inside the thread with forge op `thread-reply`, passing the thread id obtained from forge op `threads-unresolved`. The reply appears inline with the code, in the same thread.

Once the fix is pushed and the reply posted, mark the thread resolved with forge op `thread-resolve`.

#### Replying to Top-Level Comments (General Conversation)

Post the reply as a top-level comment with forge op `change-comment`, mentioning the original author (`@username`) and referencing their point so the reply is unambiguous.

Top-level comments cannot be resolved; the reply itself closes the loop.

### Phase 3: Push Changes and Review

1. **After all feedback has been addressed, use the `code-reviewer` agent to review changes**

2. **Push commits to the remote branch:**
   ```bash
   git add <files>
   git commit -m "Concise commit message without Claude attribution"
   git push
   ```

3. **Repeat for each round of feedback** until the PR/MR is approved

---

## Workflow B: Full Iteration

### Phase 1: Check PR/MR Status and CI

All logs need to be downloaded and extracted to:
- `./checks/<repo-name>-<issue-number>/`
- Example: `./checks/repo-name-123/` for issue 123

#### 1.1 Initial Status Check

Fetch state, branches, head SHA, mergeability, review state, files, commits and the CI rollup with forge op `change-view`, and the per-check CI status for the PR/MR with forge op `ci-status`.

#### 1.2 Monitor CI

List the recent workflow runs (GitHub) / pipelines (GitLab) for the branch with forge op `ci-runs`. Note the id of the latest run for the head branch and which jobs/steps failed.

#### 1.3 Debug Failed Checks

Fetch the failed job logs with forge op `ci-logs`, save them under `./checks/<repo-name>-<issue-number>/`, then search them locally:

```bash
# Find specific errors in the saved logs
grep -E "(FAILED|ERROR|ruff check|Found.*errors|mypy)" ./checks/<repo-name>-<issue-number>/*.log | head -20

# Get error context (shows lines before and after)
grep -B10 -A10 "<error_pattern>" ./checks/<repo-name>-<issue-number>/<job>.log | head -30

# Common error patterns to search for:
# - Ruff: "E501|W291|W293|Found.*errors"
# - Pytest: "FAILED|assert|TypeError"
# - Mypy: "mypy.*error|type.*error"
```

If a failed job published artifacts you need, download them into the same directory with forge op `ci-artifacts`.

### Phase 2: Fix CI Issues

#### 2.1 Code Quality Checks

Always run linting checks before committing:

```bash
# Check and fix linting in the correct environment
pyenv activate <project-env>  # Use project-specific environment
python -m ruff check --fix api/ tests/  # Auto-fix issues
```

**Important:** Use `pyenv activate` to access project-specific ruff installation.

#### 2.2 Fix All Issues

Address all mypy, ruff, pylint, pytest issues found in the logs.

1. Navigate to the worktree directory
2. Fix each issue systematically
3. Run local tests to verify fixes
4. Push to remote branch with concise commit message (no Claude attribution)

#### 2.3 Iterate on CI Results

1. Push fixes
2. Wait for CI to run (about a minute), then list the new runs with forge op `ci-runs`
3. Check the new per-check status with forge op `ci-status`
4. Repeat until all checks pass

### Phase 3: Fetch and Address All Comments

Follow the same process as **Workflow A - Phase 1 & 2** above:
1. Get review threads (GitHub) / diff discussions (GitLab) — the inline comments
2. Get top-level comments (general conversation)
3. Address each comment
4. Reply using the appropriate method, then resolve the threads

### Phase 4: Push Changes and Review

Same as **Workflow A - Phase 3** above.

### Phase 5: Common Workflow for Fixing CI Issues

1. Check the per-check CI status for the PR/MR with forge op `ci-status`
2. Find the failing run/pipeline with forge op `ci-runs`
3. Fetch the failed job logs with forge op `ci-logs`
4. Grep the saved logs for `FAILED|ERROR` to pinpoint the failure
5. After pushing fixes, wait a minute, then re-check with forge op `ci-runs` and forge op `ci-status`

---

## Phase 6 (Both Workflows): Reflect and Update

1. **Reflect on what you learned:**
   - Design discussions with user
   - PR/MR comments received
   - Issues found during testing
   - CI/CD patterns and failures

2. **Propose updates to documentation:**
   - README.md at project root
   - Folder-level README files
   - File-level documentation comments
   - Base prompt (CLAUDE.md)
   - This custom command prompt if needed

3. **Update your notes** with anything learned

4. **Clean up logs:**
   ```bash
   rm -rf ./checks/<repo-name>-<issue-number>/
   ```

---

## Important Rules

- **ALWAYS get explicit approval before moving to each subsequent phase**
- Break down problems into manageable components
- Consider edge cases and error handling
- Use research tools to understand the codebase before proposing changes
- Examine similar functionality to follow established patterns
- When in doubt, clarify with the user rather than making assumptions
- **NEVER commit code directly to the `main` or `develop` branch**
- **ALWAYS start by checking out default branch and pulling latest changes before any analysis**
- Create a new Git worktree in order to work in parallel (see https://git-scm.com/docs/git-worktree)
- Add to your working note whenever you discover new information
- Keep your note file well organized with proper headings
- Your note file MUST be formatted in markdown
- When taking notes include permalinks to both internal and external resources
- Always use MermaidJS when documenting designs or diagramming
- Remember to use the `forge` skill (forge ops) for all platform tasks, on both GitHub and GitLab
- Use context7 and sequential-thinking tools when appropriate

## Reply Methods Summary

- **Review-thread replies** (forge op `thread-reply`) appear inline with the code, attached to specific lines; mark the thread handled with forge op `thread-resolve`
- **Top-level comments** (forge op `change-comment`) appear in the general conversation timeline
- **Never confuse the two** - they appear in different places, and only review threads can be resolved

## Workflow Summary

```
Issue URL (argument) → Extract repo/issue number → Read note file → Extract PR/MR URL
                                                                     Extract worktree path
                                                                            ↓
                                                                     Ask user: A or B?
                                                                            ↓
                                     ┌──────────────────────────────────────┴────────────────────────────────┐
                                     ↓                                                                        ↓
                         A) Comments Only                                                        B) Full Iteration
                                     ↓                                                                        ↓
                     1. Fetch inline + top-level comments                                      1. Check CI status
                     2. Address each comment                                                   2. Fix CI issues
                     3. Reply appropriately                                                    3. Fetch comments
                     4. Push & review                                                          4. Address comments
                                                                                               5. Push & review
```
