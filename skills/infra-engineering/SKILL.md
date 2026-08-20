---
name: infra-engineering
description: Design, write, and review infrastructure-as-code and container orchestration for Docker, Docker Compose, Kubernetes, Terraform, and Ansible. Use when the task involves a Dockerfile, docker-compose.yml, Kubernetes manifests or Helm charts, Terraform (.tf/.tfvars/state/modules), Ansible playbooks/roles/inventory, or general questions about containerizing an app, orchestrating services, provisioning cloud infrastructure, or automating server configuration.
---

# Infra Engineering

You are executing the `infra-engineering` skill. Read this file fully, then load only the reference files relevant to the task.

## Core stance

Infrastructure changes are **production changes with blast radius beyond the current commit**: a bad Terraform apply can delete a database, a bad Kubernetes rollout can take down every replica at once, a bad Ansible playbook can misconfigure a whole fleet in one run. Treat every change here with the same care as a database migration.

Non-negotiables:

1. **Never run an apply/rollout/playbook against production without a dry-run or plan step first**, and without showing the user what it will do. `terraform plan`, `kubectl diff`, `--check --diff` (Ansible), `docker compose config` are not optional.
2. **Never hardcode secrets, credentials, tokens, or keys into a file that gets committed.** Use secret managers, env injection, or encrypted stores (Vault, SOPS, sealed-secrets, `ansible-vault`) — see the relevant reference for the mechanism.
3. **State and inventory are sources of truth.** Never hand-edit Terraform state, never assume Ansible inventory is current, never `kubectl edit` a resource that's managed by GitOps/Helm/Terraform without expecting it to be reverted on next sync.

## Step 1 — Establish the environment before writing anything

Do not guess the tool version, target platform, or existing conventions. Determine:

| Question | How to find out |
|---|---|
| Tool versions | `docker --version`, `docker compose version`, `kubectl version`, `terraform version`, `ansible --version`. Version skew (e.g. Compose v1 vs v2 syntax, Terraform 0.12 vs 1.x, K8s API deprecations) changes what's valid. |
| Existing conventions | Look for `Dockerfile`, `docker-compose*.yml`, `k8s/`, `helm/`, `charts/`, `*.tf`, `terraform.tfvars`, `ansible.cfg`, `inventory/`, `roles/`, `playbooks/` before introducing a new pattern. |
| Target environment | Local dev, CI, staging, prod — each may have different manifests, tfvars, or inventory groups. Never apply a dev config assumption to prod. |
| Who/what owns state | Terraform backend (S3+DynamoDB, Terraform Cloud, etc.), Helm release history, GitOps controller (ArgoCD/Flux) — changes made outside that system get silently reverted or cause drift. |
| Cloud/cluster context | `kubectl config current-context`, `aws sts get-caller-identity` / equivalent, `terraform workspace show`. Confirm you're pointed at the environment you think you are before any mutating command. |

If the environment is genuinely unknowable and the answer differs by version/platform, say so and give the answer for the most likely one rather than blocking.

## Step 2 — Route to the right reference

Load only what you need:

- **Dockerfiles, image builds, layer caching, multi-stage builds, image size/security** → `references/docker.md`
- **docker-compose.yml, multi-container local/dev stacks, networking, volumes, profiles** → `references/docker-compose.md`
- **Kubernetes manifests, Helm charts, workloads, networking, RBAC, autoscaling, troubleshooting** → `references/kubernetes.md`
- **Terraform modules, state, providers, workspaces, plan/apply safety** → `references/terraform.md`
- **Ansible playbooks, roles, inventory, idempotency, vault** → `references/ansible.md`

Cross-cutting tasks (e.g. "containerize this app and deploy it to k8s via Terraform") span references — read all that apply, in the order the pipeline runs (Docker → Compose/K8s → Terraform → Ansible).

## Step 3 — Design defaults

Apply unless the task gives a reason not to.

**Least privilege everywhere.** Containers run as non-root. Kubernetes service accounts get scoped RBAC, not `cluster-admin`. Terraform-provisioned IAM roles get the narrowest policy that works. Ansible connects with a dedicated, key-based, minimally-privileged user, escalating with `become` only for the tasks that need it.

**Idempotency and reproducibility.** Pin versions: base images by digest or exact tag (not `latest`), Terraform provider versions in a lock file, Ansible collection/role versions, Helm chart versions. A playbook or apply run twice should converge to the same state, not double-apply.

**Everything as code, reviewed like code.** No manual `kubectl create`, `docker run` in prod, or console clicks for anything that should be reproducible. If a manual step was necessary, capture it back into IaC afterward or flag the drift.

**Secrets never touch plaintext files in the repo.** Kubernetes: sealed-secrets/External Secrets Operator/SOPS, not raw `Secret` manifests with base64 "encoded" values checked in (base64 is not encryption). Terraform: `sensitive = true` on variables/outputs, secrets sourced from a vault/parameter store, never inlined in `.tf`/`.tfvars` that get committed. Ansible: `ansible-vault` for anything sensitive in inventory or `group_vars`/`host_vars`.

**Small blast radius.** Prefer changes that can be rolled out gradually and rolled back cheaply: rolling updates over recreate, canary/blue-green where it matters, `terraform plan` reviewed before every apply, Ansible runs against a `--limit`-scoped subset before the full fleet.

## Step 4 — Writing the change

1. State what the change will do before running anything mutating — in plain language, not just "here's the diff."
2. For Terraform: run `terraform fmt`, `terraform validate`, and `terraform plan`, and show the plan output. Flag any `-/+` (destroy-and-recreate) explicitly — that's data loss for stateful resources.
3. For Kubernetes: prefer `kubectl diff` / `helm diff` / a GitOps PR over direct `apply` against a live cluster when the tooling exists. Note the rollout strategy (`RollingUpdate` params, `maxUnavailable`/`maxSurge`) for anything with >1 replica.
4. For Ansible: run with `--check --diff` first against a real or representative host, and call out any task that isn't idempotent (i.e., would show "changed" on a second identical run).
5. For Docker/Compose: build locally and confirm the container starts and passes its healthcheck before treating the change as done.
6. One logical change per commit/PR — don't bundle a provider upgrade with a resource change with a secret rotation.

## Step 5 — Report honestly

When you finish, state:

- What the plan/diff/dry-run actually showed, not what you expect it to show.
- Whether the change is backward compatible with what's currently running (in-place rollout vs. requires downtime/recreate).
- The rollback procedure for this specific change, and whether it's lossless.
- What you verified by running a tool versus what you inferred by reading files. Do not present an inference as a verified result.

If you couldn't verify something — no cluster/cloud access, no way to run `plan`/`--check`, unknown current state — say which check is outstanding and what the user should run.

## Red flags — stop and raise these

- Any `terraform apply` targeting prod state/workspace without a reviewed plan shown first
- A Terraform plan showing destroy-and-recreate (`-/+`) on a stateful resource (database, PVC-backed volume, anything holding data)
- `terraform state rm`/`mv`/hand-editing state without understanding exactly why the state and reality diverged
- Kubernetes `Secret` manifests with real values committed to a repo, even base64-encoded
- A container image built and run as root with no `USER` directive, when the app doesn't need root
- `docker-compose.yml` or K8s manifests exposing a database or admin port directly to the internet (`0.0.0.0` bind, public `LoadBalancer`/`NodePort` with no network policy)
- An Ansible playbook run against `all` / production inventory without first running `--check` or against a `--limit`-scoped test group
- `ansible-vault` encrypted files being decrypted and written to disk in plaintext, or vault passwords committed alongside encrypted content
- Any `kubectl delete namespace`, `terraform destroy`, or Ansible task with `state: absent` targeting a scope wider than explicitly requested
- Pinning nothing: `latest` image tags, unconstrained provider version blocks, or role/collection installs without a version pin in a change meant to be reproducible
