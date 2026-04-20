---
name: provision-project
description: PM colleague interface for Silberpuls OS. Handles three intents from natural language: (1) provision a new won project via GitHub Actions, (2) check live team capacity before committing, (3) create/update/delete Everhour assignments via the Planner API. Use when user says "provision a project", "we won a project", "assign X to project Y", "check capacity", "update Fran's assignment", or any variant of project onboarding or assignment management.
---

# Provision Project — PM Colleague Skill

Three intents, all confirmed before execution:

| Intent | Trigger | Backend |
|--------|---------|---------|
| **Provision** | "We won a project", "Set up X for client Y" | `provision-project.yml` GitHub Actions |
| **Capacity check** | "Who has capacity in May?", "Is Lina available?" | Planner API `/api/planner/weekly` |
| **Assignments** | "Assign X to Y on Mon–Wed", "Remove Fran's booking" | Vercel `/api/planner/assignments` |

See [REFERENCE.md](REFERENCE.md) for detailed workflows, API shapes, and example curl commands.

## Safety rules (non-negotiable)

- Ghost run (dry-run) always runs before any provisioning real run
- `total_budget_eur` requires explicit "confirm" before real provisioning run fires
- Assignment creates/updates/deletes always show a plain-language summary before execution
- Never delete assignments with tracked time (`trackedSeconds > 0`)
- Never delete time-off entries (`type == "time-off"` — enforced by API, flag it anyway)
- All Everhour writes go through the Vercel API, never Everhour directly

## Quick commands (Claude Code)

```bash
# Provision — ghost run
gh workflow run provision-project.yml --repo marcsilberpuls/silberpuls-os --ref main \
  --field project_name="…" --field client_name="…" --field total_budget_eur="…" \
  --field pm_name="…" --field confirm="false"

# Provision — real run (after explicit budget confirmation)
# Same command but --field confirm="true"

# Vercel API auth + list projects
curl -s -c /tmp/pc.txt -X POST https://silberpuls-pipeline.vercel.app/api/auth/unlock \
  -H "Content-Type: application/json" -d '{"password":"PLANNER_PASSWORD"}'
curl -s -b /tmp/pc.txt https://silberpuls-pipeline.vercel.app/api/planner/assignments
```

## Notes

- `pm_name` must match team directory exactly — ghost run will flag a mismatch
- `proposal_url` is optional in the workflow but the Python script flags it as a soft warning in ghost mode (non-blocking)
- Capacity queries use the Planner API (`/api/planner/weekly?start=YYYY-MM-DD`) — same auth cookie as assignments, no Supabase MCP needed
- For colleague / browser-only usage: see [COWORK.md](COWORK.md) for system instructions to paste into the Claude.ai project
