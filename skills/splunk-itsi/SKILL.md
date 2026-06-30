---
name: splunk-itsi
description: >-
  Splunk ITSI (IT Service Intelligence) — all tasks. Trigger for: ITSI service
  design, service trees, KPIs, KPI base searches, thresholds, adaptive
  thresholds, time variate thresholds, glass tables, deep dives, entity rules,
  episode review, notable events, correlation searches, alert actions, episode
  grouping, maintenance windows, health scores, KPI weighting, urgency, ITSI
  REST API. Default: Splunk Enterprise 9.x, ITSI latest.
---

# Splunk ITSI

Router only. Read `references/itsi.md` for all ITSI tasks.

## Routing

| Task | Action |
|---|---|
| Any ITSI task — services, KPIs, glass tables, episodes | Read `references/itsi.md` |

## Shared conventions

- Define KPI base searches at the service level and share across KPIs to minimize search concurrency.
- Use `tstats` for ITSI KPI base searches that run over wide time ranges — never raw `search` against `itsi_summary`.
- Use adaptive thresholds for KPIs with natural periodicity (daily/weekly); static for stable metrics.
- Keep service trees ≤ 3 levels deep — deeper trees cascade health-score delays.
- `is_entity_breakdown=1` on KPIs that need per-entity visibility.
- Tag ITSI entities with consistent field aliases so entity rules match across sourcetypes.
- Validate KPI search performance before enabling — a slow base search degrades the entire ITSI scheduler.

## Output format

Prose for service design. Fenced `splunk` for KPI searches. JSON for REST API payloads. ⚠️ Best Practice callouts. Flag scheduler performance and health-score propagation implications.
