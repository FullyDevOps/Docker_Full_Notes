# **Part 9: Docker Compose — Multi-Container Applications**

> 📌 Running one container is easy. Running **many containers together** — databases, APIs, caches, frontends — requires orchestration.  
> **Docker Compose** is the **standard tool** for defining and managing multi-container apps on a single host.

---

## 1. **Introduction to Docker Compose**

### 🔹 What is Docker Compose?

**Docker Compose** is a tool for **defining and running multi-container Docker applications** using a **YAML file** (`docker-compose.yml`).

With a single command:
```bash
docker compose up
```
→ It starts **all services** (web, db, cache, etc.), connects them via networks, mounts volumes, and sets environment variables.

> ✅ Think of it as **"Docker for microservices" on a single machine**.

---

### 🔹 Why Use Compose?

| Without Compose | With Compose |
|----------------|--------------|
| Manual `docker run` commands for each service | One file defines the entire app |
| Hard to manage dependencies (e.g., "wait for DB") | Built-in dependency control |
| Repetitive, error-prone | Declarative, version-controlled |
| No easy way to share config | Share `docker-compose.yml` |
| Difficult to scale or restart | `docker compose up`, `down`, `scale` |

> ✅ Ideal for:
- Local development
- CI/CD pipelines
- Single-host staging environments
- Microservices demos

> 🚫 Not for large-scale production clusters → use **Kubernetes** or **Docker Swarm** instead.

---

## 2. **`docker-compose.yml` Structure**

The **Compose file** is a YAML configuration that defines your application’s services, networks, and volumes.

### 🔹 Basic Structure
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data

  redis:
    image: redis:7

volumes:
  db-data:

networks:
  default:
    name: app-network
```

> ✅ This defines a full stack: web app + PostgreSQL + Redis.

---

## 3. **Compose File Deep Dive**

Let’s break down each section.

---

### 🔹 `services`

Each **service** is a container template.

#### Key Service Options:

| Option | Purpose | Example |
|-------|--------|--------|
| `image` | Use pre-built image | `image: nginx:alpine` |
| `build` | Build from Dockerfile | `build: ./web` or `build: .` |
| `ports` | Publish ports | `- "8080:80"` |
| `environment` | Set env vars | `- DB_HOST=db` |
| `env_file` | Load from `.env` file | `env_file: .env` |
| `volumes` | Mount volumes/bind mounts | `- db-data:/var/lib/postgresql/data` |
| `networks` | Attach to custom networks | `- backend` |
| `depends_on` | Start order dependency | `- db` |
| `command` | Override default command | `command: python app.py` |
| `entrypoint` | Override entrypoint | `entrypoint: ["sh", "-c"]` |
| `restart` | Restart policy | `restart: unless-stopped` |
| `healthcheck` | Define health check | See below |

---

### 🔹 `volumes`

Define **named volumes** managed by Docker.

```yaml
volumes:
  db-data:
  cache-data:
```

> ✅ Use for persistent data (e.g., databases).  
> ❌ Don’t use for code (use bind mounts in dev).

---

### 🔹 `networks`

Define custom networks for isolation and DNS.

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access
```

Attach services:
```yaml
services:
  web:
    networks:
      - frontend
  db:
    networks:
      - backend
  api:
    networks:
      - frontend
      - backend
```

> ✅ Enables secure tiered architecture.

---

### 🔹 `secrets`

Securely pass sensitive data (passwords, keys) without exposing them in code.

#### Example:
```yaml
services:
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-pass
    secrets:
      - db-pass

secrets:
  db-pass:
    file: ./db-password.txt
    # Or: external: true (pre-created)
```

> ✅ Secrets are mounted as tmpfs — not visible in `docker inspect`.

---

## 4. **Environment Variables and Configuration**

### 🔹 Inline Environment
```yaml
environment:
  - NODE_ENV=production
  - API_URL=https://api.example.com
```

### 🔹 From `.env` File
```yaml
env_file:
  - .env
  - .env.secrets
```

`.env`:
```
DB_USER=admin
DB_PASS=supersecret
```

> ✅ Use `.env` for local config — never commit secrets!

---

## 5. **Dependency Management (`depends_on`)**

Control **startup order** — but not readiness.

```yaml
depends_on:
  - db
  - redis
```

> ⚠️ `depends_on` only waits for containers to **start**, not for the app inside to be ready.

### 🔹 Fix: Wait for Service Readiness

Use a **wait script** or sidecar.

#### Option 1: Custom Script
```yaml
web:
  depends_on:
    - db
  command: >
    sh -c "
    until pg_isready -h db -p 5432; do
      echo 'Waiting for PostgreSQL...';
      sleep 2;
    done;
    python app.py
    "
```

#### Option 2: Use `wait-for-it` (External Tool)
```yaml
web:
  depends_on:
    - db
  command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]
```

> ✅ Always **wait for readiness**, not just startup.

---

## 6. **Profiles for Different Environments**

Use **profiles** to enable/disable services per environment.

```yaml
services:
  web:
    image: myapp
    profiles:
      - dev
      - prod

  debugger:
    image: busybox
    command: tail -f /dev/null
    profiles:
      - debug

  logger:
    image: fluentd
    profiles:
      - logging
```

#### Control with `--profile`:
```bash
docker compose --profile debug up
# Starts: web + debugger

docker compose --profile logging up
# Starts: web + logger
```

> ✅ Perfect for:
- Dev-only tools
- Monitoring in prod
- Feature flags

---

## 7. **Compose Commands and Workflows**

Master the CLI for daily use.

| Command | Purpose |
|--------|--------|
| `docker compose up` | Start all services (build if needed) |
| `docker compose up -d` | Start in detached mode |
| `docker compose down` | Stop and remove containers, networks |
| `docker compose down -v` | Also remove volumes |
| `docker compose build` | Rebuild service images |
| `docker compose logs` | View logs (`-f` to follow) |
| `docker compose exec` | Run command in running container |
| `docker compose ps` | List containers |
| `docker compose config` | Validate and view final config |
| `docker compose pull` | Pull all images |
| `docker compose stop` | Stop without removing |
| `docker compose start` | Restart stopped containers |

---

### 🔹 Running One-Off Commands

Run temporary tasks (e.g., database migrations):

```bash
docker compose run web python manage.py migrate
```

> ✅ Creates a **new container** from the `web` service, runs the command, then removes it.

Use `--rm` (default in v2+) to auto-remove.

---

### 🔹 Scaling Services

Run **multiple instances** of a service:

```bash
docker compose up --scale web=3
```

> ✅ Useful for load testing or simulating clusters.

> ⚠️ All instances share the same port → use a **load balancer** (e.g., nginx) in front.

---

## 8. **Real-World Compose Examples**

---

### 🔹 Example 1: Web App + Database + Cache

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
      - CACHE_HOST=redis
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_PASSWORD=devpass
    volumes:
      - db-data:/var/lib/postgresql/data

  redis:
    image: redis:7

volumes:
  db-data:
```

Run:
```bash
docker compose up -d
```

---

### 🔹 Example 2: Microservices Architecture

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./gateway
    ports:
      - "8000:8000"
    depends_on:
      - auth
      - payments

  auth:
    build: ./auth
    environment:
      - DB_URL=mongodb://auth-db:27017/auth

  payments:
    build: ./payments
    environment:
      - STRIPE_KEY=${STRIPE_KEY}

  auth-db:
    image: mongo:6
    volumes:
      - auth-data:/data/db

  logger:
    image: fluentd
    profiles:
      - logging

volumes:
  auth-data:
```

> ✅ Each service is isolated, scalable, and independently deployable.

---

### 🔹 Example 3: Development vs. Production Compose Files

#### `docker-compose.yml` (Base)
```yaml
services:
  web:
    build: .
    environment:
      - ENV=${ENV:-development}
```

#### `docker-compose.dev.yml`
```yaml
services:
  web:
    volumes:
      - .:/app
    ports:
      - "5000:5000"
    environment:
      - DEBUG=true
```

#### `docker-compose.prod.yml`
```yaml
services:
  web:
    image: myregistry.com/myapp:${TAG:-latest}
    restart: unless-stopped
    ports:
      - "80:5000"
    environment:
      - DEBUG=false
```

#### Use with `--file`:
```bash
# Dev
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Prod
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

> ✅ **Merge files** — base + override = environment-specific config.

---

## ✅ Summary: Docker Compose Best Practices

| Practice | Why |
|--------|-----|
| Use `docker-compose.yml` for base config | Reusable across environments |
| Split dev/prod with override files | Avoid duplication |
| Use `.env` for environment variables | Keep config out of code |
| Use `depends_on` + wait script | Ensure services are ready |
| Name your networks and volumes | Better organization |
| Use `profiles` for optional services | Dev tools, logging |
| Run migrations with `docker compose run` | One-off tasks |
| Use `restart: unless-stopped` in prod | Self-healing |
| Never store secrets in files | Use `secrets` or external vault |
| Validate with `docker compose config` | Catch errors early |

---

## 🚀 Pro Tips

1. **Always run `docker compose config`** before `up` — validates YAML.
2. **Use `--progress=plain`** to see full build output.
3. **Set `COMPOSE_PROJECT_NAME`** to control container prefixes.
4. **Use `docker compose logs -f service`** to monitor specific app.
5. **Scale only stateless services** — databases can’t be scaled this way.
6. **Use `down -v` carefully** — deletes named volumes!
7. **Integrate with CI/CD**: `docker compose` is great for test environments.

---
