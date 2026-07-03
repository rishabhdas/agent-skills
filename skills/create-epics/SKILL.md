# Create Epics Skill

You are executing the `create-epics` skill. Read these instructions fully before taking any action.

## Purpose

Parse a spec document, BRD, or business requirement description and produce logical delivery Epics. Each Epic represents an end-to-end feature delivery that spans one or more project components (frontend, microservices, data layer, infrastructure). Epics must have clear objectives, component-level scope, and verifiable acceptance criteria.

---

## Step 1 — Gather inputs

If the user has not provided all of the following, ask for them before proceeding:

1. **Input document** — a spec file path, a GitHub/GitLab issue URL, or inline text. If none is given, ask: "What spec or requirement doc should I use to generate epics?"

2. **Operating mode** — ask if not stated:
   - `github` — create epics as GitHub issues via the `gh` CLI (labels: `epic`, plus any component labels)
   - `gitlab` — create epics as GitLab epics via `glab` CLI
   - `local` — write epics into an `epics.md` file in the relevant directory

3. **GitHub/GitLab target** (only for `github`/`gitlab` mode) — repo or group path (e.g. `org/repo`). If the working directory is already a git remote, detect it from `git remote get-url origin` and confirm before using it.

4. **Scope clarification** — if the input document is broad (covers multiple independent domains), ask: "Should I generate one epic per major feature area, or one epic per user-facing workflow?"

Only ask questions that are genuinely unanswerable from the input. Do not ask about things you can infer.

---

## Step 2 — Analyse the input document

Read and analyse the provided document. Extract:

- **Business goals** — what outcome does this deliver for users or operators?
- **User workflows** — what does a user do, end to end?
- **Affected components** — which services, frontend, datastores, or infra layers are touched?
- **Cross-component dependencies** — which service calls which; which schema changes are needed?
- **Constraints or non-functional requirements** — performance, security, compliance, availability.

Group related requirements into logical delivery units. Each unit becomes one Epic. An Epic should be completable in a sprint or two; if one unit is clearly much larger, split it.

---

## Step 3 — Draft Epics

For each Epic, produce a structured definition using the template in `TEMPLATE.md` (same directory as this skill). Read that file before drafting. Do not skip any section. Populate every field from what you learned in Step 2.

---

## Step 4 — Clarify before finalising

Before writing output, briefly summarise the epics you plan to create (title + one-sentence description each) and ask: "Does this breakdown look right, or should any epics be merged, split, or reframed?"

Incorporate feedback. Only proceed to Step 5 once the user confirms the breakdown.

---

## Step 5 — Output epics

### Local mode

Write epics to `epics.md` in the most relevant directory:
- If the input doc lives inside a service directory, write `epics.md` there.
- If the input covers the whole platform, write to the repo root or `docs/`.
- If `epics.md` already exists, append new epics below a `---` divider with a header noting the date they were added. Do not overwrite existing content.

Format: use the Epic template from `TEMPLATE.md`, one `##` section per epic, in priority order (highest-value / least-blocked first).

### GitHub mode

For each Epic, run:

```bash
gh issue create \
  --repo <owner/repo> \
  --title "Epic: <title>" \
  --label "epic,<component-labels>" \
  --body "<rendered epic body>"
```

The body should contain all sections from the template rendered as plain GitHub Markdown (no code fences around the whole body).

After creating all issues, print a summary table:

| Epic | Issue URL |
|------|-----------|
| <title> | <url> |

### GitLab mode

For each Epic, use `glab api` to create a GitLab epic on the group:

```bash
glab api groups/<group-id>/epics \
  --method POST \
  --field title="Epic: <title>" \
  --field description="<rendered epic body>"
```

Print the same summary table as GitHub mode.

---

## Constraints

- Every Epic must touch at least one service boundary or data contract — pure frontend or pure config changes that don't cross a service are issues, not epics.
- Every Epic must include observability acceptance criteria (OpenTelemetry traces and Prometheus-compatible metrics) in line with project standards.
- If an Epic modifies any API surface, it must include an acceptance criterion that the OpenAPI/Swagger spec is updated.
- If an Epic requires a new microservice, it must include a Dockerfile and a separate sub-item for service scaffolding.
- Do not number epics sequentially (Epic-1, Epic-2) — use descriptive titles only. Numbering becomes stale as scope changes.
- Write for an engineer who has not read the source spec — the Epic body must be self-contained.
