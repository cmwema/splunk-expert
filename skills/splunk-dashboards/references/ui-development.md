# UI Development — Splunk UI Toolkit & Dashboard Studio

Two distinct ways to build Splunk UI. Always confirm or assume which one the user means, and tailor
output accordingly — they share almost no code.

| | UI Toolkit | Dashboard Studio |
|---|---|---|
| Tech | React, `@splunk/react-ui`, SplunkJS Stack | JSON definition, no code |
| Best for | Custom interactive apps, bespoke visualizations | Fast operational dashboards, tokens, standard viz |
| Skillset | Frontend dev (Node, React, hooks) | Splunk admin / analyst |

## Contents
- Choosing between them
- UI Toolkit: project, components, data, best practices
- Dashboard Studio: JSON structure, tokens, data sources

## Choosing between them

- Need pixel-level control, custom React components, or to embed Splunk in a larger app → **UI Toolkit**.
- Need a maintainable operational dashboard with dropdowns/tokens and standard charts fast →
  **Dashboard Studio**.

## UI Toolkit

Scaffold with the Splunk UI Toolkit / `@splunk/create`. Build component-driven, modern React with
hooks and functional components.

```jsx
import React, { useState } from 'react';
import Button from '@splunk/react-ui/Button';
import Select from '@splunk/react-ui/Select';
import WaitSpinner from '@splunk/react-ui/WaitSpinner';
import { SplunkThemeProvider } from '@splunk/themes';

// Data via SplunkJS Stack search jobs, or @splunk/search-job
import SearchJob from '@splunk/search-job';

export default function TrafficPanel() {
    const [index, setIndex] = useState('network');
    const [rows, setRows] = useState(null);
    const [error, setError] = useState(null);

    const run = () => {
        setRows(null); setError(null);
        const search = SearchJob.create({
            search: `| tstats count from datamodel=Network_Traffic where index=${index} by All_Traffic.action`,
            earliest_time: '-1h', latest_time: 'now',
        });
        search.getResults().subscribe(
            (results) => setRows(results.results),
            (err) => setError(err),
        );
    };

    return (
        <SplunkThemeProvider family="prisma" colorScheme="dark" density="comfortable">
            <Select value={index} onChange={(e, { value }) => setIndex(value)}>
                <Select.Option label="network" value="network" />
                <Select.Option label="security" value="security" />
            </Select>
            <Button appearance="primary" onClick={run} label="Run" />
            {error && <div role="alert">Search failed: {String(error)}</div>}
            {rows === null ? <WaitSpinner size="medium" /> : <pre>{JSON.stringify(rows, null, 2)}</pre>}
        </SplunkThemeProvider>
    );
}
```

Relevant `package.json` deps:
```json
{
  "dependencies": {
    "@splunk/react-ui": "^4",
    "@splunk/themes": "^0.18",
    "@splunk/search-job": "^4",
    "react": "^17",
    "react-dom": "^17"
  }
}
```

⚠️ **Best Practice** — Always handle loading and error states explicitly (the example shows both).
Lazy-load heavy visualizations so initial render stays fast. Sanitize any user input that becomes
part of an SPL string — concatenating a raw dropdown value into a search risks SPL injection; prefer
fixed option values or validate against an allowlist.

## Dashboard Studio

Dashboards are JSON. Three top-level sections: `dataSources`, `visualizations`, and `layout`,
plus `inputs` for tokens.

```json
{
  "title": "Network Traffic",
  "inputs": {
    "input_idx": {
      "type": "dropdown",
      "token": "idx",
      "title": "Index",
      "options": { "items": [
        { "label": "network", "value": "network" },
        { "label": "security", "value": "security" }
      ], "defaultValue": "network" }
    }
  },
  "dataSources": {
    "ds_traffic": {
      "type": "ds.search",
      "options": {
        "query": "| tstats count from datamodel=Network_Traffic where index=$idx$ by All_Traffic.action",
        "queryParameters": { "earliest": "-1h", "latest": "now" }
      }
    }
  },
  "visualizations": {
    "viz_traffic": {
      "type": "splunk.column",
      "dataSources": { "primary": "ds_traffic" },
      "title": "Traffic by action"
    }
  },
  "layout": {
    "type": "grid",
    "structure": [
      { "item": "viz_traffic", "type": "block", "position": { "x": 0, "y": 0, "w": 1200, "h": 400 } }
    ]
  }
}
```

⚠️ **Best Practice** — Drive all dynamic behavior through tokens (`$idx$` above); reference macros or
saved searches rather than pasting long SPL into each panel, so logic stays in one place. Use base +
chain searches (`ds.chain`) to reuse one expensive search across multiple panels instead of running
it per panel.
