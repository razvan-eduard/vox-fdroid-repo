# Usage / maintainer guide

How `deploy.yml` actually builds and publishes this repo, how to force a rebuild, how to add a new
app to the mirror, and a note on the one real bug hit so far. See `README.md` for the quick-start
(adding the repo to a device, one-time secrets setup).

## What the workflow does, step by step

`.github/workflows/deploy.yml`, on every run (`workflow_dispatch` or the daily 02:00 UTC cron):

1. Installs `apksigner` (apt) and `fdroidserver` (**pipx**, not apt — see "Why pipx" below).
2. For each app prefix in the hardcoded `APPS` list (`commander calendar expenses notes vision
   hub`), resolves the **newest non-draft release whose tag starts with `<app>-`** in
   `razvan-eduard/VoxApps` and downloads its `.apk` asset into `repo/`. This per-prefix resolution
   (not GitHub's "latest release") matters because VoxApps tags releases per app, not one combined
   release for the whole monorepo.
3. Decodes the `KEYSTORE_BASE64` secret to `keystore.p12`, appends `keystore`/`keystorepass`/
   `keypass`/`repo_keyalias` to `config.yml` **at runtime** (never committed).
4. Runs `fdroid update -c`, which builds `repo/index-v1.json`, `index-v2.json`, `entry.json`, and
   signs the legacy `index-v1.jar` + `entry.jar`.
5. **Re-signs `index-v1.jar` with SHA-256** (see "Known gotcha" below) — a step this workflow adds
   on top of plain `fdroid update`.
6. Deletes the decoded keystore, uploads `repo/` (plus generated `metadata/`, `archive/` etc.) as
   the Pages artifact, deploys via `actions/deploy-pages@v4`.

Nothing here is committed back to the repo — `repo/`, `metadata/`, `archive/`, etc. are gitignored
and rebuilt fresh every run.

## Forcing a rebuild

The daily cron picks up new VoxApps releases automatically, but to publish immediately (e.g. right
after cutting a new release in VoxApps):

```
gh workflow run deploy.yml --repo razvan-eduard/vox-fdroid-repo
```

or via the GitHub UI: **Actions → "Deploy F-Droid repo" → Run workflow**.

## Adding a new app to the mirror

Edit the `APPS` variable in the "Download the latest APK per VoxApps app" step of `deploy.yml`
(space-separated prefixes matching that app's release-tag prefix in VoxApps, e.g. `commander`).
No other file needs to change — `fdroid update` picks up any `.apk` dropped into `repo/`
automatically from its manifest.

## Known gotcha: index-v1.jar signed with a disabled algorithm

`fdroid update`'s own `jarsigner` call signed `index-v1.jar` with **SHA1withRSA** on this runner —
a signature algorithm modern JDKs and F-Droid clients treat as unsigned:

```
jarsigner -verify -verbose -certs index-v1.jar
...
WARNING: The jar will be treated as unsigned, because it is signed with a weak algorithm
```

This silently produced an F-Droid repo that looked correct (valid JSON, correct package data) but
whose legacy-protocol index a client would reject outright, or whose trust step could fail
depending on client. Fixed by explicitly re-signing `index-v1.jar` with `SHA256withRSA` /
`SHA-256` right after `fdroid update` runs (see the "Re-sign the index with a modern algorithm"
step). `entry.jar` (the v2-protocol trust bundle most current clients actually use) was not
affected — `fdroid update` already signs it with SHA-256.

If apps stop showing up in a client again, `jarsigner -verify -verbose <file>.jar` against the
live `https://razvan-eduard.github.io/vox-fdroid-repo/repo/<file>.jar` is the first thing to check
before assuming the JSON data itself is wrong.

## Why pipx, not apt, for fdroidserver

Ubuntu's packaged `fdroidserver` (2.2.1 as of writing) ships an old `androguard` that can't parse
modern AAPT2-compiled `resources.arsc`:

```
androguard.core.bytecodes.axml.ResParserError: res1 must be zero!
```

`pipx install fdroidserver` installs the current PyPI release in its own venv, which is also what
F-Droid's own docs recommend for CI use.
