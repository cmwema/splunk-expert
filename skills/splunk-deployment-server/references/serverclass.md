# serverclass.conf — design and syntax

Location: `$SPLUNK_HOME/etc/system/local/serverclass.conf` on the DS. Apps live under
`$SPLUNK_HOME/etc/deployment-apps/`.

## Two stanza types

A class stanza defines *who*; a class:app stanza defines *what they get*.

```ini
[serverClass:<class_name>]
whitelist.0 = <pattern>
blacklist.0 = <pattern>
restartSplunkd = false
stateOnClient = enabled

[serverClass:<class_name>:app:<app_name>]
stateOnClient = enabled
restartSplunkd = false
```

`<app_name>` must match a directory name under `deployment-apps/`.

## Client targeting

Whitelist/blacklist are **ordered** (`.0`, `.1`, …). Blacklist wins over whitelist.

```ini
[serverClass:linux_prod]
whitelist.0 = *prod-linux*
whitelist.1 = 10.20.30.0/24
blacklist.0 = *prod-linux-canary*
```

Match keys beyond hostname: you can match on DNS name, IP, or client-supplied
`clientName`/`machineTypes`. Set a stable `clientName` in the client's `deploymentclient.conf` when
hostnames are unreliable, and target it with `whitelist.0 = <clientName>`.

Filter by platform with `machineTypes` (e.g. `linux-x86_64`, `windows-x64`):

```ini
[serverClass:windows_uf]
machineTypes = windows-x64, windows-x86
```

## Inheritance / global defaults

Settings at the `[global]` stanza or a class stanza cascade to its app stanzas; per-app stanzas
override. Keep `restartSplunkd`/`stateOnClient` at the class level unless a specific app differs.

```ini
[global]
restartSplunkd = false
stateOnClient = enabled
repositoryLocation = $SPLUNK_HOME/etc/deployment-apps
```

## restartSplunkd vs restartIfNeeded

- `restartSplunkd = true` — client restarts splunkd after receiving this class/app. Use for inputs
  or props changes that require a restart.
- Newer Splunk supports a lighter `restartSplunkd = false` plus reload-capable apps; many config
  types reload without a restart. Default to `false` and only escalate where required.

⚠️ **Best Practice — minimize overlap and restarts.** A flat set of role classes with a single broad
base class is easier to reason about than deep overlapping classes. Every `restartSplunkd = true`
multiplies across the fleet — scope it tightly.

## Filtering apps within a deployment app

`whitelist`/`blacklist` at the `:app:` level can ship only part of an app dir, but this is rarely
worth the complexity — prefer splitting into separate deployment apps.

## After editing

```bash
splunk reload deploy-server          # picks up serverclass.conf + new/changed apps
splunk reload deploy-server -class <class_name>   # scope the reload to one class
```
