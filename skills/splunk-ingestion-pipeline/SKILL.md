---
name: splunk-ingestion-pipeline
description: >-
  Design the end-to-end Splunk data ingestion architecture — choosing the path (UF / HF / HEC / SC4S),
  index design and naming, retention tiers, sizing, and where parsing/CIM happen — for a new or
  changing data source. Use this whenever the user is deciding HOW to get a source in and lay it out,
  not the line-level config of one component — including phrasings like "how should I ingest this source",
  "design the pipeline for this device or app", "which index should this go in", "how many indexes do I
  need", "retention / hot-warm-cold / frozen for this data", "estimate index storage", "UF vs HF vs
  HEC vs SC4S for this", "where should I parse/route this", or "summary indexing vs acceleration".
  Covers source→collection→parse→index→CIM design decisions, index strategy, retention/sizing, and
  summary indexing. Defer SC4S parser internals, props/transforms line config, CIM field mapping, and
  cluster mechanics to their dedicated skills — this skill is the architecture/decision layer above them.
---

# Splunk ingestion pipeline design

This is the **architecture and decision** layer: given a new source, decide the collection method,
where parsing and routing happen, which index it lands in, how long it's retained, and how it gets
normalized — before anyone writes a single `props.conf` line. The detailed per-component config lives
in the SC4S, conf-files, CIM, and cluster skills.

## The canonical pipeline

```
Source  →  Collection (UF | HF | HEC | SC4S)  →  Parse/Route (HF or SC4S)  →  Index  →  CIM (search-time)  →  ES/ITSI/dashboards
```

Every design answers five questions in order:

1. **Collection** — how does the data leave the source? (file, syslog, API/HEC, scripted)
2. **Parse/route** — where do timestamping, line-breaking, filtering, and index routing happen?
3. **Index** — which index, by data domain, with what access controls?
4. **Retention** — hot/warm/cold/frozen tiers and sizing.
5. **Normalization** — which CIM model, applied search-time in a TA.

## Routing table

| If the user is deciding… | Read |
|---|---|
| which collection method (UF/HF/HEC/SC4S) | `references/collection-methods.md` |
| index layout, naming, access | `references/index-design.md` |
| retention tiers + storage sizing | `references/retention-and-sizing.md` |
| summary indexing vs report/DM acceleration | `references/summarization.md` |

## Decision quick-reference

| Source kind | Default path |
|---|---|
| Files on a server (auth.log, app logs, Windows Event Log) | **UF** → indexers |
| Syslog from network/appliance devices | **SC4S** → HEC → indexers |
| App/cloud pushing events over HTTP | **HEC** (token per source) |
| Needs masking/anonymization/complex routing before indexing | route through an **HF** |
| API pull (REST, DB) | modular/scripted input on a **HF** (or UF for simple cases) |

## Shared conventions

- **Filter early, parse close to the source.** Drop noise before it costs indexing license/volume —
  in SC4S for syslog, or on an HF for file/HEC paths. Don't index-then-filter.
- **UFs collect; HFs parse/route.** Reserve Heavy Forwarders for parsing, masking, and routing
  decisions; use Universal Forwarders for plain endpoint collection. Don't put parsing on every UF.
- **Separate indexes by data domain** (`windows`, `linux`, `network`, `syslog`, …) for RBAC,
  retention, and search efficiency — not one mega-index.
- **Parse once.** Decide the single place timestamping/line-breaking happens (first full Splunk
  instance in the path — HF or indexer) and don't double-parse.
- **Normalize at search time.** CIM mapping is a TA on the search heads, never index-time fields.
- **TLS everywhere**, explicit `index`/`sourcetype`/`host` per input, never rely on defaults, never
  the `default` catch-all sourcetype in production.

## Next steps after a design

1. Hand the per-component specs to the right skill (SC4S parser, conf-files, CIM TA).
2. Define the index on the Cluster Manager (clustered) before forwarders send to it.
3. Validate end-to-end with a sample event: correct index, sourcetype, timestamp, and CIM fields.
