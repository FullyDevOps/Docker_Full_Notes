# **Part 3: Docker Images and Dockerfile Mastery**

---

## **1. Understanding Docker Images**

### ✅ What is a Docker Image?

A **Docker image** is a **read-only template** that contains:
- A base operating system (e.g., Ubuntu, Alpine)
- Application code
- Dependencies (libraries, binaries)
- Environment variables
- Metadata (e.g., exposed ports, default command)

It serves as a **blueprint** for creating containers.

> 💡 Think of an image like a **photograph of a fully configured machine** — ready to be turned on (run) as a container.

---

### 🧩 Why "Image" and Not "Container"?

- **Image**: Static, immutable, portable file.
- **Container**: Dynamic, running instance of an image.

> Just like a `.exe` file vs. a running program.

---

## **2. Image Anatomy: Layers, Metadata, and Manifests**

A Docker image is not a single file — it's a **structured collection of components**.

### 🔹 A. Layers (Union File System)

Each instruction in a `Dockerfile` creates a **layer** — a filesystem difference.

#### Example:
```Dockerfile
FROM ubuntu:20.04         → Layer 1: Base OS
RUN apt-get update        → Layer 2: Updated package list
COPY app.py /app/         → Layer 3: App code
RUN pip install flask     → Layer 4: Python dependencies
CMD ["python", "/app/app.py"] → Layer 5: Metadata (not a filesystem layer)
```

#### How Layers Work:
- Each layer is **cached**.
- If you change `app.py`, only Layer 3 and later are rebuilt.
- **Copy-on-write (CoW)**: When a container writes to a file, it copies the file from a read-only layer to a writable top layer.

> ✅ Benefit: Fast builds, efficient storage, shared layers across images.

---

### 🔹 B. Metadata

Metadata defines **how to run** the container:
- `CMD`: Default command
- `EXPOSE`: Which ports the app listens on
- `ENV`: Environment variables
- `LABEL`: Arbitrary key-value labels (e.g., `version=1.0`)

> ❗ Metadata doesn't change the filesystem — it’s stored separately.

---

### 🔹 C. Manifest

The **manifest** is a JSON file that describes:
- Image architecture (e.g., `amd64`, `arm64`)
- OS (`linux`)
- Layer digests
- Configuration

Used when pulling images:
```bash
docker pull nginx:latest
```
→ Docker checks the manifest to find the correct image for your system.

> 🌐 Enables **multi-architecture images** (e.g., one tag for Intel and Apple M1).

---

## **3. Image IDs, Digests, and Tags**

### 🔹 A. Image ID
- A **short hash** (e.g., `a1b2c3d`) representing the image.
- Generated from the image’s configuration and layers.
- Shown in `docker images`.

> ❗ Not content-addressed — can be duplicated.

---

### 🔹 B. Digest
- A **SHA256 hash** of the entire image content.
- Format: `sha256:abc123...`
- Immutable and globally unique.

```bash
docker pull nginx@sha256:abc123...
```

> ✅ Use digests in production for **reproducibility and security**.

---

### 🔹 C. Tags
- Human-readable labels (e.g., `v1.2`, `latest`, `dev`).
- Point to a specific image digest.
- Can be changed (e.g., `latest` can point to different images over time).

```bash
docker tag myapp:v1 myapp:latest
```

> ⚠️ Never use `latest` in production — it’s mutable and unreliable.

---

## **4. Immutable Nature and Content Addressing**

### ✅ Immutability
- Once built, an image **cannot be changed**.
- Ensures consistency: same image behaves the same everywhere.

> 🔄 To update, build a **new image** with a new tag.

### ✅ Content Addressing
- Each layer and image is identified by its **cryptographic hash**.
- If content changes, hash changes.
- Enables caching, deduplication, and trust.

> 🔐 Security benefit: You can verify that an image hasn’t been tampered with.

---

## **5. Writing Effective Dockerfiles**

A `Dockerfile` is a **script** that tells Docker how to build an image.

### 📝 Basic Structure:
```Dockerfile
# Base image
FROM ubuntu:20.04

# Install dependencies
RUN apt-get update && apt-get install -y python3

# Copy application
COPY app.py /app/

# Set working directory
WORKDIR /app

# Define environment variable
ENV NAME=World

# Expose port
EXPOSE 5000

# Run command
CMD ["python3", "app.py"]
```

Let’s break down **every instruction**.

---

## **6. Every Instruction Explained**

| Instruction | Purpose | Example |
|-----------|--------|--------|

### **`FROM`**
Sets the **base image**. Must be first instruction (except `ARG`).

```Dockerfile
FROM ubuntu:20.04
FROM node:18-alpine
FROM scratch  # Empty base (for minimal images)
```

> ✅ Use minimal base images (Alpine, distroless) in production.

---

### **`RUN`**
Execute commands **during build time** (creates a new layer).

```Dockerfile
RUN apt-get update && apt-get install -y curl
RUN pip install -r requirements.txt
```

> ⚠️ Each `RUN` creates a layer — **chain commands** with `&&` to reduce layers.

---

### **`COPY`**
Copy files/directories **from host to image**.

```Dockerfile
COPY app.py /app/
COPY . /app/
```

> ✅ Prefer `COPY` over `ADD` unless you need auto-extraction.

---

### **`ADD`**
Like `COPY`, but can:
- Download from URLs
- Auto-extract `.tar`, `.gz` files

```Dockerfile
ADD https://example.com/app.tar.gz /app/
ADD local.tar.gz /app/  # Extracts automatically
```

> ⚠️ Avoid `ADD` for simple copying — use `COPY` for clarity.

---

### **`CMD`**
Default command to run **when container starts**.

```Dockerfile
CMD ["python3", "app.py"]          # Exec form (preferred)
CMD python3 app.py                 # Shell form
```

> 🔄 Can be **overridden** at runtime:
```bash
docker run myapp echo "hello"
```

---

### **`ENTRYPOINT`**
Sets the **primary executable** of the container. Makes image act like a binary.

```Dockerfile
ENTRYPOINT ["python3", "app.py"]
```

Now you can pass arguments:
```bash
docker run myapp --debug
# → runs: python3 app.py --debug
```

> ✅ Use `ENTRYPOINT` for tools (e.g., CLI apps).

> 🔁 `CMD` becomes default arguments to `ENTRYPOINT`.

---

### **`ENV`**
Set environment variables **available during build and runtime**.

```Dockerfile
ENV NODE_ENV=production
ENV PATH=/app/node_modules/.bin:$PATH
```

> ✅ Use for configuration (e.g., database URL, feature flags).

---

### **`EXPOSE`**
Documents which ports the container **listens on**.

```Dockerfile
EXPOSE 80/tcp
EXPOSE 443
```

> ❗ Does **not publish** the port — use `-p` in `docker run`:
```bash
docker run -p 8080:80 myapp
```

---

### **`VOLUME`**
Creates a **mount point** for persistent data.

```Dockerfile
VOLUME ["/data"]
```

> ✅ Use for databases (e.g., PostgreSQL data directory).

> 🔄 Can be overridden at runtime with `-v`.

---

### **`USER`**
Set the user for `RUN`, `CMD`, and `ENTRYPOINT`.

```Dockerfile
USER node
USER 1000:1000
```

> ✅ Best practice: Run as non-root user for **security**.

---

### **`ARG`**
Define **build-time variables**.

```Dockerfile
ARG VERSION=1.0
RUN echo $VERSION > version.txt
```

Pass at build time:
```bash
docker build --build-arg VERSION=2.0 -t myapp .
```

> 🔄 Not available at runtime — use `ENV` for that.

---

### **`LABEL`**
Add metadata to the image.

```Dockerfile
LABEL maintainer="dev@example.com"
LABEL version="1.2"
LABEL description="Web API server"
```

> ✅ Use for auditing, compliance, CI/CD tracking.

---

### **`HEALTHCHECK`**
Define how to check if the container is healthy.

```Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1
```

> ✅ Used by orchestrators (Kubernetes, Swarm) to restart unhealthy containers.

---

## **7. Best Practices: Minimizing Layers, Ordering Instructions**

### ✅ Minimize Layers
- Each `RUN`, `COPY`, `ADD` creates a layer.
- Too many layers = slower builds, larger images.

#### Combine Commands:
```Dockerfile
# ❌ Bad: 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good: 1 layer
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

---

### ✅ Order Instructions by Change Frequency

Put **rarely changed** instructions first, **frequently changed** last.

```Dockerfile
FROM python:3.9

# Install dependencies (rarely changes)
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt

# Copy code (changes often)
COPY . /app/

CMD ["python", "/app/app.py"]
```

> ✅ Benefit: Leverages **build cache** — only rebuilds from `COPY .` onward.

---

## **8. Using `.dockerignore` to Optimize Builds**

Like `.gitignore`, `.dockerignore` excludes files from the **build context**.

Create `.dockerignore`:
```
.git
node_modules
*.log
Dockerfile
README.md
.env
```

> ✅ Why?
- Reduces context size → faster builds
- Prevents sensitive files (like `.env`) from being copied into image

> ⚠️ Without `.dockerignore`, **everything** in the build directory is sent to Docker daemon.

---

## **9. Building and Managing Images**

### 🔹 `docker build`: Syntax, Flags, and Examples

#### Basic Syntax:
```bash
docker build [OPTIONS] PATH | URL | -
```

#### Common Flags:
| Flag | Purpose |
|------|--------|
| `-t` | Tag the image: `docker build -t myapp:v1 .` |
| `--build-arg` | Pass ARG values: `--build-arg VERSION=2.0` |
| `-f` | Use custom Dockerfile: `-f Dockerfile.prod` |
| `--no-cache` | Ignore cache and rebuild all layers |
| `--target` | Build to a specific stage (multi-stage) |

#### Examples:
```bash
# Build with tag
docker build -t mywebapp:1.0 .

# Use specific Dockerfile
docker build -f Dockerfile.debug -t myapp:debug .

# Build without cache
docker build --no-cache -t myapp .
```

---

## **10. Tagging Images for Different Environments**

Use tags to distinguish environments:

```bash
docker build -t myapp:dev .
docker build -t myapp:test .
docker build -t myapp:prod-v1.2 .
```

Or use **repositories**:
```bash
docker tag myapp:prod-v1.2 registry.com/myapp:1.2
docker push registry.com/myapp:1.2
```

> ✅ Best Practice: Use semantic versioning (`v1.2.0`) and avoid `latest`.

---

## **11. Viewing, Inspecting, and Removing Images**

| Command | Purpose |
|--------|--------|
| `docker images` | List all images |
| `docker image ls` | Same as above |
| `docker image inspect <image>` | View detailed metadata (JSON) |
| `docker image rm <image>` | Delete an image |
| `docker image prune` | Remove dangling images |
| `docker image prune -a` | Remove all unused images |

> 💡 Dangling images: `<none>` tag — intermediate layers not referenced.

---

## **12. Saving and Loading Images (Offline Transfer)**

Useful for air-gapped environments.

### Save Image to Tarball:
```bash
docker save myapp:v1 -o myapp-v1.tar
```

### Load on Another Machine:
```bash
docker load -i myapp-v1.tar
```

> ✅ Can compress:
```bash
docker save myapp:v1 | gzip > myapp-v1.tar.gz
gunzip -c myapp-v1.tar.gz | docker load
```

---

## **13. Image Optimization Techniques**

### Goal: **Smaller, faster, more secure** images.

---

## **14. Multi-Stage Builds (with Real Examples)**

Build dependencies in one stage, copy only artifacts to final image.

### Example: Python App
```Dockerfile
# Stage 1: Build
FROM python:3.9 AS builder
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: Runtime
FROM python:3.9-slim
COPY --from=builder /root/.local /root/.local
COPY app.py /app/
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "/app/app.py"]
```

> ✅ Final image doesn’t include `pip`, build tools, or source code.

---

### Example: Node.js App
```Dockerfile
# Build stage
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

> ✅ Only ships static files — no Node.js runtime.

---

## **15. Using Minimal Base Images (Alpine, Distroless)**

### ✅ Alpine Linux
- Very small (~5MB)
- Uses `musl` libc (not `glibc`) — may cause compatibility issues

```Dockerfile
FROM python:3.9-alpine
RUN apk add --no-cache curl
```

> ✅ Use `--no-cache` to avoid needing `clean` commands.

---

### ✅ Distroless (Google)
- No shell, no package manager — **only your app and runtime**
- Extremely secure

```Dockerfile
FROM gcr.io/distroless/python3-debian11
COPY app.py /
CMD ["/app.py"]
```

> ❌ Can’t `docker exec` into it — but that’s the point.

---

## **16. Cleaning Up in the Same Layer**

Avoid creating large intermediate layers.

### ❌ Bad:
```Dockerfile
RUN apt-get update
RUN apt-get install -y wget
RUN rm -rf /var/lib/apt/lists/*
```

→ Each `RUN` is a layer — the `rm` doesn’t shrink the previous layer.

### ✅ Good:
```Dockerfile
RUN apt-get update && \
    apt-get install -y wget && \
    rm -rf /var/lib/apt/lists/*
```

→ All in one layer — deleted files don’t persist.

---

## **17. Leveraging Build Cache Effectively**

Docker caches each layer. If a layer hasn’t changed, it reuses the cache.

### Cache Rules:
- `COPY`, `ADD`: Cache invalidated if file content changes
- `RUN`: Cache invalidated if command changes or previous layer changed

### Optimize for Cache:
1. Copy `package.json` before full source
2. Install deps before copying app
3. Use `.dockerignore`

```Dockerfile
COPY package.json .
RUN npm ci  # Cached if package.json unchanged
COPY . .
```

> ✅ First 3 steps cached until `package.json` changes.

---

## **18. BuildKit: Faster, Safer, More Powerful Builds**

BuildKit is Docker’s **next-generation builder**.

### ✅ Enable BuildKit
Set environment variable:
```bash
export DOCKER_BUILDKIT=1
docker build -t myapp .
```

Or enable in `daemon.json`:
```json
{
  "features": {
    "buildkit": true
  }
}
```

---

### ✅ BuildKit Benefits

| Feature | Description |
|--------|-------------|
| **Faster builds** | Parallel processing, smarter caching |
| **Secret mounting** | Securely pass secrets without leaking into image |
| **SSH mounting** | Access private Git repos during build |
| **Garbage collection** | Automatic cleanup |
| **Progress UI** | Modern, clean output |
| **Advanced syntax** | Use `# syntax=docker/dockerfile:1` for new features |

---

### Example: Mounting Secrets
```Dockerfile
# syntax=docker/dockerfile:1.4
FROM alpine
RUN --mount=type=secret,id=mytoken \
  echo "Token: $(cat /run/secrets/mytoken)"
```

Build with:
```bash
docker build --secret id=mytoken,src=token.txt .
```

> 🔐 Secret never enters image layers — **not visible in `docker history`**.

---

### Example: SSH Mount
```Dockerfile
# syntax=docker/dockerfile:1.4
FROM alpine
RUN --mount=type=ssh git clone git@github.com:me/private-repo.git
```

Run with:
```bash
docker build --ssh default .
```

> ✅ No need to bake SSH keys into image.

---

## ✅ Summary: Dockerfile Best Practices Checklist

| Practice | Why |
|--------|-----|
| Use minimal base images | Smaller, faster, more secure |
| Combine `RUN` commands | Reduce layers, avoid bloat |
| Order instructions wisely | Maximize cache hits |
| Use `.dockerignore` | Exclude unnecessary files |
| Multi-stage builds | Ship only what you need |
| Avoid `latest` tags | Ensure reproducibility |
| Use `COPY` over `ADD` | Clear intent |
| Run as non-root user | Security |
| Use `HEALTHCHECK` | Self-monitoring |
| Enable BuildKit | Faster, safer builds |
| Use `ARG` and `ENV` properly | Separate build vs runtime config |

---
