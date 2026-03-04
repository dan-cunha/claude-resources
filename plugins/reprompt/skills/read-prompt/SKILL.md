---
name: read-prompt
description: Load a reprompt to continue where a previous session left off
allowed-tools: Read, Glob, Bash
---

# Reprompt: Load a Living Continuation Prompt

Load a previously saved reprompt to bootstrap a new session with full context from prior work.

**Do NOT ask for confirmation. Do NOT use AskUserQuestion. The skill was invoked intentionally.**

## Step 1: Resolve the Reprompts Directory

```bash
echo "CONFIG_DIR: ${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
```

```bash
# Find the encoded project path
ls "${CLAUDE_CONFIG_DIR:-$HOME/.claude}/projects/"
```

- `REPROMPTS_DIR` = `${CLAUDE_CONFIG_DIR}/projects/{encoded-project-path}/reprompts/`
  - Use the same encoded project path as the `memory/` directory that already exists for this project

## Step 2: List Available Reprompts

```bash
# List reprompt files sorted by modification time (most recent first)
ls -t "${REPROMPTS_DIR}"*.md 2>/dev/null
```

If no files exist, output:
```
No reprompts found for this project.
Use /reprompt:create-prompt to create one.
```
And stop.

## Step 3: Select the File

Parse `$ARGUMENTS`:

| Input | Resolution |
|---|---|
| No arguments | Pick the most recent file by mtime (first in the `ls -t` output) |
| Exact slug (e.g. `mysql-tests`) | Match `mysql-tests.md` exactly |
| Partial/substring (e.g. `mysql`) | Find files containing the substring. If exactly one match, use it. If multiple matches, list them and ask the user to pick. |

If no match is found:
```
No reprompt matching "{ARGUMENTS}" found.

Available reprompts:
  - {slug1} (updated {date})
  - {slug2} (updated {date})

Use /reprompt:read-prompt {slug} to load one.
```
And stop.

## Step 4: Read and Display

Use the **Read** tool to read the selected file.

Display the full content of the file to the user — frontmatter and body.

## Step 5: Set Active State and Output

After displaying the content, output:

```
--- reprompt loaded ---
Active reprompt: {SLUG}

/reprompt:create-prompt will update this file when you're done.
```

**Important:** The slug is now the "active slug" for this session. If the user later invokes `/reprompt:create-prompt` without arguments, it should UPDATE this file rather than creating a new one.

## Step 6: Begin Work

After loading the reprompt, immediately begin working on the **Next Action** described in the reprompt. Do not wait for the user to tell you what to do — the reprompt contains the continuation context.

## Notes

- Only shows reprompts for the current project (scoped by the encoded project path)
- Session history in frontmatter provides breadcrumbs to prior sessions: `claude --resume {id}`
- The active slug persists only within this session — it is not stored on disk
