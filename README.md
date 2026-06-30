# splunk-expert

A Claude Code plugin bundling 18 expert-level Splunk Enterprise and SC4S skills. Each skill is a focused, independently-triggering `SKILL.md` router with scoped reference files — no monolithic skill, no token waste on irrelevant context.

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
| `splunk-cim-compliance` | CIM normalization workflow — FIELDALIAS, eventtypes, tags, data model validation |
| `splunk-sc4s` | SC4S parser development, syslog-ng config, filtering, routing, tuning |
| `splunk-log-triage` | Raw log/syslog identification — sourcetype, format, ingestion path triage |
| `splunk-ingestion-pipeline` | End-to-end ingestion architecture — UF/HF/HEC/SC4S path, index design, retention |
| `splunk-itsi` | ITSI — service trees, KPIs, glass tables, episodes, correlation searches |
| `splunk-soar` | SOAR playbooks, custom Python actions, asset config |
| `splunk-distributed` | Indexer clustering, SHC, cluster manager, deployer, MC |
| `splunk-deployment-server` | Deployment server — serverclass.conf, deploymentclient.conf, forwarder management |
| `splunk-dashboards` | Dashboard Studio — JSON, tokens, data sources, trellis, drilldown |
| `splunk-ui-toolkit` | Splunk UI Toolkit (SUIT) — @splunk/react-ui, @splunk/create, SplunkJS Stack |
| `splunk-linux` | Linux host tuning/hardening for Splunk and SC4S hosts |

All skills default to **Splunk Enterprise 9.x** and assume admin access unless a task says otherwise.

## Install

```
/plugin marketplace add cmwema/splunk-expert
/plugin install splunk-expert@splunk-expert-marketplace
/reload-plugins
```

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

## Versioning

Current version: **1.1.0** — 18 skills, post-merge of `splunk-ui-toolkit`, `splunk-cim-compliance`, `splunk-deployment-server`, and `splunk-ingestion-pipeline`.

## License

MIT — see `LICENSE`.
