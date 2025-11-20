# Repository Guidelines

## Project Structure & Module Organization
Inventory files live under `inventory/`, with `production.ini` for real nodes, `prepare.ini` for bootstrap access, and `group_vars/` for shared variables or role-specific overrides. Playbooks are grouped by lifecycle in `playbooks/prepare`, `playbooks/cluster`, `playbooks/infra`, and `playbooks/teardown`. Roles (NFS, Slurm, Munge, etc.) sit in `roles/`, and `ansible.cfg` already points `roles_path` there. Keep new assets (templates, handlers, files) inside the matching role to retain idempotent boundaries.

## Build, Test, and Development Commands
Run workflows with `ansible-playbook -i inventory/production.ini playbooks/cluster/setup_cluster.yml --vault-password-file ./inventory/vault/.ansible_vault_pass`. Bootstrap accounts via `playbooks/prepare/bootstrap_prepare.yml`, then verify with `playbooks/prepare/post_bootstrap_verify.yml`. Infrastructure extras (NFS exports, slurmdbd, login node) come from `playbooks/infra/setup_infra.yml`. Use `playbooks/teardown/*` when resetting a lab cluster, keeping the same inventory and Vault file.

## Coding Style & Naming Conventions
Stick to YAML using two-space indentation and lowercase keys; favor descriptive variable names such as `vault_slurmdbd_db_password`. Modules should be idempotent and aligned with existing role naming (verbs for plays, nouns for roles). Variables shared fleet-wide belong in `inventory/group_vars/all/main.yml`, while host-class overrides go in controller/compute/infra files. Use `_path` for filesystem strings and `_hosts` for host lists to keep inventory self-documenting.

## Testing Guidelines
Before committing, run `ansible-playbook <playbook>.yml --syntax-check` and `ansible-playbook -i inventory/production.ini playbooks/prepare/preflight.yml --check` to catch host connectivity issues. Lint role changes with `ansible-lint playbooks` if available in your environment; otherwise ensure tasks include `become`, `tags`, and proper `when` guards. Document any manual test (e.g., GPU enablement) in the PR description so others can replay it.

## Commit & Pull Request Guidelines
Commits in this repo follow short imperative prefixes such as `docs: refresh workflow` or `refactor: consolidate teardown plays`; continue that style and keep messages under 72 characters. Every PR should describe the targeted lifecycle stage (prepare/cluster/infra), list tested commands, and link related inventory changes. Attach redacted command output or screenshots when altering user-facing documentation so reviewers can validate the exact flow.

## Security & Configuration Tips
Store the Vault password at `inventory/vault/.ansible_vault_pass`, keep it out of version control, and use the same file across all commands to avoid leaking `--ask-pass`. Sensitive defaults belong only in `inventory/group_vars/all/vault.yml`; never commit decrypted copies. When editing remote-user or sudo settings, verify that `ansible_become` paths remain consistent and avoid embedding organization-specific identifiers in shared playbooks.
