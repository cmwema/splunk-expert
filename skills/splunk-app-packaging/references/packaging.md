# Splunk App Packaging — Building a Clean, Installable Tarball

A Splunk app is distributed as a `.tar.gz` (or `.spl`, same format) containing exactly one
top-level directory named after the app. Getting this wrong is easy and mostly invisible until
someone tries to install it — the two failure modes below account for most broken packages.

## Contents
- Step 1: Decide what ships
- Step 2: Version bump + changelog
- Step 3: Build without self-inclusion
- Step 4: Verify before shipping
- Common failure modes
- Worked example

## Step 1: Decide what ships

The top-level directory name **must** match `[package] id` in `default/app.conf` exactly — that's
what Splunk uses as the installed app's directory name.

Always exclude, regardless of project:

| Path | Why |
|---|---|
| `.git/` | Full history of every blob ever committed — the single biggest source of bloated packages. A repo with a few MB of current content can carry hundreds of MB of history. |
| `local/` | User/instance-specific runtime overrides — never ship, only `default/` ships. |
| `__pycache__/`, `*.pyc`, `*.pyo` | Build artifacts, not source. |
| `.DS_Store`, editor swap files | OS/editor noise. |
| Any existing `.tar.gz`/`.tgz`/`.spl` already in the app root | The previous package — never let it get packaged into the next one. |
| Any stray extracted-copy directory sitting in the app root | See "nested self-inclusion" below — these accumulate if a bad package was ever extracted for inspection *inside* the source tree. |

Always ship: `default/` (all `.conf` files and `data/ui/*`), `bin/` (source `.py`/scripts only),
`lookups/`, `metadata/default.meta`, `static/` (icons, if present), and `README.md` if the project
treats it as Splunkbase-facing documentation.

Project-specific dev-only material (design docs, research logs, example screenshots, historical
reference data, roadmaps) should also be excluded — but don't assume a directory is dev-only just
because it looks internal. Check the project's own docs first: many repos state this explicitly
(e.g. a `TODO.md` header saying "excluded from the packaged .tar.gz", or a `references/` /
`docs/` directory that's clearly a workbench, not app content). When in doubt, ask rather than
guess — deleting the wrong thing from a shipped package is invisible until a user hits a missing
file; deleting it from the *source repo* by mistake loses work.

⚠️ **Best Practice** — Never delete dev-only material from the *working tree* to keep it out of the
package. Exclude it from the `tar` invocation instead (see Step 3). Packaging should never mutate
the source repo.

## Step 2: Version bump + changelog

Before building the package, bump the release version so the shipped `app.conf` matches what
you're about to distribute:

```ini
# default/app.conf
[launcher]
version = 3.10.0
```

Check `git log -p -- default/app.conf` first to see the project's own established pattern —
some apps bump `[install] build` on every release too, others never touch it after initial setup.
Don't guess; match what's already there.

If the project keeps a user-facing changelog (commonly in `README.md`), add an entry in the same
style as the existing ones — look at the last 2-3 entries for tone and level of detail before
writing a new one.

⚠️ **Best Practice** — Bump the version in the *same commit* as the new package, so the tag/commit
and the shipped `app.conf` version never drift apart.

## Step 3: Build without self-inclusion

The most common way a package gets corrupted: running `tar` from *inside* the app directory while
writing the output *into that same directory*. If `.git` isn't excluded, or if the app directory
already contains a leftover extracted copy from a previous bad build, that gets absorbed into the
new archive too — producing a tarball that contains another (partial or full) copy of itself.

Build from the **parent** directory, referencing the app folder by name, with explicit excludes,
and write the output to a **scratch path first** — never directly into the app root:

```bash
cd /path/to/parent/of/app
tar -czf /tmp/scratch/<app_id>.tar.gz \
  --exclude='<app_id>/.git' \
  --exclude='<app_id>/local' \
  --exclude='<app_id>/<app_id>.tar.gz' \
  --exclude='<app_id>/<any-dev-only-dir-or-file>' \
  --exclude='__pycache__' \
  --exclude='*.pyc' \
  <app_id>/

# only after the build succeeds and is verified (Step 4):
mv /tmp/scratch/<app_id>.tar.gz /path/to/parent/of/app/<app_id>/<app_id>.tar.gz
```

⚠️ **Best Practice** — The scratch-then-move pattern is what prevents self-inclusion. Writing
straight into the source tree while `tar` is still reading that same tree is the exact race that
causes nesting.

## Step 4: Verify before shipping

Never ship a package you haven't inspected. At minimum:

```bash
# Exactly one top-level directory, matching app.conf's [package] id
tar -tzf <app_id>.tar.gz | head -1

# No VCS metadata, dev docs, build artifacts, or nested archives
tar -tzf <app_id>.tar.gz | grep -iE '\.git|__pycache__|\.pyc|\.tar\.gz|\.spl'
# (no output = clean)

# Version inside the package matches what you just bumped
tar -xzOf <app_id>.tar.gz <app_id>/default/app.conf | grep version

# Sanity-check size: a sudden 10-20x jump usually means something leaked in
ls -la <app_id>.tar.gz
```

If anything unexpected shows up, fix the exclude list and rebuild — don't hand-edit the tarball.

## Common failure modes

- **`.git` bloat**: a package that's 10-100x larger than the app's actual source content almost
  always means `.git/` (or another VCS/build directory) got included. `tar -tzvf` and look at which
  top-level entry has the most members.
- **Nested self-inclusion**: if a bad package was ever extracted *inside the source tree* to
  inspect it (rather than to a scratch/temp location), the extracted copy — complete with its own
  `.git` — becomes a stray untracked directory sitting in the app root. The next naive `tar czf ...
  .` run from inside that directory re-absorbs it, and the problem compounds on every rebuild.
  Check `git status --short` in the app root before packaging; anything large and untracked that
  looks like a duplicate of the app itself should be investigated (and confirmed with the user)
  before deleting.
- **Destructive "exclusion"**: a packaging script that `rm -rf`s dev-only directories from the
  working tree instead of excluding them from the `tar` invocation. This looks like it worked
  (the package is clean) but has silently deleted source material. If you ever find expected
  project files missing from disk but still present in the last commit, `git restore` them before
  doing anything else — the deletion was almost certainly never staged.

## Worked example

A DA-style app (`aarify_da_server_monitoring`) had a package that ballooned from 8.2MB to 162MB.
Diagnosis: `tar -tzvf` showed 704 of 761 entries under `.git/` — the tarball had been built from
inside the repo without excluding `.git`. Separately, two ~157MB directories had appeared inside
the app root (`aarify_da_server_monitoring/` and `aarify_da_server_monitoring (1)/`) — leftover
extractions of that same bad tarball, each with its own nested `.git`, from someone inspecting it
in place. Fix: restored the project's own `references/` directory (which a prior "cleanup" step
had deleted from disk instead of excluding from the tar), removed the two stray extraction copies
after confirming with the user, then rebuilt with the parent-directory + explicit-excludes +
scratch-path pattern above. Result: 8.2MB → 50KB (the original 8.2MB had also been carrying
dev-only reference material that a text/config-only app never needed to ship).
