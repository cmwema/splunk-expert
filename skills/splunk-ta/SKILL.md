---
name: splunk-ta
description: >-
  Splunk Technology Add-on (TA) and app development. Trigger for: building a TA,
  creating a TA skeleton, CIM normalization, FIELDALIAS, EVAL-based field mapping,
  eventtypes.conf, tags.conf, AppInspect, app.conf, default.meta,
  custom REST handlers, non-EAI persistent REST handlers,
  PersistentServerConnectionApplication, restmap.conf, web.conf, SA- or DA-
  add-ons. Default: Splunk Enterprise 9.x, Python 3.x.
---

# Splunk TA / App Development

Router only. Read `references/ta-development.md` for all TA and app tasks.

## Routing

| Task | Action |
|---|---|
| TA scaffold, CIM mapping, AppInspect, REST handlers | Read `references/ta-development.md` |
| Building the distributable .tar.gz itself | Defer to `splunk-app-packaging` |

## Shared conventions

- Naming: `TA-<vendor>-<product>`, `SA-` for supporting, `DA-` for domain add-ons.
- Knowledge objects (props/transforms/eventtypes/tags) in the TA; keep app UI logic separate.
- CIM normalization at search time via `FIELDALIAS`/`EVAL` — never index-time CIM fields.
- Never hardcode credentials — use `passwords.conf` via the Splunk secret store.
- Ship `default/` only in distributed packages; exclude `local/` and `__pycache__`.
- `metadata/default.meta` must be present; set sharing levels explicitly.
- Run `scripts/validate_ta.py` (local pre-flight) then Splunk AppInspect before distribution.
- Logger pattern: `setup_logger(level)` with `RotatingFileHandler` to `$SPLUNK_HOME/var/log/splunk/<name>.log`, `logger.propagate = False`.
- REST handlers: always return `{'payload', 'status'}` on every path; guard query params against SPL injection.

## Output format

Fenced `ini` for conf files, `python` for handlers, `bash` for packaging. Section headers: Directory Structure / props.conf / transforms.conf / eventtypes.conf + tags.conf / REST Handler / AppInspect Notes / Best Practice Notes. Add ⚠️ callouts.
