---
name: splunk-conf-transforms
description: >-
  transforms.conf — all tasks. Trigger for: field transforms, regex transforms,
  REGEX + FORMAT, DEST_KEY, index routing via transforms, _MetaData:Index,
  data masking, anonymization, SEDCMD, CSV lookups, KV Store lookups, scripted
  lookups, external lookups, automatic lookups, metric extraction, null queue
  routing, _blacklist, CLONE_SOURCETYPE. Default: Splunk Enterprise 9.x.
---

# transforms.conf

Router only. Read `references/transforms-conf.md` for all transforms.conf tasks.

## Routing

| Task | Action |
|---|---|
| Any transforms.conf stanza — lookups, routing, masking, extractions | Read `references/transforms-conf.md` |

## Shared conventions

- Transforms referenced by `TRANSFORMS-` in props.conf are index-time and run on the parsing tier (HF/indexer), not UF.
- Transforms referenced by `REPORT-` in props.conf are search-time.
- Index routing (`DEST_KEY = _MetaData:Index`) and masking (`DEST_KEY = _raw`) are index-time only.
- Masking is irreversible once data is indexed — test in non-prod first.
- CSV lookup files go in `lookups/` inside the app; reference by filename only in transforms.conf.
- Never hardcode credentials in `external_cmd` scripted lookups — use `passwords.conf`.

## Output format

Fenced `ini` blocks. Pair transforms.conf stanzas with their props.conf reference. ⚠️ Best Practice callouts. Flag masking, routing, and data-loss implications.

## Spec reference

Full transforms.conf spec: https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configuration-file-reference/9.4.0-configuration-file-reference/transforms.conf
