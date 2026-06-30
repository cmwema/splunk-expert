# Distributed Architecture — Deployment Server, Indexer Cluster, SHC

Reference for distributed Splunk: deployment server (DS), indexer clustering, search head clustering
(SHC), sizing, and safe bundle operations.

## Contents
- Roles and separation
- Deployment server
- Indexer cluster (manager + peers)
- Search head cluster
- Monitoring Console
- Sizing and best practices

## Roles and separation

⚠️ **Best Practice** — Dedicate roles. The Cluster Manager (CM) must not run on an indexer peer. The
SHC **deployer** is separate from the **deployment server**. The Monitoring Console (MC) runs on its
own instance, not on a search head or CM. Co-locating these roles causes resource contention and
makes failures cascade.

`pass4SymmKey` must be consistent and securely managed across members of a given cluster.

## Deployment server

Manages app distribution to forwarders via `serverclass.conf`.

```ini
[serverClass:linux_uf]
whitelist.0 = host-linux-*
blacklist.0 = host-linux-lab-*
restartSplunkd = true
stateOnClient = enabled

[serverClass:linux_uf:app:TA-nix]
stateOnClient = enabled
restartSplunkd = false
```

- `whitelist`/`blacklist` select clients by host pattern; design class hierarchies to minimize app
  conflicts across forwarder groups.
- Forwarders use `deploymentclient.conf` to phone home; troubleshoot connectivity and the phone-home
  interval there.

⚠️ **Best Practice** — A single DS scales to roughly 1000–2000 UFs before you should consider
splitting load or alternative distribution. Beyond that, phone-home churn degrades responsiveness.

## Indexer cluster (manager + peers)

`server.conf` on the CM and peers:

```ini
# Cluster Manager
[clustering]
mode = manager
replication_factor = 3
search_factor = 2
pass4SymmKey = <shared-secret>

# Peer node
[clustering]
mode = peer
manager_uri = https://cm.example.com:8089
pass4SymmKey = <shared-secret>
```

- Define `indexes.conf` once in the CM's manager-apps bundle; push to peers. Never edit a peer's
  `indexes.conf` directly.
- Operations: `splunk show cluster-status`, bucket fixup, peer recovery, and rolling restarts.

⚠️ **Best Practice** — `replication_factor=3` and `search_factor=2` are sensible production minimums.
Validate `indexes.conf` on the CM before pushing — a bad index def propagates to every peer. Always
`splunk rolling-restart` rather than restarting peers simultaneously.

## Search head cluster

```ini
[shclustering]
shcluster_label = shc_prod
replication_factor = 3
pass4SymmKey = <shared-secret>
# captain election handled by the cluster; see conf_deploy_fetch_url for deployer
```

- Knowledge objects replicate automatically among members; **app/config changes** are pushed from the
  **deployer** via `splunk apply shcluster-bundle`.
- A load balancer fronts the members; configure health checks and session persistence.

⚠️ **Best Practice** — Use `splunk apply shcluster-bundle` to push app changes — it coordinates a
rolling update without taking the cluster down. Don't hand-copy apps onto individual members.

## Monitoring Console

Configure the MC on a dedicated instance as the single pane of glass for distributed health. Watch
`splunkd.log` on the CM and SHC captain for replication and bundle-push errors; watch `scheduler.log`
for skipped searches.

## Sizing and best practices

- Estimate index storage with the ~10:1 raw-to-index ratio guideline.
- Plan indexer count from daily ingest volume, retention, replication/search factors, and concurrent
  search load — not ingest alone.
- Keep `pass4SymmKey` in a secrets manager; rotate carefully (requires coordinated restarts).
- Use configuration management (Ansible/Puppet/Chef) and Git for all cluster config.
