# Docker

## Dockerfile structure

Order instructions from least to most frequently changing, so the build cache survives:

```dockerfile
FROM node:20.17.0-bookworm-slim AS base
WORKDIR /app

# deps change less often than source — copy manifests first
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM base AS build
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.17.0-bookworm-slim AS runtime
WORKDIR /app
RUN useradd --uid 10001 --create-home appuser
COPY --from=base /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD node healthcheck.js || exit 1
ENTRYPOINT ["node", "dist/main.js"]
```

Key moves: multi-stage build (build tools never reach the runtime image), copy dependency manifests before source so `npm ci`/`pip install`/etc. is cache-hit unless dependencies actually changed, non-root `USER`, pinned base image tag.

## Base image choice

- Pin to a specific version tag, not `latest` — `latest` silently changes what ships on every rebuild. Pin by digest (`@sha256:...`) for maximum reproducibility when supply-chain integrity matters.
- Prefer `-slim`/`-alpine`/distroless over full images when the toolchain supports it. Alpine's musl libc occasionally breaks native extensions (Python/Node C bindings) — if you hit weird native-module failures, that's the first suspect; `-slim` (Debian-based, glibc) is the safer default when unsure.
- Distroless (`gcr.io/distroless/*`) or `scratch` for compiled binaries (Go, Rust) — no shell, no package manager, smallest attack surface. Debugging requires `kubectl debug`/ephemeral containers since there's no shell to exec into.

## Layer caching rules

- Each `RUN`/`COPY`/`ADD` is a layer; a cache miss on one invalidates every layer after it.
- Combine related `RUN` commands with `&&` to avoid leaving stale files in intermediate layers (a later `rm` in a new layer does not shrink the image — the deleted files are still in the earlier layer).
- Use `.dockerignore` aggressively (`.git`, `node_modules`, build artifacts, `.env`) — anything not ignored gets sent to the build context and can bust cache or leak secrets.
- BuildKit cache mounts for package managers avoid re-downloading on every build without polluting the image:
  ```dockerfile
  RUN --mount=type=cache,target=/root/.npm npm ci
  RUN --mount=type=cache,target=/var/cache/apt apt-get update && apt-get install -y curl
  ```

## Secrets in builds

Never `ARG`/`ENV` a secret — both persist in image history (`docker history` reveals them even after later layers overwrite the file) and layer cache. Use BuildKit secret mounts instead:

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```
```bash
docker build --secret id=npm_token,src=$HOME/.npm_token .
```
The secret is available only during that `RUN` and never lands in a layer.

## Multi-stage builds

Use separate stages for: dependency install, build/compile, and runtime. Only `COPY --from=<stage>` the artifacts the runtime actually needs. This is the single biggest lever for both image size and reducing what ships to production (no compilers, no dev dependencies, no source maps unless wanted).

## Running as non-root

```dockerfile
RUN groupadd --gid 10001 appgroup && useradd --uid 10001 --gid appgroup --create-home appuser
USER appuser
```
- Bind to a port ≥1024 if the app must run unprivileged (ports <1024 need `CAP_NET_BIND_SERVICE` or root).
- If the base image ships a non-root user already (`node` image has `node`, e.g.), use it rather than creating a new one.
- Root inside a container is not fully isolated from the host without user namespace remapping — treat "runs as root in the container" as a real risk, not a formality.

## Image size and security scanning

- Multi-stage + slim/distroless base is 80% of the win. After that: combine `RUN` layers, clean package manager caches in the same layer (`apt-get clean && rm -rf /var/lib/apt/lists/*`), avoid installing recommended/suggested packages (`apt-get install --no-install-recommends`).
- Scan built images before pushing: `docker scout cves <image>`, `trivy image <image>`, or `grype <image>`. Wire this into CI, not just ad hoc.
- Verify what's actually in the final image when in doubt: `docker history <image>`, `dive <image>`.

## Networking

- User-defined bridge networks (`docker network create`) give containers DNS-based service discovery by container/service name — don't rely on the default bridge network, which doesn't.
- Publish only the ports that need external access (`-p 127.0.0.1:5432:5432` to bind loopback-only for a local DB you don't want reachable from the LAN, vs `-p 5432:5432` which binds all interfaces).
- `--network host` bypasses Docker's network isolation entirely — only for cases that specifically need it (some monitoring agents), never as a default fix for connectivity issues.

## Volumes and persistence

- Named volumes (`docker volume create`, or implicit via Compose) for data that must survive container recreation — database files, uploaded content.
- Bind mounts (`-v $(pwd):/app`) for local dev source-code live-reload; avoid in production images — they couple the container to host filesystem layout.
- Anonymous volumes (`VOLUME` in a Dockerfile with no host path) are easy to leak — they persist after `docker rm` unless removed with `-v`, silently consuming disk over time. Prefer named volumes so they're inspectable and prunable (`docker volume ls`, `docker volume prune`).

## Healthchecks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```
`--start-period` matters for slow-starting apps — without it, health checks during startup count as failures toward `--retries` and can cause a false "unhealthy" before the app is even up. Orchestrators (Compose `depends_on: condition: service_healthy`, Kubernetes readiness probes) key off this.

## Common failure modes

- **Build works locally, fails in CI**: usually a `.dockerignore` gap (file present locally, git-ignored, not sent to context) or an architecture mismatch (building `arm64` on Apple Silicon, deploying to `amd64` — fix with `docker buildx build --platform linux/amd64`).
- **Container exits immediately**: the main process (PID 1) exited. Check `ENTRYPOINT`/`CMD` actually runs a foreground process, not something that daemonizes/backgrounds itself.
- **Signals not handled / slow shutdown**: PID 1 doesn't forward signals by default for shell-form `CMD`. Use exec form (`CMD ["node", "server.js"]`, not `CMD node server.js`) or `--init`/`tini` so `SIGTERM` reaches the app and containers stop promptly instead of waiting out the full grace period.
- **"No space left on device" over time**: unpruned images/containers/volumes/build cache. `docker system df` to see what's consuming space, `docker system prune` (careful — confirm scope, `-a` also removes unused images and `--volumes` removes unused volumes).
