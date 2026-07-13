---
name: splunk-dashboards
description: >-
  Splunk dashboard development — Dashboard Studio and UI Toolkit. Trigger for:
  Dashboard Studio JSON, tokens, dynamic inputs, data source bindings,
  visualizations in Dashboard Studio, React components, @splunk/react-ui,
  @splunk/create, SplunkJS Stack, custom visualizations, dashboard performance,
  trellis, drilldown, time range pickers. Also trigger for validating a classic
  Simple XML view (`default/data/ui/views/*.xml`) after editing it — checking
  it's well-formed before shipping. Default: Splunk Enterprise 9.x, Dashboard
  Studio preferred unless user specifies UI Toolkit.
---

# Splunk Dashboards

Router only. Read `references/ui-development.md` for all dashboard tasks.

## Routing

| Task | Action |
|---|---|
| Dashboard Studio JSON — tokens, inputs, data sources, visualizations | Read `references/ui-development.md` |
| UI Toolkit — React, @splunk/react-ui, custom components | Read `references/ui-development.md` |

## Shared conventions

- Dashboard Studio: use tokens for all dynamic behavior; reference macros or saved searches instead of hardcoded SPL.
- UI Toolkit: functional React components with hooks; lazy-load heavy visualizations; handle loading/error states explicitly.
- Never hardcode credentials in dashboard SPL or JS.
- Clarify which method the user is targeting (Dashboard Studio vs UI Toolkit) if ambiguous — they have different output formats.
- Dashboard Studio: valid JSON only; tokens must be declared before use.
- UI Toolkit: follow AppInspect requirements if the app is distributed; deduplicate `node_modules` dependencies.
- Classic Simple XML: after editing a view file, validate well-formedness before considering the edit
  done — Splunk Web fails to render a malformed view with little useful error detail:
  ```bash
  xmllint --noout path/to/view.xml && echo "XML OK"
  # fallback if libxml2 isn't installed:
  python3 -c "import xml.etree.ElementTree as ET; ET.parse('path/to/view.xml'); print('XML OK')"
  ```
  ⚠️ **Best Practice** — this only proves the file is syntactically valid XML, not that it's a working
  Splunk dashboard (a `base=` referencing a nonexistent search `id`, an invalid `<option>` name, or SPL
  that's syntactically fine but wrong at runtime will all pass this check). Treat a pass as "safe to
  load," not "verified correct" — confirm in Splunk itself that panels render and drilldowns fire.

## Output format

Fenced `json` for Dashboard Studio. Fenced `javascript`/`jsx` for UI Toolkit. Section headers: Dashboard JSON / Tokens / Data Sources / Visualization Options / Best Practice Notes. ⚠️ callouts for performance and token-binding issues.
