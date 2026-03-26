---
name: refine-project
description: Post-kickoff project refinement — budget math, phase-aware allocation shaping, and assignment writing for a single project. Claude detects project type, applies phase split profiles (delivery-heavy or discovery-heavy), computes role mixes per phase, and proposes multi-person weekly allocations respecting PM heavy/light patterns and phase sequencing. PM iterates on the full team allocation before confirming writes to Everhour via the Vercel API. Use when user says "refine [project]", "plan assignments for [project]", "set up allocations for [project]", or any variant of project-level resource allocation planning.
---

# Refine Project — Allocation Planning Skill

Post-kickoff refinement: detect project type, apply phase splits and role mixes, compute budget, shape multi-person allocations, write assignments for one project.

| Step | What happens | Who acts |
|------|-------------|----------|
| **1. Parse** | Extract project name/slug from PM request. Resolve against `projects/2-active/*/`. If not provisioned, redirect to `/provision-project`. | Claude |
| **2. Project type & phases** | Ask PM for project type to determine active disciplines. Apply phase split profile (delivery-heavy default). Compute phase budgets and role mixes. Show breakdown, PM confirms. | Claude + PM |
| **3. Budget math** | Read `project-meta.json` budget, tracked costs, ask PM for external costs. Compute estimate budget using phase splits and role mixes. Show breakdown, PM confirms. | Claude + PM |
| **4. Allocation shaping** | Build multi-person proposal across all phases. Apply PM heavy/light pattern. Enforce phase sequencing (discovery before design before dev). Show all team members in one table per week. PM iterates on all together. | Claude + PM |
| **5. Confirm** | Show final summary of all assignments to be written. PM types **confirm**. | PM |
| **6. Write allocation file** | Write `everhour-allocation-weekly.json` to the project planning directory. Commit to main. | Claude |
| **7. Write assignments** | Auth to Vercel API. One POST per person-project with `dayAllocations` spanning all weeks. Distribute weekly hours across weekdays, skip time-off. Report success/failure. | Claude |

See [REFERENCE.md](REFERENCE.md) for the full workflow, budget formulas, API shapes, and example commands.

## Safety rules (non-negotiable)

- Ghost/dry-run summary before any write — PM sees every assignment before it hits the API
- PM must type **confirm** before execution
- Never delete time-off entries (`type == "time-off"`)
- Never delete assignments with tracked time (`trackedSeconds > 0`)
- All Everhour writes go through the Vercel API, never Everhour directly
- Always query time-off before proposing assignments — time-off days are hard constraints
- Allocation file committed to main only after PM confirms
- Min 15-minute increments (round to nearest 0.25h)

## Quick commands (Claude Code)

```bash
# Vercel API auth (once per session)
curl -s -c /tmp/pc.txt -X POST https://silberpuls-pipeline.vercel.app/api/auth/unlock \
  -H "Content-Type: application/json" -d '{"password":"PLANNER_PASSWORD"}'

# Read weekly snapshot (time-off + capacity for all people)
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/weekly?start=2026-03-30"

# List active Everhour projects (to resolve IDs)
curl -s -b /tmp/pc.txt \
  "https://silberpuls-pipeline.vercel.app/api/planner/assignments"

# Read project metadata
cat "projects/2-active/<slug>/planning/project-meta.json"

# Read tracked costs
cat "projects/2-active/<slug>/planning/phase-budget-status.json"
```

## Notes

- This skill handles **one project at a time** — for weekly cross-project planning, use `/plan-week`
- Budget math is not skippable — PM must confirm the budget breakdown before allocation shaping
- Allocation shaping is interactive — PM iterates until satisfied
- `personName` must match `team/team-directory.yaml` exactly (or an alias)
- Project IDs use Everhour format: `as:XXXXXXXXX` (from `project-meta.json` or API project list)
- The Vercel API auto-syncs the Supabase snapshot after writes — no manual rebuild needed
