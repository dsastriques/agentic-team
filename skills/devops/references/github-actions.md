# GitHub Actions and repository automation

## 1. Workflow layout
```
.github/
  workflows/
    ci.yml                 # pull_request + push to main: gates for every app (affected only)
    cd.yml                 # push to main / release: build once, deploy dev → staging → prod
    preview.yml            # pull_request opened/synchronize/closed: preview env up / down
    infra.yml              # infra/** changes: fmt, validate, scan, plan per env, apply on merge
    release.yml            # release-please / changesets
    drift.yml              # schedule: plan all stacks, open an issue on drift
    _build-image.yml       # reusable (workflow_call): build, scan, SBOM, sign, push
    _deploy.yml            # reusable: deploy one app to one environment by digest
  actions/<name>/action.yml  # composite actions: toolchain setup (mise), cache, login
  CODEOWNERS
renovate.json
```
Reusable workflows keep one implementation of build and deploy; app workflows only pass inputs.

## 2. Pipeline shape
```
PR:    lint + typecheck ─┬─ unit ─┬─ build image (sha tag) ─ scan / SBOM ─ integration / contract
                         │        └─ web build + bundle budget + Lighthouse CI
                         └─ infra: fmt / validate / tflint / trivy config / infracost / plan
       → preview deploy → e2e smoke (qa) → designer review URL commented on the PR
main:  same gates → push image by digest + sign + attest → deploy dev → smoke
       → deploy staging → e2e / regression (qa) → prod (environment approval) → smoke → done
```

## 3. Speed
- Cache dependencies through the setup actions (`setup-node` with pnpm, `astral-sh/setup-uv` with `enable-cache`, `gradle/actions/setup-gradle`) and Docker layers with buildx (`cache-from/cache-to: type=gha` or a registry cache).
- Affected only: `turbo run … --filter=...[origin/main]` / `nx affected` for JS, path filters (`on.<event>.paths` or a paths-filter step) per service; a single "ci-ok" job aggregates results so required checks stay stable when jobs are skipped.
- `concurrency: { group: ${{ github.workflow }}-${{ github.ref }}, cancel-in-progress: ${{ github.event_name == 'pull_request' }} }`; deploy jobs use a per-environment group without cancel.
- `timeout-minutes` on every job; matrices for versions and shards (Playwright `--shard`); arm64 and larger runners when the bill says it's worth it.
- Testcontainers runs on GitHub-hosted Ubuntu runners as is; upload JUnit, coverage and Playwright reports as artifacts and summaries (`$GITHUB_STEP_SUMMARY`).

## 4. Security
- Top-level `permissions: contents: read` (or `{}`); grant per job only what it needs (`id-token: write` for OIDC, `packages: write` to push to GHCR, `pull-requests: write` to comment, `attestations: write`).
- Pin third-party actions to a full commit SHA with a version comment (`uses: org/action@<sha> # v4.2.0`); Renovate keeps them current. Enforce SHA pinning in the repository or organization Actions policy.
- Never interpolate untrusted input (`github.event.pull_request.title`, branch names, issue bodies) into `run:`; pass it through `env:` and quote it.
- Avoid `pull_request_target` and `workflow_run` with a checkout of PR code; forks get no secrets.
- Cloud access through OIDC (`references/terraform-and-clouds.md` §6); `GITHUB_TOKEN` over PATs; a GitHub App token for cross-repo work.
- Secrets and variables scoped to environments; production secrets only in the `prod` environment.
- Self-hosted runners: private repos only, ephemeral (one job per runner), isolated network; never for untrusted PRs.
- Lint workflows with `actionlint` and `zizmor` in CI.
- Provenance: `actions/attest-build-provenance` (and SBOM attestation) for images and release artifacts.

## 5. Environments and protection
- `dev`: deploys on merge to main. `staging`: after dev smoke passes. `prod`: required reviewers (the owner or delegates), optional wait timer, deployment branches limited to `main` / tags. `preview-*`: per PR, no reviewers.
- Environment variables hold non-secret config (URLs, account ids); secrets hold the few that can't be keyless (Hetzner API tokens, EAS token, Sentry auth token).
- Record deployments through the environment so GitHub shows what runs where.

## 6. Repository governance
- Ruleset on `main`: PR required, required status checks (the aggregate "ci-ok" plus infra plan when relevant), dismiss stale approvals, block force pushes and deletions, linear history; merge queue when main gets busy.
- CODEOWNERS: `infra/`, `.github/`, `deploy/` → devops owner; services and apps → their owners.
- Renovate (or Dependabot): grouped updates per ecosystem, schedule, automerge for patch / dev dependencies with green CI, digests for Docker images and actions, Terraform providers and Helm charts included.
- Branch and commit conventions from the architect's config (`branch_pattern`, conventional commits).

## 7. Releases
- Conventional commits → release-please (per package or service) or changesets (JS monorepo) → version bump, changelog, tag, GitHub release.
- Images tagged `sha-<short>` always, plus the semver tag on release; deployments reference the digest.

## 8. Terraform / OpenTofu in CI
- PR: `fmt -check`, `validate`, tflint, Trivy config (or Checkov), `terraform test` for changed modules, Infracost comment, `plan` per affected stack and environment with OIDC; plan summary posted to the PR, plan file stored as an artifact.
- Merge: apply the reviewed plan for `dev`; `staging` / `prod` apply jobs run in their protected environments. One concurrency group per stack and environment so two applies never race on state.
- Scheduled drift plan (`-detailed-exitcode`); exit code 2 opens or updates an issue.

## 9. Toolchain notes
- One toolchain source: `mise.toml` (or `.tool-versions`) read by both developers and CI (`jdx/mise-action`), so versions never diverge.
- Python: `uv sync --frozen`, `uv run pytest`. Java: Gradle wrapper with the setup-gradle cache, `./gradlew check`. JS: `pnpm install --frozen-lockfile`, `pnpm turbo run lint typecheck test`.
- Playwright: cache browsers or use the official container image; Maestro / Detox for mobile run on macOS or Android emulator runners only when the plan asks for it (they are slow and costly).
