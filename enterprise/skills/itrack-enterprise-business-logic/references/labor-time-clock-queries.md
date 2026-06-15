# Labor and Time Clock Reference

> ⚠️ **Time clock data (`activity`, `workclock` tables) is not currently available via the GraphQL API.** For raw clock queries, the **itrack-enterprise-mysql skill** may be available if you have admin-level access. See `labor.md` for verified query patterns. If the MySQL skill is not available, direct the user to built-in labor reports (check `reports` first).

## Column Name Reference

These column names are non-obvious and commonly hallucinated. Keep for reference regardless of query method:

| Table | Column | Notes |
|---|---|---|
| `activity` | `clockin`, `clockout` | NOT `starttime`, `endtime`, `intime`, `outtime` |
| `workclock` | `intime`, `outtime` | NOT `clockin`, `clockout`, `datein`, `starttime` |
| `workclock` | `userid` | Joins to `useraccount.useraccountid` — NOT `useraccount.userid` |
| `workclock` | — | No `workorderid` or `clocktype` columns — join through `job` to reach `workorder` |
| `activity` | `storeid` | Employee's **home store** — not WO store |

## Architecture — activity is Primary, workclock is a Subset

- **`activity`** = primary ledger — covers all employees, all time types. Full columns: `activityid`, `activitytypeid`, `userid`, `clockin`, `clockout`, `wagerateid`, `storeid`, `cost`.
- **`workclock`** = subset — WO job time only. Full columns: `workclockid`, `userid`, `jobid`, `storeid`, `intime`, `outtime`, `wagerateid`, `cost`.

**Employees without WO work have zero `workclock` records.** Always use `activity` as the base for attendance totals.

## The Double-Row Pattern — CRITICAL

Every clock-in writes **two rows** to `activity`:
1. `activitytypeid = 1` — attendance record (`involvement = 'Blocker'`)
2. The specific activity type (`involvement = 'Exclusive'` or `'Open'`)

**Filter to `activitytypeid = 1` for attendance totals** — summing all rows doubles every hour.

## Store Filter — Critical Distinction

- **Employee's home store** (`activity.storeid`): use for attendance totals, total hours clocked, and any query where you want all employees belonging to a store. Base for most labor summary queries.
- **WO's store** (`workorder.storeid` via join): use when the goal is specifically "all work performed on this store's WOs" regardless of which store the tech belongs to. Produces a different employee set.

## Total Hours Clocked (Attendance Basis)

> Not available via GraphQL API. If MySQL skill is available, filter `activity` to `activitytypeid = 1` scoped by `activity.storeid`. See `labor.md` — "Attendance Hours (Activity-Based)" query.

## WO Hours by Category

> Not available via GraphQL API. Category breakdown requires joining `activitytype.workordertypeid` to `workordertype` flags. See `labor.md` — "WO Hours by Category" query.

## Efficiency per Technician

> Clock-based efficiency is not available via the GraphQL API. `job.billinghours` is available via GraphQL and can be used for billing-hours-only analysis. For full efficiency (billed vs. clocked), the MySQL skill is required.

**Critical warnings — apply regardless of query method:**

- `billinghours` is the correct numerator — NOT `expectedhours`
- Store filter for labor reporting: scope by `activity.storeid` (employee home store), not `workorder.storeid`
- `billinghours` without a closed-WO gate is inflated — always filter `wo.closed = 'True'` for period metrics
- `SUM(j.expectedhours)` without proportional weighting produces wrong results — do not use as a period metric
- Shared job attribution: multiple techs on one job requires `billinghours × (tech_clock / total_clock)` — raw queries do not apply this automatically

### Confirmed Query Pattern (MySQL skill)

> If MySQL skill is available, drive from `workorder` to avoid full table scans on `workclock`. See `labor.md` — "Confirmed Pattern: Drive from workorder."

## Orgs Without Time Clock

Not all organizations use the time clock feature. If `workclock` and `activity` are empty or sparsely populated, the org likely enters labor hours manually on WO jobs rather than clocking in/out. In this case:

- `job.billinghours` is still populated — available via GraphQL
- Efficiency calculation is not possible (no actual clocked hours to compare against)
- Labor output can still be reported via `job.billinghours` grouped by worker group
- Always check whether clock data exists before attempting an efficiency query

## Forgotten Clock-Out Detection

> Not available via GraphQL API. If MySQL skill is available, see `labor.md` — "Forgotten Clock-Out Detection" query.

## activitytype involvement Values

| Value | Meaning |
|---|---|
| `Blocker` | Payroll attendance record (activitytypeid=1 only) |
| `Exclusive` | Specific activity — only one active per user at a time |
| `Open` | Can overlap with other activities — rarely used, typically system/legacy |

## Efficiency Concepts

**Per-WO efficiency**: `job.billinghours / actual_clocked_hours` — can exceed 100% when billed hours exceed clocked hours. `billinghours` available via GraphQL; clocked hours require MySQL skill.

**Per-employee efficiency** is not a standard report metric — it is customer-specific. If a customer-specific skill is present, follow its efficiency formula.

## Shared Job Hour Attribution Formula

```
Tech_attributed_billing_hours =
  job.billinghours × (tech_clock_hours / total_job_clock_hours)
```

Built-in labor reports apply this automatically. Raw clock queries do not.

## Billing Hours vs. Clocked Hours

- `job.billinghours` = hours billed (flag hours) — available via GraphQL
- `job.expectedhours` = flat-rate guide estimate (may be 0) — available via GraphQL
- `workclock.cost` = internal labor cost — MySQL skill only
- `job.laborcharge` = what customer was billed — available via GraphQL
- When "Bill Actual Hours" is on, `billinghours` = clocked — efficiency ratio is meaningless. Use `expectedhours / clocked_hours` instead.
