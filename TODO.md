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

---

## Image-based verification

For visual work (UI, design-system conformance, mockup fidelity, visual regression), reference an image in the verification. The supervisor and subagents both have the `Read` tool, which supports image inputs — they can actually see the image, not just check it exists.

### Pattern A — Bash: file exists + dimensions check

Use when the image is ground truth (a baseline screenshot, a reference design) and you want a fast, cheap check that the asset is on disk:

```markdown
- [ ] Task: Add the dashboard hero illustration
  Verification: `test -f assets/illustrations/hero.svg && file assets/illustrations/hero.svg | grep -q "SVG"` exits 0
  Depends On: none
```

Bash can't "see" the image — it only verifies presence and format. Pair with an LLM judge if visual fidelity matters.

### Pattern B — LLM judge with image reference

Use when the work must match a visual target (mockup, baseline, reference design). The LLM judge subagent reads both the diff AND the image:

```markdown
- [ ] Task: Refactor the dashboard hero to match the v2 mockup
  Verification-LLM: Read both `git diff HEAD` and the mockup at `assets/mockups/dashboard-hero-v2.png`. The implementation must match: (1) headline placement top-left with 48px margin, (2) CTA button at bottom-right with primary brand color, (3) background gradient stops at the angles shown in the mockup. Reject if any visual mismatch is obvious.
  Depends On: none

- [ ] Task: Verify the new login screen matches the baseline
  Verification-LLM: Render `src/screens/Login.tsx` (or run `npm run storybook` and capture) and visually compare to the baseline at `assets/baselines/login.png`. Report any pixel-level differences in: input field borders, button color, spacing between fields. Reject if any difference is visible without zoom.
  Depends On: Add login screen scaffold
```

### Pattern C — visual regression suite

For projects with a screenshot diff tool (Percy, Chromatic, Playwright snapshots), wrap the tool in a Verification command and reference the baseline path:

```markdown
- [ ] Task: Update the pricing page layout
  Verification: `npx playwright test pricing.spec.ts --update-snapshots` exits 0 AND the snapshot diff at `tests/__snapshots__/pricing.spec.ts-snap.png` matches the baseline at `assets/baselines/pricing.png` (no pixel diff > 1% area)
  Depends On: none
```

### Path conventions for images

Put reference images somewhere predictable so the supervisor and subagents can find them:

| Asset type | Suggested location |
|------------|-------------------|
| Design mockups | `assets/mockups/` |
| Visual baselines (regression) | `assets/baselines/` |
| Reference screenshots | `assets/references/` |
| Brand assets (logos, etc.) | `assets/brand/` |

Add these to `.gitignore` only if they're generated; track them in git if they're the source of truth that verifications depend on.

### What works / doesn't work

| Verification mode | Image support |
|-------------------|---------------|
| `Verification:` (bash) | File existence, dimensions, format check — no visual judgment |
| `Verification-LLM:` (judge) | Full visual judgment — reads image + diff, evaluates against rubric |
| `Verification-LLM:` (judge) for visual regression | Works only if you can render the screen and provide both baseline + current screenshot |

The LLM judge approach is the only one that does actual visual reasoning. Budget accordingly — each visual judgment is an LLM call.

---

## Generating TODO.md with Claude Code

You don't have to hand-write every task. Ask Claude Code to read your spec, codebase, or design doc and produce a well-shaped `TODO.md` for you. This is the fastest way to bootstrap a backlog.

### Template prompt — from a spec doc

```
Read SPEC.md (or describe your feature/goal here) and produce TODO.md
with 8-15 tasks that fully implement it.

Each task must follow the schema at the top of TODO.md exactly:
- Task: short, imperative description
- Verification: a bash command a human could run in 10s
- Verification-LLM: a rubric for visual/architectural checks (optional)
- MaxAttempts: only set if a hard cap is needed
- Depends On: exact Task: string of another task, or "none"

Rules:
- Each task should be completable in under 15 minutes
- Verification must be binary — exit 0 + expected output, not "looks good"
- Order tasks so dependencies form a DAG: foundation → features → polish
- Use [RETRY] prefix only for tasks being manually requeued (leave blank)
- Leave 1-2 blank lines between tasks for readability

After writing, list any tasks you couldn't shape properly and explain why.
```

### Template prompt — from an existing codebase

```
Look at the current state of this repo and produce TODO.md with 5-10
tasks that would meaningfully improve it. Examples: missing tests, TODO
comments in source, lint warnings, deprecation notices.

For each task:
- Reference the specific file path or symbol you're targeting (gives
  the supervisor path hints for context hydration)
- Write a Verification: command that proves the task is done
- Set MaxAttempts: 1 for trivial mechanical work, otherwise leave unset
```

### Template prompt — from a design mockup

```
Look at the mockup at assets/mockups/main.png and produce TODO.md with
the implementation tasks needed to build it.

For each UI region in the mockup, create one or more tasks. Use
Verification-LLM: with rubric referencing the mockup file for any task
that's primarily visual.
```

### Iterating on a generated TODO.md

Generated backlogs are first drafts. After Claude Code writes `TODO.md`, ask for targeted refinements:

```
Reorder TODO.md so authentication-related tasks come before
dashboard tasks. The auth foundation needs to land first.

Rewrite the Verification: for the "Add login form" task to actually
test that login succeeds with valid credentials, not just that the
form renders.

Add a task to add Storybook stories for the new components, with
Verification: `npx storybook test` exits 0.

Split the "Build user profile page" task into 3 smaller tasks
(profile header, profile edit form, profile settings). Each must have
its own Verification.
```

### What good generated output looks like

A well-shaped `TODO.md` from Claude Code will have:

- 5-20 tasks (more than 20 means the spec was too coarse — split it)
- A clear dependency chain forming a DAG (no cycles, every dependency in the same file)
- Every `Verification:` ending in "exits 0" or "appears in <file>" or similar binary assertion
- 1-3 `Verification-LLM:` tasks for genuinely visual work, not slapped on every task
- Task descriptions that name specific files, functions, or symbols (gives hydration hints)

### Anti-patterns in the generation prompt

Things that produce bad `TODO.md`:

- ❌ "Write TODO.md for a SaaS app" — too vague, generates fluff
- ❌ "Make 50 tasks" — generates padding to hit the count
- ❌ Asking for "all the tests" without scope — generates tautological tasks
- ❌ Skipping the schema reference — Claude invents its own format

Always: scope the work, reference the schema, give the LLM the spec/codebase to read, ask for binary verifications explicitly.
