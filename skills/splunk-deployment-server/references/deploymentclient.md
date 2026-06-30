# deploymentclient.conf — the forwarder side

Location on each client: `$SPLUNK_HOME/etc/system/local/deploymentclient.conf`. Ironically, the best
way to manage it at scale is to ship it *as a deployment app* once the client is pointed at the DS —
but the initial pointer must be set locally (or baked into the UF install / image).

## Minimal config

```ini
[deployment-client]
clientName = linux-prod-web-01     # stable id for targeting; optional but recommended

[target-broker:deploymentServer]
targetUri = ds.example.com:8089
```

`targetUri` is the DS management port (default 8089).

## Phone-home interval

```ini
[deployment-client]
phoneHomeIntervalInSecs = 60       # default 60; raise to reduce DS load at large fleet sizes
```

⚠️ **Best Practice — tune phone-home for fleet size.** At thousands of clients, a 60s interval
hammers the DS. Stagger/raise the interval (e.g. 300–600s) as the fleet grows; forwarders don't need
sub-minute config freshness.

## Setting the pointer via CLI (bootstrap)

```bash
splunk set deploy-poll ds.example.com:8089
splunk restart
```

## What clients pull

Each phone-home, the client sends its facts (hostname, IP, clientName, machineType) and the DS
returns the set of apps for every server class it matches. The client downloads changed apps into
`$SPLUNK_HOME/etc/apps/` and applies the class/app `stateOnClient` + `restartSplunkd` directives.

## Excluding apps from DS management

Apps installed locally that you don't want the DS to touch should simply not be mapped to any class.
The DS only manages apps under classes the client matches; it won't delete unmanaged local apps
unless an app of the same name is removed from a class the client matches (then it's removed on the
client).

⚠️ **Best Practice — don't dual-manage.** A given app should be owned by exactly one mechanism (DS,
or manual, or deployer for SHC). Mixing causes apps to flap between states on each phone-home.

## Verifying the client is connected

```bash
splunk list deploy-clients          # run on the DS
# or hit the DS REST endpoint:
# /services/deployment/server/clients
```
On the client, check `$SPLUNK_HOME/var/log/splunk/splunkd.log` for `DeployedApplication` /
`DeploymentClient` lines confirming successful pulls.
