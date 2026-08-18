# Starkeness Desktop — builds

Public download and update feed for the Starkeness desktop app. The source lives in the
private `_starkenessDESKTOPWRAPPER` repo; only built artifacts land here.

`electron-updater` in the shipped app reads the **latest GitHub Release** of this repo, so
the Releases *are* the update feed.

## Nothing is committed, and old releases are pruned

The installer is never added to git. `publish.js` uploads it straight to a GitHub Release,
and release assets are stored outside the repository. That means:

- the repo stays a few KB forever, no matter how many versions ship
- no 100 MB file limit (release assets may be up to 2 GB) and no need for Git LFS

The `.exe`, `.blockmap` and `latest.yml` sitting in this folder are just the staging area
for the next upload. They're covered by `.gitignore`.

After a successful upload, older releases and their tags are deleted. **`KEEP_RELEASES` in
`publish.js` defaults to 2** — the current version plus one previous.

### Why 2 and not 1

Because of how differential updates work. `electron-updater` does *not* re-download the old
installer from GitHub: it patches the copy of `installer.exe` already cached on the user's
machine, pulling only the changed byte ranges from the new URL. The old **release** is read
for exactly one thing — its ~80 KB `.blockmap`.

And that blockmap gets cached locally: after any download completes, `AppUpdater` copies
`pending/current.blockmap` into the updater cache root. So from a client's *second* update
onward, the old release is never consulted at all.

That leaves one case that genuinely needs the previous release on GitHub: **a client
updating for the first time**, which has no cached blockmap yet. Keeping one previous
release covers it. Keeping two previous releases buys nothing further.

Set it to 1 if you'd rather show only the current version — updates still work, first-time
updaters just download the full ~78 MB instead of a delta, and you lose any rollback target.

```powershell
node publish.js --keep 1     # only the version just published
node publish.js --keep all   # keep everything
```

> Whatever you choose, a pruned version can't be re-published without rebuilding it. If
> that matters, keep a git tag per released version in the wrapper repo so any version can
> be rebuilt from source.

## Publishing a release

From the wrapper repo — bump, build, and copy the artifacts here:

```powershell
npm run release   # bump patch version, build, and stage the artifacts here
```

Then from this folder:

```powershell
npm run publish
```

That creates the `v<version>` Release, uploads all three files, then prunes older releases
down to `KEEP_RELEASES`. Re-running it against an existing version replaces that release's
assets rather than erroring, so a botched upload is just a re-run. Pruning happens only
*after* the uploads succeed, so a failed run never leaves the page with nothing to download.

| Command | What it does |
| --- | --- |
| `npm run publish` | Upload, keep the newest 2 releases (default) |
| `npm run publish:dry` | Check everything and contact GitHub, change nothing |
| `npm run publish:latest-only` | Upload, then delete every other release |
| `npm run publish:keep-all` | Upload, delete nothing |

There are no dependencies to install — `publish.js` uses only Node built-ins, so `npm run`
works in a fresh clone without `npm install`.

> `package.json` is marked `"private": true`, so npm will never publish this folder to the
> npm registry. (Note that bare `npm publish` also runs the `publish` script as a lifecycle
> hook — it does the same GitHub upload, just via a confusing route. Prefer `npm run publish`.)

### Checks it runs before uploading

- **all three artifacts present** — `latest.yml`, `Starkeness-Setup.exe` and its `.blockmap`.
  The installer filename is deliberately version-less, so that
  `releases/latest/download/Starkeness-Setup.exe` is a permanent download URL the website can
  hard-code. The version is carried by `latest.yml` and the release tag, which is all
  electron-updater reads — it never parses the filename.
- **the staged version matches the wrapper's `package.json`** — catches bumping the version
  and forgetting to rebuild, which would otherwise publish the previous build
- **the installer is newer than everything that goes into it** (`src/`, `build/`,
  `package.json`, `starkeness.config.json`) — warns if you edited source without rebuilding

The first two abort; the last is a warning, since a touched-but-unchanged file is a common
false positive.

The updater only detects a *higher* version, so the version bump is what makes a release
visible to installed clients. They pick it up within ~3 hours, or on next launch.

## One-time setup: a GitHub credential

Uploading to GitHub needs to authenticate as you. Either option works; the script prefers
`GH_TOKEN` and falls back to the GitHub CLI.

**A. GitHub CLI** — no token to store or rotate:

```powershell
# install from https://cli.github.com/
gh auth login
```

**B. Personal access token** — create one at <https://github.com/settings/tokens> (classic,
`repo` scope; or fine-grained with *Contents: Read and write* on this repo):

```powershell
[Environment]::SetEnvironmentVariable('GH_TOKEN', 'ghp_...', 'User')
```

Open a new terminal afterwards. If `GH_TOKEN` is already set but expired, the script says so
explicitly — replace it, or clear it with
`[Environment]::SetEnvironmentVariable('GH_TOKEN', $null, 'User')` to fall through to `gh`.

## Code signing

Builds are unsigned, so Windows SmartScreen shows "Windows protected your PC" on first
install; users choose **More info → Run anyway**. If you sign later, keep using the same
certificate — `electron-updater` verifies a downloaded installer's publisher against the
running app's, so switching between two different certificates breaks auto-updates and
forces a manual reinstall.
