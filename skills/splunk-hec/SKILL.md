---
name: splunk-hec
description: >-
  Splunk HTTP Event Collector (HEC) — all tasks. Trigger for: HEC token creation
  and management, HEC inputs.conf stanzas, HEC endpoint (/services/collector,
  /services/collector/event, /services/collector/raw), HEC batching, HEC
  indexer acknowledgment (ack), HEC load balancing, HEC troubleshooting
  (4xx/5xx errors, queue full, dropped events), HEC global settings, HEC TLS,
  HEC from Fluent Bit / Fluentd / Logstash / vector / app code, HEC token
  rotation, HEC per-token index/sourcetype override, multi-line events over HEC.
  Default: Splunk Enterprise 9.x.
---

# Splunk HEC

Router only. Read `references/hec.md` for all HEC tasks.

## Routing

| Task | Action |
|---|---|
| Token creation, inputs.conf HEC stanzas | Read `references/hec.md` |
| HEC batching, ack, load balancing | Read `references/hec.md` |
| Troubleshoot dropped events, 4xx/5xx | Read `references/hec.md` |
| HEC from Fluent Bit / Fluentd / vector | Read `references/hec.md` |
| Ingestion path decision (HEC vs SC4S vs UF) | Defer to `splunk-ingestion-pipeline` |

## Shared conventions

- One HEC token per source/application — never share tokens across teams or apps.
- Always set `index`, `sourcetype` in the token config; override at the event level only when necessary.
- Enable TLS on HEC (`enableSSL = 1`); use a valid cert — clients must verify in production.
- Batch events: POST up to 1 MB or 1000 events per request; split larger payloads.
- Enable indexer acknowledgment (`useACK = 1`) for at-least-once delivery; clients must poll `/ack`.
- HEC queue depth: `maxQueueSize` in `inputs.conf`; monitor `splunkd.log` for `QueueFull` errors.
- Rotate tokens via Splunk UI or REST — never embed tokens in client-side code; use secrets managers.
- Load balance across indexers by posting to each HEC endpoint or via a layer-7 LB with sticky disabled.
- Troubleshoot: `index=_internal sourcetype=splunkd component=HttpInputDataHandler` for HEC errors.

## Output format

Fenced `ini` for inputs.conf, `bash`/`curl` for HEC POST examples, `yaml` for Fluent Bit/vector config. ⚠️ callouts for token security, ack reliability, and queue saturation.
