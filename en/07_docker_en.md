# 07 · Docker for Developers

Docker allows you to package an application **with everything it needs to run** (runtime, libraries, configuration files) in an isolated and reproducible unit: the **container**.

The promise: the famous *"it works on my machine"* becomes *"it works everywhere, in the same way"*.

> [!NOTE]
> **Chapter Objectives**
> - Understand the difference between an image, a container, and a volume
> - Launch, inspect, stop and delete containers
> - Write a `Dockerfile` and build your own image
> - Orchestrate multiple services with `docker compose`
> - Set up a complete, disposable development environment

## Table of Contents

1. [Container or Virtual Machine?](#container-or-virtual-machine)
2. [Installation](#installation)
3. [The Three Basic Concepts](#the-three-basic-concepts)
4. [Manipulating Containers](#manipulating-containers)
5. [Manipulating Images](#manipulating-images)
6. [Writing a Dockerfile](#writing-a-dockerfile)
7. [Multi-Stage Build](#multi-stage-build)
8. [Building Your Image](#building-your-image)
9. [Docker Compose](#docker-compose)
10. [A Complete Development Environment](#a-complete-development-environment)
11. [Repository Examples](#repository-examples)
12. [Summary](#summary)

---

## Container or Virtual Machine?

A **virtual machine** emulates a complete computer: it includes its own kernel and operating system. A **container** shares the host machine's kernel and isolates only what is necessary (processes, networking, filesystem).

```text
   Virtual Machines                     Containers

  ┌──────┐ ┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐ ┌──────┐
  │ App  │ │ App  │ │ App  │          │ App  │ │ App  │ │ App  │
  ├──────┤ ├──────┤ ├──────┤          ├──────┴─┴──────┴─┴──────┤
  │ OS   │ │ OS   │ │ OS   │          │    Docker Engine       │
  ├──────┴─┴──────┴─┴──────┤          ├────────────────────────┤
  │      Hypervisor        │          │   Linux Kernel (host)  │
  ├────────────────────────┤          ├────────────────────────┤
  │   Linux Kernel (host)  │          │       Hardware         │
  └────────────────────────┘          └────────────────────────┘
```

| | Virtual Machine | Container |
|---|---|---|
| Startup | Tens of seconds | Dozens of ms |
| Size | Several GB | A few MB to hundreds of MB |
| Isolation | Strong (separate kernel) | Good, but kernel **shared** |
| Typical Use | Host different OSs | Package an application |

> [!IMPORTANT]
> A container **is not a small VM**. It is one or more processes from the host, isolated by two Linux kernel mechanisms seen indirectly in [chapter 05](05_securite.md): **namespaces** (what the process sees) and **cgroups** (what it has the right to consume).
>
> Direct consequence: on macOS and Windows, Docker runs a **Linux VM** in the background, because a Linux kernel has to live somewhere.

---

## Installation

On Debian / Ubuntu, via the official repository:

```bash
# Prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Docker repository GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installation
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verification:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

### Use Docker Without `sudo`

```bash
sudo usermod -aG docker $USER
```

You must then **log out and log back in** for the new group to take effect (see [chapter 02.1](02.1_user_management.md)).

> [!CAUTION]
> The `docker` group is equivalent to **root** access on the machine: any member can mount `/` in a container and modify the entire system. Only grant it to users you would already give `sudo` without a password. [Chapter 08](08_docker_administration.md#rootless-mode) presents *rootless* mode, which solves this problem.

> [!NOTE]
> `docker compose` (with a space) is the current plugin. `docker-compose` (with a hyphen) is the old Python script, still present on many machines but deprecated. The `docker-compose.yml` files themselves remain identical.

---

## The Three Basic Concepts

| Concept | Analogy | Description |
|---------|---------|-------------|
| **Image** | A class | A read-only model: a filesystem + a startup command |
| **Container** | An instance | An execution of an image, with a write layer on top |
| **Volume** | An external drive | A storage space that **survives** container deletion |

```text
  Image  ──── docker run ───►  Container  ──── docker commit / build ───►  Image
 (model)                      (instance)
```

> [!IMPORTANT]
> **A container is disposable.** Everything written to its filesystem disappears with it. Any data to preserve (database, uploaded files, logs) must live in a **volume**.

An image is identified by a **name** and a **tag**:

```text
postgres:16-alpine
│        │
│        └── tag: the version (default `latest`)
└── repository name (here on Docker Hub)
```

> [!WARNING]
> The `latest` tag has nothing magical: it's just the default tag, and it **changes** over time. A `node:latest` image today and in six months are not the same thing. In projects, always pin a version: `node:22-alpine`.

---

## Manipulating Containers

### Launch a Container

```bash
docker run nginx
```

The terminal stays blocked: the container runs in the foreground.

The options that come up all the time:

| Option | Effect |
|--------|--------|
| `-d` | *detached* — runs in the background |
| `--name my-nginx` | Gives the container a name (otherwise Docker generates a random one) |
| `-p 8080:80` | Publishes the container's port **80** on the host's port **8080** |
| `-v $(pwd):/app` | Mounts a folder from the host in the container |
| `-e VAR=value` | Sets an environment variable |
| `--rm` | Automatically removes the container on stop |
| `-it` | Interactive mode with a terminal (for a shell) |

In practice:

```bash
docker run -d --name web -p 8080:80 nginx
```

The application is available at <http://localhost:8080>.

> [!TIP]
> **The sense of `-p`**: `-p HOST:CONTAINER`. Always read left to right, outside to inside. Same logic for `-v HOST_SOURCE:CONTAINER_TARGET`.

### A Disposable Shell

```bash
docker run -it --rm debian:12 bash
```

This is exactly the sandbox environment offered in [README](README.md#-mise-en-place-de-lenvironnement) for training manipulations.

### List

```bash
# Running containers
docker ps

# All, including stopped ones
docker ps -a
```

```text
CONTAINER ID   IMAGE     COMMAND                  STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx     "/docker-entrypoint.…"   Up 2 minutes   0.0.0.0:8080->80/tcp   web
```

### Control

```bash
docker stop web        # clean shutdown (SIGTERM, then SIGKILL after 10s)
docker start web       # restart
docker restart web
docker rm web          # deletion (container must be stopped)
docker rm -f web       # stop + deletion
```

### Enter a Running Container

```bash
docker exec -it web bash
```

If the image is based on Alpine, `bash` doesn't exist:

```bash
docker exec -it web sh
```

> [!TIP]
> `docker exec` launches a **new** process in an existing container. `docker run` creates a **new container**. This is confusion #1 for beginners.

### See What's Happening

```bash
docker logs web            # the logs (stdout / stderr of the main process)
docker logs -f web         # continuously, like a tail -f
docker logs --tail 50 web  # the last 50 lines
docker stats               # CPU / RAM consumption in real-time
docker inspect web         # all configuration, in JSON
docker top web             # the container's processes
```

> [!IMPORTANT]
> In a container, an application should **not** write its logs to a file: it writes to **standard output**, and Docker handles the rest. This is what makes `docker logs` (and all centralization tools) possible.

### Copy Files

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./nginx.conf web:/etc/nginx/nginx.conf
```

---

## Manipulating Images

```bash
docker pull node:22-alpine   # download an image
docker images                # list local images
docker rmi node:22-alpine    # delete an image
docker history nginx         # see the layers of an image
```

### Layers

An image is a **stack of read-only layers**. Each instruction in a `Dockerfile` creates a layer, and layers are **shared** between images and cached.

```text
┌──────────────────────────┐  ← write layer (the container, ephemeral)
├──────────────────────────┤
│ CMD ["node", "app.js"]   │  ┐
│ COPY . .                 │  │
│ RUN npm ci               │  ├── the image (read-only)
│ COPY package*.json ./    │  │
│ FROM node:22-alpine      │  ┘
└──────────────────────────┘
```

This is why the second `docker build` is nearly instant: Docker reuses unchanged layers. It's also why the **order of instructions matters** — more on that below.

---

## Writing a Dockerfile

The `Dockerfile` is the recipe for building the image: a sequence of **steps**, executed top to bottom, each producing a layer.

> [!TIP]
> All the `Dockerfile` examples in this chapter exist in full, commented line by line and **actually buildable** in the [`07_docker/`](07_docker/) folder of the repository. The first, [`01_simple/Dockerfile`](07_docker/01_simple/Dockerfile), reproduces exactly the steps below.

An example for a small Node application:

```dockerfile
# --- STEP 1 · the base image ---
FROM node:22-alpine

# --- STEP 2 · the working directory (created if it doesn't exist) ---
WORKDIR /app

# --- STEP 3 · the dependency manifests, alone ---
COPY package.json package-lock.json ./

# --- STEP 4 · installation: this layer is rebuilt
#     only if package.json or package-lock.json changes ---
RUN npm ci --omit=dev

# --- STEP 5 · the source code, which changes at every commit ---
COPY src/ ./src/

# --- STEP 6 · execution configuration ---
ENV NODE_ENV=production
EXPOSE 3000

# --- STEP 7 · the unprivileged user ---
USER node

# --- STEP 8 · the health check ---
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -qO- http://127.0.0.1:3000/health || exit 1

# --- STEP 9 · the startup command ---
CMD ["node", "src/server.js"]
```

Each step produces a layer, visible after the build:

```bash
docker history my-app:1.0
```

```text
IMAGE          CREATED BY                                      SIZE
a1b2c3d4e5f6   CMD ["node" "src/server.js"]                    0B
<missing>      HEALTHCHECK &{["CMD-SHELL" "wget -qO- http:…     0B
<missing>      USER node                                       0B
<missing>      EXPOSE map[3000/tcp:{}]                         0B
<missing>      ENV NODE_ENV=production                         0B
<missing>      COPY src/ ./src/                                12.4kB
<missing>      RUN npm ci --omit=dev                           4.21MB
<missing>      COPY package.json package-lock.json ./          1.02kB
<missing>      WORKDIR /app                                    0B
<missing>      /bin/sh -c #(nop) ADD file:… in /               142MB
```

**Metadata** instructions (`ENV`, `EXPOSE`, `USER`, `CMD`, `HEALTHCHECK`) weigh 0 bytes: they only annotate the image. Only those that **write files** (`COPY`, `RUN`, `ADD`) take up space.

### Instructions to Know

| Instruction | Role |
|-------------|------|
| `FROM` | The base image — always the first instruction |
| `WORKDIR` | The working directory for subsequent instructions |
| `COPY src dst` | Copy from build context to image |
| `ADD` | Like `COPY`, but handles URLs and extracts archives — **prefer `COPY`** |
| `RUN` | Execute a command **during build** |
| `ENV VAR=value` | Environment variable present in the image |
| `ARG VAR` | Variable available **only during build** |
| `EXPOSE` | Documents a port (doesn't publish it — `-p` does) |
| `USER` | The user executing what follows |
| `VOLUME` | Declares a persistent mount point |
| `CMD` | The default command, **replaceable** on command line |
| `ENTRYPOINT` | The container's executable, **non-replaceable** by default |
| `HEALTHCHECK` | The command that says if the container is healthy |

> [!NOTE]
> **`CMD` or `ENTRYPOINT`?** With `ENTRYPOINT ["ping"]` and `CMD ["localhost"]`, `docker run my-image google.com` runs `ping google.com`. `ENTRYPOINT` fixes the program, `CMD` provides default arguments. For a typical application, `CMD` alone is enough.

### `.dockerignore`

Like `.gitignore`, but for the build context. Without it, the entire folder is sent to the Docker daemon — including `node_modules` and `.git`.

```text
.git
node_modules
dist
*.log
.env
```

> [!WARNING]
> Never copy secrets (`.env`, private key, token) into an image. Even if deleted by a later instruction, it remains readable in the layer where it was added. Secrets are passed at runtime via environment variables or a mounted file.

### Order of Instructions

This is the main source of slow builds:

```dockerfile
# ❌ Bad: any code change rebuilds all dependencies
COPY . .
RUN npm ci

# ✅ Good: dependencies are rebuilt only if package.json changes
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

**Rule**: from most stable to most volatile.

---

## Multi-Stage Build

**Building** an application and **running** it don't require the same tools. Compilation requires an SDK, a compiler, development dependencies, sources; running often requires just a binary or a few static files.

A `Dockerfile` can contain **multiple `FROM`** statements: each opens a new stage, starting fresh. We work in the early ones and keep only the last — explicitly copying from it what we need.

```text
  STAGE 1 "builder"              STAGE 2 "runtime"
 ┌───────────────────────┐        ┌───────────────────────┐
 │ node:22-alpine        │        │ nginx:alpine          │
 │ + npm, node_modules   │        │                       │
 │ + source code         │        │                       │
 │ + /app/dist ──────────┼───────►│ /usr/share/nginx/html │
 └───────────────────────┘ COPY   └───────────────────────┘
   discarded at the end      --from=builder    the delivered image
        235 MB                                   93 MB
```

> [!IMPORTANT]
> What is not explicitly copied with `COPY --from=` **doesn't exist** in the final image. The compiler, sources, private repository credentials used during build: all this remains in the intermediate stage, which is never published.

### Step by Step

The complete example is in [`07_docker/02_multistage_node/`](07_docker/02_multistage_node/):

```dockerfile
# ============ STAGE 1/2 · builder ============
# `AS builder` names the stage so we can copy from it later.
FROM node:22-alpine AS builder

WORKDIR /app

# 1.a · dependencies — here we install EVERYTHING,
#       devDependencies too: we need them to compile
COPY package*.json ./
RUN npm ci

# 1.b · sources
COPY build.js ./
COPY src/ ./src/

# 1.c · build: produces /app/dist
RUN npm run build

# ============ STAGE 2/2 · runtime ============
# New FROM = fresh image. Nothing from stage 1 follows,
# except what we explicitly copy.
FROM nginx:alpine

# 2.a · only the build result
COPY --from=builder /app/dist /usr/share/nginx/html

# 2.b · our configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

```bash
docker build -t demo:02 .

# We can also stop at a stage, to inspect or debug it
docker build --target builder -t demo:02-builder .
docker images | grep demo
```

If you have multiple dockerfiles:
```shell
docker build -f ./Path/To/Target/Dockerfile -t demo:02 .
```

```text
demo   02-builder   235MB
demo   02          92.7MB
```

### Three Concrete Cases

| Language | Build Stage | Final Stage | What We Leave Behind |
|----------|-------------|-------------|----------------------|
| **Node → Static** ([example 02](07_docker/02_multistage_node/Dockerfile)) | `node:22-alpine` | `nginx:alpine` | Node, npm, `node_modules`, sources — 235 MB → 93 MB |
| **Go** ([example 03](07_docker/03_multistage_go/Dockerfile)) | `golang:1.23-alpine` | `scratch` | The entire compilation chain — just an 8 MB binary remains |
| **Python** ([example 04](07_docker/04_python/Dockerfile)) | `python:3.12-slim` + `build-essential` | `python:3.12-slim` | `gcc`, development headers, `pip` cache — 204 MB total |

The Go case is the most dramatic, because a static binary needs **nothing** to run:

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
# CGO_ENABLED=0    → static binary, no libc dependency
# -ldflags "-s -w" → no symbol table: smaller binary
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./main.go

FROM scratch
COPY --from=builder /out/server /server
USER 65534:65534
ENTRYPOINT ["/server"]
```

About **8 MB** total, versus over 300 MB if we delivered the complete `golang` image.

> [!WARNING]
> `scratch` contains **no** files: no shell, no `/etc/passwd`, no root certificates. Practical consequences:
> - `docker exec -it … sh` is impossible — debugging is logs-only;
> - `USER app` fails (no username exists), you must use a **numeric UID**;
> - any outgoing HTTPS call fails unless you copy `ca-certificates.crt` from an intermediate stage.
>
> The reasonable alternative: `gcr.io/distroless/static` or `alpine`, which keep the strict minimum.

### What Multi-Stage Really Brings

| Benefit | Why |
|---------|-----|
| **A smaller image** | Less to transfer at each deployment, faster startup |
| **Fewer vulnerabilities** | A missing compiler is a compiler with no CVE |
| **No secret leaks** | A private repository token used at build doesn't leave the stage where it was used |
| **Parallelizable stages** | BuildKit builds independent stages simultaneously |
| **One file, multiple targets** | `--target dev` or `--target runtime` — see below |

### A Dev Image and a Prod Image, in the Same File

This is the pattern used by the example stack ([`07_docker/05_stack_compose/api/Dockerfile`](07_docker/05_stack_compose/api/Dockerfile)): a common `base` stage, then two targets.

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app
COPY package.json package-lock.json ./

# --- dev target: full tooling, hot reload ---
FROM base AS dev
RUN npm install
COPY src/ ./src/
CMD ["node", "--watch", "src/server.js"]

# --- prod target: minimal dependencies, non-root user ---
FROM base AS runtime
RUN npm ci --omit=dev && npm cache clean --force
COPY --chown=node:node src/ ./src/
USER node
CMD ["node", "src/server.js"]
```

```bash
docker build --target dev     -t stack-api:dev .
docker build --target runtime -t stack-api:1.0 .
```

In Compose, this is chosen per service:

```yaml
services:
  api:
    build:
      context: ./api
      target: dev        # `runtime` in the production override file
```

---

## Building Your Image

```bash
# -t: the image name (tag); the final "." is the build context
docker build -t my-app:1.0 .
```

```bash
# Launch the built image
docker run -d -p 3000:3000 --name app my-app:1.0
```

Some useful options:

```bash
docker build -t my-app:1.0 --no-cache .            # ignore cache
docker build -t my-app:1.0 --target builder .      # stop at a stage
docker build -t my-app:1.0 --build-arg VERSION=2 . # pass an ARG
```

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
# CGO_ENABLED=0    → static binary, no libc dependency
# -ldflags "-s -w" → no symbol table: smaller binary
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./main.go
RUN echo $EXTERNAL_ARG

FROM scratch
COPY --from=builder /out/server /server
USER 65534:65534
ENTRYPOINT ["/server"]
```

> [!TIP]
> Always tag twice: an exact version **and** a moving alias.
>
> ```bash
> docker build -t my-app:1.4.2 -t my-app:latest .
> ```

---

## Docker Compose

A real application is never a single container: an API, a database, a cache, a reverse proxy… Chaining five `docker run` commands with their options is not maintainable. **Compose** describes the entire stack in a `docker-compose.yml` file.

The file at the root of this repository, for example:

```yaml
services:
  db:
    image: postgres:12-alpine
    environment:
      POSTGRES_PASSWORD: 1234
      POSTGRES_USER: postgres
      POSTGRES_DB: app
    ports:
      - '5435:5432'
    volumes:
      - ./:/init_db
```

```bash
docker compose up -d
```

One command, and the database is running.

### A More Complete File

```yaml
services:
  api:
    build: .                    # build from local Dockerfile
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://postgres:1234@db:5432/app
      NODE_ENV: development
    volumes:
      - .:/app                  # host code, mounted directly
      - /app/node_modules       # …except node_modules, which stays in the image
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: 1234
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7-alpine

volumes:
  pgdata:
```

Key points about this file:

- **`build: .`** builds the image from the local `Dockerfile`; **`image:`** downloads a pre-made one.
- **The network is automatic**: services reach each other by **name** (`db`, `cache`). The API connects to `db:5432`, not `localhost`.
- **`depends_on` with `condition`** waits for the database to be truly *ready*, not just started.
- **`pgdata`** is a named volume: database data survives `docker compose down`.
- **`- .:/app`** mounts source code directly: modify a file on the host, the app sees it immediately.

> [!IMPORTANT]
> Since Compose v2, the `version:` key at the top of the file is **obsolete** and ignored (it even generates a warning). It's still present in many files, including this one.

### Compose Commands

| Command | Effect |
|----------|--------|
| `docker compose up -d` | Start the entire stack in the background |
| `docker compose up -d --build` | …rebuilding images |
| `docker compose down` | Stop and remove containers and network |
| `docker compose down -v` | …**and volumes** (so data) |
| `docker compose ps` | Status of services |
| `docker compose logs -f api` | Follow a service's logs |
| `docker compose exec api sh` | Open a shell in a service |
| `docker compose restart api` | Restart a service |
| `docker compose config` | Display the final resolved configuration |

> [!CAUTION]
> `docker compose down -v` **removes volumes**, so your database data. It's very handy for starting fresh in development, and catastrophic elsewhere.

### Environment Variables

Compose automatically reads a `.env` file next to `docker-compose.yml`:

```text
POSTGRES_PASSWORD=1234
API_PORT=3000
```

```yaml
services:
  api:
    ports:
      - "${API_PORT}:3000"
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

> [!TIP]
> Version a `.env.example` documenting the expected variables, and add `.env` to `.gitignore`.

---

## A Complete Development Environment

The typical scenario: a new developer joins the project and must be operational in one command.

```text
my-project/
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
├── .env.example
└── src/
```

```bash
git clone git@github.com:org/my-project.git
cd my-project
cp .env.example .env
docker compose up -d
```

No Node, PostgreSQL, or Redis to install on the machine — and the same version for the entire team.

### Useful Reflexes for Daily Work

```bash
# Run a one-off command in a service
docker compose exec api npm run migrate

# Run a command in a disposable container
docker compose run --rm api npm test

# Open a psql client in the database
docker compose exec db psql -U postgres -d app

# Start completely from scratch
docker compose down -v && docker compose up -d --build
```

> [!TIP]
> These commands are long: they're perfect candidates for aliases or shell functions, as seen in [chapter 10](10_shell_config.md).
>
> ```bash
> alias dcu='docker compose up -d'
> alias dcd='docker compose down'
> alias dcl='docker compose logs -f'
> dcx() { docker compose exec "$1" "${@:2}"; }
> ```

### Debugging: The Three Reflexes

| Symptom | Command |
|---------|---------|
| Container stops on its own | `docker compose logs api` |
| "Connection refused" to the database | Check the **service name** (`db`, not `localhost`) |
| A code change doesn't appear | Check the mount `-v` / rebuild the image |

> [!WARNING]
> `localhost` **inside** a container refers to the container itself, not your machine. To reach another service, use its Compose name; to reach the host, use `host.docker.internal` (or the gateway IP on Linux).

---

## Repository Examples

The [`07_docker/`](07_docker/) folder contains five standalone projects, to build and run. Each `Dockerfile` is commented **step by step**: this is the extended version of this chapter.

| Folder | What It Shows | To Go See |
|--------|---------------|-----------|
| [`01_simple/`](07_docker/01_simple/) | The 9 steps of a `Dockerfile`, `.dockerignore`, layer caching | `docker history` after a code change |
| [`02_multistage_node/`](07_docker/02_multistage_node/) | Multi-stage: Node compiles, `nginx` serves — 235 MB → 93 MB | `docker build --target builder` to compare |
| [`03_multistage_go/`](07_docker/03_multistage_go/) | A `scratch` image of 8 MB | Try `docker exec … sh`: there's no shell |
| [`04_python/`](07_docker/04_python/) | The `venv` built in one stage, copied to the other | The role of `PYTHONUNBUFFERED=1` |
| [`05_stack_compose/`](07_docker/05_stack_compose/) | API + PostgreSQL + Redis, `healthcheck`, `.env`, production overrides | The API reaches `db` **by its name** |

```bash
cd 07_docker/05_stack_compose
cp .env.example .env
docker compose up -d --build
curl http://localhost:3000
```

```json
{
  "message": "Demo stack API",
  "services": {
    "db":    { "host": "db",    "ip": "172.22.0.2", "reachable": true },
    "cache": { "host": "cache", "reachable": true }
  }
}
```

> [!TIP]
> The detailed instructions for each example, with manipulations to really see what's happening, are in [`07_docker/README.md`](07_docker/README.md).

---

## Summary

| Command | Role |
|----------|------|
| `docker run -d -p 8080:80 --name web nginx` | Launch a container in the background |
| `docker ps` / `docker ps -a` | List active / all containers |
| `docker logs -f name` | Follow a container's logs |
| `docker exec -it name bash` | Open a shell in a running container |
| `docker stop` / `start` / `rm -f` | Control and delete a container |
| `docker images` / `docker rmi` | List / delete images |
| `docker build -t name:tag .` | Build an image from a `Dockerfile` |
| `docker compose up -d --build` | Start the entire stack |
| `docker compose down -v` | Stop everything, **volumes included** |
| `docker compose exec svc cmd` | Run a command in a service |

### The Five Rules to Remember

> [!TIP]
> 1. **A container is disposable** — data goes in a volume.
> 2. **An image pins its version** — never `latest` in a project.
> 3. **Most stable to most volatile** in the `Dockerfile` — for caching.
> 4. **Logs go to stdout**, never to a file.
> 5. **No secrets in an image** — they're passed at runtime.

---

⬅️ [Previous: 06.6 · Functions](06_shellscript/06_fonctions/fonctions.md) · 🏠 [Table of Contents](README.md) · [Next: 08 · Docker for the Administrator ➡️](08_docker_administration.md)
