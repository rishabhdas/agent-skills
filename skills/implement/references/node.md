# Node build & test guidelines

Detected via `package.json` at the service root.

All commands below run inside the service's Docker container (Step 4.2 of `SKILL.md`), never directly on the host.

## Containerized install & build

- Build the image: `docker build -t <service>-dev .` (or `docker compose build <service>` if the repo has a compose file). Rebuild whenever the Dockerfile or lockfile changes.
- Run every command in this file as: `docker run --rm -v "$(pwd):/app" -w /app <service>-dev <command>`, or `docker compose run --rm <service> <command>` if a compose file defines this service — prefer compose when available since it already carries the service's networking, env vars, and volume mounts.
- Install dependencies and run any build script (e.g. `tsc`, `webpack`, `next build`) inside the container using the commands below before running tests, lint, or type-check.

## Package manager

Pick based on the lockfile present — don't introduce a new one:

- `pnpm-lock.yaml` → `pnpm install`, run scripts via `pnpm run <script>`
- `yarn.lock` → `yarn install`, run scripts via `yarn <script>`
- `package-lock.json` (or none) → `npm install`, run scripts via `npm run <script>`

## Test

- Runner: whatever `package.json`'s `scripts.test` invokes (Jest, Vitest, Mocha, etc.) — use that, don't add a second test framework.
- Run only the affected package/workspace's tests during Step 4 iterations; run the full suite at Step 5.
- Mock external calls with the runner's mocking utilities (`jest.mock`, `vi.mock`, `nock` for HTTP) — not real network/DB/filesystem access.

## Lint & format

- Format: `prettier --write .` if `.prettierrc*` exists.
- Lint: `eslint .` (or `npm run lint` if defined) using the repo's existing config — don't add new rules as part of a feature change.

## Type checking

- If TypeScript (`tsconfig.json` present): `tsc --noEmit`.
- Treat new type errors introduced by the change as failures, even if the repo has pre-existing unrelated ones.

## Service standards

- Entry point and `package.json` live at the service directory root (monorepo layout from `CLAUDE.md`); use npm/yarn/pnpm workspaces for shared internal packages, not relative `../` imports across services.
- Dockerfile: multi-stage build — install/build in one stage, copy only `dist`/`build` output plus production `node_modules` into the runtime stage.
- Tracing/metrics: instrument with `@opentelemetry/sdk-node` and the relevant auto-instrumentations for the framework in use (Express, Fastify, etc.).
- API spec: keep the OpenAPI spec in sync — generate from decorators/annotations if the framework supports it (e.g. `@nestjs/swagger`), otherwise hand-edit the spec file alongside route changes.
</content>
</invoke>
