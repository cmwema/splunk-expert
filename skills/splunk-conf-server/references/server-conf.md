# server.conf Reference

Splunk Enterprise 9.x. Full spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/server.conf

## [general]

```ini
[general]
serverName    = idx-01              # Human-readable instance name
mgmtHostPort  = 0.0.0.0:8089       # Management port binding; restrict to mgmt VLAN if possible
sessionTimeout = 1h                 # Web session timeout
```

## [diskUsage]

```ini
[diskUsage]
minFreeSpace = 10240                # MB; Splunk stops indexing and warns when free disk < this
                                    # Never set below 2000; 5000–10240 recommended for production
```

⚠️ **Best Practice** — Never let index volumes reach 80%+ capacity. Set `minFreeSpace` to at least 5–10 GB (5120–10240 MB) and monitor `splunkd.log` for `Disk space check failed` messages. When Splunk hits `minFreeSpace` it stops indexing silently from the indexer's perspective.

## [clustering] — Indexer Cluster

**Cluster Manager:**
```ini
[clustering]
mode               = manager
replication_factor = 3              # Copies of each bucket; minimum 2 for HA, 3 for production
search_factor      = 2              # Searchable copies; must be ≤ replication_factor
pass4SymmKey       = <shared-secret>
cluster_label      = prod_cluster
multisite          = false          # Set true and add site stanzas for multisite clustering
```

**Peer node:**
```ini
[clustering]
mode        = peer
manager_uri = https://cm.example.com:8089
pass4SymmKey = <shared-secret>      # Must match CM exactly
```

**Search head (cluster-aware, non-SHC):**
```ini
[clustering]
mode        = searchhead
manager_uri = https://cm.example.com:8089
pass4SymmKey = <shared-secret>
```

⚠️ **Best Practice** — `replication_factor=3`, `search_factor=2` are sensible production minimums. A peer failure with RF=2 means Splunk immediately starts emergency re-replication, which stresses the remaining peers. RF=3 gives you one failure tolerance with headroom.

## [shclustering] — Search Head Cluster

**SHC member:**
```ini
[shclustering]
shcluster_label     = shc_prod
replication_factor  = 3
pass4SymmKey        = <shared-secret>
mgmt_uri            = https://sh-01.example.com:8089   # This member's own URI
conf_deploy_fetch_url = https://deployer.example.com:8089
```

Knowledge objects replicate automatically among SHC members. App and config changes must be pushed from the **deployer** — never hand-copy apps onto members:
```bash
# On the deployer
splunk apply shcluster-bundle --answer-yes -target https://sh-01.example.com:8089
```

## [sslConfig] — TLS

```ini
[sslConfig]
enableSplunkdSSL    = true
sslVersions         = tls1.2,tls1.3        # Disable older protocols
cipherSuite         = ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256
requireClientCert   = false                # Set true for mutual TLS
sslRootCAPath       = /opt/splunk/etc/auth/cacert.pem
serverCert          = /opt/splunk/etc/auth/server.pem
sslPassword         = <cert-password>
```

⚠️ **Best Practice** — Enforce TLS 1.2+ only; disable TLS 1.0 and 1.1. Use valid CA-signed certificates in production — self-signed is acceptable for dev/test only. Never set `sslVerifyServerCert = false` on inter-Splunk communications.

## [kvstore]

```ini
[kvstore]
port        = 8191
replication = false            # Set true in SHC for KV Store replication across members
```

In an SHC, KV Store replication requires all members to be healthy before enabling — rolling changes aren't supported without care.

## [license]

```ini
[license]
master_uri = https://license-server.example.com:8089   # Peer pointing to license manager
active_group = Enterprise
```

Standalone license manager:
```ini
[license]
master_uri = self
active_group = Enterprise
```

## Cluster operations

```bash
# Show cluster status
splunk show cluster-status --verbose

# Rolling restart of indexer peers (run on CM)
splunk rolling-restart cluster-peers

# Show bundle status after push
splunk show cluster-bundle-status

# SHC rolling restart
splunk rolling-restart shcluster-members
```

⚠️ **Best Practice** — Always use `splunk rolling-restart` in clustered environments — never restart all peers simultaneously. A simultaneous restart drops search availability and can cause bucket fixup storms.

## Pitfalls

- **`pass4SymmKey` mismatch** — peers and CM fail to communicate; check `splunkd.log` for `Invalid key` errors.
- **`replication_factor` > number of peers** — Splunk enters a degraded state and emits continuous warnings; can't promote buckets to searchable.
- **Editing `server.conf` on a cluster peer's `default/`** — the CM bundle push overwrites it. Use the CM's `manager-apps/` for shared cluster config.
- **`minFreeSpace` too low** — indexer silently stops indexing when disk fills; monitor proactively.
- **SHC deployer and deployment server are the same instance** — this is unsupported. The deployer must be a separate instance.

## btool verification

```bash
splunk btool server list --debug                        # Full effective server.conf
splunk btool server list clustering --debug             # Clustering stanza only
splunk btool server list general --debug
```
