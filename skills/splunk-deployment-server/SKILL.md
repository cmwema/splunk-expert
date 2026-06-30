---
name: splunk-deployment-server
description: >-
  Manage the Splunk Deployment Server (DS) for distributing apps and configs to Universal/Heavy
  Forwarders at scale. Use this whenever the user is working with serverclass.conf, deploymentclient.conf,
  forwarder app deployment, server class whitelists/blacklists, phone-home behavior, or DS scaling —
  including phrasings like "deploy an app to my forwarders", "serverclass.conf", "deploymentclient.conf",
  "forwarders not phoning home", "reload deploy-server", "restartSplunkd / stateOnClient", "server
  class for my UFs", "how many forwarders can one DS handle", or "forwarder management UI". Covers
  server class design, app-to-class mapping, client targeting, restart/state semantics, troubleshooting
  phone-home, and DS scaling limits. Defer indexer-cluster and search-head-cluster bundle management
  (cluster manager, deployer, apply shcluster-bundle) to the distributed-architecture skill — the DS
  manages forwarders, not cluster peers.
---

# Splunk Deployment Server

The Deployment Server (DS) is Splunk's native mechanism for pushing apps and configuration to
**forwarders** (and other non-clustered instances). Clients run `deploymentclient.conf`, phone home
on an interval, and pull the apps mapped to the server classes they match.

⚠️ **Scope boundary.** The DS manages forwarders and standalone instances. It does **not** manage
indexer-cluster peers (that's the Cluster Manager) or search-head-cluster members (that's the
Deployer). Don't point cluster peers at a DS for their cluster apps.

## Core model

```
deployment-apps/ (on DS)  →  serverclass.conf (maps apps→classes→clients)  →  forwarders pull on phone-home
```

- **Deployment app** — a directory under `$SPLUNK_HOME/etc/deployment-apps/` on the DS.
- **Server class** — a named group of clients (by hostname/IP/DNS/etc.) plus the apps they receive.
- **Deployment client** — a forwarder with `deploymentclient.conf` pointing at the DS.

## Routing table

| If the user is… | Read |
|---|---|
| designing server classes / writing serverclass.conf | `references/serverclass.md` |
| configuring the client side / phone-home | `references/deploymentclient.md` |
| forwarders not updating / troubleshooting | `references/troubleshooting.md` |

## Shared conventions

- **Edit, then reload — don't restart the DS.** After changing `serverclass.conf` or adding apps:
  `splunk reload deploy-server`. A full DS restart is rarely needed and disrupts phone-homes.
- **Design classes to minimize app overlap.** A client matching multiple classes gets the union of
  their apps; conflicting apps across classes cause confusion. Prefer a clean hierarchy: a broad base
  class (all UFs) + narrow role classes (windows, linux, network).
- **`restartSplunkd` only when needed.** Set it on classes/apps whose configs require a restart
  (inputs that need reload, some props). Gratuitous restarts ripple across the whole fleet.
- **`stateOnClient = enabled`** keeps the app enabled on the client after deploy; `disabled` ships it
  disabled; `noop` leaves the client's current state. Use `enabled` for apps you want active.
- **Naming.** Mirror the TA naming convention: server classes named by role
  (`all_forwarders`, `windows_uf`, `linux_uf`, `network_hf`).
- **Version control `deployment-apps` and `serverclass.conf`.** Treat the DS config as code.

## Quick example

```ini
# $SPLUNK_HOME/etc/system/local/serverclass.conf
[serverClass:all_forwarders]
whitelist.0 = *
restartSplunkd = false
stateOnClient = enabled

[serverClass:all_forwarders:app:TA-unix-base]
stateOnClient = enabled
restartSplunkd = false

[serverClass:linux_uf]
whitelist.0 = *linux*
blacklist.0 = *win*

[serverClass:linux_uf:app:TA-nix]
restartSplunkd = false
```

Then: `splunk reload deploy-server`.

## Next steps

1. Verify clients in **Settings → Forwarder Management** (or `/services/deployment/server/clients`).
2. Confirm the app's checksum/version updated on a sample client.
3. For >~1000–2000 UFs, plan DS scaling (`references/troubleshooting.md`).
