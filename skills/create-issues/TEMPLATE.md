## <short imperative title — verb + noun phrase>

**Parent Epic**
<!-- Link to the parent Epic: GitHub/GitLab URL, or `epics.md#<section-anchor>` for local mode -->

**Summary**
One paragraph. What discrete outcome does this issue deliver? Why is it needed at this point in the delivery sequence?

**Size estimate**
`XS` / `S` / `M` / `L` / `XL` — one-line rationale. Estimated implementation context: ~N k tokens.

---

### Scope

What this issue covers:

- <!-- bullet per concrete deliverable: file, endpoint, schema, component, config, etc. -->

**Out of scope**
What this issue explicitly does NOT include (prevents scope creep and clarifies handoff points):

- <!-- bullet per explicit exclusion -->

---

### Implementation guide

<!-- Describe WHAT must exist when done, not HOW to implement it. Name the files/modules/interfaces to create or modify. -->

**Files / modules to create or modify**

| Path | Action | Notes |
|------|--------|-------|
| `src/...` | create / modify | brief note |

**Interfaces and contracts**

<!-- List any API endpoints, event schemas, or data models this issue must produce or consume. -->

- Endpoint: `METHOD /path` — request/response shape (or reference to OpenAPI spec section)
- Event: `<topic>` — payload schema
- Model: `<Table/Collection>` — fields and constraints

**Observability requirements**

<!-- Required even if the parent Epic did not call it out explicitly. -->

- Traces: list the spans this issue must emit (operation name, key attributes)
- Metrics: list Prometheus metric names, types, and labels
- Logs: any structured log events with fields

---

### Acceptance criteria

<!-- Each item must be independently testable and binary (pass/fail). -->

- [ ] <!-- functional: API returns expected response for happy-path input -->
- [ ] <!-- functional: API returns correct error code and body for invalid input -->
- [ ] <!-- auth: unauthenticated requests are rejected with 401 -->
- [ ] <!-- observability: span `<name>` is emitted with attributes `<list>` -->
- [ ] <!-- observability: counter `<metric_name>` increments on each request -->
- [ ] <!-- contract: OpenAPI spec updated and passes schema validation (if API surface changed) -->
- [ ] <!-- build: Dockerfile builds without error and container passes health check (if service modified) -->
- [ ] <!-- tests: unit tests pass with ≥ 80 % line coverage for new code -->
- [ ] <!-- tests: integration test covers the end-to-end flow introduced by this issue -->

---

### Dependencies

**Blocked by** (this issue cannot start until these are done)
- <!-- Issue title + number/file, reason the dependency exists -->

**Blocks** (these issues cannot start until this one is done)
- <!-- Issue title + number/file, reason the dependency exists -->

**External dependencies** (outside this repo or this sprint)
- <!-- third-party APIs, platform features, other teams' work -->

---

### Labels / components

<!-- Comma-separated: e.g. `backend`, `frontend`, `data`, `infra`, `auth`, `observability` -->
