# Ansible

## What is Ansible?

Ansible is a configuration management and automation tool.

Instead of configuring every server manually:

```text
SSH → server1 → install Docker
SSH → server2 → install Docker
SSH → server3 → install Docker
```

we describe the desired configuration once:

```text
                    ┌── server1
Ansible → Playbook ─┼── server2
                    └── server3
```

Ansible connects to servers over SSH and applies the configuration.

---

# Core Architecture

```text
Control Node
     │
     │ SSH
     ↓
Managed Nodes
```

### Control Node

The machine where Ansible is installed and executed.

It contains:

```text
Inventory
Playbooks
Roles
Variables
ansible.cfg
```

### Managed Nodes

Servers or VMs managed by Ansible.

Usually Ansible does not need an agent installed on them.

---

# Inventory

Inventory answers:

> WHERE should Ansible run?

Example:

```ini
[servers]
vm1 ansible_host=192.168.8.101
vm2 ansible_host=192.168.8.102

[docker_hosts]
vm1
vm2
```

Architecture:

```text
Inventory
   ↓
Groups
   ↓
Hosts
```

Inventory can be:

```text
Static
   ↓
hosts.ini

Dynamic
   ↓
Proxmox / Cloud / API
```

Dynamic inventory is useful when infrastructure changes frequently.

---

# Playbook

A Playbook answers:

> WHAT should Ansible do?

Example:

```yaml
- hosts: servers
  become: true

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
```

Basic structure:

```text
Playbook
   ↓
Play
   ↓
Tasks
   ↓
Modules
```

---

# Modules

Modules are the tools Ansible uses to perform actions.

Examples:

```text
apt       → manage packages
file      → manage files/directories
user      → manage users
copy      → copy files
service   → manage services
shell     → execute shell commands
```

Conceptually:

```text
Task
 ↓
Module
 ↓
Change on server
```

Prefer a dedicated module when one exists instead of using `shell` for everything.

---

# Roles

As Playbooks grow, putting everything into one YAML file becomes difficult to maintain.

Roles split configuration into reusable components.

Example:

```text
roles/
├── common/
├── docker/
├── nginx/
└── k3s/
```

A role can contain:

```text
docker/
├── tasks/
├── handlers/
├── templates/
├── files/
├── vars/
├── defaults/
└── meta/
```

Instead of one huge Playbook:

```text
site.yml
 ├── 200 lines Docker
 ├── 100 lines Nginx
 └── 150 lines common config
```

we get:

```text
site.yml
   ↓
common
docker
nginx
```

Roles make automation easier to reuse and maintain.

---

# Ansible Galaxy

Ansible Galaxy provides reusable Roles and Collections.

Create a role:

```bash
ansible-galaxy role init docker
```

Install roles from `requirements.yml`:

```yaml
roles:
  - name: geerlingguy.docker
    src: geerlingguy.docker
```

Then:

```bash
ansible-galaxy install -r requirements.yml -p ./roles
```

Collections provide groups of modules and plugins.

Example:

```bash
ansible-galaxy collection install community.docker
```

Think of Galaxy as:

```text
Reusable Ansible components
        ↓
Roles + Collections
```

---

# Variables

Variables allow the same automation to work with different environments.

Instead of hardcoding:

```yaml
user: ops
```

we can use:

```yaml
user: "{{ server_user }}"
```

Variables may come from different places:

```text
Inventory
group_vars
host_vars
Playbook
Role
vars_files
Extra variables (-e)
```

This separates:

```text
Automation logic
       +
Environment data
```

---

# Ansible Vault

Passwords, tokens and other secrets should not be stored as plain text in the repository.

Example:

```text
vars.yml       → normal configuration
secrets.yml    → sensitive configuration
```

A Playbook can load both:

```yaml
vars_files:
  - vars.yml
  - secrets.yml
```

Encrypt secrets:

```bash
ansible-vault encrypt secrets.yml
```

Edit encrypted secrets:

```bash
ansible-vault edit secrets.yml
```

Run:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Concept:

```text
Git Repository
├── playbooks
├── roles
├── vars.yml
└── secrets.yml 🔐
```

The secret values remain encrypted at rest.

---

# Idempotency

One of the most important Ansible concepts.

Suppose the Playbook says:

```text
Docker must be installed
```

First run:

```text
Docker missing
      ↓
Install Docker
      ↓
CHANGED
```

Second run:

```text
Docker already installed
      ↓
Nothing to change
      ↓
OK
```

The goal is:

```text
Run 1 → configure server
Run 2 → no unnecessary changes
Run 3 → no unnecessary changes
```

This is why Ansible is more than a collection of remote shell scripts.

---

# Desired State

With manual administration we think:

```text
Run this command.
```

With configuration management we try to think:

```text
This server should be in this state.
```

For example:

```text
Docker       → installed
nginx        → running
ops user     → exists
config file  → correct
directory    → present
```

Ansible compares the current state with the desired state and changes what is necessary.

---

# Become

Many infrastructure operations require root privileges.

Ansible normally connects as a regular user:

```text
Ansible
   ↓ SSH
ops
   ↓ sudo
root privileges
```

This is controlled with:

```yaml
become: true
```

or:

```bash
-b
```

Do not use the root account for every SSH connection when normal user + controlled privilege escalation is sufficient.

---

# Handlers

Handlers run only when something changes.

Example:

```text
Change nginx.conf
       ↓
task = CHANGED
       ↓
notify handler
       ↓
reload nginx
```

If the configuration did not change:

```text
task = OK
   ↓
no reload
```

This avoids unnecessary service restarts.

---

# Project Structure

A practical Ansible repository can look like:

```text
ansible-infra/
├── inventory/
│   ├── hosts.ini
│   └── inventory.proxmox.yml
│
├── playbooks/
│
├── roles/
│   ├── common/
│   ├── docker/
│   └── k3s/
│
├── group_vars/
├── host_vars/
├── requirements.yml
├── ansible.cfg
└── site.yml
```

The important idea is separation:

```text
Inventory  → WHERE
Playbook   → WHAT
Role       → reusable configuration
Module     → HOW
Variables  → DATA
Vault      → SECRETS
```

---

# Typical Workflow

```text
Write configuration
        ↓
Check inventory
        ↓
Test SSH connectivity
        ↓
Syntax check
        ↓
Dry run
        ↓
Apply
        ↓
Verify
```

Ansible turns manual server configuration into repeatable infrastructure automation.