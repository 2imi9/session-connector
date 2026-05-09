# session-connector

A [Claude Code](https://claude.com/claude-code) skill that bridges sessions for iterative research and notebook-driven work.

## Who this is for

Researchers, data scientists, and engineers doing **multi-session work**: long-running notebook iteration, experiments that span days or weeks, investigations with multiple parallel threads.

If you've ever opened a new session and spent the first 20 minutes re-reading your last three findings docs and grepping `git log` to figure out what you were doing yesterday — this skill is for you.

## What it does

Treats each session as a contributor to long-running **investigation threads**, not an isolated time-snapshot. Three operations:

- **save** — at end of session: snapshot what advanced, update each thread's HEAD, write a per-session folder.
- **resume** — at start of session: surface the active threads (not just the latest session), so you see your full plate of in-flight work.
- **status** — anytime: quick survey of active threads.

You invoke it by phrase, not by name. "save session" / "see you tomorrow" / "where were we" trigger the right mode automatically.

## Inspiration: the KV-cache analogy

The design borrows a mental model from transformer KV caches. In an LLM, each new token attends to a cache of all prior K/V pairs rather than recomputing attention from scratch. `session-connector` applies the same shape to sessions: each new session attends to the accumulated state of active investigation threads rather than re-deriving them.

| KV cache (transformer) | session-connector |
|---|---|
| Each new token attends to prior K/V pairs | Each new session attends to active thread HEADs |
| O(1) marginal cost per token | O(1) marginal cost per session |
| Multi-head attention runs heads in parallel | Multiple investigation threads run in parallel |
| Layers at different abstractions | session log → thread state → project memory |
| Cache eviction / compression | Threads transition active → dormant → resolved |

It's an analogy, not an architecture — but it frames why "snapshot/resume" alone isn't enough.

## Layout produced

```
.claude/sessions/
├── HEAD.md                        # last session + active thread list
├── threads/
│   └── <thread-name>.md           # one file per investigation arc
└── <YYYY-MM-DD_HHMM>/             # per-session folders
    ├── state.md                   # narrative — what we did, where we are, files in flight
    ├── next.md                    # numbered next-actions, lead with most likely
    ├── advanced_threads.md        # which threads this session moved
    ├── links.md                   # PRs, files, memory pointers, external refs
    └── git_snapshot.txt           # raw git/gh output, unprocessed
```

Three layers, in order from most concrete to most durable:

- **Session log** (per-session, time-stamped). Concrete narrative of "what we did today."
- **Thread state** (per investigation arc). Has a HEAD that's rewritten in full each session, plus a chronology of one-liners that grows.
- **Project memory** (already handled by Claude Code's auto-memory). Threads *reference* memory entries; they don't duplicate them.

## Install

### Claude Code (user-level skill)

```bash
git clone https://github.com/2imi9/session-connector ~/.claude/skills/session-connector
```

The skill is auto-discovered next time Claude Code starts a session.

### Claude.ai

Upload `SKILL.md` to your project's skills, or paste its contents into a custom instruction.

## Usage

Invoke by phrase, not by name. The skill listens for natural session-boundary signals:

- **End of session**: "save session" / "see you tomorrow" / "wrap up" / "before I sleep"
- **Start of session**: "where were we" / "continue from last session" / "resume"
- **Anytime**: "what's on my plate" / "show active threads" / "what investigations are open"

## Why this beats simple snapshot/resume

- **Resume is fast**: read 1 file (`HEAD.md`) instead of N (one per past session).
- **Multi-session arcs are first-class**: chronology in a thread file is the actual chain of reasoning, preserved.
- **Parallel investigations don't get tangled**: each thread maintains its own HEAD.
- **Promotion to memory is explicit**: when a thread reaches a durable conclusion, you call `consolidate-memory` and mark the thread `resolved`. The split between in-flight and settled stays clean.

## License

MIT. See [LICENSE](LICENSE).
