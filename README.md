# Ansible Mid-Level Lab — BlueWave Systems

![Ansible](https://img.shields.io/badge/Ansible-2.20.x-red)
![License](https://img.shields.io/badge/License-MIT-blue)
![Platform](https://img.shields.io/badge/Platform-Linux-lightgrey)

Automated deployment and configuration of a multi-server environment using **Ansible**.

The project demonstrates how Ansible can be used to configure a Web Server and a Database Server, manage packages and services, deploy dynamic content using Jinja2, and create application-level users, groups, and directories.

---

## Quick Start

Clone the repository:

```bash
git clone https://github.com/M0az2/ansible-infrastructure-automation.git
cd ansible-infrastructure-automation
```

Run the playbook:

```bash
ansible-playbook site.yml
```

> **Note:** This project was developed and tested in a **KodeKloud hands-on lab environment**. The committed inventory does not contain lab credentials or secrets.

---

## Project Overview

**BlueWave Systems** requires an automated infrastructure setup for a simple internal web application.

The environment consists of two Linux servers:

* `node01` — Web Server
* `node02` — Database Server

Ansible is used to automate the complete configuration process.

The project was developed and tested in a **KodeKloud hands-on lab environment**, which provided the Linux servers used as managed nodes.

---

## Architecture

```text
                 KodeKloud Lab Environment
                           │
                           │
                  Ansible Control Node
                           │
                    Ansible Playbook
                       site.yml
                      /         \
                     /           \
                node01          node02
             Web Server      Database Server
                  │                 │
                Nginx             MariaDB
```

### Server Roles

| Host     | Role            | Software |
| -------- | --------------- | -------- |
| `node01` | Web Server      | Nginx    |
| `node02` | Database Server | MariaDB  |

---

## Objectives

The project demonstrates the following Ansible concepts:

* Inventory management
* Project-level Ansible configuration
* Multi-play playbooks
* Group variables
* Jinja2 templates
* Package management
* Service management
* File management
* User and group management
* Privilege escalation using `become`
* Ansible builtin modules
* Infrastructure automation
* Ansible best practices

---

## Project Structure

```text
ansible-infrastructure-automation/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
│   ├── web_server.yml
│   └── database_server.yml
├── templates/
│   └── index.html.j2
├── files/
│   └── welcome.txt
├── .gitignore
├── LICENSE
└── README.md
```

### File Description

| File / Directory | Purpose                                                 |
| ---------------- | ------------------------------------------------------- |
| `ansible.cfg`    | Project-level Ansible configuration                     |
| `inventory.ini`  | Defines managed hosts and groups                        |
| `site.yml`       | Main multi-play automation                              |
| `group_vars/`    | Group-specific configuration variables                  |
| `templates/`     | Jinja2 templates                                        |
| `files/`         | Static files deployed to managed hosts                  |
| `.gitignore`     | Prevents unwanted or sensitive files from being tracked |
| `LICENSE`        | Defines the project usage and distribution terms        |
| `README.md`      | Project documentation                                   |

---

## Inventory

The inventory separates the servers into two logical groups:

```ini
node01 ansible_host=node01
node02 ansible_host=node02

[web_server]
node01

[database_server]
node02
```

This allows the playbook to target each server according to its role.

> The inventory committed to this repository is sanitized and does not contain lab credentials.

---

## Ansible Configuration

The project uses a local `ansible.cfg`:

```ini
[defaults]
host_key_checking = False
inventory = ./inventory.ini
```

This allows Ansible to automatically use the project inventory without requiring the `-i` option.

> `host_key_checking = False` is used here for the controlled lab environment. In production environments, SSH host key verification should be handled securely.

---

## Group Variables

Configuration is separated from the playbook logic using `group_vars`.

### Web Server Variables

`group_vars/web_server.yml`

```yaml
package_name: "nginx"
webstate: "present"

web_service: "nginx"
web_root: "/usr/share/nginx/html"
app_directory: "/opt/bluewave"

app_user: "bluewave"
app_group: "bluewave"
```

These variables control the Web Server configuration.

### Database Server Variables

`group_vars/database_server.yml`

```yaml
db_package: "mariadb-server"
dbstate: "present"

db_service: "mariadb"
```

These variables control the Database Server configuration.

Separating variables from the playbook makes the automation easier to maintain and reuse.

---

# Playbook

The main automation is contained in:

```text
site.yml
```

The playbook contains two plays.

---

## 1. Web Server Configuration

The Web Server play targets:

```yaml
hosts: web_server
```

### Install Nginx

```yaml
- name: Install Nginx
  ansible.builtin.dnf:
    name: "{{ package_name }}"
    state: "{{ webstate }}"
```

The `dnf` module installs Nginx using variables defined in `group_vars`.

---

### Start and Enable Nginx

```yaml
- name: Start Nginx service
  ansible.builtin.service:
    name: "{{ web_service }}"
    state: started
    enabled: true
```

This ensures that Nginx:

* is running
* starts automatically after reboot

---

## Deploy Dynamic Web Page

The project uses the Ansible `template` module:

```yaml
- name: Deploy web page
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ web_root }}/index.html"
    mode: '0644'
```

The Jinja2 template is rendered on the managed server.

The template uses variables such as:

```jinja2
{{ inventory_hostname }}
{{ package_name }}
```

This makes the generated page dynamic.

For example, the page can display:

```text
Server: node01
Web Server: nginx
Deployment: Ansible
Status: Operational
```

---

## Deploy Static File

The `copy` module is used to deploy:

```text
files/welcome.txt
```

to the Nginx web root:

```yaml
- name: Copy welcome file
  ansible.builtin.copy:
    src: welcome.txt
    dest: "{{ web_root }}/welcome.txt"
    mode: '0644'
```

---

## Create Application Directory

An application directory is created using the `file` module:

```yaml
- name: Create application directory
  ansible.builtin.file:
    path: "{{ app_directory }}"
    state: directory
    mode: '0755'
```

The resulting directory is:

```text
/opt/bluewave
```

---

## Create Application Group

The `group` module creates:

```yaml
- name: Create BlueWave group
  ansible.builtin.group:
    name: "{{ app_group }}"
    state: present
```

Result:

```text
bluewave
```

---

## Create Application User

The `user` module creates a dedicated application user:

```yaml
- name: Create BlueWave user
  ansible.builtin.user:
    name: "{{ app_user }}"
    group: "{{ app_group }}"
    state: present
    create_home: true
```

Result:

```text
User:  bluewave
Group: bluewave
```

---

# Database Server Configuration

The second play targets:

```yaml
hosts: database_server
```

### Install MariaDB

```yaml
- name: Install MariaDB
  ansible.builtin.dnf:
    name: "{{ db_package }}"
    state: "{{ dbstate }}"
```

---

### Start and Enable MariaDB

```yaml
- name: Start MariaDB service
  ansible.builtin.service:
    name: "{{ db_service }}"
    state: started
    enabled: true
```

This ensures that MariaDB is running and enabled at boot.

---

# Jinja2 Template

The project uses:

```text
templates/index.html.j2
```

The template generates a styled BlueWave Systems web page.

Dynamic Ansible variables are embedded inside the HTML:

```jinja2
{{ inventory_hostname }}
{{ package_name }}
```

This demonstrates how Ansible can generate host-specific configuration and content.

---

# Ansible Modules Used

The project demonstrates the following Ansible builtin modules:

| Module                     | Purpose                 |
| -------------------------- | ----------------------- |
| `ansible.builtin.dnf`      | Install packages        |
| `ansible.builtin.service`  | Manage services         |
| `ansible.builtin.template` | Deploy Jinja2 templates |
| `ansible.builtin.copy`     | Copy static files       |
| `ansible.builtin.file`     | Manage directories      |
| `ansible.builtin.user`     | Manage users            |
| `ansible.builtin.group`    | Manage groups           |

---

# Result Handling

Ansible provides the `register` feature for storing the result of a task:

```yaml
register: task_result
```

The stored result can then be inspected using:

```yaml
ansible.builtin.debug:
  var: task_result
```

These features were studied as part of the lab and are useful for debugging, conditional automation, and inspecting task results.

A registered result can contain information such as:

```text
changed
failed
rc
stdout
stderr
```

---

# Verification

The automation was successfully executed and verified in the KodeKloud lab environment.

### Playbook Result

The final execution completed successfully:

```text
node01 : ok=9 changed=2 unreachable=0 failed=0 skipped=0 ignored=0
node02 : ok=4 changed=0 unreachable=0 failed=0 skipped=0 ignored=0
```

### Nginx Verification

Nginx was verified as active:

```text
node01 | CHANGED | rc=0 >>
active
```

### Dynamic HTML Verification

The generated page displayed:

```text
Welcome to BlueWave Systems
Server: node01
Web Server: Nginx
Deployment: Ansible
Status: Operational
```

### Static File Verification

The deployed file contained:

```text
Welcome to BlueWave Systems!
This server is managed by Ansible.
```

### Application User Verification

The `bluewave` user was successfully created:

```text
uid=1001(bluewave) gid=1001(bluewave) groups=1001(bluewave)
```

---

# Best Practices Applied

The project follows several Ansible best practices:

* Fully qualified module names are used.
* Variables are separated from playbook logic.
* Group-specific configuration is stored in `group_vars`.
* Jinja2 is used for dynamic content.
* Meaningful task names are used.
* Privilege escalation is used where required.
* The project follows a clear directory structure.
* Sensitive credentials are excluded from the final repository.
* Project-level configuration is used through `ansible.cfg`.

---

# Security Considerations

The KodeKloud lab environment used credentials for accessing the managed nodes.

**Credentials are intentionally excluded from this repository.**

The inventory committed to GitHub contains no passwords, private keys, API keys, or other sensitive information.

For production environments, secrets should be managed using secure solutions such as:

* Ansible Vault
* Environment variables
* Secret management systems
* CI/CD secret stores

---

# Environment and Testing

The project was implemented and tested using a **KodeKloud hands-on lab environment**.

KodeKloud provided the Linux infrastructure used during the practical execution of the Ansible automation.

The final project was then organized separately into a clean repository structure for documentation and version control.

---

# Learning Outcomes

After completing this project, the following Ansible concepts were practiced:

* Inventory management
* Group-based host organization
* Ansible configuration
* Multi-play playbooks
* Group variables
* Jinja2 templating
* Package installation
* Service management
* File deployment
* User and group management
* Privilege escalation
* Result registration concepts
* Debugging concepts
* Infrastructure automation
* Basic Ansible project organization

---

# Conclusion

This project demonstrates a practical approach to automating a multi-server environment using Ansible.

Instead of manually configuring each server, the infrastructure configuration is represented as code and can be applied consistently through an Ansible playbook.

The project provides a foundation for extending the environment with additional features such as:

* Database configuration
* Firewall management
* Nginx virtual hosts
* Application deployment
* Ansible Vault
* Handlers
* Roles
* Environment-specific variables
* CI/CD automation

---

## License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.
