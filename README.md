# Auto Save and Push — Claude Code Skill

A Claude Code skill that prevents **context rot**, organizes your project files, and automatically pushes everything to a private Git repo. Designed for developers who want to keep their AI coding sessions fresh and their projects backed up.

## What is Context Rot?

Context rot is the gradual degradation of AI response quality as conversations get longer. Research shows that once context usage exceeds 50%, AI coding assistants start pulling wrong information, hallucinating, and forgetting earlier decisions. This skill fights that by saving structured handoff documents so you can start fresh sessions without losing any context.

## What This Skill Does

Every time it runs (manually or on a 30-minute loop), it performs **4 steps**:

### 1. Pre-Flight Validation
- Checks git repo status, merge conflicts, remote configuration
- Creates or validates `.gitignore` with sensible defaults for your project type
- Audits the `context-logs/` folder for consistency and correct formatting
- Monitors context window health and flags when degradation is likely

### 2. File Organization Audit
- Flags root directory clutter, misplaced source files, and naming inconsistencies
- Detects stale files, empty directories, and version-suffix files (`_v2`, `_final`, `_old`)
- Checks for standard project structure (`src/`, `tests/`, `docs/`)
- Never auto-fixes — reports issues and lets you decide

### 3. Context Save (Anti-Context-Rot Format)
Saves two files to `context-logs/`:
- **Handoff summary** — structured document with Session Intent, Key Decisions, Changes Made, Current State, Next Steps, and Context Health. Designed so a fresh session can pick up exactly where you left off.
- **Conversation transcript** — compact record with tool calls summarized as one-liners.

### 4. Smart Commit and Push
- Stages all changes and generates detailed, standardized commit messages that serve as **long-term AI memory** (future sessions read `git log` to catch up)
- Commit messages include: What was built, Key decisions, and AI layer improvements
- Pushes to your remote — warns if no remote is configured, never force-pushes

### 5. Proactive Reminders
Every run ends with actionable nudges based on current state:
- Context health warnings (from "you're in the sweet spot" to "critical: start fresh now")
- Tips like "separate planning and implementation into different sessions"
- Warnings about missing `.gitignore` entries or stale handoff files

## Built on Research

This skill incorporates strategies from:
- **WISC Framework** (Write, Isolate, Select, Compress) — from 2000+ hours of Claude Code usage
- **GSD Framework** — keeping tasks in the 0-30% context sweet spot
- **VNX Rotation Protocol** — automatic context rotation via handoff documents
- **Chroma Research** — on how increasing input tokens impacts LLM performance

## Installation

```bash
claude install-skill auto-save-and-push.skill
```

## Usage

### One-time save
```
/auto-save-and-push
```

### Recurring auto-save (customizable interval)
```
/loop 30m /auto-save-and-push   # every 30 minutes (recommended)
/loop 15m /auto-save-and-push   # every 15 minutes (heavy sessions)
/loop 1h /auto-save-and-push    # every hour (light work)
/loop 10m /auto-save-and-push   # every 10 minutes (rapid iteration)
```
Set whatever interval works for your workflow. 30 minutes is a good default — frequent enough to prevent context rot, infrequent enough to not interrupt your flow.

### Starting a fresh session from a handoff
```
Read context-logs/ and find the latest _summary.md file. Also run
git log --oneline -20 to see recent commits as long-term memory.
Use the handoff to continue from the "Next Steps" section.
```

## How It Prevents Context Rot

| Strategy | How This Skill Applies It |
|---|---|
| Externalize memory | Handoff summaries + detailed git commits as long-term memory |
| Fresh sessions | Context health monitoring with proactive "start fresh" recommendations |
| Plan/implement split | Reminds you to separate planning and coding into different sessions |
| Focused compaction | When you can't start fresh, advises specific `/compact` instructions |
| File organization | Keeps project structure clean so AI wastes fewer tokens navigating |

## License

MIT
