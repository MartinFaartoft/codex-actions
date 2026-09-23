# codex-actions

Reusable GitHub Actions workflows for deploying to the [codex](../codex) VPS.

## What this repo provides

- **`.github/workflows/deploy-site.yml`** — Reusable workflow that builds a Docker image, pushes it to GHCR, and deploys it to the codex VPS. Call it from any site repo.

## Setup (once)

1. Create this repo as **private** on GitHub.
2. **Settings → Actions → General → Access** → *"Accessible from repositories owned by the user account"*. Without this, `uses:` calls from other private repos silently fail.
3. Tag a release: `git tag v1 && git push --tags`. Site repos will pin to `@v1`.

## Using this from a site repo

### 1. Add a `codex.yml` to the repo root

```yaml
service: mkw           # ^[a-z][a-z0-9-]{1,31}$ — used as compose project & dir name
domain: mkw.ftft.dk    # hostname Caddy will serve on
port: 80               # internal port the container listens on
type: static           # "static" or "app"
```

### 2. Add a `Dockerfile` to the repo root

For a static site using Caddy as a file server:

```dockerfile
FROM caddy:2-alpine
COPY dist /srv
CMD ["caddy", "file-server", "--root", "/srv", "--listen", ":80"]
```

Adjust `COPY dist /srv` to point at wherever your build output lives.

### 3. Add `.github/workflows/deploy.yml`

```yaml
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: MartinFaartoft/codex-actions/.github/workflows/deploy-site.yml@v1
    permissions:
      contents: read
      packages: write
    secrets:
      VPS_HOST: ${{ secrets.VPS_HOST }}
      VPS_SSH_PRIVATE_KEY: ${{ secrets.VPS_SSH_PRIVATE_KEY }}
```

**Note on `permissions`:** the reusable workflow pushes to GHCR and requires `packages: write`. If the caller repo's default `GITHUB_TOKEN` permissions are set to "Read repository contents and packages permissions" (Settings → Actions → General → Workflow permissions), the call will fail validation with `The workflow is requesting 'packages: write', but is only allowed 'packages: read'`. Granting the permissions at the job level as shown above works regardless of repo defaults.

### 4. Set repo secrets

Site repo → Settings → Secrets and variables → Actions:

- `VPS_HOST` — public IP or hostname of the codex VPS
- `VPS_SSH_PRIVATE_KEY` — full contents of `~/.ssh/codex-deploy` (the private key matching the `codex_deploy_public_key` authorized on the VPS by the codex Ansible role)

These live in **each site repo** — reusable workflows read secrets from the caller, not from this repo. On a personal GitHub account there are no org-level secrets to share, so per-repo duplication is the trade-off.

### 5. Push to `main`

The workflow will:

1. Read `codex.yml`
2. Build the Dockerfile, push image to GHCR as `ghcr.io/<owner>/<repo>:sha-<short>` and `:latest`
3. Render `docker-compose.yml` + `Caddyfile` fragment
4. SSH to VPS, deliver a JSON payload to `/usr/local/bin/deploy-site.sh` via stdin
5. Deploy script atomically swaps in the new site, runs `docker compose up -d --wait`, reloads Caddy, and smoke-tests

## Deploy contract

The stdin payload passed to `deploy-site.sh` is:

```json
{
  "version": 1,
  "service": "mkw",
  "image": "ghcr.io/dkmaskfa/mkw:sha-abc1234",
  "domain": "mkw.ftft.dk",
  "port": 80,
  "type": "static",
  "compose": "<base64 of docker-compose.yml>",
  "caddyfile": "<base64 of Caddyfile fragment>"
}
```

Full contract: see `hetzner/ansible/roles/platform/files/deploy-site.sh` in the codex repo.

Exit codes returned by the deploy script (bubble up as workflow failure):

| code | meaning |
| --- | --- |
| 0 | success |
| 2 | validation failure (bad payload) — no VPS state changed |
| 3 | container failed healthcheck — rolled back |
| 4 | Caddy reload failed — rolled back |
| 5 | internal error (lock held, docker down) |

## Rollback

Redeploy the previous SHA:

```
gh workflow run deploy.yml --ref <previous-sha>
```

Or manually re-run the previous successful deploy from the Actions UI.

Last successful deployment's rendered files live on the VPS at `/srv/sites/<service>/`. Logs at `/var/log/codex-deploy/<service>/last.log`.
