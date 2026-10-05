---
name: fix-issue
description: End-to-end workflow for issues with analysis, design, implementation, review, and submission
argument-hint: <issue-url-or-number>
---

**Requirements:** Claude Code v2.0.21+ (uses interactive question tool for structured decision capture)

## Platform

Load the `forge` skill first. It detects whether this repo is on GitHub or GitLab and gives the exact command for each forge op named below. "PR/MR" means a GitHub pull request or a GitLab merge request.

You will guide the user through a complete issue resolution workflow. This command uses the `AskUserQuestion` tool to capture structured decisions at key approval gates and design choice points, ensuring clear documentation of all choices and rationale.

Follow these phases carefully, using structured questions for all approval gates and clarifications.

## Setup Standards

**Worktree Convention:**
- Create worktrees under `./worktrees/<repo-name>-<issue-number>` relative to repository root
- Example: Working in `/path/to/my-project` on issue 92 → Create at `/path/to/my-project/../worktrees/my-project-92`
- Must be created from within the target repository

**Note File Convention:**
- Create notes under `./notes/<repo-name>-<issue-number>.md` relative to repository root
- Example: Working in `/path/to/my-project` on issue 223 → Create at `/path/to/my-project/../notes/my-project-223.md`
- Use template from `~/.claude/templates/issue-note.md`

**Commit Format:**
- Follow the commit message format specified in the project's CLAUDE.md
- Typically uses conventional commit format (e.g., `fix:`, `feat:`, `refactor:`)
- Keep messages concise (usually under 72 characters)
- No Claude attribution in commit messages

## Interactive Question Tool

**Overview:**
This command uses the `AskUserQuestion` tool to capture structured decisions at key points throughout the workflow. This provides clear documentation of choices, consistent UX, and explicit approval gates.

**Benefits:**
- **Traceability**: All decisions and rationale are explicitly captured
- **Consistency**: Standard question format across all phases
- **Nuanced approval**: 3+ options instead of binary yes/no capture levels of confidence
- **Documentation**: Answers automatically inform note-taking and design documentation

**Handling Subagent Questions:**
Subagents (issue-analyzer, solution-designer) may identify clarification questions or ambiguities. Convert their output to structured questions:

1. **Review subagent output** for "Questions for Clarification" or decision points
2. **Analyze each question** to identify 2-4 likely answers or approaches
3. **Create structured question** using `AskUserQuestion`:
   - Clear question text based on what subagent identified
   - 2-4 options representing likely approaches
   - Descriptions explaining implications of each choice
   - Appropriate header (e.g., "Pagination", "Auth", "Approach")
4. **Batch related questions** when appropriate (multiple questions in one call)
5. **Document answers** in notes under the appropriate phase section

**Key Principles:**
- All approval gates use 3+ option questions (not binary yes/no)
- Use multiSelect only for truly independent choices (e.g., documentation selection)
- Always include clear descriptions of consequences for each option

## Session Management

**Check if resuming existing work or starting fresh**

1. **Calculate note file path**:
   - Extract repository name and issue number from {{ARGS}}
   - {{ARGS}} may be a GitHub URL (`https://github.com/org/my-project/issues/123`), a GitLab URL (`https://gitlab.com/group/my-project/-/issues/123`), or a bare issue number
   - If {{ARGS}} is a bare number, take the project from the current directory via forge op `context`
   - Example: `https://github.com/org/my-project/issues/123`
     → Repo: `my-project`, Issue: `123`
   - Calculate path: `./notes/my-project-123.md`

2. **Check if note file exists**:
   - Attempt to read the note file
   - If file does NOT exist → **NEW SESSION**: Skip to Pre-Flight Checklist
   - If file EXISTS → **EXISTING WORK FOUND**: Continue below

3. **Analyze existing work** (if note file exists):
   - Read the entire note file
   - Check for a PR/MR URL in Phase 5 section

   **If PR/MR URL found**:
   ```
   ⚠️  This issue already has a submitted PR/MR!

   PR/MR: [show URL from notes]

   To work on this PR/MR, use:
   - /change-iteration - For iterating on the PR/MR (review comments, CI fixes)

   This command is for initial development only (before PR/MR submission).
   ```
   - STOP execution here - redirect user to appropriate command

   **If NO PR/MR found** (work in progress):
   - Determine last completed phase by checking note sections:
     - Phase 1 complete? → Check for analyzer output in Phase 1 section
     - Phase 2 complete? → Check for design content in Phase 2 section
     - Phase 3 started? → Check for worktree path in Phase 3 section
     - Phase 4 complete? → Check for review notes in Phase 4 section

4. **Present resume options** (if work in progress):
   - Show user the current status of all phases with visual indicators (✓ complete, ⏸ in progress, ☐ not started)
   - Include relevant details (worktree path, branch name, etc.)
   - Use `AskUserQuestion` to ask how they want to proceed:
     - Question: "How would you like to proceed with this work in progress?"
     - Header: "Resume work"
     - Option 1: "Resume from [Phase X]" - Continue from where they left off, preserving all existing analysis and design work
     - Option 2: "Start fresh" - Archive existing work and re-analyze the issue from scratch (previous work will be saved in archive/)
     - Set multiSelect to false (exclusive choice)

5. **Handle user choice**:

   **If user chooses "Resume"**:
   - Determine target repository from note file or {{ARGS}}
   - Load project context: `/prime-context <repository-directory>`
   - Read Phase 1 and Phase 2 sections from notes to refresh understanding
   - If worktree path exists in notes, verify it exists: `git worktree list`
   - Navigate to worktree if it exists
   - Skip to the next incomplete phase
   - Continue workflow from that phase

   **If user chooses "Start Fresh"**:
   - Create archive directory if it doesn't exist: `mkdir -p ./notes/archive`
   - Archive old note file with timestamp:
     - Move to: `./notes/archive/<repo-name>-<issue-number>-YYYY-MM-DD-HHMMSS.md`
     - Use current timestamp
   - If worktree path exists in old notes, remove it: `git worktree remove <path>`
   - Proceed to Pre-Flight Checklist (new session)

**Key Rules:**
- ✅ Check for existing work before starting
- ✅ Redirect to `/change-iteration` if a PR/MR exists
- ✅ Let user choose resume vs fresh start
- ✅ Preserve old work in archive (don't delete)
- ❌ Don't proceed with PR/MR iteration in this command

## Pre-Flight Checklist

**Only reached if starting NEW session (no existing note file or user chose "Start Fresh")**

1. **Read this entire command** to understand the full workflow

2. **Determine target repository**:
   - Extract the repository/project name from the issue URL in {{ARGS}} (GitHub or GitLab)
   - If {{ARGS}} is a bare number, take the project from the current directory via forge op `context`
   - Example: `https://github.com/org/my-project/issues/123`
     → Repository: `my-project`

3. **Load project context**:
   - Run `/prime-context <repository-directory>`
   - This loads CLAUDE.md, README.md into your context for all phases
   - Example: `/prime-context my-project`

4. **Create note file**:
   - Use template from `~/.claude/templates/issue-note.md`
   - Location follows Note File Convention (see Setup Standards)

5. **Verify issue exists**:
   - Fetch the issue with forge op `issue-view`
   - Confirm issue is valid and accessible

6. **Verify assignment**:
   - Ensure the issue is assigned to the current user: get the current user from forge op `context` and compare against the assignees reported by forge op `issue-view`

7. **Ensure latest code**:
   - Verify working from latest default branch (from forge op `context`, typically main/develop)
   - Pull latest changes: `git pull`

## Phase 1: Task Analysis

**Goal:** Comprehensive issue analysis with specialized subagent

1. **Launch issue-analyzer**:
   - Use Task tool with `subagent_type: "issue-analyzer"`
   - Provide the issue URL: {{ARGS}}
   - The analyzer will:
     - Load project context internally via `/prime-context`
     - Analyze the issue thoroughly
     - Examine codebase for relevant files and patterns
     - Create structured task breakdown
     - Identify risks and testing requirements
     - Flag clarification questions if needed

2. **Review analyzer output**:
   - Examine the structured analysis returned by the analyzer:
     - **Issue Summary**: High-level understanding
     - **Requirements**: Explicit and implicit requirements identified
     - **Affected Components**: Which parts of codebase need changes
     - **Task Breakdown**: Specific tasks with files, tests, dependencies, complexity
     - **Implementation Order**: Suggested sequence
     - **Risks and Considerations**: Potential challenges
     - **Testing Strategy**: Unit, integration, manual testing needs
     - **Questions for Clarification**: Any ambiguities found

3. **Document analyzer output**:
   - Copy the complete analyzer output to your note file under "Phase 1: Task Analysis" section
   - Preserve the structured format for reference in later phases

4. **Address clarification questions**:
   - If analyzer identified ambiguities in "Questions for Clarification" section
   - Convert analyzer questions into structured `AskUserQuestion` calls
   - For each clarification question:
     - Present the question clearly
     - Provide 2-4 options based on likely answers or approaches
     - Include clear descriptions of what each option means and its implications
     - Use appropriate header (e.g., "Pagination", "Auth method", "Approach")
   - Batch related clarification questions into a single tool call when appropriate
   - Document all answers in notes under Phase 1 section
   - Update understanding based on clarifications

5. **Confirm understanding with user**:
   - Present the analyzer's summary and task breakdown to the user
   - Use `AskUserQuestion` to confirm understanding:
     - Question: "Does the analysis accurately capture what needs to be done?"
     - Header: "Confirm"
     - Option 1: "Yes, proceed" - Analysis is complete and accurate, ready to move to solution design
     - Option 2: "Needs revision" - Some requirements are misunderstood or missing, user will provide corrections
     - Option 3: "Major concerns" - Significant issues with the analysis, need to re-analyze
     - Set multiSelect to false
   - If user chooses "Needs revision" or "Major concerns", gather feedback and update notes
   - Document user's feedback and any adjustments in notes

6. **Mark Phase 1 complete**:
   - Once user confirms understanding is correct
   - Proceed to Phase 2 ONLY after explicit user approval

**Key Rules:**
- ❌ Don't write any implementation code during this phase
- ✅ Use issue-analyzer for comprehensive analysis
- ✅ Document analyzer output in notes for later reference
- ✅ Address all clarification questions before proceeding
- ✅ Get explicit user confirmation before Phase 2

## Phase 2: Solution Design

**Goal:** Create comprehensive solution design using specialized subagent

1. **ONLY after user confirms Phase 1 understanding**, begin solution design

2. **Launch solution-designer subagent**:
   - Use Task tool with `subagent_type: "solution-designer"`
   - Provide context from Phase 1:
     - The issue URL or description
     - Repository directory name
     - Analyzer's task breakdown from Phase 1 notes
   - The designer will:
     - Load project context internally via `/prime-context`
     - Review Phase 1 analysis (task breakdown, affected components, risks)
     - Design comprehensive solution with multiple approaches
     - Create detailed implementation plan
     - Develop testing strategy
     - Identify deployment considerations

3. **Review solution-designer output**:
   - Examine the detailed solution design returned by the designer:
     - **Approach Overview**: High-level strategy
     - **Implementation Details**: Files to modify/create, code changes, database changes
     - **Testing Strategy**: Unit tests, integration tests, validation approach
     - **Deployment Plan**: Step-by-step deployment and rollback procedures
     - **Risks and Mitigations**: Potential issues and how to address them
     - **Assumptions**: Any assumptions made during design
     - **Follow-up Tasks**: Future improvements

4. **Document solution-designer output**:
   - Copy the complete solution design to your note file under "Phase 2: Solution Design" section
   - Preserve the structured format for reference during implementation

5. **Discuss design alternatives with user**:
   - If designer proposed multiple approaches, present them clearly with pros/cons
   - Reference analyzer's risk assessment from Phase 1
   - Use `AskUserQuestion` to capture design decisions:

     **When multiple design approaches exist:**
     - Question: "Which design approach should we implement?"
     - Header: "Approach"
     - Provide 2-4 options representing different design approaches
     - Each option should include clear description of trade-offs (complexity, performance, maintainability, etc.)
     - Set multiSelect to false

     **For design priorities and trade-offs (can ask multiple questions together):**
     - Question 1: "What's your priority for this implementation?"
       - Options: "Speed to market" (simplicity, quick delivery), "Performance" (optimize for efficiency), "Maintainability" (code clarity, testing)
       - Header: "Priority"
     - Question 2: "Should we follow existing patterns or introduce improvements?"
       - Options: "Follow existing" (consistency), "Introduce better" (modernize), "Mixed approach" (balanced)
       - Header: "Pattern"
     - Set multiSelect to false for each

   - Adapt questions based on what the designer proposed and what decisions need to be made
   - Document all answers and rationale in notes

6. **Refine based on user feedback**:
   - Incorporate user's preferences into the design
   - Adjust solution to address user concerns
   - Resolve any design conflicts or questions
   - Update notes with selected approach and rationale

7. **Additional considerations**:
   - **For integration tasks**: Check if related repositories need coordinated changes (create issues if needed)
   - **For CI/CD integration**: Verify all required workflow files are identified
   - Ensure final design aligns with analyzer's testing strategy from Phase 1

8. **Finalize design documentation**:
   - Ensure "Phase 2: Solution Design" section in notes is complete
   - Add MermaidJS diagrams if helpful (or use diagrams from solution-designer)
   - Document final design decisions and rationale
   - Reference how design addresses each task from Phase 1 breakdown
   - Note any deviations from analyzer's or designer's suggestions with justification

9. **Request explicit approval**:
   - Summarize the complete design approach
   - Highlight key decisions made
   - Use `AskUserQuestion` to get approval:
     - Question: "Are you ready to proceed with implementation?"
     - Header: "Proceed"
     - Option 1: "Yes, proceed" - Design is approved, begin implementation in Phase 3
     - Option 2: "Need changes" - Design needs refinement, user will provide feedback on specific areas
     - Option 3: "Reconsider approach" - Fundamental concerns with the approach, let's explore alternatives
     - Set multiSelect to false
   - If user chooses "Need changes" or "Reconsider approach", gather feedback and refine design
   - Proceed to Phase 3 ONLY after explicit user approval

**Key Rules:**
- ❌ Don't write any implementation code during this phase
- ✅ Use solution-designer for comprehensive design
- ✅ Build on analyzer's task breakdown from Phase 1
- ✅ Document solution-designer output in notes for later reference
- ✅ Discuss alternatives with user before finalizing
- ✅ Use MermaidJS for diagrams when helpful
- ✅ Get explicit approval before Phase 3

## Phase 3: Implementation

**Goal:** Execute the approved solution using specialized implementation subagent

1. **ONLY after explicit approval**, begin implementing the solution

2. **Launch issue-implementer subagent**:
   - Use Task tool with `subagent_type: "issue-implementer"`
   - Provide the note file path as input (e.g., `./notes/<repo-name>-<issue-number>.md`)
   - The implementer will:
     - Read note file to understand context from Phase 1 and Phase 2
     - Prime project context via `/prime-context`
     - Create git worktree following convention
     - Implement all tasks in order from Phase 1 breakdown
     - Follow solution design from Phase 2
     - Update Phase 3 section of note file with progress
     - Handle edge cases and error resilience
     - Test functionality incrementally
     - Update CHANGELOG.md if it exists in the project
     - Document all changes for Phase 5 commit preparation

3. **Monitor implementation progress**:
   - Review the Phase 3 section updates in the note file
   - The implementer will document:
     - Worktree path and branch name
     - Progress on each task from Phase 1
     - Key decisions made during implementation
     - Any deviations from Phase 2 design with rationale
     - Test results
     - Changelog updates

4. **Review completed implementation**:
   - Once implementer completes, review the note file updates
   - Verify all tasks from Phase 1 are implemented
   - Check that design from Phase 2 was followed
   - Confirm tests are passing

5. **Request explicit approval**:
   - Summarize what was implemented
   - Highlight any deviations from the design
   - Use `AskUserQuestion` to get approval:
     - Question: "Is the implementation ready for code review?"
     - Header: "Review"
     - Option 1: "Yes, review it" - Implementation looks good, proceed to code review phase
     - Option 2: "Let me test first" - User wants to manually test or review changes before formal review
     - Option 3: "Needs changes" - User identified issues that need to be fixed first
     - Set multiSelect to false
   - If user chooses "Let me test first" or "Needs changes", wait for their feedback
   - Proceed to Phase 4 ONLY after explicit user approval

**Key Rules:**
- ✅ Use issue-implementer for implementation
- ✅ Build on Phase 1 breakdown and Phase 2 design from notes
- ✅ Monitor progress through note file updates
- ✅ Get explicit approval before Phase 4
- ❌ Don't implement code directly in this command (use the subagent)

## Phase 4: Review

**⚠️ Important: Use the code-reviewer subagent - don't skip this step**
- The code-reviewer is required for all changes
- Don't perform manual reviews instead
- Even for "simple" changes, the subagent provides structured analysis

**Goal:** Ensure code quality and correctness

**Use code-reviewer for ALL changes including:**
- ✅ Application code changes (Python, TypeScript, etc.)
- ✅ Configuration file changes (YAML, JSON, TOML)
- ✅ CI workflow files (GitHub Actions `.github/workflows/`, GitLab `.gitlab-ci.yml`)
- ✅ Infrastructure as Code (Terraform, CloudFormation)
- ✅ Documentation changes (Markdown, RST)
- ✅ Simple find-replace operations
- ✅ Single-line changes
- ✅ Dependency updates (pyproject.toml, package.json)

**NO EXCEPTIONS - Always launch the subagent regardless of change complexity**

1. Launch `code-reviewer` subagent to review the changes

2. Based on feedback, suggest improvements for robustness, performance, or readability

3. Update the note file with learnings from the review

4. Notify user about any new TODOs or areas needing attention

5. **Validate against analyzer's testing strategy**:
   - Review the Testing Strategy from Phase 1 (in your notes)
   - Ensure unit tests cover areas identified by analyzer
   - Verify integration tests address components flagged by analyzer
   - Confirm manual testing checklist matches analyzer's recommendations

6. **For Python changes**:
   - Run `ruff check --fix --exclude alembic`
   - Run `pytest` (unit and integration tests)
   - Run `mypy`
   - Iterate until all checks pass

7. **For Terraform changes**:
   - Run `terraform fmt -check` to verify formatting
   - Check for obvious syntax errors in the diff
   - Note that full `terraform plan` may require specific environment access

8. **Decide on next steps after review**:
   - Present code review findings to the user
   - Use `AskUserQuestion` to determine how to proceed:
     - Question: "How should we proceed after code review?"
     - Header: "Next step"
     - Option 1: "Address issues" - Fix the issues identified in review before submitting the PR/MR
     - Option 2: "Submit anyway" - Issues are minor, create the PR/MR and address in review comments
     - Option 3: "Major revision" - Significant issues found, need to revisit design or implementation
     - Set multiSelect to false
   - If user chooses "Address issues", make necessary fixes and re-run tests
   - If user chooses "Major revision", return to appropriate phase (Phase 2 or Phase 3)
   - If user chooses "Submit anyway", proceed to Phase 5

**Key Rules:**
- ✅ Use code-reviewer subagent for all changes
- ✅ Validate against analyzer's testing strategy
- ✅ All tests should pass before proceeding
- ✅ Document learnings in notes
- ✅ Get explicit decision on how to handle review findings

## Phase 5: Submit

**Goal:** Create the PR/MR with proper commit and push

1. **Get final approval before submission**:
   - Use `AskUserQuestion` to confirm readiness:
     - Question: "Ready to create the PR/MR?"
     - Header: "Submit"
     - Option 1: "Yes, create it" - All checks passed, create and submit the PR/MR
     - Option 2: "Review changes" - User wants to review the diff one more time before submission
     - Option 3: "Wait" - Not ready yet, user needs to make additional changes or tests
     - Set multiSelect to false
   - If user chooses "Review changes", show the diff and wait for confirmation
   - If user chooses "Wait", pause until user is ready
   - ONLY proceed with submission after user chooses "Yes, create it"

2. **Navigate to worktree**:
   - Change to the worktree directory from Phase 3 (documented in notes)
   - Verify you're on the correct branch: `git status`

3. **Verify changelog is updated if required**:
   - Check if project has CHANGELOG.md: `find . -iname "*changelog*" -type f`
   - If changelog exists and you made bug fixes or features, verify it was updated in Phase 3
   - If not updated, add appropriate entry now

4. **Stage only relevant project files**:
   - Stage implementation changes: `git add <files-modified-in-phase-3>`
   - Stage CHANGELOG.md if it exists and was modified
   - **EXCLUDE all files from notes/ directory**
   - **EXCLUDE CLAUDE.md file**
   - Verify staging: `git status`

5. **Create commit message**:
   - Follow project's commit format from CLAUDE.md (typically conventional commits)
   - Format: `<type>: <concise description under 72 chars>`
   - Types: `fix:`, `feat:`, `refactor:`, `docs:`, `test:`, `chore:`
   - Base message on changes from Phase 3
   - **DO NOT include "Generated with Claude Code" attribution**
   - Example: `feat: add pagination support to user list endpoint`

6. **Commit and push to remote**:
   - Commit with the message: `git commit -m "<commit-message>"`
   - Push to remote: `git push -u origin <branch-name>`

7. **Create the PR/MR**:
   - Open the PR/MR with forge op `change-create`, targeting the default branch (from forge op `context`)
   - Include detailed description following the project's PR/MR template if one exists (`.github/pull_request_template.md` on GitHub, `.gitlab/merge_request_templates/` on GitLab)
   - Include sections: Summary, Problem, Solution, Test Plan
   - Do not include "Generated with Claude Code" footer

8. **Add PR/MR URL to notes**:
   - Copy the PR/MR URL returned by forge op `change-create`
   - Add to Phase 5 section in your note file
   - Document commit message used

**Key Rules:**
- ❌ Don't commit notes/ or CLAUDE.md files
- ✅ Verify changelog was updated during Phase 3
- ✅ Create conventional commit message based on changes
- ✅ Use the PR/MR template
- ✅ Document PR/MR URL and commit message in notes
- ✅ Get approval before submitting

## Phase 6: Reflect

**Goal:** Capture learnings for future improvements

1. **Reflect on learnings**:
   - Design discussions with user
   - Issues found during testing
   - Code review feedback
   - Anything learned about the codebase or domain

2. **Propose documentation updates**:
   - Use `AskUserQuestion` to identify which documentation should be updated:
     - Question: "Which documentation should be updated based on learnings?"
     - Header: "Docs"
     - Set multiSelect to true (allow multiple selections)
     - Option 1: "README.md" - Update project root README with new features or setup instructions
     - Option 2: "CLAUDE.md" - Update base prompt with new patterns or learnings
     - Option 3: "This command" - Improve fix-issue.md workflow based on this experience
     - Option 4: "Code comments" - Add/improve inline documentation and docstrings
   - For each selected documentation type, make appropriate updates

3. **Update notes with key learnings**

4. **Use note-header-generator** subagent:
   - Analyze the note file
   - Add structured metadata header

5. **Workflow complete**:
   - Initial development work is done
   - For PR/MR iteration (review comments, CI fixes), use:
     - `/change-iteration` - For iterating on the PR/MR

**Key Rules:**
- ✅ Document all learnings
- ✅ Propose documentation improvements
- ✅ Complete note file with header metadata
- ✅ Direct user to `/change-iteration` for next steps

## Important Rules Summary

- ❌ **Don't run tools concurrently**
- ❌ **Don't write code during Phase 1 or 2**
- ❌ **Don't commit directly to main or develop branch**
- ❌ **Don't commit notes/ or CLAUDE.md files**
- ❌ **Don't skip approval gates between phases**
- ❌ **Don't use binary yes/no questions for approval gates**
- ❌ **Don't handle PR/MR iteration in this command (use `/change-iteration`)**
- ✅ **Check for existing work at session start**
- ✅ **Redirect to `/change-iteration` if a PR/MR already exists**
- ✅ **Use AskUserQuestion for all approval gates and key decisions**
- ✅ **Provide 3+ options for approval gates (not binary yes/no)**
- ✅ **Convert subagent clarification questions to structured AskUserQuestion calls**
- ✅ **Document user's answers and rationale in notes**
- ✅ **Get explicit approval before moving to next phase**
- ✅ **Start by checking out default branch and pulling latest changes**
- ✅ **Create git worktree following convention**
- ✅ **Use issue-analyzer for Phase 1 analysis**
- ✅ **Update notes when discovering new information**
- ✅ **Use MermaidJS for diagrams**
- ✅ **Include permalinks to resources in notes**
- ✅ **Use the `forge` skill (forge ops) for all platform operations, on both GitHub and GitLab**
- ✅ **Use context7 and sequential-thinking tools**

## Forge Ops Used in This Command

| Forge op | Where |
|---|---|
| `context` | Session management / pre-flight: project path, default branch, current user |
| `issue-view` | Pre-flight: verify the issue exists, is accessible, and is assigned to the current user |
| `change-create` | Phase 5: open the PR/MR and get its URL |
