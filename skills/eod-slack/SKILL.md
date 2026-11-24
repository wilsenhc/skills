---
name: eod-slack
description: Generates an end-of-day summary listing the ClickUp tasks worked during the day (from Clockify entries), for manual posting to Slack. Use when the user asks for an EOD message, end-of-day update, "my EOD", daily summary, or runs /eod-slack.
---

# EOD Slack Message

Generate the user's EOD summary and print it for them to paste into Slack.
Never post it anywhere.

List only the tasks that come back from Clockify for the day. No pending or
completed sections.

## Data sources

### Clockify — the day's entries

The Clockify MCP report tools (`get_detailed_report`, `get_summary_report`)
return 403 for this account (the Reports API requires the workspace-admin
role). Read entries from the REST API instead:

```bash
KEY=$(grep -oP '"x-api-key":\s*"\K[^"]+' ~/.config/opencode/opencode.jsonc)
curl -sS -H "x-api-key: $KEY" \
  "https://api.clockify.me/api/v1/workspaces/<workspaceId>/user/<userId>/time-entries?start=<ISO>&end=<ISO>&page-size=200"
```

If the grep yields `{env:CLOCKIFY_API_KEY}`, use `$CLOCKIFY_API_KEY` instead.
Each entry's `description` may end with a ClickUp task ref — `... | #<task-id>`.
That ref is the ClickUp task id.

### ClickUp — task details

For each unique ref, call `clickup_get_task`:

```
clickup_get_task({ task_id: "<ref>" })
```

It returns `name` (title), `status` (status name) and `url`.

## Steps

1. `get_current_user_profile` (Clockify MCP) → userId, workspaceId, timezone.
2. Range: today in that timezone, or the day/range the user names ("yesterday
   and today" → that whole range). ISO 8601 with offset, e.g.
   `2026-10-16T00:00:00-04:00`.
3. Curl the entries; collect the unique `#<id>` refs, oldest entry first.
4. Fetch each ref with `clickup_get_task` (one call per unique id). Omit refs
   that fail to resolve.
5. Check the user's assigned pending work (below) to decide the closing line.
6. Print exactly one fenced code block with the message, nothing else around it.

## Message structure

```text
**EOD Summary** - *[Date YYYY/MM/DD]*:
- **[Task Title]** ([Status]) - [Task URL]
- **[Task Title]** ([Status]) - [Task URL]
```

Rules:

- One `- **Title** (Status) - URL` line per task, using the status name ClickUp
  returns (e.g. `(ready for qa)`). No pending/completed sections.
- A task logged more than once appears once.
- No ClickUp refs found at all → indicate that no tasks were tracked that day.

## Closing line — assigned pending work

Pending statuses: `backlog`, `to do`, `blocked`, `triage`, `in progress`.

1. `clickup_resolve_assignees({ assignees: ["me"] })` → the user's id.
2. Read the ClickUp space id from `$CLICKUP_EOD_SPACE_ID` (ask the user if it is unset), then:
   `clickup_filter_tasks({ space_ids: ["<space-id>"], assignees: ["<id>"], statuses: ["backlog", "to do", "blocked", "triage", "in progress"] })`

If it returns zero tasks, append the closing line after a blank line, with
`<when>` set to `next week` when the current day (in the user's timezone, per
`get_current_time`) is a Friday, and `tomorrow` on any other day:

```text
I have no tasks assigned for <when>. Could you please assign me some tasks?
```

Otherwise, no closing line.
