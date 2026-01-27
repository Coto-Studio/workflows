# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Reusable GitHub Actions workflows for Coto Studio projects. Other repositories consume these workflows via `workflow_call`, making this shared CI/CD infrastructure.

**Impact awareness**: Changes here affect multiple downstream repositories. Test carefully before pushing.

## Key Commands

```bash
# Validate workflow syntax (requires actionlint)
actionlint .github/workflows/*.yml

# Validate a specific workflow
actionlint .github/workflows/docker-build-image-v5.yml

# Test workflows locally (requires nektos/act)
act -W .github/workflows/<workflow>.yml --secret-file .secrets

# List all workflows
ls -la .github/workflows/
```

## Architecture

### Reusable Workflows (`.github/workflows/`)

| Workflow | Purpose | Current Version |
|----------|---------|-----------------|
| `docker-build-image-v*.yml` | Build and push Docker images to GHCR | v5 |
| `docker-stack-deploy-v*.yml` | Deploy Docker stacks via SSH/Tailscale | v5 |
| `check-base-image-v*.yml` | Check if base image updated (triggers rebuilds) | v3 |
| `playwright-test-staging.yml` | Run Playwright tests against staging | - |

### Version Evolution

Workflows are versioned with `-v*` suffix to allow gradual migration. Consuming repos reference specific versions.

| Version | Key Changes |
|---------|-------------|
| v1 | Initial implementation with hardcoded 1Password references |
| v2 | Added caching, multi-platform builds |
| v3 | Switched to repository variables for 1Password (`vars.WORKFLOWS_OP_REF`, `vars.CLIENTS_VAULT_ID`, `vars.ITEM_ID`) |
| v4 | Added LFS support, improved build args handling |
| v5 | Environment-based deployment (production/staging) via `inputs.environment`, uses `docker buildx bake` |

### Why Multiple Versions Exist

Downstream repos may not be ready to migrate. Keep old versions until all consumers upgrade. Check GitHub's "Used by" or search the org for references before deprecating.

### Templates (`templates/`)

Starter workflows for consuming repositories. Copy to downstream repo's `.github/workflows/` and replace placeholders:

- `{{PROJECT_NAME}}` — Human-readable project name
- `{{STAGING_DEPLOY_WORKFLOW_NAME}}` — Name of the staging deploy workflow to trigger after

### Archived (`archived/`)

Deprecated workflows kept for reference. **Do not use or update these.**

- `hugo-build-and-deploy.yml` — Replaced by direct Linode Object Storage integration
- `ghost-theme-test-and-build.yml` — Superseded by docker-based workflow
- `static-site-deploy.yml` — Legacy static deployment

## Secrets & Configuration Pattern

All workflows use 1Password for secrets via `1password/load-secrets-action@v2`. This centralizes secret management and allows rotation without updating GitHub secrets.

### Required Repository Variables (consuming repos)

| Variable | Purpose |
|----------|---------|
| `WORKFLOWS_OP_REF` | 1Password reference for GitHub PAT (e.g., `op://vault/item/field`) |
| `CLIENTS_VAULT_ID` | 1Password vault ID containing project secrets |
| `ITEM_ID` | 1Password item ID for this specific project |
| `TS_OAUTH_CLIENT_ID_OP_REF` | 1Password ref for Tailscale OAuth client ID |
| `TS_OAUTH_SECRET_OP_REF` | 1Password ref for Tailscale OAuth secret |
| `PUSHOVER_USER_KEY_OP_REF` | 1Password ref for Pushover user key |
| `PUSHOVER_API_TOKEN_GITHUB_OP_REF` | 1Password ref for Pushover API token |

### Required Repository Secrets (consuming repos)

| Secret | Purpose |
|--------|---------|
| `OP_SERVICE_ACCOUNT_TOKEN` | 1Password service account token for secret retrieval |

### 1Password Item Structure

Consuming projects need a 1Password item with these fields:

```
deploy/
  host     — Target server hostname (via Tailscale)
  user     — SSH user for deployment
  stack    — Docker stack name prefix
  service  — Service name within stack
  image    — Full GHCR image path (e.g., ghcr.io/coto-studio/myproject)
domain/
  main/url     — Production URL
  dev/url      — Staging URL (for Playwright tests)
```

## Workflow Patterns

### Docker Build (v5)

Uses `docker buildx bake` instead of `docker/build-push-action`. Consuming repos need a `docker-bake.hcl` file:

```hcl
// docker-bake.hcl example
target "default" {
  platforms = ["linux/amd64", "linux/arm64"]
  tags = ["ghcr.io/coto-studio/myproject:latest"]
}

target "dev" {
  inherits = ["default"]
  tags = ["ghcr.io/coto-studio/myproject:dev"]
}
```

- `inputs.environment = "staging"` → builds `dev` target
- `inputs.environment = "production"` (default) → builds default target

### Stack Deploy (v5)

Expects template files in consuming repo:

- `docker-stack-op.yaml.tpl` — Production stack (default)
- `docker-stack-staging-op.yaml.tpl` — Staging stack (when `inputs.environment = "staging"`)

Uses `op inject` to substitute 1Password references at deploy time:

```yaml
# docker-stack-op.yaml.tpl example
services:
  web:
    image: {{ op://${VAULT_ID}/${ITEM_ID}/deploy/image }}:latest
    environment:
      SECRET_KEY: {{ op://${VAULT_ID}/${ITEM_ID}/app/secret_key }}
```

Stack naming: `{DEPLOY_STACK}-{DEPLOY_SERVICE}[-staging]`

### Tailscale Connection

Two actions for different contexts:

```yaml
# For local testing with `act`
- uses: Coto-Studio/tailscale-action@main
  if: ${{ github.event.act }}

# For GitHub-hosted runners
- uses: tailscale/github-action@v3
  if: ${{ !github.event.act }}
```

### Base Image Update Check (v3)

Triggers rebuilds when upstream base images change:

```yaml
# In consuming repo
jobs:
  check:
    uses: Coto-Studio/workflows/.github/workflows/check-base-image-v3.yml@main
    with:
      base-image: node:20-alpine
      image-name: ghcr.io/coto-studio/myproject
      image-tag: latest
```

Returns `outputs.rebuild = true/false` for conditional build triggering.

### Playwright Testing

Runs after staging deploy. Expects:

- `tests/ci/` directory with Playwright tests
- `package.json` with Playwright dependencies
- 1Password item with `domain/{branch}/url` field

Required repository variables:

- `WORKFLOWS_OP_REF` — GitHub PAT for npm authentication (needed if using private GitHub Packages)
- `CLIENTS_VAULT_ID` — 1Password vault ID
- `ITEM_ID` — 1Password item ID for this project

The workflow loads secrets before `npm ci` to authenticate with GitHub Packages for private dependencies.

## Custom Actions

| Action | Purpose | Repo |
|--------|---------|------|
| `Coto-Studio/stack-deploy-action` | SSH-based Docker stack deployment | Internal |
| `Coto-Studio/tailscale-action` | Tailscale connection (act-compatible) | Internal |

## Local Testing with `act`

```bash
# Create .secrets file (not committed)
echo "OP_SERVICE_ACCOUNT_TOKEN=ops_xxxxx" > .secrets

# Run a workflow
act -W .github/workflows/docker-build-image-v5.yml \
    --secret-file .secrets \
    -e event.json

# event.json for workflow_call simulation
{
  "act": true
}
```

The `github.event.act` flag switches to `Coto-Studio/tailscale-action` which works in local containers.

## Adding a New Workflow Version

1. Copy the latest version: `cp docker-build-image-v5.yml docker-build-image-v6.yml`
2. Update the `name:` field if needed
3. Make your changes
4. Test with a single consuming repo first
5. Update this documentation
6. Notify team to migrate when ready

## Deprecating Old Versions

1. Search GitHub org for references: `uses: Coto-Studio/workflows/.github/workflows/<workflow>@`
2. Migrate all consumers to newer version
3. Move to `archived/` (don't delete — keeps history accessible)
4. Update this documentation

## What NOT to Do

- **Don't break backwards compatibility** without migrating all consumers first
- **Don't hardcode 1Password paths** — use `vars.*` references for portability
- **Don't remove old versions** until all downstream repos are migrated
- **Don't commit secrets** — all secrets flow through 1Password
- **Don't test changes in production** — use a test repo or `act` first
- **Don't modify archived workflows** — they exist for reference only

## Workflow Diagram

```
Consuming Repo                          This Repo (workflows)
─────────────────                       ────────────────────
.github/workflows/                      .github/workflows/
  my-deploy.yml ──workflow_call──────►  docker-build-image-v5.yml
                                              │
                                              ▼
                                        1Password (secrets)
                                              │
                                              ▼
                                        GHCR (images)
                                              │
                                              ▼
  my-deploy.yml ──workflow_call──────►  docker-stack-deploy-v5.yml
                                              │
                                              ▼
                                        Tailscale → Target Server
```

## Common Issues

### "Secret not found" errors
- Verify `CLIENTS_VAULT_ID` and `ITEM_ID` are set in consuming repo
- Check 1Password item has the expected field path
- Ensure service account has access to the vault

### Build cache misses
- GHCR cache uses `buildcache` tag — check it exists
- Multi-platform builds cache separately per architecture

### Tailscale connection fails
- Check OAuth client has `tag:cicd` permission
- Verify `--accept-dns` flag is set
- For `act`: ensure Docker network allows outbound connections
