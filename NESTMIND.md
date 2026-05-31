# NestMind
> Multi-agent shared memory system. Pure markdown. Pure filesystem. No daemon. No server.
> Built for parallel agents on the same repo — token-efficient by design.

**Core guarantee:** Any number of agents can work simultaneously on the same project without token waste, duplicate discovery, or write conflicts — by design, not by luck.

**Token guarantee:** Every session starts with exactly 2 files. Everything else is on-demand.

---

## The Self-Bootstrap Rule

```
Before ANY work:
  1. Does memory/<project_id>/ exist?
     NO  → create it using Section 4 (Full Structure)
     YES → follow Session Start in Section 9
  2. Read shared/index/digest.md FIRST — one file, full project picture
  3. Read agents/<my_id>/sessions/handoff.md — reconstruct your own last state
  4. Only THEN, when starting a task → read work-map.md to check locks
  5. Proceed with work
```

This is non-negotiable. Every agent. Every session. No exceptions.

---

## 1. PROJECT ISOLATION

Memory is strictly per project. Never mix across projects.

```
memory/<project_id>/
```

Determine `project_id` from:
1. Git repository name (preferred)
2. Workspace or folder name
3. Fallback: `global`

---

## 2. DESIGN PRINCIPLES

| Principle | Why |
|---|---|
| Digest-first session start | Saves tokens — one file gives full project picture |
| On-demand deeper reads | Agent only loads what the current task actually needs |
| Shared zone is read-only for agents | Eliminates write conflicts on critical files |
| work-map.md is append-only | Rows never conflict — no locking needed |
| Agents write only to their own lane | Zero cross-agent interference during work |
| Promotion requires human review | Shared memory stays high quality |
| Registry is the discovery mechanism | Agents find each other's solutions without pre-coordination |
| TTL-based locks, no daemon | Crashed agents auto-release, no manual cleanup |

---

## 3. THE THREE ZONES

```
memory/<project_id>/
  shared/      → read by ALL agents, written by human or consolidation only
  agents/      → per-agent private lanes, readable cross-agent via registry
  sync/        → coordination primitives (locks, event log)
```

### Zone authority

| Zone | Who reads | Who writes | When |
|---|---|---|---|
| `shared/core/` | All agents | Human / consolidation | Project-wide facts only |
| `shared/specs/` | All agents | Human only | Preferences and guardrails |
| `shared/index/` | All agents | Agents (registry, via lock) + Human | Discovery + coordination |
| `agents/<id>/` | Any agent (via registry path) | Owner agent only | Working memory + proposals |
| `sync/` | All agents | All agents | Lock claims + event log |

---

## 4. FULL STRUCTURE

```
memory/<project_id>/
│
├── shared/
│   ├── core/
│   │   ├── project-brief.md
│   │   ├── tech-context.md
│   │   ├── system-patterns.md
│   │   ├── active-context.md
│   │   └── progress.md
│   ├── specs/
│   │   ├── guardrails.md        ← NEVER violate these
│   │   ├── preferences.md
│   │   ├── style.md
│   │   ├── workflow.md
│   │   └── communication.md
│   └── index/
│       ├── digest.md            ← ALWAYS read first — single file project summary
│       ├── registry.md          ← master memory index, append-only rows
│       └── work-map.md          ← live agent ownership, append-only rows
│
├── agents/
│   └── <agent_id>/
│       ├── active/              ← working memories (readable cross-agent via registry path)
│       ├── sessions/
│       │   ├── handoff.md       ← loaded at every session start (second file after digest)
│       │   └── SESSION-YYYY-MM-DD.md
│       └── propose/             ← memories queued for promotion to shared/
│
└── sync/
    ├── events.log               ← append-only audit trail
    └── locks/
        └── <resource>.lock      ← TTL-based claim files
```

---

## 5. THE DIGEST FILE (Read This First, Every Session)

`shared/index/digest.md` is the **single entry point** for every agent at every session start.
It is a human-maintained summary of the entire project state — updated after major milestones or consolidation.

**This file replaces reading all of `shared/core/*` at session start.**
Agents read deeper core files only when the current task specifically requires that detail.

```markdown
---
id: digest-project-snapshot
type: spec
zone: shared-core
updated: YYYY-MM-DD
---

## Project
[One paragraph: what this is, who it's for, what stage it's at]

## Stack
[Language, framework, database, deployment — one line each]

## Current Focus
[What is actively being worked on RIGHT NOW — by whom]

## Recent Changes
[Last 3-5 significant changes across any agent]

## Active Agents
[Which agents have live work-map claims — from work-map.md]

## Open Decisions
[Choices pending that affect current work]

## Known Blockers
[Cannot proceed + who owns unblocking]

## Key Guardrails
[Top 3-5 hard constraints agents must never violate]

## Memory Index Hint
[Tag clusters that are most active — e.g. "auth, db, deploy" — so agent knows where to scan registry]

## Last Digest Update
[YYYY-MM-DD — by whom]
```

### Digest update rule

The digest is updated:
- At the end of any full consolidation (Section 16)
- After any major milestone completes
- When `active-context.md` or `progress.md` changes significantly
- Human may update it at any time

**Agents never write to digest.md directly.** Queue changes via `propose/` if needed.

---

## 6. SHARED CORE FILES

These files exist for deep-read when a task requires them. They are NOT loaded at session start automatically — digest.md covers the summary. Load these only when the task specifically needs that level of detail.

### When to load each core file

| File | Load when |
|---|---|
| `project-brief.md` | Starting a new feature, onboarding, scope questions |
| `tech-context.md` | Dependency questions, stack decisions, setup issues |
| `system-patterns.md` | Architecture decisions, refactors, new module design |
| `active-context.md` | Need full detail on who is doing what right now |
| `progress.md` | Planning, milestone review, blocked task resolution |

### shared/core/project-brief.md

```markdown
---
id: brief-project-overview
type: spec
zone: shared-core
score: 1.00
created: YYYY-MM-DD
---

## TL;DR
[One paragraph: what this project is, what it does, who it is for]

## Goals
[What are we trying to achieve?]

## Scope
[What is in? What is explicitly out?]

## Key Constraints
[Budget, timeline, technical, team size]
```

### shared/core/tech-context.md

```markdown
---
id: spec-tech-context
type: spec
zone: shared-core
score: 1.00
created: YYYY-MM-DD
---

## TL;DR
[One sentence: primary language/framework and deployment target]

## Stack
[Languages, frameworks, databases, infrastructure]

## Dependencies
[Key libraries, external services, APIs]

## Dev Setup
[How to run locally, test, deploy]

## Constraints
[Version pinning, compatibility requirements, platform limits]
```

### shared/core/system-patterns.md

```markdown
---
id: spec-system-patterns
type: spec
zone: shared-core
score: 1.00
created: YYYY-MM-DD
---

## TL;DR
[Primary architectural pattern]

## Architecture
[Components and their relationships]

## Key Patterns
[Design patterns in active use]

## Conventions
[Naming, file structure, commit format]

## Known Anti-Patterns
[Things tried that failed — NEVER repeat]
```

### shared/core/active-context.md

```markdown
---
id: ctx-active-work
type: plan
zone: shared-core
score: 1.00
updated: YYYY-MM-DD
---

## TL;DR
[What is happening RIGHT NOW across the project]

## Current Focus Areas
[What features/tasks are actively in progress — by whom]

## Recent Changes
[What changed in the last few sessions — any agent]

## Open Decisions
[Pending choices that affect current work]

## Known Issues
[Active bugs or blockers]
```

### shared/core/progress.md

```markdown
---
id: ctx-progress-tracker
type: plan
zone: shared-core
score: 1.00
updated: YYYY-MM-DD
---

## TL;DR
[Overall project status in one sentence]

## Completed
[Done features, milestones, releases]

## In Progress
[Being worked on — agent + % or state]

## Not Started
[Planned but not begun]

## Blocked
[Cannot proceed and why — who owns unblocking]
```

---

## 7. SHARED SPECS (Highest Authority)

Specs override all other memory when conflicts arise. Agents never modify these directly.
Load these **on demand** — when a behavior, style, or workflow question arises during work.

| File | Purpose | Priority |
|---|---|---|
| `guardrails.md` | Hard constraints — things to NEVER do | 1 (highest) |
| `preferences.md` | Architecture and technology preferences | 2 |
| `style.md` | Coding style, formatting, naming | 3 |
| `workflow.md` | How the team works — debugging, review, deployment | 4 |
| `communication.md` | Output format, verbosity, tone | 5 (lowest) |

**Conflict rule:** Higher-priority spec always wins. If agent behavior consistently contradicts a spec (3+ corrections of same type), append an Evolution Note to the spec — never silently override it.

---

## 8. INDEX FILES

### shared/index/registry.md

Master discovery index. Every memory in the entire system has a row here.
**Append only. Never edit or delete rows. Mark superseded memories with `[SUPERSEDED]`.**

Before appending, claim the registry lock (Section 11).

```markdown
# Memory Registry

| ID | Type | Zone | Agent | TL;DR | Score | Tags | Last Used |
|---|---|---|---|---|---|---|---|
| problem-auth-jwt-race-a1b2 | problem | active | agent_A | JWT refresh race on concurrent requests | 0.72 | auth,jwt,race | 2025-05-01 |
| pattern-db-conn-pool-x9k2 | pattern | active | agent_B | Connection pool tuning for high concurrency | 0.85 | db,postgres,pool | 2025-05-02 |
```

The `Agent` column contains the agent ID whose `active/` folder holds the full memory file.
Any agent can read any file at path `memory/<project>/agents/<Agent>/active/<ID>.md`.

### shared/index/work-map.md

Live broadcast of who owns what. **Append only. No locking needed.**
Agents read this **only when starting a task** — not at session open.
Agents append their own claim row before touching any module.

```markdown
# Work Map

| Agent | Module / Path | Claimed At | TTL | Status |
|---|---|---|---|---|
| agent_A | src/apps/contacts/ | 2025-05-02T10:30 | 4h | active |
| agent_B | src/apps/leads/ | 2025-05-02T11:00 | 4h | active |
| agent_A | src/apps/contacts/ | 2025-05-01T09:00 | 4h | expired |
```

**TTL rules:**
- Default TTL: 4 hours. For quick tasks, use 1h. For long refactors, use 8h.
- Expired rows are dead claims — any agent can take that module.
- Never delete rows. Add a new row with `status: released` when done early.
- At session end, append a `released` row for every module you claimed.

---

## 9. PER-AGENT FILES

### agents/<agent_id>/sessions/handoff.md

Loaded at every session start — immediately after digest.md. Written at every session end.

```markdown
# Handoff — <agent_id>

## Last Session
- Date: YYYY-MM-DD
- Duration: Xh Ym

## What Was Done
[Completed items this session]

## In Progress
[Incomplete work + exact state — enough for a fresh agent to resume]

## Blocked
[What cannot proceed + who owns unblocking]

## Next Steps
[Prioritized for next session]

## Open Questions
[Needs human input before proceeding]

## Environment State
[Uncommitted changes, running processes, feature flags, test state]

## Memory Commits
[Evolution events: what was encoded, promoted, or archived this session]

## Modules Claimed
[Which entries in work-map.md I added — to clean up at next session start]
```

### agents/<agent_id>/active/<memory-id>.md

Full memory file. Follows the standard memory format (Section 10).
Other agents may read these files — their paths are discoverable via registry.md.

### agents/<agent_id>/propose/<memory-id>.md

Memories queued for promotion to shared/. Same format as active memories.
Human reviews propose/ periodically and moves approved files to shared/core/ or shared/index/.
Stale proposals (> 7 days unreviewed) get archived or discarded.

---

## 10. SESSION LIFECYCLE

### Session Start — Exactly 2 Files Guaranteed

```
Step 1 — Read shared/index/digest.md
  → Full project picture in one file
  → Know current focus, recent changes, active agents, key guardrails
  → Note the memory index hints for later tag scanning

Step 2 — Read agents/<my_id>/sessions/handoff.md
  → Reconstruct my own last state
  → Know what I was doing, what's blocked, what's next

─── Session open complete. Token cost: 2 files. ───

Step 3 — When starting a task → read shared/index/work-map.md
  → Check who owns what RIGHT NOW
  → Do not claim or touch modules with live TTL
  → Note expired claims — those modules are available

Step 4 — Append claim row to work-map.md
  → Format: | <agent_id> | <module_path> | <ISO timestamp> | <TTL> | active |

Step 5 — Load targeted memories only if task needs them
  → Scan shared/index/registry.md for tag matches
  → Max 3 files from agents/<any_id>/active/ (highest score + tag match)
  → Max 2 passive zone memories (only if explicitly needed)
  → NEVER load archive unless human explicitly requests

Step 6 — Load core/spec files only if task needs them
  → See Section 6 for when to load each core file
  → Load specs only when a behavior/style/workflow question arises

Step 7 — Begin work
```

**Hard token budget: 2 files at open. Max ~8 additional files during work. Never exceed 10 total.**

### During Work — What to Encode

Flag moments for memory encoding immediately, not at session end:

| Situation | Encode as | Threshold |
|---|---|---|
| "This took > 15 min to figure out" | `problem` + `solution` | Always |
| "We chose X over Y because Z" | `design` | Always |
| "This broke, here is the fix" | `solution` | Always |
| "I keep seeing this same issue" | Strengthen existing `pattern` | 3+ occurrences |
| "That approach made things worse" | `anti` | Always |
| "Another agent already solved this" | `reinforce` their memory (bump score in registry) | When reused |
| "User always wants it done this way" | Propose spec update | Always |

### Session End — Always Complete This

```
Step 1 — Write session log
  → agents/<my_id>/sessions/SESSION-YYYY-MM-DD.md

Step 2 — Update handoff.md
  → Precise enough that a different agent could resume this work

Step 3 — Encode memories from this session
  → Write to agents/<my_id>/active/
  → Append rows to shared/index/registry.md (claim lock first)
  → Queue high-value candidates in agents/<my_id>/propose/

Step 4 — Run evolution loop on accessed memories
  → Update frequency, last_used, recalculate score
  → Check zone thresholds — migrate if needed

Step 5 — Release work-map claims
  → Append released rows for every module claimed this session

Step 6 — Append to sync/events.log
  → One line per significant event (encode, promote, release, conflict)

Step 7 — Release any held locks
  → Delete sync/locks/<resource>.lock if still held

Step 8 — If significant project state changed → flag digest.md for human update
  → Append note to handoff.md: "digest.md needs update — reason: X"
```

---

## 11. MEMORY FILE FORMAT

Every memory uses this structure:

```markdown
---
id: <type>-<context>-<descriptor>-<hash>
type: problem|pattern|solution|anti|design|plan|spec|context
zone: active|passive|archive
agent: <agent_id who created this>
tags: [tag1, tag2, tag3]
triggers: [symptom, error-message, scenario-keyword]
confidence: high|medium|low
frequency: <integer — access count across ALL agents>
last_used: YYYY-MM-DD
created: YYYY-MM-DD
score: <float 0.00-1.00>
related: [memory-id-1, memory-id-2]
supersedes: <memory-id if this replaces another>
failure_count: <integer>
cross_agent_uses: <integer — how many other agents have used this>
---

## TL;DR
[1-2 sentences — what this memory IS and when to use it]
[This is what appears in registry.md — make it scannable]

## Context
[What was happening when this was learned]

## Content
[The actual knowledge — steps, rationale, pattern, code, commands]

## Evidence
[Code snippets, error messages, logs, links]

## Evolution Notes
[Append-only audit trail — never overwrite]
[YYYY-MM-DD] learn(this-id): initial encoding by agent_A — score: 0.60
[YYYY-MM-DD] reinforce(this-id): reused by agent_B, worked correctly — score: 0.60 → 0.72
```

### Memory ID Format

`<type>-<context>-<descriptor>-<hash>`

| Component | Values |
|---|---|
| `type` | `problem` · `pattern` · `solution` · `anti` · `design` · `plan` · `spec` · `context` |
| `context` | Module, feature, or domain (e.g. `auth`, `leads`, `db`, `deploy`) |
| `descriptor` | Short kebab-case summary |
| `hash` | 4-char suffix for uniqueness |

**IDs are immutable after creation. Never rename a memory file.**

### Three-Layer Encoding Rule

Every memory must be readable at three depths:
1. **TL;DR** — scannable from registry.md in one pass
2. **Context + Content** — enough to act without reading Evidence
3. **Evidence** — raw proof, code, logs for verification

---

## 12. LOCK PROTOCOL

Only one shared resource requires locking: `shared/index/registry.md`.
Everything else is either append-only (work-map, events.log) or agent-private.

### Lock File Format

```
sync/locks/registry.lock
---
agent_id: agent_A
claimed_at: 2025-05-02T10:31:45
ttl_seconds: 60
task: appending memory problem-auth-import-x4k2
```

### Lock Procedure

```
BEFORE writing to registry.md:

1. Check: does sync/locks/registry.lock exist?
   NO  → proceed to step 2
   YES → read claimed_at + ttl_seconds
         if (now - claimed_at) > ttl_seconds → lock is stale, delete it, proceed
         else → wait 5 seconds, retry (max 6 retries = 30 seconds total)
               if still locked after 30s → log conflict to events.log, skip this write,
               add to propose/ instead and note in handoff.md

2. Write sync/locks/registry.lock with your agent_id, timestamp, TTL=60s, task description

3. Perform registry.md append

4. Delete sync/locks/registry.lock

AFTER session end (cleanup):
  Delete any lock file you hold. If you crashed mid-task, the TTL handles it automatically.
```

### Stale Lock Rule

A lock is stale if `(current_time - claimed_at) > ttl_seconds`.
Any agent may delete a stale lock and reclaim it. Log the deletion in events.log.
This handles crashed agents automatically — no human intervention needed.

---

## 13. CROSS-AGENT MEMORY DISCOVERY

This is how agents avoid duplicating each other's work.

### Finding memories from other agents

```
1. Scan shared/index/registry.md for rows where tags overlap with current task
2. Sort by: score DESC, cross_agent_uses DESC, last_used DESC
3. Read full memory file at: memory/<project>/agents/<Agent>/active/<ID>.md
4. If memory helps → increment frequency in registry row + append reinforce event
5. If memory misleads → increment failure_count in registry row + append fail event
```

### When to propose a memory to shared/

A memory in `agents/<id>/active/` should be queued to `propose/` when:
- `cross_agent_uses >= 3` AND score >= 0.75
- A pattern has spawned from it (problem → pattern, problem → solution)
- It directly relates to shared/core/ content (architecture, stack, anti-patterns)

Human reviews `propose/` and promotes to `shared/core/` or archives. Proposal files are not auto-promoted — quality gate is intentional.

---

## 14. DIFFUSION SCORING

Every memory has a score that determines its zone and retrieval priority.

### Score Formula

```
Score = (confidence      × 0.25)
      + (frequency       × 0.20)
      + (recency         × 0.15)
      + (context_match   × 0.20)
      + (spec_alignment  × 0.15)
      - (failure_penalty × 0.25)
      + (cross_agent_bonus × 0.10)
```

| Factor | Range | Calculation |
|---|---|---|
| `confidence` | 0.0–1.0 | high=1.0, medium=0.7, low=0.4 |
| `frequency` | 0.0–1.0 | `min(access_count / 10, 1.0)` |
| `recency` | 0.0–1.0 | `0.95 ^ months_since_last_access` |
| `context_match` | 0.0–1.0 | Tag overlap with current task |
| `spec_alignment` | 0.0–1.0 | Alignment with shared/specs/ |
| `failure_penalty` | 0.0–1.0 | `failure_count / total_uses` |
| `cross_agent_bonus` | 0.0–1.0 | `min(cross_agent_uses / 5, 1.0)` |

### Zone Thresholds

| Zone | Score | Load behavior |
|---|---|---|
| Core (shared) | >= 0.80 | Always available — not auto-loaded, but digest summarizes |
| Active | >= 0.60 | Loaded on tag match during task |
| Passive | >= 0.30 | Loaded on direct query only |
| Archive | < 0.30 | Never auto-loaded |

---

## 15. ZONE MIGRATION

### Promotion

```
passive → active:    score >= 0.60 AND at least 1 recent successful use
active  → propose/:  score >= 0.80 AND cross_agent_uses >= 3
propose/→ shared:    human review approval
```

### Demotion

```
active  → passive:   score < 0.60 OR unused for 90 days
passive → archive:   score < 0.30 AND unused for 180 days
```

### Lateral Evolution (Memory Spawning)

When a memory matures, it spawns new linked memories:

```
problem   → pattern    (when 3+ problems share root cause)
problem   → solution   (when resolution becomes repeatable)
problem   → anti       (when a "fix" made things worse)
pattern   → propose/   (when pattern is cross-agent and high-score)
```

Original memory is preserved. New memory cross-links via `related` field.

---

## 16. TOKEN OVERLOAD — ELIMINATION ORDER

When the system grows too large, eliminate in this strict order:

| Priority | What to eliminate | Rule |
|---|---|---|
| 1st | `agents/<id>/active/` score < 0.40 | Demote to passive or archive |
| 2nd | `sessions/SESSION-*.md` older than 14 days | Archive or delete (handoff.md is enough) |
| 3rd | `registry.md` rows with score < 0.30 | Mark `[ARCHIVED]`, move file to archive zone |
| 4th | `agents/<id>/propose/` older than 7 days | Promote or discard — do not let queue rot |
| 5th | `agents/<id>/passive/` score < 0.20 AND age > 180 days | Archive |
| Last resort | Never drop `shared/core/` or `shared/specs/` or `digest.md` | Losing these costs more tokens to reconstruct |

**Hard token budget: 2 files at session open. Max ~8 additional during work. Total cap: 10.**

---

## 17. CONSOLIDATION

### When to consolidate

| Cadence | Trigger |
|---|---|
| Weekly (light) | File count > 30 OR session consistently reading > 8 files during work |
| Monthly (full) | After major milestone OR file count > 80 |
| On demand | Human requests it OR cross-agent conflicts detected |

### Light Consolidation

1. Archive sessions older than 14 days
2. Recalculate all diffusion scores
3. Migrate memories across zones if thresholds crossed
4. Clean up expired work-map rows (add summary row: `[COMPACTED] N rows archived YYYY-MM-DD`)
5. Update registry.md scores in-place (this requires the registry lock)
6. **Update digest.md** to reflect current project state

### Full Consolidation

1. Everything in light, plus:
2. Merge duplicate patterns (> 70% tag overlap + same root cause → one memory, cross-link)
3. Check problem clusters → spawn patterns (lateral diffusion)
4. Review propose/ queue — promote or discard
5. Verify cross-links are bidirectional in related fields
6. Flag specs that contradict recent successful patterns (append Evolution Note, do not edit)
7. Generate `shared/index/digest-YYYY-MM.md` — archived snapshot of this month's digest

---

## 18. EVENTS LOG FORMAT

`sync/events.log` — append only, never edit or delete.

```
[YYYY-MM-DD HH:MM] learn(problem-auth-import-x4k2) agent_A: encoded new problem — score: 0.60
[YYYY-MM-DD HH:MM] reinforce(pattern-db-pool-x9k2) agent_B: reused successfully — score: 0.72 → 0.78
[YYYY-MM-DD HH:MM] fail(solution-cache-invalidate-z1p9) agent_A: led to wrong outcome — failure_count: 0 → 1
[YYYY-MM-DD HH:MM] promote(pattern-db-pool-x9k2) agent_B: queued to propose/ — score: 0.82
[YYYY-MM-DD HH:MM] lock_stale(registry.lock) agent_B: stale lock deleted, claimed_at was 10m ago
[YYYY-MM-DD HH:MM] claim(src/apps/leads/) agent_B: TTL 4h
[YYYY-MM-DD HH:MM] release(src/apps/leads/) agent_B: session end
[YYYY-MM-DD HH:MM] conflict(registry.lock) agent_C: lock held by agent_A, waited 30s, queued to propose/
[YYYY-MM-DD HH:MM] digest_stale() agent_A: flagged digest.md for human update — reason: major milestone complete
```

---

## 19. CONFLICT RESOLUTION

| Situation | Action |
|---|---|
| Two agents want same module | Earlier work-map claim with live TTL wins. Second agent picks a different module or waits. |
| Registry lock held > TTL | Stale — delete and reclaim. Log in events.log. |
| Registry lock held < TTL | Wait up to 30s. If still held, write memory to propose/ and note in handoff.md. |
| Two memories contradict each other | Higher score wins. Log conflict in events.log. If equal score, prefer newer. |
| Memory contradicts a spec | Spec always wins. Demote the memory. Append Evolution Note to spec if pattern is emerging. |
| Memory led to wrong outcome | Increment failure_count. Recalculate score. Log fail event. |
| propose/ has conflicting candidates | Human resolves during review. Do not auto-merge. |
| digest.md is stale | Flag in handoff.md. Human updates. Agents never write to digest directly. |

---

## 20. AGENT INTEGRATION

### Claude Code — add to CLAUDE.md

```markdown
## Memory Bank (NestMind)

Every session starts with exactly 2 files — no exceptions:
  1. Read shared/index/digest.md — full project picture
  2. Read agents/<my_id>/sessions/handoff.md — my own last state

When starting a task:
  3. Read shared/index/work-map.md — check who owns what
  4. Append claim row before touching any module
  5. Scan registry.md for tag-matched memories if needed (max 3 files)
  6. Load core/spec files only if task specifically requires them

At session end (always):
  1. Write session log to agents/<my_id>/sessions/SESSION-YYYY-MM-DD.md
  2. Update agents/<my_id>/sessions/handoff.md
  3. Encode new memories to agents/<my_id>/active/
  4. Append rows to shared/index/registry.md (claim lock first)
  5. Queue high-value memories to agents/<my_id>/propose/
  6. Append released rows to work-map.md for every module claimed
  7. Append session events to sync/events.log
  8. Delete any held lock files
  9. If project state changed significantly → flag digest.md for human update in handoff.md

Token budget: 2 files at open. Max 10 total per session. Enforce strictly.
Project isolation: NEVER read memory from a different project_id.
```

### Cline / Cursor / Aider — add to .clinerules or custom instructions

```
Every session — read these 2 files first, always:
  1. memory/<project>/shared/index/digest.md
  2. memory/<project>/agents/<my_id>/sessions/handoff.md

When starting a task:
  Read work-map.md, check locks, append your claim before touching any module.

At session end:
  Write session log. Update handoff.md.
  Encode memories. Update registry.md (lock first). Release work-map claims.
  Flag digest.md for human update if project state changed.
```

### OpenCode / KimiCode / Gemini CLI / Codex

Same instructions — NestMind is plain markdown. Any agent that reads and writes files works.

---

## 21. GUARDRAILS (Non-Negotiable)

- **DO NOT** skip the 2-file session open — digest.md + handoff.md, always, no exceptions
- **DO NOT** write to digest.md directly — flag for human update via handoff.md
- **DO NOT** hallucinate memory — if you do not have it, say so and create a new one from resolution
- **DO NOT** ignore specs — shared/specs/ overrides everything including your own active memories
- **DO NOT** write to shared/core/ or shared/specs/ directly — use propose/ queue
- **DO NOT** promote untested memory — score must cross threshold AND show recent success
- **DO NOT** mix project memory — isolation is absolute, project_id is the hard boundary
- **DO NOT** store secrets — never inline passwords, API keys, tokens, or credentials
- **DO NOT** skip session end — encode, release, log, every session, no exceptions
- **DO NOT** modify memory IDs — immutable after creation
- **DO NOT** overwrite Evolution Notes — append only, audit trail is sacred
- **DO NOT** claim a module another agent owns with a live TTL — read work-map.md first
- **DO NOT** hold registry lock longer than 60 seconds — write fast, release immediately
- **DO NOT** exceed 10 files total per session — enforce the token budget

---

## Quick Reference

### Session open — always exactly 2 files

```
shared/index/digest.md              ← read this first, every session
agents/<my_id>/sessions/handoff.md  ← read this second, every session
─────────────────────────────────────────────────────────────────────
Token cost at open: 2 files. Everything else is on-demand.
```

### On-demand reads — only when task needs them

```
shared/index/work-map.md            ← when starting a task (lock check)
shared/index/registry.md            ← when scanning for relevant memories
agents/<any_id>/active/<id>.md      ← max 3, by tag match + score
shared/core/<file>.md               ← only when task specifically requires it
shared/specs/<file>.md              ← only when behavior/style question arises
─────────────────────────────────────────────────────────────────────
Max total: ~10 files per session
```

### Session end — always write these

```
agents/<my_id>/sessions/SESSION-YYYY-MM-DD.md   (new)
agents/<my_id>/sessions/handoff.md               (update)
agents/<my_id>/active/<memory-id>.md             (new memories)
shared/index/registry.md                         (append rows, lock first)
agents/<my_id>/propose/<memory-id>.md            (if promotion candidate)
shared/index/work-map.md                         (release rows)
sync/events.log                                  (append events)
```

### Encode triggers

```
> 15 min to solve              → problem + solution
Design decision made           → design
Something broke + fixed        → solution
Same issue 3rd time            → pattern
Approach made things worse     → anti
User corrected same thing 3×   → propose spec update
Another agent's memory reused  → reinforce (update registry score)
Major milestone complete        → flag digest.md for human update
```

---

*NestMind — encode what is novel, share what recurs, decay what is stale, evolve what matures.*
*Every session makes the next session smarter. Every agent makes every other agent smarter.*
*Two files to open. Ten files to work. Zero files forgotten.*
