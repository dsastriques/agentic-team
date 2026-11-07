# Release and delivery

## 1. Flow
Trunk-based: short-lived branches, PRs with preview environments, `main` always deployable. Feature flags decouple deploying from releasing.

## 2. Versioning
Conventional commits → release-please (services) or changesets (JS packages); semver tags and changelogs; images tagged `sha-<short>` + semver; mobile apps keep store version and build number automated (EAS `autoIncrement`).

## 3. Promotion
Build once per commit → push and sign the image → deploy the digest to dev (automatic on merge) → staging (automatic after dev smoke; e2e and regression from qa) → prod (GitHub environment approval) → smoke and release-health check. With GitOps, each promotion is a change to the digest in `deploy/<env>/`; without it, the reusable deploy workflow runs per environment. Deployments are recorded per environment.

## 4. Strategies
Rolling updates by default; blue-green when versions can't run side by side; canary with automated analysis when traffic and availability NFRs justify it. Feature flags through the provider the architect chose (OpenFeature SDKs with Unleash, Flagsmith, GrowthBook, LaunchDarkly or the like); flags have owners and removal dates.

## 5. Database migrations
- A dedicated pipeline step (or pre-deploy Job) runs migrations before the new version rolls out; it uses its own migration credentials.
- Every migration is compatible with the version still running: expand → migrate data → contract across separate releases, coordinated with the backend. Long data migrations run as background jobs, not in the deploy step.
- Destructive steps (drop, rename, type narrowing) are never automatic; they ship in a later release after the code stopped using the column, with a snapshot taken first in production.
- Rollback of data is a forward fix; down migrations run in prod only when planned.

## 6. Rollback
Applications: redeploy the previous digest (one command or one revert), rehearsed in staging. Infrastructure: revert the change and plan / apply through the pipeline; stateful resources are protected from replacement. Each service's runbook states its rollback.

## 7. Configuration per environment
Images never rebuilt per environment. Non-secret config from ConfigMaps, parameter stores or environment variables defined in IaC; secrets from the secret manager; per-environment values live next to the deployment definitions and are reviewed like code.

## 8. Preview environments
Per PR: a namespace (Kubernetes), a tagged revision (Cloud Run, Container Apps) or a static deploy (web), with its own seeded database (an ephemeral Postgres from the seed scripts) or a shared preview database with per-PR schemas when that's cheaper. The URL is posted on the PR for the designer and qa; torn down on close or after `preview_ttl`; a cost cap on the total. Previews never use production data.

## 9. Web hosting
- Static SPA / SSG: object storage + CDN (S3 + CloudFront, Cloud Storage + Cloud CDN, Blob Storage + Front Door, or Cloudflare Pages / R2) — hashed assets with long immutable caching, `index.html` and config with short or no caching, SPA fallback routing, security headers at the edge (`security-and-supply-chain.md` §6), cache invalidation of the entry files on deploy.
- SSR: container runtime with the same promotion flow; CDN in front for static assets.
- Runtime config injected per environment (no rebuilds); source maps uploaded to error tracking and not publicly served.
- Micro frontends (when adopted): each remote deployed and versioned independently, a manifest per environment maps remotes to versions, rollback per remote.

## 10. Mobile (Expo EAS)
- `eas.json` profiles: `development` (dev client), `preview` (internal distribution for testers and the designer), `production`.
- EAS Build from CI (an `EXPO_TOKEN` in the environment secrets) on tags or release branches; signing credentials managed by EAS or stored in the secret manager, never in the repo.
- EAS Update for JS-only changes: one channel per environment, `runtimeVersion` policy so updates only reach compatible binaries, staged rollout and quick rollback.
- EAS Submit to the App Store and Google Play only with the owner's approval; account for store review time in the release plan; staged rollouts on Play and phased release on the App Store.

## 11. Release notes and comms
Release PRs carry the changelog, migrations, flags to flip and rollback notes; a deployment message to the team channel or the owner after production deploys.
