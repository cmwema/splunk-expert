# Linux Host Tuning & Hardening for Splunk / SC4S

Reference for OS-level configuration of RHEL/Rocky 8/9 hosts running Splunk (indexer, SH, HF) or
SC4S. Scripts use `set -euo pipefail` and explicit error handling.

## Contents
- File-descriptor and process limits
- Kernel and network tuning (sysctl)
- Transparent Huge Pages
- Time synchronization
- Filesystem and storage
- Users and permissions
- Firewall and network security
- systemd service management
- Log management and auditd
- Patching, SELinux

## File-descriptor and process limits

`/etc/security/limits.d/splunk.conf`:
```bash
splunk  hard  nofile  65535
splunk  soft  nofile  65535
splunk  hard  nproc   20480
splunk  soft  nproc   20480
```
Low ulimits cause Splunk to fail under load with "too many open files."

## Kernel and network tuning (sysctl)

`/etc/sysctl.d/99-splunk.conf` — socket buffers matter most on SC4S hosts taking high-volume syslog:
```bash
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.rmem_default = 8388608
net.core.wmem_default = 8388608
net.ipv4.udp_mem = 65536 131072 262144
vm.swappiness = 10
fs.inotify.max_user_watches = 524288
```
Apply: `sysctl -p /etc/sysctl.d/99-splunk.conf`.

⚠️ **Best Practice** — Undersized UDP socket buffers on an SC4S host silently drop syslog packets at
high rates. Raise `rmem_max`/`udp_mem` and prefer TCP+TLS transport over UDP where the device allows.

## Transparent Huge Pages

Always disable THP on Splunk hosts — it causes latency spikes under Splunk's memory pattern.
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
Make persistent via a systemd unit or `/etc/rc.local`; the runtime echo doesn't survive reboot.

## Time synchronization

Timestamp accuracy is critical (search correctness, cluster replication).
```bash
# RHEL/Rocky
systemctl enable --now chronyd
chronyc tracking
```

⚠️ **Best Practice** — Time drift causes Splunk timestamp mismatches and cluster replication issues.
Run chrony/ntpd everywhere and monitor offset.

## Filesystem and storage

- Use **XFS** for index volumes — handles Splunk's small random writes better than ext4 at scale.
- Mount with `noatime,nodiratime`:
  ```
  /dev/sdX  /opt/splunk/var/lib/splunk  xfs  defaults,noatime,nodiratime  0 0
  ```
- Hot/warm on SSD; cold/frozen on HDD or object storage.
- Never exceed ~80% capacity on index volumes; set `minFreeSpace` in `server.conf` accordingly.
- Use LVM in physical deployments for online resize.
- Watch I/O with `iostat -x 2`, `iotop`, `df -h`.

## Users and permissions

- Run Splunk as a dedicated non-root `splunk` account — never root. Run the SC4S container as
  non-root where the runtime supports it.
- Scoped sudo via `/etc/sudoers.d/splunk`:
  ```bash
  splunk ALL=(ALL) NOPASSWD: /opt/splunk/bin/splunk start, /opt/splunk/bin/splunk stop, /opt/splunk/bin/splunk restart
  ```
- SSH: key-based only.
  ```bash
  PasswordAuthentication no
  PermitRootLogin no
  PubkeyAuthentication yes
  ```

## Firewall and network security

Keep the host firewall on; open only required ports.
```bash
# Splunk
firewall-cmd --permanent --add-port=9997/tcp   # S2S
firewall-cmd --permanent --add-port=8089/tcp   # mgmt/REST
firewall-cmd --permanent --add-port=8088/tcp   # HEC
firewall-cmd --permanent --add-port=8000/tcp   # Web (restrict to mgmt VLAN)
# SC4S
firewall-cmd --permanent --add-port=514/udp
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --permanent --add-port=6514/tcp   # syslog TLS
firewall-cmd --reload
```
Restrict 8000/8089 to management networks. Use TLS for S2S, HEC, and Splunk Web.

## systemd service management

Manage Splunk and SC4S as systemd services for restart and boot persistence.
```ini
[Unit]
Description=Splunk Enterprise
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
User=splunk
Group=splunk
ExecStart=/opt/splunk/bin/splunk start --no-prompt --answer-yes
ExecStop=/opt/splunk/bin/splunk stop
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
LimitNPROC=20480

[Install]
WantedBy=multi-user.target
```
`systemctl daemon-reload` after edits; check with `journalctl -u splunk -f`.

## Log management and auditd

Forward host logs to Splunk via UF: `/var/log/secure` (auth), `/var/log/messages`, `audit.log`,
`/var/log/cron`, journald. Install auditd on infra hosts:
```bash
# /etc/audit/rules.d/splunk-host.rules
-w /opt/splunk/etc/ -p wa -k splunk_config_change
-w /etc/sudoers -p wa -k sudoers_change
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
```

## Patching, SELinux

- Patch on a regular cycle; never update the Splunk binary via the OS package manager — use Splunk's
  upgrade procedure. Pin dependency versions where needed.
- Keep SELinux `enforcing`. If it blocks Splunk, generate a targeted policy instead of disabling:
  ```bash
  ausearch -c 'splunkd' --raw | audit2allow -M splunk-policy
  semodule -i splunk-policy.pp
  ```
- For SC4S under Podman on RHEL, set container context on mounts:
  ```bash
  chcon -Rt container_file_t /opt/sc4s/
  ```

⚠️ **Best Practice** — Don't disable SELinux to make Splunk work; build a scoped policy module. A
disabled-SELinux host is a finding in any serious audit.
