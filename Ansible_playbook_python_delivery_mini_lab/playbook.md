# Flaskerino Two-Server Lab: Deploying a Flask App with Ansible

![Ansible](https://img.shields.io/badge/Automation-Ansible-EE0000?logo=ansible&logoColor=white)
![AWS EC2](https://img.shields.io/badge/Cloud-AWS%20EC2-FF9900?logo=amazonaws&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%2022.04-E95420?logo=ubuntu&logoColor=white)

A hands-on lab (about 60 minutes) that provisions **two Ubuntu EC2 instances** and uses **Ansible** to deploy a Python Flask application ("Flaskerino") to both, serving on port 80. The deployment is built incrementally with four small playbooks so each stage can be run and verified on its own.

Application source: [professordiogodev/devops.flaskerino](https://github.com/professordiogodev/devops.flaskerino)

---

## Table of Contents

- [Objectives](#objectives)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1: Run the App Locally](#step-1-run-the-app-locally)
- [Step 2: Provision the EC2 Instances](#step-2-provision-the-ec2-instances)
- [Step 3: Project Layout and Configuration](#step-3-project-layout-and-configuration)
- [Step 4: Incremental Playbooks](#step-4-incremental-playbooks)
- [Step 5: Verify](#step-5-verify)
- [Module Cheat Sheet](#module-cheat-sheet)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Possible Improvements](#possible-improvements)
- [Author](#author)

---

## Objectives

1. Run the app locally to understand its requirements.
2. Provision two Ubuntu EC2 instances.
3. Automate the full deployment with Ansible so both servers run the Flask app on port 80.

## Architecture

```
                     ┌────────────────────────┐
                     │   Laptop (control node)│
                     │  Ansible + SSH key     │
                     └───────────┬────────────┘
                        SSH (22) │
              ┌──────────────────┴──────────────────┐
              ▼                                     ▼
   ┌─────────────────────┐               ┌─────────────────────┐
   │  flaskerino-01      │               │  flaskerino-02      │
   │  Ubuntu 22.04       │               │  Ubuntu 22.04       │
   │  Flask app :80      │               │  Flask app :80      │
   │  "number 1"         │               │  "number 2"         │
   └─────────────────────┘               └─────────────────────┘
```

## Prerequisites

| Tool | Quick check | Install if missing |
|---|---|---|
| Python 3.9+ | `python3 --version` | `brew install python` / `sudo apt install python3` |
| pip | `pip --version` | `python3 -m ensurepip --upgrade` |
| Git | `git --version` | `brew install git` / `sudo apt install git` |
| Ansible 2.15+ | `ansible --version` | `pip install --user ansible` |
| AWS CLI (optional) | `aws sts get-caller-identity` | `brew install awscli` / `sudo apt install awscli` |

You also need an **AWS key pair (`.pem`)** and either a default VPC or permission to create one.

---

## Step 1: Run the App Locally

Get a feel for the app before automating it.

```bash
# 1. Clone
git clone https://github.com/professordiogodev/devops.flaskerino.git
cd devops.flaskerino

# 2. Install dependencies
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 3. Run (defaults to port 80, so pick another port locally)
export PORT=8080
python3 app.py
# Visit http://localhost:8080
```

Stop the server with `Ctrl-C` once you see the greeting.

The app is configured through environment variables, which the deployment later reuses:

| Variable | Purpose |
|---|---|
| `PORT` | Port the app listens on |
| `DESIRED_PATH` | URL path that serves the greeting |
| `NUMBER` | Server number shown in the greeting |

---

## Step 2: Provision the EC2 Instances

| Setting | Value |
|---|---|
| AMI | Ubuntu Server 22.04 LTS (64-bit x86) |
| Instance type | `t3.micro` (free tier) |
| Key pair | Select or upload your `.pem` |
| Inbound TCP 80 | `0.0.0.0/0` (app traffic) |
| Inbound TCP 22 | Your IP only (SSH) |
| Tags | `Name=flaskerino-01` and `Name=flaskerino-02` |

Record each instance's public IP for the inventory.

---

## Step 3: Project Layout and Configuration

```
flaskerino-lab/
├── inventory.ini
├── ansible.cfg
├── 01_ping.yml
├── 02_clone.yml
├── 03_deps.yml
└── 04_run.yml
```

### `inventory.ini`

```ini
[flaskers]
flask1 ansible_host=<IP-1> ansible_user=ubuntu ansible_ssh_private_key_file=./my-key.pem
flask2 ansible_host=<IP-2> ansible_user=ubuntu ansible_ssh_private_key_file=./my-key.pem
```

### `ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False
```

> `host_key_checking = False` skips the first-connection SSH prompt. It is convenient for a lab but should not be used in production.

---

## Step 4: Incremental Playbooks

Run each playbook after writing it and confirm it works before moving on:

```bash
ansible-playbook <file>.yml
```

| # | What it does | File |
|---|---|---|
| 1 | Pings both hosts (connectivity proof) | `01_ping.yml` |
| 2 | Clones the repo into `/opt/flaskerino` | `02_clone.yml` |
| 3 | Installs Python, pip, and Flask dependencies | `03_deps.yml` |
| 4 | Launches the app in the background | `04_run.yml` |

### 4.1 `01_ping.yml`

```yaml
---
- name: 1 - Ping hosts
  hosts: flaskers
  gather_facts: no

  tasks:
    - name: Say hello
      ansible.builtin.ping:
```

> Despite the name, the `ping` module is not an ICMP ping. It confirms that Ansible can log in over SSH and run Python on the target.

### 4.2 `02_clone.yml`

```yaml
---
- name: 2 - Clone repo
  hosts: flaskers
  tasks:
    - name: Ensure git is present
      ansible.builtin.apt:
        name: git
        state: present
        update_cache: yes
      become: yes

    - name: Clone Flaskerino
      ansible.builtin.git:
        repo: https://github.com/professordiogodev/devops.flaskerino.git
        dest: /opt/flaskerino
        version: main
      become: yes
```

### 4.3 `03_deps.yml`

```yaml
---
- name: 3 - Python & packages
  hosts: flaskers
  become: yes
  tasks:
    - name: Install Python, pip, virtualenv
      ansible.builtin.apt:
        name:
          - python3
          - python3-pip
          - python3-venv
        state: present
        update_cache: yes

    - name: Create virtualenv
      ansible.builtin.command: python3 -m venv /opt/flaskerino/venv
      args:
        creates: /opt/flaskerino/venv

    - name: Install requirements
      ansible.builtin.pip:
        requirements: /opt/flaskerino/requirements.txt
        virtualenv: /opt/flaskerino/venv
```

The `creates:` argument makes the `command` task idempotent: it is skipped if the virtualenv already exists.

### 4.4 `04_run.yml`

```yaml
---
- name: 4 - Run Flaskerino
  hosts: flaskers
  become: yes
  vars:
    flask_port: 80
    flask_path: /
    flask_number: "{{ (inventory_hostname == 'flask1') | ternary('1', '2') }}"
  tasks:
    - name: Kill any previous Flaskerino
      ansible.builtin.shell: |
        pkill -f "/opt/flaskerino/app.py" || true
      args:
        executable: /bin/bash
      register: kill_result
      failed_when: false        # never fail even if no process is found

    - name: Launch Flaskerino (detached)
      ansible.builtin.shell: |
        source /opt/flaskerino/venv/bin/activate
        export PORT={{ flask_port }} DESIRED_PATH={{ flask_path }} NUMBER={{ flask_number }}
        nohup python3 /opt/flaskerino/app.py > /var/log/flaskerino.log 2>&1 &
      args:
        chdir: /opt/flaskerino
        executable: /bin/bash
```

`nohup ... &` keeps the server running after the playbook finishes.

> **Changes from the original lab material:**
> - The `flask_number` expression is parenthesized. Without the parentheses, Jinja applies the `ternary` filter to `'flask1'` first, so the variable does not resolve to `1` / `2` as intended.
> - Output is redirected to `/var/log/flaskerino.log`, which avoids the task hanging on an open output stream and gives you a log to inspect.

---

## Step 5: Verify

In a browser:

| URL | Expected |
|---|---|
| `http://<IP-1>/` | "Hello from server / number 1!" |
| `http://<IP-2>/` | "Hello from server / number 2!" |

Health check:

```bash
curl http://<IP-1>/healthcheck
# It works!
```

---

## Module Cheat Sheet

| Module | Purpose | Docs |
|---|---|---|
| `ping` | Test SSH reachability | [ansible.builtin.ping](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html) |
| `git` | Clone or update repositories | [ansible.builtin.git](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html) |
| `apt` | Install Debian/Ubuntu packages | [ansible.builtin.apt](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html) |
| `pip` | Install Python packages from `requirements.txt` | [ansible.builtin.pip](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pip_module.html) |
| `shell` | Run ad-hoc commands (for example, starting the server) | [ansible.builtin.shell](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html) |

---

## Security Notes

- Never commit your `.pem` key to a repository. Add `*.pem` to `.gitignore`.
- Restrict SSH (port 22) to your own IP in the security group.
- Set key permissions before first use: `chmod 400 my-key.pem`.
- Stop or terminate the EC2 instances when you finish to avoid unexpected charges.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `UNREACHABLE` / connection timed out | Security group blocks SSH, or wrong IP | Confirm port 22 allows your current IP and the inventory IPs are correct |
| `Permissions 0644 for key are too open` | `.pem` file too permissive | `chmod 400 my-key.pem` |
| `Permission denied (publickey)` | Wrong key or user | Use `ansible_user=ubuntu` and the key pair chosen at launch |
| Browser cannot reach the app | Port 80 not open, or the app is not running | Check the security group; run `pgrep -af app.py` on the instance; read `/var/log/flaskerino.log` |
| Both servers report the same number | `flask_number` expression not evaluated as intended | Use the parenthesized `ternary` expression shown above |
| App stops after a reboot | The app is launched with `nohup`, not a service | See [Possible Improvements](#possible-improvements) |

---

## Possible Improvements

- Replace the `nohup` launch with a **systemd unit** so the app restarts on failure and on reboot.
- Run the app with a production WSGI server such as Gunicorn behind Nginx.
- Combine the four playbooks into a single `site.yml`, or refactor them into an **Ansible role**.
- Provision the EC2 instances with Terraform instead of the console.

---

## Author

**Gerald Oti** - DevOps / Platform / SRE Engineer, Berlin

- GitHub: [goti13](https://github.com/goti13)
- LinkedIn: [gerald-oti](https://www.linkedin.com/in/gerald-oti/)
