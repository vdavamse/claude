---
name: prime-context
description: Load essential project context (CLAUDE.md, README.md) from a specific directory
argument-hint: <project-directory>
---

You are loading essential project context into the main agent's session for a specific project directory. Execute this sequence precisely to ensure comprehensive project understanding.

## Execution Sequence

### 1. Determine Target Directory

- If argument provided: Use `{{ARGS}}` as the target directory
- If no argument: Use current directory (`.`)

### 2. Explore Project Structure

Navigate to the target directory and visualize its structure:

```bash
cd {{ARGS}} && eza . --tree --git-ignore
```

If `eza` is not available, fallback to:
```bash
cd {{ARGS}} && ls -laR
```

Analyze the output to identify:
- Key directories and their purpose
- Technology stack (from file extensions and directory names)
- Project organization patterns

### 3. Read Core Documentation

Read the following files from the target directory (skip any that don't exist):

1. **CLAUDE.md**:
   - Read `{{ARGS}}/CLAUDE.md`
   - This contains project-specific instructions and conventions

2. **README.md**:
   - Read `{{ARGS}}/README.md`
   - Contains project overview, setup instructions, and usage

### 4. Provide Context Summary

After loading all files, provide a brief summary:

```
✅ Context Loading Complete for: {{ARGS}}

Project Structure: [Brief description of key directories]

Files Loaded:
- ✓ {{ARGS}}/CLAUDE.md: [one-line summary]
- ✓ {{ARGS}}/README.md: [one-line summary]
- ✗ [any missing files]

Key Project Details:
- Technology: [identified stack]
- Purpose: [project purpose]
- Special Instructions: [any critical CLAUDE.md instructions]
```

## Usage Examples

```bash
# Load context for auto-discovery service
/prime-context my-service

# Load context for backend service
/prime-context my-backend-service

# Load context for current directory
/prime-context .
```

## Important Guidelines

- ✅ Only load CLAUDE.md from the specified directory
- ✅ Read files in the order specified
- ✅ Continue even if some files are missing
- ✅ Use Read tool to load file contents into context
- ❌ Don't load CLAUDE.md from other directories
- ❌ Don't skip files that exist in the target directory
- ❌ Don't read files beyond the specified list

## Error Handling

- If target directory doesn't exist, report error and stop
- If `eza` command fails, use `ls -laR` instead
- If CLAUDE.md or README.md don't exist, note it and continue
- Never halt the sequence due to missing files (except directory itself)

Your goal is to efficiently load the essential project context for the SPECIFIC project directory so the main agent has targeted access to that project's documentation and history throughout the session.
