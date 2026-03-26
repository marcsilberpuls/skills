# Plan Week — Reference

## Overview

This skill plans weekly assignments in two modes:

**Single-person mode** (Steps 1–7):
1. PM says "plan [person] next week" (or "plan [person] week of March 30")
2. Claude gathers all data from the Vercel planner API + provisioning baselines, proposes assignments with deviation flags
3. PM reviews, adjusts, confirms
4. Claude writes confirmed assignments via the Vercel API
5. Claude detects deviations from provisioning plan, proposes rebalancing of future weeks

**Full-team mode** (Steps 8–11):
1. PM says "plan next week" (no person name)
2. Claude fetches the weekly snapshot + last week's snapshot once, iterates through all active people alphabetically
3. For each person: propose with deviation flags, budget-check, PM confirms, write immediately, then rebalance
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

### 3b. Project hour inputs (baseline priority)

Claude uses this priority order to determine starting hours per project:

1. **Provisioning plan** (`everhour-allocation-weekly.json`, Step 2d) — primary baseline. Use the entry matching the target person and target week.
2. **Last week's actual assignments** — fallback if no provisioning plan entry exists for a project. Fetch via `GET /api/planner/weekly?start=<last-monday>` (same API, previous week's Monday). Use the person's `projects[].assignedHours` from last week as this week's starting point.
3. **PM input** — overrides everything. "18h Vivatura" replaces whatever the baseline says.

**Fetching last week's data (Step 2e):**

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=<last-monday>"
```

Where `<last-monday>` is the Monday before the target week (target Monday minus 7 days). Extract the target person's projects and hours. Cache this — it is used both for baseline fallback and for projects without allocation files.

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

### 4a. Assignment grid

Show a clear per-day, per-project grid. Format:

```
=== Fran Marin — Week of 2026-03-30 ===

Capacity: 40.75h | Time-off: 0h | Available: 40.75h

        Mon 30    Tue 31    Wed 01    Thu 02    Fri 03    TOTAL
Vivat.   4.5       4.5       4.5       4.5       —        18.0h
ESN      1.5       1.5       1.5       1.5       —         6.0h
Intern   1.5       1.5       —         1.5       —         4.5h
──────────────────────────────────────────────────────────────
TOTAL    7.5       7.5       6.0       7.5       0.0      28.5h
Avail.   8.15      8.15      8.15      8.15      0.0      32.6h
Gap      0.65      0.65      2.15      0.65      0.0       4.1h
```

### 4b. Deviation flags (inline, before PM adjusts)

Compare the proposed hours for each project against the provisioning plan (`everhour-allocation-weekly.json`). Show deviations inline, directly below the grid:

```
Deviations from provisioning plan:
⚠ Vivatura: plan says 12h, proposing 18h. +6h deviation (+€540 at €90/h)
⚠ ESN: plan says 8h, proposing 6h. -2h deviation (-€180 at €90/h)
```

**Rules:**
- Only flag projects that **have** an entry in `everhour-allocation-weekly.json` for this person and week.
- Projects **without** an allocation file (or without an entry for this person/week) use last week's actuals as the starting point — **no deviation flag** for these.
- Show deviation in hours (primary) and EUR (secondary). EUR = deviation hours x person's hourly rate from `team/team-directory-pm.yaml` → `finances.hourly_rate`.
- Positive deviation = more hours than planned (over plan). Negative = fewer hours (under plan).
- Zero deviation = no flag needed.

### 4c. Change summary

Show what's changing from current assignments (from snapshot `projects[]`):

```
Changes from current assignments:
  Vivatura: 16h → 12h (-4h) — provisioning plan says 12h
  ESN: 8h → 8h (no change)
  Intern: 0h → 4.5h (new)
```

### 4d. Constraint highlights

Flag anything the PM should know:
- Time-off days (hard constraint — from snapshot `timeOffByDay`)
- Half-day time-off (reduced capacity — from snapshot `timeOffHoursByDay`)
- Existing assignments with tracked time (cannot delete — from Step 2c `trackedSeconds`)
- **Unallocated capacity:** If gap is >2h or >25% of weekly capacity, prompt explicitly:
  > "Fran has 12h unallocated this week — no active project assignments. What should he work on?"

  For small gaps (≤2h and ≤25% capacity), just show in the grid's Gap row without an explicit prompt. Claude never invents work — always wait for PM input.

```
Constraint notes:
- Fri: off day
- ⚠ 12h unallocated — what should Fran work on?

Existing assignments that will be replaced:
- Vivatura 16h (assignment #12345, 0s tracked) → DELETE + RECREATE
- ESN 8h (assignment #12346, 3600s tracked) → KEEP (has tracked time)
```

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

## Step 7: Deviation detection and rebalancing

After writing this week's assignments (Step 6), Claude checks for deviations from the provisioning plan and offers to rebalance future weeks. This step runs per person — before moving to the next person in full-team mode.

### 7a. Identify deviations

For each project assigned to this person this week, compare the **confirmed hours** against the **provisioning plan** (`everhour-allocation-weekly.json`):

```
deviation = confirmed_hours - planned_hours
deviation_eur = deviation × hourly_rate
```

- `planned_hours`: from the matching entry in `everhour-allocation-weekly.json` for this person + week
- `hourly_rate`: from `team/team-directory-pm.yaml` → person's `finances.hourly_rate`
- **Skip** projects that have no allocation file or no entry for this person/week (these are unplanned — no deviation to detect)

Only proceed if at least one project has a non-zero deviation.

### 7b. Compute rebalancing proposals

For each project with a deviation, compute an even-spread rebalancing across remaining future weeks:

1. **Determine future weeks:** All weeks in `everhour-allocation-weekly.json` where `week_start > target_week` (i.e., weeks after the current planning week). The last week is defined by the last `week_start` entry in the file (project horizon).

2. **Compute adjustment per week:**
   ```
   remaining_weeks = count of future weeks with entries for this person
   adjustment_per_week = -deviation / remaining_weeks
   new_hours_per_week = original_planned_hours + adjustment_per_week
   ```
   Round to nearest 0.5h. Absorb rounding remainder into the last future week.

3. **Guard rails:**
   - `new_hours_per_week` must be >= 0. If even spread would go negative, cap at 0 and distribute the shortfall across fewer weeks.
   - If no future weeks remain in the allocation file, warn PM: "No future weeks to rebalance against. Deviation will affect total project budget."

### 7c. Present rebalancing summary

Show all deviations for this person in one block:

```
=== Fran Marin — Rebalancing after week 14 ===

Deviations this week:
  Vivatura: +6h over plan (+€540 at €90/h)
    → Even spread across weeks 15-20 (6 weeks): reduce from 12h to 11h each
  ESN: -2h under plan (-€180 at €90/h)
    → Even spread across weeks 15-18 (4 weeks): increase from 8h to 8.5h each

Confirm rebalance? (confirm / reject / modify)
```

### 7d. PM response handling

| PM says | Claude does |
|---------|-----------|
| **"confirm"** | Update allocation files + write future assignments (Steps 7e + 7f) |
| **"reject"** | Warn about budget consequence, move to next person. Current week is already applied. |
| **"modify"** / natural language | PM adjusts distribution (e.g., "front-load Vivatura reduction", "put all ESN catch-up in week 15"). Claude recomputes and re-displays. |

**Reject warning format:**
> "Without rebalancing, Vivatura will be over budget by ~€540 across the remaining project timeline. Future week allocations unchanged."

### 7e. Update allocation files (on rebalance confirm)

For each project with a confirmed rebalance:

1. Read `projects/2-active/<project>/planning/everhour-allocation-weekly.json`
2. Update the `hours` field for this person in each affected future week entry
3. Write the file back
4. Commit directly to main:

```bash
cd "/Users/marcsiefert/Claude/Silberpuls OS" && \
git add "projects/2-active/<project>/planning/everhour-allocation-weekly.json" && \
git commit -m "chore(planning): rebalance <project-name> allocations after week <N> adjustment" && \
git push
```

**Rules:**
- One commit per project (or batch all affected projects in one commit if preferred by PM)
- No branch, no PR — same partial commit principle as Everhour writes
- If the session goes stale after this point, allocation files are already persisted

### 7f. Write future assignments to Everhour (on rebalance confirm)

For each project with a confirmed rebalance, write future assignments to Everhour in a single batched API call per person-project:

```bash
curl -s -b /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "personName": "Fran Marin",
    "projectId": "as:1212371417282301",
    "projectName": "Vivatura Website & Gesundheitsportal",
    "startDate": "2026-04-06",
    "endDate": "2026-05-15",
    "dayAllocations": [
      {"day": "2026-04-06", "hours": 2.75},
      {"day": "2026-04-07", "hours": 2.75},
      {"day": "2026-04-08", "hours": 2.75},
      {"day": "2026-04-09", "hours": 2.75},
      ...
    ]
  }'
```

**Rules:**
- One API call per person-project combo — `dayAllocations` spans all future weeks through the project horizon
- Distribute each future week's hours evenly across weekdays (Mon–Fri), same as Step 3c logic
- `startDate` = Monday of the first future week; `endDate` = Friday of the last future week
- Skip weekends — only include Mon–Fri in `dayAllocations`
- Before writing, check for existing future assignments and delete them (same safety rules as Step 6a — respect `trackedSeconds`)

### 7g. Verify and report

After all allocation file commits and future assignment writes:

```
Rebalancing applied for Fran Marin:

  ALLOCATION FILES UPDATED:
    ✅ Vivatura — weeks 15-20 updated (12h → 11h each)
    ✅ ESN — weeks 15-18 updated (8h → 8.5h each)

  FUTURE ASSIGNMENTS WRITTEN:
    ✅ Vivatura — 66h across weeks 15-20 (11h/week)
    ✅ ESN — 34h across weeks 15-18 (8.5h/week)

  GIT COMMITS:
    ✅ chore(planning): rebalance Vivatura allocations after week 14 adjustment
    ✅ chore(planning): rebalance ESN allocations after week 14 adjustment
```

Then move to the next person (full-team mode) or finish (single-person mode).

---

## Data source reference

| Data | Source | What it provides |
|------|--------|-----------------|
| Weekly snapshot | `GET /api/planner/weekly?start=YYYY-MM-DD` | Capacity, time-off, assignments, budget per person per project — all pre-computed |
| Last week's snapshot | `GET /api/planner/weekly?start=<last-monday>` | Previous week's actual assignments — fallback baseline when no provisioning plan exists |
| Assignment metadata | `GET /api/planner/assignments?personName=X&startDate=Y&endDate=Z` | Assignment IDs + tracked seconds (needed for safe deletion) |
| Project list | `GET /api/planner/assignments` (no params) | `{ projects: [{ id, name }] }` — for resolving project IDs |
| Project allocation plans | `projects/2-active/*/planning/everhour-allocation-weekly.json` | Provisioning baseline (planned hours per person per week), future weeks for rebalancing |
| Team directory | `team/team-directory.yaml` | Person name resolution (names + aliases) |
| Team directory (PM-safe) | `team/team-directory-pm.yaml` | Hourly rates (`finances.hourly_rate`) for budget warnings and deviation EUR amounts |

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

## Step 8: Full-team flow — snapshot and sync offer

When the PM says "plan next week" (or "plan the team", "weekly planning" — no person name), use the full-team flow.

### 8a. Authenticate

Same as Step 2a.

### 8b. Fetch the weekly snapshot + last week's snapshot

Fetch both the target week and last week's snapshots:

- Target week: `GET /api/planner/weekly?start=YYYY-MM-DD` (same as Step 2b)
- Last week: `GET /api/planner/weekly?start=<last-monday>` (for baseline fallback, same as Step 2e)

This returns all people in one call each. Cache both — reused for every person in the iteration.

Show the snapshot timestamp to the PM:

> "Using planner data last synced at 08:15 today. If time-off was approved in the last hour, it may not be reflected. Want me to trigger a fresh sync first? (~5 minutes)"

### 8c. Optional: trigger fresh sync

If the PM wants fresh data:

```bash
gh workflow run daily-holiday-sync.yml
gh workflow run hourly-planner-snapshot-sync.yml
```

Wait for both workflows to complete (~5 minutes), then re-fetch both snapshots (Step 8b).

---

## Step 9: Iterate through people

### 9a. Sort and filter

From the snapshot `people[]` array:

1. **Sort alphabetically** by `name`
2. **Skip** anyone where all entries in `timeOffByDay` are `true` (full-week time-off). Note to PM:
   > "[Name] is on holiday this week — skipped."
3. The remaining people are the active list to iterate through.

### 9b. Per-person proposal

For each active person, follow the single-person flow:
- **Compute proposal:** Same as Steps 3a–3d (available hours, project baselines with priority from 3b, distribute, handle over-allocation)
- **Present proposal:** Same as Step 4 (per-day, per-project grid with deviation flags + constraints)
- **Budget check:** See Step 9c below — add budget warnings to the proposal when applicable
- **PM iteration:** Same as Step 5 (PM adjusts, Claude re-displays)
- **Apply on confirm:** See Step 10 below (write immediately, partial commit)
- **Rebalance:** After writing current week, run Step 7 (deviation detection + rebalancing) before moving to next person

Then move to the next person.

### 9c. Budget warnings

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

### 9d. Project ID resolution

When PM mentions a project by name during iteration (e.g., "add 4h Sixt"):

1. Fetch the project list: `GET /api/planner/assignments` (no params) — returns `{ projects: [{ id, name }] }`
2. Match the PM's input against project names:
   - **Exactly one match** → use it
   - **Multiple matches** → ask: "Did you mean '[Full Name A]' (as:1234...) or '[Full Name B]' (as:5678...)?"
   - **Zero matches** → ask: "No project found matching '[input]'. Available projects: [list]. Which one?"

---

## Step 10: Apply per person (partial commit)

When the PM confirms a person's plan, write assignments immediately. Do not wait for the full team to be done. After writing, run Step 7 (deviation detection + rebalancing) before moving to the next person.

### 10a. Assignments with tracked time (PATCH)

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

### 10b. Clean assignments (DELETE + CREATE)

For assignments where `trackedSeconds == 0` (`canDelete == true`): same as Step 6a (DELETE) + Step 6b (CREATE).

### 10c. Verify per person

After writing, show a brief summary (same as Step 6c), then proceed to Step 7 (rebalancing). After rebalancing is complete (or skipped), move to the next person.

**If PM stops mid-way** (closes conversation, says "stop", etc.): already-confirmed people's assignments are applied (current week + any confirmed rebalances). Unconfirmed people are unchanged.

---

## Step 11: Reconciliation check

After the last person is confirmed, written, and rebalanced, run a final reconciliation.

### 11a. Re-fetch snapshot

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=YYYY-MM-DD"
```

### 11b. Check for issues

Compare the fresh snapshot against what was planned. Flag:

1. **Budget overruns:** Any project where `budgetSpentEur > budgetTotalEur` after all writes
   > "⚠ [Project]: budget is now €[spent] / €[total] — €[overrun] over budget."

2. **Over-utilization:** Any person where `utilizationPct > 100`
   > "⚠ [Name]: utilization is [X]% — over-allocated by [Y]h."

3. **API errors:** Any assignments that failed to write during the session (track these as you go in Step 10)
   > "⚠ Failed to write [Name]'s [Project] assignment — [error message]. Retry manually."

4. **Rebalancing failures:** Any allocation file commits or future assignment writes that failed during Step 7
   > "⚠ Failed to commit rebalanced allocations for [Project] — [error message]. Run manually."

### 11c. Clean report

If no issues are detected:

> "All assignments applied. All rebalances committed. No issues detected."
