# Slurm HPC Ansible Playbooks

This repository contains the end-to-end automation used to bootstrap, configure, and maintain a small Slurm cluster (controller + compute nodes). It is optimized for Ubuntu-based nodes with NVIDIA GPUs, but most steps work on any Debian/Ubuntu derivative once the prerequisites are met.

## Prerequisites

- Ansible 2.15+ installed on the control machine.
- SSH access from the control machine to every node listed in `inventory/production.ini`.
- SSH key available locally (defaults to `~/.ssh/id_rsa`). Override via `inventory/group_vars/all.yml` if you use a different key.
- Python + sudo available on the managed nodes (the bootstrap stage will create the management user and sudoers drop-in).

## Inventories and Variables

- `inventory/production.ini` – canonical list of controller and compute nodes plus their IPs.
- `inventory/prepare.ini` – optional overlay that only sets per-host `ansible_user` / SSH options for the pre-bootstrap phase. Use it when the target machines still run with OS-default accounts (e.g., `ubuntu`, `debian`, etc.).
- `inventory/group_vars/all.yml` – shared defaults such as `mgmt_user`, SSH key path, etc.
- `inventory/group_vars/controller.yml` / `compute.yml` – role hints (e.g., `slurm_role`).

When running playbooks that require the temporary overlay (bootstrap/teardown), pass both inventories **in that order** so `prepare.ini` overrides per-host variables from `production.ini`:

```bash
ansible-playbook -i inventory/production.ini -i inventory/prepare.ini playbooks/prepare/bootstrap_prepare.yml
```

`inventory/prepare.ini` is typically used just to ensure SSH connectivity before the management account exists. You **must** specify an account that already exists on each node (the OS default user, root, etc.) so Ansible can log in. The simplest approach is to redeclare the hosts there with the desired `ansible_user`, e.g.:

```
[controller]
slurmctld ansible_user=tommy

[compute]
slurm ansible_user=tommy
```

Only when a node needs a different login should you adjust its entry. For all other playbooks, just use `inventory/production.ini`.

The management account (`mgmt_user`) that gets created by the bootstrap flow lives in `inventory/group_vars/all.yml`. Modify it there when you want to switch to a different cluster-wide admin user; every playbook and role reads from that single source of truth, so you don’t need to tweak inventory files or CLI flags.

## Typical Workflow

1. **Bootstrap management user** (creates `mgmt_user`, installs keys, primes controller known_hosts):
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini playbooks/prepare/bootstrap_prepare.yml
   ```
2. **Post-bootstrap verification** (sudoers, SSH, controller→compute reachability):
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/prepare/post_bootstrap_verify.yml
   ```
3. **Configure Slurm cluster** (MUNGE distribution, Slurm config, service restarts, controller validation):
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/cluster/setup_cluster.yml
   ```
4. **(Optional) Enable NVIDIA GPUs** on compute nodes (after Slurm is installed so `scontrol` exists on the controller):
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/cluster/enable_nvidia_gpu.yml
   ```
   Hosts without NVIDIA hardware are skipped automatically.
5. **Teardown management user** (only when you want to remove the bootstrap user):
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini playbooks/teardown/bootstrap_user.yml
   ```

## Teardown Playbooks

All teardown entrypoints now live under `playbooks/teardown`:

1. **GPU teardown (optional)** – purge NVIDIA drivers, remove the nouveau blacklist, and reboot if needed:
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/teardown/nvidia_gpu.yml
   ```
   Use `-e reboot_after=true` if you want to force a reboot even when nothing changed.
2. **Slurm/MUNGE stack teardown** – stop services, purge packages, delete config/state, and clean `/etc/hosts`:
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/teardown/cluster_stack.yml
   ```
   Toggle `-e purge_stack=false` or `-e wipe_system_users=false` if you want to keep packages/users.
3. **Bootstrap user teardown** – remove controller known_hosts entries plus the `mgmt_user` account:
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini playbooks/teardown/bootstrap_user.yml
   ```
   Run this as the pre-bootstrap account specified in `inventory/prepare.ini`, and supply its SSH / sudo password via `--ask-pass --ask-become-pass` if necessary.

## Notes and Tips

- If you hit `Permission denied` errors related to Ansible temp dirs (typical in sandboxed environments), set `ANSIBLE_LOCAL_TEMP=$PWD/.ansible/tmp` before running the playbooks.
- GPU scheduling currently only supports NVIDIA cards (uses `nvidia-smi` and `/dev/nvidia*`). AMD/other GPUs will just be treated as CPU-only nodes unless additional roles/templates are added.
- Always update `inventory/production.ini` when nodes or IPs change; `prepare.ini` should only override per-host SSH usernames for the bootstrap stage.

Feel free to fork/extend these playbooks for more complex clusters—just keep the role-oriented structure intact so new functionality is easy to reuse.***
