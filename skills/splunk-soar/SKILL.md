---
name: splunk-soar
description: >-
  Splunk SOAR — all tasks. Trigger for: SOAR playbooks, custom Python actions,
  phantom.act(), phantom.debug(), phantom.error(), decision blocks, action
  results, custom functions, REST API integration, SOAR assets, custom lists,
  playbook idempotency, SOAR automation, container, artifact, vault, HUD.
  Default: Splunk SOAR (Phantom) latest, Python 3.x.
---

# Splunk SOAR

Router only. Read `references/soar.md` for all SOAR tasks.

## Routing

| Task | Action |
|---|---|
| Any SOAR task — playbooks, Python actions, assets, integrations | Read `references/soar.md` |

## Shared conventions

- Write idempotent playbooks — every action must handle re-runs without duplicate side effects.
- Always wrap custom Python in `try/except` with meaningful errors logged via `phantom.error()`.
- Use decision blocks to handle action failures explicitly — don't let failures silently pass through.
- Parameterize all endpoints, credentials, and thresholds via SOAR custom lists or asset configuration.
- `phantom.debug()` during development; gate behind a debug flag in production.
- Least-privilege: SOAR assets only get the permissions required for their specific actions.
- Never `verify=False` on outbound REST calls in production.

## Output format

Fenced `python` for playbook code. Full functions with try/except. ⚠️ Best Practice callouts. Flag idempotency and credential handling implications.
