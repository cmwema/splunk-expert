# Validating CIM compliance and accelerating

## Step 1 — verify tags landed

Search the sourcetype and check `tag::eventtype` and the model tags appear:

```spl
sourcetype=acme:gateway:auth | head 100 | fields tag tag::eventtype user src dest action
```

In the field picker, confirm `tag::eventtype` shows your eventtype and the events carry the model
tags (e.g. `authentication`). If tags are missing, the eventtype search isn't matching or sharing
isn't global.

## Step 2 — verify fields against the model

Query the data model directly and filter to your sourcetype:

```spl
| datamodel Authentication Authentication search
| search sourcetype=acme:gateway:auth
| table user src dest action
```

Every CIM field you mapped should be populated. Empty columns mean an alias/EVAL/lookup didn't fire —
revisit the search-time order (extractions → aliases → EVAL → lookups).

Pivot/Datasets are the GUI equivalent: open the dataset, filter by sourcetype, split by `user`, and
eyeball that real values appear.

## Step 3 — CIM Validator

The CIM add-on ships a custom command for validation. Run it against the source to get a structured
report of which required fields are present, missing, or malformed for the targeted model. Treat
missing **required** fields as blockers; recommended fields as best-effort.

## Step 4 — acceleration (production only)

Accelerate a data model only when frequent searches/correlations span large time ranges (ES, ITSI,
SOC dashboards). Acceleration builds tsidx summaries on a schedule and costs disk + indexer CPU.

```spl
| tstats summariesonly=t count
    FROM datamodel=Authentication
    WHERE Authentication.sourcetype=acme:gateway:auth
    BY Authentication.action
```

⚠️ **Best Practice — `tstats` over `stats` on accelerated models.** Always query accelerated data
with `tstats` for performance. Use `summariesonly=t` to read only accelerated buckets (fast, but
misses un-summarized recent data); use `summariesonly=f` to include raw fallback when completeness
matters more than speed.

⚠️ **Best Practice — accelerate selectively.** Don't accelerate every model "just in case." Each
adds persistent disk overhead and indexer load. Factor CIM acceleration into Splunk storage sizing.

## Common "data not in the model" causes

| Symptom | Likely cause |
|---|---|
| `datamodel` search returns nothing | tags not enabled, or eventtype search doesn't match |
| events appear but CIM fields empty | FIELDALIAS/EVAL/LOOKUP not firing or wrong field order |
| works in one app, not ES | knowledge objects not shared global (`default.meta`) |
| `tstats summariesonly=t` empty but `=f` works | acceleration not built yet / backfill incomplete |
| severity shows numbers in ES | mapped to `severity_id` not string `severity` |

## Next steps

1. Document the custom field mappings for the team.
2. Re-run the CIM Validator after any source format change.
3. Monitor acceleration health under Settings → Data Models (build %, last run).
