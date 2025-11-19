# Slurm HPC Ansible Playbooks

以最少步驟完成 Slurm 叢集部署：只要設定 inventory、建立一個 Vault、照著工作流程依序執行 playbook，即可完成 bootstrap → cluster → infra 的自動化。  
Automate the entire lifecycle (prepare → cluster → infra) with the same `--vault-password-file`.

---

## Workflow 一覽

> 全流程共用參數：`--vault-password-file ./inventory/vault/.ansible_vault_pass`

1. **(可選) 連線檢查**
   ```bash
   ansible-playbook -i inventory/production.ini playbooks/prepare/preflight.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
2. **Bootstrap 管理帳號 + SSH 金鑰**
   ```bash
   ansible-playbook -i inventory/production.ini -i inventory/prepare.ini \
     playbooks/prepare/bootstrap_prepare.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
3. **Post-bootstrap 驗證（sudo、.ssh、無密碼 SSH）**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/prepare/post_bootstrap_verify.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
4. **Cluster：/etc/hosts、NFS client、MUNGE、Slurm**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/cluster/setup_cluster.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
5. **Infra：NFS exports、slurmdbd、login node**
   ```bash
   ansible-playbook -i inventory/production.ini \
     playbooks/infra/setup_infra.yml \
     --vault-password-file ./inventory/vault/.ansible_vault_pass
   ```
6. **Optional**  
   - `playbooks/cluster/enable_nvidia_gpu.yml`  
   - Teardown flows under `playbooks/teardown/`

---

## Vault 只要這一個指令

專案使用自動載入的 `inventory/group_vars/all/vault.yml`。把 Vault 密碼放在 `inventory/vault/.ansible_vault_pass` 後，直接執行：

```bash
ansible-vault create inventory/group_vars/all/vault.yml \
  --vault-password-file ./inventory/vault/.ansible_vault_pass
```

在編輯器中填入常用機敏變數即可，例如：

```yaml
ansible_password: "ssh-password"
ansible_become_password: "sudo-password"
vault_slurmdbd_db_password: "db-password"
```

之後任何 playbook 僅需附帶同一個 `--vault-password-file`，就可一次取得 SSH / sudo / Slurm DB 密碼，無需再輸入 `--ask-pass --ask-become-pass`。

---

## 必要檔案與變數

| 檔案 | 內容 |
| --- | --- |
| `inventory/production.ini` | controller / compute / infra 實際主機與 IP |
| `inventory/prepare.ini` | bootstrap 前控制端用來登入的帳號設定 |
| `inventory/group_vars/all/main.yml` | `mgmt_user`、NFS 匯出、Slurm UID/GID 等全域設定 |
| `inventory/group_vars/controller.yml` / `compute.yml` | 節點角色、NFS 掛載覆寫 |
| `inventory/group_vars/infra.yml` | infra 節點的 NFS exports、slurmdbd、login node 套件 |
| `inventory/group_vars/all/vault.yml` | Vault 加密檔（上節指令建立） |

建議在修改完 inventory 與變數後立即建立/更新 Vault，確保流程中所有 playbook 都能讀到一致的憑證。

---

## 目錄快速看

```
inventory/
├── production.ini
├── prepare.ini
├── group_vars/
│   ├── all/
│   │   ├── main.yml
│   │   └── vault.yml        # ansible-vault create … 產生
│   ├── controller.yml
│   ├── compute.yml
│   └── infra.yml
└── vault/
    └── .ansible_vault_pass  # 自備密碼檔（也可改用 --ask-vault-pass）
playbooks/
├── prepare/  (preflight, bootstrap, post verify)
├── cluster/  (setup_cluster, enable_nvidia_gpu)
├── infra/    (setup_infra)
└── teardown/
roles/…        # munge, slurm, nfs, etc.
```

照著「Vault → Workflow」兩節操作，就能快速完成 Slurm 叢集的部署與驗證。Enjoy!

