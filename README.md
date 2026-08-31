# Multi-Tier Ansible Lab

A simulated 3-tier infrastructure — **load balancer → web servers → database** —
fully provisioned, configured, and secured with **Ansible**, running on **Docker
containers acting as virtual machines**.

Built as a hands-on learning project to practice real Ansible patterns: roles,
inventories, Jinja2 templating, Ansible Vault, tags, handlers, and dynamic
upstream configuration — the same patterns used to manage real fleets of
servers in production.

---

##  Architecture

```
                          Client
                            │
                            ▼
                  ┌───────────────────┐
                  │       lb01        │
                  │  nginx (reverse   │
                  │  proxy / LB)      │
                  │  :8080 → :80      │
                  └─────────┬─────────┘
                            │  round robin
                ┌───────────┴───────────┐
                ▼                       ▼
      ┌───────────────────┐   ┌───────────────────┐
      │       web01        │   │       web02        │
      │  nginx + app page   │   │  nginx + app page   │
      │  :8081 → :80        │   │  :8082 → :80        │
      └──────────┬──────────┘   └──────────┬──────────┘
                 └───────────┬───────────────┘
                             ▼
                  ┌───────────────────┐
                  │        db01        │
                  │  MySQL server      │
                  │  (vault-protected  │
                  │   credentials)     │
                  └───────────────────┘
```

All four nodes are Docker containers on a shared bridge network (`lab-net`),
each reachable over SSH from the control node, and configured entirely
through Ansible playbooks — no manual `docker exec` configuration.

---

##  Tech Stack

| Layer            | Tool                                |
|-------------------|--------------------------------------|
| Provisioning       | Docker + Docker Compose             |
| Configuration mgmt | Ansible                             |
| Load balancer      | nginx (reverse proxy)               |
| Web servers        | nginx                               |
| Database           | MySQL                               |
| Secrets            | Ansible Vault                       |
| Collections        | `community.general`, `community.mysql` |

---

##  Project Structure

```
ansible-multitier-lab/
├── site.yml                  # master playbook
├── ansible.cfg
├── requirements.yml           # collection dependencies
├── docker/
│   ├── Dockerfile.node        # SSH + Python base image for all nodes
│   └── docker-compose.yml     # spins up lb01, web01, web02, db01
├── inventories/
│   ├── staging/
│   │   ├── hosts.ini
│   │   └── group_vars/
│   └── prod/
│       ├── hosts.ini
│       └── group_vars/
└── roles/
    ├── common/                # base packages, sudo user, firewall (all hosts)
    ├── webserver/              # nginx + templated app page (web group)
    ├── database/               # MySQL install + secured app database (db group)
    └── loadbalancer/           # nginx reverse proxy, dynamic upstream config (lb group)
```

---

##  Key Features

- **Role-based design** — each tier (`common`, `webserver`, `database`,
  `loadbalancer`) is a self-contained, reusable Ansible role.
- **Dynamic load balancer config** — the nginx upstream block is generated
  from a Jinja2 template that loops over the live `[web]` inventory group:
  ```nginx
  upstream backend {
      {% for host in groups['web'] %}
      server {{ host }}:80;
      {% endfor %}
  }
  ```
  Add a `web03` to the inventory and rerun — the load balancer picks it up
  automatically, no manual config editing required.
- **Secrets management** — DB root password, app DB credentials, and the
  sudo user's password are stored encrypted with **Ansible Vault**, never
  committed in plaintext.
- **Multi-environment support** — `inventories/staging` and
  `inventories/prod` share the same roles/playbook but supply different
  variables (e.g. page titles, environment labels) per environment.
- **Tags** — `install` vs `config` tags let you re-run just the config
  portion of a role (e.g. `--tags config`) without reinstalling packages
  every time.
- **Idempotent throughout** — every task can be run repeatedly with no
  unintended side effects, verified by re-running the full playbook and
  confirming `changed=0` on a clean state.

---

##  Prerequisites

- Docker & Docker Compose
- Ansible ≥ 2.14
- `sshpass` (for password-based SSH auth in this lab)

Install required Ansible collections:

```bash
ansible-galaxy install -r requirements.yml
```

---

## Usage

**1. Spin up the containers**

```bash
cd docker
docker compose up -d --build
```

**2. Run the full playbook against staging**

```bash
ansible-playbook -i inventories/staging/hosts.ini site.yml \
  --vault-password-file ~/.vault_pass.txt
```

**3. See the load balancer in action**

```bash
curl http://localhost:8080
curl http://localhost:8080
curl http://localhost:8080
```

Watch the `Served by:` field alternate between `web01` and `web02` —
proof the reverse proxy is round-robining across both backends.

**4. Run against production instead**

```bash
ansible-playbook -i inventories/prod/hosts.ini site.yml \
  --vault-password-file ~/.vault_pass.txt
```

Same roles, same playbook — only the environment's variables differ.

**5. Run only a subset with tags**

```bash
ansible-playbook -i inventories/staging/hosts.ini site.yml \
  --tags config --vault-password-file ~/.vault_pass.txt
```

---

##  Secrets

Sensitive variables live in `roles/*/vars/secrets.yml`, encrypted with
Ansible Vault. To view or edit:

```bash
ansible-vault edit roles/database/vars/secrets.yml
```

The plaintext vault password file (`~/.vault_pass.txt`) is **never**
committed — see `.gitignore`.

---

## Notes on Containers vs. Real VMs

A couple of deliberate design decisions worth calling out, since Docker
containers behave differently from real VMs in ways that matter for
config management:

- **Firewall management (`ufw`)** — gated behind an `enable_firewall`
  variable (default `false`). Containers lack the `NET_ADMIN` capability
  by default, so kernel-level firewall rules can't be managed from inside
  one. In real cloud environments, firewalling is typically handled at the
  security-group/cloud-firewall level rather than per-instance anyway —
  the toggle demonstrates the pattern without requiring
  `--cap-add=NET_ADMIN` on every container.
- **MySQL without systemd** — containers don't run a full init system, so
  the `database` role relies on Ubuntu's `mysql-server` package to
  self-initialize via `apt`, and authenticates the very first admin action
  through the auto-generated `debian-sys-maint` account rather than
  assuming a passwordless root — mirroring how Debian/Ubuntu actually
  manages MySQL internally.

---

## What This Project Demonstrates

- Ansible roles, playbooks, inventories, and `ansible.cfg`
- Jinja2 templating driven by live inventory data (`groups`, `hostvars`)
- Handlers and notify-based restarts
- Tags for selective execution
- Ansible Vault for secrets
- Multi-environment variable layering
- Using community collections (`community.general`, `community.mysql`)
  for functionality beyond `ansible.builtin`
- Debugging real-world container quirks (init systems, capabilities,
  auth plugins) rather than idealized textbook conditions
