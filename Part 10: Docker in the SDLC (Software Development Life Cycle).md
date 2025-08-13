# **Part 10: Docker in the SDLC (Software Development Life Cycle)**

> 📌 Docker isn’t just a tool — it’s a **platform for modern software delivery**.  
> From day-one development to production operations, Docker enables **consistency, speed, and reliability** across the entire SDLC.

---

## 1. **Development: Standardized Environments**

### 🔹 "Works on My Machine" Eliminated

One of the biggest pain points in software development:
> *"It works on my machine!"*

**Root causes**:
- Different OS versions
- Missing dependencies
- Conflicting libraries
- Divergent environment variables

### ✅ How Docker Solves It:
- Every developer runs the **same containerized environment**
- The app runs in **isolation** with **identical dependencies**
- No more "setup scripts" or "it worked yesterday"

> 🐳 **Result**: Everyone works in a **bit-for-bit identical environment**.

---

### 🔹 Onboarding New Developers in Minutes

Without Docker:
> ❌ "Install Node 18, Python 3.11, PostgreSQL 15, Redis, Kafka, set up config files…" → 1–2 days

With Docker:
> ✅ `git clone`, `docker compose up` → **app running in 5 minutes**

#### Example Onboarding Flow:
```bash
git clone https://github.com/company/myapp.git
cd myapp
docker compose up -d
```

Now the app, database, cache, and API are all running — **zero setup required**.

> 🚀 Faster ramp-up, fewer blockers, consistent dev experience.

---

### 🔹 Benefits in Development
| Benefit | Impact |
|--------|--------|
| **No setup scripts** | Eliminates "dependency hell" |
| **Live code reload** | Use bind mounts for instant feedback |
| **Isolated toolchains** | Run multiple projects with different Node/Python versions |
| **No host pollution** | No need to install tools globally |
| **Reproducible builds** | Same image built locally and in CI |

---

## 2. **Testing: Consistent Test Environments**

### 🔹 Unit, Integration, and E2E Testing in Containers

#### ✅ Unit Testing
Run tests inside a container with exact dependencies:
```yaml
# docker-compose.test.yml
services:
  test:
    build: .
    command: pytest /app/tests/unit
    volumes:
      - ./tests:/app/tests
```

#### ✅ Integration Testing
Spin up **full stack** (app + DB + cache) for integration tests:
```yaml
# docker-compose.integration.yml
services:
  app:
    build: .
    environment:
      - DB_HOST=db
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=testdb
```

Run:
```bash
docker compose -f docker-compose.integration.yml up --exit-code-from app
```

#### ✅ End-to-End (E2E) Testing
Use tools like **Cypress, Selenium, Playwright** in containers:
```yaml
services:
  e2e:
    image: cypress/included:12.0
    volumes:
      - ./cypress:/cypress
    depends_on:
      - web
```

> ✅ All tests run in **identical environments** — no flaky tests due to env drift.

---

### 🔹 Test-Specific Compose Files

Use **override files** for testing:

#### `docker-compose.yml` (Base)
```yaml
services:
  web:
    build: .
    environment:
      - ENV=development
```

#### `docker-compose.test.yml` (Override)
```yaml
services:
  web:
    environment:
      - ENV=test
      - DATABASE_URL=postgresql://testdb
    command: python manage.py test
```

Run tests:
```bash
docker compose -f docker-compose.yml -f docker-compose.test.yml up
```

> ✅ Clean separation between dev, test, and prod configs.

---

## 3. **CI/CD Pipelines with Docker**

Docker is the **foundation of modern CI/CD**.

### 🔹 Why Docker in CI/CD?
- **Consistent build environment** (no "agent drift")
- **Fast setup** (pull image vs. install tools)
- **Parallel testing** (isolated containers)
- **Artifact reuse** (build once, promote)
- **Easy rollback** (redeploy old image)

---

### 🔹 Example: GitHub Actions
```yaml
name: CI Pipeline

on: [push]

jobs:
  build-test:
    runs-on: ubuntu-latest
    container: python:3.11-slim

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install deps
        run: pip install -r requirements.txt

      - name: Run tests
        run: python -m pytest

      - name: Build Docker image
        run: |
          docker build -t myapp:$GITHUB_SHA .
          docker tag myapp:$GITHUB_SHA myapp:latest

      - name: Push to Registry
        run: |
          echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USER }} --password-stdin
          docker push myapp:$GITHUB_SHA
          docker push myapp:latest
```

---

### 🔹 Example: GitLab CI
```yaml
stages:
  - build
  - test
  - deploy

build:
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  script:
    - docker build -t registry.gitlab.com/user/myapp:$CI_COMMIT_SHA .
    - docker push registry.gitlab.com/user/myapp:$CI_COMMIT_SHA

test:
  image: registry.gitlab.com/user/myapp:$CI_COMMIT_SHA
  script:
    - python -m pytest

deploy-prod:
  stage: deploy
  script:
    - docker pull registry.gitlab.com/user/myapp:$CI_COMMIT_SHA
    - docker tag registry.gitlab.com/user/myapp:$CI_COMMIT_SHA myapp:stable
    - docker compose -f docker-compose.prod.yml up -d
  only:
    - main
```

---

### 🔹 Example: Jenkins
```groovy
pipeline {
  agent { docker { image 'node:18' } }

  stages {
    stage('Test') {
      steps {
        sh 'npm test'
      }
    }
    stage('Build & Push') {
      steps {
        script {
          docker.build("myapp:${env.BUILD_ID}")
          docker.withRegistry('https://registry.hub.docker.com', 'docker-creds') {
            docker.image("myapp:${env.BUILD_ID}").push()
          }
        }
      }
    }
  }
}
```

---

## 4. **Building, Testing, and Pushing Images**

### 🔹 Golden Workflow:
```
Code → Build Image → Test in Container → Push to Registry → Deploy
```

#### Key Principles:
- **Build once, promote everywhere**
- **Tag with Git SHA** (e.g., `myapp:abc123`) for traceability
- **Use immutable images** — never change a tag after push
- **Scan for vulnerabilities** before promotion

> ✅ This is **immutable infrastructure** in action.

---

## 5. **Artifact Management and Versioning**

### 🔹 Image Tagging Strategies

| Strategy | Example | Use Case |
|--------|--------|--------|
| **Git SHA** | `myapp:abc123` | Traceable, immutable |
| **Semantic Version** | `myapp:v1.2.0` | Production releases |
| **Branch + SHA** | `myapp:main-abc123` | CI builds |
| **Build Number** | `myapp:build-456` | Jenkins/GitLab CI |
| **Time-based** | `myapp:20240501-1400` | Auditing |

> ✅ **Never use `latest` in production** — it’s mutable and unreliable.

---

### 🔹 Image Promotion
Move images from dev → staging → prod **without rebuilding**:
```bash
# In staging:
docker pull myregistry.com/myapp:abc123
docker tag myapp:abc123 myapp:staging
docker compose -f docker-compose.staging.yml up

# In production:
docker tag myapp:abc123 myapp:production
docker compose -f docker-compose.prod.yml up
```

> ✅ Same image, different environments.

---

## 6. **Deployment Strategies**

### 🔹 Blue/Green Deployment
- Run two identical environments: **blue** (current), **green** (new)
- Test green
- Switch traffic (via load balancer)
- Decommission blue

```bash
# Deploy green
docker compose -f docker-compose.green.yml up -d

# After testing, switch DNS/load balancer
# Then: docker compose -f docker-compose.blue.yml down
```

> ✅ Zero downtime, fast rollback.

---

### 🔹 Canary Deployment
- Roll out new version to **small % of users**
- Monitor metrics
- Gradually increase traffic

```bash
# Run 1 canary instance + 10 stable
docker compose up --scale web=10
docker run -d --name web-canary myapp:canary
```

Use **service mesh** (e.g., Istio) or **load balancer** to route traffic.

> ✅ Safe rollout, real-user testing.

---

### 🔹 Rolling Updates
- Replace containers **one by one**
- Keep app available during update

```bash
docker compose up --no-deps --detach web
# Compose replaces old containers with new ones gradually
```

> ✅ Supported in Docker Swarm and Kubernetes.

---

### 🔹 Immutable Infrastructure
- **Never modify** a running container
- To update: **build new image**, **deploy new container**, **remove old**

> ✅ Benefits:
- Predictable deployments
- Easy rollback
- No configuration drift
- Auditable changes

---

### 🔹 GitOps with ArgoCD / Flux

**GitOps** = "Infrastructure as Code" for Kubernetes.

- Declare desired state in Git
- Tools like **ArgoCD** or **Flux** sync cluster to Git
- Every change is **versioned, reviewed, auditable**

#### Example ArgoCD App:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: https://github.com/company/myapp.git
    path: k8s/production
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

When you push to `main`, ArgoCD **automatically deploys** the new image.

> ✅ Full CI/CD pipeline with **Git as the source of truth**.

---

## 7. **Production Operations**

### 🔹 Monitoring and Logging

#### 📊 Monitoring (Prometheus + Grafana)
- Export metrics from apps (e.g., `/metrics`)
- Scrape with **Prometheus**
- Visualize in **Grafana**

```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

#### 📋 Logging (ELK Stack: Elasticsearch, Logstash, Kibana)
- Containers log to stdout
- Collect with **Filebeat** or **Fluentd**
- Index in **Elasticsearch**
- View in **Kibana**

> ✅ Centralized, searchable logs across services.

---

### 🔹 Security and Compliance

| Practice | Tool/Method |
|--------|------------|
| **Image scanning** | Trivy, Clair, Docker Scout |
| **Secrets management** | HashiCorp Vault, AWS Secrets Manager |
| **RBAC** | Kubernetes, Docker Swarm |
| **Network policies** | Calico, Cilium |
| **CIS Benchmarks** | Docker Bench for Security |
| **FIPS compliance** | Use hardened base images |

> ✅ Automate security checks in CI/CD.

---

### 🔹 Backup, Recovery, and Disaster Planning

| Component | Backup Strategy |
|---------|----------------|
| **Volumes** | `docker run --rm -v data:/data -v /backup:/backup alpine tar czf /backup/data.tar.gz -C /data .` |
| **Images** | Push to **private registry** (ECR, GCR, Harbor) |
| **Compose files** | Store in **Git** |
| **Database** | Use `pg_dump`, `mongodump`, or volume snapshots |
| **Disaster Recovery** | Replicate registry to another region; use Git for config recovery |

> ✅ Test recovery regularly.

---

## ✅ Summary: Docker in the SDLC — Phase-by-Phase

| Phase | Docker Benefit |
|------|----------------|
| **Development** | Standardized, zero-setup environments |
| **Testing** | Identical test environments, parallel runs |
| **CI/CD** | Build once, promote, immutable artifacts |
| **Deployment** | Blue/green, canary, rolling updates |
| **Production** | Monitoring, logging, scalability |
| **Security** | Image scanning, secrets, compliance |
| **Operations** | Backup, recovery, GitOps |

---

## 🚀 Pro Tips

1. **Always use `.dockerignore`** — speeds up builds and avoids leaks.
2. **Tag images with Git SHA** — enables full traceability.
3. **Use multi-stage builds** — smaller, secure production images.
4. **Never run as root** — use `USER` in Dockerfile.
5. **Scan images in CI** — catch vulnerabilities early.
6. **Use `--health-check`** — for self-healing systems.
7. **Automate rollback** — part of deployment strategy.
8. **Treat infrastructure as code** — store everything in Git.

---
