# Docker Installation and Commands

Comprehensive guide to Docker concepts, installation, and essential commands for containerization.

## Table of Contents
- [Docker Fundamentals](#docker-fundamentals)
- [Installation](#installation)
- [Post-Installation Setup](#post-installation-setup)
- [Essential Docker Commands](#essential-docker-commands)
- [Working with Containers](#working-with-containers)
- [Port Mapping and Networking](#port-mapping-and-networking)
- [Overriding Container Defaults](#overriding-container-defaults)
- [Publishing and Exposing Ports](#publishing-and-exposing-ports)

---

## Docker Fundamentals

### Core Concepts

**Container vs VM:**
- **VM:** Entire OS with kernel, drivers, programs (heavyweight)
- **Container:** Isolated process with required files (lightweight)
  - Self-contained
  - Isolated filesystem
  - Independent and portable

**Container Image:**
- Standardized package with files, binaries, libraries, configs
- Immutable and layered
- Template for creating containers

**Registry:**
- Centralized storage for container images
- Can be public (Docker Hub) or private
- Examples: Docker Hub, AWS ECR, Google GCR, Azure ACR

### Docker Components

**Docker Engine:**
- Core technology powering Docker containers
- Command-line focused (for DevOps)

**Docker Desktop:**
- User-friendly GUI for container management
- Includes Docker Engine
- Can run simultaneously with Docker Engine (may cause conflicts)

⚠️ **Recommendation:** Use commands instead of GUI for DevOps work.

---

## Installation

### Stop Existing Docker Services

```bash
# Stop Docker Engine services
sudo systemctl stop docker docker.socket containerd

# Prevent auto-start on boot
sudo systemctl disable docker docker.socket containerd
```

### Official Installation (Recommended for Production)

#### 1. Remove Conflicting Packages

```bash
# Uninstall unofficial Docker packages
sudo apt remove docker.io docker-compose docker-compose-v2 \
    docker-doc podman-docker containerd runc
```

#### 2. Set Up Docker's APT Repository

```bash
# Add Docker's official GPG key
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
    -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add repository to APT sources
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "\${UBUNTU_CODENAME:-\$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

#### 3. Install Docker Packages

```bash
sudo apt install docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin
```

#### 4. Start and Verify Docker

```bash
# Start Docker service
sudo systemctl start docker

# Check status
sudo systemctl status docker

# Verify installation
sudo docker run hello-world
```

### Quick Installation (Learning/Development Only)

⚠️ **Not recommended for production**

```bash
# Ubuntu/Debian
sudo apt install docker.io

# Check version
docker --version
```

### Starting Docker Daemon

```bash
# Using systemctl (preferred)
sudo systemctl start docker

# Manual start (if systemctl unavailable)
sudo dockerd &
```

**Prerequisites for manual start:**
- containerd installed and running
- iptables
- cgroup support

---

## Post-Installation Setup

### Docker Group Configuration (Recommended)

**Run Docker without sudo:**

```bash
# Create Docker group
sudo groupadd docker

# Add current user to Docker group
sudo usermod -aG docker $USER

# Activate changes
newgrp docker

# Verify
docker run hello-world  # Should work without sudo
```

### Set Permissions

```bash
# Change ownership of Docker directory
sudo chown "$USER":"$USER" /home/"$USER"/.docker -R

# Set proper permissions
sudo chmod g+rwx "$HOME/.docker" -R
```

### Auto-Start on Boot

```bash
# Enable services to start on boot
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Disable auto-start
sudo systemctl disable docker.service
sudo systemctl disable containerd.service
```

---

## Essential Docker Commands

### Image Management

```bash
# Pull image from registry
docker pull <image-name>
docker pull nginx
docker pull ubuntu:22.04

# List images
docker images
docker image ls

# Remove image
docker rmi <image-name>
docker rmi nginx

# Remove dangling images
docker image prune

# Remove all unused images
docker image prune -a

# View image history
docker image history <image-name>
```

### Container Lifecycle

```bash
# Run container
docker run <image-name>
docker run nginx

# Run interactively
docker run -it ubuntu bash
# -i: interactive
# -t: attach terminal

# Run in detached mode
docker run -d nginx

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List container IDs only
docker ps -aq
```

### Container Control

```bash
# Start stopped container
docker start <container-id>

# Stop running container
docker stop <container-id>

# Restart container
docker restart <container-id>

# Pause container
docker pause <container-id>

# Unpause container
docker unpause <container-id>

# Remove container
docker rm <container-id>

# Force remove running container
docker rm -f <container-id>

# Remove all stopped containers
docker rm $(docker ps -aq)
docker container prune
```

### Interacting with Containers

```bash
# Execute command in running container
docker exec <container-id> <command>
docker exec mycontainer ls -la

# Open interactive terminal
docker exec -it <container-id> bash
docker exec -it <container-id> sh

# View container logs
docker logs <container-id>
docker logs -f <container-id>  # Follow logs
docker logs --tail 100 <container-id>  # Last 100 lines

# Inspect container
docker inspect <container-id>

# View container stats
docker stats
docker stats <container-id>

# View processes in container
docker top <container-id>
```

---

## Working with Containers

### Dangling Images

**What are dangling images?**
- Images with `<none>:<none>` tags
- Intermediate build layers or outdated images
- Not used by any container

```bash
# Identify dangling images
docker images -f "dangling=true"

# Example output:
# REPOSITORY   TAG      IMAGE ID      CREATED        SIZE
# <none>       <none>   abc12345defg  10 mins ago    300MB

# Remove dangling images
docker image prune

# Remove all unused images
docker image prune -a
```

### Container Logs

**Important:** Docker only captures logs from the main process.

```bash
# Python example
CMD ["python", "app.py"]
# ✅ print() statements appear in docker logs

# File logging
logging.basicConfig(filename="/var/log/app.log")
# ❌ docker logs shows nothing

# View logs
docker logs <container-name>
docker logs -f <container-name>  # Follow
docker logs --since 1h <container-name>  # Last hour
```

### Building Images

```bash
# Build from Dockerfile
docker build -t <image-name>:<tag> <directory>

# Examples
docker build -t myapp:v1 .
docker build -t myapp:latest /path/to/dockerfile

# Build with custom Dockerfile name
docker build -t myapp:v1 -f Dockerfile.prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp .
```

---

## Port Mapping and Networking

### Understanding Port Mapping

**The Problem:**
- Containers are isolated
- Services run on container's internal ports
- Cannot access directly from host

**Example:**
```bash
# Run nginx (listens on port 80 inside container)
docker run nginx

# Try accessing localhost:80
# ❌ Won't work - browser connects to host port 80, not container
```

**The Solution: Port Mapping**

```bash
# Map host port to container port
docker run -p <HOST-PORT>:<CONTAINER-PORT> <image>

# Examples
docker run -p 8080:80 nginx
# Access nginx at localhost:8080

docker run -p 3000:3000 node-app
# Access app at localhost:3000
```

### Port Mapping Examples

```bash
# Web server
docker run -d -p 8000:80 nginx
# Access: http://localhost:8000

# Database (PostgreSQL)
docker run -d -p 5432:5432 postgres
# Connect: localhost:5432

# Multiple containers, same image, different ports
docker run -d -p 5432:5432 --name pg1 postgres
docker run -d -p 5433:5432 --name pg2 postgres
# pg1: localhost:5432
# pg2: localhost:5433
```

---

## Overriding Container Defaults

Containers start with defaults defined in image (CMD, ENTRYPOINT, ports, env vars). Docker allows overriding at runtime.

### 1. Override Network Ports

**Multiple instances without conflicts:**

```bash
# Default
docker run -d -p 5432:5432 postgres

# Second instance (different host port)
docker run -d -p 5433:5432 postgres

# Third instance
docker run -d -p 5434:5432 postgres
```

### 2. Set Environment Variables

```bash
# Single variable
docker run -e POSTGRES_PASSWORD=secret postgres

# Multiple variables
docker run -e DB_HOST=localhost \
           -e DB_PORT=5432 \
           -e DB_NAME=mydb \
           postgres

# From file
docker run --env-file .env postgres
```

**Example `.env` file:**
```
POSTGRES_PASSWORD=secretpass
POSTGRES_USER=admin
POSTGRES_DB=mydatabase
```

### 3. Restrict Resource Usage

```bash
# Limit memory
docker run --memory="512m" nginx

# Limit CPU
docker run --cpus="0.5" nginx

# Both
docker run --memory="1g" --cpus="2" myapp

# Monitor usage
docker stats
```

### 4. Custom Docker Networks

```bash
# Create custom network
docker network create mynetwork

# Run container on custom network
docker run --network mynetwork nginx

# Why use custom networks?
# - Name-based DNS resolution
# - Better isolation
# - Easier service discovery
```

### 5. Override CMD and ENTRYPOINT

**In docker-compose.yaml:**
```yaml
services:
  postgres:
    image: postgres
    entrypoint: ["docker-entrypoint.sh", "postgres"]
    command: ["-h", "localhost", "-p", "5432"]
```

**In docker run:**
```bash
# Override CMD
docker run ubuntu echo "Hello World"

# Override ENTRYPOINT
docker run --entrypoint /bin/sh ubuntu
```

---

## Publishing and Exposing Ports

### Understanding the Problem

**Container isolation:**
- Each component runs in isolated environment
- React frontend, Python API, database in separate containers
- Cannot access directly from host

**Port publishing:**
- Creates forwarding rule from host to container
- Makes container services accessible

### Basic Port Publishing

```bash
# Publish specific port
docker run -d -p <HOST-PORT>:<CONTAINER-PORT> <image>

# Examples
docker run -d -p 8080:80 nginx
docker run -d -p 3000:3000 node-app
docker run -d -p 5432:5432 postgres
```

⚠️ **Security Note:** Published ports are accessible to all network interfaces. Don't publish databases in production!

### Ephemeral Ports

**Let Docker choose the host port:**

```bash
# Docker assigns random available port
docker run -d -p 80 nginx

# View assigned port
docker ps
# PORTS
# 0.0.0.0:32768->80/tcp
```

**Use case:** Avoid port conflicts in development/testing

### Publishing All Exposed Ports

```bash
# In Dockerfile
EXPOSE 80
EXPOSE 443

# Publish all exposed ports to ephemeral ports
docker run -d -P nginx

# Docker automatically maps:
# 80 -> random port (e.g., 32768)
# 443 -> random port (e.g., 32769)
```

### Dockerfile vs Runtime

**Dockerfile EXPOSE instruction:**
```dockerfile
EXPOSE 8080
```
- Documents which port the app uses
- Does NOT actually publish the port
- Ports must be published at runtime with `-p` or `-P`

---

## Dockerfile Instructions Reference

### RUN vs CMD vs ENTRYPOINT

**RUN:**
- Executes during **image build**
- Creates new layer
- Used for installing packages, setting up files

```dockerfile
RUN apt update && apt install -y curl
RUN npm install
```

**CMD:**
- Provides **default command** when container starts
- Can be overridden at runtime
- Only last CMD is used

```dockerfile
CMD ["python", "app.py"]

# Override:
docker run myimage python other_script.py
```

**ENTRYPOINT:**
- Defines **main executable**
- Not easily overridden (requires `--entrypoint` flag)
- Often used with CMD for default arguments

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]

# Runs: python app.py
# Override args: docker run myimage other.py
# Runs: python other.py
```

### Combined Usage

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]

# Container runs: docker-entrypoint.sh nginx -g daemon off;
# Can override CMD: docker run myimage nginx -g daemon on;
```

---

## Docker in Docker (DinD)

### Using Host Docker Socket (Recommended)

```bash
# Mount host Docker socket
docker run -v /var/run/docker.sock:/var/run/docker.sock -it ubuntu

# Container can use host's Docker daemon
# No need to install Docker inside container
```

### True Docker in Docker

```bash
# Official DinD image
docker run --privileged --name dind-test -d docker:dind

# Access with:
docker exec -it dind-test sh
```

⚠️ **Security Warning:** `--privileged` gives container extensive host access.

---

## Best Practices

### Image Management

1. **Use specific tags:**
   ```bash
   docker pull nginx:1.25  # Good
   docker pull nginx       # Bad (uses :latest)
   ```

2. **Clean up regularly:**
   ```bash
   docker system prune -a  # Remove unused resources
   ```

3. **Multi-stage builds:**
   - Keep images small
   - Separate build and runtime

### Container Management

1. **Name containers:**
   ```bash
   docker run --name myapp nginx
   ```

2. **Use restart policies:**
   ```bash
   docker run --restart unless-stopped nginx
   ```

3. **Resource limits:**
   ```bash
   docker run --memory="1g" --cpus="2" myapp
   ```

### Security

1. **Don't run as root:**
   ```dockerfile
   USER appuser
   ```

2. **Use private registries** for sensitive images

3. **Scan images:**
   ```bash
   docker scan myimage
   ```

4. **Minimal base images:**
   ```dockerfile
   FROM alpine:3.18
   # Smaller attack surface
   ```

---

## Quick Reference

### Essential Commands

```bash
# Images
docker pull <image>
docker images
docker rmi <image>

# Containers
docker run <image>
docker ps
docker stop <container>
docker rm <container>

# Execution
docker exec -it <container> bash
docker logs <container>

# Build
docker build -t <name> .

# Network
docker network ls
docker network create <name>

# Volume
docker volume ls
docker volume create <name>

# Cleanup
docker system prune -a
```

### Common Flags

```
-d          Detached mode
-it         Interactive with terminal
-p          Port mapping
-v          Volume mount
-e          Environment variable
--name      Container name
--network   Custom network
--rm        Auto-remove on exit
```

---

This comprehensive guide covers Docker installation, core concepts, and essential commands for containerization workflows.