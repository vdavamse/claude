# claude

My Claude Code toolkit: skills, sub-agents and slash commands, kept here so they can be installed on any machine.

## Layout

| Directory | Installs to |
|---|---|
| `agents/` | `~/.claude/agents/` |
| `commands/` | `~/.claude/commands/` |
| `skills/` | `~/.claude/skills/` |

## Install

Symlink each file or directory into `~/.claude`, so a `git pull` updates the toolkit:

```bash
for d in agents commands; do
  mkdir -p ~/.claude/$d
  for f in "$PWD"/$d/*.md; do ln -sf "$f" ~/.claude/$d/; done
done
mkdir -p ~/.claude/skills
for s in "$PWD"/skills/*/; do ln -sfn "${s%/}" ~/.claude/skills/; done
```

## Environment variables

| Variable | Used by | Purpose |
|---|---|---|
| `OBSIDIAN_VAULT` | `obsidian-*` commands, `obsidian-vault-manager` agent | Path to the Obsidian vault |
| `QWEN_API_KEY` | `ask-qwen` skill | API key for the Qwen endpoint (also read from `~/.zshrc`) |
| `ASK_QWEN_MODEL` | `ask-qwen` skill | Default opencode model id, `<provider>/qwen3.8-max` (also read from `~/.zshrc`) |
| `ASK_QWEN_FLASH_MODEL` | `ask-qwen` skill | Model id for bulk mechanical edits, `<provider>/qwen3.8-flash` |
| `FORGE` | `forge` skill | Optional platform override: `FORGE=github` or `FORGE=gitlab`; by default the skill detects the platform from the git remote |

The `forge` skill backs the issue and PR/MR agents and commands (`fix-issue`, `change-iteration`, `review-change`, `review-inline`) and wraps every platform CLI call: it requires `gh` for GitHub repos and `glab` for GitLab repos.

## Not in this repo

Never commit `~/.claude` wholesale: it holds credentials, session history and file history. `.gitignore` blocks the usual runtime state. Personal skills (job search) stay local only.
