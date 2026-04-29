---
name: cowork-sync
description: "Generate a comprehensive Claude Code sync handoff at the end of every Cowork session that modifies files. This skill should trigger whenever the session is wrapping up, the user asks what changed, asks for a summary of work done, says 'commit this', 'push this', 'tell claude code', 'sync', 'what do I need to tell claude code', or any time Cowork has made changes to dashboard files, memory files, CLAUDE.md, spreadsheets, or any repo files. Also trigger proactively when you detect the conversation is winding down and changes have been made. Do not wait to be asked — surface the sync automatically."
---

# Cowork → Claude Code Sync Skill

## Why this exists

Cowork and Claude Code operate in the same git repo but in separate sessions with no shared memory. When Cowork makes changes, Claude Code has no idea what happened unless someone tells it. If the sync is thin or vague, Claude Code will operate on stale assumptions — wrong function names, outdated data structures, missing context about why something was built a certain way. This leads to bugs, reverted work, and wasted time.

The goal is to produce a handoff so detailed that Claude Code could pick up exactly where Cowork left off, as if it had been watching the whole session.

## When to generate this

Generate the sync report:
- At the end of every session where any files were modified
- When the user asks what changed or what to tell Claude Code
- Proactively when you sense the conversation is wrapping up and work was done
- When the user says "commit", "push", "sync", or similar

If no files were changed in the session, say so explicitly: "No Claude Code sync needed this session."

## The sync report structure

The report has 4 required sections. All 4 must be present every time — don't skip any.

### 1. Files changed

List every file that was created, modified, or deleted. Use paths relative to the repo root. Group logically (e.g., dashboard files together, memory files together).

For each file, include a brief note about what changed — not just the filename.

Example:
```
1. `monthly financial analysis.html` — Added MRR vs Billing tab, new MRR_BILLING_MAP data array
2. `index.html` — Mirror copy of above
3. `.claude/memory/dashboard.md` — Updated tab count, added new functions to Key Functions list
```

### 2. Git instructions (copy-paste ready)

Provide the exact commands Claude Code should run. These must be copy-paste ready — no placeholders, no "fill in the blank." Always include:

- `cd` to the repo directory (`/Users/scott/coding-projects/datapath`)
- `git add` with specific filenames (never `git add .` or `git add -A`)
- `git commit -m` with a descriptive multi-line message that captures the why, not just the what
- `git push origin main`

Use a heredoc or multi-line commit message when there are multiple logical changes. The commit message should be detailed enough that someone reading `git log` six months from now understands what was done and why.

### 3. Detailed code changes

This is the section most people skip and it's the most important one. For every code change, explain:

**New functions/variables added:**
- Function name, what it does, where it lives (line number or "added before renderPnL()")
- Parameters and return values
- Any aliases, fallbacks, or era-handling logic

**Modified functions:**
- What changed and why
- The before/after behavior difference
- Any new parameters or options added

**New data structures:**
- Array/object name, field schema, number of entries
- What populates it and what consumes it

**CSS changes:**
- New classes, what elements they target, visual effect

**Bug fixes:**
- What the bug was, what caused it, how it was fixed
- Why the old code was wrong (so Claude Code doesn't accidentally reintroduce it)

**Wiring/plumbing:**
- Nav entries, routing additions, state variables
- Any new scheduled tasks or automation

Think of this section as a code review for someone who can't see the diff. They need to understand not just what changed, but the architectural decisions behind it.

### 4. Memory and context changes

Summarize what Claude Code needs to know about the project that changed during this session. This includes:

- New conventions or patterns established
- Data model changes (new arrays, renamed fields, deprecated approaches)
- Business logic changes (new calculations, formula changes)
- Anything that was discussed or decided that would affect how Claude Code works on the project
- Updated numbers (revenue figures, deal counts, dates)
- New scheduled tasks or automation
- Changes to CLAUDE.md or memory files and what specifically was added/removed

If memory files were updated, summarize the key additions so Claude Code gets the information even before re-reading the files.

## Quality checks

Before presenting the sync:

- Did you list every file that was touched? Check your work — scroll through what you did this session.
- Are the git commands actually copy-paste ready? No missing quotes, no placeholder paths.
- Would Claude Code understand a new function you added well enough to modify it correctly?
- If you fixed a bug, did you explain what caused it so it doesn't get reintroduced?
- Did you mention any new helper functions, data objects, or CSS classes by name?
- If a scheduled task was created or modified, is it documented?

## Example output format

Here's the shape of a good sync (abbreviated):

---

**Claude Code Sync — [Date] Session**

**Files changed:**
1. `monthly financial analysis.html` — [what changed]
2. `index.html` — Mirror copy
3. `.claude/memory/dashboard.md` — [what changed]

**Git instructions:**
```
cd /Users/scott/coding-projects/datapath
git add "monthly financial analysis.html" index.html .claude/memory/dashboard.md
git commit -m "Add MRR vs Billing tab with 62 Q1 deals mapped to CW status

- MRR_BILLING_MAP tracks billing/unbilled/verified/unverified per deal
- Stacked bar chart + filter buttons + deal tables
- mrrMonthIdx() helper for proper month comparison
- Weekly audit scheduled task (Wednesdays 9am)"
git push origin main
```

**Detailed code changes:**

[Function-by-function, variable-by-variable breakdown with enough detail that Claude Code could modify any of these without re-reading the full file]

**Memory/context changes:**

[What Claude Code's mental model of the project should now include that it didn't before]

---

The sync should be thorough enough that if someone handed it to a developer who'd never seen the project, combined with the existing CLAUDE.md and memory files, they could pick up exactly where you left off.
