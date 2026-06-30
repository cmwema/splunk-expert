---
name: splunk-conf-server
description: >-
  server.conf — all tasks. Trigger for: serverName, pass4SymmKey, diskUsage,
  minFreeSpace, clustering mode, replication_factor, search_factor,
  manager_uri, shclustering, shcluster_label, SSL/TLS settings, kvstore,
  licenser, general server settings, mgmtHostPort, web settings, sslConfig.
  Default: Splunk Enterprise 9.x.
---

# server.conf

Router only. Read `references/server-conf.md` for all server.conf tasks.

## Routing

| Task | Action |
|---|---|
| Any server.conf directive — general, clustering, SHC, SSL, disk | Read `references/server-conf.md` |

## Shared conventions

- `pass4SymmKey` must be identical across all members of a given cluster — CM + peers, SHC deployer + members.
- Store `pass4SymmKey` in a secrets manager; never commit it to Git in plaintext.
- `minFreeSpace` protects the instance from running out of disk — set to at least 5–10 GB (5120–10240 MB).
- Editing `server.conf` in a cluster requires a coordinated restart; use rolling restarts.
- `local/server.conf` overrides `default/server.conf`; always layer in `local/`.

## Output format

Fenced `ini` blocks. Full stanzas. ⚠️ Best Practice callouts. Flag cluster, SSL, and restart implications.

## Spec reference

Full server.conf spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/server.conf
