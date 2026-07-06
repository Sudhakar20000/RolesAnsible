# RolesAnsible — Infrastructure Automation

> Ansible roles-based automation to provision, configure, and manage the full e-commerce application stack on AWS.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Components](#components)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Usage](#usage)
  - [1. Provision AWS Infrastructure](#1-provision-aws-infrastructure)
  - [2. Configure a Component](#2-configure-a-component)
- [Secrets Management](#secrets-management)
- [Inventory](#inventory)
- [Group Variables](#group-variables)
- [Roles](#roles)
- [Author](#author)

---

## Overview

This repository automates the end-to-end setup of the microservices-based e-commerce platform. It uses Ansible roles to install, configure, and connect all application services — databases, message queues, microservices, and the frontend — across AWS EC2 instances.

Infrastructure is provisioned dynamically via `aws_setup.yaml`, DNS records are managed via AWS Route 53, and each service is configured using a dedicated Ansible role invoked through the generic `component.yaml` playbook.

---

## Architecture

```
                         Internet
                             |
                       [ frontend ]
                       sudhakar.shop
                             |
          ┌──────────────────┼──────────────────┐
          |                  |                  |
     [catalogue]          [cart]             [user]
          |                  |                  |
          └──────────┬────────────────┐─────────┘
                     |                |
                 [payment]        [shipping]
                     |
        ┌────────────┼─────────────┐
        |            |             |
    [mongodb]    [mysql]       [redis]
                              [rabbitmq]
```

All internal services communicate via private DNS records under `*.sudhakar.shop`. The frontend is the only service exposed via a public IP.

---

## Repository Structure

```
RolesAnsible/
├── ansible.cfg              # Ansible configuration (inventory path)
├── inventory.ini            # Static inventory — all hosts grouped by service
├── aws_setup.yaml           # Playbook: provision/delete EC2 instances + Route53 records
├── component.yaml           # Generic playbook: apply a role by ROLENAME variable
├── group_vars/              # Group-level variables (applied to inventory groups)
├── roles/                   # Individual Ansible roles for each service
│   ├── mongodb/
│   ├── redis/
│   ├── mysql/
│   ├── rabbitmq/
│   ├── catalogue/
│   ├── user/
│   ├── cart/
│   ├── shipping/
│   ├── payment/
│   └── frontend/
└── secrets_ansible/         # Ansible Vault encrypted secrets
```

---

## Components

| Service | Host | Type | Description |
|---|---|---|---|
| `mongodb` | mongodb.sudhakar.shop | Database | Document store for product catalogue |
| `redis` | redis.sudhakar.shop | Cache | Session/cart caching |
| `mysql` | mysql.sudhakar.shop | Database | Relational data (user, shipping) |
| `rabbitmq` | rabbitmq.sudhakar.shop | Message Queue | Async messaging between services |
| `catalogue` | catalogue.sudhakar.shop | Microservice | Product listing API |
| `user` | user.sudhakar.shop | Microservice | User auth and profile management |
| `cart` | cart.sudhakar.shop | Microservice | Cart operations |
| `shipping` | shipping.sudhakar.shop | Microservice | Shipping cost calculation |
| `payment` | payment.sudhakar.shop | Microservice | Payment processing |
| `frontend` | sudhakar.shop | Web UI | Nginx-served frontend (public IP) |

---

## Prerequisites

- **Ansible** ≥ 2.12 installed on the control node
- **Python** ≥ 3.8
- **AWS credentials** configured (`~/.aws/credentials` or environment variables)
- `amazon.aws` Ansible collection installed:
  ```bash
  ansible-galaxy collection install amazon.aws
  ```
- SSH key pair configured for EC2 access
- AWS Route 53 hosted zone for `sudhakar.shop` already created

---

## Configuration

### `ansible.cfg`

```ini
[defaults]
inventory = ./inventory.ini
```

Points Ansible to the local static inventory file. Override if needed with `-i` at runtime.

### `inventory.ini`

Groups each service as a separate inventory group. All hosts follow the pattern `<service>.sudhakar.shop`:

```ini
[local]
localhost

[catalogue]
catalogue.sudhakar.shop

[cart]
cart.sudhakar.shop

[user]
user.sudhakar.shop

[shipping]
shipping.sudhakar.shop

[payment]
payment.sudhakar.shop

[mongodb]
mongodb.sudhakar.shop

[redis]
redis.sudhakar.shop

[mysql]
mysql.sudhakar.shop

[rabbitmq]
rabbitmq.sudhakar.shop

[frontend]
frontend.sudhakar.shop
```

---

## Usage

### 1. Provision AWS Infrastructure

`aws_setup.yaml` creates or deletes all EC2 instances and their Route53 DNS records in one run.

**Create all instances:**
```bash
ansible-playbook aws_setup.yaml -e action=create
```

**Delete all instances:**
```bash
ansible-playbook aws_setup.yaml -e action=delete
```

What this playbook does under the hood:

- Loops over all 10 services and creates a `t3.micro` EC2 instance per service using AMI `ami-0220d79f3f480ecf5`
- Assigns security groups `roboshop-<service>` and `roboshop-common` to each instance
- Creates Route53 A records pointing `<service>.sudhakar.shop` → private IP
- Creates an additional Route53 A record for `sudhakar.shop` → frontend public IP
- On delete, removes all DNS records and terminates all instances

### 2. Configure a Component

`component.yaml` is a single reusable playbook that applies the correct role based on a `ROLENAME` variable:

```yaml
- name: configure="{{ ROLENAME }}"
  hosts: "{{ ROLENAME }}"
  become: yes
  roles:
    - "{{ ROLENAME }}"
```

**Run it for a specific service:**
```bash
ansible-playbook component.yaml -e ROLENAME=mongodb
ansible-playbook component.yaml -e ROLENAME=catalogue
ansible-playbook component.yaml -e ROLENAME=frontend
```

**Configure all services in dependency order:**
```bash
# Databases & queues first
ansible-playbook component.yaml -e ROLENAME=mongodb
ansible-playbook component.yaml -e ROLENAME=redis
ansible-playbook component.yaml -e ROLENAME=mysql
ansible-playbook component.yaml -e ROLENAME=rabbitmq

# Application microservices next
ansible-playbook component.yaml -e ROLENAME=catalogue
ansible-playbook component.yaml -e ROLENAME=user
ansible-playbook component.yaml -e ROLENAME=cart
ansible-playbook component.yaml -e ROLENAME=shipping
ansible-playbook component.yaml -e ROLENAME=payment

# Frontend last
ansible-playbook component.yaml -e ROLENAME=frontend
```

---

## Secrets Management

Sensitive values (database passwords, API keys, etc.) are stored in the `secrets_ansible/` directory using **Ansible Vault** encryption.

**Decrypt and view a secret:**
```bash
ansible-vault view secrets_ansible/main.yaml
```

**Edit a secret:**
```bash
ansible-vault edit secrets_ansible/main.yaml
```

**Run a playbook with vault password:**
```bash
ansible-playbook component.yaml -e ROLENAME=mysql --ask-vault-pass
```

Or use a vault password file:
```bash
ansible-playbook component.yaml -e ROLENAME=mysql --vault-password-file ~/.vault_pass
```

> **Note:** Never commit plaintext secrets. Keep your vault password file outside the repository and out of version control.

---

## Group Variables

The `group_vars/` directory holds variables that apply to all hosts in an inventory group. For example, `group_vars/all.yaml` applies to every host, while `group_vars/catalogue.yaml` applies only to the `[catalogue]` group.

Common variables typically defined here include:
- Application port numbers
- Inter-service hostnames (e.g., `mongodb_host: mongodb.sudhakar.shop`)
- Package versions
- Service user names

---

## Roles

Each role in `roles/` follows the standard Ansible role structure:

```
roles/<service>/
├── tasks/
│   └── main.yaml     # Installation and configuration steps
├── templates/         # Jinja2 config file templates
├── vars/
│   └── main.yaml     # Role-specific variables
├── defaults/
│   └── main.yaml     # Default variable values
├── handlers/
│   └── main.yaml     # Service restart handlers
└── files/             # Static files to be copied
```

The repo is written entirely in **Jinja2** (100% of the language breakdown), which means templates are central to how config files are dynamically generated per environment.

---

## Author

**Sudhakar** — [GitHub: Sudhakar20000](https://github.com/Sudhakar20000)

---

## License

This project is open source. No license file is currently included in the repository.
