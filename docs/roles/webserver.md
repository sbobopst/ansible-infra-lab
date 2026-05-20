# Role: webserver

Installs and configures **nginx** on web server hosts.  
Deploys a virtual host with a custom landing page showing host information.

---

## What it does

| Task | Description |
|------|-------------|
| Install nginx | Installs nginx via apt |
| Enable nginx | Ensures nginx starts on boot and is running |
| Deploy virtual host | Configures nginx virtual host from Jinja2 template |
| Enable virtual host | Creates symlink in sites-enabled |
| Disable default site | Removes the default nginx placeholder site |
| Create web root | Creates the document root directory |
| Deploy index page | Deploys a custom HTML page with host information |

---

## Variables

Defined in `roles/webserver/defaults/main.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `nginx_vhost_name` | `lab` | Virtual host config filename |
| `nginx_web_root` | `/var/www/lab` | Document root directory |
| `nginx_server_name` | `{{ ansible_hostname }}` | nginx server_name directive |
| `nginx_port` | `80` | Port nginx listens on |

---

## Templates

**`nginx.conf.j2`** — nginx virtual host configuration  
**`index.html.j2`** — Landing page showing hostname, IP, OS and provisioning info

---

## Handlers

| Handler | Triggered by | Action |
|---------|-------------|--------|
| Reload nginx | Virtual host config changes | `systemctl reload nginx` |
| Restart nginx | - | `systemctl restart nginx` |

---

## Usage

Run the dedicated playbook:
```bash
ansible-playbook playbooks/webserver.yml
```

Or target a specific host:
```bash
ansible-playbook playbooks/webserver.yml --limit webserver01
```

Verify nginx is running:
```bash
curl http://192.168.56.10
```

---

## Technical notes

**Handlers** — nginx is reloaded (not restarted) when configuration changes. Reload applies new config without dropping active connections, which is the correct approach for production-like environments.

**Jinja2 templates** — The virtual host config and HTML page use Ansible facts (`ansible_hostname`, `ansible_default_ipv4.address`, `ansible_distribution`) to populate host-specific values automatically at deploy time.

**Idempotency** — All tasks are idempotent. Re-running the playbook produces no changes if nginx is already correctly configured.
