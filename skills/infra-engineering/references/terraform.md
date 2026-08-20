# Terraform

## State: the thing that must never be lost or corrupted

State is the single source of truth mapping your config to real resources. Losing it means Terraform no longer knows what it manages; corrupting it means the next plan/apply can do the wrong thing to real infrastructure.

- **Remote backend, always, for anything beyond a solo throwaway experiment.** Local state (`terraform.tfstate` on disk) has no locking and can't be shared — two people applying concurrently corrupt it.
  ```hcl
  terraform {
    backend "s3" {
      bucket         = "company-terraform-state"
      key            = "prod/network/terraform.tfstate"
      region         = "us-east-1"
      dynamodb_table = "terraform-locks"   # state locking — prevents concurrent apply
      encrypt        = true
    }
  }
  ```
  Equivalent locking exists for other backends (Terraform Cloud/Enterprise does it natively; Azure Storage + blob leases; GCS has native locking).
- **Never hand-edit `.tfstate`.** Use `terraform state` subcommands (`mv`, `rm`, `import`) which understand the format and update serials correctly; a manual edit is a common way to silently corrupt state.
- **State contains secrets in plaintext** (e.g. a generated DB password in a resource's attributes) — treat the state backend itself as a secret store: encrypt at rest, restrict access, never commit state to git.
- Split state by blast radius (network / data / app-per-environment, not one giant state for everything) so a bad apply in one area can't touch unrelated resources, and so plans stay fast and reviewable.

## Providers and version pinning

```hcl
terraform {
  required_version = ">= 1.7.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```
Commit `.terraform.lock.hcl` — it pins exact provider versions/checksums so `terraform init` is reproducible across machines and CI. An unpinned or loosely-pinned provider can introduce breaking behavior changes on a routine `init` in a new environment.

## Modules

- A module boundary should match a reusable unit of infrastructure (e.g. "a VPC," "an app service + its autoscaling + its LB"), not just be a folder split for organization's sake.
- Root modules per environment (`environments/prod`, `environments/staging`) that call shared modules with different variables, rather than duplicating resource blocks per environment or — worse — branching logic inside one module based on an environment variable.
- Version-pin remote module sources the same as providers: `source = "git::https://.../vpc.git?ref=v1.4.0"` or a registry version constraint — not `ref=main`, which changes under you.
- Outputs are the module's public interface; keep internal resource details un-exported unless another module genuinely needs them.

## Variables and workspaces

- `variable` blocks with explicit `type` and, for anything sensitive, `sensitive = true` (redacts the value from plan/apply CLI output — does not encrypt it in state).
- `.tfvars` per environment (`prod.tfvars`, `staging.tfvars`), never one file with commented-out blocks toggled by hand before each apply.
- Terraform workspaces (`terraform workspace new/select`) are for minor variations of the *same* config (e.g. ephemeral PR-preview environments) — they share a backend config and can be easy to apply against the wrong one by accident. For environments with meaningfully different topology or blast-radius requirements (prod vs. dev), prefer separate root modules/state files over workspaces — it makes "which environment am I about to touch" an explicit directory, not an easy-to-miss `workspace show`.

## Plan and apply discipline

- `terraform fmt -check` and `terraform validate` before every plan — cheap, catches syntax/type errors before they cost a slow `plan` cycle.
- **Always run and read `terraform plan` before `apply`.** Never `terraform apply -auto-approve` against anything shared/prod outside of a CI pipeline that already required a reviewed plan as a gate.
- Read the plan's action symbols carefully:
  - `+` create, `-` destroy, `~` update in place — all fine in the right context.
  - **`-/+` (destroy and recreate)** — this is the one that causes outages/data loss. It means a change forces replacement (often due to changing an immutable attribute). For a stateful resource (RDS instance, EBS/PV-backed volume, anything holding data), this is a red flag requiring explicit confirmation, not a routine diff to wave through.
- `terraform plan -out=tfplan` then `terraform apply tfplan` in CI so the applied plan is exactly the reviewed plan — no drift between "what was reviewed" and "what ran" from a second `plan` evaluation picking up a concurrent change.
- `terraform apply -target=...` is a scalpel for recovering from a specific problem, not a routine workflow — targeted applies can leave the rest of the config out of sync with state and mask what a full plan would have shown.

## Import and drift

- `terraform import` (or the `import` block in newer versions) to bring existing infrastructure under management — write the matching resource config first, then import, then plan to confirm zero diff.
- Drift (real infrastructure changed outside Terraform — console click, another tool) shows up as an unexpected diff on next plan. Investigate before applying "to fix it" — the plan might be about to revert someone's intentional emergency change.
- `terraform plan -refresh-only` to see drift without proposing changes to reconcile it; decide deliberately whether to accept the drift into state (`apply -refresh-only`) or revert it back to config.

## Common failure modes

- **"resource already exists" on apply**: something was created outside Terraform (console, another apply, a different state file) with the same identity. Import it or rename to avoid the collision — don't force-create over it.
- **State lock stuck ("Error acquiring the state lock")**: a previous apply crashed/was killed mid-run and didn't release the DynamoDB/backend lock. Confirm no apply is actually still running before `terraform force-unlock <lock-id>` — force-unlocking while something is genuinely running is how state gets corrupted.
- **Plan shows changes on every run with no config change**: provider-level drift-detection quirk (some attributes normalize differently than submitted) or a data source whose upstream value changes each run (e.g. an AMI lookup using `most_recent = true`) — pin the value instead of re-resolving it every plan if stability matters more than always-latest.
- **Circular dependency between modules/resources**: usually a sign two resources should be combined, or an implicit dependency needs to be made explicit via `depends_on` in a direction that breaks the cycle.
- **Sensitive values leaking anyway**: `sensitive = true` hides a value in CLI plan/apply output, but it's still in `terraform show -json`, state, and any output consumed by another tool — don't treat it as encryption.
