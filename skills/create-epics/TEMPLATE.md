## Epic: <short title — verb + noun phrase>

**Summary**
One paragraph. What is being built, why it matters, and what user or business problem it solves.

**Scope**
Which project components are involved:
- Frontend: <yes/no — what UI changes>
- Services: <list of microservices and what each must do>
- Data layer: <schema changes, new tables, migrations>
- Infrastructure: <config, secrets, env, Docker, CI changes>
- Observability: <new metrics, traces, or dashboards>

**Out of scope**
What this Epic explicitly does NOT include (reduces ambiguity during planning).

**User workflow**
Step-by-step narrative of the end-to-end user journey this Epic delivers. Reference the services and datastores touched at each step.

**Objectives**
Numbered list of concrete outcomes the Epic must achieve.

**Acceptance criteria**
Checklist. Each item must be testable and binary (pass/fail). Cover:
- Functional behaviour (API responses, UI states)
- Error and edge cases
- Non-functional requirements (latency targets, auth enforcement)
- Observability (traces and metrics are emitted)
- Documentation (OpenAPI spec updated if API surface changes)

**Dependencies**
- Blocked by: <other Epics or external work this Epic cannot start without>
- Blocks: <other Epics that cannot start until this one is done>

**Estimated size**
S / M / L / XL — use gut feel from scope. Add a one-line rationale.

**Labels / components**
Comma-separated list of applicable labels (e.g. `epic`, `backend`, `frontend`, `data`, `infra`, `auth`).
