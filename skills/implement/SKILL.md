# Implement Skill

You are executing the `implement` skill. Read these instructions fully before taking any action.

## Purpose

Implement a single issue, an Epic with multiple linked issues, or inline instructions from the user. Work happens on a dedicated branch, follows Test-Driven Development, and ends with the user reviewing and choosing whether to publish upstream.

---

## Step 1 — Gather inputs

Ask for any of the following that the user has not provided:

1. **What to implement** — one of:
   - A single issue (GitHub URL, GitLab URL, local issue file, or issue number in a repo already known from context)
   - An Epic (fetch it and its linked/child issues — `gh issue view <number> --json title,body,labels` plus any issues referencing it, or `glab api` equivalents, or a local `epics.md` section and its associated issue files)
   - Inline instructions typed directly by the user
2. **Target repo/path** — if operating in `github`/`gitlab` mode, confirm `owner/repo` via `git remote get-url origin` if not stated.

Do not ask about things determinable from the input or repo state.

---

## Step 2 — Resolve scope and dependency order

- If given a single issue or inline instructions, scope is that one unit of work.
- If given an Epic, collect every linked/child issue. For each, identify:
  - What it depends on (blocked-by relationships, shared data contracts, shared services)
  - What depends on it
  - Build a dependency-ordered implementation sequence — never implement an issue before something it's blocked by. Independent issues may be grouped, but still implemented one at a time, each in its own branch.
- Present the resolved order to the user as a short numbered list and confirm before starting. If any dependency is ambiguous, ask rather than guess.

---

## Step 3 — Start a branch per issue

Before implementing each issue:

1. Confirm the working tree is clean (`git status`). If there are uncommitted changes unrelated to this work, stop and ask the user how to proceed — do not stash or discard silently.
2. Create and check out a new branch named after the issue, e.g. `issue-<number>-<short-slug>` (or `<epic-slug>/issue-<number>-<short-slug>` when working through an Epic). Derive the slug from the issue title.
3. Branch from the repo's default branch (`git remote show origin` or `main`/`master`) unless the user specifies otherwise.

Each issue gets its own branch — never combine multiple issues on one branch, even within the same Epic.

---

## Step 4 — Implement with TDD

For each issue, in order:

1. **Detect the language(s) in play** for the service(s) being touched and read the matching reference before writing any code or tests:
   - `pyproject.toml`, `requirements.txt`, `setup.py`, or `Pipfile` present → read `references/python.md`
   - `package.json` present → read `references/node.md`
   - Both present (polyglot repo/monorepo) → read both, and apply each to the service(s) written in that language.
   - Use the reference's build/test/lint/type-check commands throughout this step and Step 5 instead of guessing generic commands.
2. **Build the service's Docker image and run all install/compile/build steps inside it** — never install dependencies or compile on the host:
   - Build (or rebuild, if the Dockerfile or lockfile changed since the last build) the image: `docker build -t <service>-dev .`, or `docker compose build <service>` if the repo has a compose file.
   - Run the install and compile/build commands from the language reference's "Containerized install & build" section via `docker run --rm -v "$(pwd):/app" -w /app <service>-dev <command>` (or `docker compose run --rm <service> <command>` if a compose file exists — prefer it, since it already carries the service's networking and env vars).
   - If the service has no Dockerfile yet, this step blocks: create one first (per the Dockerfile requirement below) rather than falling back to a host install.
   - Test, lint, and type-check commands also run through the same containerized invocation in the remaining items below and in Step 5, since they depend on the environment installed here.
3. **Write failing tests first**, before any implementation code:
   - **Positive tests** — expected inputs produce expected outcomes per the issue's acceptance criteria.
   - **Negative tests** — invalid input, error conditions, edge cases, and failure modes the issue implies.
   - Confirm the tests fail for the right reason (missing implementation, not a typo) before writing implementation.
4. **Mock external calls.** Any call crossing a service boundary — other microservices, third-party APIs, databases, message queues, filesystems outside the service — must be mocked in tests. Never let tests depend on live external systems.
5. **Implement the minimum code to pass the tests**, then refactor for clarity without changing behavior.
6. Follow project standards from `CLAUDE.md`:
   - Every microservice: its own directory (monorepo layout), a Dockerfile, OpenTelemetry tracing/metrics (Prometheus-compatible), and — if it exposes an API — an OpenAPI/Swagger spec kept in sync with the change.
7. Run that issue's test suite (inside the container, per item 2 above) and confirm it's green before moving to the next issue in the dependency order from Step 2.

---

## Step 5 — Full validation after each issue

After an issue's own tests pass, run the full test suite for the affected service(s) (and any dependent services if this issue unblocked them), via the same containerized invocation from Step 4, to confirm nothing regressed and the issue's acceptance criteria are met end-to-end. Report pass/fail plainly — do not mark an issue done on a red suite.

If implementing an Epic, repeat Steps 3–5 for each issue in dependency order. After the last issue, run the full suite across all touched services once more as an Epic-level acceptance check.

---

## Step 6 — Wait for user feedback

Present what was implemented (issue(s), branch(es), test results, any deviations from the acceptance criteria) and stop. Do not proceed to publishing until the user explicitly reviews and accepts the work. If the user requests changes, apply them on the same branch and re-run Step 5's validation before asking again.

---

## Step 7 — Offer to publish

Once the user accepts the implementation, ask whether they want to publish it upstream. Only proceed on an explicit yes.

If accepted:

1. Push the branch: `git push -u origin <branch-name>`.
2. Create a PR/MR referencing the source issue:
   - GitHub: `gh pr create --title "<title>" --body "<summary + Closes #<issue>>"`
   - GitLab: `glab mr create --title "<title>" --description "<summary + Closes #<issue>>"`
3. For an Epic, open one PR/MR per issue (matching the one-branch-per-issue rule), unless the user asks for them to be combined.
4. Report back the PR/MR URL(s).

Never push or open a PR/MR without the explicit go-ahead in this step.

---

## Step 8 — Reset context between features

Once an issue or Epic is fully implemented, reviewed, and (if accepted) published, tell the user this feature's work is complete and recommend clearing context / starting a new session before implementing the next issue or Epic. Do not carry Epic/issue state from one feature into the next session.

---

## Constraints

- Never implement code before its tests exist and fail correctly (TDD is mandatory, not optional).
- Never skip mocking an external call in tests without explicit user sign-off.
- Never combine multiple issues into a single branch or PR/MR without the user asking for it.
- Never push to a remote or open a PR/MR without explicit user acceptance in Step 7.
- Never silently discard or stash uncommitted work found at branch-start time — ask first.
- Respect dependency order across an Epic's issues even if it's less convenient than an alternate order.
- Never install dependencies, compile, or build on the host — those steps always run inside the service's Docker container (Step 4.2).
</content>
