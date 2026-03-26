---
name: plan-week
description: Weekly assignment planning for one person at a time. Reads team capacity, calendar, time-off, and project plans, then proposes per-day assignments. PM adjusts in natural language, confirms, and Claude writes to Everhour via the Vercel API. Use when user says "plan Fran next week", "plan assignments for Lina", "weekly planning for Yannick", or any variant of per-person weekly resource planning.
---

# Plan Week — Single-Person Assignment Skill

One person at a time: read, reason, propose, iterate, write.

| Step | What happens | Who acts |
|------|-------------|----------|
| **1. Gather** | Read weekly snapshot from Vercel API (capacity, time-off, assignments, budget — one call) | Claude |
| **2. Propose** | Build per-day, per-project assignment grid with constraint highlights | Claude |
| **3. Review** | PM adjusts in natural language ("move Thursday to ESN", "reduce Vivatura to 12h") | PM |
| **4. Confirm** | PM types **confirm** to approve the final plan | PM |
| **5. Apply** | Delete old work assignments, create new ones via Vercel API | Claude |
| **6. Verify** | Show applied assignments, flag any API errors | Claude |

See [REFERENCE.md](REFERENCE.md) for the full workflow, data sources, capacity logic, API shapes, and example commands.

## Safety rules (non-negotiable)

- Never delete time-off entries (`type == "time-off"`)
- Never delete assignments with tracked time (`trackedSeconds > 0`)
- All writes require explicit PM confirmation — no silent applies
- Show a dry-run summary before any apply step
- All Everhour writes go through the Vercel API, never Everhour directly
- Always query Everhour time-off before proposing any assignments
- Time-off days are hard constraints — zero available hours, no exceptions
- After applying assignments, sync + rebuild + publish the Supabase snapshot

## Quick commands (Claude Code)

```bash
# Vercel API auth (once per session)
curl -s -c /tmp/pc.txt -X POST https://silberpuls-pipeline.vercel.app/api/auth/unlock \
  -H "Content-Type: application/json" -d '{"password":"PLANNER_PASSWORD"}'

# Read existing assignments for a person + week
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments?personName=Fran%20Marin&startDate=2026-03-30&endDate=2026-04-03"

# List active Everhour projects (to resolve IDs)
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments"

# Propose assignments (existing script, read-only)
python3 .agent/scripts/propose_assignments.py --person "Fran Marin" --week 2026-03-30
```

## Notes

- `personName` must match `team/team-directory.yaml` exactly (or an alias listed there)
- Week always starts on Monday (ISO week)
- Primary data source: `GET /api/planner/weekly?start=YYYY-MM-DD` — returns capacity, time-off, assignments, and budget in one call
- Project IDs use Everhour format: `as:XXXXXXXXX` (from API project list)
- Provisioning baselines in `projects/2-active/*/planning/everhour-allocation-weekly.json` are advisory — PM has final say
- The Vercel API auto-syncs the Supabase snapshot after writes — no manual rebuild needed
