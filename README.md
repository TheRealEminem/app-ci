# app-ci

Shared GitHub Actions pieces for Mark's desktop apps (Trailmix, TrailmixFamily, Traverse, TheLodge): Tauri + Rust +
Python + Node, built for up to six targets (Mac arm64/Intel, Windows x64/ARM, Linux x86_64/ARM).

Each app keeps its own `release.yml` (its build scripts, smoke tests and release naming are its own) and calls these for
the parts that should be the same everywhere. Apps pin a tag (`@v1`); a change here reaches an app only when that app
moves its tag, on purpose.

The rules behind it are in `~/.claude/skills/app-ci/SKILL.md`. In short: in a public repo minutes are free and the cost
is waiting, so a test build builds only what a change can affect, and a release always builds and checks everything.

## What's here

| Piece | Kind | What it does |
|---|---|---|
| `.github/workflows/plan.yml` | reusable workflow | Which release a run makes (pushed tag, or "Release it" from main, with the version checks), which targets it builds (`platforms`: all, mac, windows or linux; a release always builds all), and whether the extra checks run (`full`: every release, or a test build with `extra_checks`). Outputs one matrix per OS. |
| `.github/workflows/linux-distros.yml` | reusable workflow | Tries the Linux build's .deb, .rpm and AppImage on other distros, every distro × arch its own job, in parallel. Outputs `drop`: the packages a release should leave out. |
| `setup` | composite action | Node, Python, uv, Rust (installing it where a runner has none), apt packages (with the right FUSE 2), and caches for npm, pip, uv, Rust and a bundled-runtime folder, each keyed on its lockfile or pin file. Rust's cache is saved only from `main` and tags. |
| `previous-release` | composite action | Fetches a file of the current release for an update test, cached under its tag so tests don't inflate download counts. |

## Using it

```yaml
on:
  push:
    tags: ["v*"]
  workflow_dispatch:
    inputs:
      platforms: { type: choice, options: [all, mac, windows, linux], default: all }
      extra_checks: { type: boolean, default: false }
      release: { type: boolean, default: false }

concurrency:   # a newer test build replaces an older one; a release is never cancelled
  group: ${{ github.workflow }}-${{ github.ref }}-${{ inputs.release && 'release' || 'build' }}
  cancel-in-progress: ${{ !startsWith(github.ref, 'refs/tags/') && !inputs.release }}

permissions:
  contents: read

jobs:
  plan:
    uses: TheRealEminem/app-ci/.github/workflows/plan.yml@v1
    with:
      platforms: ${{ inputs.platforms }}
      extra_checks: ${{ inputs.extra_checks || false }}
      release: ${{ inputs.release || false }}
      # targets: '[{"os":"mac","arch":"arm64","runner":"macos-15","target":"aarch64-apple-darwin"}]'  # a Mac-only app

  linux:
    needs: plan
    if: needs.plan.outputs.linux != '[]'
    strategy: { fail-fast: false, matrix: { include: "${{ fromJSON(needs.plan.outputs.linux) }}" } }
    runs-on: ${{ matrix.runner }}
    timeout-minutes: 50
    steps:
      - uses: actions/checkout@v4
      - uses: TheRealEminem/app-ci/setup@v1
        with:
          node-lockfiles: package-lock.json
          rust-workspace: desktop/src-tauri
          rust-key: linux-${{ matrix.arch }}
          runtime-cache-path: build/cache
          runtime-cache-files: scripts/build-linux.py
      # ... the app's own build and smoke test ...
      - uses: TheRealEminem/app-ci/previous-release@v1   # update test: only when it's a full run
        if: needs.plan.outputs.full == 'true'
        # ...

  distros:
    needs: [plan, linux]
    if: always() && needs.plan.outputs.full == 'true' && needs.plan.outputs.linux != '[]'
    uses: TheRealEminem/app-ci/.github/workflows/linux-distros.yml@v1
    with:
      arches: ${{ needs.plan.outputs.linux_arches }}
      artifact_prefix: MyApp-linux-
      smoke_command: python3 scripts/smoke-test-linux.py

  publish:
    needs: [plan, linux, distros]
    if: always() && needs.plan.outputs.tag != ''
    permissions:
      contents: write   # the only job that gets it
    # ...
```

Trailmix's `.github/workflows/release.yml` is the full example.

## The checklist each app's workflow still owns

`plan.yml` can't set these for the caller, so every app's workflow needs them itself:

- `concurrency` as above (never cancel a release)
- `timeout-minutes` on every job, about twice its normal time
- `permissions: contents: read` at the top, `contents: write` only on the publish job
- secrets (signing, notarisation, a `RELEASE_TOKEN`) only in jobs that run for a release
- `retention-days` of a week or less on test-build artifacts
- untrusted values (branch names, inputs) passed to `run:` through `env:`

## Changing it

1. Change it on a branch; `lint.yml` runs actionlint (and shellcheck) on every push.
2. Try it from an app by pointing that app at the branch (`@my-branch`) on one of the app's branches.
3. Merge, then tag. A fix that doesn't change any input or output: move `v1` (`git tag -f v1 && git push -f origin v1`)
   and say so in the release notes. Anything that changes what callers pass or get: `v2`, and move each app over when
   it's ready.

Third-party actions (anything not `actions/*`) are pinned to a full commit SHA with the version in a comment;
Dependabot keeps them current.
