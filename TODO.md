# Task Backlog

> Tasks the loop will pick up on the next tick. Do not edit INPROGRESS.md or DONE.md by hand — the supervisor owns those files.

## Schema

Every task must follow this exact format. The supervisor parses line-by-line; sloppy formatting breaks the pipeline.

```markdown
- [ ] Task: <short, imperative description>
  Verification: <concrete, binary test — what command, file, or output proves success?>
  Depends On: <none | exact Task: string of another task>
```

## Example tasks

- [ ] Task: Add a /health endpoint to the API server
  Verification: `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health` returns `200` and the response body contains `{"status":"ok"}`
  Depends On: none

- [ ] Task: Write a Jest test for the /health endpoint
  Verification: `npx jest health.test.js` exits 0 and stdout contains `Tests: 1 passed`
  Depends On: Add a /health endpoint to the API server

- [ ] Task: Add a health-check section to the README
  Verification: The string `## Health check` appears in `README.md`
  Depends On: Add a /health endpoint to the API server
