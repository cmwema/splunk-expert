---
name: splunk-ui-toolkit
description: >-
  Build React-based Splunk apps with the Splunk UI Toolkit (SUIT) — @splunk/create scaffolding,
  @splunk/react-ui components, @splunk/visualizations, SplunkJS Stack / search bindings, and the
  Webpack/build setup. Use this whenever the user is building a custom React front-end for a Splunk
  app rather than a Dashboard Studio (JSON) or Simple XML view — including phrasings like "Splunk
  React app", "@splunk/create", "@splunk/react-ui", "SUIT", "SplunkJS", "custom Splunk visualization",
  "package my React Splunk app as a .spl", "yarn link:app", or Webpack/styled-components dedup errors
  in a Splunk app. Covers scaffolding a monorepo, wiring components to Splunk search via SearchJob/
  DataSource, the link/build/package workflow, and AppInspect packaging. Do NOT use for Dashboard
  Studio JSON dashboards (that is a separate skill) or for non-UI add-on/TA work.
---

# Splunk UI Toolkit (React app development)

The Splunk UI Toolkit (SUIT) is the modern, React-based way to build custom Splunk app front-ends.
It supersedes the older SplunkJS-on-SimpleXML and XML-extension approach. Use it when you need a real
single-page app — custom layout, third-party libraries, complex interactions — that goes beyond what
Dashboard Studio or Simple XML can express.

⚠️ **Best Practice — pick the right tool.** If the deliverable is a standard dashboard (charts +
inputs), Dashboard Studio is faster and lower-maintenance. Reach for SUIT only when the app needs
genuine custom React: bespoke components, routing, external npm libraries, or a non-dashboard UX.

## Current stack (verify versions per environment)

- `@splunk/create` 10.x — scaffolding CLI (`npx @splunk/create`).
- `@splunk/react-ui` 5.x — the component library (Button, Table, Select, Modal, etc.).
- `@splunk/visualizations` — React chart components (Line, Bar, SingleValue, …).
- `@splunk/webpack-configs` — shared Webpack config; enables styled-components SC_ATTR dedup by default.
- Node + yarn classic (v1) toolchain; React 18 / functional components + hooks.

## Routing table

| If the user is… | Read |
|---|---|
| scaffolding a new app / choosing project layout | `references/scaffolding.md` |
| wiring components to Splunk search results | `references/data-binding.md` |
| hitting Webpack / react-ui / styled-components dedup errors | `references/build-and-dedup.md` |
| linking, building, packaging to `.spl`, AppInspect | `references/packaging.md` |

## Shared conventions

- **Component-driven architecture.** One component per concern; lift search/data logic into a parent
  or a custom hook, keep presentational components dumb. Always handle three states explicitly:
  loading, error, and empty — never render a table against `undefined` data.
- **Lazy-load heavy visualizations.** Use `React.lazy` + `Suspense` for chart-heavy panels so the
  initial bundle stays small; SUIT bundles can balloon quickly.
- **Single instance of `@splunk/react-ui`.** Declare it with a *major* range (`^5.0`), never a minor
  pin, and dedupe via Webpack alias. Multiple instances on one page break theming and Layer stacking.
- **Fonts aren't bundled.** react-ui renders in "Splunk Platform Sans" (Proxima Nova) by default but
  ships no fonts — load them via `@font-face` or accept the fallback stack.
- **Theme provider at the root.** Wrap the app in `SplunkThemeProvider` (family `enterprise`,
  colorScheme `light`/`dark`) so all components theme consistently.
- **Never hardcode credentials or tokens** in the React bundle — it ships to the browser. Talk to
  Splunk through the proxied REST layer using the session, not embedded secrets.

## Minimal component example

```jsx
import React, { useState, useEffect } from 'react';
import { SplunkThemeProvider } from '@splunk/themes';
import Table from '@splunk/react-ui/Table';
import WaitSpinner from '@splunk/react-ui/WaitSpinner';
import Message from '@splunk/react-ui/Message';

export default function AgentTable({ dataSource }) {
    const [state, setState] = useState({ status: 'loading', rows: [] });

    useEffect(() => {
        let cancelled = false;
        dataSource
            .fetch()
            .then((rows) => !cancelled && setState({ status: 'done', rows }))
            .catch((err) => !cancelled && setState({ status: 'error', error: err }));
        return () => { cancelled = true; };
    }, [dataSource]);

    if (state.status === 'loading') return <WaitSpinner size="medium" />;
    if (state.status === 'error') return <Message type="error">{String(state.error)}</Message>;
    if (!state.rows.length) return <Message type="info">No results.</Message>;

    return (
        <Table stripeRows>
            <Table.Head>
                <Table.HeadCell>Agent</Table.HeadCell>
                <Table.HeadCell>Score</Table.HeadCell>
            </Table.Head>
            <Table.Body>
                {state.rows.map((r) => (
                    <Table.Row key={r.agent}>
                        <Table.Cell>{r.agent}</Table.Cell>
                        <Table.Cell>{r.score}</Table.Cell>
                    </Table.Row>
                ))}
            </Table.Body>
        </Table>
    );
}
```

Wrap the rendered tree once at the entry point:

```jsx
<SplunkThemeProvider family="enterprise" colorScheme="light" density="comfortable">
    <AgentTable dataSource={ds} />
</SplunkThemeProvider>
```

## Next steps after any task

1. `yarn start` and verify the local demo (`http://localhost:8080`) renders without console errors.
2. `yarn link:app` + `splunk restart`, confirm the app appears in Splunk Web.
3. Run AppInspect before distributing (`references/packaging.md`).
