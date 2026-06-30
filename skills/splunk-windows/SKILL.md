---
name: splunk-windows
description: >-
  Windows host configuration and tuning for Splunk data collection. Trigger for:
  Windows Universal Forwarder (UF) install/config, WinEventLog inputs, Sysmon
  deployment and config, Windows perfmon inputs, scripted inputs on Windows,
  inputs.conf WinEventLog stanzas, WMI inputs, powershell scripted inputs,
  Windows Security/System/Application event channels, custom event log channels,
  Windows UF deployment, Splunk Add-on for Windows (SA-WindowsInternals),
  Sysmon XML config, Windows host hardening for Splunk, Windows firewall rules
  for Splunk, forwarder service account permissions. Default: Windows Server 2019/2022.
---

# Splunk Windows

Router only. Read `references/windows-host.md` for all Windows host tasks.

## Routing

| Task | Action |
|---|---|
| WinEventLog `inputs.conf` stanzas | Read `references/windows-host.md` |
| Sysmon deployment, XML config, event IDs | Read `references/windows-host.md` |
| Perfmon / WMI inputs | Read `references/windows-host.md` |
| UF install, service account, firewall rules | Read `references/windows-host.md` |
| CIM normalization of Windows events | Defer field-mapping to `splunk-cim-compliance` |

## Shared conventions

- Run the UF service under a dedicated low-privilege service account — not SYSTEM or Administrator.
- `WinEventLog://Security` requires the service account to be in the local `Event Log Readers` group.
- Always set `index`, `sourcetype`, and `host` explicitly in every input stanza — never rely on defaults.
- Collect Security channel at a minimum; add Sysmon (`Microsoft-Windows-Sysmon/Operational`) for endpoint visibility.
- Sysmon: use a community-vetted XML config (SwiftOnSecurity or olaf); hash algorithm `SHA256` only.
- Perfmon inputs are CPU-intensive — collect at ≥60s intervals in production; tune object/counter list.
- Disable WMI inputs when event log inputs suffice — WMI adds overhead and is fragile.
- Windows Firewall: allow outbound TCP 9997 (Splunk to-indexer) from UF host only; block inbound 9997.
- Forward encrypted: `[tcpout]` with `clientCert` + `sslCertPath` in `outputs.conf`.

## Output format

Fenced `ini` for conf files, `powershell` for deployment scripts, `xml` for Sysmon configs. Section headers: inputs.conf / outputs.conf / Sysmon Config / Service Account / Firewall. ⚠️ callouts for privilege, volume, and encryption issues.
