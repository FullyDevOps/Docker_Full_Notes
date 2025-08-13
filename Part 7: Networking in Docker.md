# **Part 7: Networking in Docker**

> 📌 Containers are isolated by default — **networking connects them**.  
> This guide teaches you how Docker handles networking, how containers talk to each other and the outside world, and how to design robust network topologies.

---

## 1. **Docker Networking Fundamentals**

Docker uses **network drivers** to manage how containers communicate:
- With each other
- With the host
- With the external world

Each container runs in a **network namespace**, providing isolation.

> 🔐 Isolation + Controlled Connectivity = Secure & Scalable Apps

---

### 🔹 The Big Picture: How Docker Networking Works

- Docker creates **virtual networks** using Linux bridges, iptables, and veth pairs.
- Containers on the same network can **communicate by name** (via embedded DNS).
- Traffic between containers is **encrypted only if you configure it** (e.g., with TLS).
- External access is controlled via **port publishing**.

---

## 2. **Default Bridge Network vs. User-Defined Networks**

### 🔹 **Default Bridge Network**

- Created automatically by Docker.
- All containers join this if no network is specified.
- **No DNS resolution** — containers can’t reach each other by name.
- Communication only via IP or `--link` (deprecated).

#### Example:
```bash
docker run -d --name db redis
docker run -d --name web nginx

# ❌ Cannot do:
docker exec web ping db  # Fails — no DNS
```

> ⚠️ **Not recommended for multi-container apps**.

---

### 🔹 **User-Defined Bridge Network**

- Created manually.
- Supports **automatic DNS resolution**.
- Better **isolation and security**.
- Containers can be **dynamically connected/disconnected**.

#### Create a Network:
```bash
docker network create app-net
```

#### Run Containers on It:
```bash
docker run -d --name db --network app-net redis
docker run -d --name web --network app-net nginx
```

#### Now This Works:
```bash
docker exec web ping db  # ✅ Resolves via DNS
```

> ✅ **Best practice**: Always use **user-defined networks** for multi-container apps.

---

### 🔹 Comparison: Default vs. User-Defined Bridge

| Feature | Default Bridge | User-Defined Bridge |
|--------|----------------|---------------------|
| DNS Resolution | ❌ No | ✅ Yes (by container name) |
| Isolation | Weak | Strong (per network) |
| Dynamic Connect/Disconnect | ❌ No | ✅ Yes |
| Custom IP/Subnet | ❌ No | ✅ Yes |
| Encryption | ❌ None | ❌ None (must be app-level) |
| Recommended for Production | ❌ No | ✅ Yes |

---

## 3. **Container Communication and DNS Resolution**

### 🔹 How DNS Works in User-Defined Networks

Docker runs an **embedded DNS server** (`127.0.0.11`) inside each container.

When you run:
```bash
docker run --name api --network app-net myapp
```

Other containers can reach it via:
```bash
curl http://api:3000
ping api
```

> ✅ DNS resolves `api` → container’s IP.

---

### 🔹 Service Discovery by Name

This is **automatic service discovery** — no need for external tools in simple setups.

#### Example: Microservices
```bash
docker network create backend

docker run -d --name auth --network backend auth-svc
docker run -d --name payments --network backend payments-svc
docker run -d --name api --network backend api-gateway
```

Inside `api-gateway`, you can call:
```bash
curl http://auth:8000/login
curl http://payments:9000/process
```

> ✅ No hardcoded IPs — **resilient to restarts and redeploys**.

---

### 🔹 Custom Aliases
Assign multiple names to a container:
```bash
docker run -d \
  --name db \
  --network app-net \
  --network-alias database \
  --network-alias mysql \
  mysql
```

Now `db`, `database`, and `mysql` all resolve to the same IP.

> ✅ Useful for backward compatibility or load balancing.

---

## 4. **Network Drivers Explained**

Docker supports multiple **network drivers** for different use cases.

---

### 🔹 `bridge` — Single-Host Communication

- Default driver for standalone containers.
- Uses a Linux bridge (`docker0`) on the host.
- Isolated from other networks.
- Requires `-p` to expose ports to host.

#### Use Case:
- Development
- Single-host microservices
- Local testing

#### Create:
```bash
docker network create -d bridge my-bridge
```

> ✅ Most common for non-clustered environments.

---

### 🔹 `host` — Direct Host Networking (No Isolation)

- **No network isolation** — container shares host’s network stack.
- No port mapping needed — container binds directly to host ports.
- Faster (no NAT overhead).

#### Example:
```bash
docker run --network host --name web nginx
# Now nginx listens on host:80 directly
```

#### Pros:
- High performance
- Low latency
- Simple port access

#### Cons:
- No isolation
- Port conflicts possible
- Security risk (container controls host network)

#### Use Case:
- Performance-critical apps
- Host-level monitoring tools
- CLI tools that need host network

> ⚠️ **Not recommended for multi-tenant or untrusted containers**.

---

### 🔹 `none` — No Network

- Container has **no network interface**.
- Only `lo` (loopback) interface exists.
- Completely isolated.

#### Example:
```bash
docker run --network none alpine ip a
# Output: only 'lo' interface
```

#### Use Case:
- Air-gapped processing
- Security-sensitive batch jobs
- Testing networkless behavior

> ✅ Safe, but limited.

---

### 🔹 `overlay` — Multi-Host Networking (Swarm/K8s)

- Enables communication **across multiple Docker hosts**.
- Built on top of VXLAN.
- Used in **Docker Swarm** and **Kubernetes**.
- Supports **encrypted traffic** (`--opt encrypted`).

#### Use Case:
- Multi-node clusters
- High availability
- Service replication

#### Example (Swarm Mode):
```bash
docker swarm init
docker network create -d overlay --attachable multi-host-net
```

Now containers on **different physical machines** can communicate securely.

> ✅ Foundation of **distributed systems**.

---

## 5. **Advanced Networking**

Now let’s dive into **fine-grained control** over Docker networking.

---

### 🔹 **Port Mapping and Publishing (`-p`, `-P`)**

Control how container ports are exposed to the host.

#### `-p host:container` — Publish Specific Port
```bash
docker run -d -p 8080:80 nginx
```
- Host port `8080` → Container port `80`
- Uses **NAT (iptables)**

#### Multiple Ports:
```bash
docker run -d -p 8080:80 -p 8443:443 nginx
```

#### UDP:
```bash
docker run -p 53:53/udp dns-server
```

#### Random Host Port:
```bash
docker run -p 80 nginx
# Docker assigns random port (e.g., 32768)
```

Check with:
```bash
docker port web
```

#### `-P` — Publish All Exposed Ports
```bash
docker run -P myapp
```
- Maps all `EXPOSE`d ports to random high-numbered host ports
- Useful for dev, not production

> ✅ Use `-p` for production, `-P` for testing.

---

### 🔹 **Custom Networks and Subnets**

Create networks with **custom IP ranges, gateways, and subnets**.

#### Example: Custom Subnet
```bash
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  custom-net
```

Now containers get IPs in `192.168.100.x` range.

#### Assign Static IP:
```bash
docker run -d \
  --name db \
  --network custom-net \
  --ip 192.168.100.10 \
  redis
```

> ✅ Useful for:
- Predictable IPs
- Integration with external systems
- Compliance requirements

---

### 🔹 **Connecting Containers Across Networks**

A container can be on **multiple networks** simultaneously.

#### Example:
```bash
docker network create frontend
docker network create backend

docker run -d --name web --network frontend nginx
docker run -d --name db --network backend redis

# Connect web to backend
docker network connect backend web
```

Now `web` can reach `db`:
```bash
docker exec web ping db
```

> ✅ Use for **tiered architectures** (web → app → db).

---

### 🔹 **Service Discovery by Name (Recap + Advanced)**

- Docker provides **automatic DNS-based service discovery**.
- Containers resolve each other by name.
- No need for external DNS or configuration.

#### Advanced: External DNS
Force container to use external DNS:
```bash
docker run \
  --dns=8.8.8.8 \
  --dns-search=example.com \
  alpine nslookup google.com
```

#### Disable DNS:
```bash
docker run --no-hosts alpine
```

---

## ✅ Summary: Docker Networking Decision Matrix

| Requirement | Recommended Network Type |
|-----------|--------------------------|
| Single-host microservices | ✅ User-defined `bridge` |
| High-performance, low-latency | ✅ `host` network |
| Air-gapped, secure job | ✅ `none` |
| Multi-host cluster (Swarm) | ✅ `overlay` |
| Custom IP addressing | ✅ `bridge` with `--subnet` |
| Service discovery by name | ✅ User-defined network |
| Expose service to outside | ✅ `-p 8080:80` |
| Avoid port conflicts | ✅ `-P` or dynamic port |
| Share network with host | ✅ `--network host` |
| Isolate database | ✅ Separate network (e.g., `db-net`) |

---

## 🚀 Pro Tips

1. **Always use user-defined networks** — they enable DNS and better isolation.
2. **Avoid the default bridge** — it’s legacy and limited.
3. **Use `host` only when necessary** — it breaks isolation.
4. **Label your networks** for organization:
   ```bash
   docker network create -l env=prod app-net
   ```
5. **Use `docker network inspect`** to debug connectivity:
   ```bash
   docker network inspect app-net
   ```
6. **Enable encryption** in overlay networks:
   ```bash
   docker network create -d overlay --opt encrypted my-overlay
   ```
7. **Never expose sensitive services** without firewall rules or TLS.

---
