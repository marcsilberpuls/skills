# Refine Project — Reference

## Overview

This skill handles post-kickoff project refinement: computing the available budget, shaping weekly allocations for team members, and writing assignments to Everhour. It operates on **one project at a time**.

**When to use:** After a project is provisioned (via `/provision-project`) and the PM has a clear picture of team, timeline, and scope.

**When NOT to use:** For weekly cross-project planning, use `/plan-week` instead.

---

## Step 1: Parse the request

Extract from the PM's message:

| Field | Required | Example | Default |
|-------|----------|---------|---------|
| Project name or slug | Yes | "Obi", "obi", "Obi Marked Detail Pages" | -- |
| Team members | No | "Kathi and Marc" | Ask in Step 3 |
| Duration | No | "2 weeks starting March 27" | Ask in Step 3 |
| Budget overrides | No | "budget is actually 15k" | Read from project-meta.json |

### 1a. Resolve the project

Match the PM's input against directories under `projects/2-active/*/`:

1. **Exact slug match** -- `projects/2-active/<input>/` exists
2. **Fuzzy name match** -- read `project-meta.json` in each active project, match on `project_name`
3. **Multiple matches** -- ask: "Did you mean '[Project A]' or '[Project B]'?"
4. **Zero matches** -- check `projects/1-pitching/` and `projects/3-archive/`. If found there, explain status. If not found anywhere: "No project found matching '[input]'. Run `/provision-project` first."

### 1b. Load project metadata

Read `projects/2-active/<slug>/planning/project-meta.json`:

```json
{
  "project_slug": "obi",
  "project_name": "Obi",
  "client_name": "OBI Corporate Center GmbH",
  "everhour_project_id": "as:1212148517388873",
  "total_budget_eur": 13100.0,
  "budget_model": "total",
  "status": "active"
}
```

Key fields:
- `total_budget_eur` -- total contract budget
- `everhour_project_id` -- needed for API assignment writes
- `budget_model` -- "total" (one-time) or "rolling" (recurring monthly)

If `everhour_project_id` is empty, resolve it from the API project list (Step 6a).

---

## Step 2: Budget math (not skippable)

### 2a. Read known costs

**Total budget:** from `project-meta.json` -> `total_budget_eur`

**Already tracked:** from `phase-budget-status.json` -> `summary.total_tracked_eur` (if file exists). Alternatively, sum tracked hours from Everhour budgets JSONL at `projects/2-active/<slug>/running/finance/latest-truth.json`.

If neither file exists or tracked is zero, ask PM: "How much has been tracked/spent so far on this project?"

### 2b. Ask PM for corrections

Before computing, confirm with PM:

1. **Budget corrections** -- "The total budget on file is EUR X. Any corrections?"
2. **External costs** -- "Are there external costs (agencies, tools, licenses) that reduce the available budget? E.g. SEO agency, stock images, dev tools."
3. **Buffer** -- Default EUR 2,000. PM can adjust.
4. **PM/Designer split ratio** -- "What's the PM vs. designer hour split? Common ratios: 20/80, 30/70, 50/50."

### 2c. Compute estimate budget

```
estimate_budget = total_budget - tracked - buffer - external_costs
```

### 2d. Compute role budgets

Using hourly rates from `team/team-directory-pm.yaml` -> person's `finances.hourly_rate`:

```
pm_budget_eur = estimate_budget * pm_share
designer_budget_eur = estimate_budget * designer_share
pm_hours = pm_budget_eur / pm_hourly_rate
designer_hours = designer_budget_eur / designer_hourly_rate
```

If a developer is involved, add a third split.

### 2e. Present budget breakdown

```
=== Obi — Budget Breakdown ===

Total budget:       EUR 13,100
Already tracked:    EUR  3,200
Buffer:             EUR  2,000
External costs:     EUR      0
                    ──────────
Estimate budget:    EUR  7,900

Split (30/70 PM/Design):
  PM (Marc, EUR 175/h):      EUR 2,370 → 13.5h
  Design (Kathi, EUR 125/h): EUR 5,530 → 44.0h

Confirm budget breakdown? (yes / adjust)
```

PM must confirm before proceeding to Step 3.

---

## Step 3: Allocation shaping (interactive)

### 3a. Gather allocation inputs

PM defines for each team member:

| Field | Required | Example |
|-------|----------|---------|
| Person | Yes | "Kathi" |
| Role | Inferred | "Designer" (from team directory) |
| Hours per week | Yes | "16h/week" |
| Duration | Yes | "2 weeks, Mar 27 - Apr 9" |
| Distribution | No | "even", "front-loaded", "back-loaded" |

Resolve person names against `team/team-directory.yaml` (match on `name` or `aliases`).

### 3b. Check time-off (mandatory)

Before computing allocations, check time-off for all named team members across the full duration.

**Endpoint:** `GET /api/planner/weekly?start=YYYY-MM-DD` (one call per week in the duration)

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-03-30"
```

From the response, for each person extract:
- `timeOffByDay` -- boolean per day (Mon-Fri)
- `timeOffHoursByDay` -- hours per day (supports half-days)
- `timeOffEntries` -- details (type, dates)

Flag any time-off to PM before proposing:
> "Kathi is off Friday Apr 3 (Holiday). Allocations will skip that day."

### 3c. Compute weekly hours

For each person, distribute their weekly hours across available weekdays (Mon-Fri):

1. **Count available days** per week: days where `timeOffByDay[i]` is `false`
2. **Distribute evenly:** `hours_per_day = weekly_hours / available_days`
3. **Round to 0.25h** (15-minute increments): `hours_per_day = round(hours_per_day * 4) / 4`
4. **Absorb rounding remainder** into the last available day of the week
5. **Respect daily capacity:** never exceed `capacityByDay[i] - timeOffHoursByDay[i]`

### 3d. Present proposal table

```
=== Obi Marked Detail Pages — Allocation Plan ===
Budget: EUR 8,000 | Duration: 2 weeks (Mar 27 - Apr 9)

        Mon 27    Tue 28    Wed 29    Thu 30    Fri 31    TOTAL
Marc     1.5       1.5       1.5       1.5       --        6.0h (EUR 1,050)
Kathi    4.0       4.0       4.0       4.0       --       16.0h (EUR 2,000)
────────────────────────────────────────────────────────────────
TOTAL    5.5       5.5       5.5       5.5       0.0      22.0h (EUR 3,050)

        Mon 06    Tue 07    Wed 08    Thu 09    Fri 10    TOTAL
Marc     1.5       1.5       1.5       1.5       --        6.0h (EUR 1,050)
Kathi    4.0       4.0       4.0       4.0       --       16.0h (EUR 2,000)
────────────────────────────────────────────────────────────────
TOTAL    5.5       5.5       5.5       5.5       0.0      22.0h (EUR 3,050)

GRAND TOTAL: 44.0h | EUR 6,100
Remaining estimate budget after allocation: EUR 1,800
```

### 3e. Budget guard

After computing the proposal, check:

```
total_cost = sum(person_hours * person_hourly_rate for all persons)
```

- If `total_cost > estimate_budget`: warn PM with exact overrun amount
  > "This allocation costs EUR 6,100 but only EUR 5,530 is available for design. Over by EUR 570. Reduce hours or confirm overrun?"
- If `total_cost <= estimate_budget`: show remaining budget

### 3f. PM iteration

PM adjusts in natural language. Common patterns:

| PM says | Claude does |
|---------|-----------|
| "Give Kathi 20h/week instead" | Recalculate Kathi's allocation, redistribute |
| "Add Fran for 8h in week 2 only" | Add Fran to week 2, recompute costs |
| "Front-load Marc's hours" | Put more PM hours in week 1, less in later weeks |
| "Reduce to stay within budget" | Scale down proportionally to fit estimate budget |
| "Looks good" / "confirm" | Proceed to Step 4 |

After each adjustment, re-display the updated table. Keep iterating until PM says **confirm**.

---

## Step 4: PM confirms

Show the final summary with all assignments that will be written:

```
=== Final Allocation — Obi ===

Allocation file: projects/2-active/obi/planning/everhour-allocation-weekly.json
Everhour project: as:1212148517388873

Assignments to create:

  Marc Siefert — 12.0h total (EUR 2,100)
    Week 14 (Mar 27-31): 6.0h — Mon-Thu 1.5h/day
    Week 15 (Apr  6-10): 6.0h — Mon-Thu 1.5h/day

  Kathi Mayer — 32.0h total (EUR 4,000)
    Week 14 (Mar 27-31): 16.0h — Mon-Thu 4.0h/day
    Week 15 (Apr  6-10): 16.0h — Mon-Thu 4.0h/day

Total: 44.0h | EUR 6,100

Type **confirm** to write allocation file and create assignments.
```

PM must type "confirm" to proceed.

---

## Step 5: Write allocation file

### 5a. Build the allocation JSON

Schema:

```json
{
  "everhour_project_id": "as:1212148517388873",
  "weeks": [
    {
      "week_start": "2026-03-30",
      "entries": [
        { "person": "Marc Siefert", "hours": 6.0, "note": "PM" },
        { "person": "Kathi Mayer", "hours": 16.0, "note": "Design" }
      ]
    },
    {
      "week_start": "2026-04-06",
      "entries": [
        { "person": "Marc Siefert", "hours": 6.0, "note": "PM" },
        { "person": "Kathi Mayer", "hours": 16.0, "note": "Design" }
      ]
    }
  ]
}
```

`week_start` is always a Monday (ISO format `YYYY-MM-DD`).

### 5b. Write the file

Write to: `projects/2-active/<slug>/planning/everhour-allocation-weekly.json`

If the file already has content, **merge** -- update existing weeks, add new weeks, preserve weeks not in the current plan. If a week already has entries for the same person, replace them.

### 5c. Commit to main

```bash
cd "/Users/marcsiefert/Claude/Silberpuls OS" && \
git add "projects/2-active/<slug>/planning/everhour-allocation-weekly.json" && \
git commit -m "chore(planning): initial allocation for <project-name>" && \
git push
```

---

## Step 6: Write assignments to Everhour

### 6a. Authenticate (once per session)

```bash
curl -s -c /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/auth/unlock \
  -H "Content-Type: application/json" \
  -d '{"password":"PLANNER_PASSWORD"}'
```

The password comes from the `PLANNER_PASSWORD` environment variable. If not set, use 1Password CLI:
```bash
op item get "Silberpuls-pipeline" --fields password --reveal
```

### 6b. Resolve project ID

If `everhour_project_id` from `project-meta.json` is empty, resolve from the API:

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments"
```

Returns `{ projects: [{ id, name }] }`. Match on project name. If ambiguous, ask PM.

### 6c. Check for existing assignments

Before creating, check if assignments already exist for these people on this project in the target weeks:

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments?personName=Kathi%20Mayer&startDate=2026-03-30&endDate=2026-04-10"
```

If existing assignments are found:
- `trackedSeconds == 0` (`canDelete == true`): delete first, then recreate
- `trackedSeconds > 0`: flag to PM, keep existing, adjust new assignments accordingly

### 6d. Delete existing assignments (if safe)

```bash
curl -s -b /tmp/pc.txt -X DELETE \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "assignmentId": 98765,
    "trackedSeconds": 0,
    "personName": "Kathi Mayer",
    "projectId": "as:1212148517388873",
    "projectName": "Obi",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03"
  }'
```

### 6e. Create assignments

One POST per person-project, using `dayAllocations` spanning all weeks:

```bash
curl -s -b /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "personName": "Kathi Mayer",
    "projectId": "as:1212148517388873",
    "projectName": "Obi",
    "startDate": "2026-03-30",
    "endDate": "2026-04-10",
    "dayAllocations": [
      {"day": "2026-03-30", "hours": 4.0},
      {"day": "2026-03-31", "hours": 4.0},
      {"day": "2026-04-01", "hours": 4.0},
      {"day": "2026-04-02", "hours": 4.0},
      {"day": "2026-04-06", "hours": 4.0},
      {"day": "2026-04-07", "hours": 4.0},
      {"day": "2026-04-08", "hours": 4.0},
      {"day": "2026-04-09", "hours": 4.0}
    ]
  }'
```

**Rules:**
- One API call per person -- `dayAllocations` spans all weeks
- `startDate` = first Monday of the allocation; `endDate` = last Friday
- Skip weekends -- only include Mon-Fri in `dayAllocations`
- Skip time-off days -- never include days where `timeOffByDay[i]` is `true`
- Min 15-minute increments -- all hours rounded to nearest 0.25h

### 6f. Report results

```
Assignments written for Obi:

  Marc Siefert:
    CREATED: 12.0h across Mar 30 - Apr 10 (1.5h/day, Mon-Thu)

  Kathi Mayer:
    CREATED: 32.0h across Mar 30 - Apr 10 (4.0h/day, Mon-Thu)

All assignments applied successfully.
```

Note: The Vercel API automatically syncs the Supabase planner snapshot after writes. No manual rebuild needed.

---

## Data source reference

| Data | Source | What it provides |
|------|--------|-----------------|
| Project metadata | `projects/2-active/<slug>/planning/project-meta.json` | Budget, Everhour project ID, client name |
| Tracked costs | `projects/2-active/<slug>/planning/phase-budget-status.json` | Already-tracked hours and EUR |
| Weekly snapshot | `GET /api/planner/weekly?start=YYYY-MM-DD` | Capacity, time-off, existing assignments per person |
| Project list | `GET /api/planner/assignments` (no params) | `{ projects: [{ id, name }] }` for resolving project IDs |
| Assignment metadata | `GET /api/planner/assignments?personName=X&startDate=Y&endDate=Z` | Assignment IDs + tracked seconds (for safe deletion) |
| Team directory | `team/team-directory.yaml` | Person name resolution (names + aliases) |
| Team directory (PM-safe) | `team/team-directory-pm.yaml` | Hourly rates (`finances.hourly_rate`) for budget math |
| Allocation plans | `projects/2-active/*/planning/everhour-allocation-weekly.json` | Existing allocation baselines |

---

## API reference

**Base URL:** `https://silberpuls-pipeline.vercel.app`

### Auth (once per session)

```bash
curl -s -c /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/auth/unlock \
  -H "Content-Type: application/json" \
  -d '{"password":"PLANNER_PASSWORD"}'
```

Returns `{"ok": true}` and sets `planner_unlocked` cookie.

### Read weekly snapshot

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-03-30"
```

### List projects

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments"
```

Returns `{ projects: [{ id, name }] }`. Use `id` (format `as:XXXXXXXXX`) in all assignment operations.

### Read assignments (for deletion metadata)

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments?personName=Kathi%20Mayer&startDate=2026-03-30&endDate=2026-04-10"
```

### Create assignment

```bash
curl -s -b /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "personName": "Kathi Mayer",
    "projectId": "as:1212148517388873",
    "projectName": "Obi",
    "startDate": "2026-03-30",
    "endDate": "2026-04-10",
    "dayAllocations": [
      {"day": "2026-03-30", "hours": 4.0},
      {"day": "2026-03-31", "hours": 4.0}
    ]
  }'
```

### Delete assignment

```bash
curl -s -b /tmp/pc.txt -X DELETE \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "assignmentId": 98765,
    "trackedSeconds": 0,
    "personName": "Kathi Mayer",
    "projectId": "as:1212148517388873",
    "projectName": "Obi",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03"
  }'
```

API rejects delete if `trackedSeconds > 0`. Always check `canDelete` from the GET response before attempting.

---

## Difference from `/plan-week`

| Dimension | `/refine-project` | `/plan-week` |
|-----------|-------------------|-------------|
| Scope | One project, multiple people, multiple weeks | One week, one or all people, multiple projects |
| Entry point | "Refine Obi" | "Plan Fran next week" |
| Budget focus | Computes estimate budget, enforces budget ceiling | Reads existing budget, warns on overrun |
| Allocation file | Creates/updates `everhour-allocation-weekly.json` | Reads it as baseline, updates during rebalance |
| Assignment span | Full project duration (multi-week) | Single week |
| Rebalancing | Not applicable (this IS the initial plan) | Adjusts future weeks when current deviates |
