Manage today's daily note in the Obsidian vault located at `$OBSIDIAN_VAULT`.

`$OBSIDIAN_VAULT` is an environment variable holding the vault path: resolve it with `echo $OBSIDIAN_VAULT` and ask the user if it is unset.

## Your Task

Help the user interact with their daily note. The daily note is located at `$OBSIDIAN_VAULT/01-Daily/YYYY-MM-DD.md` (use today's date).

### Common Operations:

**1. Show Status** - If user provides no additional context:
- Get today's daily note using `obsidian_get_periodic_note("daily")`
- Show summary: number of todos by project (ProjectA/ProjectB/Personal), completed vs pending
- Show work log entries
- Show any notes in the learning section

**2. Add Todo** - If user wants to add a task:
- Parse which project (ProjectA, ProjectB, or Personal)
- Use `obsidian_patch_content` to append under the correct heading
- Add as markdown checkbox: `- [ ] task description`

**3. Mark Todo Done/Blocked** - If user wants to update a task:
- Find the task in the appropriate section
- Mark done: Change `- [ ]` to `- [x]`
- Mark blocked: Change `- [ ]` to `- [ ] ⛔` and optionally add blocker note

**4. Add Work Log Entry** - If user wants to log work:
- Get current time
- Use `obsidian_patch_content` to append to "📝 Work Log" section
- Format: `- **HH:MM** - description`

**5. Add Learning Note** - If user learned something today:
- Append to "📚 Learning Today" section
- Can include links to permanent notes or source notes

### Important Notes:

- If today's daily note doesn't exist, create it using the Daily-Note template from `$OBSIDIAN_VAULT/Templates/Daily-Note.md`
- Use Obsidian MCP tools only (obsidian_get_periodic_note, obsidian_patch_content, obsidian_put_content)
- Maintain the daily note structure (don't break the headings)
- Be conversational - understand natural language requests like "mark the PR review task as done"

### Examples:

- "Show me today's status" → Display todo counts and work log
- "Add todo: Review PROJA-456 PR under ProjectA" → Append to ProjectA section
- "Mark the ProjectB deployment task as done" → Find and check off task
- "Log: Finished implementing authentication feature" → Add work log entry
- "I learned about Kubernetes CronJobs today" → Add to learning section
