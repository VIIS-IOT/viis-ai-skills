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
- Reports and dashboards
- Catalogs
- People
- Zones and notifications
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
| `viis.item.list` (`search`, `item_group`) | Items with stock UOM and `uoms[]` conversions | `item_code`, `item_id`, `product_id` |
| `viis.item-group.list`, `viis.uom.list` | Item groups, units | `item_group`, `uom` / `stock_uom` |
| `viis.plant.list`, `viis.subculture-position.list`, `viis.stage-template.list` (`search`) | Plants, subculture positions, stage templates | `plant_id`, `vietplants_subculture_position`, `state_id` |
| `viis.item-monthly.list` (`item_id`, `year`, `month`), `viis.item-monthly.effective` | Monthly KPI coefficients | `configId` |
| `viis.role.list`, `viis.job-title.list` | Roles (with sections), job titles | `iot_dynamic_role`, `position_id` |
| `viis.stock.customer.list`, `viis.stock.supplier.list`, `viis.stock.customer-group.list`, `viis.stock.supplier-group.list` | Buyers, suppliers and their groups | `customer`, `supplier`, `group` |
| `viis.project.list` | Projects | `project_id` in `viis.zone.create` |

The newer list tools take `page` and `limit` (default 50, max 200) and return `total`. Their rows are trimmed (no ERP bookkeeping fields, nested objects one level deep); use the matching `get` for the full record.

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
| `viis.task.material.save` / `.delete` | W (D) | Materials (vật tư) of a task: one row per item; an item already on the task is updated. Quantities in the row's `uom` (default the stock UOM); stored in the stock UOM. Refused on Verified tasks. |
| `viis.task.output.save` / `.delete` | W (D) | Outputs (sản lượng), same rules. For EnvPour tasks exp/draft > 0 auto-creates label requests, as on the web. Does not post stock. |
| `viis.task-template.create` / `.update` / `.delete` (D) | W | Task templates (công việc mẫu) with `items` / `production_tasks`; on update those lists replace the stored ones. System Users only, as on the web. |
| `viis.task-template.schedule.list` (R), `.schedule.link` / `.unlink` (W) | | Device schedules linked to a template (by schedule `name`). |
| `viis.genetics.chains` (`task_id`) | R | Phả hệ: the task, its outputs and the chain of tasks it received material from. |

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
Pest / disease reports (sâu bệnh): `viis.crop.pest.list` (`crop_id`, R), `.create` (`crop_id`, `label`, optional `state_ids`, `user_ids`), `.update` (lists replace the links), `.delete` (D).

## QC
| Tool | Notes |
|---|---|
| `viis.qc.wip.list`, `viis.qc.sample.list`, `viis.qc.final.list` (R) | Find `quality_control_id`. Filters: `status` (`Draft`/`Approved`/`Denied`), `task_id`, `assignee_id` (WIP/sample), `date_from`/`date_to` on the task start day. WIP = Kiểm chất lượng (not Passed), sample = Kiểm mẫu (Passed). |
| `viis.qc.wip.get` (also sample checks), `viis.qc.final.get` (R) | One record. |
| `viis.qc.subculture.work`, `.confirm`, `.confirm-batch`, `.revoke`, `.stock-entry` | Subculture QC. Confirm, revoke and stock-entry post or reverse stock ⚠. |
| `viis.qc.wip.submit`, `.approve`, `.approve-batch`, `.delete`, `.stock-entry` | Work-in-progress QC ⚠ on approve and stock-entry. |
| `viis.qc.final.submit`, `.submit-batch`, `.approve`, `.approve-batch`, `.delete` | Final QC ⚠ on approve. |
| `viis.qc.sample.approve`, `.delete` | Initial sampling QC. |
| `viis.qc.env-pour.approve` | See env tasks. |
QC times use `YYYY-MM-DD HH:mm:ss`.

## Labels (in tem)
`viis.label.list` (`print_status` pending/printed/reprinting, `task_id`, `product_id`) and `viis.label.get` (R) find the request `id`.
`viis.label.auto-create` (requests for a task), `bulk-create`, `update-quantity`, `report-damage`, `update-printed`, `print` ⚠, `bulk-print` ⚠. Printing marks labels printed and can reach a physical printer.

## Stock (kho)
| Tool | R/W |
|---|---|
| `viis.stock.summary`, `viis.stock.item.get`, `viis.stock.warehouse.list` | R |
| `viis.stock.entry.list` (`purpose`, `docstatus`, `task_id`, days) / `.get` | R |
| `viis.stock.receipt.list` / `.get`, `viis.stock.reconcile.list` / `.get`, `viis.stock.delivery.list` / `.get` (`docstatus`, days on `posting_date`) | R |
| `viis.stock.entry.save` → `.submit` ⚠ / `.cancel` ⚠ / `.delete` (draft, D) | W |
| `viis.stock.receipt.save` → `.submit` ⚠ / `.cancel` ⚠ / `.delete` (draft, D) | W |
| `viis.stock.reconcile.save` → `.submit` ⚠ / `.cancel` ⚠ / `.delete` (draft, D) | W |
| `viis.stock.delivery.save` → `.submit` ⚠ / `.cancel` ⚠ / `.delete` (draft, D) | W |
| `viis.stock.customer.create`, `viis.stock.supplier.create` | W |
| `viis.stock.ledger` (sổ cái), `viis.stock.report.in-out` (nhập xuất), `viis.item.vouchers` (history of one item) | R |
`save` creates a draft. `submit` posts it to the ledger, and `cancel` reverses a posted document. Posting date and time are UTC. Only drafts (docstatus 0) can be deleted.
Delivery note (phiếu xuất kho): `customer` from `viis.stock.customer.list`, each row needs `warehouse` or `set_warehouse`. **Submit recomputes totals from the body it gets: send the full document again** (all items and `add_taxes` / `other_charges` / `discount` / `paid_amount`). It refuses a future date, qty ≤ 0 and not enough stock.

## Reports and dashboards (R)
| Tool | Web page |
|---|---|
| `viis.report.inventory` (`date_from`, warehouse default "Kho thành phẩm") | Báo cáo tồn kho |
| `viis.report.garden-export` (`customer` optional) | Xuất vườn |
| `viis.report.production-results` (`grouping` week/month/year, ≤ 365 days) | Kết quả sản xuất |
| `viis.report.production-plan` (`plan_id`, `item_id`, `plant_id`) | Chi tiết kế hoạch sản xuất |
| `viis.report.kpi` (`department_id`, `month` YYYY-MM) | KPI tháng (large result) |
| `viis.dashboard.productivity` (`type` subculture / final-quality-inspection / env-pour, `group_by_time`) | Home productivity cards |
| `viis.dashboard.personnel` | Home personnel card |

## Catalogs (danh mục)
Each has `create` / `update` / `delete` (D): `viis.item`, `viis.item-group`, `viis.uom`, `viis.plant`, `viis.environment`, `viis.stage-template`, `viis.subculture-position`, `viis.department`, `viis.item-monthly`. They need a caller with a customer. Create always inserts, so check with the matching `.list` that the record doesn't already exist first. `viis.item-monthly.bulk-month` creates or updates one month for many items (web "Tạo tháng").

## People (nhân sự)
`viis.employee.create` ⚠ (creates a login account) / `update` (also `position_id`, `is_deactivated`) / `delete` (D); `viis.role.*`; `viis.score.*` (dates `YYYY-MM-DD`); `viis.attendance.*`. Lists: `viis.score.list`, `viis.attendance.list` (rows are employees with `atd[]`; `atd[].name` is the attendance id).
Job titles (chức danh): `viis.job-title.list` (R), `.create` / `.update` / `.delete` (D) — System Users only, as on the web.

## Zones and notifications
`viis.zone.create` (`label`, `project_id`), `.update`, `.delete` (D: detaches the zone from its project, as the web does). `viis.notification.list` (R) and `viis.notification.read` (`notification_id` from the list — not its `name` — or `all: true`).

## Devices, alarms, schedules, greenhouses
| Tool | R/W | Notes |
|---|---|---|
| `viis.device.search`, `.get`, `.presence`, `.telemetry.latest` | R | |
| `viis.device.telemetry.history` (`keys`, `from`/`to` ISO, ≤ 31 days, `agg` + `interval_minutes`, `limit` ≤ 5000) | R | Chart data. |
| `viis.device.command.send` | W ⚠⚠ | Fire-and-forget RPC to a real device. See SKILL.md. |
| `viis.alarm.list` (`days` ≤ 90), `viis.alarm.get` | R | |
| `viis.alarm.acknowledge` | W | |
| `viis.alarm.rule.list` (`device_id`) | R | Alarm rules of a device. |
| `viis.alarm.rule.create` / `.update` / `.enable` / `.delete` (D) | W ⚠⚠ | Rewrites the ThingsBoard device profile, shared by every device of that profile. |
| `viis.schedule.plan.list`, `.plan.get` (`plan_id`), `viis.schedule.running` | R | |
| `viis.schedule.plan.create` / `.update` / `.delete` (D), `viis.schedule.create` / `.update` / `.delete` (D) | W ⚠⚠ | Every write is pushed to the real gateway at once. `schedule.update` with `enable: 0` stops a running schedule on the device. Unsent fields keep their stored value. |
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
