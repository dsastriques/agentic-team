# Terraform / OpenTofu and the clouds

## 1. Tool
Terraform (Business Source License since 1.6) or OpenTofu (open source, Linux Foundation) — one per project, decided in an ADR. Pin `required_version` and every provider; commit `.terraform.lock.hcl` with hashes for all CI and developer platforms.

## 2. Layout
```
infra/
  bootstrap/              # state backend, CI OIDC identities, budgets; applied once by / with the owner
  modules/<name>/         # reusable: network, cluster, postgres, redis, bucket, registry, dns, ci-identity…
                          #   no provider blocks, typed variables with validation, outputs, examples, tests
  live/<env>/<stack>/     # root modules: network, runtime, data, edge, observability
```
Small stacks limit blast radius and plan time; stacks read each other through outputs (`terraform_remote_state` or data sources). Terragrunt only when the env × stack repetition hurts.

## 3. State
- Remote, encrypted, versioned, locked, access restricted — state holds secrets.
- AWS: S3 backend with `use_lockfile = true` (S3-native locking; DynamoDB locking is deprecated). GCP: GCS backend (built-in locking). Azure: `azurerm` backend (blob lease locking).
- Hetzner-only projects: the S3 backend against Hetzner Object Storage works if it supports the conditional writes S3-native locking relies on — verify; otherwise keep state in a bucket of another cloud or a managed state service.
- OpenTofu can encrypt state client-side; use it when state lives outside a KMS-protected bucket.
- One state per stack and environment.

## 4. Practices
- Naming `<project>-<env>-<component>`; default tags / labels on everything: `project`, `env`, `owner`, `cost-center`, `managed-by=terraform`.
- `lifecycle { prevent_destroy = true }` plus the provider's deletion protection on databases, buckets with data, volumes and state.
- `for_each` with stable keys over `count`; `moved` and `import` blocks instead of manual state commands; no provisioners.
- Secrets generated or stored in the secret manager; mark sensitive values; never output them.
- Module tests with `terraform test` / `tofu test`; tflint with the cloud rule sets; Trivy config or Checkov; Infracost on every PR; scheduled drift detection.

## 5. Landing zone
- Separate accounts / projects / subscriptions at least for prod vs non-prod (per environment when the budget allows); organization policies (AWS SCPs, GCP org policies, Azure Policy) for region restrictions (residency), no public buckets, required tags.
- Humans through SSO with MFA (IAM Identity Center, Google groups, Entra ID); no shared or long-lived user keys; a documented break-glass account.
- Central audit logs (CloudTrail organization trail, GCP Cloud Audit Logs, Azure Activity Log), budgets with alerts at 50 / 80 / 100 %.

## 6. CI identities (keyless)
- **AWS**: IAM OIDC provider for `token.actions.githubusercontent.com`; role trust conditioned on `aud = sts.amazonaws.com` and `sub = repo:<owner>/<repo>:environment:<env>`; `aws-actions/configure-aws-credentials`.
- **Google Cloud**: Workload Identity Federation pool and provider with an attribute condition on the repository (and environment); `google-github-actions/auth`.
- **Azure**: user-assigned managed identity or app registration with federated credentials per environment subject; `azure/login`.
- **Hetzner**: no OIDC federation — one API token per project and environment, stored as a GitHub environment secret, read-only where possible, rotated on a schedule.
Plan roles are read-only; apply roles exist per environment and are only usable from that protected environment.

## 7. Service mapping
| Need | AWS | Google Cloud | Azure | Hetzner |
|---|---|---|---|---|
| VMs | EC2 | Compute Engine | Virtual Machines | Cloud servers (x86 and ARM), dedicated servers |
| Serverless containers | ECS on Fargate | Cloud Run | Container Apps | — (containers on VMs) |
| Kubernetes | EKS | GKE | AKS | self-managed (`kubernetes.md` §2) |
| Postgres | RDS / Aurora PostgreSQL | Cloud SQL / AlloyDB | Database for PostgreSQL Flexible Server | self-run (CloudNativePG, VMs) |
| Redis / Valkey | ElastiCache | Memorystore | Azure Managed Redis | self-run |
| Object storage | S3 | Cloud Storage | Blob Storage | Object Storage (S3-compatible) |
| Registry | ECR | Artifact Registry | ACR | — (GHCR) |
| Secrets | Secrets Manager / SSM Parameter Store | Secret Manager | Key Vault | — (Vault / OpenBao, SOPS, or another cloud's manager) |
| Queues / events | SQS, SNS, EventBridge | Pub/Sub, Cloud Tasks | Service Bus, Event Grid | — (self-run broker or Redis Streams) |
| DNS | Route 53 | Cloud DNS | Azure DNS | Hetzner DNS |
| CDN / WAF | CloudFront + WAF | Cloud CDN + Cloud Armor | Front Door + WAF | — (Cloudflare, Bunny) |
| Load balancer | ALB / NLB | Cloud Load Balancing | Application Gateway / Load Balancer | Load Balancers |
| Workload identity | IAM roles (Pod Identity / IRSA) | Workload Identity | Managed identity / Workload ID | — (API tokens) |
| Monitoring | CloudWatch, X-Ray | Cloud Monitoring, Logging, Trace | Azure Monitor, App Insights | — (self-run or Grafana Cloud) |
| Keys | KMS | Cloud KMS | Key Vault keys | — |
Catalogues change: confirm a service exists in the chosen region before designing around it.

## 8. Hetzner notes
- Excellent price / performance and EU locations (Germany, Finland) plus US and Singapore; ARM servers are cheaper still.
- You run more yourself: Kubernetes control plane, Postgres, Redis, backups, monitoring, secrets. Weigh the ops time against the savings in the ADR; a common split is Hetzner for compute and self-run data with tested backups, Cloudflare in front for DNS / CDN / WAF, and an external error-tracking / observability SaaS.
- Use the `hcloud` Terraform provider; firewalls on every server, private networks for internal traffic, no public SSH (bastion or a VPN such as WireGuard / Tailscale), snapshots and backups enabled, primary / floating IPs for stable endpoints.

## 9. Multi-cloud
One primary cloud per system and environment set. Stay portable at the workload layer (OCI images, Helm / Kustomize, OpenTelemetry, Terraform modules with the same interface per provider); don't force lowest-common-denominator data services unless an NFR requires portability. Cross-cloud traffic costs egress — keep chatty components together.
