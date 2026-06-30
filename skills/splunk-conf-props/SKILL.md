---
name: splunk-conf-props
description: >-
  props.conf — all tasks. Trigger for: event breaking, line breaking,
  SHOULD_LINEMERGE, LINE_BREAKER, timestamp parsing, TIME_FORMAT, TIME_PREFIX,
  MAX_TIMESTAMP_LOOKAHEAD, MAX_DAYS_AGO, TZ, TRUNCATE, field extractions,
  EXTRACT-, REPORT-, EVAL-, FIELDALIAS-, KV_MODE, LOOKUP-, TRANSFORMS-,
  search-time extractions, index-time extractions, source type configuration,
  multiline events, event merging. Default: Splunk Enterprise 9.x.
---

# props.conf

Router only. Read `references/props-conf.md` for all props.conf tasks.

## Routing

| Task | Action |
|---|---|
| Any props.conf directive — write, debug, tune | Read `references/props-conf.md` |

## Shared conventions

- Props run on the parsing tier (HF or indexer for UF-collected data). Index-time settings on a UF have no effect.
- `local/` overrides `default/`; parsing pipeline runs where data is first cooked.
- Always set `TIME_FORMAT`, `MAX_TIMESTAMP_LOOKAHEAD`, and `MAX_DAYS_AGO` for custom sourcetypes.
- `SHOULD_LINEMERGE = false` + explicit `LINE_BREAKER` is the reliable event-breaking combo.
- Prefer search-time extractions (`EXTRACT-`, `REPORT-`, `KV_MODE`) over index-time.
- CIM normalization via `FIELDALIAS`/`EVAL` at search time — never index-time CIM fields.
- Use `btool` to verify effective merged config: `splunk btool props list <sourcetype> --debug`.

## Output format

Fenced `ini` blocks with inline comments. Full stanzas. ⚠️ Best Practice callouts. Flag timestamp, event-breaking, and performance implications.

## Spec reference

Full props.conf spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/props.conf
