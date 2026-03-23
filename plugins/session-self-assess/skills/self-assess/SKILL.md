---
name: self-assess
description: Review the current session for errors, inefficiencies, and repeated workarounds, then produce actionable recommendations to improve skills, scripts, CLAUDE.md instructions, and tooling for future sessions.
user-invocable: true
allowed-tools: Read, Glob, Grep, Write, Edit, Bash(ls *), Bash(cat *)
---

# Session Self-Assessment

Audit the current conversation for errors, wasted tokens, and repeated workarounds. Produce concrete action items that reduce future friction.

---

## When to invoke

- User runs `/self-assess`
- User asks to "review the session", "what went wrong", "lessons learned", etc.
- Before a context compaction, if the user requests it

---

## Procedure

### 1. Scan the conversation for friction events

Walk through the full conversation history and identify every instance of:

| Category | What to look for |
|---|---|
| **Tool errors** | Bash exit codes != 0, failed file reads, permission denials, command-not-found |
| **Retry loops** | Same or similar command attempted 2+ times with adjustments (escaping, paths, flags) |
| **Workarounds** | Switching approaches after the first attempt failed (e.g., bash fallback after a skill failed) |
| **Missing knowledge** | Time spent searching for a file/skill/config that should have been known from CLAUDE.md or memory |
| **Encoding/platform issues** | Platform-specific gotchas (path separators, encoding, shell escaping, special characters in filenames) |
| **Token waste** | Long tool outputs that weren't needed, unnecessary exploratory reads, redundant searches |
| **Skill/instruction gaps** | Situations where a CLAUDE.md instruction, memory entry, or skill would have prevented the error |

### 2. For each friction event, produce a diagnosis

For each event, record:

- **What happened**: 1-2 sentence description
- **Root cause**: Why did it fail? (missing config, platform bug, bad escaping, missing instruction, etc.)
- **Tokens wasted**: Rough estimate (low / medium / high) based on how many tool calls were spent on it
- **Recurrence risk**: Will this happen again in future sessions? (one-off / likely / certain)

### 3. Generate action items

For each event with recurrence risk "likely" or "certain", propose a concrete fix:

| Fix type | Examples |
|---|---|
| **Update CLAUDE.md** | Add instruction, clarify existing rule, add to glossary |
| **Update/create a skill** | Fix a broken skill script, add a new skill for a repeated task |
| **Update memory** | Save a feedback or reference memory for future sessions |
| **Fix a script/config** | Patch a helper script, update an environment, fix a PATH issue |
| **Package/tool update** | Update a broken dependency, install a missing tool |
| **No action (document only)** | One-off issue not worth fixing, but worth noting |

Each action item must include:
- **Priority**: P0 (fix now, blocks future work), P1 (fix soon, causes repeated friction), P2 (nice to have)
- **Effort**: trivial (< 1 min), small (< 5 min), medium (< 30 min), large (> 30 min)
- **Specific change**: Exact file to edit, exact instruction to add, exact command to run. Not vague suggestions.

### 4. Check existing instructions for staleness

While reviewing, also flag any CLAUDE.md instructions, skills, or memories that were:
- **Incorrect or outdated** based on what happened in this session
- **Ignored or overridden** because they didn't match reality
- **Missing** — situations where an instruction would have saved time

---

## Output format

```markdown
# Session Self-Assessment
**Date**: [date]
**Session summary**: [1-2 sentences on what was accomplished]

## Friction Events

### 1. [Short title]
- **What happened**: ...
- **Root cause**: ...
- **Tokens wasted**: low / medium / high
- **Recurrence risk**: one-off / likely / certain

[repeat for each event]

## Action Items (ordered by priority)

### P0 — Fix now
[numbered list, or "None"]

### P1 — Fix soon
[numbered list, or "None"]

### P2 — Nice to have
[numbered list, or "None"]

## Stale/Missing Instructions
[numbered list of CLAUDE.md entries, memories, or skills that need updating, or "None"]

## Session Stats
- Total friction events: N
- Estimated token waste: low / medium / high
- Top recurring pattern: [the most common category of error]
```

---

## Rules

- **Be specific and actionable.** "Fix the trash skill" is not enough. "Edit `trash.sh` line 19 to use full PowerShell path instead of bare `powershell.exe`" is.
- **Don't flag things that worked.** This is an error/friction audit, not a session summary.
- **Don't propose large refactors** unless the friction is severe and recurring. Prefer minimal targeted fixes.
- **Ask before implementing** any P0 or P1 fixes. Present the assessment first, then offer to execute.
- **If the session was clean**, say so briefly and skip the detailed format.
