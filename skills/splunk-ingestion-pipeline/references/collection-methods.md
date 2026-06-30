# Choosing the collection method

Four mainstream paths. Pick by how the data leaves the source and what must happen before indexing.

## Universal Forwarder (UF) — default for files

Lightweight agent on the endpoint. Tails files, reads Windows Event Log, runs simple scripted inputs.
Does **not** parse (no timestamp/line-break decisions) — it forwards raw to a parsing tier
(indexers, or an HF).

Use for: `auth.log`/`secure`, `syslog`/`messages`, `audit.log`, app logs, Windows Security/System/
Application, Sysmon, PowerShell logs.

⚠️ **Best Practice — explicit inputs.** In `inputs.conf`, set `index=`, `sourcetype=`, and `host=`
per stanza; never rely on defaults. Use `allowlist`/`denylist` to cut noise at collection time. Set
`checkpointInterval` on modular inputs to avoid re-ingestion on restart.

## Heavy Forwarder (HF) — when you must parse/route/mask first

A full Splunk instance that parses and can transform before forwarding. Use when you need:
- masking/anonymization (SEDCMD, TRANSFORMS) before data hits the indexers,
- routing decisions (send sourcetype A to index X, B to index Y, drop C),
- API/DB pulls via modular inputs (DBConnect, REST) that need a parsing engine.

⚠️ **Best Practice — don't overuse HFs.** They cost CPU and add a hop. Only insert an HF when a
real parse/mask/route requirement exists; otherwise UF straight to indexers.

## HEC (HTTP Event Collector) — for apps/cloud pushing over HTTP

Token-authenticated HTTPS endpoint (default 8088). Apps, cloud services, and SC4S deliver here. One
token per source/use-case so you can set per-token index/sourcetype and revoke independently.

⚠️ **Best Practice — HEC hygiene.** Use HTTPS with valid certs, scope each token's allowed
index(es), and enable indexer acknowledgement where delivery guarantees matter.

## SC4S — for syslog from devices/appliances

Containerized syslog-ng that receives RFC 3164/5424/CEF/LEEF, parses and routes, then delivers to
HEC. The right path for firewalls, switches, load balancers, and any appliance emitting syslog.

⚠️ **Best Practice — SC4S, not a raw syslog UF tail.** Don't point devices at a file that a UF tails;
SC4S handles framing, source identification, parsing, and routing far better. Use TCP+TLS where the
device supports it; UDP only when forced. (Parser internals → SC4S skill.)

## Decision flow

```
Is it syslog from a device?            → SC4S → HEC
Is an app/cloud pushing over HTTP?     → HEC (dedicated token)
Is it files on a host?                 → UF  (→ HF only if masking/routing needed)
Is it an API/DB pull?                  → modular input on HF (or UF if trivial)
Need masking/anonymization pre-index?  → insert an HF in the path
```

## Where parsing happens

The **first full Splunk instance** in the path does timestamping and line-breaking: an HF if present,
otherwise the indexers. UFs and SC4S→HEC don't change that rule — set `props.conf` parsing
(`TIME_FORMAT`, `LINE_BREAKER`, `SHOULD_LINEMERGE`, `MAX_TIMESTAMP_LOOKAHEAD`) on that instance, once.
