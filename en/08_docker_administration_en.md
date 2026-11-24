# 08 · Docker for System Administrators

[Chapter 07](07_docker.md) showed Docker from the developer's perspective: building an image, launching a stack, working. This chapter takes the other viewpoint — that of the person who must **run** these containers on a server, monitor them, secure them and prevent the disk from filling up on a Sunday evening.

> [!NOTE]
> **Chapter Objectives**
> - Understand the Docker daemon architecture and how to configure it
> - Manage volumes, networks and data persistence
> - Master logs, rotation and resource consumption
> - Harden the configuration: unprivileged user, capabilities, *rootless* mode
> - Backup, restore, supervise and clean a Docker host

## Table of Contents

1. [Daemon Architecture](#daemon-architecture)
2. [Configuring the Daemon](#configuring-the-daemon)
3. [Volumes and Persistence](#volumes-and-persistence)
4. [Networking](#networking)
5. [Logs](#logs)
6. [Limiting Resources](#limiting-resources)
7. [Securing Containers](#securing-containers)
8. [Rootless Mode](#rootless-mode)
9. [Cleanup and Disk Space](#cleanup-and-disk-space)
10. [Backup and Restore](#backup-and-restore)
11. [Docker in Production](#docker-in-production)
12. [Supervision and Diagnosis](#supervision-and-diagnosis)
13. [Summary](#summary)

---

## Daemon Architecture

Docker is not a program, it's a **client/server architecture**:

```text
   docker ps          ┌──────────────────────────────────┐
  (CLI client)  ────► │  dockerd  (daemon, runs as root) │
                      │     └── containerd               │
                      │            └── runc ── container │
                      └──────────────────────────────────┘
                        socket: /var/run/docker.sock
```

| Component | Role |
|-----------|------|
| `docker` | The command-line client — just sends HTTP requests |
| `dockerd` | The daemon: images, networks, volumes, API |
| `containerd` | Container lifecycle manager |
| `runc` | What actually creates the container (namespaces + cgroups) |

The daemon is a standard `systemd` service, controlled like any other service from [chapter 04](04_installation_et_services.md):

```bash
systemctl status docker
systemctl restart docker
systemctl enable --now docker
journalctl -u docker -f
```

> [!CAUTION]
> **`/var/run/docker.sock` is the key to the kingdom.** Anyone who can write to this socket can launch a privileged container mounting `/`, becoming root on the host. This is true for the `docker` group, and it's true for any container the socket is mounted into (`-v /var/run/docker.sock:/var/run/docker.sock`, a classic for monitoring tools). Do this only knowingly.

### Where Docker Stores Its Stuff

```bash
docker info
```

```text
Storage Driver: overlay2
Docker Root Dir: /var/lib/docker
Cgroup Driver: systemd
```

```text
/var/lib/docker/
├── overlay2/     # image and container layers — this is what grows
├── volumes/      # named volumes
├── containers/   # metadata and log files
└── image/        # image index
```

> [!WARNING]
> Never modify `/var/lib/docker` by hand (or `rm -rf` a subfolder). Everything is controlled via `docker` commands. Manual deletion leaves the daemon with inconsistent metadata.

> [!TIP]
> On a server, `/var/lib/docker` deserves its **own partition** or its own LVM volume. An extra-large image must not fill `/` and block the entire system.

---

## Configuring the Daemon

Configuration is done in `/etc/docker/daemon.json` (create it if it doesn't exist):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "data-root": "/srv/docker",
  "default-address-pools": [
    { "base": "172.30.0.0/16", "size": 24 }
  ],
  "live-restore": true,
  "userland-proxy": false
}
```

| Key | Effect |
|-----|--------|
| `log-driver` / `log-opts` | Default log driver and rotation |
| `data-root` | Move `/var/lib/docker` elsewhere (dedicated disk) |
| `default-address-pools` | IP ranges for created networks — useful if conflicts with corporate network |
| `live-restore` | Containers continue running during daemon restart |
| `icc: false` | Forbid container communication on the default network |

```bash
# Verify syntax before restarting
dockerd --validate --config-file /etc/docker/daemon.json

sudo systemctl restart docker
```

> [!IMPORTANT]
> Invalid JSON in `daemon.json` **prevents the daemon from starting** — and all containers from coming up. Always validate before restarting, and keep a copy of the previous file.

To move existing data to another disk:

```bash
sudo systemctl stop docker
sudo rsync -aHAX /var/lib/docker/ /srv/docker/
# then "data-root": "/srv/docker" in daemon.json
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

---

## Volumes and Persistence

Three ways to persist data, not to be confused:

| Type | Syntax | Usage |
|------|--------|-------|
| **Named Volume** | `-v pgdata:/var/lib/postgresql/data` | Application data — **the default choice** |
| **Bind Mount** | `-v /srv/conf:/etc/app:ro` | Configuration files, development code |
| **tmpfs** | `--tmpfs /tmp` | Sensitive or temporary data, RAM only |

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata
docker volume prune            # remove unused volumes
```

A named volume lives in `/var/lib/docker/volumes/<name>/_data`. We can inspect it from the host — preferably read-only.

> [!TIP]
> Prefer **named volume** to bind mount for data: Docker manages permissions, portability and cleanup. Bind mount depends on the host's UID/GID — a container running with a different UID than the directory owner gets immediate `Permission denied` (see [chapter 02](02_file_permissions.md)).

### Mount Read-Only

```bash
docker run -d \
  -v /srv/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v webdata:/usr/share/nginx/html \
  nginx
```

The `:ro` suffix is free and prevents a compromised container from rewriting its own configuration.

---

## Networking

```bash
docker network ls
```

```text
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
b2c3d4e5f6a1   host      host      local
c3d4e5f6a1b2   none      null      local
```

| Driver | Behavior |
|--------|----------|
| `bridge` | Isolated virtual network, NAT outward — the default mode |
| `host` | Container shares the host's network stack, no isolation |
| `none` | No network |
| `overlay` | Multi-host network (Swarm / Kubernetes) |
| `macvlan` | Container gets its own MAC address on the physical network |

### Create an Application Network

```bash
docker network create app-net
docker run -d --name db --network app-net postgres:16-alpine
docker run -d --name api --network app-net -p 3000:3000 my-api
```

On a **user-created network** (unlike the default `bridge`), Docker provides **internal DNS resolution**: `api` reaches the database by simply connecting to `db:5432`. This is exactly what Compose does automatically.

```bash
docker network inspect app-net      # see connected containers and their IPs
docker network connect app-net web  # connect an existing container
docker network disconnect app-net web
```

### Published Ports and Firewall

> [!CAUTION]
> **Docker writes its own rules in `iptables` and bypasses `ufw`.** A container launched with `-p 5432:5432` is reachable from outside **even if `ufw` blocks port 5432** — the Docker rule is evaluated first. Many databases end up exposed to the Internet this way.
>
> The workaround: publish only on the loopback interface.
>
> ```bash
> docker run -d -p 127.0.0.1:5432:5432 postgres:16-alpine
> ```
>
> A service that doesn't need to be reached from the host has **no reason** to be published: on a Docker network, containers already talk to each other without `-p`. See [chapter 05 · Security](05_securite.md) for firewall configuration.

---

## Logs

By default, Docker writes each container's logs to a JSON file:

```text
/var/lib/docker/containers/<id>/<id>-json.log
```

**Without configured rotation, this file grows indefinitely.** It's the first cause of full disk on a Docker host.

### Configure Rotation Globally

In `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

That's 30 MB of logs maximum per container.

> [!IMPORTANT]
> This configuration applies only to containers **created after** daemon restart. Existing containers keep their setting: they must be recreated (`docker compose up -d --force-recreate`).

Per container:

```bash
docker run -d --log-opt max-size=10m --log-opt max-file=3 nginx
```

In Compose:

```yaml
services:
  api:
    image: my-api
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Send Logs to journald

To centralize in the host's `journalctl` from [chapter 04](04_installation_et_services.md#journald):

```json
{ "log-driver": "journald" }
```

```bash
journalctl CONTAINER_NAME=api -f
```

> [!WARNING]
> With `journald` (or `syslog`, or `fluentd`) driver, the `docker logs` command doesn't work the same way: you must query the destination system.

### Find What's Filling the Disk

```bash
sudo du -sh /var/lib/docker/containers/* | sort -rh | head
```

---

## Limiting Resources

By default, **a container can consume all the RAM and CPU of the host**. A leaking process takes the entire server down with it.

```bash
docker run -d \
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.5 \
  --pids-limit 200 \
  --restart unless-stopped \
  my-api
```

| Option | Effect |
|--------|--------|
| `--memory 512m` | Maximum RAM — beyond that, the container is killed (OOM) |
| `--memory-swap` | RAM + swap; equal to `--memory` to forbid swap |
| `--cpus 1.5` | Equivalent of one and a half cores |
| `--cpuset-cpus 0,1` | Pin the container to specific cores |
| `--pids-limit 200` | Max number of processes (fork bomb protection) |

In Compose:

```yaml
services:
  api:
    image: my-api
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
```

### Restart Policies

| Policy | Behavior |
|--------|----------|
| `no` | Default — no restart |
| `on-failure[:5]` | Restart if exit code ≠ 0, at most 5 times |
| `always` | Always restart, even after host reboot |
| `unless-stopped` | Like `always`, but respects manual stop — **the right choice in production** |

### Health Check

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
```

```bash
docker ps --format '{{.Names}}\t{{.Status}}'
```

```text
api     Up 4 minutes (healthy)
db      Up 4 minutes (healthy)
```

> [!TIP]
> A health check is only valuable if something looks at it: your supervision, a reverse proxy, or `depends_on: condition: service_healthy`. On its own, it just displays a status.

---

## Securing Containers

### Don't Run as Root

By default, a container's process runs as **root** — and UID 0 in the container is UID 0 on the host. In case of escape, the damage is maximal.

In the `Dockerfile`:

```dockerfile
RUN addgroup -S app && adduser -S -G app app
USER app
```

Or at runtime:

```bash
docker run -d --user 1000:1000 my-api
```

### Standard Hardening

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  my-api
```

| Option | Effect |
|--------|--------|
| `--read-only` | Root filesystem read-only |
| `--tmpfs /tmp` | Writable space in RAM, since everything else is read-only |
| `--cap-drop ALL` | Remove **all** Linux capabilities |
| `--cap-add NET_BIND_SERVICE` | …then grant only what's needed (listen below port 1024) |
| `--security-opt no-new-privileges` | Forbid escalation via SUID binaries |

The same in Compose:

```yaml
services:
  api:
    image: my-api
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    user: "1000:1000"
```

> [!CAUTION]
> **Never use `--privileged`.** This option removes almost all isolation: the container gets all capabilities and access to host devices. It's the equivalent of `chmod 777` on machine security. If a tool "needs" it, it actually needs one or two specific capabilities — grant those.

### Other Reflexes

- **Pin image versions**, and prefer `-alpine` or `-slim` variants (less code, fewer CVEs).
- **Scan images** before deployment:

  ```bash
  docker scout cves my-api:1.0
  # or
  trivy image my-api:1.0
  ```

- **Rebuild regularly**: a frozen image accumulates vulnerabilities in its system dependencies.
- **Pass secrets at runtime**, never in the image (see chapter 07). In Compose:

  ```yaml
  services:
    api:
      secrets: [db_password]
  secrets:
    db_password:
      file: /srv/secrets/db_password.txt
  ```

- **AppArmor / SELinux** (chapter 05) also apply to containers: Docker loads a `docker-default` profile by default.

---

## Rootless Mode

The Docker daemon runs as root. **Rootless** mode runs it under an unprivileged user, using *user namespaces*: root in the container maps to an unprivileged UID on the host.

```bash
sudo apt install -y uidmap docker-ce-rootless-extras
dockerd-rootless-setuptool.sh install
```

```bash
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
systemctl --user enable --now docker
sudo loginctl enable-linger $USER   # daemon survives logout
```

| Advantage | Limitation |
|-----------|-----------|
| Container escape doesn't give root on the host | Can't listen below port 1024 without configuration |
| Each user has their own isolated daemon | Some network and storage drivers unavailable |
| No need for `docker` group | Slightly lower network performance |

> [!TIP]
> This is the default mode to prefer on a shared server or CI machine, where multiple users launch containers. For a dedicated application server, the standard properly hardened daemon remains common.

---

## Cleanup and Disk Space

```bash
docker system df
```

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          24        4         8.2GB     6.9GB (84%)
Containers      6         4         120MB     40MB (33%)
Local Volumes   11        3         2.1GB     1.8GB (85%)
Build Cache     142       0         3.4GB     3.4GB
```

```bash
docker container prune          # stopped containers
docker image prune              # dangling images (no tag)
docker image prune -a           # all unused images
docker builder prune            # build cache
docker volume prune             # unused volumes
docker system prune -a          # all of the above, except volumes
docker system prune -a --volumes # all, volumes included
```

> [!CAUTION]
> `docker system prune -a --volumes` removes **volumes not attached to an existing container**. A database whose container was deleted but whose volume should be preserved disappears permanently. On a server, prune images and build cache; handle volumes manually, knowingly.

Automatic cleanup without volumes, via `cron` (chapter 04):

```bash
# /etc/cron.d/docker-prune
0 4 * * 0 root /usr/bin/docker system prune -af --filter "until=168h" >> /var/log/docker-prune.log 2>&1
```

The `until=168h` filter spares anything less than 7 days old.

---

## Backup and Restore

**What must be backed up are volumes and configuration files.** Images are rebuilt, containers are recreated; data is not.

### Backup a Volume

```bash
docker run --rm \
  -v pgdata:/data:ro \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .
```

Launch a disposable container mounting the volume read-only and the current folder in write mode, then archive (chapter 03).

### Restore

```bash
docker run --rm \
  -v pgdata:/data \
  -v "$(pwd)":/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/pgdata-2026-08-02.tar.gz -C /data"
```

### Application Dump

For a database, a logical dump is often preferable to file copying: it's consistent, portable between versions and partially restorable.

```bash
docker compose exec -T db pg_dump -U postgres app | gzip > app-$(date +%F).sql.gz
```

### Export an Image or Container

```bash
docker save my-api:1.0 | gzip > my-api-1.0.tar.gz   # an image, with its layers
docker load < my-api-1.0.tar.gz                      # re-import it

docker export web > web-fs.tar    # a container's filesystem, flattened
```

> [!IMPORTANT]
> `save` / `load` work on **images** (with their history). `export` / `import` flatten a **container** into a filesystem without history or metadata. To transfer an application, it's `save` you need.

---

## Docker in Production

### Image Registry

```bash
docker login registry.example.be
docker tag my-api:1.0 registry.example.be/team/my-api:1.0
docker push registry.example.be/team/my-api:1.0
docker pull registry.example.be/team/my-api:1.0
```

> [!TIP]
> On a server, don't build images: your CI builds them, scans them, pushes them to the registry; the server just `pull` an immutable tag. Build needs tools, sources and secrets — things that have no place in production.

### Launch a Compose Stack at Boot

Compose doesn't manage startup at boot. Two approaches:

**1. `restart: unless-stopped`** on each service — the Docker daemon (enabled via `systemctl enable docker`) relaunches containers at startup. Simple and sufficient in most cases.

**2. A dedicated systemd unit**, when you want to control order and dependencies (chapter 04):

```ini
# /etc/systemd/system/my-app.service
[Unit]
Description=Application stack my-app
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/my-app
ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose down
ExecReload=/usr/bin/docker compose up -d --remove-orphans
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-app
```

### Deploy a New Version

```bash
cd /srv/my-app
docker compose pull            # fetch new images
docker compose up -d            # recreate only modified services
docker image prune -f           # clean old images
```

> [!WARNING]
> `docker compose up -d` recreates containers whose configuration or image changed: there's a **service outage** of a few seconds. For zero-downtime, you need a reverse proxy and blue/green deployment — or an orchestrator (Swarm, Nomad, Kubernetes).

### Layered Compose Files

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

The second file **overrides** the first: keep a common base and express only production differences (no code bind mount, resource limits, automatic restart, ports on `127.0.0.1`).

A complete example of both files is in [`07_docker/05_stack_compose/`](07_docker/05_stack_compose/) — compare `docker-compose.yml` and [`docker-compose.prod.yml`](07_docker/05_stack_compose/docker-compose.prod.yml).

> [!CAUTION]
> **Lists are concatenated, not replaced.** Redeclaring `ports:` in the override file **adds** to the base, and writing `ports: []` removes nothing at all. The trap is serious: you think you've restricted the base to `127.0.0.1`, but it stays published on `0.0.0.0` — or the container refuses to start for port conflict.
>
> Since Compose 2.24, two markers solve the problem:
>
> ```yaml
> services:
>   api:
>     ports: !override        # replace inherited list
>       - "127.0.0.1:3000:3000"
>     volumes: !reset []      # clear inherited list
> ```

**Always check the merge result before deploying:**

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml config
```

```bash
# The question that matters: what's actually published?
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps --format '{{.Service}} {{.Ports}}'
```

```text
api    127.0.0.1:3000->3000/tcp
cache  6379/tcp
db     5432/tcp
```

Here only the API is reachable from the host, and only locally: `cache` and `db` exist only on the Docker network.

---

## Supervision and Diagnosis

```bash
docker stats                          # CPU / RAM / network / I/O live
docker stats --no-stream              # one snapshot, exploitable in script
docker events                         # the daemon's event stream
docker ps --filter "health=unhealthy" # unhealthy containers
docker inspect --format '{{.State.ExitCode}}' api
```

A small check script, in the spirit of [chapter 06](06_shellscript/01_Base/base.md):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Alert on any stopped or unhealthy container
mapfile -t down < <(docker ps -a --filter "status=exited" --format '{{.Names}}')
mapfile -t sick < <(docker ps --filter "health=unhealthy" --format '{{.Names}}')

if (( ${#down[@]} + ${#sick[@]} > 0 )); then
    printf 'Stopped containers: %s\n' "${down[*]:-none}"
    printf 'Sick containers: %s\n' "${sick[*]:-none}"
    exit 1
fi

echo "All containers are operational."
```

### Common Failures

| Symptom | Lead |
|---------|------|
| Container restarts in a loop | `docker logs --tail 100 name` then `docker inspect --format '{{.State.ExitCode}}' name` |
| Exit code **137** | Killed by SIGKILL — almost always OOM killer: `--memory` too low or memory leak |
| Exit code **139** | Segfault in the application |
| `no space left on device` | `docker system df`, then prune images and build cache |
| `port is already allocated` | `ss -tulpn \| grep :8080` — another service uses the port |
| Container resolves no names | Daemon DNS: check `dns` in `daemon.json` |
| General host slowness | `docker stats` — a container without resource limits |

> [!TIP]
> To debug a container that stops immediately, bypass its startup command:
>
> ```bash
> docker run -it --rm --entrypoint sh my-api:1.0
> ```
>
> You get a shell inside the image, with its exact contents, and can run the command manually to see the error.

---

## Summary

| Command | Role |
|----------|------|
| `systemctl status docker` / `journalctl -u docker` | Daemon status and logs |
| `docker info` / `docker system df` | Configuration / disk usage |
| `docker volume ls` / `inspect` / `prune` | Manage volumes |
| `docker network create` / `inspect` | Manage networks |
| `docker run --memory --cpus --pids-limit` | Limit resources |
| `docker run --cap-drop ALL --read-only --user` | Harden a container |
| `docker system prune -af --filter "until=168h"` | Periodic cleanup |
| `docker save` / `load` | Export / import an image |
| `docker stats` / `docker events` | Real-time supervision |
| `docker compose pull && docker compose up -d` | Deploy a new version |

### Files to Know

| Path | Contents |
|------|----------|
| `/etc/docker/daemon.json` | Daemon configuration |
| `/var/lib/docker/` | Images, containers, volumes |
| `/var/lib/docker/volumes/<name>/_data` | Data of a named volume |
| `/var/run/docker.sock` | API socket — to be protected |

### The Seven Rules to Remember

> [!TIP]
> 1. **Configure log rotation** at installation — else the disk fills.
> 2. **Limit memory, CPU and PID** of each container.
> 3. **Publish to `127.0.0.1`** everything that doesn't need exposure — Docker bypasses `ufw`.
> 4. **Never `--privileged`**, never mount the Docker socket without reason.
> 5. **Don't run as root** in the container.
> 6. **Backup volumes**, not containers.
> 7. **Regular prune of images and cache**, never volumes automatically.

---

⬅️ [Previous: 07 · Docker for Developers](07_docker.md) · 🏠 [Table of Contents](README.md) · [Next: 09 · tmux ➡️](09_tmux.md)
