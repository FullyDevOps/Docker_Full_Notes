# **Part 2: Installation and Setup**

---

## **1. Installing Docker**

Before you can use Docker, you need to install the **Docker Engine** or **Docker Desktop**, depending on your operating system.

> 🛠️ Docker Engine: The core runtime (Linux)  
> Docker Desktop: GUI + Engine (macOS & Windows)

We'll cover installation on all major platforms, including remote setups.

---

## **2. Docker Engine on Linux (Ubuntu, CentOS, etc.)**

### ✅ Why Install on Linux?
- Most production environments run Docker on Linux.
- Best performance and full feature support.
- Direct access to the kernel and container runtime.

---

### **A. Installing on Ubuntu (20.04/22.04)**

#### Step 1: Update System & Install Dependencies
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

#### Step 2: Add Docker’s Official GPG Key
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

#### Step 3: Add Docker APT Repository
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### Step 4: Install Docker Engine
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> ✅ `docker-ce`: Docker Community Edition  
> ✅ `containerd.io`: Container runtime  
> ✅ `docker-buildx-plugin`: Enables advanced builds (BuildKit)  
> ✅ `docker-compose-plugin`: Adds `docker compose` command

#### Step 5: Start and Enable Docker
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

### **B. Installing on CentOS / RHEL / Rocky Linux**

#### Step 1: Set Up Repository
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### Step 2: Install Docker
```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Step 3: Start Docker
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

> 💡 On RHEL/CentOS, ensure SELinux and firewalld are configured properly.

---

### **C. Verify Installation**
After installation, verify:
```bash
docker version
docker info
```

You should see client and server (daemon) details.

---

## **3. Docker Desktop for macOS and Windows**

### ✅ What is Docker Desktop?

A complete Docker environment with:
- Docker Engine
- Docker CLI
- Docker Compose
- Kubernetes (optional)
- Dashboard (GUI)
- Integration with host OS (file sharing, networking)

Available for:
- **macOS** (Intel & Apple Silicon)
- **Windows 10/11 Pro, Enterprise, or Education** (requires WSL2 or Hyper-V)

---

### **A. macOS Installation**

#### Prerequisites:
- macOS 11 (Big Sur) or later
- Intel or Apple M1/M2 chip
- At least 4GB RAM

#### Steps:
1. Download from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
2. Open `.dmg` file and drag Docker to Applications
3. Launch Docker Desktop
4. Grant permissions when prompted

> 🍏 M1/M2 Macs: Use ARM64 version for best performance.

---

### **B. Windows Installation**

#### Prerequisites:
- Windows 10/11 Pro, Enterprise, or Education
- **WSL2 (Windows Subsystem for Linux 2)** installed
- BIOS virtualization enabled

#### Steps:
1. Enable WSL2:
   ```powershell
   wsl --install
   ```
2. Reboot and log in
3. Download Docker Desktop for Windows
4. Run installer and follow prompts
5. Launch Docker Desktop

> 💡 Uses WSL2 backend (not Hyper-V by default) — faster and more efficient.

---

### **C. What Docker Desktop Includes**
| Feature | Description |
|--------|-------------|
| **GUI Dashboard** | View containers, images, logs, networks |
| **Kubernetes** | Optional local cluster |
| **Dev Environments** | Experimental feature for cloud dev |
| **File Sharing** | Mount host directories into containers |
| **Settings** | Configure resources, proxies, registries |

> ⚠️ Free for personal use, small businesses, and education. Paid for enterprise.

---

## **4. Remote Docker Hosts and Contexts**

### ✅ Why Use Remote Hosts?
- Run containers on powerful servers or cloud VMs
- Keep local machine lightweight
- Centralized container management

---

### **A. What is a Docker Context?**
A **context** defines a Docker endpoint (local or remote). You can switch between them.

Default context: `default` → local Docker daemon

---

### **B. Create a Remote Docker Host (Example: Ubuntu VM)**

#### On the Remote Server:
1. Install Docker (as shown above)
2. Configure Docker Daemon to accept remote connections

Edit `/etc/docker/daemon.json`:
```json
{
  "hosts": ["tcp://0.0.0.0:2375", "unix://"]
}
```

> ⚠️ Only use `tcp://` if secured (see TLS below)

Restart Docker:
```bash
sudo systemctl restart docker
```

> 🔒 Open port 2375 in firewall (not secure without TLS)

---

### **C. Connect from Local Machine**

#### Option 1: Using DOCKER_HOST (Temporary)
```bash
export DOCKER_HOST=tcp://<remote-ip>:2375
docker info  # Now talks to remote host
```

#### Option 2: Create a Named Context (Permanent)
```bash
docker context create remote-vm \
  --docker "host=tcp://<remote-ip>:2375"

docker context use remote-vm
```

Now all `docker` commands go to the remote host.

Switch back:
```bash
docker context use default
```

> ✅ Use contexts to manage dev/staging/prod environments.

---

### **D. Securing Remote Access (TLS)**
Exposing port 2375 without encryption is dangerous.

Use **TLS certificates** to encrypt communication:
- Generate CA, server, and client certs
- Configure daemon.json with `tlsverify`
- Copy client certs to local machine

> 🔐 Production best practice: Always use TLS or SSH tunneling.

---

## **5. Verifying Installation: `docker version`, `docker info`**

After installation, always verify Docker is working.

---

### **`docker version`**
Shows version of **Client** (CLI) and **Server** (Daemon)

```bash
$ docker version
Client:
 Version:    24.0.7
 API version:  1.43
 Go version:   go1.20.7

Server:
 Engine:
  Version:     24.0.7
  API version: 1.43 (minimum version 1.12)
```

> ✅ If both appear, Docker is installed and daemon is running.

---

### **`docker info`**
Detailed system-wide info:

```bash
$ docker info
Client:
 Context:    default
 Debug Mode: false

Server:
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2
 Images: 5
 Server Version: 24.0.7
 Storage Driver: overlay2
 CPUs: 4
 Memory: 7.7GiB
```

Key things to check:
- **Storage Driver**: Should be `overlay2` (modern default)
- **Cgroup Driver**: `systemd` (recommended for systemd-based distros)
- **Number of Containers/Images**: Confirms functionality

> ✅ Run `docker run hello-world` to test end-to-end.

---

## **6. Post-Installation Configuration**

After installation, configure Docker for security, performance, and usability.

---

### **A. Adding User to Docker Group (Security Implications)**

By default, only **root** can run Docker commands.

To avoid typing `sudo docker ...`, add your user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

> Log out and back in for changes to apply.

#### ✅ Benefit:
- Run `docker` commands without `sudo`

#### ⚠️ Security Risk:
- Members of the `docker` group have **root-level privileges**
- A malicious container can escalate to host root
- Example: `docker run -v /:/host ubuntu chroot /host`

> 🛡️ Best Practice:
- Only add trusted users
- In production, restrict access via IAM or RBAC
- Consider using **rootless mode** for development

---

### **B. Configuring Docker Daemon Settings (`daemon.json`)**

The Docker daemon is configured via `/etc/docker/daemon.json`.

Create or edit the file:
```bash
sudo nano /etc/docker/daemon.json
```

#### Example Configuration:
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "features": {
    "buildkit": true
  },
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["myregistry.local:5000"],
  "tlsverify": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "hosts": ["unix://", "tcp://0.0.0.0:2376"]
}
```

#### Key Options Explained:

| Option | Purpose |
|-------|--------|
| `exec-opts` | Use `systemd` cgroup driver (recommended for Ubuntu/CentOS) |
| `log-driver` | Control log format and rotation |
| `storage-driver` | Filesystem driver (always `overlay2` on modern systems) |
| `features.buildkit` | Enable modern build system |
| `registry-mirrors` | Speed up pulls via regional mirrors |
| `insecure-registries` | Allow HTTP (non-TLS) private registries |
| `tls*` | Secure remote access |
| `hosts` | Define daemon endpoints |

> ⚠️ After editing `daemon.json`, restart Docker:
```bash
sudo systemctl restart docker
```

---

### **C. Setting Up Private Registries and Proxies**

#### 1. **Private Registry**
Use a private Docker registry (e.g., AWS ECR, GitLab Registry, self-hosted).

##### Login:
```bash
docker login myregistry.local:5000
```

##### Push Image:
```bash
docker tag myapp myregistry.local:5000/myapp:v1
docker push myregistry.local:5000/myapp:v1
```

> 🔐 Use TLS and authentication (Basic Auth or OAuth).

---

#### 2. **Registry Mirrors**
Speed up image pulls by using mirrors (e.g., in China or behind slow links).

Example: Use Google’s mirror
```json
{
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

> No login needed — transparent caching.

---

#### 3. **HTTP/HTTPS Proxies**
If behind a corporate firewall:

Create `/etc/systemd/system/docker.service.d/http-proxy.conf`:
```ini
[Service]
Environment="HTTP_PROXY=http://proxy.corp.com:8080"
Environment="HTTPS_PROXY=https://proxy.corp.com:8080"
Environment="NO_PROXY=localhost,127.0.0.1,.internal"
```

Reload and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

### **D. Enabling Experimental Features (BuildKit, etc.)**

Docker includes **experimental features** that may become stable.

#### 1. **Enable BuildKit (Highly Recommended)**

BuildKit is a modern, faster, more secure builder.

Enable in `daemon.json`:
```json
{
  "features": {
    "buildkit": true
  }
}
```

Or use environment variable:
```bash
export DOCKER_BUILDKIT=1
docker build -t myapp .
```

#### ✅ BuildKit Benefits:
- Faster builds (parallel processing)
- Better caching
- Secret mounting: `--mount=type=secret`
- SSH mounting
- Cleaner output

> 🛠️ Use `# syntax=docker/dockerfile:1` in Dockerfile for advanced syntax.

#### 2. **Other Experimental Features**
- **Docker Dev Environments** (in Docker Desktop)
- **Stargz snapshotter** (lazy pulling)
- **Image signing** (Notary)

> ⚠️ Experimental = may change or break. Avoid in production unless tested.

---

## ✅ Summary: Installation & Setup Checklist

| Task | Command / File |
|------|----------------|
| Install Docker Engine | `apt install docker-ce` or `yum install docker-ce` |
| Install Docker Desktop | Download from docker.com |
| Verify Install | `docker version`, `docker info` |
| Add user to docker group | `sudo usermod -aG docker $USER` |
| Configure daemon | `/etc/docker/daemon.json` |
| Set up remote host | `docker context create` |
| Enable BuildKit | `"features": { "buildkit": true }` |
| Configure proxy | systemd drop-in or `daemon.json` |
| Use private registry | `docker login`, `insecure-registries` |

---

## 📌 Best Practices

1. **Never run untrusted containers** — they can compromise the host.
2. **Use `docker group` cautiously** — it’s equivalent to root.
3. **Enable BuildKit** — better builds, security, and speed.
4. **Use `overlay2` storage driver** — it’s the default and most stable.
5. **Set log rotation** — avoid disk filling up with logs.
6. **Use TLS for remote hosts** — never expose `2375` publicly.
7. **Keep Docker updated** — security patches are critical.

---
