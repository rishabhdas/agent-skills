---
name: refine-plan
description: Run an interactive, one-question-at-a-time brainstorming session against an existing plan or spec to surface missing scenarios, corner cases, error paths, and ambiguities, then produce a new spec version capturing every decision. Use when the user wants to review, refine, pressure-test, or find gaps in a plan or spec before implementation.
---

# Refine Plan Skill

You are executing the `refine-plan` skill. Read these instructions fully before taking any action.

## Purpose

Run an interactive brainstorming session against an existing plan or spec to surface missing scenarios, corner cases, and ambiguities, then produce a new spec version that captures every decision reached. This skill is conversational, not a one-shot rewrite — the plan only improves through back-and-forth with the user.

---

## Step 1 — Gather the input

If not already provided, ask for the plan/spec to refine: a file path, issue/PR URL, or inline text. Read it in full before proceeding. If the document is very large, confirm which section(s) are in scope for this session rather than guessing.

---

## Step 2 — Build a review list (silent analysis)

Before talking to the user, analyze the spec and privately draft a list of review items. For each, look for:

- **Missing scenarios** — user goals or flows implied but not written down.
- **Corner cases** — empty/null inputs, boundary values, concurrent access, partial failure, retries, duplicate requests.
- **Error and failure paths** — what happens when a dependency is down, times out, or returns malformed data.
- **Ambiguous or conflicting requirements** — statements that could be read two ways.
- **Non-functional gaps** — security, auth/authz, performance/scale, observability, data retention, backward compatibility.
- **Cross-cutting consistency** — does this align with project standards (e.g. `.claude/CLAUDE.md` monitoring, Dockerfile, OpenAPI requirements)?

Order the list roughly by risk/impact (highest first). This list drives the session but is never dumped on the user at once.

---

## Step 3 — Run the brainstorming session

Present exactly **one** item at a time. For each item:

1. State the scenario or gap plainly, and why it matters (concretely — what breaks or is undefined if left unresolved).
2. Ask an open question, or if the user explicitly asks for a recommendation, give **one** concrete recommendation framed with three response options:
   - **Accept** — adopt the recommendation as-is.
   - **Modify** — the user adjusts it; incorporate their change.
   - **More alternatives** — propose 1-2 additional options before they decide.
3. Do not move to the next item until the current one reaches an explicit resolution from the user. Silence or an ambiguous reply is not acceptance — ask again or rephrase.
4. Record the resolution against that item (short note: decision + rationale) as you go, so context isn't lost across turns.

Use `AskUserQuestion` for items with clean discrete options; use plain conversation for open-ended ones. Never bundle multiple items into a single question.

---

## Step 4 — Track session state

Maintain a running status per review item: `Open`, `Resolved`, or `Deferred` (user explicitly chose to skip it). Periodically (e.g. every few items, or when asked) show a short progress recap: counts of resolved vs. open, not the full detail.

New items may surface mid-conversation (the user's answers often reveal further gaps) — add them to the list and work them the same way, one at a time.

---

## Step 5 — End the session

The session ends when either:

- **All items reach `Resolved` or `Deferred`** and the user gives explicit acceptance of the full set (e.g. "yes, that covers it") — do not treat implicit progress as final sign-off.
- **The user explicitly chooses to end early.** Honor this immediately even with items still `Open`; carry those forward into the output as explicitly open/unresolved rather than silently dropping them.

---

## Step 6 — Produce the new spec version

Synthesize the **original spec** plus **every decision from the session** into a new, self-contained document — do not require the reader to reconstruct decisions from the conversation.

- Write to a new file alongside the original (e.g. `plan.md` → `plan-v2.md`, or increment an existing version suffix). Do not overwrite the original.
- Preserve original structure/sections where still valid; insert new or revised content in place rather than appending an unstructured changelog.
- Include a short "Changes from previous version" section summarizing what was added/changed and why, plus any items left `Deferred`/`Open` at session end so they aren't lost.
- Apply relevant project standards from `.claude/CLAUDE.md` (observability, Dockerfile, per-service directories, OpenAPI) when a resolved item touches them.

---

## Constraints

- One question at a time — never present a batch or numbered list of open items for the user to answer in one go.
- Never silently resolve an ambiguity yourself; every resolution must trace to an explicit user decision.
- Recommendations are opinions, not defaults — do not apply one until the user accepts, modifies, or replaces it.
- Do not fabricate scenarios that don't plausibly apply to the system described in the spec.
- The output spec must be readable standalone by someone who did not see the brainstorming session.
