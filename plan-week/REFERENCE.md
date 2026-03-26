# Plan Week — Reference

## Overview

This skill plans weekly assignments in two modes:

**Single-person mode** (Steps 1–6):
1. PM says "plan [person] next week" (or "plan [person] week of March 30")
2. Claude gathers all data from the Vercel planner API, proposes assignments
3. PM reviews, adjusts, confirms
4. Claude writes confirmed assignments via the Vercel API

**Full-team mode** (Steps 7–10):
1. PM says "plan next week" (no person name)
2. Claude fetches the weekly snapshot once, iterates through all active people alphabetically
3. For each person: propose, budget-check, PM confirms, write immediately
4. After last person: reconciliation check

---

## Step 1: Parse the request

Extract from the PM's message:

| Field | Required | Example | Default |
|-------|----------|---------|---------|
| Person name | Yes | "Fran Marin", "Fran", "Francisco" | — |
| Target week | No | "next week", "week of March 30" | Next Monday |

**Resolve the person name** against `team/team-directory.yaml`. Match on `name` or any entry in `aliases`. If ambiguous, ask.

**Resolve the week** to the Monday date (ISO format `YYYY-MM-DD`). Compute the five weekday dates: Mon through Fri.

---

## Step 2: Gather data

### 2a. Authenticate (once per session)

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

### 2b. Read the weekly snapshot (primary data source)

**Endpoint:** `GET /api/planner/weekly?start=YYYY-MM-DD`

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-03-30"
```

This single call returns **everything** pre-computed for the target week:

**Response structure:**
```typescript
{
  mode: "weekly",
  startDate: "2026-03-30",
  endDate: "2026-04-03",
  days: [{ isoDate: "2026-03-30", label: "Monday", shortLabel: "Mon" }, ...],
  people: [
    {
      personId: "fran-marin",
      name: "Fran Marin",
      role: "Senior Designer",
      capacityHours: 40.75,          // weekly total capacity
      capacityByDay: [8.15, 8.15, 8.15, 8.15, 0],  // Mon–Fri
      timeOffHours: 0,               // total time-off hours this week
      timeOffByDay: [false, false, false, false, false],  // per-day boolean
      timeOffHoursByDay: [0, 0, 0, 0, 0],  // per-day hours (supports half-days)
      timeOffEntries: [              // detailed time-off entries
        { type: "Holiday", startDate: "2026-04-03", endDate: "2026-04-03", durationDays: 1 }
      ],
      assignedTotalHours: 28.5,      // currently assigned
      assignedDays: [7.5, 7.5, 6.0, 7.5, 0],  // per-day assigned totals
      utilizationPct: 70,
      projects: [
        {
          projectId: "as:1212371417282301",
          projectName: "Vivatura Website & Gesundheitsportal",
          assignedHours: 16,
          trackedHours: 2.5,
          budgetTotalEur: 25000,      // total project budget (may be null)
          budgetSpentEur: 18000,      // already spent (may be null)
          blocks: [                   // per-day assignment detail
            { dayIndex: 0, assignedHours: 4.0, trackedHours: 1.0 },
            { dayIndex: 1, assignedHours: 4.0, trackedHours: 0.5 },
            { dayIndex: 2, assignedHours: 4.0, trackedHours: 1.0 },
            { dayIndex: 3, assignedHours: 4.0, trackedHours: 0 }
          ]
        }
      ]
    }
  ]
}
```

**From this single response, extract for the target person:**
- **Capacity:** `capacityHours` (weekly total), `capacityByDay` (per-day)
- **Time-off:** `timeOffByDay` (boolean per day), `timeOffHoursByDay` (hours per day — supports half-days), `timeOffEntries` (details)
- **Current assignments:** `projects[]` with `assignedHours`, `trackedHours`, `blocks[]`
- **Budget:** `budgetTotalEur`, `budgetSpentEur` per project (for Phase 2 budget warnings)

**If the person is not in the snapshot:** They may be on full leave or inactive. Flag this to the PM.

### 2c. Read existing assignments for deletion metadata

The weekly snapshot shows what's assigned but **does not include `assignmentId` or `trackedSeconds`** needed for safe deletion. To get these, also call:

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments?personName=Fran%20Marin&startDate=2026-03-30&endDate=2026-04-03"
```

This returns project-type assignments with:
```typescript
{
  assignments: [{
    id: number,          // assignmentId — needed for DELETE
    userId: number,
    projectId: string,
    startDate: string,
    endDate: string,
    timeSeconds: number,
    totalHours: number,
    trackedSeconds: number,  // if > 0, cannot delete
    trackedHours: number,
    canDelete: boolean       // true if trackedSeconds <= 0
  }]
}
```

### 2d. Project allocation plans (for baseline comparison)

**Files:** `projects/2-active/*/planning/everhour-allocation-weekly.json`

Each file contains planned weekly hours for a project:
```json
{
  "everhour_project_id": "as:XXXXXXXXX",
  "weeks": [
    { "week_start": "2026-03-30", "entries": [{ "person": "Fran Marin", "hours": 12 }] }
  ]
}
```

Scan all files under `projects/2-active/*/planning/` for entries matching the target person and target week. These form the **baseline plan** — the starting point for the proposal.

If a project has no allocation file or no entry for this person/week, it may still appear via existing assignments or PM input.

---

## Step 3: Compute the proposal

### 3a. Available hours per day

From the weekly snapshot, for each weekday (Mon–Fri):

```
available_hours[i] = capacityByDay[i] - timeOffHoursByDay[i]
```

If `timeOffByDay[i]` is `true`, available is 0 for that day.

No manual capacity computation needed — the snapshot pre-computes this from team-directory.yaml, Everhour time-off, and Timetastic holidays (merged and deduplicated).

### 3b. Project hour inputs

Start with the provisioning baseline (Step 2d) as weekly hour buckets per project. If no baseline exists, use existing assignments from the snapshot (Step 2b) as the starting point.

The PM may override these — "18h Vivatura" replaces whatever the baseline says.

### 3c. Distribute hours across days

For each project's weekly hours, distribute across available days:

1. **Fill evenly:** Spread hours evenly across available days (days where `timeOffByDay` is false and `capacityByDay > 0`).
2. **Respect daily caps:** Never exceed a day's available hours. If a day is full, spill to the next available day.
3. **Time-off days get zero:** Never assign hours on time-off days.

Note: Calendar-based meeting anchoring (placing project hours on days with project meetings) is not available while gcal data files are empty. When calendar sync is restored, also read `.agent/data/config/calendar_project_mapping.json` and `.agent/data/normalized/gcal_events.jsonl` to anchor project hours on meeting days.

### 3d. Handle over-allocation

If total project hours exceed total available hours:
- Flag the over-allocation with exact numbers
- Show which days are maxed out
- Ask PM how to resolve (reduce hours, shift to next week, accept overtime)

---

## Step 4: Present the proposal

Show a clear per-day, per-project grid. Format:

```
=== Fran Marin — Week of 2026-03-30 ===

Capacity: 40.75h | Time-off: 0h | Available: 40.75h

        Mon 30    Tue 31    Wed 01    Thu 02    Fri 03    TOTAL
Vivat.   4.0       4.0       4.0       4.0       —        16.0h
ESN      2.0       2.0       2.0       2.0       —         8.0h
Intern   1.5       1.5       —         1.5       —         4.5h
──────────────────────────────────────────────────────────────
TOTAL    7.5       7.5       6.0       7.5       0.0      28.5h
Avail.   8.15      8.15      8.15      8.15      0.0      32.6h
Gap      0.65      0.65      2.15      0.65      0.0       4.1h

Constraint notes:
- Fri: off day
- Gap: 4.1h unallocated — PM to decide

Existing assignments that will be replaced:
- Vivatura 16h (assignment #12345, 0s tracked) → DELETE + RECREATE
- ESN 8h (assignment #12346, 3600s tracked) → KEEP (has tracked time)
```

### Constraint highlight format

Flag anything the PM should know:
- Time-off days (hard constraint — from snapshot `timeOffByDay`)
- Half-day time-off (reduced capacity — from snapshot `timeOffHoursByDay`)
- Existing assignments with tracked time (cannot delete — from Step 2c `trackedSeconds`)
- Unallocated capacity (gap hours)

---

## Step 5: PM iteration

PM adjusts in natural language. Common patterns:

| PM says | Claude does |
|---------|-----------|
| "Move Thursday to ESN" | Reassign Thu hours from current project to ESN, recalculate |
| "Reduce Vivatura to 12h" | Adjust Vivatura total, redistribute across days |
| "Add 4h Sixt on Monday" | Add Sixt to Mon, reduce gap or flag overflow |
| "Fran is off Wednesday" | Zero out Wed, redistribute remaining hours |
| "Swap Tue and Thu" | Exchange all project hours between those days |
| "Looks good, confirm" | Proceed to apply step |

After each adjustment, re-display the updated grid. Keep iterating until PM says **confirm**.

---

## Step 6: Apply assignments

### 6a. Delete existing work assignments

For each existing assignment (from Step 2c) where `canDelete == true`:

```bash
curl -s -b /tmp/pc.txt -X DELETE \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "assignmentId": 98765,
    "trackedSeconds": 0,
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03"
  }'
```

**If `trackedSeconds > 0`:** Do NOT delete. Flag to PM:
> "Assignment #12345 (ESN, 1.0h tracked) cannot be deleted — has tracked time. Keeping as-is."

### 6b. Create new assignments

For each project in the confirmed plan, create one assignment per project spanning the relevant days:

```bash
curl -s -b /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03",
    "dayAllocations": [
      {"day": "2026-03-30", "hours": 4.0},
      {"day": "2026-03-31", "hours": 4.0},
      {"day": "2026-04-01", "hours": 4.0},
      {"day": "2026-04-02", "hours": 4.0}
    ]
  }'
```

**API body options:**
- `dayAllocations` array: variable hours per day (preferred — most flexible)
- `hours` + `days` array: same hours per day on specific days
- `hours` alone with `startDate`/`endDate`: same hours every day in range

Use `dayAllocations` when hours vary by day. Skip days with zero hours for that project.

### 6c. Verify and report

After all API calls complete, show a summary:

```
Applied assignments for Fran Marin — week of 2026-03-30:

  DELETED:
    ✅ Vivatura (assignment #12345) — deleted
    ⚠️ ESN (assignment #12346) — kept (1.0h tracked)

  CREATED:
    ✅ Vivatura — 16h across Mon–Thu (4h/day)
    ✅ ESN — 8h across Mon–Thu (2h/day)
    ✅ Intern — 4.5h across Mon, Tue, Thu (1.5h/day)

  SKIPPED:
    ESN — existing assignment with tracked time kept as-is
```

Note: The Vercel API automatically syncs the Supabase planner snapshot after writes (`publishSnapshotAfterAssignmentChange`). No manual snapshot rebuild needed.

---

## Data source reference

| Data | Source | What it provides |
|------|--------|-----------------|
| Weekly snapshot | `GET /api/planner/weekly?start=YYYY-MM-DD` | Capacity, time-off, assignments, budget per person per project — all pre-computed |
| Assignment metadata | `GET /api/planner/assignments?personName=X&startDate=Y&endDate=Z` | Assignment IDs + tracked seconds (needed for safe deletion) |
| Project list | `GET /api/planner/assignments` (no params) | `{ projects: [{ id, name }] }` — for resolving project IDs |
| Project allocation plans | `projects/2-active/*/planning/everhour-allocation-weekly.json` | Provisioning baseline (planned hours per person per week) |
| Team directory | `team/team-directory.yaml` | Person name resolution (names + aliases) |
| Team directory (PM-safe) | `team/team-directory-pm.yaml` | Hourly rates (`finances.hourly_rate`) for budget warnings |

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
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments?personName=Fran%20Marin&startDate=2026-03-30&endDate=2026-04-03"
```

### Create assignment

```bash
curl -s -b /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03",
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
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03"
  }'
```

API rejects delete if `trackedSeconds > 0`. Always check `canDelete` from the GET response before attempting.

### Update assignment (replace)

```bash
curl -s -b /tmp/pc.txt -X PATCH \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "assignmentId": 98765,
    "userId": 111,
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03",
    "hours": 6
  }'
```

---

## Existing scripts (reference only — do not modify)

These scripts implement similar logic. Read them for context, but the skill instructs Claude to perform the logic inline rather than calling them:

| Script | Purpose |
|--------|---------|
| `.agent/scripts/propose_assignments.py` | Single-person proposal generator — reads calendar, Asana, budgets |
| `.agent/scripts/apply_weekly_assignments.py` | Assignment writer — deletes old + creates new via Vercel API |
| `.agent/oslib/capacity.py` | Capacity computation from team-directory.yaml |

---

## Confirmation summary format

Before any write, present a clear summary and wait for explicit confirmation:

> **Assignment plan for Fran Marin — week of 2026-03-30:**
>
> | Action | Project | Days | Hours |
> |--------|---------|------|-------|
> | DELETE | Vivatura (assignment #12345, 0s tracked) | Mon–Fri | 20h |
> | CREATE | Vivatura | Mon–Thu | 16h (4h/day) |
> | CREATE | ESN Website | Mon–Thu | 8h (2h/day) |
> | CREATE | Silberpuls Intern | Mon, Tue, Thu | 4.5h (1.5h/day) |
> | KEEP | ESN Website (assignment #12346, 1h tracked) | — | — |
>
> Type **confirm** to apply.

---

## Step 7: Full-team flow — snapshot and sync offer

When the PM says "plan next week" (or "plan the team", "weekly planning" — no person name), use the full-team flow.

### 7a. Authenticate

Same as Step 2a.

### 7b. Fetch the weekly snapshot

Same as Step 2b — `GET /api/planner/weekly?start=YYYY-MM-DD`. This returns all people in one call.

Show the snapshot timestamp to the PM:

> "Using planner data last synced at 08:15 today. If time-off was approved in the last hour, it may not be reflected. Want me to trigger a fresh sync first? (~5 minutes)"

### 7c. Optional: trigger fresh sync

If the PM wants fresh data:

```bash
gh workflow run daily-holiday-sync.yml
gh workflow run hourly-planner-snapshot-sync.yml
```

Wait for both workflows to complete (~5 minutes), then re-fetch the snapshot (Step 7b).

---

## Step 8: Iterate through people

### 8a. Sort and filter

From the snapshot `people[]` array:

1. **Sort alphabetically** by `name`
2. **Skip** anyone where all entries in `timeOffByDay` are `true` (full-week time-off). Note to PM:
   > "[Name] is on holiday this week — skipped."
3. The remaining people are the active list to iterate through.

### 8b. Per-person proposal

For each active person, follow the single-person flow:
- **Compute proposal:** Same as Steps 3a–3d (available hours, project baselines, distribute, handle over-allocation)
- **Present proposal:** Same as Step 4 (per-day, per-project grid with constraints)
- **Budget check:** See Step 8c below — add budget warnings to the proposal when applicable
- **PM iteration:** Same as Step 5 (PM adjusts, Claude re-displays)
- **Apply on confirm:** See Step 9 below (write immediately, partial commit)

Then move to the next person.

### 8c. Budget warnings

After computing the proposal for a person, check each project's budget. Only warn when this week's proposed hours would exceed the remaining budget.

**Budget data from snapshot:** `project.budgetTotalEur` and `project.budgetSpentEur`

**Hourly rate from team directory:** `team/team-directory-pm.yaml` (PM-safe) or `team/team-directory.yaml` — field `finances.hourly_rate`

**Calculation:**
```
remaining_eur = budgetTotalEur - budgetSpentEur
proposed_cost = proposed_hours × hourly_rate
overrun_eur = proposed_cost - remaining_eur
overrun_hours = overrun_eur / hourly_rate
safe_hours = remaining_eur / hourly_rate  (floored to nearest 0.5h)
```

**Warning format** (only show when `proposed_cost > remaining_eur`):

> "[Project]: assigning [Name] [X]h this week would exceed the remaining budget by ~[overrun_hours]h (€[overrun_eur] over €[remaining_eur] remaining at €[hourly_rate]/h). Reduce to [safe_hours]h to stay within budget, or confirm anyway?"

**When NOT to warn:**
- `budgetTotalEur` is null (no budget set) — skip budget check
- `proposed_cost <= remaining_eur` — within budget, no warning needed

### 8d. Project ID resolution

When PM mentions a project by name during iteration (e.g., "add 4h Sixt"):

1. Fetch the project list: `GET /api/planner/assignments` (no params) — returns `{ projects: [{ id, name }] }`
2. Match the PM's input against project names:
   - **Exactly one match** → use it
   - **Multiple matches** → ask: "Did you mean '[Full Name A]' (as:1234...) or '[Full Name B]' (as:5678...)?"
   - **Zero matches** → ask: "No project found matching '[input]'. Available projects: [list]. Which one?"

---

## Step 9: Apply per person (partial commit)

When the PM confirms a person's plan, write assignments immediately. Do not wait for the full team to be done.

### 9a. Assignments with tracked time (PATCH)

For assignments where `trackedSeconds > 0`, use PATCH to update in place — this preserves tracked time:

```bash
curl -s -b /tmp/pc.txt -X PATCH \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "assignmentId": 98765,
    "userId": 111,
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-03-30",
    "endDate": "2026-04-03",
    "hours": 6
  }'
```

Required fields: `assignmentId`, `userId`, `personName`, `projectId`, `projectName`, `startDate`, `endDate`, `hours`.

Get `assignmentId` and `userId` from the GET assignments call (Step 2c).

### 9b. Clean assignments (DELETE + CREATE)

For assignments where `trackedSeconds == 0` (`canDelete == true`): same as Step 6a (DELETE) + Step 6b (CREATE).

### 9c. Verify per person

After writing, show a brief summary (same as Step 6c) and move to the next person.

**If PM stops mid-way** (closes conversation, says "stop", etc.): already-confirmed people's assignments are applied. Unconfirmed people are unchanged.

---

## Step 10: Reconciliation check

After the last person is confirmed and written, run a final reconciliation.

### 10a. Re-fetch snapshot

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=YYYY-MM-DD"
```

### 10b. Check for issues

Compare the fresh snapshot against what was planned. Flag:

1. **Budget overruns:** Any project where `budgetSpentEur > budgetTotalEur` after all writes
   > "⚠ [Project]: budget is now €[spent] / €[total] — €[overrun] over budget."

2. **Over-utilization:** Any person where `utilizationPct > 100`
   > "⚠ [Name]: utilization is [X]% — over-allocated by [Y]h."

3. **API errors:** Any assignments that failed to write during the session (track these as you go in Step 9)
   > "⚠ Failed to write [Name]'s [Project] assignment — [error message]. Retry manually."

### 10c. Clean report

If no issues are detected:

> "All assignments applied. No issues detected."
