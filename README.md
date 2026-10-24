# Shared GitHub Actions & Reusable Workflows

Centralized, generic, and reusable GitHub Actions workflows and composite actions for Node.js, NestJS, Next.js, Prisma, and Docker-based CI/CD pipelines.

---

## 📁 Repository Structure

```text
shared-workflows/
├── .github/
│   ├── actions/
│   │   ├── free-disk-space/
│   │   │   └── action.yml              # Cleans up runner disk space before heavy Docker builds
│   │   └── prisma-nestjs-hygiene/
│   │       └── action.yml              # Validates NestJS DTOs, Prisma migrations, and RLS policies
│   └── workflows/
│       ├── ci-node-app.yml             # Reusable CI: Typecheck, Lint, Test, Build & Hygiene
│       └── cd-docker-ssh-deploy.yml    # Reusable CD: Matrix Docker Build, Push & SSH Compose Deploy
└── README.md
```

---

## 🔒 Security & Version Pinning

Because this repository is public, **never reference mutable branches like `@main`** in production pipelines. Always pin workflows to an immutable release tag (e.g., `@v1.0.0`) or a full 40-character commit SHA.

```yaml
# Recommended (Tag Pinning)
uses: tanzim077/shared-workflows/.github/workflows/cd-docker-ssh-deploy.yml@v1.0.0

# Maximum Security (Commit SHA Pinning)
uses: tanzim077/shared-workflows/.github/workflows/cd-docker-ssh-deploy.yml@<40-char-commit-sha>
```

---

## 🚀 1. Reusable Workflow: Docker Build & SSH Deploy (`cd-docker-ssh-deploy.yml`)

Builds one or multiple Docker images (supports both single-service repos and multi-service monorepos via a matrix), pushes them to a container registry, and deploys them to a remote server over SSH using Docker Compose.

### Path
`tanzim077/shared-workflows/.github/workflows/cd-docker-ssh-deploy.yml@v1.0.0`

### Inputs (`with:`)

| Input | Type | Required | Default | Description |
| :--- | :--- | :---: | :--- | :--- |
| `service-names` | `string` | **Yes** | — | JSON array of service/image names (e.g., `'["ledger-backend"]'` or `'["api","auth"]'`). |
| `target-dir` | `string` | **Yes** | — | Remote directory path on the server (e.g., `'~/hrms-admin-backend'`). |
| `image-prefix` | `string` | No | `''` | Optional prefix added before each service name (e.g., `'hrms-'` produces `hrms-api`). |
| `use-build-arg-app-name` | `boolean` | No | `false` | Passes `--build-arg APP_NAME=<service>` during `docker build` (for monorepos). |
| `dockerfile-target` | `string` | No | `''` | Target stage for multi-stage Dockerfiles (e.g., `'production'`). Leave empty to skip. |
| `dockerfile-path` | `string` | No | `'Dockerfile'` | Relative path to the Dockerfile. |
| `free-disk-space` | `boolean` | No | `true` | Frees ~20GB+ on the Ubuntu runner before building images. |
| `registry-host` | `string` | No | `'registry.gitlab.com'` | Container registry hostname used for `docker login`. |
| `compose-file` | `string` | No | `'docker-compose.production.yml'` | Docker Compose file name on the remote host. |
| `deploy-branch` | `string` | No | `'production'` | Git branch pulled on the remote server prior to `docker compose up`. |

### Secrets (`secrets:`)

| Secret | Required | Description |
| :--- | :---: | :--- |
| `REGISTRY_URL` | **Yes** | Registry base path for image tagging (e.g., `registry.gitlab.com/group/project`). |
| `REGISTRY_USER` | **Yes** | Container registry username. |
| `REGISTRY_PASSWORD` | **Yes** | Container registry password or Personal Access Token. |
| `SERVER_IP` | **Yes** | Target server IPv4 address or hostname. |
| `SSH_USER` | **Yes** | Remote SSH username. |
| `SSH_PRIVATE_KEY` | **Yes** | Private SSH key authorized on the remote server. |

---

## 🧪 2. Reusable Workflow: Node.js CI (`ci-node-app.yml`)

Standardized continuous integration pipeline for Node.js applications (NestJS, Express, Next.js, React). Supports both `npm` and `yarn`, custom working directories, Prisma client generation, and optional NestJS/Prisma security hygiene checks.

### Path
`tanzim077/shared-workflows/.github/workflows/ci-node-app.yml@v1.0.0`

### Inputs (`with:`)

| Input | Type | Required | Default | Description |
| :--- | :--- | :---: | :--- | :--- |
| `working-directory` | `string` | No | `'.'` | Subdirectory of the package/app (e.g., `'api'` or `'web'`). |
| `node-version` | `string` | No | `'22'` | Node.js version to install. |
| `package-manager` | `string` | No | `'npm'` | Package manager to use (`'npm'` or `'yarn'`). |
| `run-prisma-generate` | `boolean` | No | `false` | Runs `npx prisma generate` before typechecking and testing. |
| `run-prisma-hygiene` | `boolean` | No | `false` | Runs DTO validation, Prisma migration checks, and RLS drift verification. |
| `run-typecheck` | `boolean` | No | `true` | Runs `npx tsc --noEmit`. |
| `lint-command` | `string` | No | `'npm run lint'` | Custom linter command. Pass `''` to skip. |
| `test-command` | `string` | No | `'npm test'` | Custom test runner command. Pass `''` to skip. |
| `build-command` | `string` | No | `'npm run build'` | Custom build command. Pass `''` to skip. |
| `build-env` | `string` | No | `'{}'` | JSON string of non-secret build-time environment variables (e.g., `NEXT_PUBLIC_*`). |

---

## 🧩 3. Composite Actions

You can also invoke the standalone composite actions directly inside custom jobs:

### A. Free Disk Space
Removes unused SDKs (`.NET`, `Android`, `GHC`, `Boost`) from GitHub-hosted Ubuntu runners.
```yaml
- name: Free Runner Disk Space
  uses: tanzim077/shared-workflows/.github/actions/free-disk-space@v1.0.0
```

### B. Prisma & NestJS Hygiene
Ensures all controller `@Body()` decorators are typed with a `Dto` class, verifies every Prisma migration folder contains a `migration.sql`, and checks for Row-Level Security (RLS) policy drift against `prisma/rls/rls-policies.sql`.
```yaml
- name: Run Prisma & NestJS Hygiene
  uses: tanzim077/shared-workflows/.github/actions/prisma-nestjs-hygiene@v1.0.0
  with:
    working-directory: 'api'
    check-rls: 'true'
```

---

## 📖 Usage Examples

### Example 1: Single Service Deployment (Frontend or Monolith Backend)
File: `.github/workflows/deploy.yml`

```yaml
name: Deploy Service to Production

on:
  push:
    branches: [production]

jobs:
  deploy:
    uses: tanzim077/shared-workflows/.github/workflows/cd-docker-ssh-deploy.yml@v1.0.0
    with:
      service-names: '["employee-panel"]'
      target-dir: '~/employee-panel'
      compose-file: 'docker-compose.production.yml'
    secrets:
      REGISTRY_URL: ${{ secrets.GITLAB_REGISTRY_URL }}
      REGISTRY_USER: ${{ secrets.GITLAB_REGISTRY_USER }}
      REGISTRY_PASSWORD: ${{ secrets.GITLAB_REGISTRY_PASSWORD }}
      SERVER_IP: ${{ secrets.GCP_SERVER_IP }}
      SSH_USER: ${{ secrets.GCP_SSH_USER }}
      SSH_PRIVATE_KEY: ${{ secrets.GCP_SSH_PRIVATE_KEY }}
```

### Example 2: Multi-Service Monorepo Deployment
File: `.github/workflows/deploy.yml`

```yaml
name: Deploy Monorepo Microservices

on:
  push:
    branches: [production]

jobs:
  deploy:
    uses: tanzim077/shared-workflows/.github/workflows/cd-docker-ssh-deploy.yml@v1.0.0
    with:
      service-names: '["api", "auth", "user", "subscription", "organization"]'
      image-prefix: 'hrms-'
      use-build-arg-app-name: true
      dockerfile-target: 'production'
      target-dir: '~/hrms-admin-backend'
      compose-file: 'docker-compose.production.yml'
    secrets:
      REGISTRY_URL: ${{ secrets.GITLAB_REGISTRY_URL }}
      REGISTRY_USER: ${{ secrets.GITLAB_REGISTRY_USER }}
      REGISTRY_PASSWORD: ${{ secrets.GITLAB_REGISTRY_PASSWORD }}
      SERVER_IP: ${{ secrets.GCP_SERVER_IP }}
      SSH_USER: ${{ secrets.GCP_SSH_USER }}
      SSH_PRIVATE_KEY: ${{ secrets.GCP_SSH_PRIVATE_KEY }}
```

### Example 3: Fullstack Monorepo CI (NestJS API + Next.js Web)
File: `.github/workflows/ci.yml`

```yaml
name: Fullstack CI

on:
  push:
    branches: [main, master]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  api-ci:
    uses: tanzim077/shared-workflows/.github/workflows/ci-node-app.yml@v1.0.0
    with:
      working-directory: 'api'
      node-version: '24'
      package-manager: 'npm'
      run-prisma-generate: true
      run-prisma-hygiene: true
      lint-command: 'npx eslint . --max-warnings 320'
      test-command: 'npx jest --ci'

  web-ci:
    uses: tanzim077/shared-workflows/.github/workflows/ci-node-app.yml@v1.0.0
    with:
      working-directory: 'web'
      node-version: '24'
      package-manager: 'npm'
      lint-command: 'npx eslint . --max-warnings 95'
      test-command: 'npm test'
      build-env: '{"NEXT_PUBLIC_API_URL":"http://localhost:3000/api/v1","NEXT_PUBLIC_SOCKET_URL":"http://localhost:3000","NEXT_PUBLIC_APP_DOMAIN":"localhost:3001"}'
```

---

## 🏷️ Releasing Updates

When updating workflows in this repository, create and push a new semantic version tag so caller repositories can upgrade safely:

```bash
git add .
git commit -m "feat: update workflow configuration"
git push origin main

# Tag new release
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin v1.0.1
```