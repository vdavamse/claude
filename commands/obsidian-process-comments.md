---
argument-hint: <note-file-path>
description: Process _NOTE_ comments in an Obsidian note
allowed-tools: mcp__mcp-obsidian__*
---

# Process Obsidian Note Comments

Process all `_NOTE_` comments in the specified Obsidian note file.

## Context

- **Note file to process**: $ARGUMENTS
- **Working directory**: $OBSIDIAN_VAULT
- `$OBSIDIAN_VAULT` is an environment variable holding the vault path: resolve it with `echo $OBSIDIAN_VAULT` and ask the user if it is unset.

Note paths should be relative to the vault root (e.g., `01-Daily/2025-01-15.md` or `03-Learning/Permanent/My-Note.md`).

## Your Task

Process all `_NOTE_` comments in the specified note following this iterative workflow:

### Phase 1: Discover Comments

**Goal**: Find all unprocessed comments in the note

1. **Read the note file** using `obsidian_get_file_contents($ARGUMENTS)`
2. **Search for all `_NOTE_` markers** (inline comments in the note)
3. **Extract each comment** with **unique surrounding context** (critical for handling duplicates):
   - The comment text itself
   - The heading/section where it appears
   - **At least 5-10 words before and after the comment** (ensures uniqueness)
   - Store this context for later use when marking as processed
4. **Display to user**:
   ```
   Found X _NOTE_ comments in $ARGUMENTS:

   1. [Section: ## Introduction]
      Comment: "_NOTE_ add explanation about the core concept"
      Context: "This is a note about... _NOTE_ add explanation..."

   2. [Section: ## Examples]
      Comment: "_NOTE_ include code example"
      Context: "The following shows... _NOTE_ include code example"
   ```
5. **Ask for confirmation**: "Would you like to process these comments? (yes/no/specific numbers)"

**Key Rules for Phase 1**:
- ✅ Show ALL found comments before proceeding
- ✅ Include enough context for user to understand each comment
- ❌ Don't process anything yet - just discovery

---

### Phase 2: Process Each Comment Iteratively

**Goal**: Handle each comment one at a time with full user control

For **each** `_NOTE_` comment found:

#### Step 1: Present the Comment
Show the user:
```
Processing comment X of Y:

Location: [Section name and context]
Comment: "_NOTE_ [comment text]"

Surrounding context:
[Show 2-3 lines before and after the comment]
```

#### Step 2: Clarify Intent
**ALWAYS ask the user**: "What would you like me to do based on this comment?"

If the comment is clear, you can suggest an interpretation:
```
It looks like you want to [interpretation]. Is that correct?
If so, where in the note should I make this change?
```

If unclear, ask questions:
```
This comment says "[comment text]". I need clarification:
- What specific content should I add/modify?
- Which section of the note should this go in?
- Should I create a new section or update an existing one?
```

#### Step 3: Understand Scope
Based on user's response, determine:
- **What to change**: New section, update existing content, add examples, reorganize, etc.
- **Where to change it**: Specific heading, frontmatter, anywhere in the note
- **How to change it**: Append, prepend, replace, or restructure

**Important**: Changes can be made ANYWHERE in the note, not just near the comment location.

#### Step 4: Propose the Change
Draft the specific changes and show to user:
```
I'll make the following changes to $ARGUMENTS:

[Section: ## Target Section]
Action: [Append/Prepend/Replace/Create new section]

New/Modified content:
---
[Show the exact content that will be added/modified]
---

Does this look correct? (yes/no/modify)
```

If user says "modify", ask what to adjust and repeat this step.

#### Step 5: Apply the Change
Once confirmed:
1. Use `obsidian_patch_content()` to modify the target section, OR
2. Use `obsidian_put_content()` if restructuring the entire note
3. Confirm the change was applied successfully

#### Step 6: Mark Comment as Processed
Replace the original `_NOTE_` marker with `_PROCESSED_`:

**CRITICAL**: Use sufficient surrounding context to avoid replacing duplicate comment text elsewhere in the note.

**Safe Replacement Strategy**:
1. **Include surrounding text** in the replacement operation (NOT just the comment text)
2. Use `obsidian_patch_content()` with enough context to make it unique
3. **Never use just `_NOTE_` alone** - always include unique surrounding text

**Example of SAFE replacement**:
```markdown
# Correct approach - uses surrounding context
old_string: "Add more details here _NOTE_ explain the benefits"
new_string: "Add more details here _PROCESSED_ explain the benefits"

# WRONG approach - would replace ALL instances
old_string: "_NOTE_ explain the benefits"  ❌ DON'T DO THIS
new_string: "_PROCESSED_ explain the benefits"
```

**Handling identical comment text on different lines**:
If the note has:
```
Line 10: Some text _NOTE_ add more details
Line 50: Other text _NOTE_ add more details
```

When processing Line 10's comment:
- ✅ Replace: "Some text _NOTE_ add more details" → "Some text _PROCESSED_ add more details"
- ❌ Don't replace just: "_NOTE_ add more details" (would affect both lines)

This ensures only the specific comment being processed is marked, not all identical comments.

#### Step 7: Move to Next Comment
Ask user: "Continue to next comment? (yes/no/stop)"
- If yes: Proceed to next comment
- If no/stop: Skip to Phase 3 (Summary)

**Key Rules for Phase 2**:
- ✅ Process ONE comment at a time
- ✅ ALWAYS get user clarification before proposing changes
- ✅ ALWAYS get user confirmation before applying changes
- ✅ Changes can be ANYWHERE in the note (not just near the comment)
- ✅ Mark each comment as `_PROCESSED_` after handling
- ✅ **CRITICAL**: Use surrounding context when marking comments to avoid affecting duplicate comment text
- ❌ NEVER assume what a comment means - always ask
- ❌ NEVER make changes without explicit user approval
- ❌ NEVER delete the comment marker - mark as processed instead
- ❌ **NEVER use just the comment text alone for replacement** - always include unique surrounding text

---

### Phase 3: Summary

**Goal**: Confirm completion and show what was accomplished

1. **Show summary**:
   ```
   Processing complete for $ARGUMENTS

   Changes made:
   - Comment 1: [Added new section "Benefits"]
   - Comment 3: [Updated examples in "Usage" section]

   Skipped:
   - Comment 2: [User chose to skip]

   Remaining unprocessed:
   - 0 _NOTE_ comments remaining
   ```

2. **Ask if user wants to review**: "Would you like me to show the updated note content?"

3. **Confirm completion**: "All requested comments have been processed!"

**Key Rules for Phase 3**:
- ✅ Show clear summary of what changed
- ✅ Indicate any remaining `_NOTE_` comments
- ✅ Offer to display the updated note

---

## Important Guidelines

### Always Ask for Clarification
- If a comment is ambiguous, ALWAYS ask the user what they mean
- Don't make assumptions about intent
- Clarify scope, location, and content before proceeding

### Get Confirmation at Each Step
- Show proposed changes before applying them
- Wait for explicit "yes" or approval
- Allow user to modify, skip, or stop at any point

### Preserve Note Structure
- Maintain existing formatting and organization
- Use proper heading levels
- Preserve links, tags, and frontmatter
- Ensure changes integrate smoothly with existing content

### Use Obsidian MCP Tools
- `obsidian_get_file_contents()` - Read the note
- `obsidian_patch_content()` - Modify specific sections (heading, block, frontmatter)
- `obsidian_put_content()` - Overwrite entire note (use sparingly)
- Never use file I/O tools - only Obsidian MCP tools

### Mark Comments Appropriately
- Replace `_NOTE_` with `_PROCESSED_` after handling
- Keep the comment text for reference
- Don't delete the marker - this helps track what's been done

### Handle Duplicate Comments Safely

**The Challenge**: The same `_NOTE_` comment text may appear multiple times in a note.

Example:
```markdown
Line 15: This section needs work _NOTE_ add more details
Line 89: That section also needs work _NOTE_ add more details
```

**The Solution**: Always use surrounding context when marking comments as processed.

**Implementation**:
1. **During Phase 1**: Store each comment with unique surrounding text (5-10 words before/after)
2. **During Step 6**: Use that surrounding context in the replacement operation

**Replacement Pattern**:
```markdown
# ✅ CORRECT - includes surrounding context
old_string: "This section needs work _NOTE_ add more details"
new_string: "This section needs work _PROCESSED_ add more details"

# ❌ WRONG - would replace all instances
old_string: "_NOTE_ add more details"
new_string: "_PROCESSED_ add more details"
```

**Why This Matters**: Without surrounding context, marking one comment as processed would inadvertently mark ALL identical comments, preventing you from processing the others properly.

---

## Examples

### Example 1: Adding Content to Existing Section

**Comment found**:
```
## Background
This explains the context. _NOTE_ add example of real-world usage
```

**Clarification**:
"What real-world usage example would you like me to add?"

**User response**:
"Add an example of how this applies to Kubernetes deployments"

**Proposed change**:
```
I'll add to the "Background" section:

### Real-World Example
In Kubernetes deployments, this concept applies when...
```

**After approval** → Apply change and mark:
```
## Background
This explains the context. _PROCESSED_ add example of real-world usage

### Real-World Example
In Kubernetes deployments, this concept applies when...
```

### Example 2: Creating New Section

**Comment found**:
```
_NOTE_ create troubleshooting section with common issues
```

**Clarification**:
"Where should I place the troubleshooting section? After which existing section?"

**User response**:
"Add it after the Examples section"

**Proposed change**:
```
I'll create a new section after "## Examples":

## Troubleshooting

### Common Issues
...
```

**After approval** → Apply change and mark comment as processed

### Example 3: Updating Frontmatter

**Comment found**:
```
_NOTE_ add tags for kubernetes and devops
```

**Proposed change**:
```
I'll update the frontmatter:

tags: [kubernetes, devops, learning]
```

**After approval** → Use `obsidian_patch_content()` with `target_type: frontmatter`

### Example 4: Handling Duplicate Comment Text

**Scenario**: Two identical comments in different locations

Note content:
```markdown
## Introduction
The basics are covered here. _NOTE_ add more details

## Advanced Topics
This section is incomplete. _NOTE_ add more details
```

**Processing Comment 1** (in Introduction section):

1. **Show comment with context**:
   ```
   Processing comment 1 of 2:

   Location: ## Introduction
   Context: "The basics are covered here. _NOTE_ add more details"
   ```

2. **Clarify**: User says "Add explanation about core concepts"

3. **Apply change**: Add content to Introduction section

4. **Mark as processed** (CRITICAL - use surrounding context):
   ```
   ✅ CORRECT replacement:
   old_string: "The basics are covered here. _NOTE_ add more details"
   new_string: "The basics are covered here. _PROCESSED_ add more details"
   ```

Result after Comment 1:
```markdown
## Introduction
The basics are covered here. _PROCESSED_ add more details

[New content added about core concepts]

## Advanced Topics
This section is incomplete. _NOTE_ add more details  ← Still unprocessed!
```

**Processing Comment 2** (in Advanced Topics section):

Now process the second comment independently with its own context.

**Key Point**: Because we used surrounding context ("The basics are covered here..."), only the first comment was marked as processed. The second identical comment remains as `_NOTE_` and can be processed separately.

---

## Usage

```bash
# Process daily note
/obsidian-process-comments 01-Daily/2025-01-15.md

# Process permanent note
/obsidian-process-comments 03-Learning/Permanent/Kubernetes-CronJobs.md

# Process work note
/obsidian-process-comments 02-Work/ProjectA/PROJA-1234-Implementation.md
```

---

## Error Handling

- **File not found**: "Could not find note at `$ARGUMENTS`. Please check the path relative to vault root."
- **No comments found**: "No `_NOTE_` comments found in this note. Nothing to process."
- **User cancels**: "Processing cancelled. X comments remain unprocessed."
- **MCP tool errors**: Show error to user and ask how to proceed
