# Summary indexing vs report / data-model acceleration

When recurring searches over large time ranges get expensive, pre-compute. Three mechanisms — pick by
shape of the query.

## Summary indexing

Run a scheduled search that aggregates raw data and writes results to a summary index (via `collect`
or `sendalert`/`summary index` action). Dashboards then read the small summary index.

Use when: you have a **specific, known aggregation** (e.g. daily per-team scores) reused across many
dashboards/reports.

```spl
index=eleveo source=subevaluations sourcetype=eleveo
| bin _time span=1d
| stats avg(score) AS avg_score count BY _time team qformname
| collect index=summary_eleveo marker="report=daily_team_scores"
```

⚠️ **Best Practice — guard against gaps/overlaps.** Schedule with an explicit, non-overlapping time
window (`earliest`/`latest` aligned to the cron), and backfill carefully. Double-running a summary
search double-counts.

## Report acceleration

Splunk auto-maintains a summary for a **single saved search** you mark for acceleration. Lower effort
than hand-rolled summary indexing; good for one heavy report you run often.

Use when: one saved report is slow and you want Splunk to manage the summary transparently.

## Data model acceleration

Builds tsidx summaries for an entire **accelerated data model**, queried with `tstats`. This is the
CIM/ES/ITSI path — broad, model-wide acceleration rather than one report.

Use when: many searches/correlations span a whole domain (the ES/ITSI case). Query with `tstats`
(`summariesonly=t` for speed). See the CIM skill for mechanics.

## Choosing

| Need | Mechanism |
|---|---|
| One specific aggregation reused widely, full control | Summary indexing |
| One slow saved report, low effort | Report acceleration |
| Whole-domain acceleration for ES/ITSI/many searches | Data model acceleration (`tstats`) |

## Cost awareness

⚠️ **Best Practice — accelerate selectively and off-peak.** Each mechanism trades disk + scheduler
load for query speed. Accelerate only frequent, large-range searches. Schedule heavy summary searches
during off-peak windows via `cron_schedule`, and monitor `scheduler.log` for skipped searches that
signal scheduler contention.

## Next steps

1. Confirm the recurring search is actually expensive and frequent before accelerating.
2. For summary indexing, document the marker and window; add gap-detection.
3. Watch summary/acceleration storage in the sizing forecast.
