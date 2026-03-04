---
name: create-prompt
description: Create or update a reprompt — a living continuation prompt that persists across sessions
allowed-tools: Write, Read, Bash, Glob, Grep, TaskList, Agent
---

# Reprompt: Create / Update a Living Continuation Prompt

Generate or update a structured continuation prompt capturing everything needed to resume in a new session. Saved as a project-scoped living document that evolves across sessions.

**Do NOT ask for confirmation. Do NOT use AskUserQuestion. The skill was invoked intentionally.**

## Step 0: Parse Arguments

`$ARGUMENTS` may contain:
- A **slug** (e.g. `mysql-tests`) — used as filename
- A **`--new` flag** (e.g. `--new api-refactor`) — forces CREATE mode even if an active slug exists
- **Emphasis text** (e.g. `focus on the API layer`) — steer the prompt emphasis; derive slug from it
- **Nothing** — use the active slug if one was set by `read-prompt`, otherwise derive slug from the conversation goal

**Slug rules:** lowercase, hyphenated, no special characters, 1-3 words (e.g. `mysql-tests`, `api-refactor`, `dashboard-fix`).

## Step 1: Resolve Paths

```bash
echo "CONFIG_DIR: ${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
```

- `REPROMPTS_DIR` = `${CLAUDE_CONFIG_DIR}/projects/{encoded-project-path}/reprompts/`
  - The encoded project path uses the same encoding as the `memory/` directory that already exists alongside it
  - Look at the existing `projects/` directory to find the correct encoded path for the current working directory
- `REPROMPT_FILE` = `${REPROMPTS_DIR}/${SLUG}.md`

```bash
# Find the encoded project path by looking at existing project directories
ls "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/projects/"
```

## Step 2: Determine Mode (CREATE vs UPDATE)

**UPDATE mode** if ALL of these are true:
- The slug file already exists at `REPROMPT_FILE`
- The file was loaded this session via `read-prompt` (i.e., the slug matches what was previously loaded)
- The `--new` flag was NOT provided

**CREATE mode** otherwise.

If UPDATE mode, read the existing file:
```bash
# Only in UPDATE mode
cat "${REPROMPT_FILE}"
```

## Step 3: Gather Fresh State (run in parallel)

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

## Step 4: Get Session ID

```bash
# Try env var first
echo "${CLAUDE_SESSION_ID:-}"
```

If empty, fall back to extracting from conversation state files:
```bash
# Check for the most recent session file in the project config
ls -t "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/projects/"*"/conversations/" 2>/dev/null | head -1
```

If no session ID can be determined, use `unknown` as the ID.

## Step 5: Analyze Conversation

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

## Step 6: Generate the File Content

### Frontmatter

**CREATE mode:**
```yaml
---
slug: {SLUG}
created: {TODAY YYYY-MM-DD}
updated: {TODAY YYYY-MM-DD}
sessions: 1
history:
  - id: {SESSION_ID}
    date: {TODAY YYYY-MM-DD}
    summary: {One-line summary of what this session accomplished}
---
```

**UPDATE mode:**
- Preserve the existing `created:` date
- Update `updated:` to today
- Increment `sessions:` count by 1
- Append a new entry to `history:` with the current session ID, date, and summary
- Preserve all existing history entries exactly as they are

### Body

Use this template. **Omit empty sections.** Let content dictate length — be comprehensive but not verbose.

**CREATE mode** — generate all sections fresh from conversation context.

**UPDATE mode** — read the existing file and:
- Preserve decisions that are still relevant
- Update **Current State**, **Done**, **Next Action**, and **Remaining** to reflect this session's progress
- Move completed items from **Remaining** to **Done**
- Add new decisions, files, and references discovered this session
- Remove stale information that no longer applies

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
- (UPDATE mode) Are existing decisions and history preserved correctly?

## Step 7: Save the File

**Use ONLY the Write tool.** No Bash commands. The `bash -c` approach triggers Claude Code's security heuristic.

Write the complete file (frontmatter + body) to `REPROMPT_FILE`.

The Write tool creates parent directories automatically — no `mkdir` needed.

## Step 8: Output

Compute `REPROMPT_FILE_DISPLAY` by replacing `$HOME` with `~` in the path.

**CREATE mode:**
```
--- reprompt saved ---
Slug: {SLUG}
Status: Created (new)
File: {REPROMPT_FILE_DISPLAY}

Next session:
  /reprompt:read-prompt {SLUG}
  /reprompt:read-prompt           <- loads latest
```

**UPDATE mode:**
```
--- reprompt saved ---
Slug: {SLUG}
Status: Updated (session {N})
File: {REPROMPT_FILE_DISPLAY}

Next session:
  /reprompt:read-prompt {SLUG}
  /reprompt:read-prompt           <- loads latest
```

## Arguments

If `$ARGUMENTS` contains emphasis text (not just a slug or `--new`), use it two ways:
1. **Emphasis** — highlight those aspects more prominently in the prompt
2. **File slug** — derive the slug from the arguments if no explicit slug is provided

## Notes

- Reads the entire conversation — works best at end of session or when context is long
- The prompt must be self-contained — a new session should understand full context without reading this conversation
- Prefer referencing plan files over duplicating their content
- If no plan file exists but significant design decisions were made, note that explicitly
- Session history in frontmatter provides breadcrumbs — `claude --resume {id}` to revisit any session
