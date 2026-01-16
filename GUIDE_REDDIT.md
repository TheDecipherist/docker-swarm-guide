# The Complete Docker Swarm Production Guide for 2026: Everything I Learned Running It for Years

📸 **[View with banner on the website](https://thedecipherist.github.io/docker-swarm-guide/?utm_source=reddit&utm_medium=post&utm_campaign=docker-swarm-guide&utm_content=v1-guide)**

## V1: Battle-Tested Production Knowledge

**TL;DR:** I've been running Docker Swarm in production on AWS for years and I'm sharing everything I've learned - from basic concepts to advanced production configurations. This isn't theory - it's battle-tested knowledge that kept our services running through countless deployments.

**What's in V1:**
- Complete Swarm hierarchy explained
- VPS requirements and cost planning across providers
- DNS configuration (the #1 cause of Swarm issues)
- Production-ready compose files and multi-stage Dockerfiles
- Prometheus + Grafana monitoring stack
- Platform comparison (Portainer, Dokploy, Coolify, CapRover, Dockge)
- CI/CD versioning and deployment workflows
- [GitHub repo](https://github.com/TheDecipherist/docker-swarm-guide) with all configs

---

## Why Docker Swarm in 2026?

Before the Kubernetes crowd jumps in - yes, I know K8s exists. But here's the thing: **Docker Swarm is still incredibly relevant in 2026**, especially for small-to-medium teams who want container orchestration without the complexity overhead.

Swarm advantages:
- Native Docker integration (no YAML hell beyond compose files)
- Significantly lower learning curve
- Perfect for 2-20 node clusters
- Built-in service discovery and load balancing
- Rolling updates out of the box
- Works with your existing Docker Compose files (mostly)

If you're not running thousands of microservices across multiple data centers, Swarm might be exactly what you need.

---

## Understanding the Docker Swarm Hierarchy

```
Swarm → Nodes → Stacks → Services → Tasks (Containers)
```

- **Swarm**: Your entire cluster. Only works with **pre-built images** - no `docker build` in production.
- **Nodes**: Managers (handle state/scheduling) and Workers (run containers). Use 3 or 5 managers for HA.
- **Stacks**: Groups of related services from a compose file.
- **Services**: Manage replicas, rolling updates, health monitoring, auto-restart.
- **Tasks**: A Task = Container. 6 replicas = 6 tasks.

---

## VPS Requirements & Cost Planning

Docker Swarm is lightweight - minimal overhead compared to Kubernetes.

### Infrastructure Presets

| Preset | Nodes | Layout | Min Specs (per node) | Use Case |
|--------|-------|--------|---------------------|----------|
| **Minimal** | 1 | 1 manager | 1 vCPU, 1GB RAM, 25GB | Dev/testing only |
| **Basic** | 2 | 1 manager + 1 worker | 1 vCPU, 2GB RAM, 50GB | Small production |
| **Standard** | 3 | 1 manager + 2 workers | 2 vCPU, 4GB RAM, 80GB | Standard production |
| **HA** | 5 | 3 managers + 2 workers | 2 vCPU, 4GB RAM, 80GB | High availability |

### Approximate Monthly Costs (2025/2026)

| Provider | Basic (2 nodes) | Standard (3 nodes) | HA (5 nodes) |
|----------|-----------------|--------------------|--------------|
| **Hetzner** | ~€8-12 | ~€20-30 | ~€40-60 |
| **Vultr** | ~$12-20 | ~$30-50 | ~$60-100 |
| **DigitalOcean** | ~$16-24 | ~$40-60 | ~$80-120 |
| **Linode** | ~$14-22 | ~$35-55 | ~$70-110 |

**Why these numbers?**
- **1GB RAM minimum**: Swarm itself uses ~100-200MB, but you need headroom for containers
- **3 or 5 managers for HA**: Raft consensus requires odd numbers for quorum
- **2 vCPU for production**: Single core gets bottlenecked during deployments

### My Recommendation

For most small-to-medium teams:
1. **Start with Basic (2 nodes)** - 1 manager + 1 worker on Vultr or Hetzner
2. **Budget ~$20-40/month** for a production-ready setup
3. **Add nodes as needed** - Swarm makes scaling easy

If you need HA from day one, the **Standard (3 nodes)** preset gives you redundancy without breaking the bank.

### What About AWS/GCP/Azure?

Cloud giants work fine with Swarm, but:
- **More expensive** for equivalent specs
- **More complexity** (VPCs, security groups, IAM)
- **Better if** you need other AWS services (RDS, S3, etc.)

We run Swarm on AWS EC2 because we're already deep in the AWS ecosystem. If you're starting fresh, a dedicated VPS provider is simpler and cheaper.

---

## Setting Up Your Production Environment

### Install Docker (Ubuntu)

```bash
# Add Docker's official GPG key and repo
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

**Important:** Use `docker compose` (space), not `docker-compose` (deprecated).

### Initialize the Swarm

```bash
# Get your internal IP
ip addr

# Initialize on manager (use YOUR internal IP)
docker swarm init --advertise-addr 10.10.1.141:2377 --listen-addr 10.10.1.141:2377

# Join token for workers (save this!)
docker swarm join --token SWMTKN-1-xxxxx... 10.10.1.141:2377
```

**Critical:** Use a fixed IP for advertise address. Dynamic IPs will break your cluster on restart.

---

## DNS Configuration (This Will Save You Hours)

**CRITICAL**: DNS issues cause 90% of Swarm networking problems.

Edit `/etc/systemd/resolved.conf` on each node:

```ini
[Resolve]
DNS=10.10.1.122 8.8.8.8
Domains=~yourdomain.io
```

Then reboot. Docker runs its own DNS at `127.0.0.11` for container-to-container resolution.

**Rule:** Never hardcode IPs in Swarm. Use service names - Docker handles routing.

---

## Network Configuration

Create an overlay network (mandatory for multi-node):

```bash
docker network create \
  --opt encrypted \
  --subnet 172.240.0.0/24 \
  --gateway 172.240.0.254 \
  --attachable \
  --driver overlay \
  awsnet
```

| Flag | Purpose |
|------|---------|
| `--opt encrypted` | IPsec encryption. Optional but recommended. **Note:** Can cause issues with NAT - use internal VPC IPs |
| `--subnet` | Prevents conflicts with VPC ranges |
| `--attachable` | Allows standalone containers to connect |

### Required Ports

- **TCP 2377**: Cluster management
- **TCP/UDP 7946**: Node communication
- **TCP/UDP 4789**: Overlay network traffic

---

## Production Compose File

```yaml
version: "3.8"

services:
  nodeserver:
    dns:
      - 10.10.1.122
    init: true  # Proper signal handling, zombie cleanup

    environment:
      - NODE_ENV=production
      - API_KEY=${API_KEY}

    deploy:
      mode: replicated
      replicas: 6
      placement:
        max_replicas_per_node: 3
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '0.50'
          memory: 400M
        reservations:
          cpus: '0.20'
          memory: 150M

    image: "yourregistry/nodeserver:latest"
    ports:
      - "61339"
    networks:
      awsnet:
    secrets:
      - app_secrets

secrets:
  app_secrets:
    external: true

networks:
  awsnet:
    external: true
```

**Key settings:**
- `init: true` - Runs tini as PID 1 for proper signal handling
- `failure_action: rollback` - Auto-rollback on failed deployments
- `order: start-first` - New containers start before old ones stop (zero downtime)
- **Always set resource limits** - A runaway container can kill your node

---

## Dockerfile Best Practices

### Multi-Stage Build (Node.js)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-bookworm-slim AS base
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends python3 make g++ && rm -rf /var/lib/apt/lists/*
COPY package.json package-lock.json ./

FROM base AS compiled
RUN npm ci --omit=dev

FROM node:20-bookworm-slim AS final
RUN ln -snf /usr/share/zoneinfo/America/New_York /etc/localtime
WORKDIR /app
COPY --from=compiled /app/node_modules /app/node_modules
COPY . .
EXPOSE 3000
ENTRYPOINT ["node", "./server.js"]
```

**Why multi-stage?** Build tools stay in temp stage. Final image is clean and small.

### Key Rules

1. **Run in foreground** - `CMD ["nginx", "-g", "daemon off;"]` (official nginx image handles this)
2. **Pin base images** - `FROM ubuntu:22.04` not `FROM ubuntu:latest`
3. **Include health checks** - Swarm uses these for rolling updates
4. **Use .dockerignore** - Exclude `.env`, `node_modules`, `.git`

### Sample .dockerignore

```
.git
.gitignore
.env
.env.*
node_modules
npm-debug.log
Dockerfile*
docker-compose*
.dockerignore
*.md
.vscode
.idea
```

This keeps your build context small and prevents secrets from accidentally ending up in images.

---

## Monitoring Stack (Prometheus + Grafana)

Full compose file in the [GitHub repo](https://github.com/TheDecipherist/docker-swarm-guide). Key points:

| Service | Purpose | Mode |
|---------|---------|------|
| Grafana | Dashboards | 1 replica on manager |
| Prometheus | Metrics collection | 1 replica on manager |
| cAdvisor | Container metrics | Global (all nodes) |
| Node Exporter | Host metrics | Global (all nodes) |

Use `mode: global` for monitoring agents - runs ONE instance on EVERY node.

**Quick setup tip:** Start with cAdvisor + Node Exporter first. Add Prometheus when you need historical data. Add Grafana when you need pretty dashboards for your team.

---

## Docker Management Platforms

Managing Swarm via CLI is powerful, but GUIs improve visibility significantly.

### Portainer

**Best for:** Teams wanting visual management without changing workflows.

```bash
# Deploy Portainer agent on each node
docker service create --name portainer_agent \
  --publish mode=host,target=9001,published=9001 \
  --mode global \
  --mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=bind,src=//var/lib/docker/volumes,dst=/var/lib/docker/volumes \
  portainer/agent:latest

# Deploy Portainer server on manager
docker service create --name portainer \
  --publish 9443:9443 --publish 8000:8000 \
  --replicas=1 --constraint 'node.role == manager' \
  --mount type=volume,src=portainer_data,dst=/data \
  portainer/portainer-ce:latest
```

**Pricing:** CE is completely free with no node limits. Business Edition adds enterprise features.

**Why Portainer?** It shows you container logs, resource usage, network topology, and lets you manage stacks visually. Perfect for teams where not everyone is a CLI wizard.

### Platform Comparison

| Platform | Swarm Support | Git Deploy | Auto SSL | Best For |
|----------|---------------|------------|----------|----------|
| **Portainer** | Full | No | No | Visual management |
| **Dokploy** | Full | Yes | Yes | Heroku-style on Swarm |
| **Coolify** | Experimental | Yes | Yes | 280+ templates, great UI |
| **CapRover** | Full (native) | Yes | Yes | Proven Swarm PaaS |
| **Dockge** | None | No | No | Simple Compose management |

**My setup:** Portainer for visibility + custom CI/CD + Prometheus/Grafana for monitoring.

**Note on Coolify:** Their Swarm support is experimental. Works for basic setups but I've hit edge cases. Great project though - watch this space.

---

## Secret Management

**Stop using environment variables for secrets.**

```yaml
secrets:
  app_secrets:
    external: true  # Created via CLI or Portainer

services:
  app:
    secrets:
      - app_secrets
```

### Creating Secrets (Do It Right)

```bash
# Method 1: From file (delete the file after!)
docker secret create app_secrets ./secrets.json && rm ./secrets.json

# Method 2: From stdin (more secure - no file on disk)
echo "my_password" | docker secret create db_password -

# Method 3: Interactive (doesn't save to shell history)
cat /dev/stdin | docker secret create api_key -
# Paste your secret, then Ctrl+D

# List secrets
docker secret ls

# Delete (must not be in use by any service)
docker secret rm old_secret
```

**Secrets are immutable.** To update, create a new versioned secret (`db_password_v2`), update your compose, redeploy, then delete the old one.

Secrets appear as files in `/run/secrets/SECRET_NAME`. They're encrypted at rest, not visible in `docker inspect`, and only sent to nodes that need them.

### Why Environment Variables Aren't "Safe"

**Common misconception:** "It's not hardcoded, it's an env var, so it's safe."

**Reality:** Any process running inside the container can read environment variables. A compromised dependency, a debug endpoint, or a memory dump can expose them.

**Better approach (Node.js example):**

```javascript
// Instead of: const apiKey = process.env.API_KEY
// Do this:
const fs = require('fs');

function getSecret(secretName) {
  const secretPath = process.env[`${secretName}_FILE`] || `/run/secrets/${secretName}`;
  return fs.readFileSync(secretPath, 'utf8').trim();
}

const apiKey = getSecret('API_KEY');
```

Your env var points to a **filename**, not the secret itself. The function reads the actual secret from the Docker secret file at runtime. This adds a layer of abstraction - even if env vars leak, attackers only get file paths, not credentials.

---

## CI/CD Versioning

```bash
BUILD_VERSION=$(cat ./buildVersion.txt)
LONG_COMMIT=$(git rev-parse HEAD)

docker compose build --build-arg GIT_COMMIT=$LONG_COMMIT --build-arg BUILD_VERSION=$BUILD_VERSION
docker compose push
docker stack deploy -c docker-compose.yml mystack
```

**Never use `latest` in production.** Use commit hashes or semantic versions.

**Why versioning matters:**
- Rollback becomes a one-liner: `docker service update --image yourapp:v1.2.3 mystack_app`
- You know exactly what's running on each node
- Audit trails for compliance
- No more "but it worked on my machine" mysteries

---

## Useful Commands

```bash
# Node management
docker node ls                                              # List all nodes
docker node update --availability=drain docker2.domain.io   # Maintenance mode
docker node update --availability=active docker2.domain.io  # Back to active
docker node inspect docker2.domain.io --pretty              # Node details

# Stack operations
docker stack deploy -c docker-compose.yml mystack   # Deploy/update stack
docker stack services mystack                       # List services in stack
docker stack ps mystack                             # List tasks (containers)
docker stack rm mystack                             # Remove stack

# Service operations
docker service scale mystack_web=4                  # Scale to 4 replicas
docker service logs -f mystack_web                  # Follow logs
docker service logs --tail 100 mystack_web          # Last 100 lines
docker service update --force mystack_web           # Force redeploy
docker service update --image yourapp:v2 mystack_web  # Update image

# Debugging
docker service ps mystack_web --no-trunc            # Full error messages
docker inspect $(docker ps -q -f name=mystack_web)  # Container details
```

**Pro tip:** `docker stack deploy` is idempotent. Run it again to update - no need to `rm` first.

---

## Common Gotchas

These issues have cost me hours. Learn from my pain.

**Containers can't communicate between nodes:**
1. Verify overlay network exists: `docker network ls`
2. Check it's attached to your service in compose file
3. Verify DNS config in `/etc/systemd/resolved.conf` on each node
4. Ensure ports 7946 (TCP/UDP) and 4789 (UDP) are open between nodes
5. If using `--opt encrypted`, try without it first (NAT issues)

**Service stuck in "Pending":**
```bash
docker service ps myservice --no-trunc
```
Common causes:
- Resource constraints - scheduler can't find a node with enough CPU/memory
- Image doesn't exist or can't be pulled (check registry auth)
- Placement constraints can't be satisfied
- All nodes are drained or paused

**Rolling update hangs:**
Health checks are usually the culprit. Your container might be healthy but Swarm doesn't know it.

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 60s  # Give your app time to start!
```

**"No such network" errors:**
Create networks BEFORE deploying stacks:
```bash
docker network create --driver overlay --attachable mynetwork
docker stack deploy -c compose.yml mystack
```

**Secrets not updating:**
Secrets are immutable. To update:
1. Create new secret with different name: `docker secret create app_secrets_v2 ./secrets.json`
2. Update compose to reference new secret name
3. Redeploy stack

---

## Final Tips

1. **Use Portainer** - Free and makes Swarm management much easier. Deploy it first.
2. **Always use external networks** - Create overlay networks before deploying stacks
3. **Tag images properly** - Never `latest` in production. Use commit hashes or semver.
4. **Set resource limits** - Always. A runaway container will take down your node.
5. **Test your rollback** - Deploy a broken image intentionally to verify auto-rollback works
6. **Monitor from day one** - Prometheus + Grafana is free and catches issues early
7. **Document your setup** - Future you will thank present you
8. **Start small** - 2 nodes is enough to learn. Scale when you need it.

---

## Backup Your Swarm State

Swarm state lives on manager nodes. Back it up:

```bash
# Stop Docker (on manager)
sudo systemctl stop docker

# Backup the Swarm state
sudo tar -cvzf swarm-backup-$(date +%Y%m%d).tar.gz /var/lib/docker/swarm

# Start Docker
sudo systemctl start docker
```

Store backups off-node. If all managers die simultaneously (rare but possible), this is your recovery path.

---

## When NOT to Use Swarm

To be fair, Swarm isn't always the answer:

- **Need advanced scheduling?** K8s has more sophisticated options
- **Running 50+ services?** K8s ecosystem is more mature at scale
- **Need service mesh?** Istio/Linkerd integrate better with K8s
- **Team already knows K8s?** Stick with what you know

For everything else - small teams, 2-20 nodes, wanting to move fast - Swarm is hard to beat.

---

## GitHub Repo

All compose files, Dockerfiles, and configs mentioned in this guide:

**[github.com/TheDecipherist/docker-swarm-guide](https://github.com/TheDecipherist/docker-swarm-guide)**

The repo includes:
- Complete monitoring stack compose file
- Production-ready multi-stage Dockerfiles
- Network configuration examples
- Portainer deployment scripts

---

## What's Coming in V2

Based on community feedback, V2 will cover:
- Deep dive into monitoring (Prometheus, Grafana, DataDog comparison)
- Blue-green deployments in Swarm
- Logging strategies (ELK, Loki, etc.)
- Traefik integration for automatic SSL

---

*What's your Swarm setup? Running it in production? Home lab? What providers are you using? Drop your configs and war stories below — I'll incorporate the best tips into V2.*

*Questions? I'll be in the comments.*
