# Log-Sample Triage

When the user pastes raw log data, follow this protocol before proposing any config. The goal is to
correctly identify the data, then route to `conf-files.md` (UF/HF ingest) or `sc4s.md` (syslog).

## Step 1 — Identify format and source

Recognize the wire format first:

| Signal | Likely format |
|---|---|
| `<134>Jan 02 15:04:05 host program[pid]:` | RFC 3164 (BSD syslog) — no year, no TZ |
| `<134>1 2024-01-02T15:04:05.000Z host app pid msgid` | RFC 5424 — full timestamp, structured data |
| `CEF:0\|Vendor\|Product\|Version\|...\|` | CEF (ArcSight Common Event Format) |
| `LEEF:1.0\|Vendor\|Product\|...` | LEEF (QRadar) |
| `{ "...": ... }` | JSON |
| `key=value key2=value2` | key-value (use `KV_MODE`/`kv-parser`) |
| free text | raw — needs regex extraction |

Then name the vendor/product from signatures in the message (program tag, CEF vendor field, distinctive tokens).

## Step 2 — Propose sourcetype and category

- Use `vendor:product` (Splunk TA convention) or `vendor_product` (SC4S convention) consistently.
- Pick a category that maps to an index domain (network, security, linux, windows, web).

## Step 3 — Propose extractions and CIM mapping

- KV or JSON → use `KV_MODE`/auto-kv or `json` extraction rather than handwritten regex.
- Raw/free text → write anchored regex named-capture groups.
- Identify the applicable CIM data model and the source→CIM field mapping (e.g.,
  `src_ip → src`, `usr → user`, verdict → `action`).

## Step 4 — Recommend ingestion path

- **Syslog over the wire (UDP/TCP/TLS from a device)** → SC4S. Go to `sc4s.md`.
- **File on a host, Windows event log, scripted/API source** → UF/HF `inputs.conf`. Go to
  `conf-files.md`.
- Network devices and appliances almost always → SC4S; host/app logs → UF.

## Step 5 — Flag data-quality issues

Call out, before writing config:
- **Timestamp ambiguity** — RFC 3164 has no year/TZ; mixed locales; epoch vs formatted.
- **Event breaking risk** — multiline stack traces; need explicit `LINE_BREAKER`.
- **Missing timezone** — will need `TZ`/`guess-timezone`.
- **Truncation** — lines near `TRUNCATE`/message-length limits.
- **CIM gaps** — fields the target data model requires that the source doesn't provide.

## Output

State the identification and quality flags in plain language first (the user is a practitioner and
wants to know *what it is* before the config). Then produce the full config from the relevant
reference: SC4S parser + `splunk_index.csv` + TA props for the syslog path, or
`inputs.conf` + props/transforms for the UF/HF path. Always pair an SC4S parser with a Splunk TA for
search-time CIM mapping.
