## Docker Networking

Docker networking allows containers to communicate with each other, the host machine, and external networks.

### Network Types

Docker provides several network drivers for different use cases:

**1. Bridge (Default):**
- Default network for containers
- Containers on the same bridge network can communicate with each other
- Provides isolation from the host network
- Best for standalone containers on a single host

**2. Host:**
- Container shares the host's network stack
- No network isolation between container and host
- Container uses host's IP address directly
- Better performance but less isolation

**3. None:**
- Disables all networking for the container
- Complete network isolation
- Used for containers that don't need network access

**4. Overlay:**
- Connects multiple Docker daemons together
- Used in Docker Swarm for multi-host networking
- Allows containers on different hosts to communicate

**5. Macvlan:**
- Assigns a MAC address to a container
- Makes container appear as a physical device on the network
- Used when containers need to look like physical hosts

### Network Commands

- `docker network ls` - List all networks
- `docker network create <network-name>` - Create a new network
- `docker network inspect <network>` - View network details
- `docker network connect <network> <container>` - Connect container to network
- `docker network disconnect <network> <container>` - Disconnect container from network
- `docker network rm <network>` - Remove a network

### Ports and Port Forwarding

Port forwarding (also called port mapping or publishing) allows external access to services running inside containers.

**The Problem:**
- Containers have their own isolated network
- By default, services inside containers are not accessible from outside
- Each container has its own IP address in Docker's internal network

**The Solution - Port Mapping:**
Map a port on the host machine to a port inside the container.

### Port Mapping Syntax

```bash
docker run -p <host-port>:<container-port> <image>
```

**Examples:**

```bash
# Map host port 8080 to container port 80
docker run -p 8080:80 nginx

# Access nginx at: http://localhost:8080
```

```bash
# Map host port 3000 to container port 3000
docker run -p 3000:3000 my-node-app

# Access app at: http://localhost:3000
```

```bash
# Map multiple ports
docker run -p 8080:80 -p 443:443 nginx
```

```bash
# Bind to specific host IP
docker run -p 127.0.0.1:8080:80 nginx

# Only accessible from localhost, not external network
```

```bash
# Let Docker assign a random host port
docker run -P nginx

# Docker automatically maps all EXPOSED ports to random high ports
# Use 'docker ps' to see which ports were assigned
```

### Understanding Port Mapping

**Format:** `HOST:CONTAINER`
- **Host Port (left side):** Port on your machine where you access the service
- **Container Port (right side):** Port where the service listens inside the container

**Example Scenario:**
```
Your machine (localhost) ──► Port 8080 ──► Docker maps to ──► Container Port 80 ──► nginx
```

When you visit `http://localhost:8080`, Docker forwards the request to port 80 inside the container.

### Checking Port Mappings

```bash
# View port mappings for running containers
docker ps

# Output shows PORT column:
# 0.0.0.0:8080->80/tcp
# This means: host port 8080 maps to container port 80
```

```bash
# Inspect specific container's network settings
docker inspect <container>

# Or use port command
docker port <container>
```

### Container-to-Container Communication

Containers on the same network can communicate using:

**1. Container Names (Recommended):**
```bash
# Create a custom network
docker network create my-network

# Run containers on the same network
docker run --name database --network my-network postgres
docker run --name webapp --network my-network my-app

# webapp can connect to database using hostname "database"
# No port mapping needed for internal communication
```

**2. Container IP Address:**
- Find IP: `docker inspect <container> | grep IPAddress`
- Less reliable (IPs can change)

### Common Port Mapping Use Cases

| Service | Container Port | Common Host Port | Command |
|---------|---------------|------------------|---------|
| Web Server (nginx/Apache) | 80 | 8080 | `docker run -p 8080:80 nginx` |
| HTTPS | 443 | 443 | `docker run -p 443:443 nginx` |
| Node.js App | 3000 | 3000 | `docker run -p 3000:3000 node-app` |
| MySQL Database | 3306 | 3306 | `docker run -p 3306:3306 mysql` |
| PostgreSQL | 5432 | 5432 | `docker run -p 5432:5432 postgres` |
| MongoDB | 27017 | 27017 | `docker run -p 27017:27017 mongo` |
| Redis | 6379 | 6379 | `docker run -p 6379:6379 redis` |

### Important Notes

⚠️ **Port Conflicts:** You can't map two containers to the same host port
```bash
# This works:
docker run -p 8080:80 nginx
docker run -p 8081:80 nginx  # Different host port

# This fails:
docker run -p 8080:80 nginx
docker run -p 8080:80 apache  # Same host port - ERROR!
```

✅ **Best Practices:**
- Use custom networks for container-to-container communication
- Only expose ports that need external access
- Use specific host IPs to limit exposure when needed
- Document port mappings in your docker-compose files


## Best Practices

### Dockerfile Best Practices

#### 1. Use Official Base Images
```dockerfile
# Good: Use official images
FROM node:18-alpine

# Avoid: Unofficial or unknown images
FROM random-user/node
```
- Official images are maintained, secure, and optimized
- Use specific versions, not `latest` tag

#### 2. Use Specific Tags (Version Pinning)
```dockerfile
# Good: Specific version
FROM node:18.17.0-alpine

# Acceptable: Major version
FROM node:18-alpine

# Bad: No version control
FROM node:latest
```
- `latest` tag can change unexpectedly
- Ensures reproducible builds

#### 3. Minimize Layer Count
```dockerfile
# Bad: Multiple RUN commands create multiple layers
RUN apt-get update
RUN apt-get install -y package1
RUN apt-get install -y package2

# Good: Combine into one layer
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    rm -rf /var/lib/apt/lists/*
```
- Fewer layers = smaller images
- Clean up in the same layer where you install

#### 4. Use .dockerignore File
Create `.dockerignore` to exclude unnecessary files:
```
node_modules
npm-debug.log
.git
.env
*.md
.vscode
```
- Reduces build context size
- Faster builds
- Prevents sensitive files from being copied

#### 5. Order Instructions by Change Frequency
```dockerfile
# Good: Least likely to change first
FROM node:18-alpine
WORKDIR /app

# Dependencies change less frequently
COPY package*.json ./
RUN npm install

# Source code changes frequently - put last
COPY . .

CMD ["npm", "start"]
```
- Leverages Docker's layer caching
- Faster rebuilds when only code changes

#### 6. Use Multi-Stage Builds
```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm install --production
CMD ["node", "dist/index.js"]
```
- Smaller final images
- Build tools not included in production
- More secure

#### 7. Run as Non-Root User
```dockerfile
# Create and use non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

USER nodejs

CMD ["node", "app.js"]
```
- Security best practice
- Limits potential damage from vulnerabilities

#### 8. Use COPY Instead of ADD
```dockerfile
# Good: Use COPY for local files
COPY package.json .

# Only use ADD for URLs or tar extraction
ADD https://example.com/file.tar.gz /tmp/
```
- `COPY` is more transparent
- `ADD` has hidden features that can be confusing

#### 9. Set WORKDIR Instead of Using cd
```dockerfile
# Good
WORKDIR /app
COPY . .

# Bad
RUN cd /app
COPY . .
```
- `WORKDIR` is persistent across layers
- `cd` only works within a single RUN instruction

#### 10. Use EXPOSE to Document Ports
```dockerfile
EXPOSE 3000
```
- Documents which ports the application uses
- Doesn't actually publish the port (use -p flag)

#### 11. Keep Images Small
```dockerfile
# Use alpine variants
FROM node:18-alpine

# Remove unnecessary packages after installation
RUN apk add --no-cache build-base && \
    npm install && \
    apk del build-base

# Use --production for Node.js
RUN npm install --production
```
- Smaller images = faster pulls and deployments
- Reduced attack surface

#### 12. Use Build Arguments for Flexibility
```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

ARG APP_ENV=production
ENV NODE_ENV=${APP_ENV}
```
- Makes Dockerfiles reusable
- Build-time customization

### Docker Compose Best Practices

#### 1. Use Version Control for Compose Files
```yaml
version: '3.8'  # Specify compose file version
```
- Use version 3.x for modern features
- Document your compose file format

#### 2. Use Environment Variables
```yaml
services:
  app:
    image: myapp:${TAG:-latest}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - API_KEY=${API_KEY}
```
Create `.env` file:
```
TAG=v1.0.0
DATABASE_URL=postgres://localhost:5432/db
API_KEY=secret-key
```
- Don't hardcode secrets
- Makes configuration flexible
- Never commit `.env` files to git

#### 3. Use Named Volumes
```yaml
# Good: Named volumes
volumes:
  postgres_data:
  redis_data:

services:
  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data

# Avoid: Anonymous volumes
services:
  db:
    volumes:
      - /var/lib/postgresql/data
```
- Named volumes are easier to manage
- Persist data between container recreations

#### 4. Define Health Checks
```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```
- Ensures services are actually working
- Enables automatic recovery

#### 5. Use Depends_on with Conditions
```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy
      
  db:
    healthcheck:
      test: ["CMD", "pg_isready"]
```
- Control startup order
- Wait for dependencies to be ready

#### 6. Use Resource Limits
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```
- Prevents resource exhaustion
- Ensures fair resource distribution

#### 7. Use Restart Policies
```yaml
services:
  app:
    restart: unless-stopped  # or 'always', 'on-failure'
```
- `no`: Never restart (default)
- `always`: Always restart
- `on-failure`: Only restart on error
- `unless-stopped`: Restart unless manually stopped

#### 8. Define Networks Explicitly
```yaml
networks:
  frontend:
  backend:

services:
  web:
    networks:
      - frontend
  
  api:
    networks:
      - frontend
      - backend
  
  db:
    networks:
      - backend
```
- Isolate services properly
- Control which services can communicate

#### 9. Don't Store Secrets in Compose Files
```yaml
# Bad: Secrets in plain text
services:
  db:
    environment:
      - POSTGRES_PASSWORD=mysecretpassword

# Good: Use environment variables
services:
  db:
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}

# Better: Use Docker secrets (Swarm mode)
services:
  db:
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Networking Best Practices

#### 1. Create Custom Networks
```bash
# Don't rely on default bridge network
docker network create my-app-network

docker run --network my-app-network --name api myapi
docker run --network my-app-network --name db postgres
```
- Better isolation
- Automatic DNS resolution
- More control over network configuration

#### 2. Use Service Names for Communication
```yaml
services:
  api:
    networks:
      - backend
  
  db:
    networks:
      - backend

# In API code, connect to: postgres://db:5432/mydb
# Not: postgres://172.17.0.3:5432/mydb
```
- Container IPs can change
- Service names are stable and readable

#### 3. Isolate Networks by Purpose
```yaml
networks:
  frontend:      # Public-facing services
  backend:       # Internal services
  database:      # Database tier

services:
  web:
    networks:
      - frontend
  
  api:
    networks:
      - frontend
      - backend
  
  db:
    networks:
      - database
```
- Principle of least privilege
- Limit blast radius of compromises

#### 4. Only Expose Necessary Ports
```yaml
services:
  # Bad: Exposing database to host
  db:
    ports:
      - "5432:5432"
  
  # Good: Only internal access
  db:
    expose:
      - 5432
    # No ports mapping - only accessible within Docker network
```
- `expose` documents ports but doesn't publish them
- Reduces attack surface

#### 5. Bind to Specific Interfaces
```bash
# Bad: Accessible from anywhere
docker run -p 3306:3306 mysql

# Good: Only accessible from localhost
docker run -p 127.0.0.1:3306:3306 mysql
```
- Limit exposure to local machine only
- Use firewall rules for additional security

### Security Best Practices Summary

✅ **Images:**
- Use official base images
- Pin specific versions
- Scan images for vulnerabilities
- Keep images updated

✅ **Runtime:**
- Run as non-root user
- Use read-only file systems where possible
- Limit resources
- Drop unnecessary capabilities

✅ **Network:**
- Minimize exposed ports
- Use custom networks
- Isolate services
- Encrypt sensitive traffic

✅ **Secrets:**
- Never commit secrets to code
- Use environment variables or secret management
- Rotate credentials regularly
- Limit secret access to necessary containers only


## How to Create Your Own Dockerfiles

A Dockerfile is a script containing instructions to build a Docker image. This guide covers creating Dockerfiles from scratch for different types of applications.

### Basic Dockerfile Structure

Every Dockerfile follows this general pattern:

```dockerfile
# 1. Choose a base image
FROM base-image:tag

# 2. Set metadata (optional)
LABEL maintainer="your-email@example.com"

# 3. Set working directory
WORKDIR /app

# 4. Copy dependency files
COPY package.json .

# 5. Install dependencies
RUN install-command

# 6. Copy application code
COPY . .

# 7. Expose ports (documentation)
EXPOSE port-number

# 8. Set environment variables (optional)
ENV KEY=value

# 9. Define the command to run
CMD ["executable", "param1", "param2"]
```

### Step-by-Step: Creating Your First Dockerfile

#### Step 1: Choose a Base Image

Start with `FROM` - every Dockerfile must begin with this.

```dockerfile
# Official language images
FROM python:3.11
FROM node:18
FROM golang:1.21
FROM openjdk:17

# Minimal Alpine variants (smaller size)
FROM python:3.11-alpine
FROM node:18-alpine

# Specific OS
FROM ubuntu:22.04
FROM debian:bullseye

# Scratch (empty image, for compiled binaries)
FROM scratch
```

**How to choose:**
- Use official images when available
- Use Alpine variants for smaller images
- Match your development environment
- Check Docker Hub for available tags

#### Step 2: Set Working Directory

```dockerfile
FROM node:18-alpine

# Create and set working directory
WORKDIR /app

# All subsequent commands run from /app
# If directory doesn't exist, it's created automatically
```

#### Step 3: Copy Dependency Files First

```dockerfile
FROM node:18-alpine
WORKDIR /app

# Copy only dependency files first (for caching)
COPY package.json package-lock.json ./

# For Python
# COPY requirements.txt .

# For Go
# COPY go.mod go.sum ./
```

**Why this order?**
- Dependencies change less frequently than code
- Leverages Docker's layer caching
- Faster rebuilds when only code changes

#### Step 4: Install Dependencies

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# For Python
# RUN pip install --no-cache-dir -r requirements.txt

# For Go
# RUN go mod download

# Clean up in same layer for smaller image
RUN npm ci --only=production && \
    npm cache clean --force
```

#### Step 5: Copy Application Code

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Copy all application files
COPY . .

# Or copy specific directories
# COPY src ./src
# COPY public ./public
```

#### Step 6: Expose Ports

```dockerfile
# Document which port the app listens on
EXPOSE 3000

# This is documentation only - doesn't actually publish the port
# Use -p flag when running: docker run -p 3000:3000 myapp
```

#### Step 7: Define the Startup Command

```dockerfile
# Shell form (runs in shell, allows variable expansion)
CMD npm start

# Exec form (preferred, doesn't start shell)
CMD ["npm", "start"]
CMD ["node", "server.js"]
CMD ["python", "app.py"]

# Or use ENTRYPOINT for fixed command with flexible params
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]  # Default params, can be overridden
```

### Complete Example: Node.js Application

```dockerfile
# Use official Node.js runtime as base
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application source
COPY . .

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Change ownership of app files
RUN chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check (optional)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js || exit 1

# Start the application
CMD ["node", "server.js"]
```

**Build and run:**
```bash
# Build the image
docker build -t my-node-app .

# Run the container
docker run -p 3000:3000 my-node-app
```

### Example: Python Flask Application

```dockerfile
# Python 3.11 slim variant
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies if needed
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Copy requirements first
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose Flask port
EXPOSE 5000

# Set environment variables
ENV FLASK_APP=app.py
ENV FLASK_ENV=production

# Run the application
CMD ["flask", "run", "--host=0.0.0.0"]
```

**requirements.txt:**
```
Flask==2.3.0
gunicorn==20.1.0
```

### Example: Go Application

```dockerfile
# Multi-stage build for Go

# Stage 1: Build
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Stage 2: Run
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy binary from builder
COPY --from=builder /app/main .

EXPOSE 8080

CMD ["./main"]
```

### Example: Multi-Stage Build (React App)

```dockerfile
# Stage 1: Build the React app
FROM node:18-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine

# Copy built files from builder stage
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Using Build Arguments

Build arguments allow customization at build time:

```dockerfile
FROM node:18-alpine

# Define build argument with default value
ARG NODE_ENV=production
ARG APP_VERSION=1.0.0

# Use build argument
ENV NODE_ENV=${NODE_ENV}
ENV APP_VERSION=${APP_VERSION}

WORKDIR /app

COPY package*.json ./

# Conditional installation based on environment
RUN if [ "$NODE_ENV" = "development" ]; then \
      npm install; \
    else \
      npm ci --only=production; \
    fi

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

**Build with arguments:**
```bash
# Use default values
docker build -t myapp .

# Override arguments
docker build --build-arg NODE_ENV=development --build-arg APP_VERSION=2.0.0 -t myapp:dev .
```

### Using .dockerignore

Create `.dockerignore` in the same directory as Dockerfile:

```
# Dependencies
node_modules
npm-debug.log
venv/
__pycache__/

# Git
.git
.gitignore

# IDE
.vscode
.idea
*.swp

# OS
.DS_Store
Thumbs.db

# Environment files
.env
.env.local

# Documentation
README.md
*.md

# Tests
tests/
**/*test.js

# Build artifacts
dist/
build/
*.log
```

### Creating Dockerfile: Best Practices Checklist

✅ **Structure:**
- [ ] Use official base images
- [ ] Pin specific versions (not `latest`)
- [ ] Order instructions from least to most frequently changing
- [ ] Use multi-stage builds for smaller images

✅ **Dependencies:**
- [ ] Copy dependency files before source code
- [ ] Install dependencies in one RUN command
- [ ] Clean up package manager caches in same layer

✅ **Security:**
- [ ] Run as non-root user
- [ ] Don't include secrets in image
- [ ] Use minimal base images (alpine)
- [ ] Keep images updated

✅ **Optimization:**
- [ ] Create and use .dockerignore
- [ ] Combine RUN commands where appropriate
- [ ] Remove unnecessary files in same layer
- [ ] Use COPY instead of ADD (unless needed)

✅ **Documentation:**
- [ ] Add LABEL for metadata
- [ ] Use EXPOSE to document ports
- [ ] Add comments explaining complex steps
- [ ] Include health checks

### Quick Reference: Dockerfile Instructions

| Instruction | Purpose | Example |
|------------|---------|---------|
| `FROM` | Set base image | `FROM node:18-alpine` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files from host | `COPY . .` |
| `ADD` | Copy and extract archives | `ADD file.tar.gz /app` |
| `RUN` | Execute commands during build | `RUN npm install` |
| `CMD` | Default command to run | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | Configure container as executable | `ENTRYPOINT ["python"]` |
| `ENV` | Set environment variables | `ENV NODE_ENV=production` |
| `ARG` | Build-time variables | `ARG VERSION=1.0` |
| `EXPOSE` | Document ports | `EXPOSE 3000` |
| `VOLUME` | Create mount point | `VOLUME /data` |
| `USER` | Set user for RUN/CMD | `USER node` |
| `LABEL` | Add metadata | `LABEL version="1.0"` |
| `HEALTHCHECK` | Configure health check | `HEALTHCHECK CMD curl localhost` |

### Summary

**Creating a Dockerfile involves:**
1. Choose appropriate base image
2. Set working directory
3. Copy and install dependencies first (caching)
4. Copy application code
5. Configure security (non-root user)
6. Expose ports and set environment
7. Define startup command

**Key Principles:**
- Order matters for caching
- Keep images small
- Run as non-root
- One concern per layer
- Document with labels
- Test thoroughly

Start simple, then optimize! 🚀
