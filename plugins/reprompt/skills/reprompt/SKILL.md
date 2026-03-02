---
name: reprompt
description: Use when ending a session or context is getting long - generates a structured continuation prompt preserving decisions, state, and references for the next session
---

# Reprompt - Session Continuation Prompt Generator

Generate a structured continuation prompt capturing everything needed to resume in a new session. Save to a descriptively named file. No user confirmation needed.

## Step 1: Gather Fresh State (run in parallel)

```bash
git branch --show-current
```

```bash
git status --short
```

```bash
git log --oneline -5
```

```bash
git diff --stat
```

```bash
git stash list
```

```bash
git worktree list
```

Also:
- Call **TaskList** tool to capture pending/in-progress work from this session
- Use **Glob** to find active plan files: `{CLAUDE_CONFIG_DIR}/plans/*.md`
- Check for auto-memory files modified today: `{CLAUDE_CONFIG_DIR}/projects/*/memory/*.md`
- Note the **working directory** (from system context)
- Check if `CLAUDE_CONFIG_DIR` is set to a non-default value (anything other than `~/.claude`). If so, note the config directory — the user may be running multiple Claude instances.

## Step 2: Analyze Conversation

Scan the full conversation history and extract:

1. **Goal** — The original request, refined through discussion
2. **Decisions** — Explicit choices made, with WHY; rejected alternatives matter as much as chosen ones
3. **Files touched** — Files that were Read, Edited, or Written
4. **Plan files** — Any plans created or referenced
5. **Design docs** — Any design documents created or referenced
6. **Progress** — What's done vs what remains
7. **Skills used** — Any skills invoked and why they're relevant going forward
8. **Blockers** — Failed attempts, unresolved issues, open questions
9. **Architecture notes** — Key patterns, conventions, or gotchas established

Focus on WHY — the code shows WHAT.

## Step 3: Generate Continuation Prompt

Use this template. **Omit empty sections.** Let content dictate length — be comprehensive but not verbose.

```markdown
## Goal
[1-2 sentence summary of the overarching task]

## Current State
- **Directory:** `path/to/working/dir`
- **Branch:** `branch-name` | **Status:** [clean / N files modified / stashed]
- **Worktree:** `path` (if working in a worktree)
- **Config:** `path/to/config/dir` (only if non-default)
- **Phase:** [where we are — e.g. "implementing step 3 of 5", "design complete, starting implementation"]
- **Last action:** [what was the last meaningful thing done]

## Active Tasks
[From TaskList: pending/in-progress only. Format: #id subject. Omit if empty.]

## Key Decisions
- [Decision + rationale — focus on WHY, the WHAT is in the files]
- [Rejected alternatives with WHY if important]

## Important Files
- `path/to/file` — [1-line why it matters]

## References
- `path/to/plan.md` — [what it covers]
- `path/to/design.md` — [what it covers]
- **Skills:** `/name` — [why relevant going forward]
- **Memory:** [anything written to MEMORY.md or auto-memory this session]

## Done
- [Completed item — one line each, keep brief]

## Next Action
[The single most important thing to do on resume. Be specific — include file paths, function names, or diagnostic data needed to act immediately.]

## Remaining
- [ ] [Additional steps after the next action]

## Context
[Environment quirks, debugging discoveries, error patterns, workarounds — things lost between sessions. Omit if nothing notable.]

## Open Questions / Blockers
- [Unresolved issue]
```

### Quality check

Before writing the file, verify:
- Can someone resume with **only** this prompt? No conversation context needed?
- Is **WHY** captured for key decisions?
- Does **Next Action** have enough detail to start immediately?
- Are **Done** items brief (one line each)?
- Are empty sections omitted?

## Step 4: Save the File

**Use ONLY the Write tool.** No Bash commands at all. The `bash -c` approach triggers Claude Code's "quoted characters in flag names" security heuristic and prompts the user.

### 4a. Determine the file path

- `REPROMPT_DIR` = `${CLAUDE_CONFIG_DIR:-$HOME/.claude}/tmp` (use the actual resolved value)
- `TIMESTAMP` = current date/time as `YYYYMMDD-HHMMSS`
- `SLUG` = a 1-2 word descriptor derived from `$ARGUMENTS` if provided, otherwise from the Goal (e.g., "interval-styling", "mobile-layout", "migration-backup"). Lowercase, hyphenated, no special characters.
- `REPROMPT_FILE` = `$REPROMPT_DIR/reprompt-${TIMESTAMP}-${SLUG}.md`
- `REPROMPT_FILE_DISPLAY` = the same path but with `$HOME` replaced by `~` (e.g., `~/.config/claude-code/sweetwater/tmp/reprompt-...`). Use this for all user-facing output.

### 4b. Write the file

Use the **Write** tool to write the generated prompt content to `REPROMPT_FILE`. The Write tool creates parent directories automatically — no `mkdir` needed.

Do NOT use Bash, cat, heredoc, or any shell command for this step.
Do NOT ask for confirmation. Do NOT use AskUserQuestion. The skill was invoked intentionally.

## Step 5: Output

Output the bootstrap command as **copiable text** so the user can select and copy it:

```
--- reprompt saved ---
File: [REPROMPT_FILE_DISPLAY path]

Bootstrap (copy this):
@[REPROMPT_FILE_DISPLAY path] — read this for full context, then continue with the Next Action.

/clear, then paste to continue.
```

## Arguments

If `$ARGUMENTS` is provided (e.g. `/reprompt focus on the API refactor`), use it two ways:
1. **Emphasis** — highlight those aspects more prominently in the prompt
2. **File slug** — derive the filename descriptor from the arguments

## Notes

- Reads the entire conversation — works best at end of session or when context is long
- The prompt must be self-contained — a new session should understand full context without reading this conversation
- Prefer referencing plan files over duplicating their content
- If no plan file exists but significant design decisions were made, note that explicitly
