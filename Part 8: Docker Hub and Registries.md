# **Part 8: Docker Hub and Registries**

> 📌 Docker images don’t run themselves — they’re stored and shared via **registries**.  
> This guide teaches you how to **find, push, pull, secure, and manage images** across public and private registries.

---

## 1. **Docker Hub: Public Image Registry**

### 🔹 What is Docker Hub?

**[Docker Hub](https://hub.docker.com)** is the **default public registry** for Docker images. It’s like **GitHub for container images** — a central place to:
- **Search** for images
- **Pull** official and community images
- **Store** your public/private images
- **Automate builds** from GitHub/GitLab

URL: `https://hub.docker.com`

---

### 🔹 Official vs. User Images

| Type | Description | Example |
|------|-------------|--------|
| **Official Images** | Curated, trusted, minimal, maintained by Docker or upstream projects | `nginx`, `redis`, `ubuntu`, `python` |
| **User (Community) Images** | Published by individuals or organizations | `bitnami/nginx`, `jvanz/redis-exporter` |

#### How to Identify Official Images:
- ✅ **No username prefix**: `redis` (official) vs. `myuser/redis` (user)
- ✅ Green "Docker Official Image" badge on Docker Hub
- ✅ High pull count and regular updates

> ✅ **Best Practice**: Prefer official images in production.

---

### 🔹 Searching and Pulling Images

#### Search via CLI:
```bash
docker search nginx
docker search --filter is-official=true ubuntu
```

#### Search via Web:
Visit [https://hub.docker.com](https://hub.docker.com) and search.

#### Pull an Image:
```bash
docker pull nginx
docker pull python:3.9-slim
docker pull alpine@sha256:abc123...  # By digest (secure)
```

> ✅ Always verify the image source before pulling.

---

### 🔹 Automated Builds from GitHub/GitLab

Docker Hub can **automatically build and publish images** when you push to a Git repo.

#### Setup Steps:
1. Link your GitHub/GitLab account to Docker Hub
2. Create an **Automated Build** repository
3. Link to a GitHub repo (e.g., `youruser/myapp`)
4. Define build context and Dockerfile path
5. Set build triggers (e.g., on `main` branch)

#### Result:
- Git push → Docker Hub builds image → Pushes to `youruser/myapp`

> ✅ Great for CI/CD without Jenkins/GitHub Actions.

> ⚠️ Deprecated in favor of **GitHub Actions** or **Docker Scout**, but still functional.

---

## 2. **Private Registries**

You shouldn’t store proprietary or sensitive images on public registries.

### 🔹 Why Use a Private Registry?
- Keep source code and configs private
- Control image access
- Meet compliance (GDPR, HIPAA)
- Reduce dependency on public internet
- Speed up pulls via local caching

---

### 🔹 Docker Registry (Open Source)

Docker’s **open-source registry server** — easy to self-host.

#### Run It:
```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  registry:2
```

Now you can push to `localhost:5000/myapp`

#### Push an Image:
```bash
docker tag myapp localhost:5000/myapp:v1
docker push localhost:5000/myapp:v1
```

#### Pull from Another Machine:
```bash
docker pull 192.168.1.100:5000/myapp:v1
```

> ⚠️ **HTTP by default** — insecure. Use TLS in production.

---

### 🔹 Secure the Registry with TLS

1. Generate SSL cert (or use Let's Encrypt)
2. Run with TLS:
```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v /path/to/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

Now use: `https://myregistry.com:5000/myapp`

---

### 🔹 Add Authentication (Basic Auth)

Use `htpasswd` to secure access:

```bash
docker run --entrypoint htpasswd httpd:alpine -Bbn user password > auth/htpasswd

docker run -d \
  --name registry \
  -p 5000:5000 \
  -v /path/to/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2
```

Now `docker login myregistry.com:5000` required.

---

## 3. **Cloud Registries**

Major cloud providers offer **managed, secure, scalable** container registries.

| Registry | Provider | URL | CLI Example |
|---------|----------|-----|-------------|
| **ECR** | AWS | `aws_account.dkr.ecr.region.amazonaws.com` | `aws ecr get-login-password` |
| **GCR** | Google Cloud | `gcr.io`, `us.gcr.io` | `gcloud auth configure-docker` |
| **ACR** | Azure | `myregistry.azurecr.io` | `az acr login --name myregistry` |
| **GHCR** | GitHub | `ghcr.io/username/repo` | `echo $TOKEN \| docker login ghcr.io -u username --password-stdin` |

---

### 🔹 AWS ECR (Elastic Container Registry)

#### Login:
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com
```

#### Push:
```bash
docker tag myapp:1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

> ✅ Integrates with IAM, ECS, and Kubernetes.

---

### 🔹 GitHub Container Registry (GHCR)

#### Push:
```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u your-github-username --password-stdin

docker tag myapp ghcr.io/your-github-username/myapp:v1
docker push ghcr.io/your-github-username/myapp:v1
```

> ✅ Tightly integrated with GitHub Actions and Packages.

---

### 🔹 Google GCR

#### Setup:
```bash
gcloud auth configure-docker
```

#### Push:
```bash
docker tag myapp gcr.io/my-project/myapp:v1
docker push gcr.io/my-project/myapp:v1
```

> ✅ Works with Google Kubernetes Engine (GKE).

---

### 🔹 Azure ACR

#### Login:
```bash
az acr login --name myregistry
```

#### Push:
```bash
docker tag myapp myregistry.azurecr.io/myapp:v1
docker push myregistry.azurecr.io/myapp:v1
```

> ✅ Integrates with Azure Kubernetes Service (AKS).

---

## 4. **Self-Hosted Harbor**

### 🔹 What is Harbor?

**[Harbor](https://goharbor.io)** is an **open-source, enterprise-grade registry server** by CNCF (same as Kubernetes).

#### Features:
- Web UI
- Role-based access control (RBAC)
- Image scanning (vulnerabilities)
- Image signing (Notary)
- Replication (sync across regions)
- AD/LDAP integration
- Multi-tenancy

> ✅ Ideal for **on-prem, air-gapped, or regulated environments**.

---

### 🔹 Run Harbor

Use Docker Compose:
```bash
git clone https://github.com/goharbor/harbor
cd harbor
cp harbor.yml.tmpl harbor.yml
# Edit: hostname, HTTPS, admin password
docker-compose up -d
```

Access via: `https://your-harbor.com`

---

### 🔹 Push to Harbor
```bash
docker login your-harbor.com
docker tag myapp your-harbor.com/library/myapp:v1
docker push your-harbor.com/library/myapp:v1
```

> ✅ Full audit trail, vulnerability scanning, and policy enforcement.

---

## 5. **Pushing and Pulling Images**

### 🔹 General Workflow

#### 1. Build and Tag:
```bash
docker build -t myapp:v1 .
```

#### 2. Tag for Registry:
```bash
docker tag myapp:v1 myregistry.com/myapp:v1
```

#### 3. Login:
```bash
docker login myregistry.com
```

#### 4. Push:
```bash
docker push myregistry.com/myapp:v1
```

#### 5. Pull (on another machine):
```bash
docker pull myregistry.com/myapp:v1
```

> ✅ Always **tag before pushing** — the registry path is part of the tag.

---

## 6. **Authentication: `docker login`**

Used to authenticate to any registry.

#### Syntax:
```bash
docker login [OPTIONS] [SERVER]
```

#### Examples:
```bash
docker login                    # Defaults to Docker Hub
docker login docker.io          # Explicit
docker login myregistry.com
docker login ghcr.io
```

#### Use Token (Recommended):
```bash
echo "your-personal-access-token" | docker login ghcr.io -u username --password-stdin
```

> ✅ Avoid plain text passwords. Use **tokens** or **credential helpers**.

---

## 7. **Tagging for Registries**

Image tags include the **registry URL** as part of the name.

#### Format:
```
[registry/][namespace/]repository[:tag|@digest]
```

#### Examples:
| Image | Meaning |
|------|--------|
| `nginx` | Docker Hub, official |
| `bitnami/nginx` | Docker Hub, user image |
| `myregistry.com/myapp:v1` | Private registry |
| `ghcr.io/user/app:v2` | GitHub Container Registry |
| `10.0.0.10:5000/internal:latest` | Local insecure registry |

> ✅ Use **semantic versioning** (`v1.2.0`) — avoid `latest` in production.

---

## 8. **Image Signing and Content Trust**

Ensure images are **authentic and untampered**.

### 🔹 Docker Content Trust (DCT)

Uses **Notary** to sign images with cryptographic keys.

#### Enable:
```bash
export DOCKER_CONTENT_TRUST=1
```

#### Push Signed Image:
```bash
docker push myregistry.com/myapp:v1
# Prompts for signing key
```

#### Pull Signed Image:
```bash
docker pull myregistry.com/myapp:v1
# Verifies signature — fails if unsigned or tampered
```

> ✅ Required for compliance (e.g., FedRAMP, SOC2).

---

### 🔹 Key Management

- **Root key**: Master key (keep offline)
- **Repository keys**: Per-image signing
- **Timestamp key**: Prevents replay attacks

Keys stored in `~/.docker/trust/`

> 🔐 Rotate keys regularly and back up securely.

---

## ✅ Summary: Registry Decision Matrix

| Use Case | Recommended Registry |
|--------|------------------------|
| Public open-source project | ✅ Docker Hub or GHCR |
| Enterprise internal apps | ✅ Harbor or ECR/ACR/GCR |
| Air-gapped environment | ✅ Harbor or Docker Registry |
| AWS-based infrastructure | ✅ ECR |
| GitHub-centric CI/CD | ✅ GHCR |
| Need vulnerability scanning | ✅ Harbor or ECR |
| Need fine-grained access control | ✅ Harbor or ACR |
| Simple local caching | ✅ Docker Registry |
| Compliance & audit trail | ✅ Harbor with Content Trust |

---

## 🚀 Pro Tips

1. **Never use `latest` in production** — always tag with version or digest.
2. **Use immutable tags** — don’t retag `v1.0` to point to a new image.
3. **Scan images** for vulnerabilities (use Harbor, Trivy, or Snyk).
4. **Enable Content Trust** in regulated environments.
5. **Use short-lived tokens** — not passwords.
6. **Rotate credentials** regularly.
7. **Monitor pull counts** — unexpected spikes may indicate misuse.
8. **Set up image retention policies** — auto-delete old tags.

---
