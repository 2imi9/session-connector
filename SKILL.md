---
name: session-connector
description: Bridge multi-session research and iteration work — keep parallel investigation threads alive across sessions instead of re-deriving context every time. Use whenever the user signals end-of-session ("save session", "checkpoint", "let's wrap up", "see you tomorrow", "before I sleep"), start-of-session ("where were we", "continue from last session", "resume", "what was I working on"), or asks to survey active threads ("what's on my plate", "show open investigations"). Always invoke if there's any chance the user wants to capture, continue, or survey session state — even if they don't say "skill" or "connector". Especially valuable for iterative work like Jupyter notebook research, multi-experiment studies, and any project where investigations span days or weeks. Complements long-lived auto-memory — memory is durable facts while this skill is the live investigation stack.
---

# session-connector

Bridge between sessions for the same project. Treats each session as a contributor to long-running **investigation threads**, not an isolated time-snapshot. Investigation arcs span days or weeks; this skill keeps them coherent across session boundaries.

## When to invoke

| User signal | Mode |
|---|---|
| "save session" / "checkpoint" / "see you tomorrow" / "before I sleep" | **save** |
| "where were we" / "continue from last session" / "resume" / "this is new session" | **resume** |
| "what's on my plate" / "show active threads" / "give me a status" | **status** |
| "set up session-connector" / "configure storage" / "where should sessions go" | **setup** |
| Truly ambiguous | Ask once: "Save, resume, show status, or set up storage?" |

**First-time bootstrap:** if no `.config.json` exists and `.claude/sessions/` is empty (or doesn't exist) on the first session-connector operation of any kind, run **setup** first — the user gets to choose between repo-local and external-drive storage before any data is written. Then proceed with the originally-requested operation. Once setup has completed (config exists), subsequent operations skip the bootstrap and read the config directly.

If the user is mid-task and not signaling a session boundary, **don't invoke**. Just do the work.

## Where state lives

By default: `<repo-root>/.claude/sessions/`, found via `git rev-parse --show-toplevel`.

```
.claude/sessions/
├── .config.json                   # optional — points to alternate storage
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

### Storage path resolution

On every operation (save / resume / status), resolve the actual sessions root in this order:

1. If `<repo-root>/.claude/sessions/.config.json` exists, use `config["storage_path"]`. The repo-side `.claude/sessions/` directory only holds the config; everything else lives at the configured path.
2. Otherwise, use `<repo-root>/.claude/sessions/` directly.

If the configured path is unreachable (e.g., USB drive not plugged in):
- Tell the user clearly: "Configured storage at `{path}` isn't accessible. Drive `{drive_label}` may be unplugged."
- Offer: plug in and retry, re-run setup, or fall back to repo-local for this session only.

**Don't auto-create state in the wrong location** when storage is unreachable — losing track of where the canonical state lives is much worse than asking the user.

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

### setup (first-run / configuration)

Triggers: "set up session-connector" / "configure storage" / "where should sessions go", or any first invocation where `.claude/sessions/` doesn't exist yet.

1. **Detect available storage** using the platform-appropriate command:
   - Windows: `Get-Volume | Select-Object DriveLetter, FileSystemLabel, DriveType, @{Name='FreeGB';Expression={[math]::Round($_.SizeRemaining/1GB,1)}} | Format-Table` (via PowerShell)
   - macOS: `diskutil list external` and `df -h`
   - Linux: `lsblk -d -o NAME,SIZE,TYPE,MOUNTPOINT,LABEL,RM` (RM=1 is removable / USB)

   Show the user the result, distinguishing internal drives from USB/external/removable.

2. **Offer numbered choices:**
   - **(default)** Repo-local: `<repo-root>/.claude/sessions/`
   - **(if external / USB drives detected)** Each one as: `<drive>/session-connector/<project-name>/`. The top-level `session-connector/` folder on the drive is **mandatory** — it isolates all session-connector data from any other files on the drive, makes backup and cleanup predictable ("delete just that folder"), and gives the project-isolation safety rule a clear scope. Derive `<project-name>` from the repo's basename.
   - **(custom)** Let the user type any absolute path. If the path lands on an external / removable drive but is not under `<drive>/session-connector/`, refuse — tell the user the drive must use the `session-connector/` top-level folder convention, and offer to auto-prepend it or have them re-pick. Custom paths on internal drives can use any structure the user prefers.

3. **Wait for the user to choose.** Don't auto-pick.

4. **Create the chosen location** if it doesn't exist, with empty `HEAD.md` and empty `threads/` directory.
   - On external drives, **create the parent `<drive>/session-connector/` folder first** if it doesn't already exist, then the `<project-name>/` subfolder inside it. The top-level `session-connector/` folder is the scope-marker for the convention — its presence on a drive declares "session-connector data lives here, and only here."
   - **If the chosen path already contains files**, stop. Show the user what's there. Ask whether to (a) use the existing data as-is (e.g., they're re-installing on a new machine and pointing at an existing session archive), (b) pick a different path, or (c) abort. Never silently merge, never overwrite.
   - If the path resolves to `<drive>/session-connector/<project-name>/` and `<project-name>` collides with a different existing project on that drive, refuse to proceed and ask the user to pick a unique name. Don't risk cross-project contamination.

5. **Write the config** to `<repo-root>/.claude/sessions/.config.json`:
   ```json
   {
     "storage_path": "<absolute path the user chose>",
     "drive_label": "<volume label if external, else null>",
     "configured_at": "YYYY-MM-DD"
   }
   ```
   The repo-side `.claude/sessions/` always exists (just to hold this config); actual session data lives at `storage_path`.

6. **Suggest gitignore once** if `.claude/` isn't already ignored. Don't edit it for them.

7. **Confirm** in 1-2 lines: storage path + a note about what to do next ("ready to use; invoke 'save session' or 'resume' as normal").

If a USB drive is selected, also note: "Sessions will live on `<drive_label>`. session-connector will check the drive is plugged in before save/resume operations."

### save (end-of-session)

0. **Resolve the sessions root** per "Storage path resolution" above. If the configured path is unreachable, stop and warn the user — don't write to a fallback without explicit consent.

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

0. **Resolve the sessions root** per "Storage path resolution" above. If the configured path is unreachable, tell the user and offer to re-run setup or work from auto-memory alone.

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

## Safety rules (don't violate these)

These prevent catastrophic data loss. They override every other instruction in this skill.

- **Never auto-delete, auto-clean, or auto-prune any session-connector files** — not session folders, not thread files, not the config, not the storage directory. A USB drive may hold years of irreplaceable research state. Status changes (`active` → `dormant` → `resolved`) are **metadata only**; the underlying files always stay on disk.

- **Treat any "clean up" / "tidy" / "remove old" request as a surgical operation, not a sweep.** List what's there, show the user, delete one item at a time after confirmation. Never use `rm -rf`, `Remove-Item -Recurse`, glob deletions, or anything similar within session-connector storage. Default answer: "I'll list what's there and you tell me what to delete."

- **Stay strictly within the configured project storage folder.** The `storage_path` from `.config.json` already namespaces by project (e.g., `D:/session-connector/<project-name>/`). Never read, write, list, or delete anything above that path. If two projects share the same USB drive, they must remain isolated by their subfolders.

- **On external / removable drives, all session-connector data must live under `<drive>/session-connector/`.** This is non-negotiable: the top-level `session-connector/` folder on the drive is the scope-marker for everything this skill does. Setup enforces it at configuration time; never accept or auto-create a path on an external drive that bypasses it.

- **Never overwrite existing data during setup.** If the chosen storage location already contains files, stop and ask the user whether to (a) use the existing data as-is, (b) pick a different path, or (c) abort. Never silently merge or overwrite.

## Other anti-patterns

- **Don't dump conversation into thread HEADs.** A HEAD is a current-state summary, rewritten each session. Chronology is for one-liners, not transcripts.
- **Don't auto-resolve threads.** Only the user knows when an investigation is closed.
- **Don't create a thread for every casual mention.** Threads are for arcs that span (or will plausibly span) multiple sessions.
- **Don't read every thread on resume.** Surface HEAD one-liners only; read full files when the user picks one to dig into.
- **Don't auto-commit, push, or stage** any session-connector files.
- **Don't duplicate auto-memory content.** Reference by filename in "Connects to".
- **Don't snapshot mid-flow.** "Save this file" while editing code is not a session boundary.
