# LoopEngineering — Master Implementation Plan

> A self-healing, version-controlled Kanban orchestrator that runs inside Claude Code's `/loop`.

---

## 1. Executive Summary

LoopEngineering is a **state-aware, agentic task loop** built on top of Claude Code's `/loop` command. It turns a one-line prompt into a strict, six-phase supervisor that:

- Reads tasks from `TODO.md` (markdown backlog)
- Dispatches independent tasks to **isolated subagents**
- Verifies their work through **two independent gates** (subagent + supervisor)
- Promotes only verified tasks to `DONE.md`
- Commits each verified task to git as a safety net
- Recovers from crashes, timeouts, and hallucinations automatically

The whole system lives in plain text — three markdown files, one slash command, one git repo. No servers, no frameworks, no external services.

---

## 2. Why This Exists

A naive `/loop` prompt like *"do my tasks"* fails fast. Within a few ticks the agent:

- Hallucinates tasks that don't exist
- Forgets its own constraints
- Skips verification and moves half-done work to `DONE`
- Leaves crashed subagents' tasks stuck in "in progress" forever
- Pollutes its own context with accumulated noise

LoopEngineering fixes each of these failure modes with a hard-coded, phase-driven state machine that re-reads its instructions fresh on every tick.

---

## 3. Architecture

```
                         ┌────────────────────────────────┐
                         │   Claude Code session          │
                         │   /loop 2m /loop-tasks         │
                         └──────────────┬─────────────────┘
                                        │ every 2 minutes
                                        ▼
              ┌─────────────────────────────────────────────┐
              │   Phase 1: Stale Recovery (Janitor)        │
              │   - sweep INPROGRESS.md > 10m old          │
              │   - move them back to TODO.md as [RETRY]   │
              └──────────────┬──────────────────────────────┘
                             ▼
              ┌─────────────────────────────────────────────┐
              │   Phase 2-3: Classification (Strategist)   │
              │   - parse every TODO task                  │
              │   - check Depends On against DONE.md       │
              │   - label: BLOCKED or READY                │
              └──────────────┬──────────────────────────────┘
                             ▼
              ┌─────────────────────────────────────────────┐
              │   Phase 4: Dispatch (Manager)              │
              │   - move READY → INPROGRESS.md             │
              │   - stamp Started: <ISO8601>               │
              │   - spawn isolated Agent per task          │
              │   - pass Task + Verification + directive   │
              └──────────────┬──────────────────────────────┘
                             ▼
              ┌─────────────────────────────────────────────┐
              │   Phase 5: Dual-Gate Audit (Auditor)       │
              │   Gate 1 = subagent self-verify            │
              │   Gate 2 = supervisor independently verify  │
              │   Both PASS → DONE.md + git commit         │
              └──────────────┬──────────────────────────────┘
                             ▼
              ┌─────────────────────────────────────────────┐
              │   Phase 6: Tick Summary (Reporter)         │
              │   print dashboard to stdout                │
              └─────────────────────────────────────────────┘
```

---

## 4. Implementation Phases (Build Order)

### Phase 0 — Decide Scope

Before writing any code, answer:

- **What kind of work will the loop process?** Bug fixes? Test writing? Migration scripts? New features?
- **Where does the work live?** A single repo? Multiple repos? The orchestrator's own repo?
- **What verification tools are available?** `bash`? A test runner? A deploy hook?

Document the answers at the top of `PLAN.md` (this file) so they survive across sessions.

### Phase 1 — Foundation

- [x] `git init` in the project root
- [x] Add `.gitignore` covering OS junk, editor state, common build outputs, env files
- [x] Default branch: `main`
- [ ] Create a feature branch: `git checkout -b feat/loop-engineering`
- [ ] Decide: do all `feat/*` work merge back via PR, or fast-forward?

### Phase 2 — Slash Command

- [ ] `mkdir -p .claude/commands`
- [ ] Create `.claude/commands/loop-tasks.md`
- [ ] Paste the 6-phase orchestrator logic with frontmatter (see `loop-tasks.md` in this repo)
- [ ] Restart Claude Code (`Ctrl+D` then `claude`)
- [ ] Type `/` in an empty prompt and confirm `/loop-tasks` is listed

### Phase 3 — State Files

- [ ] `TODO.md` — schema header + 2-3 example tasks
- [ ] `INPROGRESS.md` — minimal header
- [ ] `DONE.md` — minimal header

### Phase 4 — First Tasks

- [ ] Add 2-3 **binary-verifiable** tasks to `TODO.md`
- [ ] Each must specify: `Task:`, `Verification:`, `Depends On:`
- [ ] Each `Verification:` must be a concrete, testable statement
- [ ] Avoid subjective criteria like "looks good" or "works correctly"

### Phase 5 — Run the Loop

- [ ] `/loop 2m /loop-tasks`
- [ ] Read every tick summary for the first hour
- [ ] If tasks get stuck at Gate 2, tighten the verification statement
- [ ] If tasks exceed 10 minutes, split them into smaller micro-tasks

### Phase 6 — Hardening

- [ ] Add a `Makefile` or `scripts/` directory for repeatable verification commands
- [ ] Move example tasks to `examples/TODO.md`; keep `TODO.md` clean
- [ ] Add a `CONTRIBUTING.md` if the system is shared
- [ ] Tune the 2-minute cadence based on observed task latency
- [ ] Add a "max retries" guard so a [RETRY] task doesn't loop forever

---

## 5. The 6-Phase Tick (Detailed)

### Phase 1 — Stale Task Recovery (The Janitor)

Every subagent has a 10-minute SLA. If a task in `INPROGRESS.md` has a `Started:` timestamp older than 10 minutes:

1. Remove the entire task block from `INPROGRESS.md`.
2. Append the same block to the bottom of `TODO.md`, prefixed with `[RETRY]`.
3. Add a line beneath it: `Fail Reason: stale — exceeded 10m timeout`.

This is the most important fix relative to a naive loop. Without it, a single crashed subagent bricks the entire pipeline.

### Phase 2 & 3 — Classification (The Strategist)

For every task in `TODO.md`:

- **BLOCKED** — its `Depends On:` field references a task that is NOT in `DONE.md`. Do not touch.
- **READY** — `Depends On: none` OR the dependency IS in `DONE.md`. Eligible for dispatch.

### Phase 4 — Dispatch (The Manager)

For every READY task:

1. Move the entire task block from `TODO.md` to `INPROGRESS.md`. Stamp with `Started: <ISO8601>`.
2. Spawn an isolated subagent via the **Agent tool** with `subagent_type: "general-purpose"`.
3. The subagent prompt MUST include: exact Task description, exact Verification Statement, and the directive: *"Self-verify against the Verification Statement using bash/read tools. Return exactly: `RESULT: PASS` or `RESULT: FAIL` plus `EVIDENCE: <proof>`."*
4. Batch independent Agent calls in a single message so they run in parallel.

### Phase 5 — Dual-Gate Verification & Checkpoint (The Auditor)

The supervisor does NOT trust the subagent.

| Gate 1 (Subagent) | Gate 2 (Supervisor) | Action |
|--------------------|----------------------|--------|
| FAIL               | (skip)               | Keep in `INPROGRESS.md`; append `Fail Reason: gate-1 — <reason>` |
| PASS               | FAIL                 | Keep in `INPROGRESS.md`; append `Fail Reason: gate-2 — <reason>` |
| PASS               | PASS                 | Move to `DONE.md`; run `git add -A && git commit -m "chore(agent): verified task - <Task Name>"` |

### Phase 6 — Tick Summary (The Reporter)

Print a dashboard:

```
── Tick @ 2026-06-17T12:34:56Z ──
  Promoted to DONE: 2
  Still In Progress: 1
  Blocked: 3
  Recovered (stale): 0
  Backlog: 5
```

---

## 6. Verification Gates — Why Two?

A single verification is a single point of failure. The subagent may:
- Hallucinate success
- Misinterpret the verification statement
- Skip the verification entirely
- Lie (in adversarial scenarios)

The supervisor's Gate 2 re-runs the verification independently using `Bash`, `Read`, or `Glob`. The cost is a second execution; the benefit is catching all four failure modes above.

**Rule of thumb:** if Gate 1 says PASS but Gate 2 disagrees, the supervisor wins. Always.

---

## 7. Failure Modes & Recovery

| Failure                                  | Detection                                | Recovery |
|------------------------------------------|------------------------------------------|----------|
| Subagent crashes mid-task                | `Started:` > 10m                         | Phase 1 retries it |
| Subagent hallucinates success            | Gate 2 fails                             | Reverted to INPROGRESS with fail reason |
| Verification statement is ambiguous       | Gate 2 keeps disagreeing with subagent   | Edit the task in `INPROGRESS.md` to clarify |
| Task is too large                        | Hits 10m timeout                         | Split into smaller tasks manually |
| Git commit fails (e.g., no `.git`)       | `git commit` returns non-zero            | Skip commit, log warning, continue |
| Loop session crashes                     | `/loop` is session-scoped                | Re-run `/loop 2m /loop-tasks` — state is on disk |
| Subagent makes destructive changes       | Supervisor catches at Gate 2; or git log shows the bad commit | `git reset --hard HEAD~1` to revert |

---

## 8. Hardening Best Practices

### Write Binary Verifications

Avoid: "Make the UI look good"
Use: "The element `.btn-primary` exists in `index.html` and has the class `visible` after page load"

### Keep Tasks Under 10 Minutes

The stale-recovery timeout is 10 minutes. Any task that takes longer will be killed and retried. Split large features into micro-tasks.

### Make `Depends On:` Exact

If task A depends on task B, the dependency string in A's `Depends On:` must match B's `Task:` field character-for-character (whitespace included). Use copy-paste.

### Don't Edit INPROGRESS.md by Hand

The supervisor owns that file. Manual edits race with the loop and cause lost updates.

### Watch the First Hour

The first 60 minutes of a new orchestrator are the most informative. Read every tick summary. Patterns that emerge:

- Tasks consistently timing out → split them
- Tasks consistently failing Gate 2 → tighten verifications
- Tasks consistently passing both gates → system is healthy

### Set a Retry Cap

A `[RETRY]` task that fails N times should be auto-quarantined. Suggested: after 3 retries, move to a `QUARANTINED.md` file and stop retrying.

---

## 9. Limitations (Honest)

1. **Fake parallelism.** Subagents in the same Claude Code context run sequentially in batch, not truly in parallel. True parallel execution requires separate Claude Code sessions launched via shell.
2. **No inter-task communication.** Each subagent sees only its own task. They cannot share findings unless they touch the filesystem.
3. **Context drift.** Over many ticks, the supervisor's context accumulates summary noise. The structured phase model helps, but very long sessions still degrade. Consider restarting Claude Code daily.
4. **Git is not optional in practice.** Without git, the checkpoint safety net disappears and a bad Gate 2 means a permanently broken state.
5. **Verification is only as good as the statement.** If `Verification:` is vague, both gates will pass on garbage work.
6. **`/loop` expires.** Session-scoped loops auto-expire after ~3 days. You'll need to re-run `/loop 2m /loop-tasks` periodically.

---

## 10. File Manifest

```
LoopEngineering/
├── .git/                              # git internals
├── .gitignore                         # OS/editor/build exclusions
├── .claude/
│   └── commands/
│       └── loop-tasks.md              # the /loop-tasks slash command
├── README.md                          # user-facing quickstart
├── PLAN.md                            # this file
├── TODO.md                            # pending tasks
├── INPROGRESS.md                      # active tasks
└── DONE.md                            # verified completed tasks
```

---

## 11. Next Steps

1. Customize the 6-phase logic in `.claude/commands/loop-tasks.md` for your stack (Node, Python, etc.)
2. Add 2-3 real tasks to `TODO.md`
3. Run `/loop 2m /loop-tasks` and observe
4. Iterate on the verification statements based on what you see

Welcome to your autonomous factory floor.
