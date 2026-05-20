# Role: common

Base system configuration applied to **all hosts** before any other role.  
This role ensures every machine starts from a clean, updated, and consistent state.

---

## What it does

| Task | Description |
|------|-------------|
| Update apt cache | Refreshes the package list (max once per hour) |
| Upgrade all packages | Upgrades all installed packages to latest version |
| Install base packages | Installs common tools available on every host |
| Set timezone | Configures system timezone |
| Create ansible user | Creates a dedicated user for Ansible operations |
| Remove useless packages | Cleans up unused packages |

---

## Requirements

- Collection: `community.general >= 8.0.0`

Install the required collection:
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Variables

Defined in `roles/common/defaults/main.yml` — all variables can be overridden at inventory or playbook level.

| Variable | Default | Description |
|----------|---------|-------------|
| `common_packages` | `[curl, vim, git, htop, unzip, wget, net-tools, python3-pip]` | Packages installed on every host |
| `ansible_deploy_user` | `ansible` | Dedicated user created for Ansible operations |
| `timezone` | `Europe/Rome` | System timezone (defined in `group_vars/all.yml`) |

---

## Usage

This role is included in the master playbook `playbooks/site.yml` and runs on all hosts:

```yaml
- name: Apply common configuration to all hosts
  hosts: all
  become: true
  roles:
    - common
```

Run it with:
```bash
ansible-playbook playbooks/site.yml
```

Or target a specific host/group:
```bash
ansible-playbook playbooks/site.yml --limit webserver01
ansible-playbook playbooks/site.yml --limit webservers
```

---

## Technical notes

**Idempotency** — The role is fully idempotent. Running it multiple times produces no changes if the system is already in the desired state. The `cache_valid_time: 3600` parameter prevents unnecessary apt cache updates within the same hour.

**community.general.timezone** — The timezone module from `community.general` is used instead of `ansible.builtin.command` because it natively checks the current timezone before making changes, ensuring clean idempotent behavior.

**ansible_deploy_user** — A dedicated `ansible` user is created on every host with sudo privileges. This separates Ansible operations from the `vagrant` user, following the principle of least privilege.