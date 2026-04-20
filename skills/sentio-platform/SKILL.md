---
name: sentio-platform
description: Build, modify, or troubleshoot Sentio projects across processors, Sentio SQL in Data Studio, alerting, and dashboards. Use for tasks involving @sentio/sdk handlers in processor.ts, sentio.yaml config, metrics/event logs/entity store, running or sharing Sentio SQL queries, creating/updating alert rules and notification channels, or managing dashboards and panels.
---

# sentio-platform

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
- No cache: add `--no-cache`; control TTL with `--cache-ttl-secs <s>` and `--cache-refresh-ttl-secs <s>`
- Specific processor version: `--version <version>`

## Data queries

- List events: `sentio data events --project <owner>/<slug>`
- List metrics: `sentio data metrics --project <owner>/<slug>`
- Query event: `sentio data query --project <owner>/<slug> --event Transfer --aggr total --group-by timestamp`
- Query metric: `sentio data query --project <owner>/<slug> --metric burn --aggr avg --filter meta.chain=1`
- Query price: `sentio data query --project <owner>/<slug> --price ETH`
- With functions: `--func 'topk(5)'`, `--func 'delta(1m)'`
- Time range: `--start <time> --end <time> --step <seconds>`
- Limit/offset: `--limit <count> --offset <count>`
- Timezone: `--timezone <tz>`
- File format docs: `sentio data query --doc`

## Alerts

- List: `sentio alert list --project <owner>/<slug>`
- Get: `sentio alert get <rule-id> --project <owner>/<slug>`
- Create metric: `sentio alert create --project <owner>/<slug> --type METRIC --subject "Burn spike" --metric burn --aggr avg --op '>' --threshold 100`
- Create event: `sentio alert create --project <owner>/<slug> --type METRIC --subject "Transfer anomaly" --event Transfer --filter amount>0 --aggr total --func 'delta(1m)' --op '>' --threshold 100`
- Create log: `sentio alert create --project <owner>/<slug> --type LOG --subject "Large transfers" --query 'amount:>1000' --op '>' --threshold 0`
- Create SQL: `sentio alert create --project <owner>/<slug> --type SQL --subject "Alert" --query 'select timestamp, amount from transfer where amount > 1000' --time-column timestamp --value-column amount --sql-aggr MAX --op '>' --threshold 1000`
- From file: `sentio alert create --project <owner>/<slug> --file alert.yaml`
- Update: `sentio alert update <rule-id> --project <owner>/<slug> --file updated.yaml`
- Delete: `sentio alert delete <rule-id> --project <owner>/<slug>`
- Duration/interval: `--for 5m --interval 1m`; between condition: `--op between --threshold 10 --threshold2 100`
- File format docs: `sentio alert create --doc`

## Endpoints

- Create: `sentio endpoint create --project <owner>/<slug> --query "SELECT * FROM t WHERE amount > \${min}" --args '{"min":"int"}'`
- List: `sentio endpoint list --project <owner>/<slug>`
- Get: `sentio endpoint get --project <owner>/<slug> --id <id>`
- Test: `sentio endpoint test --project <owner>/<slug> --id <id> --args '{"min":1000}'`
- Delete: `sentio endpoint delete --project <owner>/<slug> --id <id>`

## Dashboards

- List: `sentio dashboard list --project <owner>/<slug>`
- Create: `sentio dashboard create --project <owner>/<slug> --title "My Dashboard"`
- Create from file: `sentio dashboard create --project <owner>/<slug> --file dashboard.json`
- Export: `sentio dashboard export <dashboardId> --project <owner>/<slug>`
- Import: `sentio dashboard import <dashboardId> --project <owner>/<slug> --file dashboard.json`
- Import from stdin: `sentio dashboard import <dashboardId> --project <owner>/<slug> --stdin`
- Import override layout: `sentio dashboard import <dashboardId> --project <owner>/<slug> --file dashboard.json --override-layouts`
- Add SQL panel: `sentio dashboard add-panel <dashboardId> --project <owner>/<slug> --panel-name "Top Holders" --type TABLE --sql "SELECT * FROM CoinBalance ORDER BY balance DESC LIMIT 50"`
- Add event panel: `sentio dashboard add-panel <dashboardId> --project <owner>/<slug> --panel-name "Transfer Count" --type LINE --event Transfer --aggr total`
- Add metric panel: `sentio dashboard add-panel <dashboardId> --project <owner>/<slug> --panel-name "ETH Price" --type LINE --metric cbETH_price`
- Panel with filters/groups: `--filter amount>1000 --group-by meta.address --func 'topk(5)'`
- Panel time range: `--time-range-start -24h --time-range-end now --time-range-step 3600`
- Chart types: `TABLE`, `LINE`, `BAR`, `PIE`, `QUERY_VALUE`, `AREA`, `BAR_GAUGE`, `SCATTER`
- Event aggr options: `total`, `unique`, `AAU`, `DAU`, `WAU`, `MAU`
- Metric aggr options: `avg`, `sum`, `min`, `max`, `count`

### Dashboard import JSON format

When constructing or editing `dashboard.json` for import, refer to `references/openapi.swagger.json` for the full schema. Key structure:

```json
{
  "dashboardId": "<target-dashboard-id>",
  "dashboardJson": {
    "name": "My Dashboard",
    "panels": {
      "<panelId>": {
        "id": "<panelId>",
        "name": "Panel Name",
        "chart": {
          "type": "<ChartType>",
          "datasourceType": "<DataSourceType>",
          "sqlQuery": "SELECT ...",          // for SQL panels
          "queries": [...],                  // for metric/event panels
          "formulas": [...],
          "config": { ... }
        }
      }
    },
    "layouts": { ... }
  },
  "overrideLayouts": false
}
```

- `Chart.type` values: `TABLE`, `LINE`, `BAR`, `PIE`, `QUERY_VALUE`, `AREA`, `BAR_GAUGE`, `SCATTER`
- `Chart.datasourceType`: determines query shape — SQL panels use `sqlQuery` string; metric/event panels use `queries` array
- `panels` is a map keyed by panel ID; each panel contains a `chart` object
- `layouts` controls responsive grid placement; omit or set `overrideLayouts: true` to auto-arrange
- See `references/openapi.swagger.json` definitions `web_service.Dashboard`, `web_service.Panel`, `web_service.Chart`, `common.Query` for complete field lists

## Project management

- List projects: `sentio project list`
- Get project: `sentio project get <owner>/<slug>`
- Create project: `sentio project create --owner <owner> --name <name> --type sentio --visibility private`
- Delete project: `sentio project delete <owner>/<slug>`
- Project types: `sentio`, `subgraph`, `action`

## Processor management

- Status: `sentio processor status --project <owner>/<slug>`
- Source files: `sentio processor source --project <owner>/<slug>`
- Activate pending version: `sentio processor activate-pending --project <owner>/<slug>`
- Pause: `sentio processor pause --project <owner>/<slug> --reason "maintenance" -y`
- Resume: `sentio processor resume --project <owner>/<slug>`
- Stop: `sentio processor stop --project <owner>/<slug>`
- Logs: `sentio processor logs --project <owner>/<slug> --limit 100`
- Follow logs: `sentio processor logs --project <owner>/<slug> -f`
- Filter logs: `--log-type execution --level ERROR --query "timeout"`

## Price

- Get price: `sentio price get --coin ETH` or `--symbol ETH` or `--address 0x... --chain 1`
- Historical price: `sentio price get --coin ETH --timestamp <unix-ts>`
- Batch prices: `sentio price batch --file prices.yaml`
- List supported coins: `sentio price coins`
- Check latest available prices: `sentio price check-latest`

## Simulation

- Run transaction simulation: `sentio simulation run --project <owner>/<slug> --file sim.yaml`
- Run bundle simulation: `sentio simulation run-bundle --project <owner>/<slug> --chain-id <id> --file bundle.yaml`
- List simulations: `sentio simulation list --project <owner>/<slug>`
- Get simulation: `sentio simulation get <simulation-id> --project <owner>/<slug>`
- Get bundles: `sentio simulation bundle --project <owner>/<slug>`
- Fetch call traces: `sentio simulation trace --project <owner>/<slug>`

## Notes

- All commands accept `--api-key <key>` or use saved credentials from `sentio login`.
- All commands accept `--token <token>` for bearer token auth.
- Use `--json` or `--yaml` for machine-readable output.
- Use `--project <owner>/<slug>` to target a project, or `--owner` + `--name` separately, or `--project-id <id>`.
- For long SQL queries, use `--async` then fetch with `--result <id>`.
- SQL alert queries must include a time column and a numeric value column; use `$startTime`/`$endTime` placeholders.
- `references/openapi.swagger.json` contains the full Sentio REST API spec — read it when constructing dashboard JSON payloads, debugging CLI calls, or building custom integrations. Key definitions for dashboards: `web_service.ImportDashboardRequest`, `web_service.Dashboard`, `web_service.Panel`, `web_service.Chart`, `common.Query`.
