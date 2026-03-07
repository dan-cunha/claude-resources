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

## Step 2: Find Available Reprompts

Use the **Glob** tool (NOT bash ls) to find reprompt files:

```
Glob: pattern = "*.md", path = REPROMPTS_DIR
```

If no files are found, output:
```
No reprompts found for this project.
Use /reprompt:create-prompt to create one.
```
And stop.

## Step 3: Select the File

Parse `$ARGUMENTS`:

| Input | Resolution |
|---|---|
| No arguments | Pick the most recently modified file (first in the Glob results, which are sorted by mtime) |
| Exact slug (e.g. `mysql-tests`) | Directly read `${REPROMPTS_DIR}/mysql-tests.md` — do NOT rely on the Glob listing to find it |
| Partial/substring (e.g. `mysql`) | Find files from Glob results containing the substring. If exactly one match, use it. If multiple matches, list them and ask the user to pick. |

**IMPORTANT:** When an exact slug is provided, go straight to reading `${REPROMPTS_DIR}/{slug}.md` with the Read tool. Do NOT require it to appear in a listing first.

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

## CRITICAL: User Arguments Override Everything

**Any arguments the user passes to this skill ALWAYS take precedence over instructions inside the reprompt file.**

The reprompt's "Next Action" and other directives are *default context* — they describe where the previous session left off. But if the user provides their own instructions (e.g. `/reprompt:read-prompt mysql-tests fix the connection pooling instead`), their intent supersedes whatever the reprompt says to do next. The user is here NOW; the reprompt is from BEFORE.

**Never ignore, deprioritize, or argue against user-supplied arguments in favor of reprompt content.**

## Notes

- Only shows reprompts for the current project (scoped by the encoded project path)
- Session history in frontmatter provides breadcrumbs to prior sessions: `claude --resume {id}`
- The active slug persists only within this session — it is not stored on disk
