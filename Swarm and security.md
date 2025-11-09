## Docker Swarm

Docker Swarm is Docker's native container orchestration tool for clustering and scheduling Docker containers across multiple hosts.

### What is Docker Swarm?

Docker Swarm turns multiple Docker hosts into a single virtual Docker host, providing:
- **Cluster Management:** Multiple Docker hosts form a cluster
- **Orchestration:** Automated deployment and scaling
- **High Availability:** Automatic failover and recovery
- **Load Balancing:** Built-in load balancing across containers
- **Service Discovery:** Containers can find each other automatically
- **Rolling Updates:** Deploy updates with zero downtime

### Swarm vs Kubernetes

| Feature | Docker Swarm | Kubernetes |
|---------|--------------|------------|
| **Setup** | Simple, built into Docker | Complex, separate installation |
| **Learning Curve** | Easy | Steep |
| **Scalability** | Good for small-medium clusters | Excellent for large clusters |
| **Community** | Smaller | Larger ecosystem |
| **Features** | Essential features | Rich feature set |
| **Best For** | Simpler deployments | Complex, enterprise needs |

### Swarm Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Swarm Cluster                      │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐│
│  │ Manager Node │  │ Manager Node │  │ Worker Node││
│  │   (Leader)   │  │  (Follower)  │  │            ││
│  │              │  │              │  │            ││
│  │ - Scheduling │  │ - Backup     │  │ - Run      ││
│  │ - Raft Store │  │ - Raft Store │  │   Tasks    ││
│  │ - API        │  │              │  │            ││
│  └──────────────┘  └──────────────┘  └────────────┘│
│                                                      │
│  ┌──────────────┐  ┌──────────────┐                │
│  │ Worker Node  │  │ Worker Node  │                │
│  │              │  │              │                │
│  │ - Run Tasks  │  │ - Run Tasks  │                │
│  └──────────────┘  └──────────────┘                │
└──────────────────────────────────────────────────────┘
```

**Components:**
- **Manager Nodes:** Control the cluster, schedule tasks, maintain state
- **Worker Nodes:** Execute tasks (containers) assigned by managers
- **Services:** Define desired state of containers
- **Tasks:** Individual container instances running a service

### Setting Up Docker Swarm

#### Initialize a Swarm (On Manager Node)

```bash
# Initialize swarm on current machine
docker swarm init

# Output:
# Swarm initialized: current node (abc123) is now a manager.
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-xxx 192.168.1.100:2377

# Initialize with specific IP (multiple network interfaces)
docker swarm init --advertise-addr 192.168.1.100

# Initialize with custom port
docker swarm init --listen-addr 0.0.0.0:2377
```

#### Join Nodes to Swarm

**Add Worker Node:**
```bash
# On worker machine, use token from init command
docker swarm join --token SWMTKN-1-xxx 192.168.1.100:2377
```

**Add Manager Node:**
```bash
# On manager node, get manager join token
docker swarm join-token manager

# On new manager machine
docker swarm join --token SWMTKN-1-manager-xxx 192.168.1.100:2377
```

#### View Swarm Information

```bash
# View swarm nodes
docker node ls

# Output:
# ID              HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123 *        manager1   Ready   Active        Leader
# def456          worker1    Ready   Active
# ghi789          worker2    Ready   Active

# View node details
docker node inspect manager1

# View current node info
docker info

# Check swarm status
docker node ps
```

### Creating and Managing Services

Services are the definition of tasks to execute on nodes.

#### Create a Service

```bash
# Create simple service
docker service create --name web nginx

# Create with replicas (multiple instances)
docker service create --name web --replicas 3 nginx

# Create with port mapping
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx

# Create with resource constraints
docker service create \
  --name web \
  --replicas 3 \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  nginx

# Create with environment variables
docker service create \
  --name api \
  --env NODE_ENV=production \
  --env DB_HOST=database \
  my-api:latest
```

#### List and Inspect Services

```bash
# List all services
docker service ls

# View service details
docker service inspect web

# View service logs
docker service logs web

# View tasks (containers) of a service
docker service ps web

# See which nodes are running the service
docker service ps web --format "{{.Node}} {{.CurrentState}}"
```

#### Scale Services

```bash
# Scale service to 5 replicas
docker service scale web=5

# Scale multiple services
docker service scale web=5 api=3 db=1

# View scaling result
docker service ls
```

#### Update Services

```bash
# Update service image
docker service update --image nginx:latest web

# Update with rolling update configuration
docker service update \
  --update-parallelism 2 \
  --update-delay 10s \
  --image nginx:latest \
  web

# Add environment variable
docker service update --env-add NEW_VAR=value web

# Remove environment variable
docker service update --env-rm OLD_VAR web

# Update port mapping
docker service update --publish-add 8080:80 web

# Rollback to previous version
docker service rollback web
```

#### Remove Services

```bash
# Remove a service
docker service rm web

# Remove multiple services
docker service rm web api db
```

### Service Deployment Modes

#### Replicated Mode (Default)
Runs specified number of replica tasks across the cluster.

```bash
docker service create \
  --name web \
  --replicas 5 \
  --mode replicated \
  nginx
```

#### Global Mode
Runs exactly one task on every available node.

```bash
# Useful for monitoring agents, log collectors
docker service create \
  --name monitoring-agent \
  --mode global \
  prometheus/node-exporter
```

### Docker Stack (Multi-Service Deployment)

Stacks are groups of services defined in a docker-compose file.

**docker-stack.yml:**
```yaml
version: '3.8'

services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
    networks:
      - frontend

  api:
    image: myapi:latest
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    networks:
      - frontend
      - backend
    secrets:
      - db_password
    environment:
      - NODE_ENV=production

  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    networks:
      - backend
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

volumes:
  db-data:

networks:
  frontend:
  backend:

secrets:
  db_password:
    external: true
```

**Deploy Stack:**
```bash
# Create secret first
echo "mypassword" | docker secret create db_password -

# Deploy stack
docker stack deploy -c docker-stack.yml myapp

# List stacks
docker stack ls

# List services in stack
docker stack services myapp

# View stack tasks
docker stack ps myapp

# Remove stack
docker stack rm myapp
```

### Swarm Networking

#### Overlay Networks
Enables communication between services across nodes.

```bash
# Create overlay network
docker network create \
  --driver overlay \
  --attachable \
  my-overlay-network

# Create service on overlay network
docker service create \
  --name web \
  --network my-overlay-network \
  nginx

# Services on same overlay network can communicate
docker service create \
  --name api \
  --network my-overlay-network \
  myapi
```

#### Ingress Network
Built-in overlay network for load balancing.

```bash
# Published ports automatically use ingress network
docker service create \
  --name web \
  --publish 80:80 \
  --replicas 3 \
  nginx

# Access via any node IP:
# http://node1-ip:80
# http://node2-ip:80
# http://node3-ip:80
# All route to service instances automatically
```

### Secrets Management

Secrets securely store sensitive data in Swarm.

```bash
# Create secret from file
docker secret create db_password ./password.txt

# Create secret from stdin
echo "mypassword" | docker secret create db_password -

# List secrets
docker secret ls

# Inspect secret (won't show actual value)
docker secret inspect db_password

# Use secret in service
docker service create \
  --name db \
  --secret db_password \
  postgres

# Secret available at: /run/secrets/db_password

# Remove secret
docker secret rm db_password
```

### Configs Management

Configs store non-sensitive configuration files.

```bash
# Create config from file
docker config create nginx_config nginx.conf

# List configs
docker config ls

# Use config in service
docker service create \
  --name web \
  --config source=nginx_config,target=/etc/nginx/nginx.conf \
  nginx

# Remove config
docker config rm nginx_config
```

### Node Management

```bash
# Promote worker to manager
docker node promote worker1

# Demote manager to worker
docker node demote manager2

# Drain node (stop scheduling new tasks)
docker node update --availability drain worker1

# Make node active again
docker node update --availability active worker1

# Add label to node
docker node update --label-add environment=production worker1

# Remove node from swarm (on the node itself)
docker swarm leave

# Force remove node (from manager)
docker node rm --force worker1
```

### Placement Constraints

Control where services run in the cluster.

```bash
# Run only on managers
docker service create \
  --name monitoring \
  --constraint 'node.role==manager' \
  prometheus

# Run only on workers
docker service create \
  --name web \
  --constraint 'node.role==worker' \
  nginx

# Run on specific node
docker service create \
  --name db \
  --constraint 'node.hostname==node1' \
  postgres

# Run on labeled nodes
docker service create \
  --name cache \
  --constraint 'node.labels.environment==production' \
  redis
```

### Health Checks in Swarm

```bash
docker service create \
  --name web \
  --replicas 3 \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  nginx

# Swarm automatically restarts unhealthy containers
```

### Swarm Monitoring

```bash
# View cluster-wide stats
docker stats

# View service logs
docker service logs -f web

# View specific task logs
docker service logs web.1

# Watch service updates
watch docker service ps web
```

### Rolling Updates

```bash
# Configure rolling update
docker service update \
  --update-parallelism 2 \      # Update 2 at a time
  --update-delay 30s \          # Wait 30s between batches
  --update-failure-action rollback \  # Rollback on failure
  --image nginx:latest \
  web

# Monitor update progress
docker service ps web
```

### Swarm Best Practices

✅ **High Availability:**
- Use odd number of manager nodes (3, 5, 7)
- Don't run workloads on manager nodes in production
- Spread managers across different failure zones

✅ **Scheduling:**
- Use constraints for critical services
- Label nodes for better organization
- Use global mode for system services

✅ **Updates:**
- Always configure rolling updates
- Test updates in staging first
- Use health checks to detect issues
- Set rollback policies

✅ **Security:**
- Use secrets for sensitive data
- Enable TLS for cluster communication
- Regularly rotate join tokens
- Keep Docker engine updated

✅ **Monitoring:**
- Implement health checks
- Monitor service logs
- Track resource usage
- Set up alerting

### Leaving and Destroying Swarm

```bash
# Leave swarm (on worker)
docker swarm leave

# Leave swarm (on manager - must force)
docker swarm leave --force

# Remove all services before destroying
docker service rm $(docker service ls -q)

# On last manager, force leave to destroy swarm
docker swarm leave --force
```


## Docker Security

Security is critical when running containers in production. Docker provides multiple layers of security features.

### Container Isolation

#### Namespaces
Docker uses Linux namespaces to isolate containers:

- **PID namespace:** Process isolation
- **Network namespace:** Network interface isolation
- **Mount namespace:** Filesystem isolation
- **IPC namespace:** Inter-process communication isolation
- **UTS namespace:** Hostname isolation
- **User namespace:** User ID isolation

```bash
# View namespaces used by container
docker inspect --format='{{.State.Pid}}' mycontainer
ls -l /proc/PID/ns/
```

#### Control Groups (cgroups)
Limit resource usage:

```bash
# Limit CPU
docker run --cpus=".5" nginx

# Limit memory
docker run --memory="512m" nginx

# Limit both
docker run --cpus=".5" --memory="512m" nginx

# Set CPU priority
docker run --cpu-shares=512 nginx
```

### Running as Non-Root User

**Security Risk:** Running as root inside container = root on host.

#### In Dockerfile:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Change ownership
RUN chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

CMD ["node", "server.js"]
```

#### At Runtime:
```bash
# Run as specific user
docker run --user 1000:1000 nginx

# Run as nobody
docker run --user nobody nginx
```

### Image Security

#### 1. Use Official Images
```bash
# Good: Official image
FROM node:18-alpine

# Risky: Random user's image
FROM random-user/node
```

#### 2. Scan Images for Vulnerabilities

```bash
# Docker Scout (built-in)
docker scout quickview nginx:latest
docker scout cves nginx:latest

# Trivy (third-party)
trivy image nginx:latest

# Snyk
snyk container test nginx:latest
```

#### 3. Keep Images Updated
```bash
# Regularly rebuild images with latest base
docker build --no-cache -t myapp:latest .

# Update base images
docker pull node:18-alpine
docker build -t myapp:latest .
```

#### 4. Minimize Image Size
```dockerfile
# Use minimal base images
FROM alpine:latest

# Multi-stage builds
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm run build

FROM node:18-alpine
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]

# Remove unnecessary tools
RUN apk add --no-cache curl && \
    curl ... && \
    apk del curl
```

#### 5. Don't Store Secrets in Images
```dockerfile
# BAD: Secret in Dockerfile
ENV API_KEY=secret123

# BAD: Secret in image layer
RUN echo "password123" > /config/password.txt

# GOOD: Use environment variables at runtime
# docker run -e API_KEY=$API_KEY myapp

# GOOD: Use Docker secrets (Swarm)
# docker secret create api_key ./key.txt
```

### Runtime Security

#### 1. Read-Only Filesystem
```bash
# Make container filesystem read-only
docker run --read-only nginx

# With temporary writable space
docker run --read-only --tmpfs /tmp nginx
```

#### 2. Drop Linux Capabilities
```bash
# Drop all capabilities
docker run --cap-drop=ALL nginx

# Add only needed capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# List container capabilities
docker inspect --format='{{.HostConfig.CapDrop}}' mycontainer
```

#### 3. Security Options (SELinux/AppArmor)
```bash
# Use AppArmor profile
docker run --security-opt apparmor=docker-default nginx

# Disable AppArmor (not recommended)
docker run --security-opt apparmor=unconfined nginx

# SELinux label
docker run --security-opt label=level:s0:c100,c200 nginx
```

#### 4. No New Privileges
```bash
# Prevent privilege escalation
docker run --security-opt=no-new-privileges nginx
```

#### 5. Limit Resources
```bash
# Prevent DoS attacks
docker run \
  --memory="512m" \
  --memory-swap="512m" \
  --cpus="0.5" \
  --pids-limit=100 \
  nginx
```

### Network Security

#### 1. Use Custom Networks
```bash
# Don't use default bridge
docker network create --driver bridge secure-network

docker run --network secure-network myapp
```

#### 2. Disable Inter-Container Communication
```bash
# Create isolated network
docker network create --internal backend-network

docker run --network backend-network db
```

#### 3. Only Expose Necessary Ports
```bash
# Bad: Expose database to host
docker run -p 5432:5432 postgres

# Good: Only internal access
docker run --network backend postgres
```

#### 4. Use Encrypted Networks
```bash
# Overlay networks are encrypted by default in Swarm
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-overlay
```

### Secrets Management

#### Using Environment Variables (Basic)
```bash
# From command line
docker run -e DB_PASSWORD=$DB_PASSWORD myapp

# From .env file (never commit this!)
docker run --env-file .env myapp
```

#### Using Docker Secrets (Swarm)
```bash
# Create secret
echo "mypassword" | docker secret create db_password -

# Use in service
docker service create \
  --name db \
  --secret db_password \
  postgres

# Access in container at: /run/secrets/db_password
```

#### Using External Secret Management
```bash
# HashiCorp Vault, AWS Secrets Manager, etc.
docker run \
  -e VAULT_ADDR=https://vault.example.com \
  -e VAULT_TOKEN=$VAULT_TOKEN \
  myapp
```

### Security Scanning and Auditing

```bash
# Scan image for vulnerabilities
docker scout cves myapp:latest

# Check Docker daemon security
docker run --rm --network host --pid host --cap-add audit_control \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker/docker-bench-security

# Audit Docker configurations
docker system info --format '{{.SecurityOptions}}'
```

### Docker Content Trust (Image Signing)

```bash
# Enable content trust
export DOCKER_CONTENT_TRUST=1

# Push signed image
docker push myrepo/myapp:latest
# Prompts for passphrase, signs image

# Pull only signed images
docker pull myrepo/myapp:latest
# Fails if image not signed

# Disable for specific command
DOCKER_CONTENT_TRUST=0 docker pull unsigned-image
```

### Logging and Monitoring

```bash
# Enable Docker daemon logging
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Container logging
docker run --log-driver=syslog nginx

# Monitor container activity
docker events

# Check container processes
docker top mycontainer

# View container resource usage
docker stats mycontainer
```

### Security Best Practices Checklist

✅ **Images:**
- [ ] Use official base images
- [ ] Pin specific versions
- [ ] Scan for vulnerabilities regularly
- [ ] Use minimal images (alpine)
- [ ] Remove unnecessary packages
- [ ] Use multi-stage builds
- [ ] Sign images with Docker Content Trust

✅ **Runtime:**
- [ ] Run as non-root user
- [ ] Use read-only filesystem where possible
- [ ] Drop unnecessary Linux capabilities
- [ ] Enable no-new-privileges
- [ ] Set resource limits
- [ ] Use security options (AppArmor/SELinux)

✅ **Network:**
- [ ] Use custom networks
- [ ] Only expose necessary ports
- [ ] Bind to specific interfaces (127.0.0.1)
- [ ] Disable inter-container communication when not needed
- [ ] Use encrypted overlay networks

✅ **Secrets:**
- [ ] Never hardcode secrets
- [ ] Use environment variables or secret management
- [ ] Use Docker secrets in Swarm
- [ ] Rotate credentials regularly
- [ ] Use external secret stores for sensitive data

✅ **Monitoring:**
- [ ] Enable audit logging
- [ ] Monitor container activity
- [ ] Set up alerting
- [ ] Regular security scans
- [ ] Keep Docker updated

✅ **Access Control:**
- [ ] Limit Docker socket access
- [ ] Use TLS for remote Docker daemon
- [ ] Implement RBAC where possible
- [ ] Regular security audits

### Security Tools

**Vulnerability Scanning:**
- Docker Scout (built-in)
- Trivy
- Snyk
- Clair
- Anchore

**Security Auditing:**
- Docker Bench Security
- Falco (runtime security)
- Sysdig

**Secret Management:**
- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Docker Secrets (Swarm)

### Common Security Pitfalls to Avoid

❌ **Don't:**
- Run containers as root
- Mount Docker socket into containers
- Use `--privileged` flag unless absolutely necessary
- Store secrets in images or environment variables visible in `docker inspect`
- Expose Docker daemon without TLS
- Use `latest` tag in production
- Disable security features for convenience

✅ **Do:**
- Follow principle of least privilege
- Regularly update and patch
- Scan images before deployment
- Implement defense in depth
- Monitor and audit continuously
- Use security policies and enforcement tools

### Summary

**Docker Swarm:**
- Native orchestration for Docker
- Simple to set up and use
- Built-in load balancing and service discovery
- Great for small to medium deployments
- Use stacks for multi-service applications

**Docker Security:**
- Multiple layers: isolation, image security, runtime security
- Always run as non-root
- Scan images for vulnerabilities
- Use secrets management
- Implement network isolation
- Monitor and audit regularly
- Security is an ongoing process, not a one-time setup

Security and orchestration go hand-in-hand for production deployments! 🔒🚀
