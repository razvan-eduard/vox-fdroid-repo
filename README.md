# Vox Apps F-Droid repo

A serverless F-Droid repository for the [VoxApps](https://github.com/razvan-eduard/VoxApps) suite,
hosted on GitHub Pages. This repo holds no source code — [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)
mirrors the latest release APK for each app from VoxApps' GitHub Releases, builds the F-Droid index
with `fdroidserver`, and publishes it via Pages. Runs daily and on manual dispatch.

## Add this repo to F-Droid

In the F-Droid app: Settings → Repositories → add
`https://razvan-eduard.github.io/vox-fdroid-repo/fdroid/repo`.

> [!NOTE]
> The repository URL has been updated to follow the standard `/fdroid/repo` structure to ensure
> compatibility with all F-Droid clients and tools. The old URL (.../repo) still works for now but
> will be deprecated.

## One-time setup (before the workflow can run)

1. **Enable Pages**: repo Settings → Pages → Source: **GitHub Actions**.
2. **Add secrets** (Settings → Secrets and variables → Actions):
   - `KEYSTORE_BASE64` — the signing keystore, base64-encoded.
   - `KEYSTORE_PASSWORD` — its store/key password (same value for both — the keystore uses one
     password for the store and every key).
3. Run the workflow once manually (Actions → Deploy F-Droid repo → Run workflow) to confirm it
   deploys cleanly, rather than waiting for the next 02:00 UTC cron run.

See [`USAGE.md`](USAGE.md) for how the deploy workflow works step by step, how to force a rebuild,
how to add a new app to the mirror, and known gotchas.
