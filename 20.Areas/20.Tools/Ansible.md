# Ansible

Quick reference for everyday Ansible work.

## Activate Environment

```bash
source ~/ansible-infra/bin/activate
```

Load environment variables:

```bash
source .env
```

---

## Inventory

### Check static inventory

```bash
ansible-inventory -i inventory/hosts.ini --graph
```

Shows hosts and groups:

```text
inventory
├── servers
├── docker_hosts
└── k3s_cluster
```

### Check dynamic Proxmox inventory

```bash
ansible-inventory -i inventory/inventory.proxmox.yml --list
```

Use this when VMs are discovered automatically from Proxmox.

---

## Test Connection

Ping all hosts:

```bash
ansible -i inventory/hosts.ini all -m ping
```

Expected:

```text
SUCCESS => {
    "ping": "pong"
}
```

This checks:

```text
Ansible
   ↓
SSH
   ↓
Remote Server
```

---

## Check Playbook Before Run

### Syntax check

```bash
ansible-playbook site.yml --syntax-check
```

Checks YAML and Ansible syntax.

### Dry run

```bash
ansible-playbook site.yml --check --diff
```

Shows what Ansible plans to change without applying most changes.

Always check before important changes.

---

## Run Playbook

```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

With sudo:

```bash
ansible-playbook -i inventory/hosts.ini site.yml -b
```

With Ansible Vault:

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  site.yml \
  -b \
  --ask-vault-pass
```

---

## Run on Specific Host

```bash
ansible-playbook \
  -i inventory/hosts.ini \
  site.yml \
  --limit vm105
```

Useful when you do not want to change every server.

---

## Ad-Hoc Commands

Ping:

```bash
ansible all -i inventory/hosts.ini -m ping
```

Run a command:

```bash
ansible all \
  -i inventory/hosts.ini \
  -m shell \
  -a "uptime"
```

Run with sudo:

```bash
ansible all \
  -i inventory/hosts.ini \
  -m shell \
  -a "apt-get update" \
  -b
```

---

## Useful Flags

| Flag | Meaning |
|---|---|
| `-i` | Inventory file |
| `-u` | Remote SSH user |
| `-b` | Become root with sudo |
| `-K` | Ask for sudo password |
| `-k` | Ask for SSH password |
| `-m` | Ansible module |
| `-a` | Module arguments |
| `-e` | Extra variables |
| `--limit` | Run only on selected hosts |
| `--check` / `-C` | Dry run |
| `--diff` | Show configuration changes |
| `--syntax-check` | Check playbook syntax |
| `--ask-vault-pass` | Ask for Ansible Vault password |
| `-v` | Verbose output |
| `-vvv` | Detailed debugging |

---

## Safe Workflow

Before changing infrastructure:

```text
Inventory
    ↓
Ping
    ↓
Syntax Check
    ↓
Dry Run + Diff
    ↓
Apply
    ↓
Verify
```

Example:

```bash
ansible-inventory -i inventory/hosts.ini --graph

ansible -i inventory/hosts.ini all -m ping

ansible-playbook site.yml --syntax-check

ansible-playbook site.yml --check --diff

ansible-playbook site.yml
```

## Remember

```text
Inventory = WHERE
Playbook  = WHAT
Module    = HOW
Variables = DATA
```