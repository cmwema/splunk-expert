---
name: splunk-spl
description: >-
  SPL — writing, debugging, and optimizing Splunk searches. Trigger for: writing
  a search, fixing a search, speeding up a search, tstats, stats, eval, where,
  dedup, transaction, join, lookups, macros, savedsearches.conf, alerts,
  scheduled searches, subsearches, data model acceleration. Default: Splunk
  Enterprise 9.x. Assume admin access.
---

# Splunk SPL

Router only. Read `references/spl.md` for all SPL tasks.

## Routing

| Task | Action |
|---|---|
| Any SPL query — write, debug, optimize | Read `references/spl.md` |

## Shared conventions

- Lead every search: `index=` + `sourcetype=` + explicit time range. Time range is the cheapest filter.
- Indexed-field constraints belong before the first pipe — they prune at the index layer.
- Use `tstats` over accelerated data models for high-frequency, wide-range searches.
- Prefer `stats` over `transaction`; prefer `stats` joins over `join`.
- Use `fields` early to drop unneeded columns.
- After the first pipe: `where` for field comparisons, bare `search` only for term filters.
- No leading wildcards on indexed fields (`*foo` defeats the index).
- Avoid `eval` inside `stats` when `fieldformat` or a pre-computed field suffices.
- Scheduled searches: always set explicit `dispatch.earliest_time` / `dispatch.latest_time`.

## Output format

Fenced `splunk` code blocks. Section headers when mixing SPL + `savedsearches.conf`. Add ⚠️ Best Practice callout when a recommendation enforces or deviates from a guideline. Flag performance and data-quality implications.
