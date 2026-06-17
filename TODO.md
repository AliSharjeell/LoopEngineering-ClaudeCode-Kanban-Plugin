# Task Backlog

> Tasks the loop will pick up on the next tick. Do not edit INPROGRESS.md, DONE.md, or QUARANTINED.md by hand — the supervisor owns those files.

---

## Schema

Every task must follow this exact format. The supervisor parses line-by-line; sloppy formatting breaks the pipeline.

```markdown
- [ ] Task: <short, imperative description>
  Verification: <bash command that proves success>
  Verification-LLM: <rubric for visual or architectural judgment>   # optional
  MaxAttempts: <N>                                                  # optional, default 3
  Depends On: <none | exact Task: string of another task>
```

### Field reference

| Field | Required | Purpose |
|-------|----------|---------|
| `Task:` | yes | Imperative description. The supervisor and subagent both see this verbatim. |
| `Verification:` | one of these two required | Bash command the supervisor runs at Gate 2. Must exit 0 on success. |
| `Verification-LLM:` | one of these two required | Rubric for an LLM judge subagent at Gate 2. Use for visual / architectural checks bash can't capture. |
| `MaxAttempts:` | no (default unlimited) | **Opt-in cap** on the inner loop. Default is **unlimited** — the loop retries until Gate 2 passes or you stop the supervisor. Set explicitly only when you need a hard ceiling (suspected unsolvable task, runaway cost concern). |
| `Depends On:` | no | Character-exact match of another task's `Task:` field. |

At least one of `Verification:` or `Verification-LLM:` must be present. Tasks with neither are flagged `gate-2 — no verification specified` and re-enter the inner loop.

---

## Example tasks

### Standard (bash verification)

```markdown
- [ ] Task: Add a /health endpoint to the API server
  Verification: `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health` returns `200` and the response body contains `{"status":"ok"}`
  Depends On: none

- [ ] Task: Write a Jest test for the /health endpoint
  Verification: `npx jest health.test.js` exits 0 and stdout contains `Tests: 1 passed`
  MaxAttempts: 5
  Depends On: Add a /health endpoint to the API server

- [ ] Task: Add a health-check section to the README
  Verification: The string `## Health check` appears in `README.md`
  Depends On: Add a /health endpoint to the API server
```

### Visual / architectural (LLM verification)

```markdown
- [ ] Task: Refactor the dashboard sidebar to use the Linear-style minimalist hierarchy
  Verification-LLM: Read the diff in `git diff HEAD`. The sidebar should: (1) use 12px font with reduced contrast for secondary items, (2) collapse to icon-only at <768px viewport, (3) match the visual weight of `src/components/Sidebar.tsx` reference file. Reject if any of (1)-(3) is missing.
  MaxAttempts: 3
  Depends On: none

- [ ] Task: Add toon-shaded rendering to the WebGL canvas
  Verification-LLM: Read the diff. The renderer must implement: (1) cel-shaded outline using inverted-hull technique, (2) 3-tone quantization on diffuse, (3) no postprocessing pipeline (toon shading replaces it). Reject if any check fails.
  MaxAttempts: 5
  Depends On: Set up WebGL canvas scaffolding
```

### Both verifications (gate must pass both)

```markdown
- [ ] Task: Add a Jest test for the sidebar component
  Verification: `npx jest sidebar.test.js` exits 0
  Verification-LLM: The test must cover: (1) collapse behavior at <768px, (2) keyboard navigation through menu items, (3) ARIA labels on all interactive elements. Reject if any test scenario is missing.
  MaxAttempts: 3
  Depends On: Refactor the dashboard sidebar to use the Linear-style minimalist hierarchy
```

---

## Good vs. bad verifications

### Good — binary, runnable in 10 seconds

```markdown
Verification: `grep -q "function" src/api/auth.ts` exits 0
Verification: `npm test -- --grep="health"` exits 0 with "1 passed" in stdout
Verification: The string `## Health check` appears in `README.md`
Verification-LLM: Diff shows the new endpoint at `/health` returns JSON with `status: ok` field
```

### Bad — vague, subjective, untestable

```markdown
# Don't:
- "Make the login work"             # vague, not testable
- "Verify the API looks good"        # subjective
- Depends on the "Add endpoint" task  # doesn't match exact Task: string
- Verification: works correctly       # no command
- Verification-LLM: looks nice       # no rubric
```

The verification rule: if a human can't run a single bash command or evaluate a one-paragraph rubric and get a pass/fail answer in 10 seconds, rewrite the verification.

---

## Path hints for context hydration

When the `Task:` description contains file paths or extensions (e.g., `src/sidecar/db.py`, `*.tsx`, `src/api/`), the supervisor scopes the hydration block to those paths automatically. This keeps subagent prompts tight on large repos.

Examples of well-hinted tasks:
- `Update the Python sidecar to alter a NeonDB query in src/sidecar/db.py` → hydrates only `src/sidecar/` history
- `Refactor the Next.js dashboard components in src/dashboard/` → hydrates only `src/dashboard/` history
- `Write Jest tests for the new auth flow` → no path hints → hydrates whole repo (use sparingly)

Tasks without path hints still work but may have less-scoped hydration on large repos.
