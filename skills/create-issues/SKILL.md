---
name: create-issues
description: Decompose an Epic into independently-developable implementation issues, each small enough to build in isolation, with acceptance criteria, labels, and blocking relationships, created via gh/glab or as local numbered Markdown files. Use when the user wants to break an epic down into issues or tickets, split work into implementable units, or asks to "create issues" from an epic.
---

# Create Issues Skill

You are executing the `create-issues` skill. Read these instructions fully before taking any action.

## Purpose

Take an Epic (a GitHub/GitLab issue, a local `epics.md` entry, or inline text) and decompose it into independently-developable implementation issues. Each issue must be:

- Small enough that a developer can implement it in isolation within a context window of roughly 120 k tokens (10 % tolerance allowed — do not exceed 132 k).
- Scoped so that it delivers a discrete, verifiable outcome — not just a task fragment.
- Linked to its parent Epic and aware of its blocking relationships with sibling issues.

---

## Step 1 — Gather inputs

Ask for any of the following that the user has not provided:

1. **Epic source** — one of:
   - A GitHub issue URL (e.g. `https://github.com/org/repo/issues/42`) — fetch with `gh issue view <number> --repo <owner/repo> --json title,body,labels`
   - A GitLab issue/epic URL — fetch with `glab api projects/<id>/issues/<iid>`
   - A path to a local `epics.md` file plus the epic title or section to break down
   - Inline text pasted directly into the conversation

2. **Operating mode** — ask if not stated:
   - `github` — create issues via the `gh` CLI in a GitHub repository
   - `gitlab` — create issues via the `glab` CLI in a GitLab project
   - `local` — write issues as individually numbered Markdown files (e.g. `001-issue.md`, `002-issue.md`) in a directory the user specifies

3. **GitHub/GitLab target** (modes `github`/`gitlab` only) — `owner/repo` or project path. If the working directory has a git remote, detect it with `git remote get-url origin` and confirm before using.

4. **Label prefix or naming convention** (optional) — if the team already uses label conventions (e.g. `be:`, `fe:`, `data:`), note them so generated labels match.

Only ask for what cannot be inferred. Do not ask about things you can determine from the input.

---

## Step 2 — Analyse the Epic

Read the Epic in full. Extract:

- **Objective** — what user or system outcome does this Epic deliver?
- **Components touched** — frontend, specific microservices, data layer, infrastructure, observability.
- **Implicit sub-problems** — list every distinct concern: API design, data modelling, service logic, auth/authz, UI, tests, Dockerfile, OpenAPI spec, observability instrumentation, migrations, CI wiring, etc.
- **Shared contracts** — data schemas, API interfaces, or event formats that multiple issues depend on.
- **External dependencies** — third-party integrations, other Epics, or platform prerequisites.

---

## Step 3 — Identify issue boundaries and size each issue

Apply these rules to define issue boundaries:

### Boundary rules

1. **One service boundary per issue** — if the work spans two microservices, split by service.
2. **Contracts before consumers** — define API specs, event schemas, and data models before the issue that implements or consumes them. The contract issue is always a blocker for its consumers.
3. **Scaffolding before logic** — if a new microservice is needed, the first issue is always service scaffolding (repo layout, Dockerfile, CI entry, health endpoint) before any domain logic.
4. **Observability is not optional** — every issue that adds or modifies a service endpoint, background job, or data pipeline must include OpenTelemetry traces and Prometheus-compatible metrics in its scope. Do not create a separate "add observability" issue unless the work is a pure retrofit.
5. **Tests are part of the issue** — unit and integration tests ship in the same issue as the code they cover. Do not create separate "write tests for X" issues.
6. **Infrastructure and config** — if infra changes (secrets, env vars, Helm values, Terraform) are needed but trivial (< 1 hour), include them in the issue that needs them. If they require significant standalone work, split them.

### Size estimation

Estimate each candidate issue using this rubric:

| Label | Approximate scope |
|-------|------------------|
| XS | A single function or config change; < 5 k tokens |
| S | A single endpoint or component; 5–25 k tokens |
| M | A full service layer or feature; 25–60 k tokens |
| L | A complex integration; 60–100 k tokens |
| XL | Near the 120 k limit; consider splitting further |

**If an issue is estimated XL and can be split along a clean boundary, split it.** Do not produce issues that would require > 132 k tokens of context to implement in a single development session.

---

## Step 4 — Build the dependency graph

Before drafting issue text, map the dependency graph:

1. List every issue candidate as a node.
2. Draw a directed edge A → B if B cannot start until A is complete.
3. Identify the critical path (longest chain of blockers).
4. Check for cycles — if you find one, the boundary between two issues is wrong; merge them or redefine the split.
5. Assign each issue a draft sequence number that reflects topological order (1 = no blockers, highest = most blocked). These sequence numbers are used for naming local files and for ordering GitHub/GitLab issue creation, not for permanent identifiers.

---

## Step 5 — Clarify before drafting

Present the user with a brief plan:

- A numbered list of proposed issues (title + one-sentence scope + size label)
- The dependency graph as a simple text list: `Issue N blocks Issue M`
- Any assumptions you made that the user should validate

Ask: "Does this breakdown look right? Should any issues be merged, split, or reordered?"

Incorporate feedback. Only proceed to Step 6 once the user confirms.

---

## Step 6 — Draft each issue using the template

Read `TEMPLATE.md` in this directory before drafting. Use it for every issue without exception. Populate every field. Do not leave sections empty or marked "TBD".

Key requirements per issue:

- **Parent Epic link** — always include a reference (URL for GitHub/GitLab, section heading for local mode).
- **Blocking relationships** — list what this issue blocks and what blocks it, by title and issue number/file name once known.
- **Acceptance criteria** — each criterion must be binary (pass/fail) and independently verifiable without running the full system where possible.
- **Implementation scope** — a bulleted list precise enough that a developer knows exactly what files to create or modify and what interfaces to implement. Do not describe *how* to implement; describe *what* must exist when done.
- **Out of scope** — explicitly state what this issue does NOT include to prevent scope creep.

---

## Step 7 — Output issues

### Local mode

Create one Markdown file per issue in the directory the user specified (default: `issues/` relative to the Epic's location).

File naming: `<NNN>-<kebab-case-title>.md` where `NNN` is the zero-padded sequence number from Step 4.

Example: `001-service-scaffolding.md`, `002-api-contract.md`, `003-order-service-logic.md`.

If the `issues/` directory already contains files, continue the sequence from the highest existing number. Do not overwrite existing files.

After writing all files, print a summary table:

| # | File | Title | Size | Blocks |
|---|------|-------|------|--------|
| 1 | 001-... | ... | S | 002, 003 |

### GitHub mode

Create issues in dependency order (least-blocked first) so that `gh` issue numbers are available for cross-linking.

For each issue:

```bash
gh issue create \
  --repo <owner/repo> \
  --title "<title>" \
  --label "<labels>" \
  --body "<rendered body>"
```

After all issues are created, add blocking relationships using the GitHub "blocks / is blocked by" convention in the body (since GitHub has no native blocker field, use the section from the template).

Then add a comment on the parent Epic issue listing all child issue URLs:

```bash
gh issue comment <epic-number> \
  --repo <owner/repo> \
  --body "## Issues created from this Epic\n\n<table of issue URLs>"
```

Print a summary table:

| # | Issue URL | Title | Size | Blocks |
|---|-----------|-------|------|--------|

### GitLab mode

Create issues via `glab api`:

```bash
glab api projects/<id>/issues \
  --method POST \
  --field title="<title>" \
  --field description="<rendered body>" \
  --field labels="<labels>"
```

Link each issue to the parent Epic after creation:

```bash
glab api projects/<id>/issues/<iid>/resource_milestone_events   # if using milestones
# or add the epic link via the UI / API depending on GitLab tier
```

Print the same summary table as GitHub mode.

---

## Constraints

- Never produce an issue that requires another issue to be *partially* done before it can start — dependencies must be all-or-nothing.
- Every issue that introduces or modifies a service must include an acceptance criterion that the service's Dockerfile builds and passes a smoke test.
- Every issue that adds or changes an API endpoint must include an acceptance criterion that the OpenAPI/Swagger spec is updated and valid.
- Every issue that modifies service code must include OpenTelemetry trace and Prometheus metric acceptance criteria per project standards.
- Do not number issues in their titles (e.g. "Issue 1: ...") — use descriptive titles only. Sequence numbers belong in file names and cross-references only.
- Write each issue as if the reader has not seen the Epic — the issue body must be self-contained enough to implement without re-reading the parent.
- Do not create placeholder or "spike" issues unless the user explicitly requests them. Discovery work should be folded into the implementation issue it unblocks.
