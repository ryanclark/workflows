# workflows

Reusable GitHub Actions workflows for Rust CLI tools (CI, multi-platform build +
Homebrew tap publishing, and release-plz). Callers stay thin; the logic lives here.

Pin callers to a tag (e.g. `@v1`) rather than `@main` so a change here can't break
every consumer at once.

## `rust-ci.yml`

Runs `fmt --check`, `clippy`, optional `clippy --features codesigned`, and `test`
across a runner matrix.

```yaml
name: CI
on:
  pull_request:
  merge_group:
  push:
    branches: [main]
jobs:
  ci:
    uses: ryanclark/workflows/.github/workflows/rust-ci.yml@v1
    with:
      runners: '["macos-latest", "ubuntu-latest"]'  # default
      codesigned-clippy: true                        # statusline only
```

## `rust-release.yml`

Builds each `target` (codesigning + the `codesigned` feature applied only to
entries with `codesign: true`), uploads tarballs to the GitHub release, then a
single `tap` job generates `Formula/<bin-name>.rb` (platform/arch blocks derived
from the target triples) and writes it to the tap via a GitHub App token.

```yaml
name: Publish
on:
  push:
    tags: ["v*"]
jobs:
  release:
    # required: the caller must grant write so the reusable workflow's
    # GITHUB_TOKEN can upload release assets (it can only downgrade, not raise).
    permissions:
      contents: write
    uses: ryanclark/workflows/.github/workflows/rust-release.yml@v1
    with:
      bin-name: statusline
      formula-class: Statusline
      formula-desc: "Claude Code statusline with usage tracking"
      build-features: codesigned
      targets: |
        [
          {"target": "aarch64-apple-darwin",        "os": "macos-latest",    "codesign": true},
          {"target": "x86_64-unknown-linux-gnu",    "os": "ubuntu-latest",   "codesign": false},
          {"target": "aarch64-unknown-linux-gnu",   "os": "ubuntu-24.04-arm","codesign": false}
        ]
    secrets:
      APPLE_CERTIFICATE_P12: ${{ secrets.APPLE_CERTIFICATE_P12 }}
      APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
      APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
      TAP_APP_KEY: ${{ secrets.RELEASE_PLZ_APP_KEY }}
```

mac-only tool (e.g. clankerbox), arm only:

```yaml
    with:
      bin-name: clankerbox
      formula-class: Clankerbox
      formula-desc: "Run Claude Code in an isolated container on macOS"
      targets: |
        [{"target": "aarch64-apple-darwin", "os": "macos-latest", "codesign": true}]
    secrets:
      APPLE_CERTIFICATE_P12: ${{ secrets.APPLE_CERTIFICATE_P12 }}
      APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
      APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
      TAP_APP_KEY: ${{ secrets.RELEASE_PLZ_APP_KEY }}
```

## `rust-release-plz.yml`

```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  release-plz:
    uses: ryanclark/workflows/.github/workflows/rust-release-plz.yml@v1
    with:
      app-id: "3903999"
    secrets:
      APP_KEY: ${{ secrets.RELEASE_PLZ_APP_KEY }}
```

## Requirements per consuming repo

- The publishing GitHub App installed on the repo **and** on the tap repo.
- Secret `RELEASE_PLZ_APP_KEY` (the App's PEM private key).
- For codesigned targets: `APPLE_CERTIFICATE_P12`, `APPLE_CERTIFICATE_PASSWORD`,
  `APPLE_SIGNING_IDENTITY`.
- Default workflow token permission set to read/write (Settings → Actions).
