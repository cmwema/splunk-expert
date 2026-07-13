# splunk-expert

A Claude Code plugin bundling 27 expert-level Splunk Enterprise and SC4S skills. Each skill is a focused, independently-triggering `SKILL.md` router with scoped reference files — no monolithic skill, no token waste on irrelevant context.

**[📖 Wiki](https://github.com/cmwema/splunk-expert/wiki)** · **[🌐 GitHub Pages](https://cmwema.github.io/splunk-expert/)**

## Skills included

| Skill | Covers |
|---|---|
| `splunk-spl` | SPL — search, stats, tstats, eval, lookups, macros, alerts, scheduled searches |
| `splunk-conf-props` | props.conf — line breaking, timestamps, field extractions, KV_MODE |
| `splunk-conf-transforms` | transforms.conf — regex transforms, routing, lookups, masking |
| `splunk-conf-server` | server.conf — clustering, SHC, KV store, licensing, SSL |
| `splunk-conf-indexes` | indexes.conf — bucket lifecycle, retention, volumes, sizing |
| `splunk-conf-inputs` | inputs.conf — monitor/scripted/modular inputs, HEC, WinEventLog |
| `splunk-ta` | TA/app development, CIM eventtypes/tags, custom REST handlers, AppInspect |
| `splunk-app-packaging` | Building a clean installable .tar.gz — exclusions, version bump, avoiding nested/self-included packages |
| `splunk-cim-compliance` | CIM normalization workflow — FIELDALIAS, eventtypes, tags, data model validation |
| `splunk-sc4s` | SC4S parser development, syslog-ng config, filtering, routing, tuning |
| `splunk-log-triage` | Raw log/syslog identification — sourcetype, format, ingestion path triage |
| `splunk-ingestion-pipeline` | End-to-end ingestion architecture — UF/HF/HEC/SC4S path, index design, retention |
| `splunk-itsi` | ITSI — service trees, KPIs, glass tables, episodes, correlation searches |
| `splunk-soar` | SOAR playbooks, custom Python actions, asset config |
| `splunk-distributed` | Indexer clustering, SHC, cluster manager, deployer, MC |
| `splunk-deployment-server` | Deployment server — serverclass.conf, deploymentclient.conf, forwarder management |
| `splunk-dashboards` | Dashboard Studio — JSON, tokens, data sources, trellis, drilldown; also Simple XML view validation |
| `splunk-ui-toolkit` | Splunk UI Toolkit (SUIT) — @splunk/react-ui, @splunk/create, SplunkJS Stack |
| `splunk-linux` | Linux host tuning/hardening for Splunk and SC4S hosts |
| `splunk-alert-actions` | Custom/modular alert actions, webhook notifications, alert throttling, AppInspect |
| `splunk-rest-api` | Splunk REST API, Python SDK (splunklib), search job lifecycle, custom endpoints |
| `splunk-windows` | Windows UF config, WinEventLog, Sysmon, perfmon, service account, firewall |
| `splunk-hec` | HEC token management, batching, indexer ack, troubleshooting, Fluent Bit/vector |
| `splunk-authentication` | LDAP/AD, SAML SSO, roles, capabilities, index access, token auth, audit |
| `splunk-kvstore` | KV Store collections, SPL lookups, REST API, SHC replication, backup/restore |
| `splunk-machine-learning` | MLTK — anomaly detection, forecasting, classification, MLflow, model persistence |
| `splunk-workload-mgmt` | Workload pools, admission rules, search quotas, scheduler tuning, capacity |

All skills default to **Splunk Enterprise 9.x** and assume admin access unless a task says otherwise.

## Install

```
/plugin marketplace add cmwema/splunk-expert
/plugin install splunk-expert@splunk-expert-marketplace
/reload-plugins
```

## Updating

When a new skill or fix lands on `main`, pull it into an existing install:

```bash
claude plugin marketplace update splunk-expert-marketplace
claude plugin update splunk-expert@splunk-expert-marketplace
```

Then restart Claude Code to apply it — both commands print "Restart to apply changes," and a new
terminal tab alone isn't enough if you're running the VS Code extension; use "Developer: Reload
Window" or fully restart the app.

**If the update reports success but the skill still doesn't show up**, check these in order:

```bash
# 1. Marketplace actually has the new commit
cat ~/.claude/plugins/known_marketplaces.json | grep -A3 splunk-expert-marketplace

# 2. Which version/scope is actually installed
cat ~/.claude/plugins/installed_plugins.json

# 3. Skill folder physically present in cache
ls ~/.claude/plugins/cache/splunk-expert-marketplace/splunk-expert/<version>/skills/

# 4. Plugin is enabled in the scope you're using
grep -A3 enabledPlugins ~/.claude/settings.json ~/.claude/settings.local.json .claude/settings.json 2>/dev/null
```

The most common cause: `claude plugin update` defaults to `--scope user`. If the plugin was
installed at `project` or `local` scope instead, pass that scope explicitly, e.g.
`claude plugin update splunk-expert@splunk-expert-marketplace --scope project`.

## Usage

Skills auto-trigger based on context, or invoke explicitly with the `plugin:skill` namespace, e.g.:

```
/splunk-expert:splunk-spl
/splunk-expert:splunk-sc4s
/splunk-expert:splunk-itsi
```

## Structure

```
splunk-expert/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md
│       └── references/
│           └── *.md
└── README.md
```

Each `SKILL.md` is a thin router; the substantive guidance lives in `references/` and is loaded on demand to keep context usage low.

## Docs

| Resource | URL |
|---|---|
| GitHub Pages | https://cmwema.github.io/splunk-expert/ |
| Wiki — Home | https://github.com/cmwema/splunk-expert/wiki |
| Wiki — Skills Reference | https://github.com/cmwema/splunk-expert/wiki/Skills-Reference |

Both are auto-updated on every push to `main` via `.github/workflows/update-docs.yml`.

## Versioning

Current version: **1.3.1** — 27 skills. `splunk-dashboards` now also covers validating classic Simple XML views (`xmllint`) before shipping.

## License

MIT — see `LICENSE`.
