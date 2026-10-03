# Ansible Roles Lab: Web Server and Load Balancer
 
![Ansible](https://img.shields.io/badge/IaC-Ansible-EE0000?logo=ansible&logoColor=white)
![Apache](https://img.shields.io/badge/Web%20Server-Apache2-D22128?logo=apache&logoColor=white)
![PHP](https://img.shields.io/badge/Language-PHP-777BB4?logo=php&logoColor=white)
![Jinja2](https://img.shields.io/badge/Templating-Jinja2-B41717?logo=jinja&logoColor=white)
 
A hands-on lab that packages automation into reusable **Ansible roles**. You scaffold two roles, `webserver` and `loadbalancer`, with `ansible-galaxy`, implement their tasks, handlers, and templates, and consume them from a single playbook that targets separate host groups.
 
---
 
## Table of Contents
 
- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step 1: Generate Role Skeletons](#step-1-generate-role-skeletons)
- [Step 2: The webserver Role](#step-2-the-webserver-role)
- [Step 3: The loadbalancer Role](#step-3-the-loadbalancer-role)
- [Step 4: Inventory and Configuration](#step-4-inventory-and-configuration)
- [Step 5: The Playbook](#step-5-the-playbook)
- [Step 6: Run and Verify](#step-6-run-and-verify)
- [How It Works](#how-it-works)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Possible Improvements](#possible-improvements)
- [Author](#author)
---
 
## Overview
 
Roles are Ansible's way to package tasks, handlers, variables, files, templates, and documentation in a predictable structure. They keep code modular, shareable, and easy to test, which matters once your automation grows beyond a single playbook.
 
In this lab you will:
 
1. Scaffold two roles (`webserver` and `loadbalancer`) with `ansible-galaxy role init`.
2. Implement tasks, handlers, files, and templates.
3. Consume the roles in a playbook that targets separate host groups.
4. Verify the deployment from a browser or with `curl`.
## Architecture
 
```
                    ┌──────────────────────┐
                    │ Control node         │
                    │ ansible-playbook     │
                    └──────────┬───────────┘
                          SSH  │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
   ┌─────────────────────┐   proxy     ┌─────────────────────┐
 ─►│ lb1  [loadbalancer] │ ──────────► │ web1  [webserver]   │
HTTP│ Apache reverse proxy│  HTTP :80  │ Apache + PHP        │
   │ balancer://webcluster│            │ index.php           │
   └─────────────────────┘             └─────────────────────┘
```
 
## Prerequisites
 
| Requirement | Notes |
|---|---|
| Control node | Linux or macOS workstation, or a small VM |
| Ansible 2.14+ | Check with `ansible --version` |
| SSH access | One VM for the `[webserver]` group, one VM for the `[loadbalancer]` group (Ubuntu) |
| Editor | VS Code or any IDE |
| Optional | AWS or Azure console to demonstrate the result |
 
> **Tip:** If you use a cloud key, never include its path or a VM's public IP in screenshots you share publicly.
 
## Project Structure
 
```
ansible-roles-lab/
├── roles/
│   ├── webserver/
│   │   ├── files/index.php
│   │   ├── handlers/main.yml
│   │   └── tasks/main.yml
│   └── loadbalancer/
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       └── templates/loadbalancer.conf.j2
└── playbooks/
    ├── ansible.cfg
    ├── hosts
    └── site.yml
```
 
`ansible-galaxy` also generates `defaults/`, `meta/`, `tests/`, `vars/`, and a role `README.md`. This lab only fills in the files shown above; the rest can stay empty or be deleted.
 
---
 
## Step 1: Generate Role Skeletons
 
From your project root:
 
```bash
ansible-galaxy role init roles/webserver
ansible-galaxy role init roles/loadbalancer
```
 
Each command creates a full role tree:
 
| Directory | Purpose |
|---|---|
| `tasks/` | The list of actions the role performs (`main.yml` is the entry point) |
| `handlers/` | Actions triggered by `notify`, such as restarting a service |
| `files/` | Static files copied as-is |
| `templates/` | Jinja2 templates rendered with variables |
| `defaults/` and `vars/` | Role variables (defaults are easy to override) |
| `meta/` | Role metadata and dependencies |
| `tests/` | A place for role tests |
 
---
 
## Step 2: The webserver Role
 
### 2.1 Static content: `roles/webserver/files/index.php`
 
```php
<?php
echo "<h1>Hello, World! - served by Ansible</h1>";
?>
```
 
### 2.2 Tasks: `roles/webserver/tasks/main.yml`
 
```yaml
- name: Install Apache and PHP
  apt:
    name:
      - apache2
      - php
    state: present
    update_cache: yes
  become: true
 
- name: Copy index.php to Apache docroot
  copy:
    src: index.php
    dest: /var/www/html/index.php
    mode: '0644'
  become: true
  notify: Restart Apache
 
- name: Remove index.html from Apache docroot
  ansible.builtin.file:
    path: /var/www/html/index.html
    state: absent
  become: true
```
 
The default `index.html` is removed because Apache's `DirectoryIndex` lists `index.html` before `index.php`. Leaving it in place would keep serving the Ubuntu default page.
 
### 2.3 Handler: `roles/webserver/handlers/main.yml`
 
```yaml
---
- name: Restart Apache
  service:
    name: apache2
    state: restarted
  become: true
```
 
---
 
## Step 3: The loadbalancer Role
 
This role runs Apache again, but only as a **reverse proxy** that balances traffic across the web node(s).
 
### 3.1 Template: `roles/loadbalancer/templates/loadbalancer.conf.j2`
 
```apache
<VirtualHost *:80>
  ProxyRequests     Off
 
  <Proxy "balancer://webcluster">
    {% for host in groups['webserver'] %}
    BalancerMember http://{{ hostvars[host]['ansible_host'] }}:80
    {% endfor %}
    ProxySet lbmethod=byrequests
  </Proxy>
 
  ProxyPass        / "balancer://webcluster/"
  ProxyPassReverse / "balancer://webcluster/"
  ErrorLog  ${APACHE_LOG_DIR}/lb-error.log
  CustomLog ${APACHE_LOG_DIR}/lb-access.log combined
</VirtualHost>
```
 
| Element | Purpose |
|---|---|
| `{% for host in groups['webserver'] %}` | Loops over every host in the `[webserver]` inventory group, generating one `BalancerMember` line each |
| `hostvars[host]['ansible_host']` | Looks up each web server's address from the inventory |
| `ProxySet lbmethod=byrequests` | Distributes requests evenly in turn (round robin by request count) |
| `ProxyPass` / `ProxyPassReverse` | Forward requests to the cluster and rewrite response headers on the way back |
 
Adding another host to `[webserver]` automatically adds it to the balancer; no template edit needed.
 
### 3.2 Tasks: `roles/loadbalancer/tasks/main.yml`
 
```yaml
---
- name: Install Apache
  apt:
    name: apache2
    state: present
    update_cache: yes
  become: true
 
- name: Enable required proxy modules
  command: a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests
  become: true
  register: a2enmod_result
  notify: Restart Apache
  changed_when: "'Enabling module' in a2enmod_result.stdout"
 
- name: Deploy load balancer vhost
  template:
    src: loadbalancer.conf.j2
    dest: /etc/apache2/sites-available/000-default.conf
    mode: '0644'
  become: true
  notify: Restart Apache
```
 
`changed_when` keeps the `command` task idempotent: `a2enmod` only reports `Enabling module` on the first run, so later runs show `ok` instead of `changed` and do not trigger a restart.
 
### 3.3 Handler: `roles/loadbalancer/handlers/main.yml`
 
```yaml
---
- name: Restart Apache
  service:
    name: apache2
    state: restarted
  become: true
```
 
---
 
## Step 4: Inventory and Configuration
 
### `playbooks/hosts`
 
```ini
[webserver]
web1 ansible_host=<WEB_SERVER_IP> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/mykey.pem
 
[loadbalancer]
lb1  ansible_host=<LB_IP>         ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/mykey.pem
```
 
### `playbooks/ansible.cfg`
 
```ini
[defaults]
inventory  = hosts
roles_path = ../roles
host_key_checking = False
 
[privilege_escalation]
become = True
```
 
> `host_key_checking = False` is convenient for a lab but should not be used in production. The `become = True` setting makes the per-task `become: true` lines redundant, which is harmless.
 
---
 
## Step 5: The Playbook
 
### `playbooks/site.yml`
 
```yaml
---
- name: Configure Web Servers
  hosts: webserver
  roles:
    - webserver
 
- name: Configure Load Balancer
  hosts: loadbalancer
  roles:
    - loadbalancer
```
 
Because `roles_path` is set, roles are referenced by name rather than by relative path.
 
---
 
## Step 6: Run and Verify
 
```bash
cd playbooks
ansible-playbook site.yml
```
 
On success you see two plays, each ending with `ok=...` and `failed=0`.
 
Test from your workstation:
 
```bash
curl http://<LB_IP>/
```
 
Or open `http://<LB_IP>/` in a browser. It should display the **Hello, World!** page.
 
### Additional checks
 
```bash
# Run the playbook a second time; expect changed=0 (idempotency)
ansible-playbook site.yml
 
# Confirm the web server works directly
curl http://<WEB_SERVER_IP>/
 
# Validate the Apache config on the load balancer
ansible loadbalancer -a "apache2ctl configtest"
 
# Inspect the rendered balancer config
ansible loadbalancer -a "cat /etc/apache2/sites-available/000-default.conf"
 
# Stop Apache on the web server; the load balancer should answer 503
ansible webserver -b -a "systemctl stop apache2"
curl -i http://<LB_IP>/
 
# Start it again; the Hello World page should return
ansible webserver -b -a "systemctl start apache2"
```
 
A `503 Service Unavailable` while the web server is stopped, and the page after it starts again, confirms the response is coming through the proxy from the backend.
 
---
 
## How It Works
 
1. **Roles keep concerns separate.** `webserver` knows how to run Apache with PHP; `loadbalancer` knows how to run Apache as a proxy. The playbook only says which hosts get which role.
2. **Handlers avoid needless restarts.** `notify` fires only when a task changes something, and handlers run once at the end of the play.
3. **The template reads the inventory.** The load balancer configuration is built from the `[webserver]` group at run time, so scaling out is an inventory change.
4. **Vhost replacement.** Writing to `sites-available/000-default.conf` works because Ubuntu already links that file into `sites-enabled`, so the proxy configuration becomes the active site.
---
 
## Security Notes
 
- Never commit `.pem` files; add `*.pem` to `.gitignore`.
- Open only the ports you need: SSH (22) from your IP, and HTTP (80) on the load balancer. Once testing is done, restrict port 80 on the web server to the load balancer's security group.
- Use private IPs for `ansible_host` in the balancer template when the VMs share a network, so backend traffic stays off the public internet.
- Do not share screenshots that show key paths or public IPs.
---
 
## Troubleshooting
 
| Symptom | Quick check |
|---|---|
| `Permission denied (publickey)` | Is the key path correct? Is the key file mode `chmod 600` (or `400`)? Is `ansible_user` right for the image? |
| Apache fails to start | Run `apache2ctl configtest` on the target and read the reported line |
| Response does not come through the load balancer (default Apache page, or `503`) | Confirm the proxy modules are enabled (`apache2ctl -M \| grep proxy`), check `/etc/apache2/sites-enabled/`, and confirm the backend answers directly |
| `503 Service Unavailable` | The backend is down or unreachable from the load balancer; check Apache on the web server and the security group |
| Default Ubuntu page instead of Hello World on the web server | `index.html` was not removed, or `php` is not installed; re-run the role |
| PHP source displayed instead of executed | The PHP module is not active; re-run the web server play and restart Apache |
| `roles/webserver` not found | Run from the `playbooks/` directory so `ansible.cfg` and `roles_path` are picked up |
| Logs | `/var/log/apache2/lb-error.log` and `lb-access.log` on the load balancer |
 
---
 
## Possible Improvements
 
- Add a **second web server** to `[webserver]` and make the page show `<?php echo gethostname(); ?>` so you can watch requests alternate between backends.
- Move values (ports, balancing method) into `defaults/main.yml` so they can be overridden per environment.
- Replace the `a2enmod` command with the `community.general.apache2_module` module.
- Use fully qualified module names (`ansible.builtin.apt`, `ansible.builtin.template`) and run `ansible-lint`.
- Add health checks and sticky sessions to the balancer, or use HTTPS with Let's Encrypt.
- Test the roles with Molecule.
---
 
## Author
 
**Gerald Oti** - DevOps / Platform / SRE Engineer, Berlin
 
- GitHub: [goti13](https://github.com/goti13)
- LinkedIn: [gerald-oti](https://www.linkedin.com/in/gerald-oti/)
