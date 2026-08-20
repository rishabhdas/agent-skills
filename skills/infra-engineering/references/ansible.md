# Ansible

## Idempotency — the core discipline

A playbook run twice against the same host should report the second run as unchanged (or as close to it as the module allows). This is what makes Ansible safe to re-run rather than a one-shot script.

- Prefer modules over `command`/`shell`. `apt`, `yum`, `copy`, `template`, `file`, `user`, `service`, etc. are all idempotent by design — they check current state before acting. `command`/`shell` run unconditionally every time and report `changed` regardless of whether anything actually changed, unless you constrain them.
- When `command`/`shell` is genuinely necessary (no module covers the case), make it idempotent explicitly:
  ```yaml
  - name: Run migration if not already applied
    command: ./migrate.sh
    args:
      creates: /app/.migrated   # skips the task if this path already exists
  ```
  or use `changed_when`/`failed_when` to make the reported status reflect reality rather than "ran a shell command" reflecting "changed."
- `register` + `changed_when: result.rc != 0` (or similar) to make custom logic report status honestly — a task that always says `changed` when it didn't erodes trust in `--check` output and dry-run diffs.

## Inventory

- Static (`inventory.ini`/`inventory.yaml`) for small/stable fleets; dynamic inventory (cloud plugin: `aws_ec2`, `azure_rm`, `gcp_compute`) for anything that autoscales or where hosts churn — a static inventory silently drifts from reality as instances come and go.
- Group by role and by environment, and let host/group variables do the differentiation rather than conditionals inside tasks:
  ```ini
  [webservers]
  web1.prod.example.com
  web2.prod.example.com

  [webservers:vars]
  http_port=443

  [prod:children]
  webservers
  dbservers
  ```
- `group_vars/<group>.yml` and `host_vars/<host>.yml` for variables scoped to that group/host — keeps `inventory` itself lean and variables discoverable in one place per scope.

## Playbook and role structure

Use the standard role layout (`ansible-galaxy init <role>` scaffolds it) rather than one flat playbook once a project grows past a handful of tasks:
```
roles/
  webserver/
    tasks/main.yml
    handlers/main.yml
    templates/
    files/
    defaults/main.yml   # lowest-precedence variable defaults, safe to override
    vars/main.yml       # role-internal constants, not meant to be overridden casually
    meta/main.yml       # role dependencies
```
- `defaults/` vs `vars/`: `defaults` is where "configurable knobs a consumer of this role should be able to override" belongs; `vars` is for values the role's own logic depends on and shouldn't casually change.
- Tags on tasks/roles (`tags: [nginx, config]`) so a full run isn't required for every change: `ansible-playbook site.yml --tags nginx`.

## Handlers and notify

```yaml
- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

handlers:
  - name: Restart nginx
    service: { name: nginx, state: restarted }
```
Handlers run once at the end of the play (even if multiple tasks notify the same handler), and only if the notifying task reported `changed`. This is how you avoid restarting a service on every run — only when its config actually changed. Don't put a `service: restarted` directly in the task list for something config-driven; that restarts it unconditionally on every playbook run regardless of whether anything changed.

## Vault — secrets

```bash
ansible-vault create group_vars/prod/vault.yml
ansible-vault encrypt_string 'supersecret' --name 'db_password'
ansible-playbook site.yml --ask-vault-pass
# or non-interactively:
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
```
- Never commit the vault password itself, and never commit a decrypted version of a vaulted file alongside its encrypted counterpart.
- Common pattern: `group_vars/prod/vars.yml` (plaintext, references variable names) + `group_vars/prod/vault.yml` (encrypted, holds actual secret values) with `vars.yml` doing `db_password: "{{ vault_db_password }}"` — keeps the plaintext file readable/diffable in PRs while the values stay encrypted.
- Rotate the vault password by re-encrypting (`ansible-vault rekey`), not by leaving old encrypted files under a retired password indefinitely.

## Privilege escalation

```yaml
- hosts: webservers
  become: true
  become_user: root
  tasks:
    - name: Only this task needs root
      become: true
      ...
```
Scope `become: true` to the specific tasks that need it rather than the whole play when only a subset of tasks are actually privileged — reduces what runs as root and makes `--check` output clearer about what's actually privileged.

## `--check` and `--diff`

- `ansible-playbook site.yml --check` — dry run; reports what *would* change without changing it. Not all modules support check mode fully (some `command`/`shell` tasks, some custom modules) — those either skip or run for real even under `--check` unless explicitly marked `check_mode: false`/handled; know which tasks in your playbook fall into that gap before trusting a `--check` run completely.
- `--diff` alongside `--check` shows the actual before/after content diff for file-based changes (`template`, `copy`, `lineinfile`) — read this, not just the changed/ok/failed counts.
- Always run `--check --diff` against production (or a representative host) before a real run when the playbook is new or has changed meaningfully.

## Limiting blast radius

- `--limit <group_or_host>` to scope a run — test against one host or a canary group before the full inventory:
  ```bash
  ansible-playbook site.yml --limit web1.prod.example.com --check --diff
  ansible-playbook site.yml --limit web1.prod.example.com
  # then, once confirmed:
  ansible-playbook site.yml --limit webservers
  ```
- `serial:` in the play to roll out across a fleet in batches rather than all hosts at once — critical for anything that restarts a service, since an unbatched run against all webservers simultaneously means simultaneous downtime across the fleet:
  ```yaml
  - hosts: webservers
    serial: "25%"
    max_fail_percentage: 10   # abort the rollout if too many hosts in a batch fail
  ```

## Common failure modes

- **Task reports `changed` every run**: a `command`/`shell` task without `creates`/`changed_when`, or a module being fed a value that's non-deterministic each run (e.g. a timestamp baked into a templated file).
- **"unreachable" host**: SSH connectivity/auth issue, not a playbook logic issue — check `ansible <host> -m ping` in isolation before assuming the playbook is at fault.
- **Variable precedence surprises**: Ansible has ~20 levels of variable precedence (role defaults lowest, `-e` extra-vars on the CLI highest). When a variable isn't taking the value you expect, `ansible-inventory --host <host> --vars` or `ansible-playbook ... -vvv` to see resolved values, rather than guessing which layer is winning.
- **Handler didn't fire**: handlers run at the *end of the play* by default, and only once even if notified multiple times — if a later task in the same play needs the restarted service to already be running, either reorder or force early execution with `meta: flush_handlers`.
- **Fact gathering slow on large inventories**: `gather_facts: false` plus explicit `setup:` calls only where needed, or cache facts (`fact_caching = jsonfile` in `ansible.cfg`) so repeated runs don't re-gather from every host every time.
