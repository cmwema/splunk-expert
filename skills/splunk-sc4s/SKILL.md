---
name: splunk-sc4s
description: >-
  Splunk Connect for Syslog (SC4S) — all tasks. Trigger for: SC4S parser
  development, custom parsers, application blocks, block parsers, syslog-ng
  config, env_file, splunk_index.csv, sc4s-test, drop filters, routing, HEC
  configuration for SC4S, disk buffering, SC4S tuning, SC4S deployment,
  container pinning. Also trigger when the user pastes a raw syslog message
  destined for SC4S. Default: latest stable pinned SC4S release, RHEL/Rocky 8/9.
---

# Splunk Connect for Syslog (SC4S)

Router only. Read `references/sc4s.md` for all SC4S tasks.

## Routing

| Task | Action |
|---|---|
| Parser development, env_file, routing, testing, tuning | Read `references/sc4s.md` |
| Raw syslog pasted for parsing | Identify format first (RFC 3164/5424/CEF/LEEF), then read `references/sc4s.md` |

## Shared conventions

- All customization in `/opt/sc4s/local/` — never touch files under `/opt/sc4s/` directly.
- Pin container version: `ghcr.io/splunk/splunk-connect-for-syslog:<version>`. Never `latest` in prod.
- Enable disk buffering (`SC4S_DEST_SPLUNK_HEC_DEFAULT_DISKBUFF_ENABLE=yes`) — required for HEC outage resilience.
- `TLS_VERIFY=yes` always. Never `verify=False`.
- Every parser sets `.splunk.sourcetype`, `fields.sc4s_vendor`, `fields.sc4s_product`, `fields.sc4s_vendor_product`.
- `sc4s_vendor_product` must be consistent across parser, `splunk_index.csv`, and the paired Splunk TA.
- Drop filters go in early (`sc4s-postfilter`) — kill noise before it reaches HEC.
- Test every parser with `sc4s-test` before deploying.
- Never use the `default` catch-all sourcetype in production.
- Pair every SC4S parser with a TA for CIM mapping at search time.

## Syslog triage (quick)

Before writing config for a pasted log: identify format → propose sourcetype (`vendor:product`) → propose ingestion path (SC4S for syslog-over-wire, UF/HF `inputs.conf` for host files) → flag timestamp, event-breaking, and TZ issues.

## Output format

Fenced `syslog-ng` blocks for parsers. `bash` for env_file. `csv` for splunk_index.csv. Section headers: SC4S Parser / env_file / splunk_index.csv / TA Notes / Best Practice Notes. Add ⚠️ Best Practice callouts. Flag data-loss and security implications.
