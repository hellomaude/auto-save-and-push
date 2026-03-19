---
name: session-health
description: >
  Check the health of your current Claude Code session. Shows context usage, session quality indicators,
  and recommendations for when to compact or start fresh. Use when you want to know if your session
  is still performing well or if context rot is setting in.
---

# Session Health Check

Run a quick diagnostic on the current session and report back with actionable guidance.

## 1. Context Usage

Check the current context window usage. Report:
- **Context used**: approximate percentage
- **Zone**: which zone the session is in (see table below)

| Zone | Usage | Status |
|------|-------|--------|
| Sweet Spot | 0-30% | Performing at peak. Keep going. |
| Caution | 30-50% | Still good, but start wrapping up complex tasks. |
| Degraded | 50-70% | Quality is dropping. Finish current task, then save and start fresh. |
| Critical | 70-85% | Context rot is active. Save immediately and start a fresh session. |
| Emergency | 85%+ | Compaction imminent. Run `/auto-save-and-push` NOW, then start fresh. |

## 2. Session Behavior Analysis

Check for these patterns and flag any that apply:

- **Task mixing**: Has this session included both research/planning AND implementation? If so, flag it — mixed sessions degrade faster.
- **Prior compactions**: Has `/compact` been run in this session? Each compaction loses fidelity. More than one is a strong signal to start fresh.
- **Conversation length**: Estimate the number of user/assistant turns. Flag if over 30 turns — long conversations accumulate noise even at low context percentages.
- **Error loops**: Has the session hit repeated failures on the same task (3+ attempts)? This is a classic context rot symptom — the AI keeps trying the same broken approach.
- **Topic switches**: Has the conversation switched between unrelated topics? Each switch fragments the context.

## 3. Last Save Check

Check `context-logs/` in the current project directory (if it exists):
- When was the last save? Report the timestamp.
- If no saves exist or the last save was 30+ minutes ago, flag it.
- If no `context-logs/` directory exists, note that `/auto-save-and-push` hasn't been run yet.

## 4. Git Status Quick Check

If this is a git repo:
- Are there uncommitted changes? How many files?
- How many commits since the last push?
- Flag if there's significant unsaved work at risk.

## 5. Output Format

Present results in this exact format:

```
╔══════════════════════════════════╗
║        SESSION HEALTH            ║
╠══════════════════════════════════╣
║ Context:  ~XX% [██████░░░░] ZONE ║
║ Turns:    ~XX                    ║
║ Last save: XX min ago / never    ║
║ Unsaved files: X                 ║
╠══════════════════════════════════╣
║ DIAGNOSIS                        ║
║ - [any flags from step 2]        ║
╠══════════════════════════════════╣
║ RECOMMENDATION                   ║
║ [one clear action to take]       ║
╚══════════════════════════════════╝
```

### Recommendation logic

Pick ONE primary recommendation based on priority:

1. If context > 70%: **"Save and start a fresh session now."**
2. If context > 50%: **"Finish your current task, then save and start fresh."**
3. If error loops detected: **"You may be stuck in a loop. Save context and retry in a fresh session."**
4. If last save > 60 min ago and context > 30%: **"Run /auto-save-and-push to checkpoint your work."**
5. If task mixing detected: **"Next time, split planning and implementation into separate sessions."**
6. If multiple compactions: **"You've compacted multiple times. Save and start fresh for best results."**
7. Otherwise: **"Session is healthy. Keep going."**

Keep the entire output compact — no extra commentary beyond the box.
