# Ansible Playbooks

![Ansible](https://img.shields.io/badge/Automation-Ansible-EE0000?logo=ansible&logoColor=white)
![YAML](https://img.shields.io/badge/Format-YAML-CB171E)
![Focus](https://img.shields.io/badge/Focus-Configuration%20Management-informational)

A study and reference guide to **Ansible playbooks**: YAML-formatted automation blueprints for managing configuration, orchestrating processes, and deploying multi-tier applications in a repeatable way.

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [What Are Playbooks?](#what-are-playbooks)
- [Structure and Syntax](#structure-and-syntax)
- [Example Playbook Explained](#example-playbook-explained)
- [Running Playbooks](#running-playbooks)
- [Validating Playbooks](#validating-playbooks)
- [Benefits](#benefits)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Author](#author)

---

## Overview

Playbooks let you describe the *desired state* of your infrastructure rather than scripting individual commands. Ansible works out how to bring each machine to that state, which makes changes consistent, auditable, and safe to repeat.

## Learning Objectives

By the end of this guide you will be able to:

1. Explain what Ansible playbooks are and why they matter.
2. Read and write basic playbook syntax and structure.
3. Write and run playbooks to manage infrastructure and applications.
4. Use check mode to test playbooks before applying changes.
5. Validate playbooks to ensure they are free from errors.

---

## What Are Playbooks?

Playbooks are structured sets of instructions, written in YAML, that define tasks to be executed on remote nodes. They allow you to:

| Capability | Description |
|---|---|
| **Automate repetitive tasks** | System updates, package installations, configuration changes. |
| **Orchestrate multi-step processes** | Coordinate operations across different servers or services. |
| **Document processes** | Serve as human-readable documentation of how infrastructure is configured. |
| **Version control configurations** | Store playbooks in a repository for collaboration, auditing, and rollback. |

---

## Structure and Syntax

A playbook is made up of one or more **plays**. Each play targets a group of hosts and defines the tasks to run on them.

| Element | Description |
|---|---|
| **Play** | A mapping containing a `name`, target `hosts`, and a list of `tasks`. |
| **Task** | A single unit of work that calls an Ansible module (such as `yum` or `template`) with the required parameters. |
| **YAML structure** | Indentation with **spaces** (never tabs) is critical; each nesting level represents a deeper structure. |

### Example Playbook

```yaml
- name: Configure web servers
  hosts: webservers
  tasks:
    - name: Install Apache
      ansible.builtin.yum:
        name: httpd
        state: latest

    - name: Deploy configuration file
      ansible.builtin.template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf

- name: Configure database servers
  hosts: databases
  tasks:
    - name: Install PostgreSQL
      ansible.builtin.yum:
        name: postgresql
        state: latest

    - name: Ensure PostgreSQL is running
      ansible.builtin.service:
        name: postgresql
        state: started
```

> **Note:** This example uses the `yum` module, which targets RHEL-family systems (RHEL, CentOS, Rocky, Fedora). On Debian or Ubuntu hosts, use `ansible.builtin.apt` instead (and package names such as `apache2`). For distribution-independent playbooks, `ansible.builtin.package` picks the right package manager automatically.

---

## Example Playbook Explained

The playbook demonstrates the core idea of Ansible automation: you describe the desired state of different groups of machines, and Ansible makes them match.

- **Play 1 (`webservers`):** Installs Apache with the `yum` module, then places a configuration file on the server using the `template` module.
- **Play 2 (`databases`):** Installs PostgreSQL, then ensures the service is running.

These are **declarative instructions**, not shell commands. You are saying "Apache should be installed" and "this config file should exist here," which makes the playbook repeatable and safer than manual setup. One playbook can manage several server roles, while each play keeps its tasks separate and focused.

A useful mental model: **a playbook is a map from a host group to a desired outcome.**

> **Gotcha:** The groups `webservers` and `databases` must exist in your inventory. Otherwise Ansible does not know which machines to target and skips the play with a warning.

---

## Running Playbooks

Execute a playbook with `ansible-playbook`:

```bash
ansible-playbook playbook.yml
```

### Common Options

| Option | Purpose | Example |
|---|---|---|
| `--check` | Dry run: preview changes without applying them. | `ansible-playbook playbook.yml --check` |
| `--diff` | Show differences between current and desired state (most useful for templated files). | `ansible-playbook playbook.yml --diff` |
| `--limit` | Restrict execution to a subset of hosts. | `ansible-playbook playbook.yml --limit webservers` |

Options can be combined. A safe preview workflow:

```bash
ansible-playbook playbook.yml --check --diff
```

> **Limitation of check mode:** Tasks that depend on the result of earlier tasks (for example, starting a service whose package would only be installed in a real run) may report failures in `--check` mode. That does not necessarily mean the playbook is broken.

---

## Validating Playbooks

Catch errors before they reach your hosts:

```bash
# Check YAML and playbook syntax without executing anything
ansible-playbook playbook.yml --syntax-check

# Preview the hosts a play would target
ansible-playbook playbook.yml --list-hosts

# Preview tasks in order
ansible-playbook playbook.yml --list-tasks

# Dry run against real hosts
ansible-playbook playbook.yml --check
```

Optionally, lint for style and best-practice issues (installed separately):

```bash
ansible-lint playbook.yml
```

---

## Benefits

| Benefit | Why it matters |
|---|---|
| **Consistency** | Every node is configured the same way, reducing errors. |
| **Reusability** | Reuse playbooks across environments and share them across teams. |
| **Scalability** | Manage many nodes with a single playbook. |
| **Transparency** | Readable YAML is easy to audit and understand. |
| **Automation** | Routine tasks run without manual intervention. |

---

## Best Practices

- Always give plays and tasks a descriptive `name`; it makes run output readable.
- Prefer modules over `command`/`shell` so tasks stay idempotent.
- Run `--syntax-check`, then `--check --diff`, before applying changes.
- Keep playbooks in version control.
- Use fully qualified module names (`ansible.builtin.yum`) to avoid ambiguity.
- Prefer `state: present` over `state: latest` in production if you want predictable package versions.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `did not find expected key` / YAML parse error | Indentation mistake or tabs used | Use spaces only and keep nesting consistent; run `--syntax-check` |
| `skipping: no hosts matched` | Group in `hosts:` not defined in the inventory | Compare with `ansible-inventory --graph` |
| `No package matching ... found` | Wrong module or package name for the OS | Use `apt` on Debian/Ubuntu or `package` for portability |
| Permission denied when installing | Task needs elevated privileges | Add `become: true` to the play or task |
| `--check` reports errors on later tasks | Dependency on earlier changes not applied in dry run | Review the task; this can be expected in check mode |

---

## Author

**Gerald Oti** - DevOps / Platform / SRE Engineer, Berlin

- GitHub: [goti13](https://github.com/goti13)
- LinkedIn: [gerald-oti](https://www.linkedin.com/in/gerald-oti/)

