---
name: auto-save-and-push
description: >
  Saves conversation context, organizes project files, and pushes to a private Git repo —
  designed to prevent context rot and keep sessions fresh. Use this skill whenever the user
  wants to back up their work, save conversation history, auto-commit and push changes,
  organize messy files, clean up project structure, or set up a recurring save-and-push loop.
  Trigger on: "save my progress", "push my work", "backup to git", "save context",
  "checkpoint", "organize my files", "clean up project", "context rot", "fresh session",
  or any mention of periodic auto-saves. Works great with /loop for recurring 30-minute intervals.
---

# Auto Save, Organize, and Push

This skill prevents context rot — the gradual degradation of AI response quality as conversations grow longer — by saving structured context snapshots, organizing project files, and pushing everything to a private Git remote. Research shows AI performance degrades significantly once context usage exceeds 50%, and compaction at 95% often loses critical details. This skill proactively captures context *before* that happens.

Designed to run on-demand or on a recurring schedule (e.g., every 30 minutes via `/loop 30m /auto-save-and-push`).

## What it does

Each invocation performs four things:

1. **Validates the project and folder structure** before touching anything
2. **Organizes project files** — flags misplaced files and enforces structure
3. **Saves conversation context** as a structured handoff document optimized for fresh sessions
4. **Smart-commits and pushes** all changes to the Git remote

---

## Step 0: Pre-Flight Validation

Every single run starts by inspecting the project directory. This catches drift, corruption, and disorganization before they compound. Run through each check in order and stop at the first blocker.

### Project-level checks

1. **Is this a Git repo?** Run `git rev-parse --is-inside-work-tree`. If not, stop and ask the user — don't initialize on their behalf.
2. **Is the working tree clean enough to proceed?** Run `git status`. If there are merge conflicts (unmerged paths), stop and tell the user to resolve them first. Other uncommitted changes are fine.
3. **Does a remote exist?** Run `git remote -v`. If no remote, warn the user but still proceed with context save and file organization (just skip the push).
4. **Does a `.gitignore` exist and is it properly configured?** Check for a `.gitignore` in the project root.
   - If missing entirely: create one with sensible defaults for the detected project type.
   - If it exists: verify it includes common entries that should never be committed. Flag anything missing.
   - Essential entries to check for (add if missing, based on project type):
     ```
     # OS files
     .DS_Store
     Thumbs.db

     # IDE / editor
     .vscode/
     .idea/
     *.swp
     *.swo
     *~

     # Environment & secrets
     .env
     .env.*
     !.env.example
     *.pem
     *.key

     # Dependencies
     node_modules/
     __pycache__/
     *.pyc
     .venv/
     venv/
     env/

     # Build output
     dist/
     build/
     *.egg-info/

     # Logs
     *.log
     npm-debug.log*
     ```
   - For Python projects, also ensure: `.pytest_cache/`, `.mypy_cache/`, `*.egg-info/`
   - For JavaScript/TypeScript projects, also ensure: `node_modules/`, `.next/`, `.nuxt/`, `coverage/`
   - **Never remove entries** from an existing `.gitignore` — only add missing ones. Warn the user about what was added.
5. **Context health check.** Run `/context` or check context window usage if available. If context usage is above 60%, flag this in the summary — it's a signal that the user should consider starting a fresh session after this save.

### `context-logs/` folder checks

5. **Does `context-logs/` exist?** If not, create it.
6. **Are there non-markdown files?** Flag any files that aren't `.md` — they don't belong here. Warn the user, don't delete.
7. **Do existing files follow naming conventions?** Every file should match `YYYY-MM-DD_HH-MM-SS_summary.md` or `YYYY-MM-DD_HH-MM-SS_transcript.md`. Flag misnamed files.
8. **Are summary/transcript pairs complete?** For each timestamp, both files should exist. Note gaps from interrupted runs.
9. **Are recent files well-formed?** Spot-check the most recent summary for expected sections and transcript for the correct heading. Note issues but don't rewrite old files.

### Reporting

- All clean: `Pre-flight: all good.`
- Warnings: list concisely, then proceed.
- Blockers (no git repo, merge conflicts): stop and explain.

---

## Step 1: Project File Organization Audit

Before saving context, audit the project structure. Messy file organization makes it harder for both humans and AI to navigate the codebase, which directly contributes to context rot — the AI wastes tokens figuring out where things are instead of doing useful work.

### What to check

1. **Root directory clutter.** List files in the project root. Flag files that likely belong in subdirectories:
   - Source files (`.py`, `.js`, `.ts`, etc.) sitting in root instead of `src/` or equivalent
   - Test files not in a `tests/` or `__tests__/` directory
   - Documentation files not in `docs/`
   - Temporary or scratch files (anything with `temp`, `tmp`, `scratch`, `test_`, `old_`, `backup_` in the name)
   - Multiple config files that could be consolidated

2. **Naming convention consistency.** Scan for inconsistencies:
   - Mixed casing (e.g., `MyFile.py` alongside `my_other_file.py`) — flag but don't rename
   - Spaces in filenames — flag these as problematic
   - Ambiguous names (e.g., `utils.py`, `helpers.py`, `misc.py` — multiple of these is a red flag)
   - Files with version suffixes (`_v2`, `_final`, `_old`, `_backup`, `_copy`) — suggest archiving

3. **Empty or near-empty directories.** Flag directories that are empty or contain only a single file — they may be leftover from abandoned work.

4. **Stale files.** Check `git log` for files that haven't been modified in the last 30 days but sit alongside active code. Mention these as candidates for archival.

5. **Standard structure audit.** Check if common directories exist based on the project type:
   - Python: `src/`, `tests/`, `docs/`, `scripts/`
   - JavaScript/TypeScript: `src/`, `tests/`, `public/`, `dist/`
   - General: `docs/`, `scripts/`, `assets/`, `config/`

### Reporting

Output a brief file organization report:
- `Files: well organized.` if nothing notable
- Otherwise, a short list of suggestions grouped by category (clutter, naming, stale files)

**Never rename, move, or delete files automatically.** Always report findings and let the user decide. If the user explicitly asks you to fix things, then proceed.

### Ongoing organization tracking

Append a `## File Organization Notes` section to the summary file (Step 2) that captures:
- Any new files created this session and where they were placed
- Organizational issues spotted
- Suggestions made to the user

This creates a trail so the next session can see if recurring issues are building up.

---

## Step 2: Save Conversation Context (Anti-Context-Rot Format)

The context save format is specifically designed to prevent context rot in future sessions. Research shows that structured handoff documents with clear sections for "intent", "state", and "next steps" significantly improve AI performance when bootstrapping new sessions.

Generate two files with a timestamped filename prefix (format: `YYYY-MM-DD_HH-MM-SS`):

### Summary / Handoff file: `context-logs/<timestamp>_summary.md`

This is the most important file — it's what a fresh session reads to get up to speed. Structure it as a **handoff document**, not just a summary. Keep it under 500 words (roughly 600-700 tokens) so it fits comfortably within the 10,000-token memory budget recommended for CLAUDE.md files.

```markdown
# Session Handoff — <timestamp>

## Session Intent
What was the user trying to accomplish? One paragraph max.

## Key Decisions
Architectural choices, library picks, trade-offs — anything a new session needs to know
to avoid re-discussing or contradicting. Bullet points.

## Changes Made
Which files were created, modified, or deleted and why. Include paths.
- `src/auth.py` — added JWT validation middleware
- `tests/test_auth.py` — created, covers token expiry edge cases

## Current State
Where things stand right now. What works, what's broken, what's partially done.

## Next Steps
Concrete, actionable items. Write these as instructions a fresh Claude session could
pick up and execute immediately.
1. Finish the refresh token endpoint in `src/auth.py`
2. Add rate limiting to the login route
3. Run `pytest tests/test_auth.py` — 2 tests were failing on token expiry

## File Organization Notes
Any organizational issues spotted or suggestions made this session.

## Context Health
- Context usage at save time: ~X% (if available)
- Session duration: approximate
- Recommendation: [continue / start fresh session]
```

The "Next Steps" section is critical — it's the bridge between sessions. Write them as if you're handing off to another developer who has full access to the codebase but zero conversation history.

The "Context Health" section helps the user decide whether to continue or start fresh. If context usage is above 60%, recommend starting a fresh session. Research shows the 0-30% context range is the "sweet spot" for AI performance.

### Transcript file: `context-logs/<timestamp>_transcript.md`

```markdown
# Conversation Transcript — <timestamp>

## Messages

### User
<user message content>

### Assistant
<assistant message content — summarize tool calls as one-liners>
```

For tool calls, write a one-line description (e.g., "Read `src/app.ts`", "Ran `npm test` — 12 passed") rather than full output. Keep it compact.

---

## Step 3: Smart Commit and Push

After saving context and organization notes:

1. **Check for uncommitted changes.** Run `git status --porcelain`.
   - If there are changes: stage everything with `git add -A`, then commit.
   - If only context files changed: stage and commit just those.
   - No changes at all: skip commit.

2. **Generate a detailed, standardized commit message.** Git log serves as long-term memory for the AI — future sessions will read recent commits to understand what's been happening. Make the message detailed enough to be useful as memory, not just a changelog entry.
   ```
   chore: auto-save checkpoint <timestamp>

   ## What was built/changed
   - <specific changes with file paths, 2-5 bullet points>

   ## Key decisions
   - <any architectural or design decisions made this session>

   ## AI layer improvements
   - <any changes to CLAUDE.md, skills, commands, or rules — track how the AI's own configuration evolves>
   ```
   The "AI layer improvements" section is inspired by the WISC framework — when the agent's own rules or commands improve during a session, documenting that in the commit helps future sessions understand how the AI setup has evolved.

3. **Push to the remote.** Run `git push`. If the current branch has no upstream, use `git push -u origin <branch-name>`. If push fails, report clearly — don't retry in a loop.

---

## Scheduling

To run every 30 minutes:

```
/loop 30m /auto-save-and-push
```

When running on a schedule, keep output minimal — just confirm what was saved and pushed, or report errors.

---

## Proactive Reminders

Every run should end with a short **action nudge** — a 1-3 line reminder telling the user what they should do next based on the current state. This is the most important user-facing output of the skill. Evaluate these conditions and output the matching reminder:

### Context health reminders

| Context Usage | Reminder |
|---|---|
| 0-30% | `Context is fresh. You're in the sweet spot — keep going.` |
| 30-50% | `Context is building up. Consider wrapping your current task soon and saving.` |
| 50-60% | `Context is getting heavy. Finish what you're doing, then start a fresh session. Run /compact with focus instructions if you need more room right now.` |
| 60-80% | `Warning: context rot is likely happening. Save now and start a fresh session. If you must continue, run: /compact Focus on: [current task specifics]` |
| 80%+ | `Critical: you're deep in context rot territory. Saving your handoff now — please start a fresh session after this. Your AI's quality has degraded significantly at this level.` |

### Session behavior reminders

Check for these patterns and remind accordingly:

- **Planning and coding in the same session**: If the conversation includes both research/planning AND implementation, remind: `Tip: next time, plan in one session, then save and implement in a fresh session. This keeps your implementation context clean and focused.`

- **Multiple compactions detected**: If `/compact` has been run before in this session, remind: `You've already compacted once this session. The best compression is not needing compression — save your handoff and start fresh.`

- **Long session without saves**: If there's no recent entry in `context-logs/` (nothing in the last 30+ minutes) and context is above 30%, remind: `It's been a while since your last save. Consider checkpointing now to avoid losing context if something goes wrong.`

- **Messy files detected (from Step 1)**: If the file organization audit found issues, remind: `Your project has some organizational issues — check the file org report above. Cleaning these up will help your AI navigate the codebase faster (fewer wasted tokens = less context rot).`

- **No .gitignore or incomplete .gitignore**: Remind: `Your .gitignore was missing some entries — I've added them. Double-check nothing sensitive has already been committed with: git log --diff-filter=A --name-only`

- **Stale handoff files**: If the most recent summary in `context-logs/` is more than 7 days old, remind: `Your last context save was over a week ago. If you've been working without saves, you may want to do a comprehensive handoff to capture recent progress.`

### Format

Output reminders in a clear, boxed format at the very end of the skill's output:

```
──────────────────────────────────
 REMINDERS
 - Context at ~45%. Wrap up soon and consider a fresh session.
 - Tip: separate planning and implementation into different sessions.
──────────────────────────────────
```

Keep it brief — max 3 reminders per run. Prioritize by urgency (context health > session behavior > file organization).

---

## Using Handoffs to Start Fresh Sessions (WISC Framework)

The WISC framework (Write, Isolate, Select, Compress) from 2000+ hours of Claude Code usage establishes that context rot causes ~80% of AI coding failures. Here's how this skill applies it:

### Bootstrapping a new session

When starting fresh after a save, the user (or an automated hook) can bootstrap context:

```
Read context-logs/ and find the latest _summary.md file. Also run
`git log --oneline -20` to see recent commits as long-term memory.
Use the handoff to continue from the "Next Steps" section.
```

### The plan/implement split

For best results, the user should separate planning from implementation:
1. **Planning session** — research, discuss architecture, create a structured plan as a markdown spec
2. **Save and push** (this skill) — capture everything
3. **Fresh implementation session** — start clean, load ONLY the structured plan. This keeps the implementation context laser-focused with zero research noise

This is far more effective than continuing a degraded long session. A fresh session with a good handoff document operates in the 0-30% context sweet spot, where AI performance is strongest.

### When compaction is unavoidable

If the user can't start fresh and must use `/compact`, advise them to provide focused instructions:
```
/compact Focus on: [specific topic they care about], the edge cases we tested,
and the key decisions from this session. Drop the research and exploration details.
```
Focused compaction preserves what matters instead of letting the system decide. But remind the user: **the best compression strategy is not needing compression.** If they've compacted more than once in a session, it's time for a handoff and fresh start.

### Git log as long-term memory

The standardized commit messages this skill generates serve as long-term memory across sessions. A new session can run `git log --oneline -20` to quickly understand what's been happening in the project without loading any context files. This is why the commit messages are detailed and structured — they're not just for git history, they're for AI memory.

---

## Important notes

- **Never force-push.** Always use regular `git push`.
- **Never auto-resolve merge conflicts.** Stop and tell the user.
- **Never rename, move, or delete files** without explicit user permission.
- **Respect `.gitignore`** — `git add -A` handles this automatically.
- **Context logs grow over time.** That's fine — text files are small. Suggest archiving logs older than 30 days to `context-logs/_archive/` if the folder gets crowded.
- **If context usage > 60%**, proactively recommend the user start a fresh session after the save. This is the single most effective way to prevent context rot.
