# Quarantined Tasks

> Tasks that exhausted their inner-loop retry budget (`MaxAttempts`) without passing both gates. **Terminal state** — the supervisor will NEVER re-dispatch a task listed here. A human must manually move a task back to `TODO.md` (with a `[RETRY]` prefix) after diagnosing and cleaning up the failure.

This file is **owned by the supervisor**. Do not edit it directly to add tasks. To recover a task, follow the recovery procedure below.

---

## Why this file exists

When the inner loop in Phase 4 exhausts `MaxAttempts` (default 3) and the task still fails both Gate 1 (subagent self-verify) and Gate 2 (supervisor re-verify), continuing to retry would:

1. Burn tokens on a task the model demonstrably cannot solve.
2. Risk runaway retry loops that pollute the working tree.
3. Mask structural problems (bad verification statement, impossible task, missing context).

Routing the task here **forces a human pause**. Quarantine is the factory line's safety valve — the supervisor declares "I am stuck" and hands off cleanly.

---

## Schema

Each quarantined task carries the full attempt history so a human can diagnose without re-running anything:

```markdown
## <Task: string>
Quarantined: <ISO8601>
Attempts: <N>/<max>
Final Fail Reason: <gate-1 | gate-2> — <one-line reason>
Verification: <the original verification statement>
Verification-LLM: <if present>
Last Diff: <output of `git diff` at time of quarantine — usually empty if Phase 4 caught the failure>
Attempt History:
- Attempt 1: <evidence> — <PASS|FAIL at Gate 1 or Gate 2>
- Attempt 2: <evidence> — <PASS|FAIL>
- Attempt 3: <evidence> — <PASS|FAIL>
Dirty Working Tree: <yes | no — yes means there are uncommitted edits from the last attempt>

---
```

---

## Recovery procedure

To re-dispatch a quarantined task:

1. **Diagnose.** Read the Final Fail Reason and the Attempt History. Common diagnoses:
   - **Verification statement is ambiguous or wrong.** Rewrite `Verification:` to be more specific. Examples: "the test passed but the verification grep matched the wrong line" → tighten the pattern.
   - **Task is too large.** Split into smaller tasks in `TODO.md` and delete this entry.
   - **Missing context.** Add path hints, scope the task to specific files, or seed the project with better skills/CLAUDE.md.
   - **Impossible requirement.** Drop the task — mark it `[ABANDONED]` in place and move on.
   - **Tool/API failure.** External service was down. Wait, retry later.

2. **Clean the dirty working tree** (if `Dirty Working Tree: yes`):
   ```bash
   # Inspect the leftover edits
   git status
   git diff

   # Option A: discard the partial work
   git checkout -- <files>

   # Option B: preserve it in a stash for manual salvage
   git stash push -m "quarantined attempt on <task name>"
   ```

3. **Move the task back to TODO.md.** Remove the entry from `QUARANTINED.md` and append to the bottom of `TODO.md` with this header:

   ```markdown
   - [ ] [RETRY] Task: <original Task: string, sans prefix>
     Verification: <possibly rewritten>
     Verification-LLM: <possibly rewritten>
     MaxAttempts: <possibly increased, e.g., 5>
     Depends On: <unchanged>
     Fail Reason: <one-line summary of why the previous attempts failed>
   ```

   The `[RETRY]` prefix tells the supervisor to throttle the task (skip if it failed within the last 2 ticks — prevents thrashing). After 2 ticks it becomes READY again.

4. **Commit your changes** so the audit trail reflects the recovery:
   ```bash
   git add TODO.md QUARANTINED.md
   git commit -m "chore: requeue <task name> after quarantine diagnosis"
   ```

---

## Anti-patterns

- **Auto-resurrecting quarantined tasks.** Never modify the supervisor to retry quarantined tasks programmatically. The whole point of quarantine is the forced human pause.
- **Bulk-resetting QUARANTINED.md.** Moving all entries back to TODO.md at once defeats the safety valve. Diagnose each one individually.
- **Resurrecting without cleaning the working tree.** Re-dispatching on top of stale edits produces confusing failures — the subagent sees a dirty tree and doesn't know which files are "theirs."

---

## Inspection tips

- The `Last Diff` field is usually empty because Phase 4 catches most failures before a subagent commits. If it's non-empty, the dirty tree is the most likely cause of cascading failures.
- `Attempt History` reveals failure patterns. If attempts 1, 2, 3 all failed at the same gate with similar evidence, the task is structurally stuck — diagnose the task itself, not the attempts.
- A task that quarantines on its **second** use in a milestone may indicate the project's tools or skills need updating. Consider improving `.claude/skills/` or `CLAUDE.md` for the next milestone.
