# Retention tiers and storage sizing

## The bucket lifecycle

```
hot (writing) → warm (rolled, searchable) → cold (searchable, slower disk) → frozen (deleted or archived) → thawed (manually restored)
```

- **hot/warm** — fast SSD; recently indexed, heavily searched.
- **cold** — cheaper/slower disk; older but still searchable.
- **frozen** — leaves the index: either deleted, or archived to `coldToFrozenDir`/script (then it is
  no longer searchable until thawed).
- **thawed** — frozen data manually restored for investigation.

⚠️ **Best Practice — hot/warm on SSD, cold on HDD/object.** Match tier to access pattern. Put
hot/warm on fast storage; relegate cold and frozen archive to cheaper media.

## Controlling retention

```ini
[network]
frozenTimePeriodInSecs = 7776000     # age-based: freeze after 90 days
maxTotalDataSizeMB = 500000          # size-based cap: ~500 GB then roll to frozen
# coldToFrozenDir = /archive/network # archive instead of delete (optional)
```

Whichever limit hits first (age or size) triggers freezing. Set both deliberately; relying on the
default size cap silently deletes old data.

## Volumes centralize tier sizing

Define storage volumes once and point indexes at them, so you size the tier, not each index:

```ini
[volume:hot]
path = /opt/splunk/hot
maxVolumeDataSizeMB = 2000000

[volume:cold]
path = /mnt/cold
maxVolumeDataSizeMB = 8000000
```

## Sizing estimate

⚠️ **Best Practice — the ~10:1 raw-to-index guideline.** Splunk's on-disk footprint (rawdata +
tsidx) lands near **50% of raw** for typical data, but plan with margin. A common planning rule:

```
daily_index_GB ≈ daily_raw_GB × ~0.5         (rawdata compresses; tsidx adds index overhead)
retention_GB   ≈ daily_index_GB × retention_days
× replication_factor (clustered indexers store RF copies)
× search_factor considerations for tsidx
```

Worked example: 100 GB/day raw, 90-day retention, RF=3:
`100 × 0.5 = 50 GB/day indexed × 90 = 4.5 TB × 3 (RF) ≈ 13.5 TB` across the cluster, before headroom.

CIM **acceleration** adds further tsidx summary storage on top — factor it in when models are
accelerated.

## Disk safety

⚠️ **Best Practice — never let index volumes exceed ~80%.** Configure `minFreeSpace` in `server.conf`
so Splunk stops indexing before the disk fills (indexing halts rather than corrupting). Monitor with
`df -h`, and watch index growth — it fills volumes faster than expected after onboarding new sources.

```ini
# server.conf
[diskUsage]
minFreeSpace = 50000    # MB; Splunk pauses indexing below this
```

## Next steps

1. Set both `frozenTimePeriodInSecs` and a size cap per index intentionally.
2. Confirm RF/SF multipliers in the cluster sizing math.
3. Add the new index's projected daily volume to the license/storage forecast.
