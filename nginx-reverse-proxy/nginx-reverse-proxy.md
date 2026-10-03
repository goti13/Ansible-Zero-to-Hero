# Nginx Reverse Proxy with Ansible and Jinja2 Templating

![Ansible](https://img.shields.io/badge/Automation-Ansible-EE0000?logo=ansible&logoColor=white)
![Nginx](https://img.shields.io/badge/Web%20Server-Nginx-009639?logo=nginx&logoColor=white)
![AWS EC2](https://img.shields.io/badge/Cloud-AWS%20EC2-FF9900?logo=amazonaws&logoColor=white)
![Jinja2](https://img.shields.io/badge/Templating-Jinja2-B41717?logo=jinja&logoColor=white)

A hands-on lab that deploys **two Ubuntu servers on AWS**, one acting as an Nginx **reverse proxy** and the other as an Nginx **upstream server**, and configures both with **Ansible**. The reverse proxy's configuration is generated from a **Jinja2 template**, so environment-specific values are filled in automatically without manual edits.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Learning Objectives](#learning-objectives)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Step 1: Provision the EC2 Instances](#step-1-provision-the-ec2-instances)
- [Step 2: Set Up the Ansible Inventory and Configuration](#step-2-set-up-the-ansible-inventory-and-configuration)
- [Step 3: Create the Jinja2 Template](#step-3-create-the-jinja2-template)
- [Step 4: Upstream Server Playbook](#step-4-upstream-server-playbook)
- [Step 5: Reverse Proxy Playbook](#step-5-reverse-proxy-playbook)
- [Step 6: Run the Playbooks and Verify](#step-6-run-the-playbooks-and-verify)
- [How It Works](#how-it-works)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Possible Improvements](#possible-improvements)
- [Author](#author)

---

## Overview

| Server | Role | Behavior |
|---|---|---|
| `reverse-proxy` | Reverse proxy | Receives client requests on port 80 and forwards them to the upstream server |
| `upstream-server` | Upstream (backend) | Listens on port 80 and serves content directly to the reverse proxy |

The lab demonstrates how Ansible and Jinja2 templates generate configuration dynamically: the proxy's server name and upstream address come from the Ansible inventory, not from hand-edited files.

## Architecture

```
                         ┌───────────────────────────┐
                         │  Local machine            │
                         │  Ansible control node     │
                         └─────────────┬─────────────┘
                                 SSH (22)
                  ┌────────────────────┴────────────────────┐
                  ▼                                         ▼
      ┌───────────────────────┐   proxy_pass    ┌───────────────────────┐
Client│  reverse-proxy        │ ──────────────► │  upstream-server      │
 ───► │  Nginx :80            │   HTTP :80      │  Nginx :80            │
 HTTP │  (Jinja2-rendered     │                 │  "Hello from the      │
      │   config)             │ ◄────────────── │   Upstream Server..." │
      └───────────────────────┘                 └───────────────────────┘
```

## Learning Objectives

By the end of this lab you will be able to:

1. Provision two Ubuntu servers on AWS (one reverse proxy, one upstream).
2. Use Ansible to install and configure Nginx on both servers.
3. Use a Jinja2 template for the reverse proxy configuration, with customizable variables.
4. Serve content from the upstream server on port 80.
5. Verify that the reverse proxy routes inbound requests to the upstream server.

## Requirements

| Requirement | Details |
|---|---|
| Local machine | Ansible installed (`ansible --version`) |
| AWS | Two Ubuntu-based EC2 instances |
| Security groups | Inbound **22** (SSH) and **80** (HTTP) allowed on both instances |
| Access | An SSH key pair (`.pem` file) |
| Knowledge | Basic AWS provisioning and Nginx concepts |

## Project Structure

```
nginx-reverse-proxy/
├── ansible.cfg
├── hosts
├── deploy_upstream.yml
├── deploy_reverse_proxy.yml
└── templates/
    └── nginx.conf.j2
```

---

## Step 1: Provision the EC2 Instances

1. In the AWS Management Console, go to **EC2, Instances, Launch Instance**.
2. Select an **Ubuntu AMI**.
3. Create two instances, named `reverse-proxy` and `upstream-server`.
4. Attach or create a security group that allows:
   - SSH (port 22) inbound
   - HTTP (port 80) inbound on both servers (for testing)
5. Use or create a key pair (`.pem` file).
6. Note the public IP or DNS name of each instance once running.

Check SSH connectivity to both servers:

```bash
ssh -i my_key.pem ubuntu@<reverse_proxy_ip_or_dns>
ssh -i my_key.pem ubuntu@<upstream_server_ip_or_dns>
```

Make sure you can log in to both without issues. If SSH rejects the key with a permissions error, run `chmod 400 my_key.pem`.

---

## Step 2: Set Up the Ansible Inventory and Configuration

Create the project directory:

```bash
mkdir nginx-reverse-proxy
cd nginx-reverse-proxy
```

### Inventory file: `hosts`

```ini
[reverse_proxy]
reverse ansible_host=<reverse_proxy_ip_or_dns> ansible_user=ubuntu ansible_private_key_file=/path/to/my_key.pem

[upstream_server]
upstream ansible_host=<upstream_server_ip_or_dns> ansible_user=ubuntu ansible_private_key_file=/path/to/my_key.pem
```

### Configuration file: `ansible.cfg`

```ini
[defaults]
inventory = hosts
```

### Validate the inventory

```bash
ansible all -m ping
```

Both hosts (`reverse` and `upstream`) should respond with `pong`.

> On first connection, SSH asks you to confirm each host's fingerprint. Type `yes`, or connect manually once beforehand. For a throwaway lab you can add `host_key_checking = False` to `ansible.cfg`.

---

## Step 3: Create the Jinja2 Template

```bash
mkdir templates
```

### `templates/nginx.conf.j2`

```nginx
server {
    listen 80;
    server_name {{ server_name }};

    location / {
        proxy_pass http://{{ upstream_server }};
    }
}
```

| Placeholder | Meaning |
|---|---|
| `{{ server_name }}` | The reverse proxy's server name or IP |
| `{{ upstream_server }}` | The IP and port of the backend upstream server |

---

## Step 4: Upstream Server Playbook

### `deploy_upstream.yml`

```yaml
---
- name: Deploy Upstream Nginx Server
  hosts: upstream_server
  become: yes

  tasks:
    - name: Update apt cache and install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Deploy a sample index page
      copy:
        dest: /var/www/html/index.html
        content: "Hello from the Upstream Server on port 80!"
      notify: reload nginx

  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

Nginx listens on port 80 by default on Ubuntu, so the upstream server needs no extra listener configuration. The playbook installs Nginx and replaces the default page with a recognizable message.

---

## Step 5: Reverse Proxy Playbook

### `deploy_reverse_proxy.yml`

```yaml
---
- name: Deploy Nginx Reverse Proxy
  hosts: reverse_proxy
  become: yes

  vars:
    server_name: "{{ hostvars['reverse'].ansible_host }}"
    upstream_server: "{{ hostvars['upstream'].ansible_host }}:80"

  tasks:
    - name: Update apt cache and install Nginx
      apt:
        name: nginx
        state: latest
        update_cache: yes

    - name: Deploy Nginx reverse proxy config from Jinja2 template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify:
        - reload nginx

    - name: Ensure Nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

---

## Step 6: Run the Playbooks and Verify

Deploy the upstream server **first**, then the reverse proxy:

```bash
ansible-playbook deploy_upstream.yml
ansible-playbook deploy_reverse_proxy.yml
```

Test through the reverse proxy:

```bash
curl http://<reverse_proxy_ip>
```

Expected output, served by the upstream server through the proxy:

```
Hello from the Upstream Server on port 80!
```

### Additional checks

Confirm the proxy is really forwarding and not serving content itself:

```bash
# 1. Inspect the rendered config on the proxy
ansible reverse -b -a "cat /etc/nginx/sites-available/default"

# 2. Validate the Nginx configuration
ansible reverse -b -a "nginx -t"

# 3. Stop the upstream Nginx; the proxy should now return 502 Bad Gateway
ansible upstream -b -a "systemctl stop nginx"
curl -i http://<reverse_proxy_ip>

# 4. Start it again; the greeting should return
ansible upstream -b -a "systemctl start nginx"
curl http://<reverse_proxy_ip>
```

A `502 Bad Gateway` while the upstream is stopped, and the greeting after it starts, proves the response comes from the upstream server.

---

## How It Works

1. **Variables are built from the inventory.** The playbook sets `server_name` and `upstream_server` from `hostvars`, which holds each host's inventory variables. Changing an IP in `hosts` changes the generated configuration.
2. **The template module renders the file.** Ansible substitutes the `{{ ... }}` placeholders and copies the result to `/etc/nginx/sites-available/default` on the proxy. On Ubuntu this file is already linked into `sites-enabled`, so it becomes the active site.
3. **The handler reloads Nginx.** `notify` triggers the `reload nginx` handler only when the rendered file actually changed, so repeated runs do not disturb a healthy server.
4. **Nginx proxies the request.** `proxy_pass` forwards each request to the upstream address and returns its response to the client.

---

## Security Notes

- Port 80 is open on **both** servers so you can test each one directly. After testing, restrict the upstream server's port 80 rule to the reverse proxy's security group (or private IP) so clients cannot bypass the proxy.
- Restrict SSH (port 22) to your own IP rather than `0.0.0.0/0`.
- When both instances are in the same VPC, point `upstream_server` at the upstream's **private** IP so proxy traffic stays off the public internet.
- Never commit `.pem` files. Add `*.pem` to `.gitignore`.
- Stop or terminate the instances when you finish to avoid charges.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ansible all -m ping` fails with `UNREACHABLE` | Wrong IP, key path, or SSH blocked | Check `ansible_host`, the `.pem` path, and the port 22 rule |
| `Permissions 0644 ... are too open` | Key file too permissive | `chmod 400 my_key.pem` |
| `Could not find or access 'templates/nginx.conf.j2'` | Playbook run from the wrong directory | Run `ansible-playbook` from the project root |
| `'server_name' is undefined` | Variable missing from `vars` | Check the `vars` block in the playbook |
| `502 Bad Gateway` | Upstream not running or not reachable | Verify the upstream with `curl http://<upstream_ip>`; check security groups |
| `504 Gateway Timeout` | Port 80 on the upstream blocked, or wrong IP used | Allow the proxy to reach the upstream on port 80 |
| Default Nginx page still appears on the proxy | Config not applied or not reloaded | Run `nginx -t`, then `sudo systemctl reload nginx` |
| Logs | Errors and access records | `sudo tail -f /var/log/nginx/error.log` |

---

## Possible Improvements

- Forward client details to the backend by adding headers inside `location /`:
  `proxy_set_header Host $host;`, `proxy_set_header X-Real-IP $remote_addr;`, and `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`.
- Template more values (listen port, upstream port, timeouts) and move them into `group_vars/`.
- Add multiple upstream servers using an `upstream {}` block for **load balancing**.
- Use fully qualified module names (`ansible.builtin.apt`, `ansible.builtin.template`) and `ansible-lint`.
- Add HTTPS with Let's Encrypt (Certbot) on the reverse proxy.
- Provision the EC2 instances with Terraform instead of the console.

---

## Author

**Gerald Oti** - DevOps / Platform / SRE Engineer, Berlin

- GitHub: [goti13](https://github.com/goti13)
- LinkedIn: [gerald-oti](https://www.linkedin.com/in/gerald-oti/)

