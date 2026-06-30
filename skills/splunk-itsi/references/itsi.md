# ITSI — IT Service Intelligence

Reference for designing and maintaining ITSI: services, KPIs, thresholds, glass tables, deep dives,
and episode review (AIOps).

## Contents
- Service and KPI design
- KPI base searches
- Thresholds
- Health scores and entities
- Glass tables and deep dives
- Episodes and notable events
- Best practices

## Service and KPI design

- Model **service trees** with real parent-child dependency. Health propagates upward, so structure
  reflects how failures actually cascade.
- A **KPI** is a recurring metric (ad-hoc SPL or data-model based) with a threshold policy. Keep KPI
  searches lean — they run on the ITSI scheduler continuously.
- Define **entity rules** for dynamic service membership using fields like `host`, `src_ip`, or
  custom entity aliases, so services pick up new members automatically.

## KPI base searches

Define a **base search** once at the service level and share it across multiple KPIs. This collapses
many scheduled searches into one, dramatically cutting scheduler load. Each KPI then references a
metric (a field/stat) from the shared base search rather than running its own query.

⚠️ **Best Practice** — Always create base searches at the service level and reuse them across KPIs.
A per-KPI ad-hoc search multiplies scheduler concurrency and is the most common cause of ITSI
scheduler backlog.

## Thresholds

- **Static** — fixed values; use when the normal range is known and stable.
- **Adaptive** — computed from historical data; use for KPIs with natural daily/weekly periodicity.
- **Time-variate** — different thresholds by time block; pair with adaptive for seasonal data.
- **Anomaly detection** — for metrics where deviation matters more than absolute level.

⚠️ **Best Practice** — Use adaptive/time-variate thresholds for periodic KPIs (login volume, request
rate) instead of static values — static thresholds on seasonal data produce constant false alerts.

## Health scores and entities

- Health score is a weighted roll-up of KPI severities; set KPI **weight** and **urgency** so the
  most important KPIs dominate the score.
- Set `is_entity_breakdown=1` on KPIs that need per-entity visibility (per-host health).
- Tag entities with consistent field aliases so entity rules match reliably across sourcetypes.

## Glass tables and deep dives

- **Glass tables**: service/KPI widgets with sparklines, threshold coloring, and drilldowns to deep
  dives or searches.
- **Deep dives**: stacked lanes correlating metric and event series over the same time window for
  root-cause work.

## Episodes and notable events

- **Correlation searches** generate ITSI notable events with severity, status, and owner.
- **Episode grouping policies** (Notable Event Aggregation Policies) deduplicate and group notables
  into episodes; configure flapping detection and grouping fields to cut noise.
- Link Splunk ES notable events to ITSI episodes via alert actions when running both.
- Manage maintenance windows and team assignments through the ITSI REST API for automation.

## Best practices

⚠️ **Best Practice** summary:
- Share KPI base searches across KPIs; validate base-search performance before enabling a KPI — a
  slow base search degrades the whole scheduler.
- Keep service trees shallow (≤3 levels) to avoid health-score propagation delays.
- Query `itsi_summary` historical data with `tstats`, never a raw `search` over `itsi_summary`.
- Set `is_entity_breakdown=1` where per-entity health is needed.
- Use adaptive thresholds for periodic data; static only for genuinely stable metrics.
