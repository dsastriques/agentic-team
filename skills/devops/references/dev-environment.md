# Dev environment and tooling

## 1. Goals
- Clone → one command → the whole stack running with seed data, in minutes.
- The same tool versions and commands locally and in CI.
- Works on macOS (Apple silicon), Linux and Windows through WSL2.
- No production or shared secrets needed locally: local fakes, emulators and throwaway dev credentials.

## 2. Building blocks
- **Tool versions**: `mise.toml` pins Node, pnpm, Python, uv, Java, Terraform / OpenTofu, kubectl, Helm and linters; CI uses the same file.
- **Task runner**: `Taskfile.yml` (or `justfile` / `Makefile`) with a stable vocabulary that the architect's `commands` reference:
  `bootstrap`, `dev:up`, `dev:down`, `dev:reset`, `dev:logs`, `lint`, `typecheck`, `test:unit`, `test:integration`, `test:e2e`, `ci` (everything CI runs), `infra:plan`.
- **Compose stack** (`compose.yaml`): the production dependencies at the same major versions — Postgres, Redis / Valkey, a broker if used — plus local stand-ins: Mailpit for email, MinIO or LocalStack for object storage and AWS APIs, the cloud emulators that exist (Pub/Sub, Azurite for Azure Storage), an OTel collector with Grafana LGTM or Jaeger behind an `observability` profile. Health checks with `depends_on: condition: service_healthy`, named volumes, ports from `.env`.
- **Apps**: run on the host with hot reload by default (fast), or in Compose with a `full` profile for people who only need to run them.
- **Config**: `.env.example` committed with safe defaults and comments; `.env` git-ignored; the backend's typed settings read it.
- **Seed and reset**: the backend's seed scripts behind `dev:reset` (drop, migrate, seed); the same scripts serve preview environments and qa.

## 3. Dev Containers / Codespaces
`.devcontainer/devcontainer.json` with the mise feature (or features per tool), `postCreateCommand: task bootstrap`, Docker-outside-of-Docker so Testcontainers and Compose work, forwarded ports with labels, recommended editor extensions. Prebuilds for Codespaces when the image is slow to build. Keep the devcontainer optional — the plain host setup must work too.

## 4. Hooks
pre-commit or lefthook running fast checks only: formatters, linters on staged files, gitleaks, `terraform fmt`, actionlint, hadolint. Anything slow stays in CI. Hooks and CI call the same task targets.

## 5. Local Kubernetes (only when needed)
When the system depends on cluster features (operators, Gateway routing across many services, mesh): kind or k3d, the same Helm charts or Kustomize bases with `values-local`, and Tilt or Skaffold for build-and-reload. Otherwise Compose is faster and simpler.

## 6. Monorepo
pnpm workspaces + Turborepo (or Nx) for JS / TS, uv workspaces for Python, Gradle multi-project for Java; the task runner orchestrates across them; CI uses the same graph for affected builds.

## 7. Documentation
`docs/devops/dev-environment.md` (linked from the README): prerequisites, the one command, the task vocabulary, service URLs and ports, how to reset data, troubleshooting (port conflicts, Apple silicon images, WSL2 file system speed, Docker memory).

## 8. Keep it working
A CI job builds the devcontainer and runs `task dev:up` + health checks on a clean runner (on changes to the dev setup and weekly), so the setup never rots silently.
