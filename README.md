# Slurm HPC Ansible Playbooks · Bilingual Guide

Automate the full Slurm lifecycle (bootstrap → infra → cluster → GPU/teardown) with a single Vault-protected workflow.  
這份專案提供完整的 Slurm 叢集自動化流程；只要設定好 inventory / 變數 / Vault，即可用一個 `--vault-password-file` 跑完整個流程。

---

## 1. Setup / 事前準備

1. **Install Ansible 2.15+ on the control node**  
   控制端需安裝 Ansible 2.15 以上版本。
2. **Prepare inventory files**  
   - `inventory/production.ini`: controller / compute / infra hosts & IPs.  
     `inventory/production.ini`：填入實際主機與 IP。  
   - `inventory/prepare.ini`: pre-bootstrap login overrides (default user `tommy`).  
     `inventory/prepare.ini`：bootstrap 前用來登入的帳號，如需不同帳號請在此覆寫。  
3. **Adjust group vars as needed**  
   - `inventory/group_vars/all/main.yml`: mgmt_user、NFS、Slurm UID/GID…  
   - `inventory/group_vars/controller.yml` / `compute.yml`: Slurm 角色與 NFS 掛載。  
   - `inventory/group_vars/infra.yml`: infra exports、slurmdbd 設定、login node 套件。  
4. **Create the auto-loaded Vault** (`inventory/group_vars/all/vault.yml`)  
   Encrypt it so Ansible can automatically load SSH/sudo/DB secrets:
   ```yaml
   ansible_password: "!QAZ2wsx"          # SSH 密碼
   ansible_become_password: "!QAZ2wsx"  # sudo 密碼
   vault_slurmdbd_db_password: "slurmdbdpass"
   ```
   Example creation (assuming you store the vault pass in `./inventory/vault/.ansible_vault_pass`):
   ```bash
   cat > /tmp/all_vault_plain.yml <<'EOF'
   ansible_password: "!QAZ2wsx"
   ansible_become_password: "!QAZ2wsx"
   vault_slurmdbd_db_password: "slurmdbdpass"
   EOF
   ansible-vault encrypt /tmp/all_vault_plain.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass \
     --output inventory/group_vars/all/vault.yml
   ```
   之後所有 playbook 只需附帶 `--vault-password-file ./inventory/vault/.ansible_vault_pass`（或 `--ask-vault-pass`），完全不需要再輸入 `--ask-pass --ask-become-pass`。

---

## 2. Execution Flow / 執行流程

> 下列指令全部使用同一個參數：  
> `--vault-password-file ./inventory/vault/.ansible_vault_pass`

1. **Bootstrap 管理帳號 / controller→compute 無密鑰登入**
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini \
     playbooks/prepare/bootstrap_prepare.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
2. **Post-bootstrap 驗證（sudoers / SSH / known_hosts）**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/prepare/post_bootstrap_verify.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
3. **Infra：NFS exports·slurmdbd·login node**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/infra/setup_infra.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
4. **Cluster：更新 hosts / NFS / MUNGE / Slurm + 跨節點驗證**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/cluster/setup_cluster.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
5. **(可選) 启用 NVIDIA GPU**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/cluster/enable_nvidia_gpu.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
6. **Teardown（在 `playbooks/teardown/`）**  
   GPU / Slurm stack / bootstrap user 等拆除流程同樣只需附上 Vault 密碼檔。

> 若想先檢查 inventory/變數/連線是否 OK，可在步驟 1 前執行 `playbooks/prepare/preflight.yml` 搭配同一個 vault pass。

---

## 3. Directory Overview / 目錄結構

```
inventory/
├── production.ini             # 實際 controller / compute / infra 主機
├── prepare.ini                # pre-bootstrap 登入帳號覆寫
├── group_vars/
│   ├── all/
│   │   ├── main.yml           # 共用變數
│   │   └── vault.yml          # 加密檔（ansible_password / ansible_become_password / vault_slurmdbd_db_password）
│   ├── controller.yml
│   ├── compute.yml
│   └── infra.yml
└── vault/
    └── .ansible_vault_pass    # Vault 密碼檔（範例，可自備）
```

---

## 4. Checklist / 檢查清單

1. `inventory/production.ini`、`inventory/prepare.ini` 是否填好？  
2. `inventory/group_vars/all/main.yml` / `infra.yml` 是否符合環境（mgmt_user、NFS、Slurm UID/GID 等）？  
3. `inventory/group_vars/all/vault.yml` 是否已加密且包含三個鍵（SSH、sudo、slurmdbd 密碼）？  
4. 執行指令是否使用 `--vault-password-file ./inventory/vault/.ansible_vault_pass`（或 `--ask-vault-pass`）？

只要以上都 OK，就能依照第 2 節指令完成整個 Slurm 叢集部署與維護。Enjoy your cluster!  
只要以上條件符合，照著流程執行即可順利完成部署。祝操作順利！
