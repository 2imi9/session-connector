# session-connector

A [Claude Code](https://claude.com/claude-code) skill that bridges sessions for the same project, designed around the KV-cache mental model.

## The idea

Long-running projects involve **investigation arcs** that span many sessions. A simple "snapshot the state at end-of-session, restore it at start-of-next" misses what makes those arcs coherent: the chain of reasoning, the open questions that matured over time, the pivots and dead ends.

`session-connector` treats sessions the way a transformer treats tokens: each new session **attends to a cache of accumulated thread state** rather than re-deriving everything. The cache is structured into **threads** — first-class records of each investigation arc, with a HEAD (current state), chronology (the chain), and connections (to other threads, files, papers, memory entries).

## The KV-cache analogy

| KV cache (transformer) | session-connector |
|---|---|
| Each new token attends to all prior K/V pairs | Each new session attends to active thread HEADs |
| O(1) marginal cost per token | O(1) marginal cost per session (only touched threads update) |
| Multi-head attention runs heads in parallel | Multiple investigation threads run in parallel |
| Layers at different abstractions | L0 = session log; L1 = thread state; L2 = project memory |
| Cache eviction / compression | Threads transition active → dormant → resolved |

Three layers, in order from most concrete to most durable:

- **L0 — Session log** (per-session, time-stamped). Concrete narrative of "what we did today."
- **L1 — Thread state** (per investigation arc). Has a HEAD that is rewritten in full each session, and a chronology of one-liners that grows.
- **L2 — Project memory**. Already handled by Claude Code's auto-memory. Threads *reference* memory entries; they don't duplicate them.

## Layout produced

```
.claude/sessions/
├── HEAD.md                          # last session + active thread list
├── threads/
│   ├── <thread-name>.md             # one file per investigation arc
│   └── ...
└── <YYYY-MM-DD_HHMM>/               # per-session folders
    ├── state.md
    ├── next.md
    ├── advanced_threads.md          # which threads this session moved
    ├── links.md
    └── git_snapshot.txt
```

## How to use

Invoke by phrase, not by name. The skill listens for natural session-boundary signals:

- **End of session**: "save session" / "see you tomorrow" / "wrap up" / "before I sleep"
- **Start of session**: "where were we" / "continue from last session" / "resume"
- **Anytime**: "what's on my plate" / "show active threads" / "what investigations are open"

## Install

### Claude Code (user-level skill)

```bash
git clone https://github.com/<user>/session-connector ~/.claude/skills/session-connector
```

The skill is auto-discovered next time Claude Code starts a session.

### Claude.ai

Upload the `SKILL.md` to your project's skills, or paste its contents into a custom instruction.

## Why this beats simple snapshot/resume

- **Resume is fast**: read 1 file (`HEAD.md`) instead of N (one per past session).
- **Multi-session arcs are first-class**: the chronology in a thread file is the actual chain of reasoning, preserved.
- **Parallel investigations don't get tangled**: each thread maintains its own HEAD.
- **Promotion to memory is explicit**: when a thread reaches a durable conclusion, you call `consolidate-memory` and the thread is marked resolved. The split between in-flight and settled stays clean.

## License

MIT. See [LICENSE](LICENSE).
