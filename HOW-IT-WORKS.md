# How LoopEngineering Works — A Technical Deep-Dive

> A complete walk-through of the LoopEngineering repo: the moving parts, the data flow, the state machine, and the design choices behind every file.

---

## 1. What This Repo Is

LoopEngineering is a **Claude Code plugin that turns `/loop` into an autonomous Kanban supervisor.** It is a single artifact — one slash command plus three markdown files — that runs inside Claude Code's recurring `/loop` cycle and turns a backlog of tasks into a stream of verified, git-committed work without a human in the loop.

It is not a framework, not a server, not a daemon. It is a **prompt** (`.claude/commands/loop-tasks.md`) wrapped around Claude Code's `/loop` scheduler, with the rest of the repo being persistent state.

The whole repo at a glance:

```
LoopEngineering/
├── .claude/
│   └── commands/
│       └── loop-tasks.md     # the slash command — defines the 6-phase supervisor
├── assets/
│   └── architecture.png      # visual diagram of one tick
├── .gitignore                # OS/editor/build exclusions
├── README.md                 # user-facing quickstart + walkthrough
├── PLAN.md                   # implementation plan + design rationale
├── TODO.md                   # pending tasks (human input)
├── INPROGRESS.md             # active tasks (supervisor-owned)
└── DONE.md                   # verified, git-committed tasks
```

No package.json. No source code. No CI. The repository is the documentation and the state, with the slash command as the only executable artifact.

---

## 2. The Mental Model

A naive `/loop` prompt breaks down fast. Within a handful of ticks, the agent:

1. **Hallucinates** tasks that don't exist
2. **Forgets** its own constraints set in earlier ticks
3. **Skips verification** and reports "done" on half-finished work
4. **Leaves** crashed subagent tasks stuck "in progress" forever
5. **Pollutes** its own context with accumulated noise

LoopEngineering fixes each of these failure modes with a **strict, phase-driven state machine that re-reads its instructions fresh on every tick.** The supervisor is invoked by `/loop` on a fixed cadence (default: 2 minutes), does exactly six things in order, prints a summary, and exits — leaving all state on disk so the next tick is stateless.

---

## 3. The Six-Phase State Machine

Every invocation of `/loop-tasks` walks the same six phases, in order, with no shortcuts:

```
   ┌─────────────────────────────────────┐
   │  Phase 1 — Stale Recovery           │
   │  sweep INPROGRESS.md > 10m old      │
   │  → send back to TODO.md as [RETRY]  │
   └──────────────┬──────────────────────┘
                  ▼
   ┌─────────────────────────────────────┐
   │  Phase 2 — Load State               │
   │  read TODO.md + DONE.md into memory │
   └──────────────┬──────────────────────┘
                  ▼
   ┌─────────────────────────────────────┐
   │  Phase 3 — Classify                 │
   │  every TODO task → BLOCKED or READY │
   │  (check Depends On: against DONE)   │
   └──────────────┬──────────────────────┘
                  ▼
   ┌─────────────────────────────────────┐
   │  Phase 4 — Dispatch                 │
   │  READY → INPROGRESS.md              │
   │  spawn isolated Agent per task      │
   │  batch independent calls (parallel) │
   └──────────────┬──────────────────────┘
                  ▼
   ┌─────────────────────────────────────┐
   │  Phase 5 — Dual-Gate Audit          │
   │  Gate 1: subagent self-verify       │
   │  Gate 2: supervisor re-verify       │
   │  PASS/PASS → DONE.md + git commit   │
   └──────────────┬──────────────────────┘
                  ▼
   ┌─────────────────────────────────────┐
   │  Phase 6 — Tick Summary             │
   │  print one-line dashboard           │
   └─────────────────────────────────────┘
```

The phases are **imperative, not declarative.** The supervisor does not "plan" a tick — it walks the same checklist every time. This is what makes the system robust: it cannot drift, because each tick is independent of the previous one's reasoning.

---

## 4. File-by-File Role

### 4.1 `.claude/commands/loop-tasks.md` — The Supervisor

This is the only "executable" file. It is a slash command — Claude Code reads its frontmatter, loads it as a system prompt, and the model becomes the Kanban supervisor for the duration of one tick.

The frontmatter declares the tool surface:

```yaml
---
description: Autonomous Kanban supervisor that processes TODO.md -> INPROGRESS.md -> DONE.md with dual-gate verification and git checkpoints.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent
---
```

`Agent` is the critical line — it's what lets the supervisor spawn isolated subagents in Phase 4. The other tools are the verification toolbox for Phase 5 Gate 2.

The body of the file is the 6-phase procedure, written in imperative markdown. Every phase is unambiguous about:
- What to read
- What to write
- What conditions trigger each branch
- What artifacts to produce

### 4.2 `TODO.md` — Human Input

This is where humans put work. The schema is deliberately tight:

```markdown
- [ ] Task: <short, imperative description>
  Verification: <concrete, binary test — what command, file, or output proves success?>
  Depends On: <none | exact Task: string of another task>
```

Three required fields, three rules:

1. **`Task:`** — imperative, specific. "Add a /health endpoint to the API server" not "Make the API better."
2. **`Verification:`** — binary, runnable. Must be expressible as a single bash command a human could execute in 10 seconds. Subjective verifications like "looks good" are explicitly rejected by Gate 2.
3. **`Depends On:`** — exact match of another task's `Task:` field, character-for-character. Used in Phase 3 to compute the dependency graph.

The file also contains example tasks with health-check / Jest / README patterns, which serve as the de-facto schema documentation.

### 4.3 `INPROGRESS.md` — Supervisor-Owned Transient State

This file is **owned by the supervisor**. The README warns humans explicitly not to edit it — manual edits race with the loop and cause lost updates.

Each entry carries:
- The full task block (Task + Verification + Depends On)
- A `Started: <ISO8601>` stamp added by Phase 4

The 10-minute timestamp is the only "volatile" field in the whole system. Phase 1 reads it to detect stale tasks (crashed subagents) and recovers them.

### 4.4 `DONE.md` — Append-Only History

Verified tasks land here, one per line, after both gates pass and `git commit` succeeds. The file grows monotonically — never edited, only appended to. It serves as:
- A human-readable audit log of what the supervisor shipped
- The lookup table for `Depends On:` checks in Phase 3
- A natural progress indicator (line count = shipped count)

### 4.5 `README.md` — User Quickstart

How to install, how to add a task, how to read the tick summary, common operations (skip, force-retry, revert a bad commit), troubleshooting, and known limitations. Optimized for someone who has never seen the system before.

### 4.6 `PLAN.md` — Architecture & Design Rationale

The "why" behind every decision. Includes:
- Failure modes that naive `/loop` exhibits and how each is fixed
- Detailed walkthrough of each phase
- The verification protocol (why two gates)
- Failure recovery matrix
- Hardening best practices (binary verifications, sub-10-minute tasks, exact dependencies)
- Honest limitations (fake parallelism, no inter-task comms, /loop expiration)

### 4.7 `assets/architecture.png` — Visual Reference

A diagram of the tick flow, intended for visual learners who skim before reading. Mirrors the ASCII art in `README.md`.

---

## 5. Anatomy of a Tick

A complete lifecycle, end-to-end:

### T=0:00 — `/loop` triggers `/loop-tasks`

The Claude Code scheduler invokes the slash command. The supervisor starts with a fresh context window containing the slash command body, plus whatever state files it chooses to read.

### T=0:01 — Phase 1: Stale Recovery

```bash
# Pseudocode
for task in INPROGRESS.md:
    if now - task.started > 10m:
        move task to TODO.md with [RETRY] prefix
        append "Fail Reason: stale — exceeded 10m timeout"
```

This is the **Janitor.** Without it, a single crashed subagent would brick the queue forever. With it, the system self-heals every tick.

### T=0:02 — Phase 2: Load State

The supervisor reads `TODO.md` and `DONE.md` into its working memory. From `DONE.md` it builds a set of completed task names — the lookup table for dependency checks.

### T=0:03 — Phase 3: Classify

For each task in `TODO.md`:

- **BLOCKED** — `Depends On:` references a task whose `Task:` string is NOT in the DONE set. Skip.
- **READY** — `Depends On: none` OR the dependency IS in DONE. Eligible.

Special case: tasks prefixed `[RETRY]` whose last failure was a gate-1 or gate-2 failure fewer than 2 ticks ago are skipped to prevent thrashing. Everything else READY gets dispatched.

### T=0:04 — Phase 4: Dispatch

For each READY task:

1. Move the task block from `TODO.md` to `INPROGRESS.md`. Add `Started: 2026-06-17T12:34:56Z`.
2. Spawn a subagent via `Agent(subagent_type="general-purpose")`.
3. The subagent prompt contains:
   - The exact `Task:` description
   - The exact `Verification:` statement
   - The directive: *"Self-verify using bash/read/glob/grep. Return exactly `RESULT: PASS` or `RESULT: FAIL` and `EVIDENCE: <proof>`."*
4. Independent Agent calls are batched in a single message — within one Claude Code context they run sequentially in batch (true parallelism requires separate sessions, see Limitations §10).

### T=0:05–0:50 — Subagents do their work

Subagents run with isolated context windows. They edit files, run commands, write code, run tests. They MUST return a structured `RESULT: PASS|FAIL` with concrete `EVIDENCE: <proof>`. No pass without proof — this rule is enforced by the subagent prompt itself.

### T=0:51 — Phase 5: Dual-Gate Audit

The supervisor **does not trust the subagents.** For each completed subagent, the supervisor independently re-runs the verification using its own `Bash`/`Read`/`Glob`/`Grep` tools.

Decision matrix:

| Gate 1 (Subagent) | Gate 2 (Supervisor) | Action                                                     |
|--------------------|----------------------|------------------------------------------------------------|
| FAIL               | (skip)               | Stay in INPROGRESS. Append `Fail Reason: gate-1 — <reason>`|
| PASS               | FAIL                 | Stay in INPROGRESS. Append `Fail Reason: gate-2 — <reason>`|
| PASS               | PASS                 | Move to DONE. `git add -A && git commit -m "..."`         |

The supervisor wins ties. Gate 2 always overrides Gate 1.

### T=0:52 — Phase 6: Tick Summary

One block to stdout:

```
── Tick @ 2026-06-17T12:34:56Z ──
  Promoted to DONE: 2
  Still In Progress: 1
  Blocked: 3
  Recovered (stale): 0
  Backlog: 5
```

This is the supervisor's only output to the human. It does not narrate; it does not summarize; it does not editorialize. Five numbers.

### T=0:53 — Tick ends

The supervisor's context is discarded. The scheduler sleeps until the next tick (default: 2 minutes). All state is on disk.

---

## 6. The Two-Gate Verification Protocol — Why Two Gates?

A single verification is a single point of failure. Subagents can:

- **Hallucinate success** — claim files were created that don't exist
- **Misinterpret** the verification statement
- **Skip verification entirely** — return early
- **Lie** (in adversarial or confused states)

Gate 2 catches all four because the supervisor runs the verification independently with its own tool calls. The subagent's report is treated as a *claim*, not as truth. Cost: a second execution per task. Benefit: the difference between "shipped verified work" and "shipped plausible work."

The protocol is symmetric — both gates must agree. Gate 2 has the final word because it has the cleaner context (no accumulated tick noise).

---

## 7. Failure Modes & Recovery

| Failure                                  | Detection                              | Recovery                                      |
|------------------------------------------|----------------------------------------|-----------------------------------------------|
| Subagent crashes mid-task                | `Started:` > 10m                       | Phase 1 retries it as `[RETRY]`               |
| Subagent hallucinates success            | Gate 2 fails                           | Task stays in INPROGRESS with `Fail Reason`   |
| Verification statement is ambiguous      | Gate 2 keeps disagreeing with Gate 1   | Human edits the verification, retries         |
| Task is too large                        | Hits 10m timeout                       | Human splits into smaller tasks               |
| Git commit fails (e.g., no `.git`)       | `git commit` returns non-zero          | Skip commit, log warning, continue            |
| Loop session crashes                     | `/loop` is session-scoped              | Re-run `/loop 2m /loop-tasks` — state on disk |
| Subagent makes destructive changes       | Gate 2 catches; or git log shows bad commit | `git reset --hard HEAD~1`                 |

The recovery story is "git is the safety net, the state files are the source of truth, and Phase 1 is the janitor." No silent failures — every failure leaves an artifact (a `Fail Reason:` line, a missing commit, a stale task).

---

## 8. Why This Design Works

### Statelessness
Each tick is independent. The supervisor does not remember the previous tick — it re-reads everything. This eliminates the #1 failure mode of long-running agent loops (context drift, accumulated noise, constraint forgetting).

### Plain Text as Database
Three markdown files replace what would otherwise be a database, queue, and audit log. The trade-off: no transactions, no concurrent writes, no indices. The win: human-readable, diff-able, git-trackable, restart-safe. For a single-user factory floor, this is a Pareto-optimal choice.

### Git as Checkpoint
Every verified task gets its own commit. This is not release engineering — it's crash recovery. If the system breaks, the audit trail is in `git log`. If a bad commit slips through, `git reset --hard HEAD~1` and move on.

### Dual-Gate Independence
The supervisor and subagents are separate Claude invocations. Gate 2 is structurally different from Gate 1 — different context, different tools used, different reasoning. The probability of both agreeing on the wrong answer is vanishingly small.

### Cadence over Latency
Two minutes per tick is not optimized for speed — it's optimized for *observability.* A human watching the tick summaries can catch problems within minutes. Optimizing for speed would mean fewer checkpoints, fewer recoveries, more catastrophic failures.

---

## 9. How It Connects to the Broader Loop Engineering Paradigm

LoopEngineering is one concrete implementation of the four-loop agent pattern:

| Loop                | Implementation in this repo                            |
|---------------------|--------------------------------------------------------|
| **Doer**            | The subagent spawned in Phase 4                        |
| **Checker**         | The two-gate verification in Phase 5                   |
| **Trigger**         | Claude Code's `/loop` scheduler invoking `/loop-tasks` |
| **Improver**        | Implicit — humans read tick summaries and tune verifications over time |

The repo deliberately does NOT implement:
- An automatic improver (the human watches and tunes)
- Inter-subagent communication (findings land on the filesystem)
- Cross-task learning (each tick is stateless)

This is intentional minimalism. The system ships with what works in production and nothing else. Everything else is documented in PLAN.md as hardening candidates.

---

## 10. Limitations (Honest)

These are not bugs — they are documented constraints of the implementation:

1. **Fake parallelism.** Within a single Claude Code session, `Agent` calls run sequentially in batch. True parallel execution requires separate shell-spawned sessions.
2. **No inter-subagent communication.** Subagents see only their own task. Cross-task findings must land on the filesystem and be picked up via a separate task in `TODO.md`.
3. **Context drift over very long sessions.** Even with the stateless design, the slash command body is re-read every tick — but very long sessions still degrade. Restart Claude Code daily.
4. **Git is not optional in practice.** Without git, the checkpoint safety net disappears and a bad Gate 2 means a permanently broken state.
5. **Verification is only as good as the statement.** Vague verifications pass on garbage work — both gates will agree on garbage.
6. **`/loop` expires.** Session-scoped loops auto-expire after ~3 days. Re-run `/loop 2m /loop-tasks` periodically.

---

## 11. Extending the System

Safe, supported extensions:

- **Custom verification runners** — put any test command (pytest, jest, go test) directly in the `Verification:` field.
- **Different cadence** — `/loop 5m`, `/loop 30s`. Trade-off: responsiveness vs. context churn.
- **Different stale timeout** — edit `> 10 minutes` in `loop-tasks.md`.
- **Add a max-retry cap** — move tasks that have `[RETRY]` ≥ 3 times to a `QUARANTINED.md` file.

Unsafe, unsupported extensions:

- Editing `INPROGRESS.md` while the loop is running
- Subjective `Verification:` statements ("looks good", "works correctly")
- Loose `Depends On:` strings that don't character-match the dependency's `Task:` field
- Tasks expected to take longer than 10 minutes (split them instead)

---

## 12. Glossary

- **Tick** — one full invocation of `/loop-tasks` by the `/loop` scheduler.
- **Gate 1** — the subagent's self-reported verification outcome.
- **Gate 2** — the supervisor's independent re-verification of Gate 1's claim.
- **READY** — a task in `TODO.md` whose dependencies are all in `DONE.md`.
- **BLOCKED** — a task in `TODO.md` with at least one unsatisfied dependency.
- **`[RETRY]`** — a task prefix marking one that has previously failed and is eligible to be re-dispatched.
- **Stale** — a task in `INPROGRESS.md` whose `Started:` timestamp is older than 10 minutes.
- **Subagent** — an isolated Claude invocation spawned via the `Agent` tool, used for one task at a time.
- **Verification Statement** — the binary, runnable test that defines "done" for a task.

---

## 13. TL;DR

LoopEngineering is one slash command and three markdown files that turn `/loop` into a self-healing Kanban factory floor. Every two minutes (configurable), the supervisor wakes up, sweeps stale tasks, classifies the backlog, dispatches ready tasks to isolated subagents, independently re-verifies their work, commits what passes both gates, and prints five numbers.

No code. No servers. No frameworks. State on disk, scheduler in `/loop`, safety net in git. The smallest possible loop that ships verified work.
