# indexes.conf Reference

Splunk Enterprise 9.x. Full spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/indexes.conf

## Cluster deployment rule

In an indexer cluster, **never edit `indexes.conf` on a peer directly**. Define indexes in the CM's `manager-apps/` bundle and push:
```bash
splunk apply cluster-bundle --answer-yes   # Run on the CM
splunk show cluster-bundle-status          # Verify push
```

⚠️ **Best Practice** — Validate `indexes.conf` on the CM (`splunk btool indexes list --debug`) before applying the bundle. A misconfigured index definition propagates to every peer and can cause bucket errors or data loss.

## Standard event index

```ini
[security]
homePath    = volume:hot/security/db           # Hot + warm bucket location
coldPath    = volume:cold/security/colddb       # Cold bucket location
thawedPath  = $SPLUNK_DB/security/thaweddb     # Thawed (manually restored) bucket path

# Bucket sizing
maxHotBuckets       = 10                       # Hot buckets open at once per index
maxDataSize         = auto_high_volume          # auto | auto_high_volume | <MB>
                                                # auto_high_volume = 10GB buckets (high-ingest indexes)

# Retention
frozenTimePeriodInSecs = 7776000               # 90 days; events older than this are frozen
maxTotalDataSizeMB     = 512000                # 500 GB cap; older data frozen when exceeded

# Archive frozen data instead of deleting it (one of these, not both)
coldToFrozenDir    = /archive/splunk/security   # Move frozen buckets here
# coldToFrozenScript = /opt/scripts/freeze.py  # Or call a script instead
```

## Volume stanzas

Volumes manage total disk usage across indexes. Use them to enforce hard limits per storage tier.

```ini
[volume:hot]
path            = /opt/splunk/hot              # Fast SSD-backed path
maxVolumeDataSizeMB = 2097152                  # 2 TB cap across all hot/warm indexes on this volume

[volume:cold]
path            = /opt/splunk/cold             # HDD or cheaper storage
maxVolumeDataSizeMB = 10485760                 # 10 TB cap
```

Reference volumes in index stanzas via `volume:<name>/`:
```ini
[network]
homePath = volume:hot/network/db
coldPath = volume:cold/network/colddb
```

⚠️ **Best Practice** — Always use `volume:` stanzas in production. Setting `maxTotalDataSizeMB` per index without volume stanzas can over-provision total disk if many indexes are near their limits simultaneously.

## Metrics index

```ini
[metrics_ops]
datatype    = metric                           # Declares this as a metrics index
homePath    = volume:hot/metrics_ops/db
coldPath    = volume:cold/metrics_ops/colddb
thawedPath  = $SPLUNK_DB/metrics_ops/thaweddb
frozenTimePeriodInSecs = 2592000               # 30 days — metrics typically need shorter retention
maxTotalDataSizeMB     = 51200                 # 50 GB
```

Metrics indexes only accept events formatted as Splunk metrics (via HEC `event_type=metric` or the Metrics input). Do not mix event and metric data in the same index.

## Summary index

```ini
[summary_netsec]
homePath    = volume:hot/summary_netsec/db
coldPath    = volume:cold/summary_netsec/colddb
thawedPath  = $SPLUNK_DB/summary_netsec/thaweddb
frozenTimePeriodInSecs = 31536000              # 1 year — summaries often kept longer than raw
maxTotalDataSizeMB     = 20480                 # 20 GB
```

## Standard domain index set

```ini
# Recommended separation by domain for RBAC, retention, and search efficiency
# Adjust maxTotalDataSizeMB and frozenTimePeriodInSecs per your retention policy

[windows]
homePath    = volume:hot/windows/db
coldPath    = volume:cold/windows/colddb
thawedPath  = $SPLUNK_DB/windows/thaweddb
frozenTimePeriodInSecs = 7776000    # 90 days
maxTotalDataSizeMB     = 204800     # 200 GB

[linux]
homePath    = volume:hot/linux/db
coldPath    = volume:cold/linux/colddb
thawedPath  = $SPLUNK_DB/linux/thaweddb
frozenTimePeriodInSecs = 7776000
maxTotalDataSizeMB     = 102400

[network]
homePath    = volume:hot/network/db
coldPath    = volume:cold/network/colddb
thawedPath  = $SPLUNK_DB/network/thaweddb
frozenTimePeriodInSecs = 7776000
maxTotalDataSizeMB     = 204800

[syslog]
homePath    = volume:hot/syslog/db
coldPath    = volume:cold/syslog/colddb
thawedPath  = $SPLUNK_DB/syslog/thaweddb
frozenTimePeriodInSecs = 2592000    # 30 days — general syslog often has shorter retention
maxTotalDataSizeMB     = 512000
```

## Key directives

| Directive | Effect |
|---|---|
| `maxHotBuckets` | Max simultaneous hot buckets; higher = more RAM and handles bursts better |
| `maxDataSize = auto` | ~750 MB buckets; `auto_high_volume` = ~10 GB; or explicit MB |
| `frozenTimePeriodInSecs` | Events older than this are rolled to frozen; **this is age since indexing, not event time** |
| `maxTotalDataSizeMB` | Hard cap; when exceeded, oldest data freezes regardless of `frozenTimePeriodInSecs` |
| `coldToFrozenDir` | Move frozen buckets to this path instead of deleting |
| `repFactor = auto` | Required in clustered environments; `auto` = replicate per cluster RF |
| `datatype = event` | Default; `metric` for metrics indexes |

## Sizing guidance

- **10:1 raw-to-index ratio** — 1 GB raw log ≈ 100 MB indexed (with compression and tsidx overhead).
- **Hot bucket RAM** — each open hot bucket holds ~50–100 MB in memory; `maxHotBuckets` × indexes affects RAM.
- **Search factor** — `search_factor` copies must exist for searches to run; set on the CM, not per-index.
- High-volume indexes (`maxDataSize = auto_high_volume`) create fewer, larger buckets — better for ingest throughput, slightly slower for small-range searches.

## Pitfalls

- **Editing `indexes.conf` on a peer directly** — the CM overwrites it on the next bundle push.
- **`thawedPath` on a volume** — thawed paths must be literal paths, not `volume:` references.
- **`frozenTimePeriodInSecs = 0`** — means events are frozen immediately (never stored); use with extreme caution.
- **`maxTotalDataSizeMB` too small** — data freezes early and silently; check `splunk list index` for disk usage.
- **Not setting `repFactor = auto` in a cluster** — index won't replicate; data loss risk if a peer fails.

## btool and verification

```bash
splunk btool indexes list <index-name> --debug   # Effective merged config
splunk list index                                  # Show all indexes with disk usage
splunk show cluster-bundle-status                  # Verify bundle push on CM
```
