## Docker CLI (Command Line Interface)

The Docker CLI is the primary way to interact with Docker. All commands start with `docker` followed by the specific command.

### Essential Docker Commands

#### Container Management:
- `docker run <image>` - Create and start a container from an image
- `docker ps` - List running containers
- `docker ps -a` - List all containers (including stopped)
- `docker start <container>` - Start a stopped container
- `docker stop <container>` - Stop a running container
- `docker restart <container>` - Restart a container
- `docker rm <container>` - Remove a container
- `docker exec -it <container> <command>` - Execute a command in a running container

#### Image Management:
- `docker images` - List all local images
- `docker pull <image>` - Download an image from registry
- `docker build -t <name> .` - Build an image from Dockerfile
- `docker push <image>` - Upload an image to registry
- `docker rmi <image>` - Remove an image
- `docker tag <image> <new-tag>` - Tag an image

#### System Information:
- `docker version` - Show Docker version
- `docker info` - Display system-wide information
- `docker logs <container>` - Fetch logs from a container
- `docker inspect <container/image>` - Return detailed information
- `docker stats` - Display resource usage statistics

#### Cleanup Commands:
- `docker system prune` - Remove unused data
- `docker container prune` - Remove stopped containers
- `docker image prune` - Remove unused images
- `docker volume prune` - Remove unused volumes

### First Docker Command - Hello World

```bash
docker run hello-world
```

**What happened:**

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
17eec7bbc9d7: Pull complete
Digest: sha256:56433a6be3fda188089fb548eae3d91df3ed0d6589f7c2656121b911198df065
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

**Key Takeaways from this command:**
1. Docker first checks locally for the image
2. If not found locally, it pulls from Docker Hub
3. A new container is created from the image
4. The container executes and produces output
5. The output is streamed back to your terminal

This demonstrates the complete Docker workflow: pull image → create container → run → output!

### Common Docker Run Options

- `docker run -d <image>` - Run container in detached mode (background)
- `docker run -p 8080:80 <image>` - Map port 8080 on host to port 80 in container
- `docker run -v /host/path:/container/path <image>` - Mount a volume
- `docker run --name mycontainer <image>` - Assign a name to the container
- `docker run -e VAR=value <image>` - Set environment variable
- `docker run -it <image>` - Interactive mode with terminal (for shells like bash)
- `docker run --rm <image>` - Automatically remove container when it exits


## Docker Init - Quick Project Setup

`docker init` is a command-line utility that helps you initialize Docker assets (Dockerfile, docker-compose.yaml, .dockerignore) for your project automatically.

### What is docker init?

- Introduced in Docker Desktop 4.18+ and Docker Engine 24.0+
- Interactive tool that analyzes your project and generates appropriate Docker files
- Supports multiple languages and frameworks
- Creates production-ready Docker configurations
- Saves time and follows best practices

### How to Use docker init

```bash
# Navigate to your project directory
cd my-project

# Run docker init
docker init

# Follow the interactive prompts
```

### Interactive Prompts

The tool will ask you questions like:
1. **Application platform:** What language/framework? (Node, Python, Go, Rust, ASP.NET, PHP, Java, Other)
2. **Project type:** (if applicable)
3. **Version:** Which version of the language?
4. **Port:** What port does your app use?
5. **Entry point:** Main file to run your application

### Example: Initializing a Node.js Project

```bash
$ docker init

Welcome to the Docker Init CLI!

This utility will walk you through creating the following files with sensible defaults for your project:
  - .dockerignore
  - Dockerfile
  - compose.yaml
  - README.Docker.md

? What application platform does your project use?  [Use arrows to move, type to filter]
> Node
  Python
  Go
  Rust
  ASP.NET Core
  PHP
  Java
  Other

? What version of Node do you want to use? (18.0.0)

? Which package manager do you want to use?
> npm
  yarn
  pnpm

? What command do you want to use to start the app? (node src/index.js)

? What port does your server listen on? (3000)

✔ Created Dockerfile
✔ Created .dockerignore
✔ Created compose.yaml
✔ Created README.Docker.md

→ Your Docker files are ready!
  Review and customize them as needed.
  
Next steps:
  1. Review the generated files
  2. Build your image: docker build -t my-project .
  3. Run your container: docker run -p 3000:3000 my-project
  4. Or use Compose: docker compose up --build
```

### Generated Files

**1. Dockerfile (Node.js example):**
```dockerfile
# syntax=docker/dockerfile:1

ARG NODE_VERSION=18.0.0

FROM node:${NODE_VERSION}-alpine

ENV NODE_ENV production

WORKDIR /usr/src/app

RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

USER node

COPY . .

EXPOSE 3000

CMD node src/index.js
```

**2. .dockerignore:**
```
# Include any files or directories that you don't want to be copied to your
# container here (e.g., local build artifacts, temporary files, etc.)
node_modules
npm-debug.log
.git
.gitignore
.dockerignore
.vscode
.idea
*.md
!README.md
.env
.env.*
.DS_Store
Thumbs.db
dist/
build/
coverage/
```

**3. compose.yaml:**
```yaml
services:
  server:
    build:
      context: .
    environment:
      NODE_ENV: production
    ports:
      - 3000:3000
```

**4. README.Docker.md:**
Contains instructions on how to:
- Build the image
- Run the container
- Use Docker Compose
- Deploy to production

### Supported Languages/Frameworks

| Platform | Details |
|----------|---------|
| **Node.js** | npm, yarn, pnpm support |
| **Python** | pip, poetry, requirements.txt |
| **Go** | Go modules, vendor support |
| **Rust** | Cargo support |
| **ASP.NET Core** | .NET SDK versions |
| **PHP** | Composer support |
| **Java** | Maven, Gradle support |
| **Other** | Generic template |

### Benefits of docker init

✅ **Best Practices Built-in:**
- Multi-stage builds where appropriate
- Non-root user for security
- Optimized layer caching
- Efficient .dockerignore files
- Production-ready configurations

✅ **Time Saving:**
- No need to write Dockerfile from scratch
- Follows language-specific conventions
- Includes helpful comments

✅ **Consistency:**
- Standardized Docker files across projects
- Team members use same structure
- Easier onboarding

✅ **Learning Tool:**
- See how proper Dockerfiles are structured
- Understand best practices
- Customize based on generated template

### Complete Workflow Example

```bash
# 1. Create new Node.js project
mkdir my-app && cd my-app
npm init -y
echo "console.log('Hello Docker!')" > index.js

# 2. Initialize Docker files
docker init
# Select Node, accept defaults

# 3. Review generated files
cat Dockerfile
cat compose.yaml

# 4. Build and run
docker compose up --build

# 5. Test
curl http://localhost:3000
```

### docker init vs Manual Creation

| Aspect | docker init | Manual |
|--------|-------------|--------|
| **Speed** | ✅ Instant | ❌ Slower |
| **Best Practices** | ✅ Built-in | ⚠️ Depends on knowledge |
| **Learning Curve** | ✅ Easy | ⚠️ Requires expertise |
| **Customization** | ⚠️ Requires editing | ✅ Full control |
| **Project Specific** | ⚠️ Generic template | ✅ Tailored |

### Quick Reference

```bash
# Basic usage
docker init

# Skip prompts (use defaults)
docker init --yes

# Check version
docker init --version

# Get help
docker init --help
```


## Docker Commit

The `docker commit` command creates a new image from a container's changes. It captures the current state of a container into a new image.

### Basic Syntax

```bash
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
```

### How Docker Commit Works

1. Make changes inside a running container
2. Commit those changes to create a new image
3. New image includes all file system changes
4. Can create new containers from this image

### Basic Example

```bash
# Start a container
docker run -it --name my-container ubuntu bash

# Inside container: Make changes
apt-get update
apt-get install -y curl
echo "Hello Docker" > /hello.txt
exit

# Commit changes to new image
docker commit my-container my-custom-ubuntu

# New image is created
docker images
# REPOSITORY          TAG       IMAGE ID
# my-custom-ubuntu    latest    abc123def456

# Run container from new image
docker run -it my-custom-ubuntu bash
# curl and /hello.txt are already there!
```

### Commit Options

```bash
# Add commit message
docker commit -m "Installed curl and added hello.txt" my-container my-image

# Add author information
docker commit --author "John Doe <john@example.com>" my-container my-image

# Combine message and author
docker commit -m "Added nginx" --author "John Doe" my-container my-nginx

# Change CMD instruction
docker commit --change='CMD ["nginx", "-g", "daemon off;"]' my-container my-nginx

# Pause container during commit (default is true)
docker commit --pause=false my-container my-image
```

### Advanced Commit with Changes

You can modify image configuration during commit:

```bash
# Change multiple instructions
docker commit \
  --change='ENV APP_VERSION=1.0' \
  --change='EXPOSE 8080' \
  --change='CMD ["python", "app.py"]' \
  my-container my-app:v1.0

# This is equivalent to adding these to a Dockerfile
```

### Commit vs Dockerfile

| Aspect | Docker Commit | Dockerfile |
|--------|---------------|------------|
| **Reproducibility** | ❌ Manual, not reproducible | ✅ Automated, reproducible |
| **Version Control** | ❌ Hard to track changes | ✅ Easy to version with git |
| **Transparency** | ❌ Changes are hidden | ✅ All steps documented |
| **Best Practice** | ❌ Not recommended for production | ✅ Recommended approach |
| **Speed** | ✅ Quick for prototyping | ⚠️ Requires writing Dockerfile |
| **Automation** | ❌ Manual process | ✅ Automated with CI/CD |

### When to Use Docker Commit

✅ **Good Use Cases:**
- Quick prototyping and experimentation
- Debugging and testing fixes
- Creating temporary snapshots
- Learning and exploration

❌ **Avoid For:**
- Production images
- Team collaboration
- CI/CD pipelines
- Long-term maintenance

### Important Notes

⚠️ **Commit Limitations:**
- Only captures filesystem changes
- Doesn't capture:
  - Volume data
  - Port mappings
  - Environment variables (unless changed with --change)
  - Running processes
  - Network configuration

⚠️ **Data in Volumes is NOT Committed:**
```bash
docker run -v my-volume:/data --name test alpine
# Add data to /data
docker commit test test-image
# Volume data is NOT in test-image!
```

### Best Practice: Dockerfile Instead of Commit

Instead of using commit, convert your manual steps to a Dockerfile:

```dockerfile
# Instead of committing after manual installation
# Write a Dockerfile:

FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

RUN echo "Hello Docker" > /hello.txt

CMD ["/bin/bash"]
```

Then build:
```bash
docker build -t my-custom-ubuntu .
```

**Advantages:**
- ✅ Reproducible
- ✅ Version controlled
- ✅ Documented
- ✅ Can be automated
- ✅ Team can collaborate


## Docker Volumes

Volumes are the preferred mechanism for persisting data generated and used by Docker containers. Unlike container storage, volumes exist outside the container lifecycle.

### Why Use Volumes?

**The Problem:**
- Containers are ephemeral (temporary)
- When a container is deleted, all data inside it is lost
- Data stored in container layers is tightly coupled to the container

**The Solution - Volumes:**
- Data persists even after container is removed
- Can be shared between multiple containers
- Easier to backup and migrate
- Better performance than bind mounts on some systems

### Types of Data Persistence

#### 1. Volumes (Recommended)
Managed by Docker, stored in Docker's storage directory.

```bash
# Create a named volume
docker volume create my-volume

# Use volume with container
docker run -v my-volume:/app/data nginx

# Or using --mount (more explicit)
docker run --mount source=my-volume,target=/app/data nginx
```

**Advantages:**
- ✅ Managed by Docker
- ✅ Easy to backup and restore
- ✅ Work on all platforms
- ✅ Can be shared safely between containers
- ✅ Drivers allow storing on remote hosts or cloud

#### 2. Bind Mounts
Mount a specific host directory into container.

```bash
# Mount host directory to container
docker run -v /host/path:/container/path nginx

# Or using --mount
docker run --mount type=bind,source=/host/path,target=/container/path nginx
```

**Use Cases:**
- Development (mount source code)
- Configuration files
- Sharing files between host and container

**Disadvantages:**
- ⚠️ Depends on host filesystem structure
- ⚠️ Less portable
- ⚠️ Host processes can modify data

#### 3. tmpfs Mounts (Linux only)
Stored in host memory, never written to filesystem.

```bash
docker run --mount type=tmpfs,target=/app/temp nginx
```

**Use Cases:**
- Temporary data
- Sensitive information that shouldn't be persisted
- Performance-critical temporary storage

### Volume Commands

```bash
# Create a volume
docker volume create my-data

# List all volumes
docker volume ls

# Inspect volume details
docker volume inspect my-data

# Remove a volume
docker volume rm my-data

# Remove all unused volumes
docker volume prune

# Remove volumes with filter
docker volume prune --filter "label=environment=dev"
```

### Using Volumes in Practice

#### Example 1: Database Persistence
```bash
# PostgreSQL with volume
docker run -d \
  --name postgres-db \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Data persists even if container is deleted
docker rm -f postgres-db

# Start new container with same data
docker run -d \
  --name postgres-db-new \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15
```

#### Example 2: Development with Bind Mounts
```bash
# Mount source code for live reloading
docker run -d \
  --name dev-app \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  node:18

# Changes to src/ directory immediately reflected in container
```

#### Example 3: Sharing Data Between Containers
```bash
# Create shared volume
docker volume create shared-data

# Container 1 writes data
docker run -v shared-data:/data alpine sh -c "echo 'Hello' > /data/message.txt"

# Container 2 reads data
docker run -v shared-data:/data alpine cat /data/message.txt
# Output: Hello
```

### Volume Backup and Restore

#### Backup a Volume
```bash
# Backup volume to tar file
docker run --rm \
  -v my-volume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/my-volume-backup.tar.gz -C /data .

# Creates: my-volume-backup.tar.gz in current directory
```

#### Restore a Volume
```bash
# Create new volume
docker volume create my-volume-restored

# Restore from backup
docker run --rm \
  -v my-volume-restored:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/my-volume-backup.tar.gz -C /data

# Data is now in my-volume-restored
```

### Best Practices for Volumes

✅ **Use Named Volumes for Important Data**
```yaml
# Good: Named volume
volumes:
  - db-data:/var/lib/postgresql/data

# Avoid: Anonymous volume
volumes:
  - /var/lib/postgresql/data
```

✅ **Don't Store Volumes in Container Images**
- Volumes should be created at runtime, not in Dockerfile

✅ **Use Read-Only Mounts When Possible**
```bash
docker run -v config:/app/config:ro myapp
```

✅ **Regularly Backup Important Volumes**
- Automate backups for production data
- Test restore procedures

✅ **Clean Up Unused Volumes**
```bash
# Remove volumes not used by any container
docker volume prune
```

### Summary

**Docker Volumes:**
- Persist data beyond container lifecycle
- Preferred method for data storage
- Use named volumes for important data
- Bind mounts for development
- Always backup important volumes

**Docker Commit:**
- Creates images from container changes
- Useful for prototyping and debugging
- NOT recommended for production
- Use Dockerfiles for reproducible images
- Good for learning and experimentation
