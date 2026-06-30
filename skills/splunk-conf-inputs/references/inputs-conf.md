# inputs.conf Reference

Splunk Enterprise 9.x. Full spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/inputs.conf

## Stanza types

| Stanza | Purpose |
|---|---|
| `[monitor://<path>]` | Tail files/directories from disk |
| `[batch://<path>]` | One-shot ingest then delete (drop box pattern) |
| `[script://<cmd>]` | Scripted input — runs a command on interval |
| `[<protocol>://<host>:<port>]` | `udp://` or `tcp://` network listener |
| `[http]` / `[http://token-name]` | HEC global settings / per-token config |
| `[WinEventLog://<log>]` | Windows Event Log collection (UF on Windows) |
| `[monitor://winevtlog://<log>]` | Alternative Windows event input |

## Monitor input — key directives

```ini
[monitor:///var/log/secure]
index          = linux               # Required — never rely on default
sourcetype     = linux:auth          # Required
host           = myhost              # or host_segment, or _TCP_ROUTING
disabled       = false

# Dedup / hash settings
crcSalt        = <SOURCE>            # Prevents collision when files share identical headers
initCrcLength  = 512                 # Bytes used for CRC; raise if header rows are long/identical

# Noise reduction
allowlist      = \.log$              # Regex — only match files matching this pattern
denylist       = \.gz$|\.bak$        # Regex — skip rotated/compressed files

# Staleness
ignoreOlderThan = 7d                 # Skip files not modified in N days; prevents stale re-read

# Latency
followTail     = 0                   # 0 = read from beginning on first discovery (default)
```

⚠️ **Best Practice** — Always set `crcSalt = <SOURCE>` when monitoring directories where files share near-identical headers (CDR files, rolling logs). Without it, Splunk's CRC dedup can mistake a new file for one it already read and skip it entirely.

## Batch input

```ini
[batch:///opt/dropbox/*.csv]
index      = ops
sourcetype = acme:batch
move_policy = sinkhole              # Deletes file after ingest; omit to keep
```

## Scripted input

```ini
[script:///opt/splunk/etc/apps/TA-acme/bin/collect.py]
index      = ops
sourcetype = acme:metrics
interval   = 60                     # Seconds between runs
passAuth   = admin                  # Passes auth token to stdin
```

## Modular input

Modular inputs define their own stanza types via `inputs.conf.spec` in the app. Key universal options:

```ini
[<modular_input_scheme>://<name>]
index               = ops
sourcetype          = acme:api
interval            = 300
disabled            = false
checkpointInterval  = 60            # Seconds; persist checkpoint to prevent re-ingest on restart
```

⚠️ **Best Practice** — Always set `checkpointInterval` on modular inputs. A restart without checkpointing causes the input to replay from the beginning and duplicate data.

## Network inputs (UDP/TCP)

```ini
[udp://514]
index      = syslog
sourcetype = syslog
connection_host = dns               # dns | ip | none — how to derive host field

[tcp://5140]
index      = syslog
sourcetype = syslog
```

⚠️ **Best Practice** — Prefer SC4S over raw `udp://`/`tcp://` inputs for syslog. SC4S provides parsing, routing, HEC delivery with disk buffering, and sourcetype assignment that raw network stanzas cannot match.

## WinEventLog input (Windows UF)

```ini
[WinEventLog://Security]
index       = wineventlog
sourcetype  = WinEventLog:Security
disabled    = false
start_from  = oldest
evt_resolve_ad_obj = 1              # Resolve AD SIDs to names
blacklist1  = EventCode="4688"      # Drop noisy process-creation events if volume is a concern
```

## HTTP Event Collector (HEC)

Global enable in `inputs.conf` on the indexer or HF:

```ini
[http]
disabled        = false
port            = 8088
enableSSL       = true
dedicatedIoThreads = 2

[http://my-token]
token           = <hec-token>
index           = main
sourcetype      = _json
disabled        = false
```

⚠️ **Best Practice** — Never set `enableSSL = false` in production. Validate TLS on the sender side too (`verify=True`); disabling cert verification allows MITM.

## host strategies

| Setting | Effect |
|---|---|
| `host = <literal>` | Fixed host value |
| `host_segment = N` | Use the Nth path segment of the file path as host |
| `_TCP_ROUTING = default` | Route to an output group |
| `connection_host = dns` | Resolve source IP to DNS name (network inputs) |

## Precedence and layering

`local/inputs.conf` overrides `default/inputs.conf` within an app. The Deployment Server pushes apps to forwarders — never edit `local/` on a managed forwarder directly; the next phone-home will overwrite it.

## Pitfalls

- **Missing explicit `index`/`sourcetype`** — data lands in `main` with sourcetype `too_small` or similar. Always explicit.
- **`crcSalt` omitted on CDR/rolling files** — Splunk skips new files it hashes identically to old ones.
- **`followTail = 1` in production** — skips existing content on first discovery; only for live-only streams.
- **`ignoreOlderThan` too short** — drops slow-writing batch files before they're fully written.
- **Props/transforms placed on the UF** — index-time settings are ignored on UFs; they run on the parsing tier (HF or indexer).

## btool verification

```bash
splunk btool inputs list --debug            # Show merged effective config with source file per setting
splunk btool inputs list <stanza-name> --debug
```
