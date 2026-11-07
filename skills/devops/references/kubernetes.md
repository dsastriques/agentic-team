# Kubernetes

## 1. When (and when not)
Kubernetes pays off with several services, needs for operators or portable platform features, steady traffic and a team that can run it. For one or two services, spiky or low traffic, or a small team, prefer serverless containers (Cloud Run, ECS on Fargate, Azure Container Apps) or a VM with Compose. Record the choice and its drivers in an ADR.

## 2. Cluster options
| Provider | Option | Notes |
|---|---|---|
| AWS | EKS (Auto Mode, or managed node groups + Karpenter) | EKS Pod Identity for workload IAM; AWS Load Balancer Controller |
| Google Cloud | GKE Autopilot or Standard | Autopilot removes node management; Workload Identity Federation for GKE |
| Azure | AKS (Automatic or Standard) | Microsoft Entra Workload ID; Application Gateway for Containers or another Gateway API implementation |
| Hetzner | No native managed Kubernetes (verify the current catalogue before deciding): k3s via kube-hetzner (Terraform) or hetzner-k3s, Talos, Cluster API with the Hetzner provider, or a third-party managed control plane | Hetzner Cloud Controller Manager (load balancers, nodes) and CSI driver (volumes); you own control-plane HA, etcd backups, upgrades and add-ons |

Non-prod environments can share one cluster with a namespace each; production gets its own cluster.

## 3. Baseline add-ons
- Traffic: **Gateway API** with one implementation (Envoy Gateway, Cilium, Istio, NGINX Gateway Fabric, Traefik, or the cloud's controller). ingress-nginx is retired (best-effort maintenance ended March 2026): don't install it on new clusters and plan migration on existing ones.
- cert-manager (Let's Encrypt or the cloud CA), external-dns, External Secrets Operator, metrics-server, node autoscaling (Karpenter / cluster-autoscaler), KEDA for event-driven scaling.
- OTel collector, Prometheus stack (or the managed equivalent), log shipping.
- Policy: Pod Security Admission `restricted` per namespace, Kyverno or Gatekeeper for org rules (signed images, required labels, no `latest`, resource requests).
- GitOps controller (Argo CD or Flux); Velero only for what GitOps can't rebuild.
Pin every add-on chart version; manage them as code (GitOps or Terraform Helm releases).

## 4. Workload checklist
- `Deployment` with resource requests for CPU and memory, a memory limit (CPU limit only when noisy neighbours demand it).
- Probes: `startupProbe` (slow starters, JVM), `readinessProbe` → the service's `/health/ready` (dependencies it truly needs), `livenessProbe` → `/health/live` (process only, never dependencies).
- Graceful shutdown: `preStop` sleep of a few seconds so endpoints update, `terminationGracePeriodSeconds` above the app's drain time.
- `PodDisruptionBudget`, `HorizontalPodAutoscaler` (CPU, or custom / KEDA for queue depth), `topologySpreadConstraints` across zones and nodes.
- Security context: `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, drop ALL capabilities, `seccompProfile: RuntimeDefault`; no host paths or host network.
- One ServiceAccount per app bound to cloud IAM through workload identity; `automountServiceAccountToken: false` when unused; RBAC minimal.
- NetworkPolicy default-deny per namespace, then allow ingress from the gateway and egress to the data stores and named external hosts.
- Image referenced by digest; config from ConfigMaps; secrets from External Secrets (never plain manifests in Git, or SOPS-encrypted when GitOps requires it).
- Migrations: a `Job` run before the rollout (Helm `pre-upgrade` hook or Argo CD `PreSync`), backward compatible with the running version; it fails the deploy if it fails.
- `CronJob` with `concurrencyPolicy: Forbid`, `startingDeadlineSeconds`, history limits; jobs idempotent.

## 5. Packaging
One Helm library or app chart shared by services of the same shape (values per service and environment), or Kustomize base + overlays. In CI: `helm lint`, `helm template | kubeconform -strict`, kube-linter or Polaris.

## 6. GitOps
Argo CD (ApplicationSets or app-of-apps) or Flux, watching `deploy/<env>/`. CI promotes by opening (or auto-merging for dev) a change to the image digest; production promotion is a reviewed PR. Sync waves order CRDs → infra → apps; drift shows in the UI; rollback is a revert. Without GitOps, deploy from Actions with `helm upgrade --install --atomic --wait` per environment.

## 7. Progressive delivery
Argo Rollouts or Flagger for canary with automated analysis on Prometheus metrics (error rate, p95 latency vs SLO) and automatic abort; blue-green for changes that can't mix versions. Only when the NFRs (traffic, availability) justify it — rolling updates with good probes are the default.

## 8. Stateful services
Prefer managed Postgres / Redis outside the cluster. Where there is none (Hetzner), use CloudNativePG (streaming replicas, backups and WAL archiving to S3-compatible object storage, PITR, scheduled restore tests) or Postgres on dedicated VMs, plus PgBouncer; Redis / Valkey through an operator or a VM with persistence matching its role.

## 9. Upgrades
Track the supported minor versions of the chosen distribution; scan for deprecated APIs (pluto, kubent) before each upgrade; upgrade dev → staging → prod with node surge; keep add-ons compatible; managed providers' auto-upgrade channels only for non-prod unless the SLO allows it.
