# **Part 5: Container Lifecycle and Runtime Management**

> 📌 Containers are not just "run and forget" — they have a **full lifecycle** and require **intentional management** for reliability, observability, and resilience.

---

## 1. **Container Lifecycle Explained**

Every container goes through a well-defined **lifecycle**:

```
[Created] → [Started] → [Running] → [Stopped] → [Removed]
```

Let’s break down each stage.

---

### 🔹 **1. Create**
The container is **prepared** but not yet running.

- Filesystem layers are set up
- Network and volume mounts are configured
- Metadata (env, ports, CMD) is applied

#### Command:
```bash
docker create [OPTIONS] IMAGE [COMMAND]
```

#### Example:
```bash
docker create --name myapp -p 8080:80 nginx
```

> ✅ Use `create` when you want to **configure first, run later**.

---

### 🔹 **2. Start**
The container begins execution.

- Process defined in `CMD` or `ENTRYPOINT` starts
- Network interfaces are activated
- Volumes are mounted

#### Commands:
```bash
docker start myapp          # Start a created container
docker run nginx            # = create + start
```

> 🔄 Containers can be **started, stopped, and restarted** multiple times.

---

### 🔹 **3. Run (Running State)**
The container is actively executing its main process.

- Logs are generated
- Accepts traffic (if networked)
- Uses CPU/memory

> ⚠️ If the **main process exits**, the container stops — even if other processes are still running.

---

### 🔹 **4. Stop**
The container is gracefully shut down.

- Docker sends `SIGTERM` to the main process
- Waits (default: 10 seconds)
- If still running, sends `SIGKILL`

#### Command:
```bash
docker stop myapp
```

> ✅ Always prefer `stop` over `kill` for graceful shutdown.

---

### 🔹 **5. Remove**
The container is deleted from the system.

- Metadata and network config removed
- **Volumes are preserved** unless `--volumes` is used

#### Command:
```bash
docker rm myapp
```

> ❌ Cannot remove a **running** container unless `-f` (force) is used.

---

### 🔁 Lifecycle Diagram

```
docker create
     ↓
   Created → docker start → Running → docker stop → Stopped → docker rm → [Gone]
     ↑_________________________________________|
                   docker restart
```

> 💡 A container can be **restarted** from the "Stopped" state.

---

## 2. **Ephemeral vs. Persistent Containers**

| Type | Description | Use Case |
|------|-------------|--------|
| **Ephemeral** | Short-lived, disposable | CI/CD jobs, one-off tasks, debugging |
| **Persistent** | Long-running, resilient | Web servers, databases, APIs |

### ✅ Ephemeral Example:
```bash
docker run --rm alpine ls /tmp
```
- Runs once, exits, auto-removes (`--rm`)
- No state, no restart

### ✅ Persistent Example:
```bash
docker run -d --restart=always --name db postgres
```
- Runs 24/7
- Auto-restarts on failure
- Uses volumes for data

> 🛠️ Design choice: **Stateless apps → ephemeral**, **stateful apps → persistent**.

---

## 3. **Understanding Exit Codes**

When a container stops, it returns an **exit code** indicating why.

| Exit Code | Meaning |
|---------|--------|
| `0` | Success — process completed normally |
| `1` | General error |
| `125` | Docker daemon error (e.g., `docker run` failed) |
| `126` | Command cannot be invoked (e.g., permission denied) |
| `127` | Command not found |
| `137` | Container killed by `SIGKILL` (often due to OOM — out of memory) |
| `143` | Container stopped gracefully with `SIGTERM` |
| `>128` | Killed by signal (e.g., `137 = 128 + 9` → `SIGKILL`) |

#### Check Exit Code:
```bash
docker inspect myapp --format '{{ .State.ExitCode }}'
```

#### Example: OOM Killer
```bash
docker run -m 10m alpine dd if=/dev/zero of=/tmp/test
# Fails with exit code 137 → OOMKilled
```

> ✅ Use `docker events` or `docker inspect` to debug failures.

---

## 4. **Running Containers in Production**

In production, containers must be **reliable, observable, and self-healing**.

---

### 🔹 **Detached Mode (`-d`) vs. Foreground**

| Mode | Command | Use Case |
|------|--------|--------|
| **Foreground** | `docker run nginx` | Debugging, learning, short tasks |
| **Detached** | `docker run -d nginx` | Production, background services |

> ✅ **Always use `-d` in production** — don’t block your terminal.

---

### 🔹 **Restart Policies (`--restart`)**
Tell Docker **when to restart** a container after it exits.

| Policy | Behavior |
|-------|--------|
| `no` | Never restart (default) |
| `on-failure[:max-retries]` | Restart only if exit code ≠ 0 |
| `always` | Always restart, even after manual `docker stop` |
| `unless-stopped` | Always restart, **except** if manually stopped |

#### Examples:
```bash
# Auto-restart on crash
docker run -d --restart=on-failure:3 myapp

# Survive reboots, but respect manual stop
docker run -d --restart=unless-stopped postgres
```

> ✅ **Best practice**: Use `unless-stopped` for critical services.

---

### 🔹 **Health Checks and Liveness Probes**

Ensure your app is **not just running, but healthy**.

#### In Dockerfile:
```Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

#### Or in `docker run`:
```bash
docker run -d \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=10s \
  --health-retries=3 \
  myapp
```

#### Check Health:
```bash
docker inspect myapp --format '{{ .State.Health.Status }}'
# → healthy / unhealthy / starting
```

> ✅ Orchestration tools (Kubernetes, Swarm) use this to restart unhealthy containers.

---

### 🔹 **Signal Handling and Graceful Shutdown**

Apps must handle `SIGTERM` to **shut down cleanly** (e.g., close DB connections, flush logs).

#### Problem:
- Many apps only handle `SIGINT` (Ctrl+C), not `SIGTERM`
- Docker sends `SIGTERM`, not `SIGINT`

#### Solution:
Use a proper init process or trap signals.

##### Example: Bash Script with Trap
```Dockerfile
COPY shutdown.sh /shutdown.sh
CMD trap "echo 'Shutting down'; exit 0" SIGTERM; \
    while :; do sleep 10; done
```

##### Or use `tini` (init process):
```Dockerfile
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "app.py"]
```

> ✅ `tini` forwards `SIGTERM` to child processes.

---

## 5. **Debugging and Troubleshooting Containers**

Even the best containers fail. Here’s how to debug.

---

### 🔹 **Reading Logs Effectively**

Use `docker logs` with smart flags.

| Flag | Purpose |
|------|--------|
| `-f` | Follow (like `tail -f`) |
| `--tail N` | Show last N lines |
| `--since TIME` | Show logs since (e.g., `1h`, `2024-04-05T10:00:00`) |
| `-t` | Show timestamps |

#### Examples:
```bash
docker logs -f --tail 50 web
docker logs --since 30m db
docker logs -t $(docker ps -lq)  # Logs from last container
```

> ✅ Use structured logging (JSON) for easier parsing.

---

### 🔹 **Executing Commands Inside Running Containers**

Use `docker exec` to inspect or fix issues.

#### Common Debug Commands:
```bash
# Open shell
docker exec -it web bash

# Check processes
docker exec web ps aux

# Test connectivity
docker exec web curl -v http://api:3000

# View config files
docker exec web cat /etc/nginx/nginx.conf
```

> ⚠️ Avoid `exec` for long-running services — use proper orchestration.

---

### 🔹 **Inspecting Container Configuration and State**

Use `docker inspect` to get **everything** about a container.

#### Key Fields:
```bash
# Is it running?
docker inspect web --format '{{ .State.Running }}'

# What’s the IP?
docker inspect web --format '{{ .NetworkSettings.IPAddress }}'

# Mounted volumes?
docker inspect web --format '{{ json .Mounts }}'

# Environment variables?
docker inspect web --format '{{ range .Config.Env }}{{ println . }}{{ end }}'

# Full config (JSON)
docker inspect web
```

> ✅ Use Go templates (`--format`) to extract specific data.

---

## 6. **Common Failure Scenarios and Fixes**

Here are **real-world problems** and how to solve them.

---

### ❌ **1. Container Exits Immediately (Exit Code 0)**
**Cause**: Command completes instantly (e.g., `echo "hello"`).

**Fix**: Run a long-running process.
```bash
# ❌ Bad
docker run alpine echo "hello"

# ✅ Good
docker run -d alpine tail -f /dev/null
```

---

### ❌ **2. Container Crashes (Exit Code 1, 127, 137)**
**Diagnose**:
```bash
docker logs myapp
docker inspect myapp --format '{{ .State.Error }}'
```

| Exit Code | Fix |
|---------|-----|
| `127` | Command not found → Fix `CMD` or `PATH` |
| `137` | OOM → Increase memory (`-m 512m`) |
| `1` | App error → Check logs, config, dependencies |

---

### ❌ **3. Port Already in Use**
```bash
Error: driver failed programming external connectivity...
```

**Cause**: Host port is taken.

**Fix**:
- Change host port: `-p 8081:80`
- Stop conflicting process: `lsof -i :8080` → `kill <PID>`

---

### ❌ **4. Can’t Connect to Service Inside Container**
**Check**:
- Is the app listening on `0.0.0.0`, not `localhost`?
- Is the port exposed (`EXPOSE`) and published (`-p`)?
- Is the firewall blocking the port?

```bash
docker exec web netstat -tuln | grep 80
```

---

### ❌ **5. File Not Found / Permission Denied**
**Cause**: Incorrect `COPY` path, wrong user, or bind mount issue.

**Fix**:
- Verify paths in `Dockerfile`
- Use `docker cp` to inspect: `docker cp web:/app .`
- Run as root temporarily: `docker run -u root ...`

---

### ❌ **6. Health Check Failing**
**Diagnose**:
```bash
docker inspect web --format '{{ json .State.Health }}'
```

**Fix**:
- Increase `--start-period` if app takes time to boot
- Fix health endpoint (`/health` should return 200)
- Test manually: `docker exec web curl localhost:8080/health`

---

## ✅ Summary: Production-Ready Container Checklist

| Requirement | Command / Setting |
|-----------|------------------|
| Run in background | `docker run -d` |
| Auto-restart on failure | `--restart=unless-stopped` |
| Monitor health | `HEALTHCHECK` or `--health-cmd` |
| Handle shutdown | Use `tini` or trap `SIGTERM` |
| Limit resources | `-m 512m --cpus=1.0` |
| Name containers | `--name web` |
| Use non-root user | `USER 1000` |
| View logs easily | `docker logs -f web` |
| Inspect state | `docker inspect web` |
| Clean up old containers | `docker system prune` |

---

## 🚀 Pro Tips

1. **Never rely on `latest`** — use semantic tags or digests.
2. **Use `--rm` for testing** — avoids clutter.
3. **Log to stdout/stderr** — don’t write to files inside container.
4. **Use `.dockerignore`** — speeds up builds.
5. **Monitor exit codes** — they tell the real story.
6. **Use restart policies** — self-healing systems.
7. **Test graceful shutdown** — simulate `docker stop`.

---
