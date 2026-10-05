Update a Map of Content (MOC) in the Obsidian vault located at `$OBSIDIAN_VAULT/04-MOCs/`.

`$OBSIDIAN_VAULT` is an environment variable holding the vault path: resolve it with `echo $OBSIDIAN_VAULT` and ask the user if it is unset.

## Your Task

Help the user update a domain MOC by adding recent permanent notes, organizing concepts, or updating learning goals.

### Available MOCs:

- **AI-MOC.md** - Artificial Intelligence, ML, deep learning, NLP, LLMs
- **Kubernetes-MOC.md** - Kubernetes, container orchestration, K8s resources
- **DevOps-MOC.md** - CI/CD, IaC, containers, observability, SRE
- **MLOps-MOC.md** - ML pipelines, model deployment, monitoring
- **Economics-MOC.md** - Economic theories, macro/micro economics, behavioral economics

### Common Operations:

**1. Add Recent Permanent Notes**
- Search for recent permanent notes in `/03-Learning/Permanent/` related to the domain
- Ask user which notes to add (or make suggestions)
- Organize under appropriate subtopic headings
- Use `obsidian_patch_content` to add links under correct heading

**2. Organize Subtopics**
- Review current structure
- Suggest better organization if notes are getting numerous
- Add new subtopic headings as needed

**3. Update Learning Goals**
- Review current learning goals
- Mark completed goals with [x]
- Add new goals based on recent activity
- Remove obsolete goals

**4. Update Questions & Gaps**
- Review questions section
- Remove questions answered by recent permanent notes
- Add new questions that emerged from learning

**5. Update Related MOCs**
- Add links to related MOCs if connections emerge
- Example: MLOps connects to both AI and DevOps

### Process:

1. **Identify domain** - Which MOC to update? (ai, kubernetes, devops, mlops, economics)
2. **Read current MOC** - Get current state using `obsidian_get_file_contents`
3. **Determine operation** - What needs updating? Recent notes? Goals? Structure?
4. **Search for relevant content** - Find recent permanent notes if needed using `obsidian_complex_search`
5. **Update MOC** - Use `obsidian_patch_content` to update specific sections
6. **Update modified date** - Update the frontmatter modified field to today

### Important Notes:

- Use Obsidian MCP tools only
- Maintain MOC structure (don't break headings)
- Keep subtopics organized logically
- Link to permanent notes using `[[note-title]]` syntax
- If MOC is getting too large (>50 notes), suggest creating sub-MOCs
- Always update the "Last updated" field at the bottom

### Examples:

- "Update Kubernetes-MOC with my recent CronJob note" → Find note, add to appropriate subtopic
- "I've completed my learning goal about transformers in AI-MOC" → Mark goal as done
- "Add new subtopic for 'Service Mesh' in Kubernetes-MOC" → Add new heading and structure
- "What questions do I have in DevOps-MOC?" → Show questions section
- "Update Economics-MOC with recent behavioral economics notes" → Search and add relevant notes

### Automation Tip:

If user says "update" without specifying content, be proactive:
1. Search for recent permanent notes in the domain (last 7-14 days)
2. Show user what you found
3. Suggest where to add them in the MOC structure
4. Add with user confirmation
