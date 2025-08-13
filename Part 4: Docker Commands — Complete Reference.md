# **Part 4: Docker Commands — Complete Reference**

---

## 🔹 Table of Contents

1. [**Essential Commands (Beginner Level)**](#1-essential-commands-beginner-level)
2. [**Image Management Commands**](#2-image-management-commands)
3. [**Container Lifecycle Commands**](#3-container-lifecycle-commands)
4. [**Intermediate Flags & Options**](#4-intermediate-commands-and-flags)
5. [**Advanced Debugging & Inspection**](#5-advanced-commands-and-debugging)
6. [**System & Cleanup Commands**](#6-system-and-cleanup-commands)
7. [**Networking & Volume Commands**](#7-networking-and-volume-commands)
8. [**Registry & Security Commands**](#8-registry-and-security-commands)
9. [**Command Patterns & Workflows**](#9-command-patterns-and-workflows)

---

## 1. Essential Commands (Beginner Level)

These are the **core commands** you’ll use daily to run, view, and manage containers.

---

### 🔹 `docker run` — Create and Start a Container

**Purpose**: Run a new container from an image.

#### Syntax:
```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

#### Examples:
```bash
# Run in foreground
docker run hello-world

# Run in background (detached)
docker run -d nginx

# Run with name, port, and interactive shell
docker run -d --name web -p 8080:80 nginx

# Interactive container (e.g., for debugging)
docker run -it ubuntu bash
```

#### Key Flags:
| Flag | Meaning |
|------|--------|
| `-d` | Detached mode (run in background) |
| `-i` | Keep STDIN open |
| `-t` | Allocate a TTY (terminal) |
| `--name` | Assign a readable name |
| `-p` | Publish port (host:container) |
| `-v` | Mount a volume |
| `-e` | Set environment variable |
| `--rm` | Remove container when stopped |

> 💡 `docker run` = `docker create` + `docker start`

---

### 🔹 `docker ps` — List Containers

**Purpose**: Show running (or all) containers.

#### Examples:
```bash
docker ps                    # Only running
docker ps -a                 # All containers (running + stopped)
docker ps -q                 # Quiet: only container IDs
docker ps -l                 # Last created container
docker ps --no-trunc         # Full command (don’t truncate)
```

#### Filter Output:
```bash
docker ps --filter "status=exited"
docker ps --filter "name=web"
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

> ✅ Use `-q` in scripts: `docker stop $(docker ps -q)`

---

### 🔹 `docker logs` — View Container Logs

**Purpose**: Fetch logs from a container’s stdout/stderr.

#### Examples:
```bash
docker logs web
docker logs -f web           # Follow (like tail -f)
docker logs -t web           # Show timestamps
docker logs --tail 50 web    # Last 50 lines
docker logs --since 1h web   # Logs from last hour
```

> 📁 Logs are stored in JSON format at `/var/lib/docker/containers/<id>/*.log`

---

### 🔹 `docker exec` — Run Commands in a Running Container

**Purpose**: Execute a command inside a **running** container.

#### Examples:
```bash
docker exec -it web bash     # Open shell
docker exec web ls /app      # Run one command
docker exec -u root web apt update  # Run as root
```

> ⚠️ Use only for debugging or maintenance — not for starting services.

---

### 🔹 `docker version` — Show Docker Version

**Purpose**: Display version of Docker Client and Server (Daemon).

#### Usage:
```bash
docker version
```

#### Output:
```
Client:
 Version:    24.0.7
 Go version: go1.20.7

Server:
 Engine:
  Version:        24.0.7
  API version:    1.43
```

> ✅ Verify Docker is installed and daemon is running.

---

### 🔹 `docker info` — System-Wide Information

**Purpose**: Show detailed info about Docker host.

#### Usage:
```bash
docker info
```

Includes:
- Number of containers/images
- Storage driver (`overlay2`)
- Cgroup driver (`systemd`)
- Kernel version
- CPUs, Memory
- Registry settings

> ✅ Use to troubleshoot installation or performance issues.

---

## 2. Image Management Commands

Manage **images** — build, pull, tag, inspect, remove.

---

### 🔹 `docker build` — Build an Image from a Dockerfile

**Purpose**: Create an image using instructions in a `Dockerfile`.

#### Syntax:
```bash
docker build [OPTIONS] PATH | URL | -
```

#### Examples:
```bash
docker build -t myapp:1.0 .                     # Tag image
docker build -f Dockerfile.dev -t myapp:dev .   # Custom Dockerfile
docker build --no-cache -t myapp .              # Ignore cache
docker build --build-arg VERSION=2.0 -t myapp . # Pass ARG
```

#### Key Flags:
| Flag | Purpose |
|------|--------|
| `-t` | Tag the image |
| `-f` | Specify Dockerfile |
| `--no-cache` | Don’t use cached layers |
| `--build-arg` | Pass build-time variables |
| `--target` | Multi-stage: build up to a stage |
| `--progress` | Display progress (auto, plain, tty) |

> ✅ Always use `.dockerignore` to exclude unnecessary files.

---

### 🔹 `docker pull` — Download an Image from Registry

**Purpose**: Manually download an image (e.g., for offline use).

#### Examples:
```bash
docker pull nginx
docker pull ubuntu:20.04
docker pull alpine@sha256:abc123...  # By digest (secure)
```

> ✅ Pull before deploying in production to avoid delays.

---

### 🔹 `docker images` — List Locally Stored Images

**Purpose**: Show all images on the host.

#### Examples:
```bash
docker images
docker images myapp          # Filter by repo
docker images -q             # Quiet (only IDs)
docker images --digests      # Show image digests
```

> 🔎 Images with `<none>` tag are **dangling** — can be safely removed.

---

### 🔹 `docker rmi` — Remove One or More Images

**Purpose**: Delete images to free up disk space.

#### Examples:
```bash
docker rmi myapp:1.0
docker rmi $(docker images -q)                    # Remove all images
docker rmi $(docker images -q -f dangling=true)   # Remove dangling only
docker rmi -f myapp                               # Force remove
```

> ⚠️ Cannot remove image used by a container.

---

### 🔹 `docker tag` — Tag an Image

**Purpose**: Assign a new name/tag to an image.

#### Syntax:
```bash
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

#### Examples:
```bash
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 registry.com/myapp:1.0
```

> ✅ Required before `docker push` to a registry.

---

### 🔹 `docker history` — Show Image Build History

**Purpose**: View layers and commands used to build an image.

#### Examples:
```bash
docker history myapp:1.0
docker history --format "{{.CreatedBy}}\t{{.Size}}" myapp
```

> 🔍 Use to:
- Debug image size
- Audit build steps
- Identify outdated packages

---

### 🔹 `docker inspect` (Image) — Get Image Metadata

**Purpose**: View detailed config of an image.

```bash
docker inspect nginx:latest
```

Shows:
- Base OS
- Architecture
- Environment variables
- Entrypoint/CMD
- Layer digests

> ✅ Use to verify image content before deployment.

---

## 3. Container Lifecycle Commands

Control **container creation, start, stop, and removal**.

---

### 🔹 `docker start` — Start a Stopped Container

```bash
docker start web
docker start -a -i web  # Attach and interactive
```

> 🔁 Retains volumes, IP, and filesystem.

---

### 🔹 `docker stop` — Gracefully Stop a Container

Sends `SIGTERM`, waits 10s, then `SIGKILL`.

```bash
docker stop web
docker stop -t 30 web  # Wait 30 seconds
```

> ✅ Use in scripts to gracefully shut down.

---

### 🔹 `docker restart` — Restart a Container

```bash
docker restart web
```

> 🔁 Equivalent to `stop` + `start`.

---

### 🔹 `docker rm` — Remove a Container

```bash
docker rm web
docker rm -f web           # Force (even if running)
docker rm $(docker ps -aq) # Remove all containers
```

> ❌ Cannot remove container with active volumes unless `--volumes`.

---

### 🔹 `docker create` — Create Container (Don’t Start)

```bash
docker create -p 8080:80 --name web nginx
# Later: docker start web
```

> ✅ Useful for pre-configuring containers.

---

## 4. Intermediate Commands and Flags

Fine-tune **resource usage, networking, and configuration**.

---

### 🔹 Resource Limits

| Flag | Purpose |
|------|--------|
| `-m` or `--memory` | Limit RAM (e.g., `-m 512m`) |
| `--memory-swap` | RAM + swap (e.g., `--memory-swap=1g`) |
| `--cpus` | CPU quota (e.g., `--cpus=1.5`) |
| `--cpuset-cpus` | Bind to specific cores (e.g., `--cpuset-cpus="0,1"`) |

#### Example:
```bash
docker run -d --memory=256m --cpus=0.5 myapp
```

---

### 🔹 Port Publishing

| Flag | Purpose |
|------|--------|
| `-p 8080:80` | Map host 8080 → container 80 |
| `-p 80` | Random host port → container 80 |
| `-P` | Publish all `EXPOSE`d ports |

> ✅ Use `-p` for production, `-P` for dev.

---

### 🔹 Environment & Configuration

| Flag | Purpose |
|------|--------|
| `-e VAR=value` | Set environment variable |
| `--env-file .env` | Load from file |
| `--name myapp` | Assign name |
| `--label tier=backend` | Add metadata |
| `--rm` | Auto-remove when stopped |

---

## 5. Advanced Debugging & Inspection

For **troubleshooting, introspection, and automation**.

---

### 🔹 `docker inspect` — Deep Dive into JSON Output

Works for **containers, images, networks, volumes**.

```bash
docker inspect web
```

#### Extract Specific Data (Go Templates):
```bash
# IP Address
docker inspect -f '{{ .NetworkSettings.IPAddress }}' web

# Mount Source
docker inspect -f '{{ range .Mounts }}{{ .Source }}{{ end }}' web

# Image Name
docker inspect -f '{{ .Config.Image }}' web

# Status
docker inspect -f '{{ .State.Running }}' web
```

> ✅ Essential for scripting and debugging.

---

### 🔹 `docker events` — Real-Time Daemon Monitoring

Stream events from Docker daemon.

```bash
docker events
docker events --filter 'event=stop'
docker events --since 30m
```

> ✅ Use in CI/CD or monitoring tools.

---

### 🔹 `docker stats` — Live Resource Usage

Like `top` for containers.

```bash
docker stats
docker stats --no-stream           # Snapshot
docker stats --format "{{.Name}} {{.MemPerc}}"
```

> ✅ Monitor CPU, memory, network, disk.

---

### 🔹 `docker cp` — Copy Files to/from Containers

```bash
docker cp host.txt container:/app/
docker cp container:/app/log.txt ./
```

> ✅ Use for logs, config, debugging.

---

### 🔹 `docker diff` — Show Filesystem Changes

See what files were changed, added, or deleted in a container.

```bash
docker diff web
```

Output:
```
C /etc
A /app/log.txt
D /tmp/file
```

- `A` = Added
- `D` = Deleted
- `C` = Changed

> ✅ Useful for auditing container behavior.

---

## 6. System and Cleanup Commands

Keep Docker **clean and efficient**.

---

### 🔹 `docker system df` — Disk Usage

Show how much space Docker is using.

```bash
docker system df
```

Output:
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         2         1.2GB     800MB (66%)
Containers      3         1         120MB     100MB (83%)
Local Volumes   2         1         50MB      25MB (50%)
Build Cache     -         -         300MB     300MB
```

---

### 🔹 `docker system prune` — Clean Unused Objects

Remove:
- Stopped containers
- Dangling images
- Unused networks
- Build cache

```bash
docker system prune
docker system prune -a               # Remove all unused images
docker system prune --volumes        # Also remove unused volumes
docker system prune -a --volumes --force  # Full cleanup
```

> ✅ Run regularly in dev environments.

---

## 7. Networking and Volume Commands

Manage **networks and persistent storage**.

---

### 🔹 Networking
```bash
docker network ls
docker network create mynet
docker network connect mynet container
docker network disconnect mynet container
docker network rm mynet
docker network inspect mynet
```

> ✅ Use user-defined networks for container communication.

---

### 🔹 Volumes
```bash
docker volume create data
docker volume ls
docker volume inspect data
docker volume rm data
docker volume prune
```

Mount in `run`:
```bash
docker run -v data:/app/storage myapp
```

---

## 8. Registry and Security Commands

Work with **registries, authentication, and image distribution**.

---

### 🔹 `docker login` — Log in to a Registry

```bash
docker login
docker login docker.io
docker login registry.com
```

> ✅ Required before `docker push`.

---

### 🔹 `docker push` — Upload Image to Registry

```bash
docker tag myapp:1.0 registry.com/myapp:1.0
docker push registry.com/myapp:1.0
```

> ✅ Use `--all-tags` to push all tags.

---

### 🔹 `docker save` / `docker load` — Offline Image Transfer

```bash
docker save myapp:1.0 -o myapp.tar
docker load -i myapp.tar
```

> ✅ Use in air-gapped environments.

---

### 🔹 `docker search` — Search Docker Hub

```bash
docker search nginx
docker search --filter is-official=true ubuntu
```

> ✅ Find official or trusted images.

---

## 9. Command Patterns and Workflows

### 🔹 Common One-Liners

| Task | Command |
|------|--------|
| Stop all containers | `docker stop $(docker ps -q)` |
| Remove all containers | `docker rm $(docker ps -aq)` |
| Remove all images | `docker rmi $(docker images -q)` |
| Clean everything | `docker system prune -a --volumes --force` |
| Follow last container | `docker logs -f $(docker ps -lq)` |
| Exec into last container | `docker exec -it $(docker ps -lq) bash` |
| Show all IPs | `docker inspect $(docker ps -q) --format '{{.Name}} {{.NetworkSettings.IPAddress}}'` |

---

### 🔹 Shell Aliases (Add to `~/.bashrc`)

```bash
alias dps='docker ps'
alias di='docker images'
alias dlogs='docker logs -f'
alias dex='docker exec -it'
alias dclean='docker system prune -a --volumes --force'
alias dip='docker inspect -f "{{.Name}} {{.NetworkSettings.IPAddress}}"'
```

Reload: `source ~/.bashrc`

---

### 🔹 CI/CD Scripting Example

```bash
#!/bin/bash
set -e

docker build -t myapp:$TAG .
docker tag myapp:$TAG $REGISTRY/myapp:$TAG
docker login -u $USER -p $PASS $REGISTRY
docker push $REGISTRY/myapp:$TAG
```

---

## ✅ Final Summary: Docker Command Index

| Command | Purpose | Full Syntax | Example |
|--------|--------|-------------|--------|
| `docker run` | Create and start a container from an image | `docker run [OPTIONS] IMAGE [COMMAND] [ARG...]` | `docker run -d --name web -p 8080:80 nginx` |
| `docker build` | Build an image from a Dockerfile | `docker build [OPTIONS] PATH\|URL\| -` | `docker build -t myapp:v1 .` |
| `docker ps` | List containers | `docker ps [OPTIONS]` | `docker ps -a` (show all containers) |
| `docker images` | List locally available images | `docker images [OPTIONS] [REPOSITORY[:TAG]]` | `docker images --filter "dangling=true"` |
| `docker logs` | Fetch logs from a container | `docker logs [OPTIONS] CONTAINER` | `docker logs -f --tail 50 web` |
| `docker exec` | Run a command in a running container | `docker exec [OPTIONS] CONTAINER COMMAND [ARG...]` | `docker exec -it db psql -U user` |
| `docker stop` | Gracefully stop a running container | `docker stop [OPTIONS] CONTAINER [CONTAINER...]` | `docker stop web` |
| `docker start` | Start a stopped container | `docker start [OPTIONS] CONTAINER [CONTAINER...]` | `docker start web` |
| `docker restart` | Restart a container | `docker restart [OPTIONS] CONTAINER [CONTAINER...]` | `docker restart redis-cache` |
| `docker rm` | Remove one or more containers | `docker rm [OPTIONS] CONTAINER [CONTAINER...]` | `docker rm -f $(docker ps -aq)` |
| `docker rmi` | Remove one or more images | `docker rmi [OPTIONS] IMAGE [IMAGE...]` | `docker rmi myapp:v1 nginx:alpine` |
| `docker pull` | Download an image from a registry | `docker pull [OPTIONS] NAME[:TAG\|@DIGEST]` | `docker pull ubuntu:20.04` |
| `docker push` | Upload an image to a registry | `docker push [OPTIONS] NAME[:TAG]` | `docker push myregistry.com/myapp:v1` |
| `docker tag` | Tag an image to a name/repository | `docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]` | `docker tag myapp:1.0 registry.com/myapp:1.0` |
| `docker inspect` | Return low-level info about Docker objects | `docker inspect [OPTIONS] NAME\|ID [NAME\|ID...]` | `docker inspect -f '{{.State.Running}}' web` |
| `docker stats` | Display live resource usage (CPU, memory, etc.) | `docker stats [OPTIONS] [CONTAINER...]` | `docker stats --no-stream` |
| `docker events` | Get real-time events from the Docker daemon | `docker events [OPTIONS]` | `docker events --filter 'event=stop'` |
| `docker system prune` | Remove unused data (containers, images, networks, build cache) | `docker system prune [OPTIONS]` | `docker system prune -a --volumes --force` |
| `docker system df` | Show Docker disk usage | `docker system df [OPTIONS]` | `docker system df` |
| `docker cp` | Copy files/folders between container and host | `docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH`<br>`docker cp [OPTIONS] SRC_PATH CONTAINER:DEST_PATH` | `docker cp ./config.txt web:/app/config.txt` |
| `docker diff` | Show changed files in container's filesystem | `docker diff CONTAINER` | `docker diff web` (shows A=Added, D=Deleted, C=Changed) |
| `docker history` | Show the history of an image | `docker history [OPTIONS] IMAGE` | `docker history --format "{{.CreatedBy}}\t{{.Size}}" myapp` |
| `docker login` | Log in to a Docker registry | `docker login [OPTIONS] [SERVER]` | `docker login docker.io` |
| `docker logout` | Log out from a Docker registry | `docker logout [SERVER]` | `docker logout myregistry.com` |
| `docker save` | Save one or more images to a tar archive | `docker save [OPTIONS] IMAGE [IMAGE...]` | `docker save myapp:v1 -o myapp-v1.tar` |
| `docker load` | Load images from a tar archive | `docker load [OPTIONS]` | `docker load -i myapp-v1.tar` |
| `docker search` | Search Docker Hub for images | `docker search [OPTIONS] TERM` | `docker search --filter is-official=true ubuntu` |
| `docker create` | Create a container without starting it | `docker create [OPTIONS] IMAGE [COMMAND] [ARG...]` | `docker create -p 8080:80 --name web nginx` |
| `docker rename` | Rename a container | `docker rename OLD_NAME NEW_NAME` | `docker rename web web-server` |
| `docker pause` / `unpause` | Pause/unpause all processes in a container | `docker pause CONTAINER`<br>`docker unpause CONTAINER` | `docker pause web` |
| `docker volume create` | Create a volume | `docker volume create [OPTIONS] NAME` | `docker volume create db-data` |
| `docker volume ls` | List volumes | `docker volume ls [OPTIONS]` | `docker volume ls` |
| `docker volume rm` | Remove a volume | `docker volume rm [OPTIONS] NAME [NAME...]` | `docker volume rm data-volume` |
| `docker volume prune` | Remove all unused local volumes | `docker volume prune [OPTIONS]` | `docker volume prune --force` |
| `docker network create` | Create a network | `docker network create [OPTIONS] NETWORK` | `docker network create backend` |
| `docker network ls` | List networks | `docker network ls [OPTIONS]` | `docker network ls` |
| `docker network rm` | Remove one or more networks | `docker network rm NETWORK [NETWORK...]` | `docker network rm isolated-net` |
| `docker network connect` | Connect a container to a network | `docker network connect [OPTIONS] NETWORK CONTAINER` | `docker network connect backend web` |
| `docker network disconnect` | Disconnect a container from a network | `docker network disconnect [OPTIONS] NETWORK CONTAINER` | `docker network disconnect backend web` |
| `docker version` | Show Docker version info | `docker version [OPTIONS]` | `docker version` |
| `docker info` | Display system-wide Docker information | `docker info [OPTIONS]` | `docker info` |

---

### 📌 How to Use This Table

- **Quick Reference**: Look up syntax and examples instantly.
- **Learning**: Study patterns (e.g., most commands follow `docker OBJECT ACTION`).
- **Scripting**: Copy full syntax into automation scripts.
- **Interview Prep**: Memorize key commands and use cases.
- **CI/CD**: Use precise flags for reproducible builds and deploys.

---

## 🚀 Pro Tips

1. **Use `--rm` for short-lived containers** (CI, debugging).
2. **Name containers** — easier than IDs.
3. **Use `.dockerignore`** — speeds up builds.
4. **Enable BuildKit** (`DOCKER_BUILDKIT=1`) — faster, safer builds.
5. **Use `--format` and `--filter`** — powerful for automation.
6. **Prune regularly** — Docker accumulates data silently.
7. **Avoid `latest` in production** — use semantic tags or digests.

---
