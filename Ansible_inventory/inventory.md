# Ansible Inventory Lab
 
![Ansible](https://img.shields.io/badge/Automation-Ansible-EE0000?logo=ansible&logoColor=white)
![Inventory](https://img.shields.io/badge/Format-INI%20Inventory-blue)
![Focus](https://img.shields.io/badge/Focus-Configuration%20Management-informational)
 
A hands-on lab that builds a static, INI-style **Ansible inventory** containing Linux servers, a Windows server, and `localhost`, wires it up through `ansible.cfg`, and verifies it with `ansible --list-hosts`.
 
---
 
## Table of Contents
 
- [Overview](#overview)
- [Core Concepts](#core-concepts)
- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Hands-On Tasks](#hands-on-tasks)
- [Verification](#verification)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)
- [Author](#author)
---
 
## Overview
 
An inventory is the collection of hosts (managed machines) that Ansible can reach. Think of it as the phone book Ansible consults to know where to execute each task.
 
## Core Concepts
 
| Term | Meaning | Typical location |
|---|---|---|
| **Host** | A single managed node, identified by FQDN, IP, or alias. | A line in `hosts`, or returned by a dynamic plugin. |
| **Group** | A label bundling hosts with something in common (role, environment, OS, region). | `[web]`, `[db]`, etc. in the inventory file. |
| **Implicit groups** | Built-in groups: `all` (every host) and `ungrouped` (hosts in no other group). | Added automatically. |
| **Inventory source** | The file or script supplying host data. | Static INI/YAML file, cloud plugin, CMDB script. |
| **`ansible.cfg`** | Tells Ansible which inventory to use and how it should behave. | Project root, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`. |
| **`host_vars/`** | Files named after hosts; variables apply only to that host. | `host_vars/web1.yml` |
| **`group_vars/`** | Files named after groups; variables apply to every host in the group. | `group_vars/web.yml` |
 
**Variable precedence (simplified):** host-level vars override group-level vars, which override inventory-wide defaults. Playbook vars and extra-vars can override all of these.
 
### Static vs. Dynamic Inventories
 
- **Static (file-based):** a plain text file in INI or YAML. Ideal for small labs and demos.
- **Dynamic (plugin-based):** pulls live host lists from AWS, Azure, VMware, Kubernetes, and more. Suited to elastic or cloud-native fleets.
### Where Ansible Looks for `ansible.cfg`
 
Ansible uses the first match in this order:
 
1. The `ANSIBLE_CONFIG` environment variable (if set)
2. `ansible.cfg` in the current directory
3. `~/.ansible.cfg` in the user's home directory
4. `/etc/ansible/ansible.cfg`
Keeping an `ansible.cfg` next to your playbooks makes the project self-contained.
 
---
 
## Objectives
 
By the end of this lab you will be able to:
 
- Explain hosts, groups, implicit groups, and variable directories.
- Create an INI-style inventory (`hosts`) containing Linux and Windows servers plus `localhost`.
- Configure `ansible.cfg` to point at that inventory.
- Verify the inventory with `ansible --list-hosts`.
## Prerequisites
 
| Requirement | Details |
|---|---|
| Text editor | VS Code, Sublime Text, Vim, etc. |
| Ansible installed | Confirm with `ansible --version` |
| Basic CLI skills | Creating folders and files from a shell |
 
## Expected Project Structure
 
```
inventory_assignment/
├── ansible.cfg
├── hosts
├── group_vars/
│   ├── all.yml
│   └── web_servers.yml
└── host_vars/
    └── db1.yml
```
 
`group_vars/` and `host_vars/` are optional (Task 3).
 
---
 
## Hands-On Tasks
 
### Task 1: Create the project skeleton
 
```bash
mkdir inventory_assignment
cd inventory_assignment
touch ansible.cfg hosts
```
 
Add the following to `ansible.cfg`:
 
```ini
[defaults]
inventory = hosts
```
 
### Task 2: Define the inventory
 
The lab environment consists of:
 
| Server | Domain / IP | OS | User |
|---|---|---|---|
| web1 | server1.ironhack.com | Linux | root |
| web2 | server2.ironhack.com | Linux | admin |
| web3 | server3.ironhack.com | Linux | root |
| db1 | db1.ironhack.com | Windows | admin |
| local | localhost | Linux | n/a |
 
Add the following to `hosts`:
 
```ini
[web_servers]
server1.ironhack.com ansible_user=root  ansible_password=<password>
server2.ironhack.com ansible_user=admin ansible_password=<password>
server3.ironhack.com ansible_user=root  ansible_password=<password>
 
[db_servers]
db1.ironhack.com     ansible_user=admin ansible_password=<password>
 
[all_servers:children]
web_servers
db_servers
 
# Run tasks locally
localhost             ansible_connection=local
```
 
> The lab uses throwaway credentials. Never commit real passwords to a repository. See [Security Notes](#security-notes).
 
The `[all_servers:children]` section defines a **group of groups**: `all_servers` contains every host in `web_servers` and `db_servers`.
 
### Task 3 (Optional): `host_vars` and `group_vars`
 
```bash
mkdir host_vars group_vars
echo "ntp_server: time.nist.gov"      > group_vars/all.yml
echo "app_port: 8080"                 > group_vars/web_servers.yml
echo "windows_temp_dir: C:\\Temp"     > host_vars/db1.yml
```
 
| File | Applies to |
|---|---|
| `group_vars/all.yml` | Every host |
| `group_vars/web_servers.yml` | Hosts in `web_servers` |
| `host_vars/db1.yml` | A single host named `db1` |
 
> **Important:** the filename in `host_vars/` must match the host's name *as written in the inventory*. In this lab the host is listed as `db1.ironhack.com`, so `host_vars/db1.yml` will **not** be picked up. Either rename the file to `db1.ironhack.com.yml`, or define an alias in the inventory (see [Troubleshooting](#troubleshooting)).
 
---
 
## Verification
 
```bash
# List every host
ansible all --list-hosts
 
# List web servers only
ansible web_servers --list-hosts
 
# List DB servers only
ansible db_servers --list-hosts
 
# Confirm the localhost entry
ansible localhost --list-hosts
```
 
Expected results:
 
| Command | Hosts listed |
|---|---|
| `ansible all --list-hosts` | All three web servers, `db1.ironhack.com`, and `localhost` |
| `ansible web_servers --list-hosts` | `server1`, `server2`, `server3` |
| `ansible db_servers --list-hosts` | `db1.ironhack.com` |
| `ansible localhost --list-hosts` | `localhost` |
 
To inspect variables as Ansible resolves them:
 
```bash
ansible-inventory --list
ansible-inventory --graph
```
 
`--list-hosts` only reads the inventory and does not connect to any machine, so the commands above work even though the example hostnames are not reachable.
 
---
 
## Security Notes
 
- Plaintext `ansible_password` values are acceptable **only** for isolated labs.
- For real environments, use [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html) or an external secrets manager, and prefer SSH key authentication over passwords.
- Add any file containing secrets to `.gitignore`.
---
 
## Troubleshooting
 
| Symptom | Likely cause | Fix |
|---|---|---|
| `[WARNING]: No inventory was parsed` | `ansible.cfg` not found or `inventory` path wrong | Run commands from the project directory; check `ansible --version` for the config file in use |
| `Could not match supplied host pattern` | Group or host name misspelled | Compare the name to your `hosts` file |
| `host_vars` values not applied | Filename does not match the inventory hostname | Rename the file, or use an alias: `db1 ansible_host=db1.ironhack.com` |
| Windows host cannot be managed | Only inventory is defined; no Windows connection settings | Add `ansible_connection=winrm` and related WinRM variables before running modules against it |
| Unexpected variable value | Precedence conflict | Check with `ansible-inventory --host <name>` and remember host vars beat group vars |
 
---
 
## Key Takeaways
 
- The inventory is the source of truth for *where* Ansible runs; `ansible.cfg` tells Ansible *which* inventory to use.
- Groups can contain other groups using the `:children` suffix, and `all` / `ungrouped` exist implicitly.
- Keep configuration data in `group_vars/` and `host_vars/` rather than inline in the inventory.
- Static inventories suit labs; dynamic inventory plugins suit cloud and elastic infrastructure.
---
 
## Author
 
**Gerald Oti** - DevOps / Platform / SRE Engineer, Berlin
 
- GitHub: [goti13](https://github.com/goti13)
- LinkedIn: [gerald-oti](https://www.linkedin.com/in/gerald-oti/)
