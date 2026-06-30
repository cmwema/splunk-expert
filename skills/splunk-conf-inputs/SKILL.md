---
name: splunk-conf-inputs
description: >-
  inputs.conf — all tasks. Trigger for: monitor inputs, file monitoring,
  scripted inputs, modular inputs, WinEventLog, udp/tcp inputs, HEC inputs,
  network inputs, batch inputs, fschange, checkpointInterval, allowlist,
  denylist, host_segment, crcSalt, initCrcLength, ignoreOlderThan, index=,
  sourcetype=, host= on inputs. Default: Splunk Enterprise 9.x, RHEL/Rocky 8/9.
---

# inputs.conf

Router only. Read `references/inputs-conf.md` for all inputs.conf tasks.

## Routing

| Task | Action |
|---|---|
| Any inputs.conf stanza — write, debug, tune | Read `references/inputs-conf.md` |

## Shared conventions

- Every stanza sets `index`, `sourcetype`, and a `host` strategy explicitly — never rely on defaults.
- Use `allowlist`/`denylist` (not `whitelist`/`blacklist`) on Splunk 8.1+.
- `crcSalt = <SOURCE>` on inputs where files share near-identical headers (e.g., CDR files) to prevent dedup collisions.
- For modular inputs: set `checkpointInterval` to prevent re-ingest on restart.
- Use Universal Forwarders for endpoint collection; reserve Heavy Forwarders for parsing/routing/masking.
- Index-time settings (props/transforms) on a UF have no effect — they apply on the parsing tier.
- `disabled = false` must be explicit in production stanzas.

## Output format

Fenced `ini` blocks. Full stanzas with inline comments explaining each directive. Section headers when multiple stanza types are involved. ⚠️ Best Practice callouts. Flag data-loss implications (crcSalt, checkpointInterval).

## Spec reference

Full inputs.conf spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/inputs.conf
