# Oracle 19c Data Guard Automation with Ansible

Ansible playbooks for fully automated deployment and zero-downtime patching of an Oracle 19c Data Guard environment (Physical Standby) on Oracle Linux 8.

> **Learning project** — built and tested in a VirtualBox lab environment. Designed to be reusable for other Data Guard setups by simply swapping the inventory file.

---

## What This Does

| Area | Description |
|------|-------------|
| OS Preparation | Kernel parameters, resource limits, ASM disk setup, users/groups, firewall |
| Grid Infrastructure | Oracle 19c GI silent install, ASM diskgroups (+DATA, +FRA), Listener |
| Database Software | Oracle 19c DB silent install (SWONLY) |
| Primary Database | DBCA, Data Guard parameters, TNS, password file |
| Standby Database | RMAN active duplicate over network, MRP apply |
| Data Guard Broker | Broker configuration, switchover validation |
| RU Patching | Zero-downtime patching via switchover (19.3 → 19.27), rollback included |

---

## Environment

| Component | Details |
|-----------|---------|
| Primary Server | dghost-001 |
| Standby Server | dghost-002 |
| Ansible Controller | anshost (AWX) |
| OS | Oracle Linux 8.10 |
| Oracle Version | 19c EE (19.27.0.0.0) |
| Storage | ASM: +DATA (500G), +FRA (100G) |
| Data Guard Type | Physical Standby (MaxPerformance) |
| RAM | 16 GB per server |

---

## Project Structure

```
├── ansible.cfg
├── inventory/
│   └── dataguard-001-002        # All hosts + variables (use as template)
├── playbooks/
│   ├── dataguard/               # Installation playbooks (run in order)
│   │   ├── 01_os_preparation.yml
│   │   ├── 02_grid_install.yml
│   │   ├── 02_grid_deinstall.yml
│   │   ├── 03_db_install.yml
│   │   ├── 04_create_primary.yml
│   │   ├── 05_create_standby.yml
│   │   └── 06_dg_broker.yml
│   └── patching/                # Zero-downtime RU patching
│       ├── patch_dg_step1_prepare.yml
│       ├── patch_dg_step2_standby.yml
│       ├── patch_dg_step3_switchover.yml
│       ├── patch_dg_step4_primary.yml
│       ├── patch_dg_step5_switchback.yml
│       ├── patch_dg_step6_datapatch.yml
│       ├── patch_dg_step7_validate.yml
│       └── patch_dg_rollback.yml
├── docs/
│   ├── Oracle_19c_DataGuard_Installation_en.md
│   ├── Oracle_19c_DataGuard_Installation_de.md
│   ├── Oracle_19c_DataGuard_Installation_ru.md
│   ├── Oracle_19c_DataGuard_Patching_en.md
│   ├── Oracle_19c_DataGuard_Patching_de.md
│   └── Oracle_19c_DataGuard_Patching_ru.md
├── files/
│   └── 19c/
│       └── patches/             # Place Oracle patch ZIPs here (not in Git)
├── LICENSE
└── README.md
```

---

## Quick Start

### 1. Prerequisites

- Ansible installed on the controller node
- Two servers with Oracle Linux 8 (or compatible RHEL-based OS)
- Oracle 19c Grid and Database ZIP files available
- SSH key-based authentication set up

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id sef@<primary-server>
ssh-copy-id sef@<standby-server>
```

### 2. Configure the Inventory

Copy and adapt the inventory file for your environment:

```bash
cp inventory/dataguard-001-002 inventory/my-dg-setup
```

Edit the key values:

```ini
[primary]
dghost-001 ansible_host=<PRIMARY_IP>

[standby]
dghost-002 ansible_host=<STANDBY_IP>
```

### 3. Set Up Vault for Passwords

**Never store passwords in plain text.** Use ansible-vault:

```bash
ansible-vault create inventory/secrets.yml
```

Add your passwords:

```yaml
vault_sys_password: "YourSecurePassword"
vault_system_password: "YourSecurePassword"
```

### 4. Run Installation Playbooks (in order)

```bash
cd /home/oracle/ansible

ansible-playbook -i inventory/dataguard-001-002 playbooks/dataguard/01_os_preparation.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/dataguard/02_grid_install.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/dataguard/03_db_install.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/dataguard/04_create_primary.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/dataguard/05_create_standby.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/dataguard/06_dg_broker.yml -K
```

### 5. Zero-Downtime Patching

```bash
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step1_prepare.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step2_standby.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step3_switchover.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step4_primary.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step5_switchback.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step6_datapatch.yml -K
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_step7_validate.yml -K
```

For rollback:

```bash
ansible-playbook -i inventory/dataguard-001-002 playbooks/patching/patch_dg_rollback.yml -K
```

---

## Documentation

Full step-by-step documentation is available in the `docs/` folder in English, German, and Russian:

- [Installation Guide (EN)](docs/Oracle_19c_DataGuard_Installation_en.md)
- [Patching Guide (EN)](docs/Oracle_19c_DataGuard_Patching_en.md)

---

## Reusing for Other DG Pairs

The playbooks are fully inventory-driven. To set up a second Data Guard pair:

1. Copy the inventory: `cp inventory/dataguard-001-002 inventory/dataguard-003-004`
2. Adjust hostnames, IPs, SIDs, and passwords
3. Run the same playbooks pointing to the new inventory

---

## Security Notes

- Passwords in the inventory use `{{ vault_sys_password }}` / `{{ vault_system_password }}` — store actual values in `ansible-vault`
- Oracle patch ZIP files are excluded from Git via `.gitignore` (too large, download from [My Oracle Support](https://support.oracle.com))
- This project was developed in a lab environment — review and adapt security settings before using in production

---

## License

[MIT](LICENSE)
