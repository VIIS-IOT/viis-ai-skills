---
name: viis-mcp
description: How to work with VIIS / VietPlants farm data through the VIIS MCP server (tools named viis.* or viis_*). It covers reading and changing farming plan tasks (công việc), production plans (kế hoạch sản xuất), allocations (phân bổ), crops and stages, env tasks (Lô Stock, Hấp, Chiết rót), QC, labels, stock entries, catalogs, employees, devices, alarms and schedules, with the preview → confirm rule for every write. Use it whenever the user asks anything about their VIIS or VietPlants data, or asks to create, assign, update, approve or delete something there, even if they don't mention MCP — e.g. "hôm nay ai có việc gì", "giao việc pha môi trường cho tổ A", "tạo kế hoạch tháng 11", "tồn kho M06 còn bao nhiêu", "bật máy bơm nhà màng 2".
---

# VIIS MCP

The VIIS MCP server exposes a VIIS tenant's farm data as tools. The tools act **as the signed-in user**: they see only that user's customer data and use the same services as the web app, with the same validation. Treat every write as if the user clicked the button in the web app themselves.

- Production: `https://iot.viis.tech/api/v2/mcp`
- Your organization may also give you a test server. Writes there are safe to try. If you can't tell whether a connection is production or test, ask before any write: on production every write changes real farm records.

If no `viis` tools are available, the user needs to connect first. See `references/connect.md`.

## Tool names

The server names tools with dots (`viis.task.update.status`). Some clients show them with underscores (`viis_task_update_status`). They are the same tools. `references/tools.md` lists every tool by area, with read/write and the key arguments. Read it before using an area for the first time in a conversation.

## Reading data

1. Start broad, then narrow. The `*.summary` and `*.overview` tools (`viis.task.summary`, `viis.stock.summary`, `viis.crop.overview`, `viis.alarm.summary`) answer most "how are things" questions in one call.
2. Older list tools take `limit` (default 20, max 100; `viis.plan.list` max 50) and have no offset. The newer `*.list` tools (QC, labels, vouchers, catalogs, reports) take `page` + `limit` (default 50, max 200) and return `total`. If a list hits the limit, say that the result may be cut off and filter it down (status, assignee, days) instead of claiming a total.
3. Use `*.get` for one record once you have its id. Ids are UUIDs, from a list result or from the user.
4. Answer in the user's language (usually Vietnamese), with names and labels the user knows (task label, employee name, lot name), not raw UUIDs. Show quantities with their unit. Env lots are stored in base units (ml or g) with a conversion factor, so convert 300000 ml to "300 Lít" when the item's active unit is Lít.

## Writing data — preview, then confirm

Every write tool takes `confirm` (boolean, default `false`).

- **`confirm` false or absent → preview.** The server validates the arguments and your access, and returns what would change, with `preview: true`. Nothing is saved.
- **`confirm: true` → executes.** Send the JSON boolean `true`. Some tools accept the string `"true"` and others don't, so always use the boolean.

Follow this sequence for every write:

1. Gather the arguments. Look up ids with read tools (`viis.department.list`, `viis.task.template.list`, `viis.environment.template.list`, `viis.crop.state.list`, `viis.employee.list`, `viis.item.list`, `viis.qc.wip.list`, `viis.label.list`, `viis.stock.*.list`; see the Lookups table); never guess an id or an enum value.
2. Call the tool **without** `confirm` and read the preview. If it reports an error (missing field, wrong status, user not in your customer, shortage), fix the arguments or explain the error to the user. Do not retry with `confirm: true` hoping it passes.
3. Show the user what will change in plain words, e.g. "Tạo 1 công việc 'Lô Stock M06 - 80 lít' ngày 22/10 cho Nguyễn Văn A, năng suất 80 Lít". Ask for a clear yes.
4. Only after that yes, call again with the same arguments and `confirm: true`. Report the result, including the new record's id or label.

Why the preview matters: writes go through the real services. They send push notifications to the assignee, post stock entries, change allocations that other people plan against, and some cannot be undone (approvals, stock submits, deletes). The user approves the specific change, not "go ahead with whatever". A new or changed request needs a new preview.

For a batch (several tasks, a plan's details), preview the whole batch, confirm once for that batch, then execute. Stop at the first failure and report what was and was not done.

### Rules that keep data correct

- **Statuses are exact strings.** Tasks: `Plan`, `Pending`, `In progress`, `Done`, `InUse`, `Waiting`, `Stopped`. `Verified` can't be set by a tool; only task validation sets it. A Verified task can't change status, and Done or Verified tasks can't be deleted. Status filters on `viis.task.list` are exact matches too (older servers' description suggests "In Progress / Completed"; those values don't exist). Plans take `plan`, `assigned`, `completed`; allocations take `planned`, `in_progress`, `completed`, `cancelled`; crops take `In progress`, `Done`.
- **Dates carry the Vietnam offset.** For task and env-task times, send ISO with an offset: `2026-10-22T07:00:00+07:00`. A bare `2026-10-22` is read as midnight UTC, which is 07:00 in Vietnam and can land on the wrong day. Plans, crops and scores take `YYYY-MM-DD`; QC takes `YYYY-MM-DD HH:mm:ss`; stock posting date and time are UTC.
- **Assignees must belong to the user's customer.** Use ids from `viis.employee.list`. The preview rejects anyone else.
- **Use `viis.task.update.status` to change a status.** Don't use a generic update for this. It runs the same checks as the web status menu: steam-pot stock lot checks, and the Done stamp for env tasks.
- **Label test data.** When the user is only trying the tools, start labels or descriptions with `[TEST]` and offer to delete the records afterwards. Prefer a test server for experiments.

### Ask twice: high-impact writes

For these, show the preview and get an explicit yes for that exact action, even if the user already said "do it":

- `viis.schedule.*` writes and `viis.alarm.rule.*` writes: they reach the real gateway or the ThingsBoard device profile immediately. Never create a "test" schedule or rule on a real device.
- `viis.device.command.send`: sends a ThingsBoard RPC straight to a real device (pump, valve, light). It is fire-and-forget: no acknowledgement, and `method` and `params` aren't validated. Never send one the user didn't ask for by device and action. Never "test" a device command. After sending, say that you cannot confirm the device acted; the user should check telemetry (`viis.device.telemetry.latest`) or the device itself.
- Deletes (`*.delete`, `viis.plan.detail.delete-range`, allocation or assignment deletes). They are marked destructive.
- Approvals and postings: `viis.qc.*.approve*`, `viis.qc.*.stock-entry`, `viis.qc.subculture.confirm*` / `revoke`, `viis.stock.*.submit` / `cancel` (including delivery notes, xuất kho). These post or reverse stock entries but are **not** marked destructive by the server.
- `viis.employee.create`: creates a login account.
- `viis.label.print` / `bulk-print`: they mark labels printed and can trigger physical printing.

## Errors

| Response | Meaning | What to do |
|---|---|---|
| 401 / "authorization required" | The token expired or the web session is gone. An access token lasts 1 h and is refreshed automatically; the connection lasts 30 days idle, 90 days at most. | Ask the user to reconnect (re-authenticate) the VIIS connector. |
| Permission / section error | The user's role lacks that section in the web app. | Explain which permission is missing. Don't retry with other tools to get around it. |
| "needs a customer" on a write | Tenant-wide System users without a customer can read but not write tasks or catalogs. | Ask the user to use a customer account. |
| "Only a System User (customer admin) can do this" | Task templates and job titles are admin-only on the web too. | Ask a customer admin to do it. |
| Write refused on a read-only connection | The connection was granted only `mcp:read`. | Tell the user. Writes need a connection authorized with write access. |
| Rate limit | 60 calls/min and 1000/day per connection. | Slow down and batch reads with summary tools. |
| Tool missing that should exist | The server's tool list changed. Clients cache it, and the server sends no change notice. | Ask the user to reconnect the connector. |

## Privacy

Results can contain staff names, phone numbers and emails. Use them only to answer the user's question, and don't paste lists of personal data into other tools or files unless the user asks. Device tokens and secrets come back as `[hidden]`; never try to recover them.
