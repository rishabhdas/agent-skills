# Docker Compose

## Compose file structure

Use the modern spec (no `version:` key needed in Compose v2+; if present and outdated, it's ignored by v2 with a warning — fine to drop it).

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    image: myapp/api:${TAG:-dev}
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/app
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "127.0.0.1:3000:3000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s
    networks:
      - backend

  db:
    image: postgres:16.4
    volumes:
      - db_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10
    networks:
      - backend

volumes:
  db_data:

networks:
  backend:
```

## `depends_on` — the ordering trap

Plain `depends_on: [db]` only waits for the container to *start*, not for the app inside it to be *ready*. Postgres/MySQL/etc. accept TCP connections briefly before they're actually ready to serve, causing intermittent startup failures. Always pair with `condition: service_healthy` and a real `healthcheck` on the dependency — don't paper over it with a `sleep`/wait-for-it script unless the image genuinely has no way to expose health.

## Networking and service discovery

- Services on the same user-defined network resolve each other by service name (`db`, not `localhost` or an IP) via Compose's embedded DNS. This is why `DATABASE_URL` above points at host `db`.
- Separate networks to segment exposure: a `frontend` network the reverse proxy and app share, a `backend` network the app and database share, with the database container never on `frontend`. Prevents anything on the public-facing network from reaching the DB directly even if it's compromised.
- Only publish ports (`ports:`) for services that need host/external access. Internal-only services (databases, caches, workers) should rely on the Compose network and omit `ports:` entirely, or bind to `127.0.0.1` if a human needs occasional local access.

## Environment variables and secrets

- `.env` file at the project root is auto-loaded for variable substitution (`${VAR}` in the compose file) — this is *not* the same as `env_file:` on a service, which injects vars into the container's runtime environment. Know which one you need.
- Never commit `.env` with real secrets — commit `.env.example` with placeholder keys, gitignore `.env`.
- For anything beyond local dev, prefer Compose's `secrets:` (file-based, mounted at `/run/secrets/<name>`, not visible in `docker inspect` env output the way `environment:` values are):
  ```yaml
  services:
    api:
      secrets: [db_password]
  secrets:
    db_password:
      file: ./secrets/db_password.txt
  ```

## Multi-environment patterns

Base file + overrides, not one file with conditionals:
```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```
Or `docker-compose.override.yml`, which Compose loads automatically alongside the base file when present (good for local-dev-only tweaks that shouldn't need an explicit `-f`).

`profiles:` to make some services opt-in (e.g. a debug tool or seed job that shouldn't start on every `up`):
```yaml
services:
  seed:
    profiles: ["tools"]
```
```bash
docker compose --profile tools up
```

## Volumes

- Named volumes for anything that must persist across `down`/`up` (database data). `docker compose down` alone leaves named volumes intact; `docker compose down -v` removes them — know which one you're running before a "quick restart."
- Bind mounts for local dev live-reload: `./src:/app/src`. Avoid bind-mounting the whole app directory over a built image's `node_modules`/`vendor` — use an anonymous volume for that subdirectory to keep the container's installed deps instead of shadowing them with the (possibly absent/wrong-platform) host copy:
  ```yaml
  volumes:
    - ./src:/app/src
    - /app/node_modules   # anonymous volume "wins" over the bind mount above for this path
  ```

## Restart policies

```yaml
restart: unless-stopped   # survives daemon/host restart, but respects a manual `docker compose stop`
```
`no` (default) for one-shot/dev services, `unless-stopped` for anything meant to stay up, `on-failure` for jobs that should retry a few times then give up (`on-failure:3`). Avoid bare `always` in dev — it fights you when you're trying to intentionally stop something to debug it.

## Resource limits

Compose respects `deploy.resources` only under Swarm; for plain `docker compose up`, use the non-`deploy` form:
```yaml
services:
  api:
    mem_limit: 512m
    cpus: "0.5"
```
Without limits, one runaway container's memory/CPU use isn't contained — worth setting even in dev to catch leaks early, and necessary in any shared CI/build-host context.

## Common failure modes

- **"port is already allocated"**: another container or host process already bound that port. `docker ps` / `lsof -i :<port>` to find it, or just change the host-side port mapping.
- **Service can't reach another by name**: they're on different networks (check `networks:` on both), or one hasn't defined `networks:` at all and fell onto Compose's default project network while the other is on a custom one.
- **Changes to code not reflected**: bind mount misconfigured (wrong path, or an anonymous volume shadowing it — see above), or the image was rebuilt-cached and needs `docker compose build --no-cache` / `up --build`.
- **`.env` values not substituting**: `.env` must be in the directory `docker compose` is invoked from (or set via `--env-file`), and only affects variable substitution in the compose YAML — it does not automatically become the container's environment unless also referenced via `environment:` or `env_file:` on the service.
- **Stale containers after compose file changes**: `docker compose up -d --remove-orphans` to clean up services removed from the file; `docker compose up -d --force-recreate` when a change (e.g. to `environment:`) isn't being picked up.
