# Role: database

Installs and configures **PostgreSQL** on database server hosts.  
Creates an application database and user with credentials managed via Ansible Vault.

---

## What it does

| Task | Description |
|------|-------------|
| Install PostgreSQL | Installs postgresql, postgresql-contrib, python3-psycopg2 |
| Enable PostgreSQL | Ensures PostgreSQL starts on boot and is running |
| Configure listen address | Sets the interface PostgreSQL listens on |
| Deploy pg_hba.conf | Configures client authentication from Jinja2 template |
| Create database | Creates the application database |
| Create user | Creates the application user with hashed password |

---

## Requirements

- Collection: `community.postgresql`
- `vault/secrets.yml` encrypted with Ansible Vault

Install the required collection:
```bash
ansible-galaxy collection install community.postgresql
```

---

## Variables

Defined in `roles/database/defaults/main.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `postgresql_version` | `14` | PostgreSQL version |
| `postgresql_packages` | `[postgresql, postgresql-contrib, python3-psycopg2]` | Packages to install |
| `postgresql_listen_addresses` | `localhost` | Interfaces PostgreSQL listens on |
| `postgresql_db_name` | `labdb` | Application database name |
| `postgresql_db_user` | `labuser` | Application database user |
| `postgresql_db_password` | `{{ vault_postgresql_db_password }}` | Password from Vault |

---

## Ansible Vault

The database password is stored encrypted in `vault/secrets.yml`.

**First time setup — encrypt the vault:**
```bash
ansible-vault encrypt vault/secrets.yml
```

**Edit the vault:**
```bash
ansible-vault edit vault/secrets.yml
```

**Run the playbook with vault password:**
```bash
ansible-playbook playbooks/database.yml --ask-vault-pass
```

---

## Templates

**`pg_hba.conf.j2`** — PostgreSQL client authentication configuration.  
Controls which users can connect, from where, and with which authentication method.

---

## Handlers

| Handler | Triggered by | Action |
|---------|-------------|--------|
| Restart postgresql | pg_hba.conf or listen_address changes | `systemctl restart postgresql` |
| Reload postgresql | - | `systemctl reload postgresql` |

---

## Usage

**Encrypt the vault first:**
```bash
ansible-vault encrypt vault/secrets.yml
```

**Run the playbook:**
```bash
ansible-playbook playbooks/database.yml --ask-vault-pass
```

**Verify PostgreSQL is running:**
```bash
vagrant ssh dbserver01
sudo -u postgres psql -c "\l"
```

---

## Technical notes

**python3-psycopg2** — Required by Ansible's `community.postgresql` modules to interact with PostgreSQL. Without it, the postgresql_db and postgresql_user modules cannot run.

**no_log: true** — The user creation task uses `no_log: true` to prevent the password from appearing in Ansible output or logs.

**pg_hba.conf** — Managed as a Jinja2 template to ensure only the application user can connect to the application database, following the principle of least privilege.

**Ansible Vault** — Passwords are never stored in plaintext in the repository. The `vault/secrets.yml` file must always be encrypted before committing.
