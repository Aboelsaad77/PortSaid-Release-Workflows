# PortSaid release workflows

Central, reusable GitHub Actions workflow for publishing application releases to
the public distribution repository (`Aboelsaad77/PortSaid-Releases`).

Product repositories keep a thin caller pinned to an **immutable commit SHA** of
this repository. Never reference this workflow by tag or branch: whoever can move
that ref could change what runs with your repository's secrets.

## What the workflow does

1. **build + digest** — picks the artifact by glob, computes sha256, uploads it.
2. **sign** (optional) — Ed25519 detached signature over a canonical manifest
   (`app`, `version`, `sha256`, `asset`, `channel`); the key is supplied as
   `SIGNING_KEY` and is never echoed or uploaded. Prints the **public** key, which
   is what the registry stores.
3. **publish** — re-hashes the downloaded asset, refuses to continue if it does
   not match the built digest, publishes a GitHub Release in the distribution
   repo, and verifies the published asset is publicly downloadable.

## Caller contract

```yaml
jobs:
  release:
    uses: Aboelsaad77/PortSaid-Release-Workflows/.github/workflows/release.yml@625b0cecb72fc9d52103a60cc9a470b8d6e52fae
    with:
      app_slug: media-downloader
      version: ${{ inputs.version }}
      artifact_glob: "dist/*.exe"
      dist_repo: Aboelsaad77/PortSaid-Releases
      prerelease: false
      sign: false
    secrets:
      DIST_REPO_TOKEN: ${{ secrets.DIST_REPO_TOKEN }}   # contents:write on the dist repo ONLY
      SIGNING_KEY: ${{ secrets.SIGNING_KEY }}           # only when sign: true
```

Templates: `callers/product-release.yml`, `callers/product-release-signed.yml`.

## Permissions and secrets

| Item | Scope |
|---|---|
| workflow `permissions` | `contents: read`; `contents: write` only on the publish job |
| `DIST_REPO_TOKEN` | fine-grained PAT, **contents: write on `PortSaid-Releases` only** |
| `SIGNING_KEY` | Ed25519 private key (PEM) as an environment secret behind a required reviewer |
| third-party actions | pinned to full commit SHAs (checkout, upload-artifact, download-artifact) |

Never expose `SIGNING_KEY` to `pull_request` runs: this workflow is reachable only
via `workflow_call`, and callers use `workflow_dispatch` on a protected branch.

## Status

Workflow installed and SHA-pinned; the caller templates reference commit `625b0cecb72fc9d52103a60cc9a470b8d6e52fae`.
The workflow has **not** been executed end to end yet — that needs a product
artifact and a `DIST_REPO_TOKEN`. Distribution was exercised independently by
publishing a test release and verifying public discovery through the store.

## Same-run artifact handoff (preferred for Windows products)

A Windows product cannot build its installer on this runner. Instead the product
workflow uploads the installer and calls this one **in the same run**:

```yaml
jobs:
  build:
    runs-on: windows-latest
    steps:
      # ... build + sign the installer ...
      - uses: actions/upload-artifact@b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882 # v4.4.3
        with:
          name: registry-installer
          path: release/installer/SomeApp-Setup.exe

  registry-publish:
    needs: build
    uses: Aboelsaad77/PortSaid-Release-Workflows/.github/workflows/release.yml@625b0cecb72fc9d52103a60cc9a470b8d6e52fae
    with:
      app_slug: media-downloader
      version: ${{ inputs.version }}
      artifact_glob: "release/installer/*.exe"   # unused when artifact_name is set
      artifact_name: registry-installer
      dist_repo: Aboelsaad77/PortSaid-Releases
      prerelease: true
      sign: false
    secrets:
      DIST_REPO_TOKEN: ${{ secrets.DIST_REPO_TOKEN }}
```

Why same-run: artifacts are scoped to a workflow run, so the publish job can
download one produced moments earlier. `DIST_REPO_TOKEN` therefore keeps
`contents: write` on the distribution repository only — it never needs read
access to the private product repository.
