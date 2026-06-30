---
name: splunk-distributed
description: >-
  Distributed Splunk architecture — deployment server, indexer clustering,
  search head clustering (SHC). Trigger for: serverclass.conf, deployment
  server, deploymentclient.conf, forwarder management, indexer cluster,
  cluster manager, CM, manager-apps, master-apps, replication_factor,
  search_factor, pass4SymmKey, SHC deployer, splunk apply shcluster-bundle,
  rolling restart, Monitoring Console, MC, bundle push, cluster-bundle,
  sizing, phone-home interval, UF scaling. Default: Splunk Enterprise 9.x.
---

# Splunk Distributed Architecture

Router only. Read `references/distributed-arch.md` for all distributed-architecture tasks.

## Routing

| Task | Action |
|---|---|
| Deployment server, serverclass.conf, forwarder management | Read `references/distributed-arch.md` |
| Indexer cluster — CM, peers, bundle push, rolling restart | Read `references/distributed-arch.md` |
| SHC — deployer, bundle push, captain election, KV Store replication | Read `references/distributed-arch.md` |
| Monitoring Console, sizing, capacity planning | Read `references/distributed-arch.md` |

## Shared conventions

- Dedicated CM — never co-locate the CM role on an indexer peer.
- Dedicated deployer for SHC — separate from the deployment server.
- Monitoring Console on a dedicated instance — not on a search head or CM.
- `pass4SymmKey` must be identical across all cluster members; store in a secrets manager.
- Always `splunk rolling-restart` in clusters — never simultaneous restarts.
- Validate `indexes.conf` on the CM before pushing — a bad definition affects every peer.
- Push SHC app changes via `splunk apply shcluster-bundle` from the deployer — never hand-copy.

## Output format

Fenced `ini` for conf files, `bash` for CLI commands. Section headers: Deployment Server / Indexer Cluster / SHC / Monitoring Console. ⚠️ Best Practice callouts. Flag HA, data-loss, and restart implications.
