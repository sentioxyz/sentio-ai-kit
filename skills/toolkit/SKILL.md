---
name: sentio-toolkit
description: Build, modify, or troubleshoot Sentio projects across processors, Sentio SQL in Data Studio, and alerting. Use for tasks involving @sentio/sdk handlers in processor.ts, sentio.yaml config, metrics/event logs/entity store, running or sharing Sentio SQL queries, or creating/updating alert rules and notification channels.
---

# sentio-toolkit

Use `npx @sentio/cli@latest` for SQL, data queries, alerts, endpoints, and project management.

## Setup

1. Check login status: `npx @sentio/cli@latest login --status`
2. If not logged in, ask the user for api key and run: `npx @sentio/cli@latest login --api-key <key>`

## SQL

- Execute: `sentio data sql --project <owner>/<slug> --query "SELECT * FROM transfer LIMIT 10"`
- Async: `sentio data sql --project <owner>/<slug> --query "..." --async`
- Fetch async result: `sentio data sql --project <owner>/<slug> --result <execution-id>`
- Paginate: `sentio data sql --project <owner>/<slug> --cursor <cursor>`
- From file: `sentio data sql --project <owner>/<slug> --file query.yaml`

## Data queries

- List events: `sentio data events --project <owner>/<slug>`
- List metrics: `sentio data metrics --project <owner>/<slug>`
- Query event: `sentio data query --project <owner>/<slug> --event Transfer --aggr total --group-by timestamp`
- Query metric: `sentio data query --project <owner>/<slug> --metric burn --aggr avg --filter meta.chain=1`
- Query price: `sentio data query --project <owner>/<slug> --price ETH`
- With functions: `--func 'topk(5)'`, `--func 'delta(1m)'`
- Time range: `--start <time> --end <time> --step <seconds>`
- File format docs: `sentio data query --doc`

## Alerts

- List: `sentio alert list --project <owner>/<slug>`
- Get: `sentio alert get <rule-id> --project <owner>/<slug>`
- Create metric: `sentio alert create --project <owner>/<slug> --type METRIC --subject "Burn spike" --metric burn --aggr avg --op '>' --threshold 100`
- Create log: `sentio alert create --project <owner>/<slug> --type LOG --subject "Large transfers" --query 'amount > 1000' --op '>' --threshold 0`
- Create SQL: `sentio alert create --project <owner>/<slug> --type SQL --subject "Alert" --query 'select timestamp, amount from transfer where amount > 1000' --time-column timestamp --value-column amount --sql-aggr MAX --op '>' --threshold 1000`
- From file: `sentio alert create --project <owner>/<slug> --file alert.yaml`
- Update: `sentio alert update <rule-id> --project <owner>/<slug> --file updated.yaml`
- Delete: `sentio alert delete <rule-id> --project <owner>/<slug>`
- File format docs: `sentio alert create --doc`

## Endpoints

- Create: `sentio endpoint create --project <owner>/<slug> --query "SELECT * FROM t WHERE amount > \${min}" --args '{"min":"int"}'`
- List: `sentio endpoint list --project <owner>/<slug>`
- Test: `sentio endpoint test --project <owner>/<slug> --id <id> --args '{"min":1000}'`
- Delete: `sentio endpoint delete --project <owner>/<slug> --id <id>`

## Project & processor management

- List projects: `sentio project list`
- Get project: `sentio project get <owner>/<slug>`
- Processor status: `sentio processor status --project <owner>/<slug>`
- Processor source: `sentio processor source --project <owner>/<slug>`
- Price lookup: `sentio price get --coin-id ETH`
- List coins: `sentio price coins`
- Run simulation: `sentio simulation run --project <owner>/<slug> --file sim.yaml`

## Notes

- All commands accept `--api-key <key>` or use saved credentials from `sentio login`.
- Use `--json` or `--yaml` for machine-readable output.
- Use `--project <owner>/<slug>` to target a project, or `--owner` + `--name` separately.
- For long SQL queries, use `--async` then fetch with `--result <id>`.
- SQL alert queries must include a time column and a numeric value column; use `$startTime`/`$endTime` placeholders.
