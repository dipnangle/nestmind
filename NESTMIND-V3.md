# NestMind v3.1
> Multi-agent shared memory for many parallel Claude Code sessions on one repo.
> Pure markdown. Pure filesystem. No daemon. No server. No locks — atomic primitives only.

**Core guarantee:** Any number of sessions work simultaneously on the same repo without token
waste, duplicate work, or write conflicts.

**Token guarantee:** Per-session cost is **constant** — it does not grow as the repo, the agent
count, or the knowledge base grows. Everything that grows (registries, logs, knowledge) is
grep-only, never read whole.

**This file is REFERENCE ONLY.** Agents never read it at runtime — the operating rules live in
`CLAUDE.md`, which Claude Code auto-loads into every session. Read this file only when changing
the memory system itself.

> **Environment assumption.** Every atomicity guarantee here (`mkdir`, `echo >>`) holds on a
> **local POSIX filesystem** and only for small (< 4 KB) appends — true for all rows below. On
> NFS or other networked mounts these weaken; do not put `memory/` on a shared mount without a
> real lock service.

---

## 0. BOOTSTRAP (first session on a fresh repo)

If `memory/digest.md` does not exist, create the core skeleton, then proceed with a normal open.
(Deferred dirs — `boards/`, `commands/`, `decisions/`, `bugs/` — are created only when their
scale feature activates; see Section 10 and the Appendix.)

```bash
mkdir -p memory/{tasks,registry,knowledge,sync/claims}
[ -f memory/digest.md ] || cat > memory/digest.md <<'EOF'
# Digest — <project>
updated: <bootstrap>

## Project
[one paragraph]
## Stack
[one line per major choice]

<!-- BOARD:START -->
## Task Board
| Task | Status | Prio | Owner | Deps | Notes |
|---|---|---|---|---|---|
<!-- BOARD:END -->

## Hot Domains
## Recent Changes
## Blockers / Open Decisions
## Guardrails
EOF
touch memory/work-map.md memory/events.log
```

---

## 1. DESIGN DECISIONS (why it is built this way)

| Decision | Why |
|---|---|
| Rules live in CLAUDE.md (auto-loaded) | Every agent gets the protocol for free; reading a big spec every session defeats the purpose |
| Agent ID from session UUID | Zero setup, collision-free; sessions are ephemeral |
| Continuity in per-TASK files, not per-agent | Session IDs change every session — the work persists, the worker doesn't |
| Atomic `mkdir` claims, no lock files | Check-then-write locks race; `mkdir` is atomic |
| Append-only via single `echo >>` | Small single-command appends are atomic — no lock needed |
| **Golden rule: growing files are grep-only** | Grepping costs ~0 tokens; reading whole grows unbounded |
| Registry sharded per domain | Small greps; agents in different domains never touch the same index |
| Hierarchical claims (`<area>/<module>`) | A owns `orders/checkout` while B owns `orders/invoices` — finer parallelism |
| Claim heartbeat (`touch owner`) | A legit 6h refactor isn't falsely reclaimed under a short TTL |
| Task status in per-task `meta.md`, digest is a *generated cache* | Removes the "every session rewrites digest" bottleneck — status writes are uncontended |
| Reuse counts derived from events.log, not stored | Keeps append-only intact; no row rewrites, no file rewrites per reuse |

---

## 2. STRUCTURE

```
<repo>/
├── CLAUDE.md                          ← the protocol, auto-loaded every session
├── nestmind.md                        ← this file, reference only
└── memory/
    ├── digest.md                      ← WHOLE-READ first: authored header + GENERATED task-board cache (< 60 lines)
    ├── boards/<ws>.md                 ← optional at scale: per-workstream board (< 60 lines each)
    ├── hot.md                         ← optional at scale: best-known solutions per hot domain (< 40 lines)
    ├── tasks/<task>/meta.md           ← structured task status (source of truth; owner writes)
    ├── tasks/<task>/handoff.md        ← WHOLE-READ second: exact resume state per task
    ├── registry/<domain>.md           ← GREP-ONLY: append-only knowledge index shards (auth.md, orders.md…)
    ├── knowledge/<domain>/<id>.md     ← encoded learnings — read max 3, only after a registry grep hit
    ├── decisions/NNNN-*.md            ← optional: architecture decision records (see Appendix)
    ├── bugs/bug-NNN.md                ← optional: recurring-bug records (see Appendix)
    ├── commands/*.md                  ← optional: reusable operational commands (see Appendix)
    ├── work-map.md                    ← GREP-ONLY: append-only claim/release history (advisory)
    ├── events.log                     ← GREP-ONLY: append-only audit + reuse signals
    └── sync/claims/<area>/<module>/   ← live claims — atomic mkdir dirs, owner file inside
```

Special claims under `sync/claims/_meta/`: `_meta/digest` guards digest/board regeneration,
`_meta/cleanup` guards cleanup runs.

**The golden rule:** only `digest.md`, one `boards/<ws>.md`, one task `handoff.md`, and optionally
`hot.md` are ever read whole. Everything else is grep-only or capped (max 3 knowledge files). This
is what keeps session cost constant forever.

---

## 3. AGENT IDENTITY

```
agent_id = "agent-" + first 8 chars of the session's UUID     e.g. agent-dd3583f0
```

Unique per session by construction. Used for claims, work-map rows, and events (audit: who did
what). IDs are ephemeral; continuity lives in per-task files, never per-agent.

---

## 4. DIGEST + TASK BOARD (a generated cache, not a hot resource)

`memory/digest.md` is read first, always. **Hard cap: 60 lines.** Its authored sections (Project,
Stack, Hot Domains, Guardrails…) change rarely, under the `_meta/digest` claim. Its **Task Board**
is a *cache generated from the per-task `meta.md` files* — so routine status updates never touch
this shared file.

```markdown
# Digest — <project>
updated: YYYY-MM-DD HH:MM · board generated: YYYY-MM-DD HH:MM

## Project      [one paragraph: what this is, current stage]
## Stack        [one line per major choice]

<!-- BOARD:START -->
## Task Board
| Task | Status | Prio | Owner | Deps | Notes |
|---|---|---|---|---|---|
| build-auth | in-progress | P1 | agent-a1b2 | api-schema | UI left |
| fix-login-500 | todo | P0 | — | — | urgent |
<!-- BOARD:END -->

## Hot Domains   [e.g. "auth, catalog" — shards/hot.md worth grepping]
## Recent Changes
## Blockers / Open Decisions
## Guardrails    [top 3–5 hard constraints]
```

Status: `todo · in-progress · blocked · done`   Priority: `P0 (critical) · P1 · P2`

**Pick order** (this is the coordination core — it prevents starvation):
1. Your own stale `in-progress` task (owner expired), else
2. The **highest-priority `todo`** whose `deps` are all `done`.

**Board truth:** the cached board is a hint. The **binding** truth is the claim directory + the
chosen task's own `meta.md`/`handoff.md`, which you read anyway. A slightly stale board can never
cause a collision. Dependencies must form no cycles.

**At scale (board > ~15 rows):** digest becomes a *router* — the board is replaced by a workstream
list pointing to `boards/<ws>.md` (each also generated, < 60 lines). A session then reads
digest + one board + one handoff: still constant cost.

### Board regeneration (a maintenance op — never per-session, never at open)
Whoever runs cleanup (or any session that wants a fresher board) claims `_meta/digest`, rebuilds
the rows between the markers from the task metas, then releases:

```bash
mkdir -p memory/sync/claims/_meta
mkdir memory/sync/claims/_meta/digest 2>/dev/null || exit 0   # someone's already doing it
# regenerate <!-- BOARD:START/END --> from the frontmatter of each memory/tasks/*/meta.md
rm -rf memory/sync/claims/_meta/digest
```

This read-all-metas cost is amortized maintenance, **not** something a normal session pays.

---

## 5. TASK FILES

Each task folder holds a small structured status file (owner writes it — uncontended) and a
resume file.

### memory/tasks/<task>/meta.md  (source of truth for the board)
```markdown
---
task: build-auth
status: in-progress          # todo · in-progress · blocked · done
priority: P1                 # P0 · P1 · P2
deps: [api-schema]           # task names, or []
owner: agent-a1b2            # or —
updated: 2026-07-07 14:30
---
```
Machine-readable frontmatter so regeneration and scripts parse it without guessing.

### memory/tasks/<task>/handoff.md  (WHOLE-READ at open)
```markdown
# Handoff — <task>
updated: YYYY-MM-DD HH:MM by <agent_id>

## State        [done / half-done — exact files, exact stop point]
## Next Steps   [prioritized, concrete]
## Blocked      [what + why + who unblocks]
## Environment  [uncommitted changes, running processes, test state]
```

---

## 6. SESSION LIFECYCLE

### Open (bounded, constant cost)
```
1. Read memory/digest.md            → state, cached board, hot domains, guardrails
   (router? → read memory/boards/<ws>.md for your workstream)
2. Pick a task by the pick order → read memory/tasks/<task>/handoff.md (if it exists)
─── Open complete. Do NOT read more "to get oriented." ───
3. Claim before touching anything (Section 7)
4. Prior knowledge: grep-gated only (Section 8)
Hard cap: ~10 whole-file reads for the entire session.
```
Open is **2 whole reads** (3 in router mode) — constant regardless of repo size.

### Close — always, no exceptions
```
1. Update memory/tasks/<task>/handoff.md — exact state, next steps, blockers, uncommitted work.
   Update memory/tasks/<task>/meta.md — status/owner/updated. (Both under your task claim — no contention.)
2. Release every claim: rm -rf the module dir + append `released` row to work-map.md.
3. Append session events to memory/events.log (incl. task_status if it changed).
4. (Optional) If the board looks stale and you want it fresh, regenerate it (Section 4).
   Otherwise cleanup will. No session is REQUIRED to rewrite the shared digest.
```

---

## 7. CLAIMS — COLLISION PREVENTION (atomic, hierarchical, heartbeat)

Claims are **directories** because `mkdir` is atomic: two racing sessions → exactly one wins.
Hierarchical `<area>/<module>` (e.g. `orders/checkout`) lets different modules of one area be
owned by different agents.

```bash
# claim one module
mkdir -p memory/sync/claims/<area>
mkdir    memory/sync/claims/<area>/<module>        # fails = owned → pick another task
echo "<agent_id> | ttl=1h | <task>" > memory/sync/claims/<area>/<module>/owner
echo "| <agent_id> | <area>/<module> | $(date +%FT%H:%M) | active |" >> memory/work-map.md

# HEARTBEAT — at the start of each significant work step, prove you're alive:
touch memory/sync/claims/<area>/<module>/owner

# who owns an area? (existence check — never read files for this)
ls memory/sync/claims/<area>/

# release (session close)
rm -rf memory/sync/claims/<area>/<module>
echo "| <agent_id> | <area>/<module> | $(date +%FT%H:%M) | released |" >> memory/work-map.md
```

**Rules:**
- NEVER edit inside another agent's live claim. Pick a different task instead.
- **TTL 1h default** (30m quick, 2h long). Because owners **heartbeat**, staleness is measured from
  the owner file's **modification time**, not creation: stale iff `now − mtime(owner) > ttl`
  (`stat -c %Y owner`). A short TTL now means *fast crash recovery* without punishing long work.
- **Stale / crashed claim:** if `now − mtime(owner) > ttl`, **or** the claim dir has no readable
  `owner` file (crashed between the two mkdir/echo steps), any agent may `rm -rf` it, log
  `claim_stale` to events.log, and take it. The ONLY allowed cross-agent delete.
- **Needing several modules:** claim them all up front. If any `mkdir` fails, `rm -rf` the ones you
  got and pick different work — never sit holding a partial set. (Strict protocol: Appendix.)
- Granularity: module/folder, never single files.

---

## 8. KNOWLEDGE — ENCODE, INDEX, RETRIEVE

### Encode the moment it happens
| Trigger | Type |
|---|---|
| > 15 min to figure out | problem + solution |
| Chose X over Y because Z | design |
| Broke + fixed | solution |
| Approach made things worse | anti |
| Same issue 3rd time | pattern (and consider a `bugs/` record — Appendix) |

**File:** `memory/knowledge/<domain>/<type>-<topic>-<4char>.md`
```markdown
---
id: solution-auth-jwt-refresh-a1b2
type: problem|pattern|solution|anti|design
verified: yes            # static quality flag; changes rarely, lives here (not in a mutable counter)
agent: <agent_id>
tags: [jwt, refresh, django, simplejwt, cookie, rotation]   # 4–8 SPECIFIC tags → better grep hits
created: YYYY-MM-DD
---
## TL;DR      [1–2 sentences — this is what the registry row shows]
## Content    [steps, rationale, code, commands]
## Evidence   [errors, logs, snippets]
```

**Index it** — sharded per domain, pure append (no truncation, no header needed for grep):
```bash
touch memory/registry/<domain>.md
echo "| <id> | <type> | <TL;DR ≤12 words> | <tags> | $(date +%F) |" >> memory/registry/<domain>.md
```

### Retrieve (grep-gated, capped)
```
1. Task in a Hot Domain (per digest)? → read memory/hot.md once (best-known solutions).
2. Else:  rg -i '<keywords>' memory/registry/<domain>.md | head -20
   (unsure of domain → rg -i '<keywords>' memory/registry/ | head -20)   ← cap the cross-shard grep too
3. Read at most 3 matching memory/knowledge/<domain>/<id>.md files.
Never read knowledge/ without a registry grep hit. Never cat a registry shard.
```

**Reuse signal (append only — counts are DERIVED, never stored):**
```bash
echo "[$(date +%FT%H:%M)] <agent_id> reuse(<id>) ok"     >> memory/events.log   # or: reuse(<id>) misled
```
`times_reused` / `last_used` are computed at cleanup by grepping `reuse(<id>) ok` in events.log and
materialized into `hot.md`. Nothing mutable ever lives in an append-only file.
Supersede outdated knowledge by appending a new registry row tagged `[SUPERSEDES <old-id>]`.

---

## 9. APPEND-ONLY DISCIPLINE

`registry/*.md`, `work-map.md`, `events.log`:
- Write ONLY via a single `echo "..." >>` (atomic for small appends). Never Edit/Write, never
  edit/delete rows, never read whole — grep only.
- `work-map.md` is **advisory audit only** — the claim directory is the binding truth, so a row
  lost to a rare compaction race is a lost log line, never a lost claim.

Rewritable files (each under its claim): `digest.md` + `boards/*.md` (claim `_meta/digest`),
`tasks/*/{meta,handoff}.md` (that task's claim), `hot.md` (cleanup only).

**events.log lines:**
```
[ts] <agent_id> claim(<area>/<module>)
[ts] <agent_id> release(<area>/<module>)
[ts] <agent_id> claim_stale(<area>/<module>) — owner <other_id>, expired|no-owner
[ts] <agent_id> learn(<knowledge-id>)
[ts] <agent_id> reuse(<knowledge-id>) ok|misled
[ts] <agent_id> task_status(<task>) <status>
[ts] <agent_id> digest_regen
```

---

## 10. SCALE FEATURES (activate only when needed)

| Feature | Activate when | What it is |
|---|---|---|
| `boards/<ws>.md` router | Board > ~15 rows | Digest routes to per-workstream boards, each < 60 lines |
| `hot.md` | A domain grepped session after session | Curated best-known solutions per hot domain, < 40 lines, refreshed at cleanup |
| Scoring/decay | knowledge/ > ~50 files after merging | Rank registry rows by confidence/frequency (derive from events.log) |

---

## 11. CLEANUP (whichever session notices; claim `_meta/cleanup` first)

Triggers: digest/board > 60 lines · a registry shard > ~100 rows · a knowledge domain > ~30 files ·
work-map > ~200 rows · board visibly stale.

1. Regenerate `digest.md`/`boards/*.md` from task metas (Section 4).
2. Move `done` tasks out of the board → one line each in Recent Changes; archive `tasks/<task>/` →
   `tasks/_archive/`.
3. Compact `work-map.md`: matched claim/release pairs older than 7 days → one
   `[COMPACTED N rows YYYY-MM-DD]` line (the only allowed exception to append-only).
4. Merge duplicate knowledge (same topic + same conclusion) → append `[SUPERSEDES]` rows.
5. Refresh `hot.md` from most-reused knowledge: `grep 'reuse(' events.log | grep ' ok'` → counts ×
   recency (ignore `misled`).
6. Still oversized? → activate the next scale feature (Section 10).

---

## 12. CONFLICT RESOLUTION

| Situation | Action |
|---|---|
| Two agents want the same module | `mkdir` decides — loser picks a different task |
| Owner mtime older than TTL, or no owner file | Stale/crashed: delete, log `claim_stale`, take it |
| Agent needs modules it can't all get | Release the ones it got, back off, pick other work |
| Two knowledge files contradict | Newer wins; append `[SUPERSEDES]` row |
| Knowledge contradicts a guardrail | Guardrail wins; log `reuse(<id>) misled` |
| Board owner ≠ claim owner | Claim directory + task meta are the truth; board self-corrects at next regen |
| `_meta/digest` claim held | Skip regen (someone else is doing it); your task meta is already updated |

---

## 13. GUARDRAILS (Non-Negotiable)

- **DO NOT** skip the bounded open (digest → handoff; 3 in router mode) or read extra "to orient"
- **DO NOT** edit anything before claiming its module; heartbeat long work
- **DO NOT** touch another agent's live claim (only documented stale/crashed takeover)
- **DO NOT** `cat` registry shards, work-map.md, events.log, or knowledge/ without a grep hit
- **DO NOT** use Edit/Write on append-only files — single `echo >>` only; never truncate a shard
- **DO NOT** store mutable counters anywhere — derive them from events.log
- **DO NOT** exceed ~10 whole-file reads per session
- **DO NOT** let digest/board exceed 60 lines or hot.md exceed 40
- **DO NOT** rewrite the shared digest every session — update your task meta; let cleanup regen it
- **DO NOT** skip session close — handoff, meta, release, log: every session
- **DO NOT** read nestmind.md during normal work — CLAUDE.md has the rules
- **DO NOT** store secrets in ANY memory file, `commands/` included · **DO NOT** mix projects

---

## Quick Reference

```
OPEN    digest.md → (board) → tasks/<task>/handoff.md        [whole-read, ≤ 3 files]
PICK    ① own stale in-progress  ② highest-priority todo whose deps are done
CLAIM   mkdir -p claims/<area> && mkdir claims/<area>/<module>   (+ owner + work-map row)
        touch owner every work step (heartbeat)
WORK    Hot Domain → read hot.md once · else rg registry/<domain>.md → ≤ 3 knowledge files
        learn → knowledge/<domain>/<id>.md + registry row (echo >>)
        reused? → echo "reuse(<id>) ok" >> events.log
CLOSE   handoff + meta → release claims → events.log   (digest regen = cleanup's job)
BUDGET  constant open cost · ~10 whole reads max · growing files are grep-only
```

---

## Appendix — Deferred stores (add when the need is real, not before)

Keep the core small; introduce these only when they earn their keep. All are grep-gated /
on-demand — none touch the session-open budget.

- **Decision records — `memory/decisions/NNNN-title.md`.** Formalized, numbered, immutable `design`
  records for **project-wide/architectural** choices (e.g. `0005-remove-redis.md`). Index them in a
  `registry/decisions.md` shard. Local "why this function does X" stays in `knowledge/`.
- **Bug records — `memory/bugs/bug-NNN.md`.** Only for **recurring** bugs with a lifecycle
  (open → fixed → regressed) and a regression-test link — your "same issue 3rd time" trigger is the
  signal to promote a `problem` into a tracked bug. One-off bugs stay as knowledge. Sections:
  Status · Cause · Fix · Regression test.
- **Command snippets — `memory/commands/deploy.md`, `migration.md`, `release.md`.** On-demand
  whole-read, bounded. **Secrets guardrail applies in full — placeholders / env-var references
  only, never real tokens or connection strings.**
- **Strict multi-claim (all-or-nothing).** When tasks routinely need 3+ modules: attempt every
  `mkdir` in a fixed global order; on any failure `rm -rf` all acquired and retry after a short
  back-off. The fixed order prevents livelock. Not needed until multi-module tasks are the norm.

*NestMind v3.1 — the board distributes the work, the claims prevent the collisions, heartbeats*
*keep long work safe, per-task metas kill the digest bottleneck, and grep keeps it cheap forever.*
