# **Part 13: Real-World Examples and Workflows**

Real-world use cases showing how Docker solves common application challenges.

---

### **Example 1: Python Flask + PostgreSQL**
- **Use Case**: Web API with persistent data
- **Tools**: `Dockerfile`, `docker-compose.yml`
- **Key Features**:
  - Flask app connects to PostgreSQL via `db` hostname
  - Volume for persistent database storage
  - Environment variables for config
- ✅ **Why It Matters**: Shows how to run stateful apps securely and reproducibly.

---

### **Example 2: Node.js + MongoDB + Nginx**
- **Use Case**: Full-stack JavaScript app with reverse proxy
- **Structure**:
  - `node-app`: Backend API
  - `mongodb`: Persistent data store
  - `nginx`: Serves frontend, proxies `/api` to Node
- **Benefits**:
  - Clean separation of concerns
  - Nginx handles SSL, compression, caching
- ✅ **Why It Matters**: Demonstrates multi-service routing and production-like setup.

---

### **Example 3: Java Spring Boot + Redis**
- **Use Case**: High-performance backend with caching
- **Setup**:
  - Spring Boot app (fat JAR) in minimal `openjdk` or `distroless` image
  - Redis for session/cache
  - Health checks and restart policies
- ✅ **Why It Matters**: Teaches efficient Java containerization and performance tuning.

---

### **Example 4: Legacy App Containerization**
- **Use Case**: Modernize old monoliths (e.g., Java WAR, .NET Framework)
- **Approach**:
  - Wrap app in minimal OS image
  - Expose ports, inject configs via env
  - Use `--restart=unless-stopped`
- ⚠️ **Challenge**: No microservices benefits unless refactored
- ✅ **Why It Matters**: Fast way to gain deployment consistency without full rewrite.

---

### **Example 5: CI/CD Pipeline with GitHub Actions**
- **Workflow**:
  1. On push → build image
  2. Run tests in container
  3. Scan for vulnerabilities (Trivy)
  4. Push to registry (GHCR or ECR)
  5. Deploy to staging (or trigger ArgoCD)
- ✅ **Why It Matters**: Fully automated, secure, auditable delivery pipeline.

---

# **Part 14: Troubleshooting and Debugging**

A structured approach to diagnosing and fixing container issues.

---

### **Systematic Troubleshooting Framework**
Follow this flow:
1. **Check container state**: `docker ps -a`
2. **Inspect logs**: `docker logs <container>`
3. **Inspect config**: `docker inspect <container>`
4. **Exec inside**: `docker exec -it <container> sh`
5. **Validate network**: `docker network inspect`, `ping`, `curl`
6. **Check resources**: `docker stats`, `df`, `free -h`

> 🔍 **Golden Rule**: Start from the container and work outward.

---

### **Common Issues and Solutions (50+ Scenarios) – Highlights**
| Issue | Fix |
|------|-----|
| Container exits immediately | Check `CMD` — is it long-running? |
| Port already in use | Change host port or stop conflicting process |
| Can’t connect to DB | Verify network, hostname, credentials |
| Permission denied on volume | Fix host file ownership or use `:Z`/`:z` labels |
| Image not found | Check tag, login to registry, verify network |
| OOMKilled (exit 137) | Increase memory limit (`-m 512m`) |
| Health check failing | Add startup delay, fix endpoint |
| Build fails due to cache | Use `--no-cache` or clean build cache |

✅ **Key Insight**: Most issues are due to **misconfigurations**, not Docker bugs.

---

### **Log Analysis and Performance Tuning**
- Use `docker logs --tail 100 -f` to stream logs
- Look for:
  - Errors, timeouts, retries
  - Memory/CPU spikes (`docker stats`)
- Optimize:
  - Use smaller base images
  - Limit resources
  - Enable health checks

---

### **Recovery Strategies**
| Scenario | Recovery |
|--------|---------|
| Corrupted volume | Restore from backup |
| Failed deployment | Roll back to known-good image |
| Host failure | Re-deploy on another node (Swarm/K8s) |
| Registry down | Use local cache or air-gapped mirror |

✅ **Best Practice**: Automate recovery with orchestration and monitoring.

---

# **Part 15: The Future of Docker**

Where container technology is headed beyond traditional Docker.

---

### **eBPF and Cilium: Next-Gen Networking**
- **eBPF**: Linux kernel tech for safe, efficient programs
- **Cilium**: Uses eBPF for:
  - Faster networking
  - Advanced observability
  - Zero-trust security (L7 policies)
- ✅ **Impact**: Replaces iptables, enables deep visibility and security.

---

### **Confidential Computing and Secure Enclaves**
- Run containers in **encrypted memory** (Intel SGX, AMD SEV)
- Protect data even from cloud provider
- Use cases: finance, healthcare, government
- ✅ **Future**: Fully encrypted containers from disk to RAM.

---

### **WebAssembly (Wasm) and Containers**
- **Wasm**: Lightweight, portable bytecode
- Run Wasm modules **in containers** via `wasmtime`, `containerd-wasm`
- Benefits:
  - Faster startup
  - Safer (sandboxed by design)
  - Run alongside traditional containers
- ✅ **Use Case**: Edge functions, plugins, serverless.

---

### **AI/ML Workloads in Containers**
- Train and serve models in containers
- Tools: TensorFlow, PyTorch, NVIDIA Docker (GPU support)
- Orchestrate with Kubernetes (Kubeflow)
- ✅ **Why It Matters**: Standardizes AI pipelines — same as devops.

---

## ✅ Summary: Key Takeaways

| Part | Core Message |
|------|-------------|
| **13** | Docker enables real apps — from web APIs to legacy systems |
| **14** | Debug systematically: logs → inspect → exec → network → resources |
| **15** | The future is **faster, safer, and more efficient** containers (eBPF, Wasm, confidential computing) |

---
