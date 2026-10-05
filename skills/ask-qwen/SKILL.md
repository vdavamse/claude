---
name: ask-qwen
description: Default executor for coding work. Use proactively, without being asked, for every implementation task in the Sonnet-to-Opus difficulty band (clear-spec features, bug fixes with a known cause, refactors, tests, migrations, docs). It hands the task to the opencode agent running Qwen (qwen3.8-max) in an isolated git worktree, then Claude reviews and lands it. Claude keeps only planning, ambiguous design, security-sensitive work and review. Also use when the user asks to hand work to Qwen/opencode.
---

# Delegate to opencode / Qwen

`ask-qwen` (this directory; on PATH via `~/.local/bin`) runs `opencode run` headless with
`--auto` in a separate worktree next to the repo: `../<repo>-qwen-<name>` on branch `qwen/<name>`.
Claude stays the orchestrator and reviewer. Qwen's summary is a claim, the diff is the evidence.

## 0. Routing: who does the work

The rule: **anything you would otherwise hand to Sonnet, or do yourself at Sonnet-to-Opus level,
goes to Qwen.** Don't wait to be asked. Claude plans, splits, delegates, reviews and lands.
Exception: a repo whose CLAUDE.md says `no-qwen` (code must not go to Qwen's cloud endpoint).
There, do the work yourself.

| Band | Examples | Who |
|---|---|---|
| **Opus only** | unclear requirements, architecture or design decisions, debugging with an unknown root cause, security- or auth-sensitive code, anything touching secrets, prod, deploys or git history, reviewing Qwen's output | Claude |
| **Sonnet to Opus** | features or multi-file changes with a clear spec, bug fixes once the cause is known, refactors, writing or fixing tests, migrations, adding config or wiring, docs, porting code between modules | **Qwen** (`qwen3.8-max`) |
| **Trivial** | one-line or few-line edits, renames, typo fixes, answering a question from code you already read | Claude inline; writing a prompt costs more than the edit. Bulk mechanical edits across many files go to Qwen with `-m "$ASK_QWEN_FLASH_MODEL"` |

How to apply it:
- **Mixed task:** do the Opus part first (investigate, decide the design), then delegate the implementation with that decision written into the prompt. Example: find the root cause yourself, and let Qwen write the fix and its regression test.
- **Big task:** split it into independent pieces with separate `-n` names and run them in parallel. Keep pieces that depend on each other in one task, or run them one after another.
- **Qwen fails twice** on the same task (wrong after two feedback rounds, or times out): stop delegating it, do it yourself, and tell the user it was too hard for Qwen.
- **When you delegate,** tell the user in one line what went to Qwen and why. They can say "do it yourself" to override for that task.
- **Exploring, reading or answering questions** stays with Claude. Delegate work that produces changes.
  A measured Claude+worker benchmark ([delegate-coder](https://github.com/pctan88/delegate-coder))
  found heavy implementation/refactors about 37% cheaper delegated, but bulk reading 20% *more*
  expensive and 3x slower. Small tasks don't pay for the brief either.
- **No open-ended research.** Tasks where the answer is unknown (kernel or API feasibility
  probes, "find out whether X works") are Opus-band even when phrased as "build a PoC". Two such
  runs (landlock/seccomp) timed out at 30 min with qwen3.8-max after only 12–19 steps.
- **Only delegate what you can check.** If there's no objective check (tests, type check,
  build, a script), write one first or keep the task.

## 1. Write the brief

Qwen sees none of this conversation and takes instructions literally, so write the brief like a
contract for a capable junior. Write it to a file in your scratchpad and pass it with `-F`; long
briefs with quotes and backticks break when passed on the command line.

```markdown
## Goal
<one or two sentences: the behavior wanted, and why if it affects choices>

## Files
- <absolute path in the worktree, ../<repo>-qwen-<name>/...> (edit)
- <absolute path> (read for context: the pattern to copy, the interface to match)

## Decisions already made
<design choices, root cause, names, signatures, so Qwen doesn't invent its own>

## Forbidden
Don't change <public API / these files / existing tests>. Don't add dependencies. Don't commit.

## Verify
<exact command(s) that must pass>

## Report
One line per changed file, plus anything you were unsure about.
```

Rules:
- Use absolute paths inside the **worktree** (`../<repo>-qwen-<name>/...`), not the user's checkout. File tools outside the worktree are blocked.
- Bad: "clean up the imports". Good: "Remove unused imports from `<path>/api.py`, sort them by isort rules, and run `uv run ruff check <path>/api.py`."
- Name what's out of scope. Qwen tends to "improve" nearby code.
- Tell it where scratch files go (e.g. `<worktree>/.scratch/`, gitignored or deleted after). `/tmp` is blocked, and Qwen burns steps retrying there.
- Make sure the verify tools are installed before delegating. Otherwise Qwen installs them itself (it once ran `pip install --break-system-packages`).
- Use one task per name, with a short kebab-case `-n` such as `fix-median` or `add-retry`.

## 2. Run it

```bash
cd <repo> && ask-qwen -n <name> -F <scratchpad>/brief-<name>.md -t "<verify command>"
```

- Always pass `-t`. It runs the check in the worktree after Qwen finishes and prints
  `VERIFY PASS` or `VERIFY FAIL`. Diffs and exit codes can't hallucinate; summaries can.
- Use `run_in_background: true` and wait for the notification. Don't poll.
- Exit codes: `0` done and verified, `3` verify failed, `124` timed out (30 min default;
  `ASK_QWEN_TIMEOUT`), `125` opencode froze at startup twice (see limits), anything else is an
  opencode or provider error (see `<log>.err`).
- The default model is `$ASK_QWEN_MODEL` (an opencode `<provider>/<model>` id, e.g. the provider
  entry for `qwen3.8-max` in `~/.config/opencode/opencode.json`); like the API key, it is read
  from `~/.zshrc` when not exported. `-m "$ASK_QWEN_FLASH_MODEL"` (the `qwen3.8-flash` id) for bulk mechanical edits; `-a <agent>` for an
  opencode agent from `~/.config/opencode/agents/`.
- Several names can run in parallel; each gets its own worktree.

The output shows the session ID, time and tokens, each tool call (`ok` / `ERR` plus the reason),
Qwen's answer, `git status` / diff stat and the verify result. Logs are in
`~/.local/state/ask-qwen/`, and `runs.jsonl` there records every run (model, time, tokens,
result). Use it to judge whether delegating pays off.

## 3. Review (always)

```bash
git -C ../<repo>-qwen-<name> diff HEAD        # Qwen's uncommitted changes
cd ../<repo>-qwen-<name> && <verify command>   # rerun the checks yourself
```

Read the whole diff, even when VERIFY PASS. Watch for edits outside the scope, deleted or
weakened tests, special-cased results that only make the check pass, and new dependencies. `ERR` lines show what it tried and was refused. If the answer contradicts
the diff, believe the diff.

To ask for changes, continue the same session (it keeps its context):
```bash
ask-qwen -n <name> -s <session-id> "<specific feedback>"
```

## 4. Land or discard

The changes are uncommitted in the worktree. Ask the user before merging into their branch.
- Land: commit in the worktree (`git -C <wt> add -A && git -C <wt> commit -m ...`), then `git merge qwen/<name>` or cherry-pick from the user's branch.
- Discard or clean up: `git worktree remove --force ../<repo>-qwen-<name> && git branch -D qwen/<name>`.

## Guardrails and limits

- Blocked through `OPENCODE_CONFIG_CONTENT`: `git push/remote/reset --hard/checkout/switch/worktree`, `rm -rf`, `sudo`, `curl`, `wget`, reading `.env` files, file tools outside the worktree, and asking questions (headless runs can't answer them). Each part of a chained command is checked.
- **These blocks are not a security boundary.** `bash -c`, `python -c`, `find -delete` and the like get around them, and Qwen's shell still has the network and the user's environment. Don't delegate tasks involving secrets or production access. Real isolation needs bwrap, which isn't set up yet.
- `OPENCODE_PERMISSION` is ignored by opencode v2; use `OPENCODE_CONFIG_CONTENT`.
- Qwen can see the skills in `~/.claude/skills`; `ask-qwen` itself is blocked for it, so it can't delegate recursively.
- Runs use a private `--standalone` server, because the shared background service ignores this process's environment (API key and guardrails). That private server sometimes freezes at startup (once for exactly 5 minutes, once indefinitely). The script kills it after 90s of no output and retries once (`ASK_QWEN_STARTUP_TIMEOUT`).
- The script loads `QWEN_API_KEY` from `~/.zshrc` when the variable isn't set, and puts `~/.opencode/bin` on PATH itself (non-interactive shells otherwise exit 127).
- Models run at `"effort": "high"` (`~/.config/opencode/opencode.json`). `xhigh` made max steps take up to 10 min of hidden thinking. Pass `--thinking` to opencode if you need to see it.
- `__pycache__/` and similar files may show up untracked in the worktree; don't commit them.
