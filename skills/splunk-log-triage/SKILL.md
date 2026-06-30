---
name: splunk-log-triage
description: >-
  Raw log or syslog sample identification and triage for Splunk ingestion.
  Trigger when the user pastes a raw log line, syslog message, or log sample
  and wants to know: what is this, what sourcetype should it be, how should it
  be ingested (SC4S vs UF/HF inputs.conf), what field extractions apply, what
  CIM data model it maps to, or what timestamp/event-breaking issues exist.
  Covers RFC 3164, RFC 5424, CEF, LEEF, JSON, key-value, and raw formats.
---

# Log-Sample Triage

Router only. Read `references/log-triage.md` for the full triage protocol.

## Routing

| Task | Action |
|---|---|
| Pasted raw log/syslog for identification, parsing, or ingestion path | Read `references/log-triage.md` |
| After triage: syslog-over-wire path | Hand off to splunk-sc4s skill |
| After triage: file/scripted/API path | Hand off to splunk-conf-inputs + splunk-conf-props skills |

## Triage protocol (quick)

Before writing any config:
1. Identify format (RFC 3164 / RFC 5424 / CEF / LEEF / JSON / KV / raw) and vendor/product.
2. Propose sourcetype (`vendor:product` convention) and index domain.
3. Propose field extractions and CIM data model mapping.
4. Recommend ingestion path: syslog-over-wire → SC4S; file/API/host log → UF/HF `inputs.conf`.
5. Flag: timestamp ambiguity, event-breaking risk, missing TZ, truncation, CIM gaps.

State the identification in plain language first. Then produce the config.

## Output format

Plain language identification first. Then fenced code blocks: `syslog-ng` for SC4S parsers, `ini` for props/inputs, `splunk` for extraction examples. ⚠️ callouts for data-quality issues found during triage.
