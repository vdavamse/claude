Create a new note in the Obsidian vault located at `$OBSIDIAN_VAULT` using the appropriate template.

`$OBSIDIAN_VAULT` is an environment variable holding the vault path: resolve it with `echo $OBSIDIAN_VAULT` and ask the user if it is unset.

## Your Task

Help the user create a new note from a template. Ask clarifying questions to gather necessary information, then create the note in the correct location.

### Note Types and Locations:

**1. Source Note** (`/03-Learning/Sources/`)
- Template: `$OBSIDIAN_VAULT/Templates/Source-Note.md`
- Ask: Title, source type (book/article/video/course/podcast/paper/blog), author, URL, domain (AI/Kubernetes/DevOps/MLOps/Economics)
- Filename: Use title with dashes (e.g., `Kubernetes-Patterns-Book.md`)

**2. Permanent Note** (`/03-Learning/Permanent/`)
- Template: `$OBSIDIAN_VAULT/Templates/Permanent-Note.md`
- Ask: Title (should describe the concept), related domain tags
- Filename: Descriptive title (e.g., `Kubernetes-uses-etcd-for-state-storage.md`)
- Reminder: One idea per note (atomic)

**3. Work Note** (`/02-Work/ProjectA/` or `/02-Work/ProjectB/`)
- Template: `$OBSIDIAN_VAULT/Templates/Work-Note.md`
- Ask: Project (ProjectA or ProjectB), JIRA ticket ID, brief title
- Filename: JIRA-ID-brief-description.md (e.g., `PROJA-456-authentication-bug.md`)

**4. Meeting Note** (`/02-Work/Meetings/`)
- Template: `$OBSIDIAN_VAULT/Templates/Meeting-Note.md`
- Ask: Meeting topic, project (ProjectA/ProjectB/General), date if not today
- Filename: YYYY-MM-DD-meeting-topic.md (e.g., `2025-11-09-sprint-planning.md`)

**5. MOC** (`/04-MOCs/`)
- Template: `$OBSIDIAN_VAULT/Templates/MOC.md`
- Ask: Domain/topic name
- Filename: Topic-MOC.md (e.g., `Distributed-Systems-MOC.md`)
- Note: Only create if it's a significant new domain, not already covered by existing MOCs

### Process:

1. **Understand intent** - What type of note does the user want to create?
2. **Gather info** - Ask necessary questions (title, metadata, etc.)
3. **Read template** - Get the appropriate template
4. **Fill template** - Replace {{date}}, {{title}}, and fill in provided metadata
5. **Create note** - Use `obsidian_put_content` to create the note
6. **Confirm** - Tell user the note was created and its location

### Important Notes:

- Use Obsidian MCP tools only
- Follow the folder structure strictly
- Use descriptive filenames (lowercase with dashes)
- Fill in as much metadata as user provides
- If creating a permanent note, remind user to update relevant MOC
- If creating a work note, remind user to link from daily note when working on it

### Examples:

- "Create a source note for the Kubernetes book I'm reading" → Ask for book details, create in /03-Learning/Sources/
- "New permanent note about etcd consensus" → Create atomic note in /03-Learning/Permanent/
- "Create work note for PROJA-456" → Create in /02-Work/ProjectA/
- "Meeting note for today's standup" → Create in /02-Work/Meetings/
