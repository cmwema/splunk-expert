# Webpack, react-ui, and styled-components deduplication

The single most common class of SUIT build failures comes from **multiple instances** of shared
libraries ending up in one bundle. Theming breaks, Layer/Modal stacking misbehaves, React throws
"invalid hook call", or styled-components emits duplicate class names.

## The rule

Exactly **one** instance of each of these must exist in the final bundle:

- `@splunk/react-ui`
- `react` and `react-dom` (required because react-ui uses hooks)
- `styled-components` (a dependency of react-ui)

## Why it happens

Transitive dependencies pull in their own copies. If package A needs `@splunk/react-ui@^5.0` and
package B accidentally pins `@~4.0`, Webpack can't collapse them and you get two instances.

## Fixes (in order of preference)

1. **Declare with a major range, not a minor pin.** Every package must declare
   `"@splunk/react-ui": "^5.0"` (or `^4.1`), never `"~4.0"`. This lets the resolver pick one version.

2. **Force a single copy via Webpack `resolve.alias`** so the whole tree uses one path:
   ```js
   // webpack.config.js
   const path = require('path');
   module.exports = {
     resolve: {
       alias: {
         '@splunk/react-ui': path.resolve(__dirname, 'node_modules/@splunk/react-ui'),
         react: path.resolve(__dirname, 'node_modules/react'),
         'react-dom': path.resolve(__dirname, 'node_modules/react-dom'),
         'styled-components': path.resolve(__dirname, 'node_modules/styled-components'),
       },
     },
   };
   ```
   `@splunk/webpack-configs` already wires most of this — extend it rather than replacing it.

3. **styled-components SC_ATTR.** styled-components should always be deduplicated and its `SC_ATTR`
   feature used so stylesheets from different bundles don't collide. `@splunk/webpack-configs`
   enables this by default; keep it on.

4. **Legacy multi-instance escape hatch.** If you genuinely must run 5.x and 4.x on the same page
   (rare, legacy), `LayerStackGlobalProvider` from the `Layer` component coordinates stacking across
   instances. Treat this as a last resort, not a design choice.

## Production build flag

To strip react-ui's dev warnings and guidance from the output, set `NODE_ENV=production` and inject
it with Webpack's `DefinePlugin`:

```js
new webpack.DefinePlugin({ 'process.env.NODE_ENV': JSON.stringify('production') })
```

⚠️ **Best Practice — diagnose dedup with the dependency tree.** When you hit a hook or theming error,
run `yarn why @splunk/react-ui` (and for `react`, `styled-components`) to see who's pulling a second
copy before touching the Webpack config.
