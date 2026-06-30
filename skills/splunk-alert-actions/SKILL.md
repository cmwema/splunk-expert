---
name: splunk-alert-actions
description: >-
  Splunk alert actions and notifications — all tasks. Trigger for: custom alert
  actions, modular alert actions, alert_actions.conf, savedsearches.conf alert
  settings, webhook alerts, email alerts, PagerDuty/ServiceNow integrations,
  adaptive response actions, alert throttling, suppression, trigger conditions,
  custom Python alert action scripts, alert_actions.conf, README.md for packaged
  alert actions, testing alert actions locally, AppInspect for alert actions.
  Default: Splunk Enterprise 9.x, Python 3.x.
---

# Splunk Alert Actions

Router only. Read `references/alert-actions.md` for all alert action tasks.

## Routing

| Task | Action |
|---|---|
| Built-in alert actions — email, webhook, run script | Read `references/alert-actions.md` |
| Custom modular alert action (Python) | Read `references/alert-actions.md` |
| Alert throttling, suppression, trigger conditions | Read `references/alert-actions.md` |
| Adaptive response / notable event actions | Defer to `splunk-itsi` or `splunk-soar` for orchestration |

## Shared conventions

- Custom alert actions live in `<app>/alert_actions.conf` + `<app>/bin/<action>.py`.
- Action script: subclass `ModularAlertBase`; override `process_event()`; return 0 on success, non-zero on failure.
- Read all parameters from `settings` dict passed to `process_event()` — never hardcode.
- Log to `$SPLUNK_HOME/var/log/splunk/<action>.log` via `logging`; never `print()`.
- `alert_actions.conf` `is_custom = 1` required for modular actions; set `payload_format = json`.
- Throttle at the saved search level (`alert.suppress`, `alert.suppress.period`) not in the action code.
- Always include a `README` and UI metadata (`<app>/default/data/ui/alerts/<action>.html`) for packaged distribution.
- Test with `splunk cmd python bin/<action>.py --execute` before deploying.

## Output format

Fenced `ini` for conf files, `python` for action scripts, `html` for UI metadata. Section headers: alert_actions.conf / Action Script / UI Metadata / savedsearches.conf Settings / AppInspect Notes. ⚠️ callouts for throttling, credential, and retry implications.
