# Role: hardening

Applies security hardening to **all hosts**.  
Covers SSH configuration, firewall rules, brute-force protection, automatic security updates and kernel-level network hardening.

---

## What it does

| Area | Task | Description |
|------|------|-------------|
| SSH | Configure sshd | Deploys hardened sshd_config from template |
| SSH | Disable root login | `PermitRootLogin no` |
| SSH | Disable password auth | `PasswordAuthentication no` — keys only |
| SSH | Limit auth attempts | `MaxAuthTries 3` |
| Firewall | Install UFW | Installs Uncomplicated Firewall |
| Firewall | Default deny incoming | Blocks all inbound traffic by default |
| Firewall | Allow SSH | Opens SSH port |
| Firewall | Allow extra ports | Per-group ports (web: 80/443, db: 5432, monitor: 9090/9100) |
| Firewall | Enable UFW | Activates the firewall |
| Fail2ban | Install fail2ban | Brute-force SSH protection |
| Fail2ban | Configure jail | Custom ban time, find time, max retries |
| Updates | unattended-upgrades | Automatic security updates |
| Kernel | sysctl hardening | Disables IP forwarding, ICMP redirects, SYN flood protection |
| Kernel | Disable filesystems | Blacklists unused/dangerous filesystem modules |

---

## Requirements

- Collections: `community.general >= 8.0.0`, `ansible.posix >= 1.5.0`

Install required collections:
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Variables

Defined in `roles/hardening/defaults/main.yml`. All variables can be overridden at group or host level.

### SSH

| Variable | Default | Description |
|----------|---------|-------------|
| `hardening_ssh_port` | `22` | SSH port |
| `hardening_ssh_permit_root_login` | `no` | Disable root SSH login |
| `hardening_ssh_password_authentication` | `no` | Disable password auth |
| `hardening_ssh_max_auth_tries` | `3` | Max failed auth attempts |
| `hardening_ssh_login_grace_time` | `30` | Seconds before unauthenticated connection drops |
| `hardening_ssh_allow_agent_forwarding` | `no` | Disable SSH agent forwarding |
| `hardening_ssh_x11_forwarding` | `no` | Disable X11 forwarding |

### Firewall

| Variable | Default | Description |
|----------|---------|-------------|
| `hardening_allowed_ports` | `[]` | Extra ports to open — defined per group in group_vars |

### Fail2ban

| Variable | Default | Description |
|----------|---------|-------------|
| `hardening_fail2ban_bantime` | `1h` | How long to ban an IP |
| `hardening_fail2ban_findtime` | `10m` | Time window for counting failures |
| `hardening_fail2ban_maxretry` | `5` | Max failed attempts before ban |

---

## Per-group firewall rules

Extra ports are defined in `inventories/lab/group_vars/`:

| Group | File | Ports opened |
|-------|------|-------------|
| webservers | `webservers.yml` | 80/tcp, 443/tcp |
| dbservers | `dbservers.yml` | 5432/tcp |
| monitoring | `monitoring.yml` | 9090/tcp, 9100/tcp |

---

## Templates

| Template | Destination | Description |
|----------|------------|-------------|
| `sshd_config.j2` | `/etc/ssh/sshd_config` | Hardened SSH daemon config |
| `jail.local.j2` | `/etc/fail2ban/jail.local` | Fail2ban SSH jail |
| `50unattended-upgrades.j2` | `/etc/apt/apt.conf.d/50unattended-upgrades` | Auto security updates config |
| `20auto-upgrades.j2` | `/etc/apt/apt.conf.d/20auto-upgrades` | Auto updates schedule |
| `hardening-modprobe.conf.j2` | `/etc/modprobe.d/hardening.conf` | Disable unused filesystems |

---

## Handlers

| Handler | Triggered by | Action |
|---------|-------------|--------|
| Restart sshd | sshd_config changes | `systemctl restart sshd` |
| Restart fail2ban | jail.local changes | `systemctl restart fail2ban` |
| Reload ufw | - | `ufw reload` |

---

## Usage

```bash
ansible-playbook playbooks/hardening.yml
```

Verify after running:
```bash
# Check UFW status on any host
vagrant ssh webserver01
sudo ufw status verbose

# Check fail2ban status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Check SSH config is valid
sudo sshd -t
```

---

## Technical notes

**sshd validate** — The template task uses `validate: /usr/sbin/sshd -t -f %s` to test the SSH config before applying it. If the config is invalid, Ansible rejects it and the SSH daemon is never restarted with a broken config — preventing lockout.

**PasswordAuthentication no** — After hardening, only SSH key authentication works. Make sure your key is in `authorized_keys` before applying, or you will lose access. In this lab, Vagrant handles this automatically.

**Per-group ports** — `hardening_allowed_ports` defaults to an empty list. Each host group defines its own ports in `group_vars/` — webservers open 80/443, dbservers open 5432, monitoring opens 9090/9100. This follows the principle of least privilege.

**sysctl TCP SYN cookies** — Protects against SYN flood attacks by enabling `net.ipv4.tcp_syncookies`. Under a SYN flood, the server sends a cookie instead of allocating memory for each half-open connection.

**ansible.posix.sysctl** — Used instead of `ansible.builtin.command` because it natively manages `/etc/sysctl.conf` entries and applies them with `sysctl -p`, ensuring settings persist across reboots.
