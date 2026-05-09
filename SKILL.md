---
name: session-connector
description: Bridge sessions for the same project by treating sessions as contributors to long-running investigation threads, not isolated time-snapshots. Use whenever the user signals end-of-session ("save session", "checkpoint", "let's wrap up", "see you tomorrow", "snapshot this", "before I sleep"), start-of-session ("what were we working on", "continue from last session", "resume", "pick up where we left off", "where were we"), or asks "what's on my plate / what threads are open / show active investigations". Always invoke this skill if there's any chance the user wants to capture, continue, or survey session state — even if they don't say "skill" or "connector" explicitly. Designed around the KV-cache mental model: each new session attends to a cache of accumulated thread HEADs rather than re-deriving prior reasoning. Complements (does not replace) the long-lived auto-memory: memory is project-level facts that hold forever; this skill is the in-flight investigation stack that grows and prunes session by session.
---

# session-connector

Bridge between sessions for the same project. Designed around the KV-cache mental model: each new session **attends to a cache of accumulated thread state** rather than re-deriving prior reasoning from scratch.

The point isn't snapshot-and-restore. It's that real research/engineering work has structure — investigation arcs that span many sessions, parallel threads that proceed independently, dormant threads that may be picked up later. session-connector preserves that structure across session boundaries.

## The KV-cache mental model

| KV cache | session-connector |
|---|---|
| Each new token attends to prior K/V pairs in a cache | Each new session attends to active thread HEADs in `HEAD.md` |
| O(1) marginal cost per token | O(1) marginal cost per session (only touched threads update) |
| Multi-head attention runs heads in parallel | Multiple investigation threads run in parallel |
| Cache has layers at different abstractions | L0 = session log, L1 = thread state, L2 = auto-memory |
| Eviction / compression of old K/V | Threads transition active → dormant → resolved |

The three layers, from most concrete to most durable:
- **L0 — Session log** (per-session): the K/V of "the most recent token." Time-stamped folder, narrative state, next-actions.
- **L1 — Thread state** (per investigation arc): aggregated K/V across sessions in that arc. Has a HEAD (current state) and chronology (the chain of advances).
- **L2 — Project memory** (auto-memory): durable facts. Already handled by Claude Code's memory system — referenced from L1, never duplicated.

## When to invoke

| User signal | Mode |
|---|---|
| "save session" / "checkpoint" / "wrap up" / "see you next session" / "before I sleep" / "good night" | **save** |
| "where were we" / "continue from last session" / "resume" / "pick up where we left off" / "what was I working on" / "this is new session" + project context | **resume** |
| "what's on my plate" / "show active threads" / "what investigations are open" / "give me a status" | **status** |
| Truly ambiguous | Ask once: "Save current state, resume previous session, or show active threads?" |

If the user is mid-task and not signaling a session boundary, **don't invoke**. Just do the work.

## Where state lives

`<repo-root>/.claude/sessions/` — found via `git rev-parse --show-toplevel`.

```
.claude/sessions/
├── HEAD.md                          # top-level: last session + active thread list
├── threads/
│   ├── <thread-name>.md             # one file per investigation arc
│   └── ...
└── <YYYY-MM-DD_HHMM>/               # per-session folders
    ├── state.md
    ├── next.md
    ├── advanced_threads.md
    ├── links.md
    └── git_snapshot.txt
```

If you're in a git worktree, `git rev-parse --show-toplevel` returns the worktree root — sessions are scoped to the line of work in that worktree. Mention this once if it might surprise the user; otherwise just use it.

Don't auto-edit `.gitignore`. If `.claude/` is already gitignored, you're covered. Otherwise mention it once and let the user decide.

## File formats

### Thread file (`threads/<name>.md`) — the L1 cache

```markdown
---
name: dinn-cross-validation
status: active                  # active | dormant | resolved
opened: 2026-05-08
last_touched: 2026-05-09
parent: dinn-architecture       # optional, if spun off another thread
---

# Thread: <human-readable title>

## HEAD
One paragraph: where this investigation stands right now. Rewrite in full each session that touches the thread — this is the "current K/V" the next session will attend to.

## Chronology
- 2026-05-08 (session 2026-05-08_1500) — <one-line of what advanced>
- 2026-05-09 (session 2026-05-09_2240) — <one-line of what advanced>

## Open questions
- ...

## Connects to
- Thread: <other-thread> (parent / sibling / blocks / blocked-by)
- Memory: <memory-file.md>
- External: <PR / paper / dashboard URL>
```

### Session folder (`<timestamp>/`) — the L0 cache

- **`state.md`** — narrative of where the session left off. Sections: "What we did this session" (3-6 bullets, past tense, load-bearing only), "Current state of investigation" (1 paragraph), "Files in flight" (paths + 1-line description, only actively-edited files).
- **`next.md`** — explicit numbered next-actions. Lead with the single most-likely-next action. Each item under two lines.
- **`advanced_threads.md`** — list of thread names this session contributed to, with one line each describing how the HEAD changed. This is what makes the cache *connective* — the explicit link from a session to the threads it advanced.
- **`links.md`** — pointers, no prose. Sections: Open PRs, Modified files, Memory entries to re-read, External references.
- **`git_snapshot.txt`** — raw concatenated output from the git/gh commands. Unprocessed truth, used to verify the prose hasn't drifted from what was actually committed.

### Top-level (`HEAD.md`)

```markdown
# session-connector HEAD

## Last session
2026-05-09_2240 — <one-line summary> (advanced threads: <names>)

## Active threads
- [<thread-name>](threads/<thread-name>.md) — last touched <date> — <HEAD one-liner>
- ...

## Dormant threads
- [<thread-name>](threads/<thread-name>.md) — last touched <date>

## Resolved threads
- [<thread-name>](threads/<thread-name>.md) — resolved <date>
```

Sort active threads by `last_touched` descending. Dormant/resolved sections are archival — don't read them on resume.

## Operations

### save (end-of-session snapshot)

1. **Identify which threads this session touched.** Read the conversation. Match to existing thread files in `threads/`. If the user worked on something genuinely new, create a new thread file (default status: active).

2. **Gather git/PR state in parallel:**
   - `git rev-parse --show-toplevel`, `git rev-parse --abbrev-ref HEAD`, `git status --short`, `git log --oneline -10`, `gh pr list --state open --author @me 2>/dev/null` (skip silently if `gh` not installed).

3. **Update each touched thread file:**
   - Append a chronology line with today's date and a one-liner of what advanced.
   - **Rewrite the HEAD section in full** to reflect the current state. Don't accumulate — replace. The HEAD is "what the next session needs to attend to right now," not a transcript.
   - Bump `last_touched` in frontmatter. Update `status` only if the user explicitly signaled (e.g., "we resolved that").

4. **Write the session folder** at `<YYYY-MM-DD_HHMM>/` with all five files.

5. **Update `HEAD.md`:**
   - Set "Last session" to this folder + the threads it advanced.
   - Re-sort active threads by `last_touched`.
   - Move any thread whose status changed to its appropriate section.

6. **Confirm to user** in 1-2 lines. Path + threads advanced + top of next-action stack. Don't recap content the user just produced with you.

### resume (start-of-session)

1. **Read `HEAD.md`.** This is the cache we're attending to. It alone is enough for a high-level resume — don't read thread files yet.

2. **Surface active threads** (not the latest session's contents). Format:

   > Last session ({date}): {one-line of what was advanced}.
   >
   > **Active threads:**
   > - **{thread-name}** — {HEAD one-liner from HEAD.md}
   > - **{thread-name}** — {HEAD one-liner from HEAD.md}
   >
   > **Top of next-action stack** (from latest session): {first item from `next.md`}
   >
   > Which thread are you picking up — or new direction?

3. **Wait for user direction** before deep-diving. Then read the relevant thread file(s) in full.

4. **Verify before recommending action:**
   - If a thread's HEAD references a PR by number, check it's still open (`gh pr view <num>` or `gh pr list`).
   - If it references a file path, check the file still exists.
   - If a thread hasn't been touched in >7 days, surface that explicitly — the HEAD may be stale.

### status (anytime)

1. Read `HEAD.md`. Surface active threads with HEAD one-liners. No deep-dive.
2. Useful for "what was I working on lately?" mid-session, or for a quick survey before deciding what to pick up.

### branch (implicit)

When the user starts a new line of investigation that doesn't fit any existing thread, create a new thread file during the next save. Set `parent` if it spun off from an existing thread. Don't ask permission for routine branching — only flag if it seems like the user might be re-opening something already resolved.

### merge / consolidate (implicit, calls another skill)

If a thread reaches a stable, durable conclusion ("we ruled out X for reason Y, this is final"), suggest invoking the `consolidate-memory` skill to promote the finding into auto-memory, and mark the thread as `resolved`. The split: thread = active investigation, memory = settled conclusion.

## What goes where (the L0/L1/L2 split)

The decision rule:
- **L0 (session log)**: anything that's true *about this specific session*. "Today we ran nb16 and verified the leak fix." Specific, time-stamped, won't be relevant in 6 months.
- **L1 (thread)**: anything that's true *about the current state of an investigation arc*. "DINNDeep is interpolation-only — random CV passes, block CV fails." Spans sessions, evolves, eventually resolves.
- **L2 (memory)**: anything that's true *forever about the user, project, or settled findings*. "User prefers terse responses." "Carroll 2022 is the active calibration target."

If you're unsure: would it feel weird to still be reading this entry next year? If yes → L0 or L1. If no → L2.

When writing L1 thread files, *reference* L2 memory by filename in the "Connects to" section rather than copying content. Memory loads automatically next session anyway.

## Anti-patterns

- **Don't dump the conversation into thread HEADs.** A HEAD is a *summary of current state*, rewritten every session. Chronology is the place for accumulated history, and it should be one-liners.
- **Don't auto-resolve threads.** Only the user knows when an investigation is truly closed. When in doubt, leave it active.
- **Don't create a thread for every casual mention.** Threads are for arcs that span (or will plausibly span) multiple sessions. A one-off task that was completed in a single session is just a session log entry.
- **Don't read every thread on resume.** Surface HEAD one-liners from `HEAD.md`; only read full thread files when the user picks one to dig into.
- **Don't auto-commit, push, or stage** any session-connector files. The user owns the gitignore decision.
- **Don't duplicate auto-memory content into threads.** Reference by filename in "Connects to."
- **Don't snapshot mid-flow.** "Save this file" while editing code is not a session boundary. Read context.

## Why this structure

Compared to a flat snapshot/resume:
- Resume is **fast**: read 1 file (`HEAD.md`) instead of N (one per past session).
- Multi-session arcs are **first-class**: chronology in a thread file is the actual chain of reasoning, preserved.
- Parallel investigations don't get tangled: each thread maintains its own HEAD.
- Promotion to memory is **explicit** (via `consolidate-memory`), so the durable-vs-in-flight split stays clean.
- The KV-cache property holds: O(1) marginal write per session, O(active threads) read on resume.
