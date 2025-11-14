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

You can override usernames per host inside `inventory/prepare.ini` by uncommenting entries under `[controller]` or `[compute]`, e.g. `slurmctld ansible_user=ubuntu`. Hosts left commented will fall back to the `[all:vars]` default. For all other playbooks, just use `inventory/production.ini`.

## Typical Workflow

1. **Bootstrap management user** (creates `mgmt_user`, installs keys, primes controller known_hosts):
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini playbooks/prepare/bootstrap_prepare.yml
   ```
2. **Post-bootstrap verification** (sudoers, SSH, controller→compute reachability):
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/prepare/post_bootstrap_verify.yml
   ```
3. **(Optional) Enable NVIDIA GPUs** on compute nodes:
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/cluster/enable_nvidia_gpu.yml
   ```
   Hosts without NVIDIA hardware are skipped automatically.
4. **Configure Slurm cluster** (MUNGE distribution, Slurm config, service restarts, controller validation):
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/cluster/setup_cluster.yml
   ```
5. **Teardown management user** (only when you want to remove the bootstrap user):
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini playbooks/prepare/teardown_prepare.yml
   ```

## Notes and Tips

- If you hit `Permission denied` errors related to Ansible temp dirs (typical in sandboxed environments), set `ANSIBLE_LOCAL_TEMP=$PWD/.ansible/tmp` before running the playbooks.
- GPU scheduling currently only supports NVIDIA cards (uses `nvidia-smi` and `/dev/nvidia*`). AMD/other GPUs will just be treated as CPU-only nodes unless additional roles/templates are added.
- Always update `inventory/production.ini` when nodes or IPs change; `prepare.ini` should only override per-host SSH usernames for the bootstrap stage.

Feel free to fork/extend these playbooks for more complex clusters—just keep the role-oriented structure intact so new functionality is easy to reuse.***
