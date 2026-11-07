# Containers

## 1. Dockerfile baseline
- Multi-stage: build tools in the builder, only the runtime in the final stage.
- Base images pinned by digest (`FROM python:3.13-slim@sha256:…`), updated by Renovate. Prefer minimal bases: `-slim`, distroless, or Wolfi / Chainguard images; Alpine only when musl is fine for the stack (Python wheels and some JVM tooling suffer).
- Run as a non-root numeric user (`USER 10001`); files owned by root and read-only; app works with a read-only root filesystem (writable `/tmp` via a volume).
- No secrets in layers or build args: BuildKit `RUN --mount=type=secret,id=…`; `.dockerignore` excludes `.git`, `.env*`, `node_modules`, build output, tests.
- Cache mounts for package managers (`--mount=type=cache,target=/root/.cache/uv`, `~/.gradle`, the pnpm store).
- Exec-form `ENTRYPOINT` / `CMD` so the process receives SIGTERM; add `tini` (or `--init`) when the app spawns children; the app drains in-flight work on SIGTERM.
- OCI labels: `org.opencontainers.image.source`, `.revision`, `.version`.
- `HEALTHCHECK` only for Compose use; Kubernetes uses probes.

## 2. Python (uv)
```dockerfile
FROM ghcr.io/astral-sh/uv:<version>@sha256:… AS uv
FROM python:3.13-slim@sha256:… AS build
COPY --from=uv /uv /usr/local/bin/uv
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never
WORKDIR /app
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-dev --no-install-project
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev

FROM python:3.13-slim@sha256:…
ENV PYTHONUNBUFFERED=1 PATH="/app/.venv/bin:$PATH"
WORKDIR /app
COPY --from=build --chown=0:0 /app /app
USER 10001
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
Worker count and server (uvicorn / gunicorn) per the backend's guidance; one process per container when the platform scales replicas.

## 3. Java (Spring Boot)
- Build with the Gradle / Maven wrapper in a JDK stage (cache mount for `~/.gradle` or `~/.m2`), extract the layered jar (`java -Djarmode=tools -jar app.jar extract --layers --launcher`) and copy layers in order of change frequency; or use Jib / `bootBuildImage` when the backend prefers.
- Runtime on a JRE image (Temurin JRE or distroless Java) matching the Java version; `-XX:MaxRAMPercentage=75` and no fixed `-Xmx` so memory follows the container limit; consider CDS / AOT cache for startup; GraalVM native only with an ADR.
- Startup probes in Kubernetes give the JVM time before liveness counts.

## 4. Web frontend
- Static SPA / SSG: build in a Node stage and publish the assets to object storage + CDN (`references/release-and-delivery.md` §9) rather than serving them from an nginx container. Only containerize when the platform requires it.
- SSR (e.g. Next.js standalone output): Node runtime image with only the standalone output and static assets.
- Build once, promote: no environment values baked into the bundle — runtime config comes from a `config.json` fetched at boot, a server-rendered config, or edge-injected headers.

## 5. Multi-arch
Build `linux/amd64,linux/arm64` when any environment runs ARM (Hetzner CAX, AWS Graviton, Ampere on GCP / Azure — usually cheaper per unit of work). Prefer native arm64 runners over QEMU emulation for speed; test both architectures.

## 6. Tags and registries
- Every build tagged `sha-<short>`; releases add semver; deployments always pin the digest. Never deploy `latest`.
- GHCR by default (same identity as the repo); cloud registries (ECR, Artifact Registry, ACR) when the runtime pulls faster or cheaper from them — replicate rather than build twice.
- Retention policies for untagged and preview images.

## 7. Supply chain
- SBOM per image (buildx `--sbom=true` or syft), attached as an attestation.
- Scan with Trivy or Grype; fail on fixable CRITICAL / HIGH; time-boxed waivers in a tracked file.
- Sign keyless with cosign (GitHub OIDC) and attach build provenance; verify signatures at admission (Kyverno / policy-controller) when on Kubernetes.

## 8. Size and checks
- `hadolint` on every Dockerfile; `dive` or `docker image ls` to watch size; rebuild weekly so base-image fixes land even without code changes.
