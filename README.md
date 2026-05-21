# ansible-infra-lab

![Ansible](https://img.shields.io/badge/Ansible-2.20+-EE0000?style=flat&logo=ansible&logoColor=white)
![Vagrant](https://img.shields.io/badge/Vagrant-2.4+-1868F2?style=flat&logo=vagrant&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat&logo=ubuntu&logoColor=white)
![QEMU](https://img.shields.io/badge/QEMU-KVM-FF6600?style=flat&logo=qemu&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

> Infrastructure automation lab built with Ansible.  
> Covers web server, database, security hardening, monitoring and Docker deployment.  
> Built on Ubuntu 22.04 LTS VMs managed with Vagrant and QEMU/KVM.

---

## Lab Architecture

```
Host Machine (Ubuntu)
│
├── webserver01 (192.168.56.10)  →  nginx
├── dbserver01  (192.168.56.11)  →  PostgreSQL
└── monitor01   (192.168.56.12)  →  Prometheus + Node Exporter
```

| Host | Role | IP |
|------|------|----|
| webserver01 | nginx web server | 192.168.56.10 |
| dbserver01 | PostgreSQL database | 192.168.56.11 |
| monitor01 | Prometheus monitoring | 192.168.56.12 |

---

## Requirements

- Ansible 2.20+
- Vagrant 2.4+
- QEMU/KVM + libvirt
- vagrant-libvirt plugin
- SSH key pair `~/.ssh/id_ansible`

---

## Setup

**1. Generate the SSH key pair for Ansible:**
```bash
ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/id_ansible
```

**2. Clone the repository:**
```bash
git clone https://github.com/yourusername/ansible-infra-lab.git
cd ansible-infra-lab
```

**3. Start the lab VMs:**
```bash
vagrant up
```
> Vagrant will automatically copy `~/.ssh/id_ansible.pub` into each VM.

**4. Verify connectivity:**
```bash
ansible all -m ping
```

Expected output:
```
webserver01 | SUCCESS => { "ping": "pong" }
dbserver01  | SUCCESS => { "ping": "pong" }
monitor01   | SUCCESS => { "ping": "pong" }
```

---

## Roles

| Role | Description | Status |
|------|-------------|--------|
| common | Base system configuration, updates, timezone | [✅ Done](docs/roles/common.md) |
| webserver | nginx installation and virtual host configuration | [✅ Done](docs/roles/webserver.md) |
| database | PostgreSQL setup, users and databases | [✅ Done](docs/roles/database.md) |
| hardening | SSH hardening, UFW firewall, fail2ban | [✅ Done](docs/roles/hardening.md) |
| monitoring | Prometheus + Node Exporter | 🔜 |
| docker | Docker Engine installation and configuration | 🔜 |

---

## Vagrant Commands

```bash
vagrant up          # Start all VMs
vagrant halt        # Stop all VMs
vagrant destroy     # Delete all VMs
vagrant status      # Check VMs status
vagrant ssh hostname   # SSH into a specific VM
```

---

## Author

**sbobopst** 