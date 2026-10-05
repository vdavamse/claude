---
name: review-change
description: Expert code review with inline comments directly on PR/MR files
argument-hint: <pr-mr-url-or-number>
---

## Platform

Load the `forge` skill first. It detects whether this repo is on GitHub or GitLab and gives the exact command for each forge op named below. "PR/MR" means a GitHub pull request or a GitLab merge request.

You are a skilled code reviewer with expertise in software engineering best practices, security, and performance optimization. You provide concise, actionable feedback through inline comments on specific code locations.

## Workflow Overview

This command performs comprehensive PR/MR reviews and posts findings as **inline comments** directly on the change, attached to specific files and lines where issues are found.

## Setup

### 1. Parse PR/MR Information

Extract from the {{ARGS}} argument:
- Project path (e.g., `acme/my-service`)
- PR/MR number (e.g., `123`)

Accepted inputs:
- GitHub URL: `https://github.com/acme/my-service/pull/123` → Project: `acme/my-service`, Number: `123`
- GitLab URL: `https://gitlab.com/acme/my-service/-/merge_requests/123` → Project: `acme/my-service`, Number: `123`
- Bare number (`123`) or empty → take the project path from forge op `context` and, when empty, the current branch's PR/MR number from forge op `change-for-branch`

### 2. Navigate to Repository

**IMPORTANT**: Navigate to the correct repository directory before performing any git operations.

From `{{ARGS}}`:
- Extract the project name from the URL
- Navigate to the repository directory
- Example: for `acme/my-service` → navigate to `./my-service`

See CLAUDE.md section **Repository Navigation** for details.

### 3. Fetch PR/MR Details

Fetch state, source and target branches, head SHA, mergeability, CI rollup, files, and commits with forge op `change-view`. Note whether the change comes from a fork or from a branch in the same project — this affects how you fetch the code in the next step.

### 4. Create Git Worktree for Review

**Create a dedicated worktree to review the code:**

Fetch the PR/MR head into a local branch `review-<NUMBER>` and add a worktree for it with forge op
`change-checkout` (this works for forks too), then:

```bash
cd ../worktrees/<repo-name>-review-<NUMBER>
```

**Why use a worktree?**
- Allows reviewing actual code in context
- Can run linters, tests, and type checkers
- Maintains clean main branch while reviewing
- Easy to switch between multiple reviews

### 5. Load Project Context

If the repository has a CLAUDE.md file, load it using:
```bash
/prime-context <repository-directory>
```

This loads project-specific conventions, standards, and best practices.

## Phase 1: Analyze Changes

### 1.1 Understand Scope

Read the PR/MR description and metadata from the forge op `change-view` output (Setup step 3), then fetch the full diff and the name-only list of changed files with forge op `change-diff`.

### 1.2 Examine Code Changes in Worktree

Navigate to the worktree and examine:

```bash
# Files modified in this PR/MR
git diff --name-only origin/<base-branch>...

# Commits in this PR/MR
git log --no-decorate origin/<base-branch>...

# Full diff content
git diff origin/<base-branch>...
```

### 1.3 Get Current Commit SHA

**CRITICAL**: You need the latest head commit SHA to anchor inline comments:

```bash
# Get the HEAD commit of the source branch
git rev-parse HEAD
```

Store this SHA - you'll need it for creating inline comments.

## Phase 2: Comprehensive Code Review

### 2.1 Use code-reviewer Agent

Launch the `code-reviewer` subagent to analyze the changes:

```bash
# From within the worktree, the agent will analyze:
# - Code quality and maintainability
# - Security vulnerabilities
# - Performance issues
# - Best practices violations
# - Test coverage
```

The code-reviewer agent will provide structured feedback with:
- Specific file paths
- Line numbers where issues occur
- Severity level (critical, major, minor)
- Specific recommendations

### 2.2 Run Project-Specific Checks

**For Python projects:**
```bash
# Linting
ruff check --exclude alembic

# Type checking
mypy

# Tests
pytest
```

**For TypeScript/JavaScript projects:**
```bash
# Linting
npm run lint

# Type checking
npm run type-check

# Tests
npm test
```

**For Terraform projects:**
```bash
# Formatting check
terraform fmt -check -recursive

# Validation
terraform validate
```

### 2.3 Review Key Areas

Focus on:
- **Security**: Authentication, authorization, input validation, secret management
- **Performance**: Database queries, API calls, caching, algorithm complexity
- **Maintainability**: Code organization, naming, documentation, test coverage
- **Best Practices**: Following project conventions, design patterns, error handling
- **Breaking Changes**: API changes, database migrations, configuration changes

## Phase 3: Post Inline Comments

### Understanding Inline Comments

**CRITICAL**: All issues found must be posted as **inline comments** on specific file lines, NOT as general PR/MR comments.

### 3.1 Prepare Review Data

For each issue found, collect:
1. **File path** (relative to repository root)
2. **Line number** where the issue occurs, and which side of the diff it belongs to (new-file side for added/modified lines, old-file side for deleted lines)
3. **Commit SHA** (from Phase 1.3)
4. **Issue description** (specific, actionable feedback)
5. **Severity** (critical/major/minor)
6. **Suggested fix** (if applicable)

### 3.2 Post Each Inline Comment

Write each comment body to a file first (e.g. with a heredoc in a scratch directory) and pass it by file reference where the forge command supports it — long markdown bodies (backticks, quotes, `$`) are fragile inside inline shell strings, and a file avoids all escaping issues.

Post every finding as a single comment on its diff line with forge op `inline-comment`, providing: the body, the head commit SHA from Phase 1.3, the file path, the line number, and the diff side.

**Example body file:**

````markdown
**🔴 Critical — Security**: SQL query is vulnerable to injection. Use parameterized queries instead.

Suggested fix:
```python
cursor.execute('SELECT * FROM users WHERE id = %s', (user_id,))
```
````

### 3.3 Comment Structure Guidelines

**Format each inline comment as:**

```markdown
**[SEVERITY]: [Issue Type]**: Brief description

[Detailed explanation of the issue]

**Why this matters:**
[Impact and consequences]

**Suggested fix:**
```[language]
[Code example showing the fix]
```

**Additional context:** [Any relevant documentation, patterns, or examples]
```

**Severity Levels:**
- 🔴 **Critical**: Security vulnerabilities, data loss risks, breaking changes
- 🟡 **Major**: Performance issues, maintainability concerns, best practice violations
- 🔵 **Minor**: Code style, documentation, minor improvements

### 3.4 Post Comments in Sequence

For efficiency, write all bodies to files first (`comment-1.md`, `comment-2.md`, …), then post them one by one with forge op `inline-comment`.

### 3.5 Verify Posted Comments (MANDATORY)

After posting, confirm the bodies actually rendered as review text — a mangled body still posts "successfully" and only shows up when you read the comments back.

Fetch the existing inline comments with forge op `comments-list-inline`, save the output to a local JSON file, and check every comment you posted:

- the body starts with its severity marker (🔴/🟡/🔵) and contains the expected markdown
- the body is not a literal file path (e.g. `@comment-1.md`) or a raw shell fragment

With the output saved as `comments.json`, a check like this must print nothing (any hit is a broken comment body):

```bash
jq -r '.[] | select(.body | startswith("@") or (test("[🔴🟡🔵]|\\*\\*") | not))
       | "BROKEN \(.id) \(.path):\(.line) :: \(.body[0:80])"' comments.json
```

A broken comment is repairable **in place** (same thread, same URL) — do not delete and repost: fix it with forge op `comment-edit`.

## Phase 4: Summary and Report

### 4.1 Post Review Summary (Optional)

After posting all inline comments, you may optionally post a high-level summary as a top-level comment with forge op `change-comment`:

```markdown
## Code Review Summary

**Total Issues Found:** X
- 🔴 Critical: X
- 🟡 Major: X
- 🔵 Minor: X

**Areas Reviewed:**
- Security ✓
- Performance ✓
- Tests ✓
- Documentation ✓

**Key Findings:**
All detailed feedback has been posted as inline comments on specific files.

**Next Steps:**
Please review the inline comments and address the issues found.
```

### 4.2 Provide User Summary

Present to the user:
```markdown
## Review Complete ✓

Posted **[N]** inline comments on PR/MR #[NUMBER]

**Breakdown:**
- 🔴 Critical: [N] issues
- 🟡 Major: [N] issues
- 🔵 Minor: [N] suggestions

**Files Reviewed:**
- [file1.py] - [N] comments
- [file2.py] - [N] comments
- [file3.py] - [N] comments

All feedback has been posted as inline comments directly on the PR/MR.
View them at: [CHANGE_URL]
```

## Phase 5: Cleanup

```bash
# Return to original directory
cd -

# Remove the review worktree
git worktree remove ../worktrees/<repo-name>-review-<NUMBER>

# Clean up the temporary branch
git branch -D review-<NUMBER>
```

## Important Rules

### ✅ DO:
- **ALWAYS** navigate to correct repository before git operations
- **ALWAYS** create worktree for reviewing code
- **ALWAYS** post findings as inline comments on specific lines
- **ALWAYS** get the latest head commit SHA before creating comments
- **ALWAYS** use code-reviewer agent for comprehensive analysis
- **ALWAYS** provide actionable, specific feedback
- **ALWAYS** include severity levels in comments
- **ALWAYS** suggest concrete fixes with code examples
- **ALWAYS** verify file paths are relative to repository root
- **ALWAYS** anchor comments to the new-file side for added/modified lines and the old-file side for deleted lines
- **ALWAYS** verify posted comment bodies after posting (Phase 3.5)

### ❌ DON'T:
- **NEVER** post general PR/MR comments instead of inline comments
- **NEVER** skip creating worktree (you need actual code to review)
- **NEVER** guess commit SHAs (always fetch the actual head SHA)
- **NEVER** post vague feedback without specific line references
- **NEVER** forget to clean up worktree after review
- **NEVER** use absolute file paths (always relative to repo root)
- **NEVER** create comments without proper severity classification
- **NEVER** leave a broken comment body in place — repair it with forge op `comment-edit`

## Troubleshooting

### Issue: "Comments not appearing on the PR/MR"
**Solution**: Verify you anchored to the correct head commit SHA and to lines that exist in the diff

### Issue: "Invalid path error"
**Solution**: Ensure file path is relative to repository root, not absolute

### Issue: "Line number not found"
**Solution**: The change may have been updated. Get a fresh head SHA and line numbers

### Issue: "Permission denied"
**Solution**: Ensure you have write access to the repository

## Workflow Summary

```
PR/MR URL or number → Parse info → Navigate to repo → Create worktree → Load context
                                                          ↓
                                                Get latest head commit SHA
                                                          ↓
                                  Review code with code-reviewer agent
                                                          ↓
                                Run linters, tests, type checkers
                                                          ↓
                                  Identify issues with severity
                                                          ↓
                        Post inline comment for EACH issue found
                        (file path, line number, specific feedback)
                                                          ↓
                                 Optional: Post review summary
                                                          ↓
                                        Cleanup worktree
```

## Example Workflow

```bash
# 1. Setup — forge op `change-checkout` creates branch review-123 and its worktree
cd ../worktrees/my-service-review-123

# 2. Get commit SHA
COMMIT_SHA=$(git rev-parse HEAD)
```

3. Review with the `code-reviewer` agent — it identifies issues with file paths and line numbers
4. Write each finding's body to a file and post it with forge op `inline-comment` (path, line, side, `$COMMIT_SHA`)
5. Verify no broken bodies with forge op `comments-list-inline` (the check must print nothing — see Phase 3.5); repair any with forge op `comment-edit`
6. Cleanup:

```bash
cd -
git worktree remove ../worktrees/my-service-review-123
git branch -D review-123
```
