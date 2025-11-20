# Slurm HPC Cluster with NVIDIA MIG Support

This Ansible project automates the deployment of a Slurm HPC cluster, featuring advanced NVIDIA GPU configuration with MIG (Multi-Instance GPU) support and persistence.

## Project Structure

```
.
├── inventory/
│   ├── production.ini       # Main inventory file
│   ├── host_vars/           # Host-specific configurations (e.g., MIG profiles)
│   ├── group_vars/          # Group-level variables
│   └── vault/               # Encrypted secrets
├── playbooks/
│   ├── prepare/             # Bootstrap and preflight checks
│   ├── infra/               # Infrastructure (NFS, DB, MIG)
│   ├── cluster/             # Slurm cluster deployment
│   └── teardown/            # Cleanup scripts
└── roles/                   # Ansible roles
```

## Prerequisites

*   Ansible installed on the control node.
*   Target nodes accessible via SSH.
*   `inventory/vault/.ansible_vault_pass` file containing the vault password.

## Configuration

### 1. Inventory
Define your hosts in `inventory/production.ini`:
*   `[controller]`: Slurm controller node.
*   `[compute]`: Compute nodes with GPUs.
*   `[infra]`: Infrastructure node (NFS, Database).

### 2. MIG Configuration (Per-Host)
Configure MIG profiles in `inventory/host_vars/<hostname>.yml`.
Example (`inventory/host_vars/slurm.yml`):

```yaml
---
slurm_gres_name: "rtx"  # Resource name in Slurm (e.g., gres=rtx:1)

nvidia_mig_config:
  - gpu_index: 0
    enable_mig: true
    profiles:
      - "2g.48gb"
      - "2g.48gb"
  - gpu_index: 1
    enable_mig: true
    profiles:
      - "2g.48gb"
      - "2g.48gb"
```

## Deployment Workflow

Run the playbooks in the following order:

### Step 1: Bootstrap Nodes
Initialize user accounts and SSH access.
```bash
ansible-playbook -i inventory/prepare.ini playbooks/prepare/bootstrap_prepare.yml --ask-pass --ask-become-pass
```

### Step 2: Setup Infrastructure
Deploy NFS Server and Slurm Database (SlurmDBD).
```bash
ansible-playbook -i inventory/production.ini playbooks/infra/setup_infra.yml --vault-password-file ./inventory/vault/.ansible_vault_pass
```

### Step 3: Enable NVIDIA GPUs
Install NVIDIA drivers, CUDA, and Fabric Manager on compute nodes.
```bash
ansible-playbook -i inventory/production.ini playbooks/cluster/enable_nvidia_gpu.yml --vault-password-file ./inventory/vault/.ansible_vault_pass
```

### Step 4: Configure MIG
Apply MIG partitioning and enable persistence mode.
```bash
ansible-playbook -i inventory/production.ini playbooks/infra/setup_mig.yml --vault-password-file ./inventory/vault/.ansible_vault_pass
```

### Step 5: Deploy Slurm Cluster
Install Slurm Controller and Compute daemons, mount NFS, and configure Munge.
*Note: This playbook automatically detects GPU/MIG resources.*
```bash
ansible-playbook -i inventory/production.ini playbooks/cluster/setup_cluster.yml --vault-password-file ./inventory/vault/.ansible_vault_pass
```

## Verification

Log in to the controller node and check the cluster status:

```bash
sinfo -N -o "%N %G %t"
# Expected output: slurm rtx:4 idle
```

Submit a test job:
```bash
srun --gres=rtx:1 --pty /bin/bash
nvidia-smi -L
```

## Maintenance

*   **Teardown**: Use playbooks in `playbooks/teardown/` to remove components.
*   **Update MIG**: Edit `host_vars`, then run `setup_mig.yml` followed by `setup_cluster.yml`.
