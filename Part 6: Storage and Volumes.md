# **Part 6: Storage and Volumes**

> 📌 Containers are ephemeral by default — but **data shouldn’t be**.  
> This guide teaches you how to **persist, protect, and manage data** in Docker.

---

## 1. **Persistent Storage in Docker**

### 🔹 Why Do We Need Persistent Storage?

By default, **all container data is temporary**:
- Files written inside a container are lost when it’s removed.
- Restarting a container creates a fresh filesystem layer.

This is fine for **stateless apps** (e.g., web frontends), but **disastrous for stateful apps** like:
- Databases (PostgreSQL, MySQL)
- File servers
- Caches (Redis with persistence)
- Logs, uploads, configurations

> 💡 **Persistent storage** ensures data survives container restarts, updates, and deletions.

---

## 2. **The Problem with Ephemeral Containers**

### ❌ Data Loss Example:
```bash
docker run -d --name db postgres
# DB writes data to /var/lib/postgresql/data
docker stop db && docker rm db
docker run -d --name db postgres  # New container → empty data directory!
```

> 📉 All your data is gone — because it was stored in the container’s **writable layer**, which is deleted.

---

### ✅ Solution: Use **Persistent Storage**

Docker provides **three types of mounts**:
| Type | Host Access | Persistence | Use Case |
|------|------------|------------|--------|
| **Named Volumes** | Managed by Docker | ✅ Yes | Production data (databases) |
| **Bind Mounts** | Direct host path | ✅ Yes | Development, config files |
| **tmpfs** | In-memory only | ❌ No (volatile) | Sensitive data, temporary files |

We’ll explore each in detail.

---

## 3. **When to Use Volumes vs. Bind Mounts vs. tmpfs**

| Feature | Named Volume | Bind Mount | tmpfs |
|--------|--------------|----------|------|
| **Managed by Docker** | ✅ Yes | ❌ No | ✅ Yes |
| **Best for Production** | ✅ Yes | ⚠️ Risky | ✅ For secrets |
| **Best for Development** | ❌ Overkill | ✅ Yes (code sync) | ✅ Temp files |
| **Host Path Control** | ❌ No | ✅ Full control | ❌ No |
| **Cross-Platform** | ✅ Yes | ❌ No (path differences) | ✅ Yes |
| **Performance** | Good | Best (direct access) | Fastest (RAM) |
| **Security** | High (isolated) | Medium (host exposure) | High (not on disk) |

### 🛠️ Decision Guide:

| Use Case | Recommended Type |
|--------|------------------|
| Database data (PostgreSQL, MySQL) | ✅ **Named Volume** |
| Live code reload in dev | ✅ **Bind Mount** |
| Configuration files | ✅ **Bind Mount** |
| Sensitive data (tokens, keys) | ✅ **tmpfs** |
| Caching (Redis, Memcached) | ✅ **tmpfs** or **volume** |
| Logs (long-term) | ✅ **Named Volume** |
| CI/CD temp workspace | ✅ **tmpfs** |

---

## 4. **Named Volumes**

### 🔹 What is a Named Volume?

A **Docker-managed volume** with a name. Docker handles the storage location (usually `/var/lib/docker/volumes/`).

#### ✅ Benefits:
- Portable across environments
- Easy to back up
- Isolated from host filesystem
- Works with orchestration (Swarm, Kubernetes)

---

### 🔹 Creating, Managing, and Inspecting Volumes

#### Create a Volume:
```bash
docker volume create db-data
```

#### List Volumes:
```bash
docker volume ls
```

#### Inspect a Volume:
```bash
docker volume inspect db-data
```
Output:
```json
{
    "CreatedAt": "2024-04-05T10:00:00Z",
    "Driver": "local",
    "Labels": {},
    "Mountpoint": "/var/lib/docker/volumes/db-data/_data",
    "Name": "db-data",
    "Scope": "local"
}
```

#### Remove a Volume:
```bash
docker volume rm db-data
```

> ⚠️ Cannot remove volume if in use by a container.

#### Clean Up Unused Volumes:
```bash
docker volume prune
docker volume prune -f  # Force, no prompt
```

---

### 🔹 Using a Named Volume in `docker run`

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v db-data:/var/lib/postgresql/data \
  postgres
```

> ✅ Data persists even if container is deleted.

#### Reuse Volume in New Container:
```bash
docker run -d --name postgres-new -v db-data:/var/lib/postgresql/data postgres
```

> 🔁 Same data — perfect for upgrades.

---

## 5. **Backup and Restore Strategies**

Since volumes are critical, **backup them regularly**.

---

### 🔹 Backup a Volume

Use a temporary container to access the volume:

```bash
# Backup: mount volume and tar contents
docker run --rm \
  -v db-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/db-data.tar.gz -C /data .
```

> Creates `db-data.tar.gz` on host.

---

### 🔹 Restore a Volume

```bash
# Restore: create volume and extract
docker volume create db-data-restored

docker run --rm \
  -v db-data-restored:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/db-data.tar.gz -C /data
```

> ✅ Now use `db-data-restored` in your container.

---

### 🔹 Automate Backups (Script Example)
```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
docker run --rm \
  -v db-data:/data \
  -v /backups:/backup \
  alpine tar czf /backup/db-$DATE.tar.gz -C /data .

# Optional: rotate old backups
find /backups -name "db-*.tar.gz" -mtime +7 -delete
```

> ✅ Schedule with `cron` or CI/CD.

---

## 6. **Volume Drivers (Local, NFS, Cloud)**

Docker supports **pluggable volume drivers** for advanced storage.

| Driver | Use Case |
|-------|--------|
| `local` | Default — stores on host disk |
| `nfs` | Share volumes across hosts via NFS |
| `aws ebs`, `gcp pd` | Cloud block storage (ECS, GKE) |
| `sshfs`, `s3fs` | Custom remote storage |
| `portworx`, `longhorn` | Enterprise storage solutions |

---

### 🔹 Example: NFS Volume
```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/nfs \
  nfs-data
```

Now use:
```bash
docker run -v nfs-data:/app/data myapp
```

> ✅ Shared across multiple containers or hosts.

---

### 🔹 Third-Party Drivers
Install plugins:
```bash
docker plugin install vieux/sshfs
```

Then:
```bash
docker volume create \
  --driver vieux/sshfs \
  --opt sshcmd=user@host:/tmp \
  --opt password=xxx \
  ssh-volume
```

> ✅ Mount remote directories securely.

---

## 7. **Bind Mounts**

### 🔹 What is a Bind Mount?

Mount a **specific host directory or file** into a container.

#### Syntax:
```bash
-v /host/path:/container/path
--mount type=bind,source=/host/path,target=/container/path
```

> ✅ `--mount` is more explicit and recommended for production.

---

### 🔹 Development Use Cases (Live Code Reload)

Perfect for **hot-reloading code** during development.

#### Example: Node.js App
```bash
docker run -d \
  --name dev-app \
  -p 3000:3000 \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/package.json:/app/package.json \
  node:18 \
  sh -c "npm install && npm run dev"
```

> ✅ Edit code on host → changes appear instantly in container.

---

### 🔹 Host Path vs. Container Path

| Part | Description | Example |
|------|-------------|--------|
| **Host Path** | Directory on the host machine | `/Users/you/project/src` |
| **Container Path** | Where it’s mounted inside container | `/app/src` |

> ⚠️ **Absolute paths only** — relative paths may not work.

---

### 🔹 Security and Performance Considerations

#### ⚠️ Security Risks:
- Container can modify host files
- Host file permissions apply inside container
- Risk of privilege escalation

#### Best Practices:
- Avoid bind-mounting sensitive dirs (`/etc`, `/root`)
- Use non-root user in container
- Prefer **named volumes** in production

#### Performance:
- ✅ **Fastest** option — direct filesystem access
- ❌ Can cause issues on macOS/Windows due to file sharing overhead

> 💡 On Docker Desktop, configure file sharing paths in Settings.

---

## 8. **tmpfs Mounts**

### 🔹 What is tmpfs?

A **temporary filesystem stored in memory (RAM)** — never written to disk.

#### Use Cases:
- Session data
- Caching
- Sensitive data (tokens, keys)
- Temporary files

> ✅ Data is **deleted when container stops**.

---

### 🔹 Create a tmpfs Mount

```bash
docker run -d \
  --name cache \
  --tmpfs /tmp \
  --tmpfs /app/cache:rw,noexec \
  myapp
```

Or with `--mount`:
```bash
docker run -d \
  --name secure-app \
  --mount type=tmpfs,target=/run/secrets \
  myapp
```

---

### 🔹 Use Cases and Limitations

#### ✅ **Use When:**
- Storing temporary files
- Holding secrets in memory
- Improving performance (RAM > disk)
- Preventing disk writes (security/compliance)

#### ❌ **Limitations:**
- **Not persistent** — lost on container stop
- Limited by available RAM
- Not suitable for large datasets
- Cannot be shared between containers (unless using `tmpfs` on host)

> 💡 Combine with `--read-only` for ultra-secure apps:
```bash
docker run --read-only --tmpfs /tmp myapp
```

---

## ✅ Summary: Storage Decision Matrix

| Requirement | Solution |
|-----------|----------|
| Database persistence | ✅ Named Volume |
| Live code reload | ✅ Bind Mount |
| Configuration files | ✅ Bind Mount |
| Sensitive runtime data | ✅ tmpfs |
| Shared storage across hosts | ✅ NFS / Cloud Volume |
| Backup strategy | ✅ Volume + `tar` in sidecar container |
| Read-only container | ✅ `--read-only` + tmpfs for writable needs |
| CI/CD workspace | ✅ tmpfs (fast, secure) |

---

## 🚀 Pro Tips

1. **Use named volumes in production** — they’re safer and more portable.
2. **Use bind mounts only in dev** — avoid host dependency in prod.
3. **Never store secrets in volumes** — use Docker Secrets or HashiCorp Vault.
4. **Backup volumes regularly** — automate with scripts.
5. **Use `--mount` syntax** — more explicit than `-v`.
6. **Monitor volume size** — runaway logs can fill disk.
7. **Use `tmpfs` for session data** — improves security and speed.

---
