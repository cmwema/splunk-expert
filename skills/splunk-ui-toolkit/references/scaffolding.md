# Scaffolding a SUIT app with @splunk/create

## Generate

```bash
mkdir -p MyProject && cd MyProject
npx @splunk/create
```

The CLI prompts for project type. The three common choices:

- **A reusable React component** — no Splunk app wrapper; develop standalone, publish to a registry.
- **A React Splunk app** — an app you install into Splunk.
- **A monorepo with a React Splunk app and a React component** — the most common production choice:
  a `packages/<app>` (the installable Splunk app) plus `packages/<page-or-component>` (your React
  source). Pick this when you want local hot-reload dev *and* a packaged app.

⚠️ **Best Practice — naming.** Use CamelCase for app/component/repository names; avoid spaces and
special characters. The generated Splunk app folder name becomes the app id, so keep it stable.

## Resulting layout (monorepo)

```
MyProject/
├── package.json            # workspace root, yarn workspaces
├── packages/
│   ├── my-splunk-app/      # the installable Splunk app (default/, appserver/, src/main/...)
│   │   └── src/main/webapp/pages/   # built bundles land here
│   └── my-page/            # your React source — edit here
│       └── src/
│           ├── MyPage.jsx
│           └── demo/       # standalone demo entry (yarn start:demo)
└── ...
```

You edit React under `packages/my-page/src`. The build bundles it into the app package's
`appserver`/`webapp` path so Splunk can serve it.

## Local development loop

Two modes:

1. **Component demo (no Splunk needed)** — fastest inner loop:
   ```bash
   cd packages/my-page
   yarn run start:demo      # http://localhost:8080
   ```
   Edit files in `packages/my-page/src` and the demo hot-reloads. Good for pure UI work where you
   mock the data.

2. **Full app against a live Splunk** — needed for real search data:
   ```bash
   cd packages/my-splunk-app
   yarn link:app            # symlinks app into $SPLUNK_HOME/etc/apps/
   ls -l $SPLUNK_HOME/etc/apps/my-splunk-app   # confirm the link
   splunk restart           # starts Splunk if not running
   cd ../../                # back to repo root
   yarn start               # watches both packages, rebundles on change
   ```
   The app appears in the Splunk Web left-hand app menu (typically `https://localhost:8000`).

⚠️ **Best Practice — keep generated config.** `@splunk/create` writes a working `app.conf`,
`webpack` config, and `package.json` scripts (`link:app`, `start`, `build`, `pack`). Don't hand-roll
these; extend them. Re-running the generator over an existing project can clobber edits.
