# Troubleshooting & scaling the Deployment Server

## Forwarder not receiving an app — diagnostic order

1. **Is the client connected?** `splunk list deploy-clients` on the DS, or Forwarder Management UI.
   If absent: check `targetUri`, network/firewall to DS:8089, and that the client restarted after
   setting the pointer.
2. **Does the client match the class?** Whitelist/blacklist patterns are case-sensitive glob/regex.
   A blacklist entry silently excludes. Test the pattern against the client's actual reported
   hostname/clientName (what the client *reports*, not what you think it's called).
3. **Is the app mapped to that class?** Confirm a `[serverClass:<class>:app:<app>]` stanza exists and
   `<app>` matches the directory name under `deployment-apps/` exactly.
4. **Did you reload?** `splunk reload deploy-server`. New apps and serverclass edits don't take effect
   until reload.
5. **Checksum unchanged?** The DS only pushes when the app's content hash changes. If you edited a
   file the DS doesn't checksum (or didn't actually change anything), nothing ships. Bump something
   real or re-create the app dir.

## Phone-home / log signals

On the client `splunkd.log`:
- `DeploymentClient - Handshake` lines confirm it's reaching the DS.
- `DeployedApplication - Installing ... <app>` confirms a pull.
- Handshake errors → wrong `targetUri`, TLS mismatch, or DS overloaded.

On the DS `splunkd.log`: look for client-handling errors and reload completion.

## Apps flapping (enabling/disabling each cycle)

Classic dual-management symptom: the app is managed by the DS *and* edited locally, or mapped to two
classes with conflicting `stateOnClient`. Pick one owner; align `stateOnClient` across all classes
the client matches.

## Restart storms

If a deploy triggers fleet-wide restarts you didn't intend, an app/class has `restartSplunkd = true`.
Audit every class the affected clients match — the union applies. Set to `false` wherever a restart
isn't actually required.

## Scaling limits

⚠️ **Best Practice — one DS handles roughly 1,000–2,000 UFs** before phone-home load and reload
times degrade. Approaching that:

- **Raise `phoneHomeIntervalInSecs`** on clients (300–600s) to cut request volume.
- **Reduce serverclass complexity** — fewer, simpler classes reload faster.
- **Scale out**: multiple DS instances behind a load balancer, or a tiered DS topology (a DS that
  itself deploys the DS-config app to lower-tier DSes). Keep `serverclass.conf` consistent across DS
  instances (manage it as code).
- Consider whether some config belongs in a base image / config-management (Ansible) instead of the
  DS at very large scale.

## DS is not for clusters

Indexer-cluster apps go through the **Cluster Manager** (`splunk apply cluster-bundle`); SHC apps go
through the **Deployer** (`splunk apply shcluster-bundle`). Routing cluster peers at a DS for their
cluster apps causes conflicts. See the distributed-architecture skill.

## Next steps

1. Confirm the sample client shows the new app version under `etc/apps/`.
2. Document the class→app mapping in the repo alongside `serverclass.conf`.
3. If scaling, capture the phone-home interval and DS count decision in runbook.
