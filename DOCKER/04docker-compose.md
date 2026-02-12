# Docker Compose

Comprehensive guide to Docker Compose for multi-container application orchestration and development.

## Table of Contents
- [Introduction to Docker Compose](#introduction-to-docker-compose)
- [Container Networking in Compose](#container-networking-in-compose)
- [Docker Compose vs Dockerfile](#docker-compose-vs-dockerfile)
- [Docker Compose Basics](#docker-compose-basics)
- [Docker Compose Watch](#docker-compose-watch)
- [Complete Examples](#complete-examples)
- [Best Practices](#best-practices)

---

## Introduction to Docker Compose

### What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications using a single YAML file.

**Key benefits:**
- ✅ Define all services in one file
- ✅ Launch everything with one command
- ✅ Automatic network creation
- ✅ Service dependency management
- ✅ Easy environment variable management

### Why Use Docker Compose?

**Without Compose (manual approach):**
```bash
# Create network
docker network create mynetwork

# Create volume
docker volume create db_data

# Run database
docker run -d --name db --network mynetwork -v db_data:/var/lib/mysql mysql

# Run backend
docker run -d --name api --network mynetwork -p 5000:5000 backend

# Run frontend
docker run -d --name web --network mynetwork -p 3000:3000 frontend

# Lots of typing, error-prone!
```

**With Compose (simple approach):**
```bash
# Just one command
docker compose up

# Everything configured in docker-compose.yaml
```

---

## Container Networking in Compose

### The localhost Problem

**Understanding container isolation:**

Each Docker container has its own isolated network namespace. This means:
- `localhost` inside Container A = Container A itself
- `localhost` inside Container B = Container B itself
- They are NOT the same!

### Example: Frontend + Backend Communication

**Scenario:**
- Frontend container (React on port 3000)
- Backend container (Node.js on port 5000)

**❌ This FAILS:**
```javascript
// frontend/src/api.js
fetch('http://localhost:5000/api/users')

// Why it fails:
// Frontend container looks for port 5000 on ITSELF
// Backend is in a different container
```

**✅ This WORKS:**
```javascript
// frontend/src/api.js
fetch('http://backend:5000/api/users')

// Why it works:
// Docker DNS resolves 'backend' to backend container's IP
```

### How Docker DNS Works

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    # Can access: http://backend:5000
    
  backend:
    build: ./backend
    # Can access: http://database:3306
    
  database:
    image: mysql:8
    # Automatically accessible by name 'database'
```

**Behind the scenes:**
1. Docker Compose creates a custom network
2. All services join the network
3. Docker DNS maps service names to container IPs
4. Services reach each other using service names

**Network visualization:**
```
┌────────────────────────────────────┐
│  Docker Network: myapp_default     │
│                                    │
│  ┌──────────┐  ┌──────────┐      │
│  │ frontend │  │ backend  │       │
│  │ 172.x.x.2│  │ 172.x.x.3│       │
│  └──────────┘  └──────────┘       │
│       ↓              ↓              │
│  DNS: frontend   DNS: backend      │
└────────────────────────────────────┘
```

### Service Discovery Example

```yaml
version: '3.8'

services:
  frontend:
    image: react-app
    environment:
      # Use service name, not localhost
      - REACT_APP_API_URL=http://backend:5000
    depends_on:
      - backend

  backend:
    image: node-api
    environment:
      # Use service name for database
      - DB_HOST=database
      - DB_PORT=3306
      # Use service name for Redis
      - REDIS_URL=redis://cache:6379
    depends_on:
      - database
      - cache

  database:
    image: mysql:8
    # Accessible as 'database'

  cache:
    image: redis:alpine
    # Accessible as 'cache'
```

**Communication flow:**
```
User Browser
    ↓ http://localhost:3000
Frontend Container
    ↓ http://backend:5000/api
Backend Container
    ↓ mysql://database:3306
Database Container
```

---

## Docker Compose vs Dockerfile

### Understanding the Difference

**Dockerfile:**
- Builds a **single container image**
- Defines: base image, dependencies, code, commands
- Portable and reusable

**Docker Compose:**
- Orchestrates **multiple containers** (services)
- Defines: services, networks, volumes, dependencies
- Runs entire application stack

### When to Use What

**Use Dockerfile when:**
- Building a single service image
- Defining how to create a container
- Sharing images via registry

**Use Docker Compose when:**
- Running multiple related containers
- Defining entire application stack
- Managing development/testing environments

### Often Used Together

```yaml
# docker-compose.yaml
version: '3.8'

services:
  frontend:
    build: ./frontend        # Uses frontend/Dockerfile
    ports:
      - "3000:3000"

  backend:
    build: ./backend         # Uses backend/Dockerfile
    ports:
      - "5000:5000"

  database:
    image: mysql:8           # Uses pre-built image
```

**Project structure:**
```
myapp/
├── docker-compose.yaml      # Orchestration
├── frontend/
│   ├── Dockerfile           # Frontend image definition
│   └── src/
└── backend/
    ├── Dockerfile           # Backend image definition
    └── src/
```

---

## Docker Compose Basics

### Basic Compose File

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html

  database:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=myapp
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### Essential Commands

```bash
# Start all services
docker compose up

# Start in detached mode
docker compose up -d

# Stop all services
docker compose down

# View logs
docker compose logs
docker compose logs -f           # Follow logs
docker compose logs backend      # Specific service

# List running services
docker compose ps

# Execute command in service
docker compose exec web sh
docker compose exec database mysql -u root -p

# Restart service
docker compose restart backend

# Stop and remove containers, networks, volumes
docker compose down -v
```

### Service Configuration

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    
    image: myapp/backend:latest
    
    container_name: myapp_backend
    
    ports:
      - "5000:5000"
    
    environment:
      - NODE_ENV=development
      - DB_HOST=database
    
    env_file:
      - .env
    
    volumes:
      - ./backend/src:/app/src
      - /app/node_modules
    
    depends_on:
      - database
      - cache
    
    networks:
      - app-network
    
    restart: unless-stopped
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
```

### Networks

```yaml
version: '3.8'

services:
  frontend:
    networks:
      - frontend-network
  
  backend:
    networks:
      - frontend-network
      - backend-network
  
  database:
    networks:
      - backend-network

networks:
  frontend-network:
  backend-network:
```

**Network isolation:**
- Frontend can talk to Backend
- Backend can talk to Database
- Frontend CANNOT talk to Database directly

---

## Docker Compose Watch

### The Old Development Workflow 😒

**Traditional approach:**

```bash
# Make changes to code
# Stop containers
docker compose down

# Rebuild images
docker compose build

# Start containers
docker compose up

# Wait... wait... wait...
# Very slow and annoying!
```

**Alternative (bind mounts):**
```yaml
services:
  web:
    volumes:
      - .:/app          # Mount entire directory
```

**Problems with bind mounts:**
- ❌ Messy
- ❌ Can break node_modules or Python venv
- ❌ Permission issues
- ❌ Performance problems (especially Windows/Mac)

### The New Development Workflow 😍

**Docker Compose Watch** provides:
- ✅ File synchronization
- ✅ Automatic rebuilds when needed
- ✅ Hot reload without manual restarts
- ✅ No bind mount issues

### How Compose Watch Works

```yaml
services:
  web:
    build: .
    develop:
      watch:
        # Sync source code changes
        - action: sync
          path: ./src
          target: /app/src
        
        # Rebuild on dependency changes
        - action: rebuild
          path: package.json
```

**Run watch mode:**
```bash
docker compose watch
```

**What happens:**

| Change | Action |
|--------|--------|
| Edit `src/app.js` | Syncs into container (instant) |
| Change `package.json` | Rebuilds image (few seconds) |
| Change `Dockerfile` | Rebuilds image |
| No manual restart needed | 🎉 |

### Watch Actions

```yaml
develop:
  watch:
    # 1. sync - Copy files to container
    - action: sync
      path: ./src
      target: /app/src
      ignore:
        - node_modules/
    
    # 2. rebuild - Rebuild entire image
    - action: rebuild
      path: package.json
    
    # 3. sync+restart - Sync and restart container
    - action: sync+restart
      path: ./config
      target: /app/config
```

### Complete Example

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    develop:
      watch:
        # Hot reload for source changes
        - action: sync
          path: ./frontend/src
          target: /app/src
        
        # Rebuild when dependencies change
        - action: rebuild
          path: ./frontend/package.json

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    develop:
      watch:
        # Sync and restart for code changes
        - action: sync+restart
          path: ./backend/src
          target: /app/src
        
        # Rebuild for dependencies
        - action: rebuild
          path: ./backend/requirements.txt
```

### When to Use Watch

**✅ Use when:**
- Developing Node, Python, Java apps
- Want hot reload without bind mounts
- Hate permission and performance issues
- Using Docker Compose v2+

**❌ Don't use when:**
- Production environments
- Simple one-off containers
- Already using in-container watchers (nodemon, uvicorn)

---

## Complete Examples

### Full-Stack Application

```yaml
version: '3.8'

services:
  # React Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:5000
    depends_on:
      - backend
    networks:
      - app-network
    develop:
      watch:
        - action: sync
          path: ./frontend/src
          target: /app/src

  # Node.js Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=development
      - DB_HOST=database
      - DB_PORT=3306
      - DB_NAME=myapp
      - DB_USER=root
      - DB_PASSWORD=secret
      - REDIS_URL=redis://cache:6379
    depends_on:
      database:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - app-network
    develop:
      watch:
        - action: sync
          path: ./backend/src
          target: /app/src
        - action: rebuild
          path: ./backend/package.json

  # MySQL Database
  database:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=myapp
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  cache:
    image: redis:alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  db_data:
```

### Microservices Example

```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    build: ./gateway
    ports:
      - "80:80"
    environment:
      - USER_SERVICE_URL=http://user-service:8001
      - ORDER_SERVICE_URL=http://order-service:8002
      - PRODUCT_SERVICE_URL=http://product-service:8003
    networks:
      - frontend
      - backend

  # User Service
  user-service:
    build: ./services/user
    environment:
      - DB_HOST=user-db
    depends_on:
      - user-db
    networks:
      - backend

  user-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=users
      - POSTGRES_PASSWORD=secret
    volumes:
      - user_data:/var/lib/postgresql/data
    networks:
      - backend

  # Order Service
  order-service:
    build: ./services/order
    environment:
      - DB_HOST=order-db
    depends_on:
      - order-db
    networks:
      - backend

  order-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_PASSWORD=secret
    volumes:
      - order_data:/var/lib/postgresql/data
    networks:
      - backend

  # Product Service
  product-service:
    build: ./services/product
    environment:
      - DB_HOST=product-db
    depends_on:
      - product-db
    networks:
      - backend

  product-db:
    image: mongo:6
    volumes:
      - product_data:/data/db
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  user_data:
  order_data:
  product_data:
```

---

## Best Practices

### 1. Use Environment Variables

```yaml
# .env file
DB_PASSWORD=secret
API_KEY=abc123

# docker-compose.yaml
services:
  backend:
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
      - API_KEY=${API_KEY}
```

### 2. Health Checks

```yaml
services:
  database:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  backend:
    depends_on:
      database:
        condition: service_healthy
```

### 3. Resource Limits

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 256M
```

### 4. Logging

```yaml
services:
  backend:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 5. Multiple Compose Files

```bash
# Base configuration
docker-compose.yaml

# Development overrides
docker-compose.override.yaml

# Production
docker-compose.prod.yaml

# Use production
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up
```

### 6. Named Volumes

```yaml
# Use named volumes instead of bind mounts in production
volumes:
  db_data:
    driver: local
  cache_data:
    driver: local
```

---

## Quick Reference

### Common Commands

```bash
# Start
docker compose up
docker compose up -d
docker compose up --build

# Stop
docker compose down
docker compose down -v

# Logs
docker compose logs
docker compose logs -f service-name

# Execute
docker compose exec service-name sh

# Scale
docker compose up -d --scale backend=3

# Watch (dev mode)
docker compose watch
```

### Compose File Structure

```yaml
version: '3.8'

services:
  service-name:
    build: ./path
    image: image:tag
    container_name: name
    ports:
      - "host:container"
    environment:
      - VAR=value
    volumes:
      - ./host:/container
    depends_on:
      - other-service
    networks:
      - network-name
    restart: unless-stopped

networks:
  network-name:

volumes:
  volume-name:
```

---

This comprehensive guide covers Docker Compose for orchestrating multi-container applications with modern development workflows.