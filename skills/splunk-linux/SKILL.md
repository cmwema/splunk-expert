---
name: splunk-linux
description: >-
  Linux host tuning and hardening for Splunk and SC4S. Trigger for: ulimits,
  file descriptors, nofile, nproc, limits.conf, sysctl, socket buffers,
  rmem_max, udp_mem, vm.swappiness, Transparent Huge Pages THP, chrony NTP,
  XFS filesystem, noatime, LVM, systemd service unit for Splunk, firewalld,
  ufw, SELinux, AppArmor, auditd, ssh hardening, non-root service account,
  Splunk host OS configuration, SC4S host OS configuration. RHEL/Rocky 8/9 default.
---

# Linux Host — Splunk / SC4S

Router only. Read `references/linux-host.md` for all Linux host tasks.

## Routing

| Task | Action |
|---|---|
| Any Linux OS tuning or hardening for Splunk/SC4S hosts | Read `references/linux-host.md` |

## Shared conventions

- Run Splunk as a dedicated non-root `splunk` service account — never as root.
- Raise `nofile` and `nproc` limits in `/etc/security/limits.d/splunk.conf`.
- Always disable Transparent Huge Pages (THP) — causes latency spikes under Splunk's memory pattern.
- Use XFS with `noatime,nodiratime` for Splunk index volumes.
- Keep `firewalld` / `ufw` on — never disable the host firewall in production.
- Keep SELinux in `enforcing` mode on RHEL/Rocky; use `audit2allow` for policy modules rather than disabling.
- Manage Splunk as a systemd service; use `systemctl daemon-reload` after unit file changes.
- Time sync via `chronyd` is critical — timestamp drift breaks Splunk search and cluster replication.

## Output format

Fenced `bash` with `set -euo pipefail` and inline comments. Full stanzas for conf files. ⚠️ Best Practice callouts. Flag security and performance implications.
