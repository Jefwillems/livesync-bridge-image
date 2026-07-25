# livesync-bridge-image

Builds a container image for [vrtmrz/livesync-bridge](https://github.com/vrtmrz/livesync-bridge)
and publishes it to GHCR, for use in the homelab cluster.

Upstream ships a `Dockerfile` but no published image, so we build our own from a
**pinned** upstream commit and consume the result **by digest** — important because this
component reads and writes a real Obsidian vault, so upstream must never change behavior
without a deliberate, reviewed bump.

Image: `ghcr.io/jefwillems/livesync-bridge`

## How it works

- `livesync-bridge/` is a git submodule pinned to a specific upstream commit.
- Upstream has its own nested submodule (`lib/`, livesync-commonlib), so all checkouts
  must be **recursive**.
- `.github/workflows/build.yml` checks out recursively, builds `./livesync-bridge` using
  upstream's own `Dockerfile` (`linux/amd64` only — the cluster is single-node amd64),
  and pushes to GHCR.
- Tags produced: `latest`, `upstream-<short-sha>`, and `sha-<commit>`. The build also
  reports the **digest** in the job summary — pin the Kubernetes Deployment to it.

## Updating upstream (deliberate bump)

```bash
git -C livesync-bridge fetch origin
git -C livesync-bridge checkout <new-sha>
git submodule update --init --recursive
git add livesync-bridge && git commit -m "Bump livesync-bridge to <new-sha>"
git push
```

Pushing to `main` triggers a rebuild. Renovate can also open these bump PRs for you.

## Local build (optional)

```bash
git submodule update --init --recursive
docker buildx build --platform linux/amd64 -t ghcr.io/jefwillems/livesync-bridge:dev ./livesync-bridge
```

## Runtime configuration

The image expects two volumes (see upstream `readme.md`):

- `/app/dat` — contains `config.json` (peers: the CouchDB vault + a storage peer).
- `/app/data` — the storage peer's files (the storage `baseDir` must live under `data/`).

In the cluster these are provided by a ConfigMap/Secret (`config.json`) and the
`vault-workspace` PVC respectively. See `apps/vault-manager/` in the homelab repo.
