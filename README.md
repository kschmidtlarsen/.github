# kschmidtlarsen/.github

Organization-level GitHub configuration for the `kschmidtlarsen` account. It hosts the **shared, reusable CI/CD pipeline** that every project repo in the Yggdrasil platform calls, so testing, quality scanning, and Docker image publishing are defined once and reused everywhere.

## Overview

GitHub treats a repository named `.github` as a special, org-wide repo: files placed here (community health files, default workflow templates, reusable workflows) apply across the account. This repo currently provides one thing — a **reusable workflow** (`Shared CI Pipeline`) invoked by downstream repos with [`workflow_call`](https://docs.github.com/en/actions/using-workflows/reusing-workflows).

Each Yggdrasil app repo (kanban, mimir, calify, …) keeps a tiny caller workflow that delegates to this pipeline, passing only the inputs that differ. This keeps the CI logic — Postgres service setup, lint, unit tests, `npm audit`, SonarCloud, Playwright E2E, Forseti result upload, and GHCR image build — in a single source of truth.

## Tech stack

- **GitHub Actions** — reusable workflow (`on: workflow_call`)
- **Node.js** (default `20`) with `npm ci` and npm cache
- **PostgreSQL** service container (default image `17`) for integration/E2E tests
- **Jest** (via `npm test`) for unit tests + lcov coverage
- **Playwright** (Chromium) for E2E tests
- **SonarCloud** for static analysis / quality gate
- **npm audit** for dependency vulnerability scanning
- **Docker Buildx** → **GitHub Container Registry (GHCR)** for image publishing
- **Forseti** (`https://forseti.exe.pm`) — centralized test-result aggregation

## Architecture

### Reusable workflow: `Shared CI Pipeline`

Defined in `.github/workflows/ci.yml`. It runs two jobs:

**1. `test` — Test & Quality** (`ubuntu-latest`, 15-min timeout)

Spins up a PostgreSQL service container and runs, in order:

1. Checkout (`fetch-depth: 0`) and Node setup with npm cache
2. `npm ci` in the configured `working-directory`
3. Optional DB init — installs the Postgres client and runs `db-init-command` (skipped when empty)
4. **Lint** — `npm run lint` (soft-fail unless `lint-hard-fail: true`)
5. **Unit tests** — configurable `test-command` (default runs Jest with lcov coverage), 5-min timeout, non-blocking
6. Optional coverage-path fix — rewrites `lcov.info` `SF:` paths so SonarCloud resolves them relative to `working-directory`
7. **npm audit** — writes JSON report at `--audit-level=moderate` (non-blocking)
8. **SonarCloud Scan** — needs `SONAR_TOKEN` (non-blocking)
9. **Playwright E2E** — caches browser binaries, installs Chromium, runs `playwright test` with optional `--grep` filter (non-blocking)
10. **Upload results to Forseti** — always runs; POSTs a per-test-type summary (eslint, jest-unit, npm-audit, sonarcloud, playwright-e2e) to `FORSETI_URL/api/results/upload/<project-id>` and writes a GitHub step-summary table
11. On E2E failure, uploads the Playwright HTML report as an artifact (5-day retention)

Most quality steps use `continue-on-error` so a single failing check does not block image publishing — results are instead reported centrally to Forseti.

**2. `docker` — Build & Push Docker** (`needs: test`)

Runs only on `push` to `main`. Logs into GHCR, extracts image metadata (tags `latest` on the default branch plus the commit SHA), and builds/pushes the image from the configured `dockerfile` (default `Dockerfile.yggdrasil`), injecting `BUILD_COMMIT` and `BUILD_TIMESTAMP` build args. Requires `packages: write` permission.

### Data flow

```
project repo (caller workflow)
        │  workflow_call + inputs
        ▼
.github / Shared CI Pipeline
        │
   ┌────┴─────────────────────────┐
   ▼                              ▼
test job                    docker job (main + push only)
  lint / unit / audit           build image
  sonarcloud / e2e         ──►  push to ghcr.io/<repo>:latest,<sha>
  upload → Forseti
```

## Getting started

There is nothing to install or run locally — this repo is consumed entirely by GitHub Actions. To use the pipeline from another repo in the org, add a caller workflow, e.g. `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: kschmidtlarsen/.github/.github/workflows/ci.yml@main
    with:
      project-id: my-app          # Forseti project ID (required)
      working-directory: backend  # where package.json lives
      dockerfile: Dockerfile.yggdrasil
    secrets: inherit
```

`secrets: inherit` forwards the tokens the pipeline reads (`SONAR_TOKEN`, `CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`); `GITHUB_TOKEN` is provided automatically.

## Configuration

### Workflow inputs

| Input | Purpose | Required | Default |
|-------|---------|----------|---------|
| `project-id` | Forseti project ID (e.g. `kanban`, `mimir`, `calify`) | yes | — |
| `node-version` | Node.js version | no | `20` |
| `working-directory` | Directory containing `package.json` | no | `backend` |
| `needs-postgres` | Whether tests need a PostgreSQL service | no | `true` |
| `postgres-version` | PostgreSQL Docker image version | no | `17` |
| `db-name` | Test database name | no | `test_db` |
| `db-user` | Test database user | no | `test_user` |
| `db-password` | Test database password | no | `testpassword` |
| `db-init-command` | Command to initialize the DB (run in `working-directory`; empty = skip) | no | `''` |
| `test-command` | Unit test command | no | `npm test -- --coverage --coverageReporters=lcov` |
| `lint-hard-fail` | Fail the build when lint fails | no | `false` |
| `run-npm-audit` | Run the `npm audit` step | no | `true` |
| `fix-coverage-paths` | Prefix lcov `SF:` paths with `working-directory` for SonarCloud | no | `false` |
| `run-e2e` | Run Playwright E2E tests | no | `true` |
| `e2e-grep` | Playwright `--grep` filter (e.g. `@smoke`; empty = all) | no | `''` |
| `cache-playwright` | Cache Playwright browser binaries | no | `true` |
| `build-docker` | Build and push the Docker image to GHCR | no | `true` |
| `dockerfile` | Dockerfile path | no | `Dockerfile.yggdrasil` |

### Secrets (read by the workflow)

| Secret | Purpose |
|--------|---------|
| `GITHUB_TOKEN` | GHCR login / SonarCloud PR decoration (provided automatically) |
| `SONAR_TOKEN` | SonarCloud authentication |
| `CF_ACCESS_CLIENT_ID` | Cloudflare Access service token — Forseti upload |
| `CF_ACCESS_CLIENT_SECRET` | Cloudflare Access service secret — Forseti upload |

The default DB credentials above are throwaway values for the ephemeral CI Postgres container, not real secrets.

## Project structure

```
.
└── .github/
    └── workflows/
        └── ci.yml   # Reusable Shared CI Pipeline (workflow_call)
```

## Deployment

This repo is not itself deployed. It defines the pipeline that deploys everything else:

1. A project repo pushes to `main`.
2. Its caller workflow invokes this **Shared CI Pipeline**.
3. The `test` job runs quality checks and reports results to Forseti.
4. The `docker` job builds and pushes `ghcr.io/kschmidtlarsen/<repo>:latest` (and `:<sha>`) to GHCR.
5. **Watchtower** on the Unraid host detects the new `latest` image and redeploys the container.

## Related services / links

- **Forseti** — https://forseti.exe.pm (test-result aggregation & platform health)
- **GHCR** — `ghcr.io/kschmidtlarsen/*` (published images)
- **Graphite Iris** — shared design system for `*.exe.pm` apps, https://design.exe.pm
- **GitHub reusable workflows** — https://docs.github.com/en/actions/using-workflows/reusing-workflows
