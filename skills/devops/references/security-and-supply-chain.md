# Security and supply chain (DevSecOps)

## 1. Identity and access
- Humans: SSO with MFA, groups mapped to least-privilege roles per environment; production write access for few people, time-bound where the cloud supports it; break-glass account documented and alerted.
- CI: OIDC federation, per-environment roles, plan roles read-only (`terraform-and-clouds.md` §6).
- Workloads: workload identity per service; no node-wide or shared credentials.
- Quarterly access review (`operate access-review`).

## 2. Secrets
- Source of truth: the cloud secret manager or Vault / OpenBao; delivered to workloads by External Secrets Operator or the platform's native integration; SOPS (age or KMS keys) only for GitOps files that must live in Git.
- Rotation schedule per secret type; database credentials dynamic or rotated; secrets never in Git, images, Terraform outputs, CI logs, crash reports or analytics.
- GitHub secret scanning with push protection, gitleaks in pre-commit and CI.

## 3. Scanning gates
| What | Tool (one of) | When | Gate |
|---|---|---|---|
| Dependencies | Dependabot alerts, osv-scanner, Trivy fs | PR + daily | fixable critical / high |
| Code (SAST) | CodeQL, Semgrep | PR | high-confidence high |
| Secrets | GitHub push protection, gitleaks | commit, PR | any |
| Images | Trivy, Grype | build + daily on deployed digests | fixable critical / high |
| IaC | Trivy config, Checkov, tflint | PR | high |
| Kubernetes manifests | kube-linter, Kyverno CLI | PR | policy violations |
| Workflows | zizmor, actionlint | PR | high |
| Running app (DAST) | OWASP ZAP baseline (with qa) | staging, nightly | high |
Waivers live in a tracked file with owner, reason and expiry date; expired waivers fail the gate.

## 4. Supply chain
Lockfiles everywhere and frozen installs; actions pinned by SHA; base images by digest; Renovate keeps them moving. Each image gets an SBOM (SPDX or CycloneDX), build provenance (GitHub artifact attestations, SLSA build level 2+) and a keyless cosign signature; admission policy verifies signatures and provenance before anything runs in staging or prod.

## 5. Network
Private by default: databases and caches never public, admin interfaces behind SSO or VPN, internal traffic on private networks. Edge: TLS 1.2+ everywhere, HSTS, WAF and rate limits at the CDN / gateway, DDoS protection from the CDN. Egress: allowlist for workloads that call external services — especially AI agents and their tools. mTLS between services only when an NFR asks for it.

## 6. Web security headers (with the frontend)
CSP (nonces or hashes; report-only first, then enforced), `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `frame-ancestors` via CSP, `Referrer-Policy`, `Permissions-Policy`, CORS only for known origins — set at the edge or gateway, tested in CI.

## 7. Data protection
Encryption at rest with KMS-managed keys, in transit everywhere; backups encrypted and copied to a separate account / region per the DR design; PII kept out of logs and traces (collector processors redact known fields); log and backup retention set per GDPR and the NFRs; EU regions when residency is required; keep a list of infrastructure processors (clouds, CDN, observability and error-tracking SaaS) for the owner's GDPR records.

## 8. Runtime
Restricted pod security, read-only root filesystems, non-root users, no privileged containers; node and base-image patch cadence (weekly rebuilds, monthly node image rotation); optional runtime detection (Falco) when the NFRs ask for it.

## 9. Audit
Cloud audit logs on and retained; GitHub audit log; every deployment recorded (who, what digest, which environment, when) through GitHub environments or the GitOps history.
