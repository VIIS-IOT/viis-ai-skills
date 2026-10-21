# VIIS MCP tool map

R = read-only. W = write (preview with `confirm` false, execute with `confirm: true`). D = destructive. ⚠ = high impact; ask twice (see SKILL.md).

Names are given with dots. A client may show `viis.plan.detail.batch-save` as `viis_plan_detail_batch-save`. Each tool's schema in the client lists its arguments, so read it before calling. This file gives you the map and the workflows.

## Contents
- Overview and summaries
- Lookups (ids for write tools)
- Tasks (công việc)
- Env tasks: Lô Stock → Hấp → Chiết rót
- Production plans and allocation
- Crops and stages
- QC
- Labels
- Stock
- Catalogs
- People
- Devices, alarms, schedules, greenhouses
- Admin-only
- Workflows

## Overview and summaries (R)
| Tool | Use |
|---|---|
| `viis.owner.get` | Public VIIS company info. It does not show the signed-in user. |
| `viis.task.summary` | Task counts by status. |
| `viis.stock.summary` | Low-stock items. |
| `viis.crop.overview` | Crops (lô sản xuất) with ids. |
| `viis.alarm.summary`, `viis.device.summary`, `viis.schedule.summary`, `viis.notification.summary` | Quick health views. `notification.summary` takes `days` (max 365). |
| `viis.employee.list` | Staff of the customer. `name` is the customer user id used as `assigned_to`. |

## Lookups (ids for write tools) (R)
Use these to find ids; never guess them. On older servers they may be missing: if a tool doesn't exist, take the ids from `viis.task.get` on a similar task, or ask the user.

| Tool | Returns | Used as |
|---|---|---|
| `viis.department.list` (`search`) | Departments (Phòng môi trường, Phòng cấy chuyền…) | `department_id` |
| `viis.task.template.list` (`task_type`, `environment_template_id`, `search`) | Task templates, e.g. "Lô Stock M06 - 80 lít" | `template_id` |
| `viis.environment.template.list` (`search`) | Environments M06, D1… | `environment_template_id` |
| `viis.crop.state.list` (`crop_id`) | Stages of a crop: `name` + `state_template_id` | `state_id` / `farming_plan_state`, `state_template_id` |

## Tasks (công việc)
| Tool | R/W | Notes |
|---|---|---|
| `viis.task.list` | R | Filters: `status` (exact: `Plan`, `In progress`, `Done`, …), `assigned_to`, `task_type`, `date_from` / `date_to` (Vietnam days `YYYY-MM-DD`; tasks whose start–end overlaps them, e.g. today), `limit`. `status_counts` covers everything that matches, `tasks` is capped by `limit`. |
| `viis.task.get` | R | One task with `farming_plan_state`, `crop_id`, `template_id`, `department_id`, `environment_template_id` (+ labels), `outputs` and `materials` with `exp_in_uom` / `draft_in_uom` in the active unit. |
| `viis.task.create` | W | Simple task in `Plan`. For env tasks use `viis.env.task.create`. |
| `viis.task.update` | W | Edit fields. |
| `viis.task.update.status` | W | The only right way to change a status. It runs the web status-menu logic. `Verified` can't be set. |
| `viis.task.assign` | W | The web "Phân bổ công việc" table: one row per user per day; `template_id`, `department_id`, ISO dates with `+07:00`. Assignees get a push notification. |
| `viis.task.delete` | W D | Not allowed for Done or Verified tasks. |

## Env tasks: Lô Stock → Hấp → Chiết rót
Task types: `EnvStock` (Lô Stock, lô môi trường), `EnvSteamPot` (Hấp, steam pot), `EnvPour` (Chiết rót, pour).

| Tool | R/W | Notes |
|---|---|---|
| `viis.env.task.create` | W | Requires `farming_plan_state` and `task_type`. Materials go in `item_list` and outputs in `prod_quantity_list`, quantities in the active UOM with `active_conversion_factor` (Lít = 1000 × ml). |
| `viis.env.task.update` | W | Same shape. `task_transfers` links sources; `replace_task_transfers: true` replaces the whole set. |
| `viis.env.task.delete` | W D | |
| `viis.qc.env-pour.approve` | W ⚠ | Approves a pour (→ Verified). It posts stock and is not reversible from MCP. |

The chain rules are enforced by the server and shown in the preview:
- A steam pot (`EnvSteamPot`) links to an `EnvStock` lot through `task_transfers`.
- A pour (`EnvPour`) links to a Done steam pot.
- A pour needs both materials and outputs.
- At execute time, the server rejects a quantity larger than what is left in the source: "Không đủ số lượng … Còn lại: X, cần dùng: Y". Explain the shortage to the user; don't shrink the quantity on your own.
- Marking an env task Done stamps `end_date` and `task_progress` (hours). If it is finished before its planned start, `end_date` = the planned start and progress = 0.

## Production plans and allocation (kế hoạch sản xuất, phân bổ)
| Tool | R/W | Notes |
|---|---|---|
| `viis.plan.list`, `viis.plan.get` | R | `plan.get` returns `yield_outputs[]` with live allocations, capped by `allocation_limit` (default 20; `allocation_count` is the real number). Narrow large plans with `yield_output_id` or `environment_code`. |
| `viis.plan.create` | W | `plant_id` and `label`; dates `YYYY-MM-DD`. It always creates a new plan; use `viis.plan.update` to change one. |
| `viis.plan.update`, `viis.plan.status` | W | Status: `plan`, `assigned`, `completed`. |
| `viis.plan.delete` | W D ⚠ | |
| `viis.plan.detail.generate` / `generate-all` | W | Build the daily detail rows for one or all yield outputs. `regenerate: true` rebuilds them. |
| `viis.plan.detail.update` / `bulk-fill` / `batch-save` | W | Edit daily rows. |
| `viis.plan.detail.delete-range` | W D ⚠ | |
| `viis.plan.assign` | W | Assign daily work (`assignments[]`: yield output, date, `room_type`, quantity). The preview runs the capacity check. `notify_users` defaults to false. |
| `viis.plan.assignment.update` / `.delete` | W (D) | |
| `viis.plan.allocate` | W | Allocate yield output quantity to a user, optionally linked to a task. Over-allocation is allowed, as on the web, and the preview shows what remains. |
| `viis.plan.allocation.status` / `.delete` | W (D) | Allocation status: `planned`, `in_progress`, `completed`, `cancelled`. |

## Crops and stages (lô sản xuất, giai đoạn)
`viis.crop.create` / `update` / `delete` (D); `viis.crop.stage.upsert` / `update` / `delete` (D). Crop status: `In progress` or `Done`.

## QC
| Tool | Notes |
|---|---|
| `viis.qc.subculture.work`, `.confirm`, `.confirm-batch`, `.revoke`, `.stock-entry` | Subculture QC. Confirm, revoke and stock-entry post or reverse stock ⚠. |
| `viis.qc.wip.submit`, `.approve`, `.approve-batch`, `.delete`, `.stock-entry` | Work-in-progress QC ⚠ on approve and stock-entry. |
| `viis.qc.final.submit`, `.submit-batch`, `.approve`, `.approve-batch`, `.delete` | Final QC ⚠ on approve. |
| `viis.qc.sample.approve`, `.delete` | Initial sampling QC. |
| `viis.qc.env-pour.approve` | See env tasks. |
QC times use `YYYY-MM-DD HH:mm:ss`.

## Labels (in tem)
`viis.label.auto-create` (requests for a task), `bulk-create`, `update-quantity`, `report-damage`, `update-printed`, `print` ⚠, `bulk-print` ⚠. Printing marks labels printed and can reach a physical printer.

## Stock (kho)
| Tool | R/W |
|---|---|
| `viis.stock.summary`, `viis.stock.item.get`, `viis.stock.warehouse.list` | R |
| `viis.stock.entry.save` → `.submit` ⚠ / `.cancel` ⚠ | W |
| `viis.stock.receipt.save` → `.submit` ⚠ / `.cancel` ⚠ | W |
| `viis.stock.reconcile.save` → `.submit` ⚠ / `.cancel` ⚠ | W |
`save` creates a draft. `submit` posts it to the ledger, and `cancel` reverses a posted document. Posting date and time are UTC.

## Catalogs (danh mục)
Each has `create` / `update` / `delete` (D): `viis.item`, `viis.uom`, `viis.plant`, `viis.environment`, `viis.stage-template`, `viis.subculture-position`, `viis.department`, `viis.item-monthly`. They need a caller with a customer. Create always inserts, so check that the record doesn't already exist first.

## People (nhân sự)
`viis.employee.create` ⚠ (creates a login account) / `update` / `delete` (D); `viis.role.*`; `viis.score.*` (dates `YYYY-MM-DD`); `viis.attendance.*`.

## Devices, alarms, schedules, greenhouses
| Tool | R/W | Notes |
|---|---|---|
| `viis.device.search`, `.get`, `.presence`, `.telemetry.latest` | R | |
| `viis.device.command.send` | W ⚠⚠ | Fire-and-forget RPC to a real device. See SKILL.md. |
| `viis.alarm.list` (`days` ≤ 90), `viis.alarm.get` | R | |
| `viis.alarm.acknowledge` | W | |
| `viis.schedule.plan.list`, `.plan.get`, `viis.schedule.running` | R | No tool starts or stops a schedule. If asked, say it must be done in the web or mobile app. |
| `viis.greenhouse.list`, `.get` | R | |

## Admin-only (Root and System users)
`viis.device.virtual.list`, `viis.vpn.member.list` (R, `page` / `pageSize`). Other users don't see them.

## Workflows

**"Giao việc pha Lô Stock M06 80 lít cho X ngày 22/10"**
1. `viis.employee.list` → X's id. `viis.crop.overview` → the crop (the environment batch). `viis.crop.state.list` → the stage ("Giai đoạn Pha Lô Stock") id. `viis.department.list` → the department. `viis.task.template.list` (`task_type: EnvStock`, `search: M06`) → the template.
2. Optionally `viis.plan.list` → the production plan covering that day (`production_plan_id`).
3. `viis.task.assign` without confirm: `crop_id` + `state_id`, `department_id`, one row with X, `2026-10-22T07:00:00+07:00` → `2026-10-22T17:00:00+07:00`, `template_id`, `exp_quantity` in the template's unit (Lít), `task_type: EnvStock`.
4. Show the preview, get a yes, then repeat with `confirm: true`. X gets "Bạn vừa được giao công việc …" on the app. Env stock tasks don't count against the plan's yield allocation.

**"Hôm nay ai có việc gì?"**
`viis.task.list` with `date_from` = today (Vietnam date) and `limit: 100`. Group by `assigned_name`. If `tasks` hits the limit, say so and use `status_counts` for totals.

**"Kế hoạch tháng 11 còn bao nhiêu chưa phân bổ?"**
`viis.plan.list` → `viis.plan.get` → for each yield output, compare `yield_value` with the sum of its allocations. Report by output and unit.

**"Đánh dấu xong việc Hấp R3-2"**
`viis.task.list` / `viis.task.get` to find it → `viis.task.update.status` (`Done`) preview → yes → confirm. If the preview reports a stock lot shortage, report it; don't change quantities.
