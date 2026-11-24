---
name: new-task
description: "Start work on a new task: assemble the task brief (ClickUp MCP fetch when connected, otherwise user input), rename the session, investigate the code paths, grill the solution, and deliver a full implementation plan written to ISO/IEC/IEEE 29148:2018. Laravel projects; explicit invocation only."
disable-model-invocation: true
metadata:
  opencode/slash: "true"
  opencode/autoinvoke: "false"
---

# New Task

Turn a bare task reference into a full implementation plan, the same way every time. Do not start investigating before the task brief is assembled.

## Step 0 — Assemble the task brief

| Field | Required | Source |
|---|---|---|
| Task URL | No | User |
| Task ID | No | Extracted from the URL, or the ClickUp task's ID |
| Task name | Yes | ClickUp MCP; user when it is unavailable |
| Task type | Yes | Always ask: Bug / Feature / Improvement / Other |
| Task description | Yes | ClickUp MCP; user when it is unavailable |

1. Reuse anything already in the invocation message (URL, ID, name, type, description).
2. Extract the Task ID from the URL when one is given: prefer a `LETTERS-123` token (e.g. `CU-123`); otherwise the last path segment of a `/t/...` path (`.../t/<list-id>/<task-id>` → `<task-id>`). Never guess from other URL shapes.
3. Ask for what is still missing with your environment's question tool (OpenCode: `question`; Claude Code: `AskUserQuestion`; otherwise a plain numbered chat prompt): the URL or ID when absent, and always the Task type. If there is still no URL or ID, treat it as a manual task — ask for the name and description in chat instead of fetching.
4. When a URL or ID is available, fetch from the ClickUp MCP:
   - Task: `clickup_get_task` with `include: ["description"]` (in OpenCode call it through the code-mode catalog as `tools.clickup.clickup_get_task`). The default summary truncates the description at 10k chars; request other sections (`attachments`, `subtasks`, …) only when the investigation needs them.
   - Comments: always `clickup_get_task_comments` too (paginate with `start`/`start_id` when more exist) and fold them into the task context — bug reports often keep reproduction steps there.
   - If any fetch fails (server not connected, not authorized, `RATE_LIMIT_EXCEEDED`), say so and fall back to asking for the name and description. Never block the task on ClickUp.
   - Read only. Never update, comment on, or delete anything in ClickUp.
5. Rename the session only after every field above is filled, using your agent's session rename tool (OpenCode: `session_rename` — use it rather than only printing the title; if the environment truly has no such tool, print the intended title). The title is always `CU-<task id>: <Task name>`, e.g. `CU-1234: Fix login redirect` — prefix a raw ClickUp id with `CU-` (`<task-id>` → `CU-<task-id>`), keep an id that already carries the prefix as-is, and prefer the task's `custom_id` when ClickUp returns one, otherwise the ID extracted from the URL. When no Task ID exists at all, use the task name alone.
6. Print a one-line brief (`CU-<task id> — <Task name> [<status>]`) and start. No confirmation gate.

## Step 1 — Investigate

This workflow targets Laravel projects. If the project has no `artisan`, stop and tell the user.

1. Analyze the Task Description.
2. Look at the code paths where the task lands.
3. Run a full `php artisan migrate:fresh --seed` and check for errors, repeating it on failure.
   - Guard: run it only when the database looks local — `APP_ENV=local` and a `localhost` / `127.0.0.1` / lerd service DB host. Otherwise stop and ask before wiping anything.
4. Before the first Pest run of the session — `php artisan test`, `vendor/bin/pest`, parallel or not — take lerd snapshots of the site database and the testing database, named `pre-pest-<Task ID>` (see "Testing-database hygiene"). Never run Pest when the snapshot failed.
5. Bug: replicate the issue with the `chrome-devtools` MCP. Feature / Improvement / Other: use the same step to explore the current behavior of the affected area. If `chrome-devtools` is unavailable, say so and continue.
   - Log into the app for browser reproduction and verification with the `DEFAULT_USER` / `DEFAULT_PASSWORD` values from the project's `.env` file(s). If that user lacks the role(s) the flow needs, assign them first through the project's role API (`assignRole()`; pharmacy-scoped roles also need the pharmacy id) and note the change in the findings.
   - Whenever a pharmacy has to be chosen — the UI context, the scope for an assigned role, the data you reproduce with — default to **pharmacy ID 6**, the most fleshed out by far (the default user is already a member of it), and switch the context in the app's pharmacy/module switcher.
6. Document the findings and potential causes in the conversation.
7. Propose a solution. When the solution leaves anything worth stress-testing, load the `grilling` skill and interview the user before finalizing. Always ask the grilling questions through your environment's question tool (OpenCode: `question`; Claude Code: `AskUserQuestion`), one round per call — never just type the questions into chat; use plain chat only if the environment has no such tool.
8. Deliver the full implementation plan following ISO/IEC/IEEE 29148:2018 (see "Implementation plan format"): the approach and ordered changes, with each requirement identified and verifiable, and traced to its implementation steps, verification method, and acceptance criteria.
9. At the end of the session, remove the extra testing databases the test runs created — every `<testing DB>_test_*` leftover — and report what was dropped (see "Testing-database hygiene").

The deliverable is the implementation plan, in chat. Stop there — do not start implementing and do not write plan files unless asked.

## Implementation plan format (ISO/IEC/IEEE 29148:2018)

Write the plan as a requirements specification for the change, and keep the implementation half traceable to it:

1. **Introduction** — purpose, scope (what the change covers and excludes), definitions, references (task URL/ID, code paths).
2. **Overall description** — current behavior, affected users and roles, assumptions, constraints, dependencies.
3. **Specific requirements** — one unique ID per requirement, one verifiable statement each: `REQ-###` functional, `NFR-###` performance/security/quality attributes, `IF-###` interfaces (only where relevant).
4. **Implementation approach** — ordered changes, each mapped to the requirement IDs it satisfies.
5. **Traceability and verification** — a table mapping every requirement to its change step(s), verification method (test, inspection, analysis, demonstration), and acceptance criteria, plus validation against the task's original acceptance criteria.

Requirement rules (Clause 5): each requirement is necessary, appropriate, unambiguous, complete, singular, feasible, verifiable, correct, and conforming; the set as a whole is complete, consistent, feasible, comprehensible, and validatable. Use `shall` for binding requirements, `should` for recommendations, `may` for permissions — never `will`. If a tester could not write a test that fails when a requirement is unmet, rewrite it.

## Testing-database hygiene (lerd)

The site database is `DB_DATABASE` in `.env`; the testing database is `DB_DATABASE` in `.env.testing`.

- **Snapshot before Pest.** Snapshot both databases with lerd's `db` tool, `snapshot` action, `name: "pre-pest-<Task ID>"` and `database:` set per database — lerd appends the timestamp itself and keeps manual snapshots out of retention. In OpenCode call it as `tools.lerd.db({ action: "snapshot", name: "pre-pest-<Task ID>", database: "<db>" })`; CLI: `lerd db:snapshot "pre-pest-<Task ID>" --database <db>`. Check `db` `snapshots` (CLI: `lerd db:snapshots`) first and skip when this session already took a `pre-pest-<Task ID>-*` set.
- **Clean up at the end.** `db` `list` (CLI: `psql -w -l`) shows the parallel-test leftovers named `<testing DB>_test_<n>`. Laravel drops them after clean runs, so any that remain are crashed-run leftovers. Drop each one, keeping the `_testing` database itself and every snapshot:

  ```bash
  psql -w -c 'DROP DATABASE IF EXISTS "<name>";'          # lerd shim, connects to the maintenance DB
  # fallback:
  podman exec lerd-postgres psql -U postgres -d postgres -c 'DROP DATABASE IF EXISTS "<name>";'
  ```

  For a MySQL/MariaDB project use the equivalent client. Report what was dropped.

## If the ClickUp MCP is not connected

- OpenCode: add this to the global config `mcp` block, then authenticate:

  ```jsonc
  "clickup": { "enabled": true, "type": "remote", "url": "https://mcp.clickup.com/mcp" }
  ```

  `opencode mcp auth clickup` opens the OAuth flow (`opencode mcp list` shows status).
- Claude Code: `claude mcp add --transport http clickup https://mcp.clickup.com/mcp`, then `/mcp`.
- Without the MCP, Step 0 simply asks for the name and description.
