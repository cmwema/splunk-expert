---
name: splunk-app-packaging
description: >-
  Package a Splunk app or TA into a clean, installable .tar.gz for Splunkbase
  or manual distribution. Trigger for: packaging a Splunk app, building an
  installable tarball, app release / version bump before shipping, fixing a
  bloated or invalid package (accidentally includes .git, node_modules, or a
  nested self-included copy), deciding what to exclude from a shipped package
  (dev-only docs, build artifacts, local/), verifying package contents before
  Splunkbase submission or manual install. Default: Splunk Enterprise 9.x.
---

# Splunk App Packaging

Router only. Read `references/packaging.md` for all packaging tasks.

## Routing

| Task | Action |
|---|---|
| Build an installable .tar.gz, decide what to exclude | Read `references/packaging.md` |
| Fix a bloated/invalid package (nested copy, `.git` included) | Read `references/packaging.md` |
| Version bump + changelog as part of a release | Read `references/packaging.md` |
| TA scaffolding, CIM mapping, REST handlers, AppInspect itself | Defer to `splunk-ta` |

## Shared conventions

- The tarball's top-level directory must exactly match `[package] id` in `app.conf`.
- Always exclude: `.git/`, `local/`, `__pycache__/`, `*.pyc`, `.DS_Store`, any existing package
  archive already sitting in the app root.
- Never build the archive from inside the app directory with the output written into that same
  tree — build to a scratch path first, then move it into place, to avoid nesting the archive
  inside itself.
- Bump `[launcher] version` in `app.conf` (and any changelog the project keeps) as part of the same
  release, before packaging — check the project's own git history for its established
  version/changelog convention rather than assuming one.
- Verify the built tarball's contents (`tar -tzf`) before shipping: exactly one top-level directory,
  no VCS metadata, no dev-only docs, size in a plausible range for the app's actual content.

## Output format

Fenced `bash` for build/verification commands, `ini` for `app.conf` version edits. ⚠️ callouts when
a step is easy to get destructively wrong (building in-place, deleting instead of excluding).
