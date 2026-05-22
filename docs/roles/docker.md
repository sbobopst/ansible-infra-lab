# Role: docker

Installs **Docker Engine** from the official Docker repository on all hosts.  
Configures the daemon with log rotation and overlay2 storage driver.  
Adds specified users to the `docker` group for non-root access.

---

## What it does

| Task | Description |
|------|-------------|
| Install dependencies | ca-certificates, curl, gnupg, apt-transport-https |
| Add Docker GPG key | Downloads and stores official Docker signing key |
| Add Docker repository | Adds official Docker apt repository |
| Install Docker Engine | docker-ce, docker-ce-cli, containerd.io, buildx, compose |
| Enable Docker | Ensures Docker starts on boot and is running |
| Add users to docker group | Allows non-root users to run Docker commands |
| Deploy daemon.json | Configures log rotation and storage driver |
| Verify installation | Prints installed Docker version |

---

## Variables

Defined in `roles/docker/defaults/main.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `docker_dependencies` | `[ca-certificates, curl, gnupg, apt-transport-https]` | Pre-install dependencies |
| `docker_packages` | `[docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin, docker-compose-plugin]` | Docker packages |
| `docker_users` | `[vagrant]` | Users added to docker group |
| `docker_log_driver` | `json-file` | Container log driver |
| `docker_log_max_size` | `10m` | Max log file size per container |
| `docker_log_max_file` | `3` | Max number of log files per container |

---

## Templates

| Template | Destination | Description |
|----------|------------|-------------|
| `daemon.json.j2` | `/etc/docker/daemon.json` | Docker daemon configuration |

---

## Handlers

| Handler | Triggered by | Action |
|---------|-------------|--------|
| Restart docker | daemon.json changes | `systemctl restart docker` |

---

## Usage

```bash
ansible-playbook playbooks/docker.yml
```

---

## How to verify

### Service status
```bash
vagrant ssh webserver01
sudo systemctl status docker
```

Expected:
```
● docker.service - Docker Application Container Engine
     Active: active (running)
```

### Docker version
```bash
docker --version
```

Expected:
```
Docker version 27.x.x, build xxxxxxx
```

### Docker Compose plugin
```bash
docker compose version
```

Expected:
```
Docker Compose version v2.x.x
```

### Non-root access
```bash
# As vagrant user (no sudo)
docker ps
```

Expected:
```
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```
Empty list — no error. Vagrant user can run Docker without sudo. ✅

### Run a test container
```bash
docker run --rm hello-world
```

Expected:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### Daemon configuration
```bash
cat /etc/docker/daemon.json
```

Expected:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

### Docker info
```bash
docker info | grep -E "Storage Driver|Logging Driver|Docker Root"
```

Expected:
```
Storage Driver: overlay2
Logging Driver: json-file
Docker Root Dir: /var/lib/docker
```

---

## Technical notes

**Official Docker repository** — Docker is installed from `download.docker.com`, not from the Ubuntu default apt repository. The Ubuntu repo ships an older version called `docker.io`. The official repo always has the latest stable version.

**GPG key verification** — The Docker apt repository is signed with a GPG key stored in `/etc/apt/keyrings/docker.asc`. Apt verifies every package against this key before installing — prevents tampered packages.

**docker group** — By default, only root can run Docker commands. Adding `vagrant` to the `docker` group allows running `docker` without `sudo`. Note: membership in the docker group is equivalent to root access — only trusted users should be added.

**overlay2 storage driver** — The recommended storage driver for Ubuntu with ext4/xfs filesystems. Uses Linux kernel's overlay filesystem for efficient copy-on-write layer management.

**Log rotation** — Without configuration, Docker container logs grow indefinitely. `daemon.json` limits each container to 3 log files of 10MB each (30MB max per container). Essential in production to prevent disk exhaustion.

**docker-buildx-plugin** — Enables multi-platform image builds. `docker-compose-plugin` enables `docker compose` (v2) as a Docker CLI plugin, replacing the older standalone `docker-compose` command.

**containerd.io** — The container runtime that Docker uses under the hood. Also used directly by Kubernetes — installing it here means the host is ready for a Kubernetes setup if needed in the future.
