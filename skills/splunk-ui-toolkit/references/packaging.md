# Building, packaging, and AppInspect

## Production build

From the repo root:

```bash
yarn build        # production bundle into the app package's webapp/appserver path
```

Confirm the built assets landed under the app package (e.g.
`packages/my-splunk-app/appserver/...` or the `src/main/webapp/pages` path the generator chose) and
that `app.conf` metadata is correct before packaging.

## app.conf metadata

⚠️ **Best Practice — complete `default/app.conf`.** AppInspect and Splunkbase require it:

```ini
[launcher]
author = <you / org>
version = 1.0.0
description = <one-line description>

[ui]
is_visible = true
label = My Splunk App

[package]
id = my-splunk-app
```

Keep `version` in sync with `package.json`. The `[package] id` must match the app folder name.

## Package to .spl

`@splunk/create` projects expose a pack script:

```bash
yarn pack        # or: yarn run pack — produces <app>-<version>.spl (a tar.gz of the app dir)
```

If you need to do it by hand, the artifact is just the app directory tar-gzipped with a `.spl`
extension, excluding `local/`, `*.pyc`, and dev cruft.

## AppInspect (run before any distribution)

```bash
# CLI (install once): splunk-appinspect inspect <app>.spl --mode precert
splunk-appinspect inspect my-splunk-app-1.0.0.spl --mode precert
```

⚠️ **Best Practice — AppInspect clean before Splunkbase or internal deploy.** Common SUIT failures:
- Missing/incomplete `app.conf` metadata (version, label, description).
- Bundled `node_modules` or source maps shipped in the package — ship only built assets.
- World-writable files or absolute paths in config.
- Unminified/dev bundle (forgot `NODE_ENV=production`).

## Install

In Splunk Web: **Apps → Manage Apps → Install app from file**, upload the `.spl`, restart Splunk Web
when prompted. For clustered SHs, deploy via the deployer bundle instead of a manual upload (see the
distributed-architecture / deployment-server skills).

## Next steps

1. Tag a git release matching `version`.
2. For SHC, stage the app on the deployer and `splunk apply shcluster-bundle`.
3. Smoke-test in a non-prod app context before promoting.
