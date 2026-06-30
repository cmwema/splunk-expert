# props.conf Reference

Splunk Enterprise 9.x. Full spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/props.conf

## Stanza keys

| Key type | Example | Matches |
|---|---|---|
| `[<sourcetype>]` | `[linux:auth]` | All events of that sourcetype |
| `[source::<glob>]` | `[source::...access.log]` | Events from matching file paths |
| `[host::<glob>]` | `[host::web-*]` | Events from matching hostnames |

Sourcetype stanzas are preferred — they're portable across hosts and paths.

## Event breaking

```ini
[acme:applog]
SHOULD_LINEMERGE = false            # Disable heuristic line merging — always set explicitly
LINE_BREAKER = ([\r\n]+)\d{4}-\d{2}-\d{2}  # Regex: break before each ISO-date line
# For truly single-line events (most common case):
# SHOULD_LINEMERGE = false
# LINE_BREAKER = ([\r\n]+)
```

⚠️ **Best Practice** — `SHOULD_LINEMERGE = false` + explicit `LINE_BREAKER` is the reliable combo. Relying on heuristic merging (`SHOULD_LINEMERGE = true` with no `LINE_BREAKER`) is the most common cause of mis-broken or fused events. If events are multiline (Java stack traces, JSON blobs), use `BREAK_ONLY_BEFORE` or `MUST_BREAK_AFTER` alongside `SHOULD_LINEMERGE = true`.

## Timestamping

```ini
[acme:applog]
TIME_PREFIX = ^                     # Where in the event the timestamp starts
TIME_FORMAT = %Y-%m-%d %H:%M:%S.%3N%z  # strptime format — always explicit for custom logs
MAX_TIMESTAMP_LOOKAHEAD = 32        # Characters to scan from TIME_PREFIX; cap it tightly
MAX_DAYS_AGO = 10                   # Reject timestamps more than N days in the past
MAX_DAYS_HENCE = 2                  # Reject timestamps more than N days in the future
TZ = UTC                            # Force timezone when the log carries no TZ
```

⚠️ **Best Practice** — Always set `TIME_FORMAT`, `MAX_TIMESTAMP_LOOKAHEAD`, and `MAX_DAYS_AGO` for custom sourcetypes. Without them, Splunk falls back to heuristic timestamp detection that silently scatters events across the timeline and breaks time-based searches.

Common `TIME_FORMAT` tokens:

| Token | Meaning |
|---|---|
| `%Y` | 4-digit year |
| `%m` | 2-digit month |
| `%d` | 2-digit day |
| `%H:%M:%S` | 24-hour time |
| `%3N` | Milliseconds |
| `%z` | UTC offset (`+0000`) |
| `%Z` | Timezone name (`UTC`) |
| `%s` | Unix epoch (seconds) |

## Field extractions (search-time)

```ini
[acme:applog]
# Inline regex — named capture groups become fields
EXTRACT-session = session_id=(?<session_id>[A-F0-9]+)

# Reference a transforms.conf stanza (for reusable or complex regex)
REPORT-kvpairs = acme_kvpairs

# Key-value auto-extraction
KV_MODE = auto                      # auto | none | multi | json | xml

# JSON auto-extraction (for JSON sourcetypes)
KV_MODE = json
```

## Field aliases and eval (CIM normalization)

```ini
[acme:fw]
# Alias source fields to CIM names — zero cost, no data duplication
FIELDALIAS-src    = source_address AS src
FIELDALIAS-dest   = dest_address AS dest
FIELDALIAS-user   = username AS user

# Computed fields — eval expression run at search time
EVAL-action = case(verdict=="permit","allowed", verdict=="deny","blocked", true(),"unknown")
EVAL-bytes  = coalesce(bytes_in, 0) + coalesce(bytes_out, 0)
```

⚠️ **Best Practice** — Normalize CIM fields at search time via `FIELDALIAS`/`EVAL`. Index-time CIM fields bloat the index and are irreversible without re-indexing. `FIELDALIAS` has zero compute cost; `EVAL` at search time adds minimal CPU.

## Lookup (automatic)

```ini
[acme:applog]
# Automatic lookup — runs at search time for every search against this sourcetype
LOOKUP-category = app_lookup app_id OUTPUT category, subcategory
```

Only use automatic lookups when every search against this sourcetype needs the enrichment. For selective enrichment, call `| lookup` explicitly in SPL.

## Transforms (index-time)

```ini
[acme:applog]
# Index-time transforms — run on the parsing tier (HF/indexer), not the UF
TRANSFORMS-route  = route_to_security   # References a transforms.conf stanza
TRANSFORMS-mask   = mask_ssn
```

Index-time `TRANSFORMS` are for routing, masking, and metric extraction — not for field normalization. Put them on the HF or indexer, never the UF.

## Truncation

```ini
[acme:applog]
TRUNCATE = 10000                    # Max raw event size in bytes; default 10000
                                    # Increase for long JSON blobs or stack traces
```

## Useful combinations

**Single-line JSON log:**
```ini
[acme:api]
SHOULD_LINEMERGE = false
LINE_BREAKER     = ([\r\n]+)
KV_MODE          = json
TIME_PREFIX      = "timestamp":"
TIME_FORMAT      = %Y-%m-%dT%H:%M:%S.%3NZ
MAX_TIMESTAMP_LOOKAHEAD = 30
MAX_DAYS_AGO     = 7
TZ               = UTC
```

**Multiline Java stack trace:**
```ini
[java:applog]
SHOULD_LINEMERGE = true
BREAK_ONLY_BEFORE = \d{4}-\d{2}-\d{2}   # New event starts at a dated line
TIME_PREFIX      = ^
TIME_FORMAT      = %Y-%m-%d %H:%M:%S,%3N
MAX_TIMESTAMP_LOOKAHEAD = 26
MAX_DAYS_AGO     = 5
```

## Pitfalls

- **Editing `default/` instead of `local/`** — upgrades overwrite `default/`. Always layer in `local/`.
- **Index-time settings on a UF** — `TRANSFORMS-` and `SEDCMD-` on a UF are silently ignored.
- **Forgetting `SHOULD_LINEMERGE = false`** with a custom `LINE_BREAKER` — Splunk may still merge.
- **`TIME_FORMAT` that doesn't match the actual log** — events timestamp as current time (very common and hard to notice until a search shows wrong results).
- **`FIELDALIAS` aliasing a non-existent source field** — the alias silently produces nothing; verify field names against real events.

## btool verification

```bash
splunk btool props list <sourcetype> --debug      # Effective merged config per setting
splunk btool props list source::<glob> --debug    # Source-keyed stanza
```
