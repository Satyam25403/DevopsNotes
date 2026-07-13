# Docker Storage, Volumes, and Data Persistence

Comprehensive guide to Docker storage mechanisms, volume management, and data persistence strategies.

## Table of Contents
- [Understanding Docker Storage](#understanding-docker-storage)
- [Storage Types Overview](#storage-types-overview)
- [Volume Mounts](#volume-mounts)
- [Bind Mounts](#bind-mounts)
- [tmpfs Mounts](#tmpfs-mounts)
- [Named Pipes](#named-pipes)
- [Data Persistence](#data-persistence)
- [Best Practices](#best-practices)

---

## Understanding Docker Storage

### The Problem

**Container filesystem is ephemeral:**
- Data written to container layer doesn't persist
- When container is destroyed, data is lost
- Difficult to share data between containers
- Writable layer is unique per container

**Why we need storage mounts:**
- ✅ Persist data beyond container lifecycle
- ✅ Share data between containers
- ✅ Share data between host and container
- ✅ Separate data from container

---

## Storage Types Overview

Docker supports four types of storage mounts:

| Type | Managed by Docker | Persistent | Use Case |
|------|-------------------|------------|----------|
| **Volume** | Yes | Yes | Databases, production |
| **Bind mount** | No | Yes | Development, configs |
| **tmpfs** | No | No | Secrets, cache |
| **Named pipe** | No | IPC only | Host ↔ container communication |

### Quick Comparison

```
┌─────────────────────────────────────────────┐
│  Docker Host                                │
│                                             │
│  ┌──────────────┐                          │
│  │  Container   │                          │
│  │              │                          │
│  │  /app/data ──┼──→ Volume (Docker)       │
│  │              │    /var/lib/docker/...   │
│  │  /app/src  ──┼──→ Bind Mount (Host)     │
│  │              │    /home/user/project/   │
│  │  /tmp/cache ─┼──→ tmpfs (RAM)           │
│  └──────────────┘                          │
└─────────────────────────────────────────────┘
```

---

## Volume Mounts

### What are Volumes?

**Docker-managed storage:**
- Created and managed by Docker
- Stored in Docker's internal location
- Don't care where on host it lives
- Best for production workloads

### Creating Volumes

```bash
# Create volume
docker volume create mydata

# List volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Remove volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune
```

### Using Volumes

```bash
# Run container with volume
docker run -d \
  --name mysql-db \
  -v mydata:/var/lib/mysql \
  mysql:8

# MySQL writes to /var/lib/mysql inside container
# Docker stores it in /var/lib/docker/volumes/mydata/_data
```

### Volume Location

```bash
# Docker stores volumes here:
/var/lib/docker/volumes/<volume-name>/_data

# Example:
/var/lib/docker/volumes/mydata/_data
```

### Volume Example: MySQL Database

```bash
# Create volume
docker volume create mysql_data

# Run MySQL with volume
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8

# Data persists even if container is removed
docker rm -f mysql

# Start new container with same volume
docker run -d \
  --name mysql-new \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8

# All data still there!
```

### Volume with Docker Compose

```yaml
version: '3.8'

services:
  database:
    image: mysql:8
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=secret

volumes:
  db_data:
    driver: local
```

### Use Cases for Volumes

**✅ Use volumes when:**
- Running databases (MySQL, PostgreSQL, MongoDB)
- Storing application state
- Production workloads
- Need data to persist
- Sharing data between containers

**Example scenarios:**
- Database data
- User uploads
- Application logs (persistent)
- Shared cache between containers

---

## Bind Mounts

### What are Bind Mounts?

**Direct link between host and container:**
- You control the exact location on host
- Map any host directory to container
- Both host and container can modify files
- Changes reflect immediately in both places

### Using Bind Mounts

```bash
# Syntax
docker run -v <HOST-PATH>:<CONTAINER-PATH> image

# Example
docker run -it \
  -v /home/user/app:/usr/share/nginx/html \
  -p 8080:80 \
  nginx
```

**Alternative syntax (recommended for production):**
```bash
docker run -it \
  --mount type=bind,source=/home/user/app,target=/usr/share/nginx/html \
  -p 8080:80 \
  nginx
```

### Bind Mount Example: Development

```bash
# Project structure
/home/user/myapp/
├── index.html
├── style.css
└── app.js

# Run nginx with bind mount
docker run -d \
  --name dev-server \
  -v /home/user/myapp:/usr/share/nginx/html \
  -p 8080:80 \
  nginx

# Edit files on host
vim /home/user/myapp/index.html

# Changes appear instantly in container
# Refresh browser → changes visible!
```

### Read-Only Bind Mounts

```bash
# Read-only mount (protect host files)
docker run -d \
  -v /home/user/config:/app/config:ro \
  myapp

# Container can read but NOT modify host files
```

### Bind Mount with Docker Compose

```yaml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      # Bind mount for development
      - ./html:/usr/share/nginx/html
      
      # Read-only config
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

### Use Cases for Bind Mounts

**✅ Use bind mounts when:**
- Local development (hot reload)
- Sharing config files
- Sharing source code
- Need to access files from both host and container
- Sharing logs with host

**⚠️ Not ideal for:**
- Production (host-dependent)
- Security-sensitive deployments
- Cross-platform compatibility

---

## tmpfs Mounts

### What is tmpfs?

**RAM-based temporary storage:**
- Stores files directly in host's RAM
- Data NOT written to disk
- Ephemeral (destroyed on container stop)
- Fast (no disk I/O)

### Using tmpfs

```bash
# Create tmpfs mount
docker run -it \
  --tmpfs /run/secrets \
  ubuntu

# Inside container
echo "SECRET=12345" > /run/secrets/key.txt

# Stop container → data gone!
```

### tmpfs with Options

```bash
# Limit size and set permissions
docker run -it \
  --tmpfs /cache:rw,size=64m,mode=1777 \
  ubuntu
```

### tmpfs Example: Sensitive Data

```bash
# Store temporary credentials in RAM
docker run -d \
  --name secure-app \
  --tmpfs /run/secrets:rw,size=10m \
  -e DB_PASSWORD_FILE=/run/secrets/db_pass \
  myapp

# Inside container
echo "super_secret_password" > /run/secrets/db_pass

# Process reads from tmpfs
# No disk traces!
```

### tmpfs Example: High-Speed Cache

```bash
# Redis with tmpfs cache
docker run -d \
  --name redis-cache \
  --tmpfs /cache:rw,size=256m \
  redis
```

### tmpfs with Docker Compose

```yaml
version: '3.8'

services:
  app:
    image: myapp
    tmpfs:
      - /run/secrets
      - /tmp/cache:size=100m
```

### Use Cases for tmpfs

**✅ Use tmpfs when:**
- Storing credentials temporarily
- Session data
- Temporary cache
- Reduce disk I/O
- Sensitive data (no disk traces)

**Examples:**
- API tokens (short-lived)
- Session cookies
- Build artifacts (temporary)
- In-memory cache

---

## Named Pipes

### What are Named Pipes?

**Inter-process communication (IPC):**
- Communication between host and container
- Commonly used on Windows
- Docker Engine API access
- Advanced use case

### Named Pipe Example: Docker API Access

**Windows:**
```powershell
# Access Docker API from container
docker run -it `
  -v \\.\pipe\docker_engine:\\.\pipe\docker_engine `
  docker

# Container can now control Docker on host
```

**Linux (FIFO pipe):**
```bash
# Create named pipe on host
mkfifo /tmp/my_pipe

# Run container with pipe
docker run -it \
  -v /tmp/my_pipe:/tmp/my_pipe \
  ubuntu

# Host writes
echo "hello" > /tmp/my_pipe

# Container reads
cat /tmp/my_pipe
# Output: hello
```

### Use Cases for Named Pipes

**✅ Use when:**
- Docker CLI inside container
- Debugging tools
- IPC with host services
- Docker-in-Docker scenarios

**⚠️ Security warning:**
Container can control host Docker!

---

## Data Persistence

### Understanding Data Loss

**Without volumes:**
```bash
# Run container
docker run -d --name db mysql:8

# Create data
docker exec db mysql -e "CREATE DATABASE myapp;"

# Remove container
docker rm -f db

# Data lost! 💥
```

**With volumes:**
```bash
# Create volume
docker volume create db_data

# Run container with volume
docker run -d \
  --name db \
  -v db_data:/var/lib/mysql \
  mysql:8

# Create data
docker exec db mysql -e "CREATE DATABASE myapp;"

# Remove container
docker rm -f db

# Start new container with same volume
docker run -d \
  --name db-new \
  -v db_data:/var/lib/mysql \
  mysql:8

# Data persists! ✅
```

### File Sharing Between Containers

**Share volume between multiple containers:**

```bash
# Create volume
docker volume create shared_data

# Container 1 (writer)
docker run -d \
  --name writer \
  -v shared_data:/data \
  alpine \
  sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# Container 2 (reader)
docker run -d \
  --name reader \
  -v shared_data:/data \
  alpine \
  sh -c "tail -f /data/log.txt"

# Both containers access same data
```

### Log Aggregation Example

```yaml
version: '3.8'

services:
  app1:
    image: myapp
    volumes:
      - logs:/var/log/app

  app2:
    image: myapp
    volumes:
      - logs:/var/log/app

  log-aggregator:
    image: fluent/fluentd
    volumes:
      - logs:/logs:ro  # Read-only

volumes:
  logs:
```

### Managing Volumes

```bash
# List all volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Remove volume (only if not attached)
docker volume rm mydata

# Remove all unused volumes
docker volume prune

# View volume contents (using temporary container)
docker run --rm -v mydata:/data alpine ls -la /data
```

### Backup and Restore

**Backup volume:**
```bash
# Backup to tar file
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/mydata-backup.tar.gz -C /data .
```

**Restore volume:**
```bash
# Create new volume
docker volume create mydata-restored

# Restore from tar file
docker run --rm \
  -v mydata-restored:/data \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /data
```

---

## Best Practices

### 1. Use Volumes for Production

```yaml
# ✅ Good - Named volume
services:
  database:
    image: postgres
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

### 2. Use Bind Mounts for Development

```yaml
# ✅ Good - Bind mount for source code
services:
  web:
    image: node:18
    volumes:
      - ./src:/app/src        # Hot reload
      - /app/node_modules     # Don't overwrite
```

### 3. Protect node_modules

```yaml
# Prevent host node_modules from overwriting container's
services:
  app:
    volumes:
      - ./src:/app/src
      - /app/node_modules    # Anonymous volume
```

### 4. Read-Only When Possible

```yaml
services:
  app:
    volumes:
      - ./config:/app/config:ro  # Read-only
```

### 5. Clean Up Regularly

```bash
# Remove unused volumes
docker volume prune

# Remove all volumes (careful!)
docker volume rm $(docker volume ls -q)
```

### 6. Name Your Volumes

```yaml
# ✅ Good - Named volume
volumes:
  db_data:
    name: myapp_db_data

# ❌ Bad - Anonymous volume
volumes:
  - /var/lib/mysql
```

### 7. Backup Important Data

```bash
# Regular backups
0 2 * * * docker run --rm -v db_data:/data -v /backup:/backup \
  alpine tar czf /backup/db-$(date +\%Y\%m\%d).tar.gz -C /data .
```

---

## Complete Examples

### Database with Persistent Storage

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=myapp
    volumes:
      # Data persistence
      - mysql_data:/var/lib/mysql
      
      # Init scripts
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      
      # Custom config
      - ./my.cnf:/etc/mysql/conf.d/my.cnf:ro
    ports:
      - "3306:3306"

volumes:
  mysql_data:
    driver: local
```

### Development Environment

```yaml
version: '3.8'

services:
  web:
    build: .
    volumes:
      # Source code (hot reload)
      - ./src:/app/src
      
      # Protect node_modules
      - /app/node_modules
      
      # Config files
      - ./config:/app/config:ro
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
```

### Multi-Container Data Sharing

```yaml
version: '3.8'

services:
  # Application server
  app:
    image: myapp
    volumes:
      - app_data:/app/data
      - logs:/var/log/app
  
  # Background worker
  worker:
    image: myworker
    volumes:
      - app_data:/app/data  # Shared data
      - logs:/var/log/worker
  
  # Log collector
  logstash:
    image: logstash
    volumes:
      - logs:/logs:ro  # Read-only logs

volumes:
  app_data:
  logs:
```

### Secure Secrets with tmpfs

```yaml
version: '3.8'

services:
  app:
    image: myapp
    tmpfs:
      # Temporary secrets in RAM
      - /run/secrets:size=10m
      
      # Cache in RAM
      - /tmp/cache:size=100m
    environment:
      - SECRET_FILE=/run/secrets/api_key
```

---

## Storage Decision Tree

```
Need to persist data?
│
├─ Yes
│  │
│  ├─ Production environment?
│  │  └─ Use VOLUMES
│  │
│  └─ Development environment?
│     └─ Use BIND MOUNTS
│
└─ No (temporary data)
   │
   ├─ Sensitive data?
   │  └─ Use TMPFS (RAM)
   │
   └─ IPC needed?
      └─ Use NAMED PIPES
```

---

## Quick Reference

### Volume Commands

```bash
docker volume create <name>
docker volume ls
docker volume inspect <name>
docker volume rm <name>
docker volume prune
```

### Mount Syntax

```bash
# Volume
-v volume-name:/container/path

# Bind mount
-v /host/path:/container/path

# Bind mount (read-only)
-v /host/path:/container/path:ro

# tmpfs
--tmpfs /container/path

# tmpfs with options
--tmpfs /container/path:rw,size=64m
```

### Compose Syntax

```yaml
volumes:
  # Named volume
  - db_data:/var/lib/mysql
  
  # Bind mount
  - ./src:/app/src
  
  # Read-only
  - ./config:/app/config:ro
  
  # Anonymous volume
  - /app/node_modules

tmpfs:
  - /run/secrets
  - /tmp/cache:size=100m
```

---

This comprehensive guide covers all Docker storage mechanisms for data persistence, sharing, and management across development and production environments.