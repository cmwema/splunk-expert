---
name: splunk-kvstore
description: >-
  Splunk KV Store — all tasks. Trigger for: KV Store collections, collections.conf,
  transforms.conf KV Store lookup, inputlookup/outputlookup against KV Store,
  KV Store REST API (/services/kvstore/collections/data), kvstore.conf, KV Store
  replication in SHC, KV Store backup and restore, KV Store size limits,
  KV Store accelerated fields, enforced_auto_lookup, KV Store Python SDK access,
  collection sharing, `_key` field, bulk upsert, DELETE from KV Store, KV Store
  migration, KV Store troubleshooting (replication lag, disk full, lock contention).
  Default: Splunk Enterprise 9.x.
---

# Splunk KV Store

Router only. Read `references/kvstore.md` for all KV Store tasks.

## Routing

| Task | Action |
|---|---|
| Define a collection (collections.conf + transforms.conf) | Read `references/kvstore.md` |
| Read/write via SPL (inputlookup / outputlookup) | Read `references/kvstore.md` |
| Read/write via REST API | Read `references/kvstore.md` |
| SHC replication, backup, restore | Read `references/kvstore.md` |
| Static CSV lookup (not KV Store) | Defer to `splunk-conf-transforms` |

## Shared conventions

- Define collection schema in `collections.conf`; expose as a lookup in `transforms.conf` with `external_type = kvstore`.
- Always declare field types in `collections.conf` (`field.<name> = string|number|boolean|time|array|cidr`) — untyped fields default to string.
- Add `accelerated_fields.<name>` for fields used in lookup WHERE clauses or REST filter queries.
- `_key` is the unique primary key — set it explicitly on upsert for idempotent writes.
- REST bulk upsert: POST JSON array to `/services/kvstore/collections/data/batch_save` — max 1000 records per request.
- SHC: KV Store replicates automatically across SHC members; avoid direct writes during elections.
- Backup: `splunk backup kvstore`; restore: `splunk restore kvstore` — include in DR runbooks.
- Monitor: `index=_internal sourcetype=splunkd component=KVStoreProvider` for lag and errors.
- Size limits: default 1 GB per node; tune `maxSize` in `kvstore.conf` cautiously — oversized collections degrade SHC replication.

## Output format

Fenced `ini` for conf files, `splunk` for SPL examples, `python`/`bash`/`curl` for REST examples. ⚠️ callouts for schema design, replication, and backup implications.
