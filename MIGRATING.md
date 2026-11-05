# Moving the other apps onto app-ci

Trailmix moved first (branch `claude/app-ci`). These notes say what TrailmixFamily, TheLodge and Traverse would
need. None of them has been changed. Written 2026-11-04 from each repo's `main` (Traverse's checkout was on
`chunk-16/local-only`; its `release.yml` there was used).

## How they differ from Trailmix

- **Mac only for releases.** Each `release.yml` builds and publishes one target, Apple silicon (`macos-15`). No
  Intel Mac, Windows or Linux release yet.
- **TrailmixFamily is the template, not an app.** `scripts/sync-family` copies `.github/workflows/{test,release,promote,desktop}.yml`
  (and the build and smoke-test scripts) from TrailmixFamily into each app, pinned by `family.lock`. So the change is
  made **once, in TrailmixFamily**, and each app takes it with `scripts/sync-family`. Changing TheLodge's or
  Traverse's workflows directly would be undone by the next sync (or flagged by `sync-family --check`).
- **Drift already exists.** TheLodge's `release.yml` and `desktop.yml` are identical to TrailmixFamily's. Traverse's
  `release.yml` is ahead: it has the "Release it" and "To everyone" inputs (tag main as v<VERSION>, promote at once)
  that the family copy lacks. That should go into TrailmixFamily first, then everyone syncs, so they converge.
- **The update test needs no download.** `scripts/smoke-test.sh` tries updating against a stand-in feed made from
  the new build, so `previous-release` isn't needed (unlike Trailmix, which updates from the real current release).
- **Publishing can go to another repo.** `app.json`'s `updateRepo` names the repo releases go to; when it isn't this
  one, the `RELEASE_TOKEN` secret is used.
- **`desktop.yml`** (TrailmixFamily, TheLodge): a hand-started Windows x64 build and smoke test, nothing published.
  It already has `permissions: contents: read`; it has no concurrency or timeout, and `Swatinem/rust-cache@v2` is
  pinned by tag.

## What the family `release.yml` needs (then sync to TheLodge and Traverse)

1. **Take Traverse's "Release it" and "To everyone" inputs** into TrailmixFamily first, so all three start the same.
2. **`plan` job** calling `plan.yml@v1` with a Mac-only target list:
   ```yaml
   targets: '[{"os":"mac","arch":"arm64","runner":"macos-15","target":"aarch64-apple-darwin"}]'
   ```
   This replaces the "tag matches VERSION" and "which version this releases" steps (same checks, same messages).
   Add `windows` to the list (and a `platforms` input) only once Windows is ready to release; until then
   `desktop.yml` stays a separate hand-started build.
3. **Split publishing into its own job**, `needs: [plan, mac]`, `if: needs.plan.outputs.tag != ''`, with
   `permissions: contents: write`. Then `RELEASE_TOKEN` and write access exist only in a release job; today they sit
   in the same job that builds every run (only the step is gated). The build uploads `mac-app`; publish downloads it.
4. **Top of file**: `permissions: contents: read` (today it's `contents: write` for the whole workflow), and
   `concurrency` as in Trailmix (a release in its own group, never cancelled).
5. **`timeout-minutes`**: about 30 on the Mac build, 10 on publish, after checking a recent run's time.
6. **Caches** via `TheRealEminem/app-ci/setup@v1`:
   - `node-lockfiles: package-lock.json` (as today)
   - `runtime-cache-path: build/cache`, `runtime-cache-files: scripts/build-app.sh` — new: `build-app.sh` downloads
     the Python runtime into `build/cache` on every run today, as nothing caches it
   - `pip-cache-files: requirements.txt` — new
7. **`retention-days: 7`** on the `mac-app` artifact.
8. `actions/checkout` with `persist-credentials: false` (nothing in these jobs pushes with git; `gh` uses `GH_TOKEN`).

## What the family `desktop.yml` needs

- `concurrency` (cancel an older run on the same branch) and `timeout-minutes: 60`.
- Replace `setup-node`/`setup-python`/`rust-cache@v2`/the `build/cache` cache with `app-ci/setup@v1`
  (`python-version: "3.12"`, `rust-workspace: desktop/src-tauri`, `rust-key: windows-${{ matrix.arch }}`,
  `runtime-cache-path: build/cache`, `runtime-cache-files: scripts/build-windows.py`,
  `node-lockfiles` with both `package-lock.json` and `desktop/package-lock.json`). That also pins rust-cache to a
  SHA and stops branches saving their own Rust caches.
- `retention-days: 7` on its artifacts.
- Later, when Windows releases: fold it into `release.yml` as the `windows` target (the `platforms` input then picks
  it) and delete `desktop.yml`.

## `test.yml` (all three, and Trailmix's too)

Not part of app-ci v1, but on the same checklist: `concurrency` with `cancel-in-progress: true`, `timeout-minutes`
on each job, `paths-ignore: ["**.md", "docs/**"]`, and `permissions: contents: read`.

## Order

1. TrailmixFamily: bring in Traverse's inputs, then the changes above, on a branch; run one hand-started build.
2. TheLodge: `scripts/sync-family`, one hand-started build, then the next tag is the real check.
3. Traverse: `scripts/sync-family` (its own `release.yml` changes are now in the family copy), same checks.
