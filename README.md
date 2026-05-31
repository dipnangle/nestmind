# NestMind

> Multi-agent shared memory system. Pure markdown. Pure filesystem. No daemon. No server.

**Two files to open. Ten files to work. Zero files forgotten.**

NestMind gives any number of AI agents a shared brain on the same repo — without token bloat, duplicate discovery, or write conflicts. It works with Claude Code, Cline, Cursor, Aider, Codex, or any agent that can read and write files.

---

## Why NestMind

When you run multiple agents on the same project, three problems emerge:

| Problem | Without NestMind | With NestMind |
|---|---|---|
| Token cost | Agent reads 10+ files every session | Agent reads 2 files at open, rest on-demand |
| Duplicate work | Agent A solves a problem Agent B already solved | Registry + cross-agent memory discovery |
| Write conflicts | Two agents edit the same file | Private agent lanes + append-only shared files |

---

## How It Works

```
Session Open (always 2 files)
    ↓
shared/index/digest.md       ← full project picture, one file
agents/<id>/sessions/handoff.md  ← your own last state
    ↓
Task Start (on-demand)
    ↓
work-map.md                  ← check who owns what
registry.md                  ← find relevant memories by tag
    ↓
Do Work → Encode Memories
    ↓
Session End
    ↓
Write handoff, encode memories, release locks, log events
```

---

## Folder Structure

```
memory/<project_id>/
│
├── shared/
│   ├── core/            → project-wide facts (brief, tech, patterns, context, progress)
│   ├── specs/           → guardrails, preferences, style, workflow, communication
│   └── index/
│       ├── digest.md    ← THE entry point — read first every session
│       ├── registry.md  ← master memory index, append-only
│       └── work-map.md  ← live agent ownership, append-only
│
├── agents/
│   └── <agent_id>/
│       ├── active/      → working memories
│       ├── sessions/    → handoff.md + session logs
│       └── propose/     → memories queued for promotion
│
└── sync/
    ├── events.log       → append-only audit trail
    └── locks/           → TTL-based lock files
```

---

## Getting Started

### 1. Add NestMind to your project

Copy `NESTMIND.md` to your project root.

### 2. Bootstrap the memory folder

Create `memory/<your-project-name>/` and the full structure from Section 4 of `NESTMIND.md`.

Fill in `shared/index/digest.md` first — this is what every agent reads at session open.

### 3. Add to your agent instructions

**Claude Code (`CLAUDE.md`):**
```markdown
## Memory Bank (NestMind)

Every session starts with exactly 2 files:
  1. Read shared/index/digest.md
  2. Read agents/<my_id>/sessions/handoff.md

When starting a task:
  Read work-map.md, check locks, append claim row before touching any module.

At session end:
  Update handoff.md, encode memories, update registry.md (lock first),
  release work-map claims, append to events.log, delete held locks.

Token budget: 2 files at open. Max 10 total per session.
```

**Cline / Cursor / Aider (`.clinerules`):**
```
Every session — read these 2 files first:
  memory/<project>/shared/index/digest.md
  memory/<project>/agents/<my_id>/sessions/handoff.md

When starting a task: read work-map.md, check locks, append your claim.
At session end: update handoff.md, encode memories, release claims.
```

### 4. Run your agents

Each agent picks an ID (e.g. `agent-1`, `claude-code`, `cline`), creates its own lane under `agents/<id>/`, and follows the session lifecycle in `NESTMIND.md` Section 10.

---

## Key Rules

- **digest.md is always first** — agents never skip this, never write to it directly
- **Agents write only to their own lane** — `agents/<my_id>/` is private
- **work-map.md and events.log are append-only** — no locking needed
- **registry.md needs a lock** — 60-second TTL, any agent can reclaim a stale lock
- **Specs override everything** — `shared/specs/guardrails.md` wins all conflicts
- **Human promotes to shared/** — agents propose, humans approve, quality stays high

---

## Token Budget

| Phase | Files loaded |
|---|---|
| Session open | 2 (digest + handoff) |
| Task start | +1 (work-map) |
| Memory retrieval | +3 max (by tag match + score) |
| Core/spec reads | +2 max (only if task needs them) |
| **Total cap** | **~10 files per session** |

---

## Compatibility

NestMind is plain markdown. No dependencies, no installation, no server.

Works with: Claude Code · Cline · Cursor · Aider · OpenCode · KimiCode · Gemini CLI · Codex · any file-capable agent.

---

## Full Documentation

See [`NESTMIND.md`](./NESTMIND.md) for the complete spec:

- Section 4 — Full folder structure to bootstrap
- Section 5 — Digest file format
- Section 10 — Session lifecycle (open, work, close)
- Section 11 — Memory file format
- Section 12 — Lock protocol
- Section 13 — Cross-agent memory discovery
- Section 14 — Diffusion scoring
- Section 17 — Consolidation schedule
- Section 21 — Guardrails

---

*Encode what is novel. Share what recurs. Decay what is stale. Evolve what matures.*
*Every session makes the next session smarter. Every agent makes every other agent smarter.*
