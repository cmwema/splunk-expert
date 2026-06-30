# SPL — Search, Optimization, and Knowledge Objects

Reference for writing and tuning Splunk Search Processing Language. Splunk Enterprise 9.x.

## Contents
- The optimization mindset
- Filtering and pipeline ordering
- tstats and accelerated data models
- stats / eval / where patterns
- Lookups, macros, saved searches, alerts
- Common anti-patterns
- Worked examples

## The optimization mindset

Splunk cost is dominated by how much data leaves the indexers and how much work the search head
does. Every optimization reduces one of: events scanned, fields carried, or transformations run.
Order operations so the cheapest, most selective filters run first.

## Filtering and pipeline ordering

- Lead every search with `index=`, `sourcetype=`, and an explicit earliest/latest. The time range
  is the single most powerful filter.
- Put indexed-field constraints in the first search clause (before the first pipe) so they prune at
  the index layer. Constraints after the first pipe run on the search head.
- Use `fields` early to drop columns you won't need — it cuts the data moved across the bus.
- After the first pipe, use `where` for field-to-field or evaluated comparisons; reserve bare
  `search` for simple term filters.

⚠️ **Best Practice** — Avoid leading wildcards (`*foo`) on indexed fields; they defeat the index and
force a full scan. Anchor terms on the left.

## tstats and accelerated data models

`tstats` runs against the tsidx files / accelerated data models and is dramatically faster than
`stats` over raw events for large ranges.

```splunk
| tstats summariesonly=t count
    from datamodel=Network_Traffic.All_Traffic
    where All_Traffic.action=blocked earliest=-24h latest=now
    by All_Traffic.src All_Traffic.dest
```

- `summariesonly=t` restricts to accelerated data only (fast, but may miss un-summarized recent
  data); use `summariesonly=f` to fill in at the cost of speed.
- `prestats=t` feeds a downstream `stats`/`timechart` for multi-stage aggregation.
- Use `tstats` for dashboard panels and ITSI KPI base searches that run frequently over wide ranges.

⚠️ **Best Practice** — Accelerate a data model only when searches against it run frequently and span
large ranges; acceleration consumes disk and indexer cycles continuously.

## stats / eval / where patterns

- Prefer `stats` over `transaction` whenever you can group by a key — `transaction` is memory-heavy
  and doesn't distribute well.
- Avoid `eval` inside `stats` when a `fieldformat` (display-only) or a pre-computed field works;
  `fieldformat` doesn't change the underlying value, keeping later math correct.
- Use `stats` aggregation rather than `dedup` for "latest per key": `| stats latest(_raw) by host`
  distributes better than `| dedup host`.

## Lookups, macros, saved searches, alerts

- **Lookups:** define in `transforms.conf` (`filename` for CSV, `external_cmd` for scripted,
  `collection` for KV Store). Use automatic lookups via `props.conf` `LOOKUP-<name>` only when every
  search of that sourcetype needs the enrichment; otherwise call `| lookup` explicitly to avoid
  search-time overhead.
- **Macros** (`macros.conf`): parameterize repeated SPL. Reference as `` `macro_name(arg)` ``.
  Centralize index/sourcetype constants in a macro so a migration is a one-line change.
- **Saved searches / alerts** (`savedsearches.conf`): always set an explicit `dispatch.earliest_time`
  and `dispatch.latest_time` — never rely on defaults. Schedule heavy jobs off-peak with
  `cron_schedule`. Use `| collect` or `sendalert` to write summaries instead of re-running expensive
  base searches.

```ini
[Blocked traffic spike - hourly]
search = | tstats count from datamodel=Network_Traffic where ... by _time span=5m
dispatch.earliest_time = -1h
dispatch.latest_time = now
cron_schedule = 5 * * * *
enableSched = 1
alert.severity = 3
action.summary_index = 1
action.summary_index._name = summary_netsec
```

⚠️ **Best Practice** — Watch `scheduler.log` and `metrics.log` for skipped searches and high latency;
skipped scheduled searches usually mean concurrency limits or an over-long base search.

## Common anti-patterns

- `index=* sourcetype=*` then filtering later — scans everything. Constrain up front.
- `| search field=value` right after the base search when `field=value` could be in the base search.
- `join` for things `stats` can do — `join` has subsearch result limits and poor performance; reach
  for `stats by` or `| append`/`| union` patterns first.
- Subsearches returning more than `maxout` rows (default 10000) silently truncate; prefer `lookup`
  or `stats` joins for large sets.

## Worked examples

**Slow → fast (raw stats to tstats):**

```splunk
index=fw sourcetype=pan:traffic action=allowed
| stats sum(bytes) by dest_ip
```
becomes
```splunk
| tstats summariesonly=t sum(All_Traffic.bytes) as bytes
    from datamodel=Network_Traffic.All_Traffic
    where All_Traffic.action=allowed earliest=-24h
    by All_Traffic.dest
```

**Latest event per host without dedup:**
```splunk
index=linux sourcetype=syslog
| stats latest(_raw) as last_event, latest(_time) as last_time by host
```
