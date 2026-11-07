# Observability and reliability

## 1. Signals and stack
- Services emit OpenTelemetry traces, metrics and structured JSON logs (the backend sets up the SDKs; the frontend reports errors and web vitals).
- An OpenTelemetry Collector (agent per node + gateway, or a sidecar-free deployment) receives, enriches (`service.name`, `service.version`, `deployment.environment`), redacts PII and exports.
- Backends as the architect decided: self-hosted Grafana LGTM (Prometheus or Mimir, Loki, Tempo, Grafana), Grafana Cloud, or the cloud-native suites (CloudWatch / X-Ray, Cloud Monitoring / Logging / Trace, Azure Monitor / Application Insights); when asked, give cost, residency and operational facts per option. Error tracking (e.g. Sentry) for backend and frontend, with source maps and releases uploaded from CI.
- Label cardinality budgets (no user ids or raw paths as metric labels); log retention per environment and GDPR.

## 2. Dashboards and SLOs
- Per service: RED (rate, errors, duration) per endpoint, saturation (CPU, memory, pool usage, queue depth), dependencies; per infrastructure component: USE. Deploy markers on every dashboard.
- SLOs come from the NFRs: availability and latency SLIs per user journey, an error budget per window.
- Alerts on SLO burn with multi-window, multi-burn-rate rules (fast burn pages, slow burn opens a ticket). Cause-based alerts (disk filling, certificate expiring, backup failed, replication lag) are tickets unless they threaten the SLO soon.
- Dashboards, alerts and SLOs are code (Grafana provider in Terraform, Jsonnet / Grafonnet, PrometheusRule CRDs, or the cloud's IaC) and reviewed like code.

## 3. Alert routing
Every alert has an owner, a severity, a runbook link and a dashboard link. Route through Alertmanager or the cloud's notifications to the team's paging and chat tools; quiet hours and escalation per the owner's support model (a solo owner may want only critical pages).

## 4. Health checks
Liveness = the process is alive (no dependency checks). Readiness = can serve now (light checks on required dependencies). Startup = finished booting. Load balancers and gateways use readiness; misused liveness probes cause restart storms.

## 5. Runbooks
`docs/devops/runbooks/<slug>.md` for each top failure mode in `03-deployment.md` and each paging alert: symptom, impact, dashboards and queries, diagnosis steps, mitigations (with exact commands or pipeline runs), rollback, escalation, how to verify recovery.

## 6. Incidents
Severity levels with response expectations; one incident lead; a timeline as you go; status updates for the owner; blameless postmortem in `docs/devops/incidents/` within days: impact, timeline, root causes and contributing factors, what went well, action items as `OPS-xx` with owners.

## 7. Backups and disaster recovery
- Managed Postgres: automated backups and PITR with retention covering the RPO; cross-region / cross-account copies when the DR design asks.
- Self-run Postgres (CloudNativePG or VMs): base backups and continuous WAL archiving to object storage in another location; Redis persistence only where its role needs durability.
- Object storage: versioning, lifecycle rules, replication per DR.
- Restore drills on a schedule (`operate dr-drill`): restore into an isolated environment, measure RTO and data loss, run smoke tests. A backup that was never restored doesn't count.
- The ultimate DR test: rebuild an environment from IaC plus backups; the time it takes is your real RTO.

## 8. Capacity and performance
- Load tests (k6 or Gatling; qa owns the scenarios, you provide runners and a production-like environment) before launch and before large changes; validate autoscaling and limits.
- Connection budgets: app pool size × replicas ≤ the database's limit minus headroom; PgBouncer in transaction mode when replicas scale out.
- Right-size from observed usage (requests near p90 of real use), revisit monthly with the cost review.

## 9. Deploy safety
Health-gated rollouts, automatic rollback on failed readiness, canary analysis against SLO metrics when progressive delivery is used, error-tracking release health checked after each deploy, deploy annotations on dashboards.
