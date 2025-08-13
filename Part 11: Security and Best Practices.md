# **Part 11: Security and Best Practices**

> 🛡️ Containers are not inherently secure — **security is a shared responsibility**.  
> This guide teaches you how to **harden Docker**, avoid common pitfalls, and follow industry best practices.

---

## 1. **Docker Security Fundamentals**

### 🔹 The `docker` Group = Root Access

#### ⚠️ Critical Risk:
Adding a user to the `docker` group grants **root-level privileges**.

Why?
- Docker daemon runs as `root`
- Containers can mount host directories
- A malicious container can escape to host

#### Example:
```bash
docker run -v /:/host alpine chroot /host /bin/sh
# You now have full root access to the host!
```

> 🚫 This is **not container breakout** — it’s **intended behavior** due to Docker’s architecture.

---

### ✅ Best Practices:
| Practice | How |
|--------|-----|
| **Limit `docker` group access** | Only trusted users |
| **Use role-based access control (RBAC)** | In Kubernetes or Docker EE |
| **Avoid `docker` group in production** | Use `sudo` or SSH-based access |
| **Audit container activity** | Use `docker events`, logging, monitoring |

> ✅ Principle: **Least privilege** — only give what’s necessary.

---

### 🔹 Rootless Docker and User Namespaces

#### 🔹 Rootless Docker
Run the Docker daemon **without root privileges**.

- Daemon runs under your user
- No need for `sudo`
- Great for shared or restricted environments

#### Enable Rootless Mode:
```bash
dockerd-rootless-setuptool.sh install
```

> ✅ Available on Linux. Requires `fuse-overlayfs`, `slirp4netns`.

> 🔐 Safer, but limited performance and networking.

---

### 🔹 User Namespaces (Userns)

Map container UIDs to unprivileged host UIDs.

#### Enable in `/etc/docker/daemon.json`:
```json
{
  "userns-remap": "default"
}
```

Now container `root` (UID 0) maps to host `dockremap` user (e.g., UID 165536).

#### Verify:
```bash
docker run alpine id
# Output: uid=65534(nobody) → remapped
```

> ✅ Prevents container root from being host root.

> ⚠️ Breaks volume sharing with host — paths must be owned by remapped UID.

---

## 2. **Secure Image Building**

### 🔹 Non-Root Users in Containers (`USER` directive)

Never run your app as `root` inside the container.

#### ❌ Bad:
```Dockerfile
FROM node:18
COPY . /app
WORKDIR /app
RUN npm install
CMD ["node", "app.js"]
# Runs as root!
```

#### ✅ Good:
```Dockerfile
FROM node:18

# Create non-root user
RUN addgroup --system nodejs && adduser --system --ingroup nodejs nodejs

COPY . /app
WORKDIR /app

RUN npm install --omit=dev
RUN chown -R nodejs:nodejs /app

USER nodejs  # Switch to non-root
CMD ["node", "app.js"]
```

> ✅ Even if compromised, attacker has limited privileges.

---

### 🔹 Minimal Base Images

Smaller image = fewer vulnerabilities.

| Image | Size | Use Case |
|------|------|--------|
| `alpine` | ~5MB | General purpose (musl libc) |
| `distroless` | ~2MB | Production-only (no shell) |
| `scratch` | 0MB | Static binaries only |
| `ubuntu:20.04` | ~70MB | Legacy apps needing glibc |

#### Example: Distroless
```Dockerfile
FROM gcr.io/distroless/nodejs:18
COPY package*.json ./
COPY build/ /app/
```

> ✅ No shell, no package manager — **extremely secure**.

> ⚠️ Can’t `docker exec` — but that’s the point.

---

### 🔹 Vulnerability Scanning (Trivy, Snyk, Clair)

Scan images for **CVEs (security vulnerabilities)**.

#### 🔹 Trivy (Open Source)
```bash
trivy image myapp:latest
```

Output:
```
Total vulnerabilities: 12
CRITICAL: 2 (e.g., CVE-2023-1234 in openssl)
HIGH: 5
```

Integrate into CI:
```yaml
- name: Scan image
  run: |
    docker build -t myapp .
    trivy image --exit-code 1 --severity CRITICAL myapp
```

> ✅ Fail build if critical vulnerabilities found.

---

#### 🔹 Snyk (Commercial)
```bash
snyk container test myapp:latest
snyk monitor  # Send results to dashboard
```

> ✅ Great for teams needing dashboards, policies, and compliance.

---

#### 🔹 Clair (CNCF, for air-gapped envs)
Run as service:
```bash
docker run -d -p 6060:6060 quay.io/coreos/clair
```

Scan with `grype` or `kube-bench`.

> ✅ Ideal for on-prem, regulated environments.

---

## 3. **Runtime Security**

Protect containers **while they’re running**.

---

### 🔹 Dropping Capabilities (`--cap-drop`)

Linux **capabilities** are fine-grained root powers (e.g., `CAP_NET_ADMIN`, `CAP_SYS_MODULE`).

By default, Docker drops most, but keeps:
- `CHOWN`, `DAC_OVERRIDE`, `FSETID`, `SETGID`, etc.

#### Drop All, Then Add Back:
```bash
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  myapp
```

Now container can bind to port 80, but can’t modify firewall or load kernel modules.

> ✅ Principle: **Drop everything you don’t need**.

---

### 🔹 Seccomp, AppArmor, and SELinux Profiles

#### 🔹 Seccomp (Secure Computing Mode)
Filter **system calls** a container can make.

Docker uses a **default seccomp profile** that blocks dangerous syscalls (`ptrace`, `mount`, `reboot`).

##### Use Custom Profile:
```bash
docker run \
  --security-opt seccomp=/path/to/seccomp.json \
  myapp
```

> ✅ Disable unnecessary syscalls to reduce attack surface.

---

#### 🔹 AppArmor (Ubuntu/Debian)
Mandatory Access Control (MAC) system.

##### Load Profile:
```bash
sudo apparmor_parser -r -W myapp-profile
```

##### Run with Profile:
```bash
docker run \
  --security-opt apparmor=myapp-profile \
  myapp
```

> ✅ Restrict file access, network, execution.

---

#### 🔹 SELinux (RHEL/CentOS/Fedora)
Security-Enhanced Linux — strict MAC policies.

##### Label Volumes:
```bash
docker run -v /data:/app/data:Z myapp
# `Z` = private unshared label
```

##### Run with SELinux enforcement:
```bash
docker run \
  --security-opt label=level:s0:c100,c200 \
  myapp
```

> ✅ Required in high-security environments (government, finance).

---

### 🔹 Read-Only Filesystems (`--read-only`)

Prevent any writes to the container’s filesystem.

```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  myapp
```

> ✅ Stops malware from writing files or backdoors.

> ⚠️ Use `tmpfs` for required writable paths (`/tmp`, `/run`).

---

## 4. **Secrets Management**

Never hardcode secrets in images or environment variables.

---

### 🔹 Docker Secrets (Swarm Mode)

Secure way to pass secrets to containers.

#### Create a Secret:
```bash
echo "mypass" | docker secret create db_password -
```

#### Use in Service:
```bash
docker service create \
  --name db \
  --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres
```

> ✅ Secrets are:
- Encrypted at rest and in transit
- Mounted as tmpfs (not on disk)
- Only available to authorized services

> ⚠️ Only works in **Swarm mode**.

---

### 🔹 Environment Variables (Not Recommended for Secrets)

```bash
docker run -e DB_PASS=secret myapp
```

#### Risks:
- Visible in `docker inspect`
- Leaked in logs
- Stored in shell history

> ❌ Avoid for production secrets.

---

### 🔹 Mount Secrets as Files (Better)

```bash
docker run \
  -v ./secrets/db-pass.txt:/run/secrets/db-pass:ro \
  -e DB_PASS_FILE=/run/secrets/db-pass \
  myapp
```

> ✅ Safer than env vars — but still requires file management.

---

### 🔹 Vault Integration (Best for Production)

Use **HashiCorp Vault** or **AWS Secrets Manager**.

#### Example: Fetch from Vault
```Dockerfile
CMD sh -c "export DB_PASS=$(vault read -field=password secret/db) && python app.py"
```

Or use **Vault Agent**, **Sidecar**, or **CSI Driver**.

> ✅ Dynamic, audited, rotated secrets.

> ✅ Ideal for microservices and zero-trust environments.

---

## ✅ Summary: Docker Security Checklist

| Area | Best Practice |
|------|---------------|
| **Host Access** | Don’t give `docker` group to untrusted users |
| **Daemon** | Use rootless mode or userns remapping |
| **Images** | Use minimal base images (Alpine, distroless) |
| **User** | Always use `USER` to run as non-root |
| **Build** | Use `.dockerignore`, multi-stage builds |
| **Scanning** | Scan with Trivy/Snyk in CI |
| **Capabilities** | `--cap-drop=ALL` + only add needed |
| **Seccomp** | Use custom or default profile |
| **Filesystem** | `--read-only` + `tmpfs` for writable dirs |
| **Network** | Use user-defined networks, limit exposure |
| **Secrets** | Use Docker Secrets (Swarm), Vault, or files — never env vars |
| **Logging** | Monitor `docker events`, logs, and audits |

---

## 🚀 Pro Tips

1. **Run `docker scan myapp`** — Docker’s built-in vulnerability scanner.
2. **Use `docker bench security`** to audit your host against CIS Docker Benchmark.
3. **Never run `--privileged`** unless absolutely necessary.
4. **Use `no-new-privileges`** to prevent privilege escalation:
   ```bash
   docker run --security-opt no-new-privileges myapp
   ```
5. **Limit container resources** to prevent DoS:
   ```bash
   --memory=512m --cpus=1.0
   ```
6. **Sign images** with Docker Content Trust (DCT) for integrity.
7. **Rotate secrets regularly** — especially in production.

---
