# Binding SUIT components to Splunk search

There are two mainstream paths to get search results into a React component. Pick based on whether
you're inside the Dashboard Framework or building a free-standing SPA.

## Path A — SearchJob (SplunkJS-style, free-standing SPA)

Use `@splunk/search-job` to dispatch a search and read results. Good for a custom app that owns its
own data fetching.

```jsx
import SearchJob from '@splunk/search-job';

const search = SearchJob.create(
    { search: '| savedsearch eleveo_agent_summary', earliest_time: '-24h', latest_time: 'now' },
    { app: 'eleveo_analytics_v2' }
);

const sub = search.getResults({ count: 0 }).subscribe((res) => {
    // res.results is an array of row objects keyed by field name
    setRows(res.results);
});
// always tear down
return () => sub.unsubscribe();
```

⚠️ **Best Practice — reference saved searches or macros, not inline SPL.** Mirrors the Dashboard
Studio rule. Put the SPL in `savedsearches.conf` (or a macro) and call it by name. Keeps SPL in
version control with the app, lets ops tune it without a rebuild, and avoids shipping brittle query
strings in the JS bundle. Always set explicit `earliest_time`/`latest_time`.

## Path B — DataSource (Dashboard Framework inside your app)

When embedding the Splunk Dashboard Framework, define data sources in the dashboard definition and
let the framework manage the search lifecycle, then consume results via the framework's APIs /
visualization components. Prefer this when you want framework-managed refresh, tokens, and the
shared search pool.

## Handling results well

- Results come back as **strings** — coerce numerics (`Number(r.score)`) before math or charting.
- Multivalue fields arrive as arrays; single values as scalars. Normalize before rendering.
- Subscribe/unsubscribe in a `useEffect` cleanup to avoid leaks and setState-after-unmount warnings.
- Debounce token/filter changes so every keystroke doesn't dispatch a new job.

## Visualizations

`@splunk/visualizations` exposes chart components that take a `dataSources` prop plus an `options`
object (the same dynamic-options syntax used in Dashboard Studio):

```jsx
import Line from '@splunk/visualizations/Line';

<Line
    dataSources={{ primary: myDataSource }}
    options={{ 'legend.placement': 'right', 'yAxis.scale': 'linear' }}
/>
```

⚠️ **Best Practice — lazy-load charts.** Visualization packages are heavy. `React.lazy` them and
gate behind `Suspense` so the first paint isn't blocked by chart code the user may never scroll to.
