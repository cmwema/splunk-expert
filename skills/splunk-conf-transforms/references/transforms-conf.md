# transforms.conf Reference

Splunk Enterprise 9.x. Full spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/transforms.conf

## How transforms are invoked

| props.conf directive | Runs at | Purpose |
|---|---|---|
| `REPORT-<name> = <stanza>` | Search time | Regex field extraction |
| `TRANSFORMS-<name> = <stanza>` | Index time (parsing tier) | Routing, masking, metric extraction |
| `LOOKUP-<name> = <stanza> ...` | Search time | Automatic lookup enrichment |

## Regex field extraction (search-time, via REPORT-)

```ini
[acme_session]
REGEX  = session_id=(?<session_id>[A-F0-9]{32})
```

Paired with `props.conf`:
```ini
[acme:applog]
REPORT-session = acme_session
```

Use named capture groups (`(?<fieldname>...)`). Multiple groups in one stanza are fine.

## Index routing (index-time, via TRANSFORMS-)

Route events matching a regex to a specific index at parse time. Runs on the HF or indexer — not the UF.

```ini
[route_to_security]
REGEX    = (?i)(failed password|authentication failure|privilege escalation)
DEST_KEY = _MetaData:Index
FORMAT   = security
```

Paired with `props.conf`:
```ini
[linux:auth]
TRANSFORMS-route = route_to_security
```

⚠️ **Best Practice** — Index routing decisions run on the parsing tier. Placing `TRANSFORMS-` in a UF's `props.conf` has no effect — the transform is silently skipped. Put it on the HF or the indexer's `props.conf`.

## Null queue (drop events)

Drop events matching a regex before they are indexed:

```ini
[drop_healthcheck]
REGEX    = /health-check|/ping
DEST_KEY = queue
FORMAT   = nullQueue
```

⚠️ **Best Practice** — Drop noisy events as early as possible in the pipeline (HF or SC4S drop filter) to reduce indexer load. Dropped events do not appear in `_internal` metrics — verify drops using a test sourcetype first.

## Data masking / anonymization (index-time)

Irreversible. Test in non-prod. Runs on the parsing tier.

```ini
[mask_ssn]
REGEX  = (\d{3})-(\d{2})-(\d{4})
FORMAT = $1-XX-XXXX
DEST_KEY = _raw                     # Modifies the raw event text in place
```

```ini
[mask_cc]
REGEX    = (?<!\d)(\d{4}[\s-]?\d{4}[\s-]?\d{4})[\s-]?(\d{4})(?!\d)
FORMAT   = XXXX-XXXX-XXXX-$2
DEST_KEY = _raw
```

Paired with `props.conf`:
```ini
[acme:payments]
TRANSFORMS-mask_ssn = mask_ssn
TRANSFORMS-mask_cc  = mask_cc
```

## CSV lookup

```ini
[app_lookup]
filename    = app_categories.csv    # File in lookups/ directory of the app
max_matches = 1                     # Return only the best match
match_type  = WILDCARD(app_name)    # Optional wildcard matching on a key field
```

`app_categories.csv` example:
```
app_id,category,subcategory
1001,finance,payments
1002,hr,onboarding
```

## KV Store lookup

```ini
[user_profile_lookup]
collection  = user_profiles         # KV Store collection name
fields_list = user, department, role, manager
external_type = kvstore
```

## Scripted (external) lookup

```ini
[threat_intel_lookup]
filename    = threat_lookup.py      # Script in bin/ directory
fields_list = src_ip, threat_score, threat_category
```

⚠️ **Best Practice** — Never hardcode credentials in `external_cmd` or the script itself. Use `passwords.conf` via the Splunk secret store and retrieve the credential in the script at runtime via the REST API or the SDK.

## Clone sourcetype (index-time copy)

Send a copy of matching events to a second index under a different sourcetype:

```ini
[clone_to_archive]
REGEX        = .
CLONE_SOURCETYPE = acme:applog:archive
DEST_KEY     = _MetaData:Index
FORMAT       = archive
```

## Metric extraction (index-time, for metrics indexes)

```ini
[acme_metrics]
REGEX         = host=(?<host>[^\s]+) cpu=(?<cpu>\d+(\.\d+)?)
METRIC-SCHEMA-TRANSFORMS = acme_metrics_fields
```

## Pitfalls

- **`TRANSFORMS-` on a UF** — index-time transforms on a UF are silently ignored.
- **Masking regex that doesn't match** — data passes through unmasked with no error logged; always verify with `splunk extract` or a test event.
- **`max_matches` too high on large CSVs** — performance impact at search time; keep it low or use KV Store for large datasets.
- **Filename path in lookup stanza** — always just the filename, no path prefix. Splunk resolves it from `lookups/` inside the app.
- **`DEST_KEY = _raw` masking after parsing** — if the sourcetype's `props.conf` already extracted fields, masking `_raw` doesn't remove the extracted field values; they're already in memory.

## btool verification

```bash
splunk btool transforms list <stanza-name> --debug
splunk btool transforms list --debug | grep -A5 "<stanza>"
```
