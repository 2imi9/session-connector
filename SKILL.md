---
name: session-connector
description: Bridge multi-session research and iteration work — keep parallel investigation threads alive across sessions instead of re-deriving context every time. Use whenever the user signals end-of-session ("save session", "checkpoint", "let's wrap up", "see you tomorrow", "before I sleep"), start-of-session ("where were we", "continue from last session", "resume", "what was I working on"), or asks to survey active threads ("what's on my plate", "show open investigations"). Always invoke if there's any chance the user wants to capture, continue, or survey session state — even if they don't say "skill" or "connector". Especially valuable for iterative work like Jupyter notebook research, multi-experiment studies, and any project where investigations span days or weeks. Complements long-lived auto-memory: memory is durable facts; this skill is the live investigation stack.
---

# session-connector

Bridge between sessions for the same project. Treats each session as a contributor to long-running **investigation threads**, not an isolated time-snapshot. Investigation arcs span days or weeks; this skill keeps them coherent across session boundaries.

## When to invoke

| User signal | Mode |
|---|---|
| "save session" / "checkpoint" / "see you tomorrow" / "before I sleep" | **save** |
| "where were we" / "continue from last session" / "resume" / "this is new session" | **resume** |
| "what's on my plate" / "show active threads" / "give me a status" | **status** |
| Truly ambiguous | Ask once: "Save current state, resume previous session, or show active threads?" |

If the user is mid-task and not signaling a session boundary, **don't invoke**. Just do the work.

## Where state lives

`<repo-root>/.claude/sessions/`, found via `git rev-parse --show-toplevel`.

```
.claude/sessions/
├── HEAD.md                        # last session + active thread list
├── threads/
│   └── <thread-name>.md           # one file per investigation arc
└── <YYYY-MM-DD_HHMM>/             # per-session folders
    ├── state.md
    ├── next.md
    ├── advanced_threads.md
    ├── links.md
    └── git_snapshot.txt
```

Sessions are scoped to the worktree root if you're in one. Don't auto-edit `.gitignore` — mention it once and let the user decide.

## File formats

### Thread file — `threads/<name>.md`

```markdown
---
name: <slug>
status: active                # active | dormant | resolved
opened: YYYY-MM-DD
last_touched: YYYY-MM-DD
parent: <other-thread>        # optional
---

# Thread: <human-readable title>

## HEAD
One paragraph: where this investigation stands right now. Rewrite in full every session that touches the thread.

## Chronology
- YYYY-MM-DD (session YYYY-MM-DD_HHMM) — <one-line of what advanced>

## Open questions
- ...

## Connects to
- Thread: <other> (parent / sibling / blocks / blocked-by)
- Memory: <memory-file.md>
- External: <PR / paper / dashboard URL>
```

### Session folder — `<timestamp>/`

- **`state.md`** — narrative. Sections: "What we did this session" (3-6 bullets, past tense, load-bearing only); "Current state of investigation" (1 paragraph); "Files in flight" (paths + 1-line description, only actively-edited files).
- **`next.md`** — explicit numbered next-actions, lead with most likely. Each item under two lines.
- **`advanced_threads.md`** — list of thread names this session contributed to, with one line each on how the HEAD changed. This is the explicit link from a session to the threads it advanced.
- **`links.md`** — pointers, no prose. Sections: Open PRs, Modified files, Memory entries to re-read, External references.
- **`git_snapshot.txt`** — raw concatenated git/gh output. Unprocessed truth.

### Top-level — `HEAD.md`

```markdown
# session-connector HEAD

## Last session
YYYY-MM-DD_HHMM — <one-line summary> (advanced: <thread names>)

## Active threads
- [<thread-name>](threads/<thread-name>.md) — last touched <date> — <HEAD one-liner>

## Dormant threads
- [<thread-name>](threads/<thread-name>.md) — last touched <date>

## Resolved threads
- [<thread-name>](threads/<thread-name>.md) — resolved <date>
```

Sort active threads by `last_touched` descending. Dormant/resolved sections are archival — don't read them on resume.

## Operations

### save (end-of-session)

1. **Identify which threads this session touched.** Read the conversation. Match to existing threads in `threads/`. If genuinely new investigation, create a new thread file (default status: active).

2. **Gather git/PR state in parallel:**
   - `git rev-parse --show-toplevel`
   - `git rev-parse --abbrev-ref HEAD`
   - `git status --short`
   - `git log --oneline -10`
   - `gh pr list --state open --author @me 2>/dev/null` (skip silently if `gh` not installed)

3. **Update each touched thread:** append a chronology line, **rewrite the HEAD section in full** (don't accumulate — replace), bump `last_touched` in frontmatter. Update `status` only if user explicitly signaled.

4. **Write the session folder** at `<timestamp>/` with all five files.

5. **Update `HEAD.md`:** set "Last session", re-sort active threads by `last_touched`, move any whose status changed to its appropriate section.

6. **Confirm** in 1-2 lines: path + threads advanced + top of next-action stack. Don't recap content the user just produced with you.

### resume (start-of-session)

1. **Read `HEAD.md` only.** Don't read thread files yet.

2. **Surface active threads** (not just the latest session's content):

   > Last session ({date}): {one-line of what was advanced}.
   >
   > **Active threads:**
   > - **{thread-name}** — {HEAD one-liner from HEAD.md}
   >
   > **Top of next-action stack** (from latest session): {first item from `next.md`}
   >
   > Which thread are you picking up — or new direction?

3. **Wait for direction** before deep-diving. Then read the chosen thread file in full.

4. **Verify before recommending:**
   - Thread mentions a PR by number → check it's still open (`gh pr view <num>`).
   - Thread mentions a file path → check the file exists.
   - Thread untouched in >7 days → flag staleness.

### status (anytime)

Read `HEAD.md`, surface active threads with HEAD one-liners. No deep-dive.

### branch (implicit)

When the user starts a new line of investigation that doesn't fit any existing thread, create a new thread file in the next save. Set `parent` if it spun off from an existing thread.

### merge / consolidate (implicit)

If a thread reaches a stable, durable conclusion, suggest the `consolidate-memory` skill to promote findings into auto-memory, and mark the thread as `resolved`.

## What goes where

- **Session log**: true *about this specific session*. Specific, time-stamped, won't matter in 6 months.
- **Thread**: true *about the current state of an investigation arc*. Spans sessions, evolves, eventually resolves.
- **Memory** (auto-memory): true *forever about the user, project, or settled findings*.

If unsure: would it feel weird to still be reading this entry next year? If yes → session log or thread. If no → memory.

When writing thread files, **reference** memory by filename in "Connects to"; don't duplicate content. Memory loads automatically next session.

## Anti-patterns

- **Don't dump conversation into thread HEADs.** A HEAD is a current-state summary, rewritten each session. Chronology is for one-liners, not transcripts.
- **Don't auto-resolve threads.** Only the user knows when an investigation is closed.
- **Don't create a thread for every casual mention.** Threads are for arcs that span (or will plausibly span) multiple sessions.
- **Don't read every thread on resume.** Surface HEAD one-liners only; read full files when the user picks one to dig into.
- **Don't auto-commit, push, or stage** any session-connector files.
- **Don't duplicate auto-memory content.** Reference by filename in "Connects to".
- **Don't snapshot mid-flow.** "Save this file" while editing code is not a session boundary.
