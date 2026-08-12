# Integritas Manifests

Private distribution repo for Edge Studio's signed update manifests and channel install artifacts. This repo has no code of its own — it's a pull target that `edge-studio`'s CI writes to and that VPS hosts serve over HTTPS.

## Why this repo exists

`update-agent` (in `edge-studio`) fetches a signed manifest from a `MANIFEST_URL` to know which digest-pinned images to deploy. The VPS hosts serving that URL are firewalled on SSH, so CI can't push directly. Instead, `edge-studio`'s `release.yml` pushes here over HTTPS (GitHub App token, no deploy key), and each VPS runs a cron job that pulls this repo and serves it via nginx. Signature verification happens in `update-agent`, not in transport — this repo and the pull step are not part of the trust boundary, only the delivery path.

See `docs/plans/manifest-deploy-pull-model.md` and `docs/adr/0007-release-channels-and-compose-generation.md` in `edge-studio` for the full design rationale.

## Layout

App-namespaced, so a second app could reuse this repo later without a folder migration:

```
edge-studio/
  <channel>/
    manifest.json       # signed manifest: {frontend, backend, updateAgent, version, createdAt}
    manifest.json.sig   # Ed25519 signature over manifest.json
  docker/
    <channel>/
      docker-compose.yml  # generated, images pinned to this channel's manifest digests
      .env.example         # MANIFEST_URL / RELEASE_CHANNEL pre-filled for this channel
```

Channels: `development`, `canary`, `release`. Channel identity lives in the folder path, not in a git branch — everything above lives on `main`.

## How content gets here

Written only by `edge-studio`'s `release.yml` (`manifest` job), authenticated via the `integritas-pi-manifest-deploy` GitHub App (installed on this repo only, `Contents: Read and write`, short-lived per-run installation token — no standing deploy key/PAT). Real release tags push to `main`; `*-test.*` dry-run tags push to `dev` (must exist ahead of time, created once from `main`).

Do not hand-edit files under `edge-studio/` — they're overwritten by the next matching CI run and any manual edit has no signature to back it.

## How content is consumed

Each VPS clones this repo read-only (separate credential from CI's write access) outside any app-managed directory, and a cron job runs `git pull --ff-only` every 5–15 minutes, logged so pull failures are visible locally. nginx aliases a location block to the `edge-studio/` subfolder only (never the repo root, so `.git/` stays unreachable over HTTP). `update-agent`'s `MANIFEST_URL` points at that nginx endpoint, e.g. `https://<host>/edge-studio/<channel>/manifest.json`.

## Security notes

- This repo is private and holds no secrets — the manifest signing key lives in `edge-studio`'s GH Actions secrets, not here.
- Manifests are meaningless without a valid signature; `update-agent` embeds the public key and verifies before trusting any digest.
- Read-only VPS pull credentials are scoped to this repo only, separate from CI's write-scoped GitHub App install.
