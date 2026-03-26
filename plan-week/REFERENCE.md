# Plan Week — Reference

## Overview

This skill plans one person's weekly assignments through a conversational flow:

1. PM says "plan [person] next week" (or "plan [person] week of March 30")
2. Claude gathers all data, computes available capacity, and proposes assignments
3. PM reviews, adjusts, confirms
4. Claude writes to Everhour via the Vercel API

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

Read these sources in parallel. Each may be empty — handle gracefully.

### 2a. Team capacity (from YAML)

**File:** `team/team-directory.yaml`

Read the person's `capacity` block. Key fields:

| Field | Meaning | Example |
|-------|---------|---------|
| `avg_hours_day` | Base hours per workday | `8.15` |
| `work_time_percent` | Employment percentage | `90` |
| `schedule` | Per-weekday hour overrides | `{ Monday: 8.0, Friday: 0.0 }` |
| `off_days` | Days with zero hours | `[Friday]` |
| `working_days` | Allowed days (array) or rotating pattern (string) | `[Monday, Tuesday, Wednesday, Thursday]` or `"Jeden 2. Freitag frei"` |

**Capacity computation** (mirrors `.agent/oslib/capacity.py`):

1. **Base per day:** `avg_hours_day * work_time_percent / 100`
   - Exception: if `working_days` is a text string (rotating pattern), use `avg_hours_day` directly — the percentage is already reflected in the average
2. **Schedule overrides:** if `schedule` dict exists, replace per-weekday values
3. **Off days:** zero out any day in `off_days`
4. **Working days (array):** if `working_days` is an array, zero out any weekday not in the list
5. **Sum Mon–Fri** for weekly total

Example for Yannick Fischer (80%, off Fridays, schedule overrides):
```
Mon: 8.0  Tue: 8.0  Wed: 8.0  Thu: 8.0  Fri: 0.0  → 32.0h/week
```

Example for Fran Marin (90%, rotating Friday, text pattern):
```
Base: 8.15h/day (work_time_percent already implicit)
Mon: 8.15  Tue: 8.15  Wed: 8.15  Thu: 8.15  Fri: 8.15  → 40.75h/week
(Every other Friday is off — check calendar/time-off for the specific week)
```

### 2b. Time-off (from holidays.jsonl)

**Important:** The Vercel `/api/planner/assignments` endpoint filters by `type === "project"` — it does NOT return time-off entries. Read time-off from the normalized data instead.

**File:** `.agent/data/normalized/holidays.jsonl`

Format: `{ user_id, start_date, end_date, type, status, duration }`

To match by person name, first resolve the person's Everhour `user_id` from:

**File:** `.agent/data/normalized/everhour_users.jsonl`

Format: `{ id, name, email, status, rate }`

Match person name → `id`, then filter `holidays.jsonl` for entries where:
- `user_id` matches
- `status == "Approved"`
- Date range overlaps with the target week

Time-off entries are **hard constraints** — zero available hours on those days, no exceptions.

If `holidays.jsonl` is empty or missing, also check Everhour resource planner assignments directly (these may include `type: "time-off"` entries from the Everhour API that aren't exposed via the Vercel route).

### 2c. Calendar meetings (from JSONL files)

**Files:**
- `.agent/data/normalized/gcal_events.jsonl` — individual events with `person`, `date`, `duration_min`, `event_category`, `mapped_project`
- `.agent/data/normalized/gcal_daily_load.jsonl` — daily aggregate with `person`, `date`, `meeting_hours`, `available_hours`, `has_all_day_event`

Filter both files for the target person and target week dates.

**If files are empty or missing:** proceed without meeting data. Flag in the proposal:
> "Calendar data not available — capacity computed from team directory only. Meeting conflicts not reflected."

**If files have data:** subtract `meeting_hours` from each day's base capacity.

### 2d. Calendar-project mapping

**File:** `.agent/data/config/calendar_project_mapping.json`

Structure:
```json
{
  "project_rules": [
    { "pattern": "vivatura", "everhour_project_id": "as:1212371417282301", "project_name": "Vivatura Website & Gesundheitsportal" }
  ],
  "internal_patterns": ["silberstammtisch", "all hands", ...]
}
```

Use `project_rules` to identify which calendar events map to which Everhour projects. This drives **meeting anchoring** (Step 3).

### 2e. Project allocation plans

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

### 2f. Existing assignments (from Vercel API)

**Endpoint:** `GET /api/planner/assignments?personName=X&startDate=YYYY-MM-DD&endDate=YYYY-MM-DD`

This returns only `type == "project"` assignments (time-off is handled separately in Step 2b). These show what is already booked and may need to be deleted + recreated.

Record `assignmentId` and `trackedSeconds` for each — needed for safe deletion.

---

## Step 3: Compute the proposal

### 3a. Available hours per day

For each weekday (Mon–Fri):

```
available_hours = base_capacity - meeting_hours - time_off_reduction
```

Where:
- `base_capacity` = from Step 2a (per-day schedule)
- `meeting_hours` = from gcal_daily_load.jsonl (0 if no calendar data)
- `time_off_reduction` = if time-off exists for that day, set available to 0

### 3b. Project hour inputs

Start with the provisioning baseline (Step 2e) as weekly hour buckets per project. If no baseline exists, use existing assignments (Step 2f) as the starting point.

The PM may override these — "18h Vivatura" replaces whatever the baseline says.

### 3c. Distribute hours across days (calendar anchoring)

For each project's weekly hours, distribute across available days using this priority:

1. **Anchor on meeting days:** If the person has a calendar event mapped to this project (via `calendar_project_mapping.json`), assign project hours on that day first. Include the meeting hours as part of the project total.
2. **Fill remaining evenly:** Spread leftover hours evenly across other available days.
3. **Respect daily caps:** Never exceed a day's available hours. If a day is full, spill to the next available day.
4. **Time-off days get zero:** Never assign hours on time-off days.

Example:
- Fran has 18h Vivatura for the week
- Wednesday has a 2h Vivatura meeting
- Available hours: Mon 8h, Tue 8h, Wed 6h (8h - 2h meeting), Thu 8h, Fri 0h (off)
- Anchor: Wed gets Vivatura hours first (meeting 2h + work = e.g. 4h)
- Remaining 14h distributed across Mon, Tue, Thu (4.67h each)

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

Capacity: 40.75h | Meetings: 6.5h | Time-off: 0h | Available: 34.25h

        Mon 30    Tue 31    Wed 01    Thu 02    Fri 03    TOTAL
Vivat.   4.0       4.0       4.0*      4.0       —        16.0h
ESN      2.0       2.0       2.0       2.0       —         8.0h
Intern   1.5       1.5       —         1.5       —         4.5h
──────────────────────────────────────────────────────────────
TOTAL    7.5       7.5       6.0       7.5       0.0      28.5h
Avail.   8.15      8.15      6.15      8.15      0.0      30.6h
Gap      0.65      0.65      0.15      0.65      0.0       2.1h

* Wed: 2h Vivatura meeting + 2h deep work

Constraint notes:
- Fri: off day (every other Friday pattern — this week is off)
- Wed: 2h meeting reduces available deep-work time
- Gap: 2.1h unallocated — PM to decide

Existing assignments that will be replaced:
- Vivatura 16h (assignment #12345, 0s tracked) → DELETE + RECREATE
- ESN 8h (assignment #12346, 3600s tracked) → KEEP (has tracked time)
```

### Constraint highlight format

Flag anything the PM should know:
- Time-off days (hard constraint)
- Days with heavy meetings (soft constraint — flag available hours)
- Over-budget projects (from Everhour budget data)
- Existing assignments with tracked time (cannot delete)
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

### 6a. Authenticate

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

### 6b. Delete existing work assignments

For each existing assignment (from Step 2f) where `type != "time-off"` and `trackedSeconds == 0`:

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

### 6c. Create new assignments

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
- `hours` + `days` array: same hours per day on specific days
- `dayAllocations` array: variable hours per day (preferred — most flexible)
- `hours` alone with `startDate`/`endDate`: same hours every day in range

Use `dayAllocations` when hours vary by day (which they usually do after calendar anchoring).

Skip days where the person has zero hours for that project — do not include `0` in dayAllocations.

### 6d. Verify and report

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

### 6e. Sync snapshot

After writes, sync + rebuild + publish the Supabase planner snapshot:

```bash
python3 .agent/scripts/sync_everhour.py
python3 .agent/scripts/build_everhour.py
npx tsx apps/resource-planner/scripts/build-planner-snapshot.ts
npx tsx apps/resource-planner/scripts/publish-planner-snapshot.ts
```

This ensures the planner UI reflects the new assignments. The planner app reads from the `planner_snapshots` Supabase table.

---

## Data source reference

| Data | File / Endpoint | Format |
|------|----------------|--------|
| Team capacity | `team/team-directory.yaml` | YAML, `team_members[].capacity` |
| Time-off | `.agent/data/normalized/holidays.jsonl` | JSONL: `{user_id, start_date, end_date, type, status, duration}` |
| Everhour users (ID lookup) | `.agent/data/normalized/everhour_users.jsonl` | JSONL: `{id, name, email, status, rate}` |
| Calendar events | `.agent/data/normalized/gcal_events.jsonl` | JSONL: `{person, date, duration_min, event_category, mapped_project}` |
| Calendar daily load | `.agent/data/normalized/gcal_daily_load.jsonl` | JSONL: `{person, date, meeting_hours, available_hours, has_all_day_event}` |
| Calendar-project map | `.agent/data/config/calendar_project_mapping.json` | JSON: `{project_rules: [{pattern, everhour_project_id, project_name}]}` |
| Project allocations | `projects/2-active/*/planning/everhour-allocation-weekly.json` | JSON: `{everhour_project_id, weeks: [{week_start, entries: [{person, hours}]}]}` |
| Everhour assignments | Vercel API: `GET /api/planner/assignments?personName=X&startDate=Y&endDate=Z` | JSON (project-type only, no time-off) |
| Everhour projects | Vercel API: `GET /api/planner/assignments` (no params) | JSON: `{projects: [{id, name}]}` |
| Everhour budgets | `.agent/data/normalized/everhour_budgets.jsonl` | JSONL: `{id, name, budget, spent}` |

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

### List projects

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments"
```

Returns `{ projects: [{ id, name }] }`. Use `id` (format `as:XXXXXXXXX`) in all assignment operations.

### Read assignments

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

Alternative body formats:
- `"hours": 4, "days": ["2026-03-30", "2026-03-31"]` — same hours per day
- `"hours": 4` (no days/dayAllocations) — hours applied per day across startDate–endDate

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

API rejects delete if `trackedSeconds > 0`. Always check before attempting.

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
| `.agent/oslib/capacity.py` | Capacity computation from team-directory.yaml — `compute_weekly_hours()` and `load_team_capacities()` |

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
