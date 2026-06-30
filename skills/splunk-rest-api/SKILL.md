---
name: splunk-rest-api
description: >-
  Splunk REST API and SDK usage — all tasks. Trigger for: Splunk REST API calls,
  search jobs (POST /services/search/jobs), results polling, management API,
  /services/data/inputs, /services/saved/searches, /services/admin, Splunk Python
  SDK (splunklib), Splunk JavaScript SDK, curl against Splunk management port,
  custom REST endpoints (restmap.conf, EAI handlers), token authentication,
  session key auth, REST API output_mode=json/xml/csv, Splunk SDK service object,
  job lifecycle, results pagination. Default: Splunk Enterprise 9.x, Python 3.x.
---

# Splunk REST API

Router only. Read `references/rest-api.md` for all REST API and SDK tasks.

## Routing

| Task | Action |
|---|---|
| Search job lifecycle — create, poll, results | Read `references/rest-api.md` |
| Management API — inputs, saved searches, indexes, users | Read `references/rest-api.md` |
| Splunk Python SDK (`splunklib`) | Read `references/rest-api.md` |
| Custom REST endpoint (EAI / non-EAI handler) | Read `references/rest-api.md` — then defer deep handler code to `splunk-ta` |

## Shared conventions

- Auth: prefer token auth (`Authorization: Bearer <token>`) over session keys in scripts; rotate tokens via `splunk create user-auth-tokens`.
- Always pass `output_mode=json`; parse with `json.loads()` — never parse XML manually.
- Search job pattern: POST → poll `dispatchState` → GET results with `offset`/`count` pagination.
- Management port is `8089`; verify TLS cert or use `-k` only in dev — never in production scripts.
- Rate-limit: add `time.sleep(0.5)` between rapid polls; use `search_listeners` for long-running jobs.
- SDK: instantiate `splunklib.client.connect()` once per script; reuse the `service` object.
- Paginate results with `count` + `offset` — never pull all rows in one call for large result sets.
- Custom endpoints: `restmap.conf` + Python handler; always validate and sanitize query params.

## Output format

Fenced `python` for SDK code, `bash`/`curl` for raw REST examples. Full working examples with error handling. ⚠️ callouts for auth, pagination, and TLS issues.
