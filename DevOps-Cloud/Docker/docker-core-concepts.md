# Docker Core Concepts

Docker is a platform for building, shipping, and running applications inside lightweight, isolated **containers**.
Interview questions focus on the difference between containers and VMs, image layering, Dockerfile best practices,
networking, volumes, and multi-stage builds.

---

## Containers vs Virtual Machines

| Aspect           | Container                              | Virtual Machine                       |
|------------------|----------------------------------------|---------------------------------------|
| Isolation        | Process-level (shared host kernel)     | Hardware-level (own kernel)           |
| Startup          | Seconds                               | Minutes                              |
| Size             | MBs (app + dependencies only)         | GBs (full OS)                        |
| Performance      | Near-native                           | Hypervisor overhead                   |
| Density          | Hundreds per host                     | Tens per host                        |
| Security         | Weaker isolation (shared kernel)      | Stronger isolation (separate kernels) |
| Portability      | Image runs anywhere Docker runs       | Requires compatible hypervisor        |

Containers are **not** lightweight VMs — they are isolated **processes** that share the host kernel, using Linux kernel
features:

- **Namespaces** — isolate what a process can see (PID, network, mount, user, UTS, IPC, cgroup).
- **Cgroups** — limit what a process can use (CPU, memory, I/O, network bandwidth).
- **Union filesystem** — layer images efficiently (OverlayFS).

---

## Architecture

```
  CLI (docker)  ──REST API──▶  Docker Daemon (dockerd)
                                     │
                          ┌──────────┼──────────┐
                          ▼          ▼          ▼
                     containerd   BuildKit   Networking
                          │
                          ▼
                      runc (OCI)
                          │
                          ▼
                    Linux kernel
                  (namespaces, cgroups)
```

| Component    | Responsibility                                                  |
|--------------|-----------------------------------------------------------------|
| `docker` CLI | User-facing client — sends commands to the daemon               |
| `dockerd`    | Daemon — manages images, containers, networks, volumes          |
| `containerd` | Container runtime — manages container lifecycle                 |
| `runc`       | OCI runtime — actually creates and runs containers using kernel primitives |
| `BuildKit`   | Modern build engine — parallel builds, cache mounts, secrets    |

---

## Images

An image is a **read-only template** for creating containers. It consists of **ordered layers** — each layer is a
filesystem diff produced by a Dockerfile instruction.

### Layers

```dockerfile
FROM node:20-alpine       # base layer
WORKDIR /app              # metadata only (no new layer)
COPY package*.json ./     # layer 1
RUN npm ci                # layer 2
COPY . .                  # layer 3
RUN npm run build         # layer 4
```

- Each `RUN`, `COPY`, `ADD` instruction creates a new layer.
- Layers are **cached** — if nothing changed in a layer or its predecessors, Docker reuses the cache.
- Layers are **shared** between images — if two images use the same base, they share those layers on disk.

### Image Naming

```
registry/repository:tag
docker.io/library/nginx:1.25-alpine
         └─ repo ──┘ └── tag ──┘

# digest (immutable, content-addressable)
nginx@sha256:abc123...
```

- `latest` is the **default** tag, not "most recent" — it's just a name. Always pin versions in production.

### Image Commands

```bash
docker build -t myapp:1.0 .            # build from Dockerfile
docker pull nginx:1.25-alpine           # download from registry
docker push myregistry/myapp:1.0        # upload to registry
docker images                           # list local images
docker image prune                      # remove unused images
docker image inspect myapp:1.0          # show image details (layers, config)
docker history myapp:1.0                # show layer history
```

---

## Dockerfile

### Instruction Reference

| Instruction    | Purpose                                                         |
|----------------|-----------------------------------------------------------------|
| `FROM`         | Base image — every Dockerfile must start with this              |
| `RUN`          | Execute command during build (creates a layer)                  |
| `COPY`         | Copy files from build context into the image                    |
| `ADD`          | Like `COPY` but can extract tarballs and fetch URLs (prefer `COPY`) |
| `WORKDIR`      | Set working directory for subsequent instructions               |
| `ENV`          | Set environment variable (persists into running container)      |
| `ARG`          | Build-time variable (not available at runtime)                  |
| `EXPOSE`       | Document which port the container listens on (does not publish) |
| `CMD`          | Default command when container starts (can be overridden)       |
| `ENTRYPOINT`   | Main executable (not easily overridden, `CMD` becomes arguments)|
| `VOLUME`       | Create a mount point for external volumes                       |
| `USER`         | Set the user for subsequent instructions and container runtime  |
| `HEALTHCHECK`  | Define a command to check container health                      |
| `LABEL`        | Add metadata (maintainer, version, description)                 |

### `CMD` vs `ENTRYPOINT`

| Scenario             | `ENTRYPOINT`             | `CMD`                       | Container runs           |
|----------------------|--------------------------|-----------------------------|--------------------------|
| CMD only             | —                        | `["node", "server.js"]`     | `node server.js`         |
| ENTRYPOINT only      | `["node"]`               | —                           | `node`                   |
| Both                 | `["node"]`               | `["server.js"]`             | `node server.js`         |
| Override at runtime  | `["node"]`               | `["server.js"]`             | `docker run myapp test.js` → `node test.js` |

- Use `ENTRYPOINT` for the fixed executable.
- Use `CMD` for default arguments that users can override.
- Prefer **exec form** `["executable", "arg1"]` over shell form `executable arg1` — exec form runs the process
  directly (PID 1), shell form wraps in `/bin/sh -c` (signal handling issues).

### Multi-Stage Builds

Reduce final image size by separating build dependencies from runtime:

```dockerfile
# --- build stage ---
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- production stage ---
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The final image only contains the `node:20-alpine` base + built artifacts — no source code, devDependencies, or build
tools.

### Dockerfile Best Practices

1. **Use specific base image tags** — `node:20.11-alpine`, not `node:latest`.
2. **Order instructions by change frequency** — static layers first, frequently changing layers last (maximizes cache).
3. **Combine `RUN` commands** — fewer layers, smaller image, and clean up in the same layer:
   ```dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends \
       curl \
       ca-certificates \
       && rm -rf /var/lib/apt/lists/*
   ```
4. **Use `.dockerignore`** — exclude `node_modules`, `.git`, `dist`, `.env`, etc. from the build context.
5. **Run as non-root** — `USER node` or `USER 1000`.
6. **Use multi-stage builds** — separate build tools from the runtime image.
7. **Use `COPY` over `ADD`** — `ADD` has implicit behavior (tar extraction, URL fetching) that can surprise.
8. **Set `HEALTHCHECK`** — so orchestrators know when the container is ready.
9. **Don't store secrets in images** — use build-time `--secret` mounts or runtime environment variables.
10. **Use `--no-install-recommends`** for apt — avoids installing unnecessary packages.

### `.dockerignore`

```
.git
node_modules
dist
*.md
.env
.env.*
Dockerfile
docker-compose*.yml
.dockerignore
```

---

## Container Lifecycle

### States

```
Created  ──▶  Running  ──▶  Paused
                │               │
                ▼               ▼
             Stopped  ◀──  (unpause)
                │
                ▼
             Removed
```

### Container Commands

```bash
docker run -d --name myapp -p 8080:3000 myapp:1.0   # create + start (detached)
docker run --rm -it myapp:1.0 /bin/sh                 # interactive + remove on exit

docker ps                     # list running containers
docker ps -a                  # list all containers (including stopped)
docker logs myapp             # view stdout/stderr
docker logs -f myapp          # follow logs
docker exec -it myapp sh      # open shell in running container
docker inspect myapp          # detailed container info (JSON)

docker stop myapp             # send SIGTERM, then SIGKILL after grace period
docker kill myapp             # send SIGKILL immediately
docker start myapp            # start a stopped container
docker restart myapp          # stop + start

docker rm myapp               # remove stopped container
docker rm -f myapp            # force remove (even running)
docker container prune        # remove all stopped containers
```

### `docker run` Key Flags

| Flag                | Purpose                                              |
|---------------------|------------------------------------------------------|
| `-d`                | Detached mode (run in background)                    |
| `-it`               | Interactive + TTY (for shell access)                 |
| `--rm`              | Remove container when it exits                       |
| `-p 8080:3000`      | Publish port (host:container)                        |
| `-v /host:/container` | Bind mount                                        |
| `--mount`           | More explicit mount syntax                           |
| `-e KEY=VALUE`      | Set environment variable                             |
| `--env-file .env`   | Load environment from file                           |
| `--name myapp`      | Assign a name                                        |
| `--network mynet`   | Connect to a network                                 |
| `--restart always`  | Restart policy                                       |
| `--memory 512m`     | Memory limit                                         |
| `--cpus 1.5`        | CPU limit                                            |
| `--user 1000`       | Run as specific user                                 |
| `--read-only`       | Read-only root filesystem                            |

---

## Networking

### Network Drivers

| Driver    | Use case                                         | Container-to-container     |
|-----------|--------------------------------------------------|----------------------------|
| `bridge`  | Default — isolated network on a single host      | By container name (DNS)    |
| `host`    | Container shares host's network namespace        | `localhost`                |
| `none`    | No networking                                    | —                          |
| `overlay` | Multi-host networking (Docker Swarm)             | By service name            |
| `macvlan` | Assign a MAC address — container appears on LAN  | Like a physical device     |

### User-Defined Bridge Networks

The default `bridge` network does not provide DNS resolution between containers. Always create a custom network:

```bash
docker network create mynet

docker run -d --name api --network mynet api:1.0
docker run -d --name db  --network mynet postgres:16

# inside the api container, "db" resolves to the postgres container's IP
```

### Port Publishing

```bash
docker run -p 8080:3000 myapp           # host:container (all interfaces)
docker run -p 127.0.0.1:8080:3000 myapp # bind to localhost only
docker run -p 8080:3000/udp myapp       # UDP
docker run -P myapp                     # publish all EXPOSE'd ports to random host ports
```

---

## Volumes and Storage

### Storage Types

| Type         | Managed by Docker? | Persistence      | Use case                    |
|--------------|--------------------|-------------------|-----------------------------|
| Named volume | Yes                | Survives container removal | Database data, uploads |
| Bind mount   | No (host path)     | Host filesystem   | Source code in development  |
| tmpfs mount  | No (in memory)     | Container lifetime | Sensitive data, temp files |

### Volume Commands

```bash
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune               # remove unused volumes
```

### Mounting

```bash
# named volume
docker run -v mydata:/var/lib/postgresql/data postgres:16

# bind mount
docker run -v $(pwd)/src:/app/src myapp:1.0

# tmpfs
docker run --tmpfs /tmp myapp:1.0

# --mount syntax (more explicit, preferred in production)
docker run --mount type=volume,source=mydata,target=/data myapp:1.0
docker run --mount type=bind,source=$(pwd)/src,target=/app/src myapp:1.0
```

### Volume in Dockerfile

```dockerfile
VOLUME /var/lib/postgresql/data
```

This creates an **anonymous volume** — data persists across container restarts but is hard to manage. Prefer explicit
named volumes at `docker run` time.

---

## Docker Compose

Defines and runs **multi-container applications** in a single YAML file:

```yaml
# compose.yaml (v2 syntax, no "version" field needed)
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:3000"
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

### Compose Commands

```bash
docker compose up -d              # start all services (detached)
docker compose up -d --build      # rebuild images before starting
docker compose down               # stop and remove containers, default network
docker compose down -v            # also remove volumes
docker compose logs -f api        # follow logs for a service
docker compose ps                 # list running services
docker compose exec api sh        # exec into a running service
docker compose build              # build/rebuild images
docker compose pull               # pull latest images
```

### `depends_on` with Health Checks

`depends_on` only controls **startup order** by default. With `condition: service_healthy`, Compose waits until the
dependency's healthcheck passes before starting the dependent service.

### Profiles

Run subsets of services:

```yaml
services:
  api:
    build: .

  debug-tools:
    image: busybox
    profiles: ["debug"]
```

```bash
docker compose up -d                 # starts only api
docker compose --profile debug up -d # starts api + debug-tools
```

---

## Security Best Practices

1. **Run as non-root** — `USER 1000` in Dockerfile, or `--user 1000:1000` at runtime.
2. **Use minimal base images** — `alpine`, `distroless`, or `scratch` (for Go binaries).
3. **Scan images for vulnerabilities** — `docker scout`, Trivy, Snyk.
4. **Don't store secrets in images** — use runtime env vars, Docker secrets, or `--secret` in BuildKit.
5. **Read-only root filesystem** — `--read-only` with explicit tmpfs/volume mounts where writes are needed.
6. **Drop capabilities** — `--cap-drop ALL --cap-add NET_BIND_SERVICE`.
7. **Use `no-new-privileges`** — `--security-opt no-new-privileges` prevents privilege escalation.
8. **Pin image digests in production** — `nginx@sha256:abc...` instead of `nginx:1.25`.
9. **Don't run SSH in containers** — use `docker exec` instead.
10. **Limit resources** — `--memory`, `--cpus` to prevent a single container from starving the host.

### Distroless vs Alpine vs Scratch

| Base image    | Size     | Shell? | Package manager? | Use case                        |
|---------------|----------|--------|------------------|---------------------------------|
| `scratch`     | 0 MB     | No     | No               | Statically linked Go/Rust binaries |
| `distroless`  | ~2-20 MB | No     | No               | Java, Node, Python (no shell = smaller attack surface) |
| `alpine`      | ~5 MB    | Yes    | apk              | General purpose, debugging possible |
| `debian-slim` | ~80 MB   | Yes    | apt              | When you need glibc and standard tools |

---

## Build Optimization

### Layer Caching

```dockerfile
# BAD — cache busted on every code change
COPY . .
RUN npm ci

# GOOD — dependencies cached until package.json changes
COPY package*.json ./
RUN npm ci
COPY . .
```

### BuildKit Features

```bash
# enable BuildKit (default in Docker 23.0+)
DOCKER_BUILDKIT=1 docker build .
```

```dockerfile
# cache mount — persist package manager cache between builds
RUN --mount=type=cache,target=/root/.npm npm ci

# secret mount — use secrets during build without leaking into layers
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci

# bind mount — avoid COPY for build-only files
RUN --mount=type=bind,source=package.json,target=package.json npm ci
```

### Build Arguments

```dockerfile
ARG NODE_VERSION=20
FROM node:${NODE_VERSION}-alpine

ARG BUILD_ENV=production
ENV NODE_ENV=${BUILD_ENV}
```

```bash
docker build --build-arg NODE_VERSION=22 --build-arg BUILD_ENV=staging .
```

`ARG` values are visible in `docker history` — never use for secrets.

---

## Debugging

```bash
# inspect running container
docker inspect myapp | jq '.[0].State'
docker stats                              # live resource usage
docker top myapp                          # processes inside container

# view filesystem changes
docker diff myapp                         # files added/changed/deleted vs image

# export filesystem for inspection
docker export myapp > fs.tar

# build debugging — run a failed intermediate layer
docker run --rm -it <intermediate-image-id> sh

# override entrypoint for debugging
docker run --rm -it --entrypoint sh myapp:1.0
```

---

## Common Interview Questions

### What is the difference between an image and a container?

An **image** is a read-only template (filesystem + metadata). A **container** is a running (or stopped) instance of an
image — it adds a writable layer on top. You can create multiple containers from the same image.

### What is a dangling image?

An image that is not tagged and not referenced by any container — typically old layers left after rebuilding.
`docker image prune` removes them. `docker image prune -a` removes all unused images (not just dangling).

### What is the difference between `COPY` and `ADD`?

Both copy files into the image. `ADD` additionally:
- Auto-extracts local `.tar` archives.
- Can fetch files from URLs (but this is discouraged — use `curl`/`wget` in a `RUN` instead).

Use `COPY` unless you specifically need tar extraction.

### How do you reduce Docker image size?

1. Use multi-stage builds — separate build tools from runtime.
2. Use minimal base images (`alpine`, `distroless`, `scratch`).
3. Combine `RUN` commands and clean up in the same layer.
4. Use `.dockerignore` to exclude unnecessary files from build context.
5. Remove caches (`rm -rf /var/lib/apt/lists/*`, `npm cache clean`).
6. Install only production dependencies (`npm ci --omit=dev`).

### What happens when a container is stopped?

`docker stop` sends **SIGTERM** to PID 1 inside the container. The application has a grace period (default 10s,
configurable with `--stop-timeout` or `--time`) to shut down cleanly. After the grace period, Docker sends **SIGKILL**.

If PID 1 is a shell (`/bin/sh -c ...`), it does **not** forward signals to child processes — use exec form in
`ENTRYPOINT`/`CMD` to avoid this.

### What is the difference between `docker stop` and `docker kill`?

- `docker stop` — graceful: SIGTERM → grace period → SIGKILL.
- `docker kill` — immediate: sends SIGKILL (or a specified signal with `--signal`).

### How do you persist data in Docker?

- **Named volumes** — managed by Docker, best for production data (databases, uploads).
- **Bind mounts** — map a host directory into the container, best for development (source code).
- **tmpfs** — in-memory mount, best for sensitive temporary data.

Data in the container's writable layer is **lost** when the container is removed.

### What is the difference between `CMD` and `ENTRYPOINT`?

- `CMD` — default command, easily overridden by `docker run <image> <command>`.
- `ENTRYPOINT` — fixed executable, `CMD` provides default arguments that can be overridden.
- Both together: `ENTRYPOINT` is the executable, `CMD` is the default arguments.

### How do containers communicate with each other?

On a **user-defined bridge network**, containers can reach each other by container name (Docker provides DNS
resolution). On the default bridge network, DNS does not work — you must use IP addresses or `--link` (deprecated).
For multi-host communication, use **overlay** networks (Docker Swarm) or an external orchestrator (Kubernetes).
