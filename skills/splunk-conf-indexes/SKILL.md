---
name: splunk-conf-indexes
description: >-
  indexes.conf — all tasks. Trigger for: defining indexes, index creation,
  bucket lifecycle, homePath, coldPath, thawedPath, frozenTimePeriodInSecs,
  maxTotalDataSizeMB, maxHotBuckets, maxDataSize, auto_high_volume, volume
  stanzas, hot/warm/cold/frozen tiers, coldToFrozenDir, coldToFrozenScript,
  maxConcurrentOptimizes, retention policy, index sizing, cluster manager
  manager-apps, master-apps, pushing indexes to peers. Default: Splunk Enterprise 9.x.
---

# indexes.conf

Router only. Read `references/indexes-conf.md` for all indexes.conf tasks.

## Routing

| Task | Action |
|---|---|
| Any indexes.conf directive — define, tune, size, freeze | Read `references/indexes-conf.md` |

## Shared conventions

- In an indexer cluster: define indexes **only** in the CM's `manager-apps` (or `master-apps`) bundle and push to peers. Never edit `indexes.conf` directly on a peer.
- Validate `indexes.conf` on the CM before pushing — a bad definition affects every peer simultaneously.
- Separate indexes by data domain (`windows`, `network`, `linux`, `syslog`) for RBAC, retention, and search efficiency.
- Use `volume:` stanzas to manage total disk usage; place hot/warm on SSD, cold on HDD.
- `frozenTimePeriodInSecs` controls roll-to-frozen; set `coldToFrozenDir` or `coldToFrozenScript` to archive instead of delete.
- Estimate storage with the ~10:1 raw-to-index ratio guideline.

## Output format

Fenced `ini` blocks. Full stanzas with comments. ⚠️ Best Practice callouts. Flag retention, sizing, and cluster-push implications.

## Spec reference

Full indexes.conf spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/indexes.conf
