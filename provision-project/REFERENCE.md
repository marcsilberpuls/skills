# Provision Project — Reference

## Intent 1: Provision a new project

### Required fields

| Field | Description | Example |
|-------|-------------|---------|
| `project_name` | Display name | `Futalis Rebrand` |
| `client_name` | Client / account name | `Futalis` |
| `total_budget_eur` | Budget as a number | `25000` |
| `pm_name` | Full name, must match team directory | `Lina Talke` |

Optional: `accepted_scope_summary`, `proposal_url`

### Flow

1. Extract fields from natural language — ask for any missing
2. **Before presenting the dry-run summary, check capacity** (see Intent 2 below) for the named PM and any designers mentioned
3. Ghost run (`confirm=false`) — validate inputs, no external writes
4. Present plain-language summary:
   - Project, client, PM, budget (highlighted), scope
   - Capacity flags if any (e.g. "Lina is at 85% through April")
   - What will be created: Asana project, Everhour project, Slack channel, repo directory
5. Ask: "The total budget is **€X,XXX**. Type **confirm** to proceed."
6. On "confirm" → real run (`confirm=true`)
7. Report result — check `#automation-test` on Slack

> **Next step:** Project setup complete. To plan allocations and write assignments for the full project duration, use `/refine-project`.

### GitHub Actions commands (Claude Code)

```bash
gh workflow run provision-project.yml \
  --repo marcsilberpuls/silberpuls-os --ref main \
  --field project_name="PROJECT_NAME" \
  --field client_name="CLIENT_NAME" \
  --field total_budget_eur="BUDGET" \
  --field pm_name="PM_NAME" \
  --field accepted_scope_summary="SCOPE" \
  --field proposal_url="URL" \
  --field confirm="false"   # change to "true" for real run

gh run list --repo marcsilberpuls/silberpuls-os --workflow provision-project.yml --limit 1
```

---

## Intent 2: Capacity check

Uses the Planner API `/api/planner/weekly` endpoint (same auth cookie as assignments).

### Fetch weekly capacity

```bash
# Get capacity for a specific week (Monday date)
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=YYYY-MM-DD"
```

Returns `WeeklyPlannerData`:
```json
{
  "mode": "weekly",
  "startDate": "2026-04-20",
  "endDate": "2026-04-24",
  "days": [...],
  "people": [
    {
      "name": "Lina Talke",
      "role": "Senior Project Manager",
      "utilizationPct": 82,
      "capacityHours": 40,
      "assignedTotalHours": 33,
      "timeOffHours": 0,
      "timeOffDays": 0,
      "timeOffByDay": [false, false, false, false, false],
      "projects": [{ "projectId": "as:123", "projectName": "Futalis Rebrand", ... }]
    }
  ]
}
```

### Check for existing projects (avoid duplicates when provisioning)

From the same response, iterate `people[].projects[].projectName` across all team members to collect the full set of active projects.

### Multi-week capacity check

To check capacity over a range (e.g. "Is Kathi available in May?"), fetch each week:
```bash
curl -s -b /tmp/pc.txt "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-05-04"
curl -s -b /tmp/pc.txt "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-05-11"
curl -s -b /tmp/pc.txt "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-05-18"
curl -s -b /tmp/pc.txt "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-05-25"
```

### Capacity thresholds to flag

- ≥ 90%: overloaded — recommend pushing kickoff or reassigning
- 75–89%: high load — flag, ask if OK
- < 75%: available — no flag needed
- timeOffHours > 0 in target week: flag days out, check `timeOffByDay` for specifics

---

## Intent 3: Assignment management

Calls the Vercel Planner API. Auth via cookie set by the unlock endpoint.

**Base URL:** `https://silberpuls-pipeline.vercel.app`
**Auth cookie:** `planner_unlocked` — set by POST `/api/auth/unlock`

### Auth (once per session)

```bash
curl -s -c /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/auth/unlock \
  -H "Content-Type: application/json" \
  -d '{"password":"PLANNER_PASSWORD"}'
```

### List active projects (to resolve project IDs)

```bash
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments"
```
Returns `{ projects: [{ id, name }] }` — use `id` for all assignment operations.

### Create assignment

```bash
curl -s -b /tmp/pc.txt -X POST \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "personName": "Lina Talke",
    "projectId": "as:12345",
    "projectName": "Futalis Rebrand",
    "startDate": "2026-04-07",
    "endDate": "2026-04-07",
    "hours": 6
  }'
```
For multi-day: add `"days": ["2026-04-07","2026-04-08","2026-04-09"]` (hours applied per day).
Or use `"dayAllocations": [{"day":"2026-04-07","hours":4},{"day":"2026-04-08","hours":6}]` for variable hours.

### Update assignment (replace)

```bash
curl -s -b /tmp/pc.txt -X PATCH \
  https://silberpuls-pipeline.vercel.app/api/planner/assignments \
  -H "Content-Type: application/json" \
  -d '{
    "assignmentId": 98765,
    "userId": 111,
    "personName": "Lina Talke",
    "projectId": "as:12345",
    "projectName": "Futalis Rebrand",
    "startDate": "2026-04-07",
    "endDate": "2026-04-07",
    "hours": 8
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
    "personName": "Lina Talke",
    "projectId": "as:12345",
    "projectName": "Futalis Rebrand",
    "startDate": "2026-04-07",
    "endDate": "2026-04-07"
  }'
```
API rejects delete if `trackedSeconds > 0`. Always surface the tracked time before attempting delete.

### Confirmation summary format

Before any write, present:
> **Assignment change:**
> - Person: Lina Talke
> - Project: Futalis Rebrand (`as:12345`)
> - Date(s): Mon 7 Apr – Wed 9 Apr 2026
> - Hours: 6h/day
> - Action: **create** [or update / delete]
>
> Type **confirm** to apply.

Planner UI reflects changes within ~3 minutes (Supabase overlay + reconcile dispatch).
