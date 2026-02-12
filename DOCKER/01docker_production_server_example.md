# Docker Production Development Example

Comprehensive guide to developing with Docker containers in a production-like environment.

## Table of Contents
- [Overview](#overview)
- [Benefits of Containerized Development](#benefits-of-containerized-development)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Container Networking](#container-networking)
- [Docker Compose for Development](#docker-compose-for-development)
- [Best Practices](#best-practices)

---

## Overview

Docker enables you to develop applications in containerized environments that closely mirror production, ensuring consistency across development, testing, and deployment.

### What You Can Achieve

Within moments, you can:

1. **Start a complete development project with zero installation effort**
   - Containerized environment provides everything needed
   - No need to install Node, MySQL, Python, or other dependencies on host
   - All you need: Docker Desktop/Engine + code editor

2. **Make changes and see them immediately**
   - Processes in containers watch and respond to file changes
   - Files are shared between host and container
   - Hot reload without rebuilding

3. **Share development environment easily**
   - Dockerfile + docker-compose.yaml = reproducible environment
   - Team members get identical setup
   - "Works on my machine" problems eliminated

---

## Benefits of Containerized Development

### Traditional Development Problems

❌ **Manual Setup:**
```bash
# Developer needs to install:
- Node.js (specific version)
- Python (specific version)
- MySQL
- Redis
- System dependencies
- Configure all services
```

❌ **Version Conflicts:**
- Project A needs Node 14, Project B needs Node 18
- Different Python versions for different projects
- Dependency hell

❌ **Environment Drift:**
- Development ≠ Testing ≠ Production
- "Works on my machine" syndrome

### Containerized Development Solutions

✅ **Instant Setup:**
```bash
# All developer needs:
git clone <repo>
docker compose up

# Everything configured and ready
```

✅ **Isolated Environments:**
- Each project in its own containers
- No version conflicts
- Clean separation

✅ **Consistency:**
- Development = Production
- Same OS, same packages, same versions
- Reproducible builds

---

## Getting Started

### Example: Full-Stack Application

**Project structure:**
```
my-app/
├── frontend/          # React app
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── backend/           # Node.js API
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── database/          # MySQL
└── docker-compose.yaml
```

### Frontend Dockerfile

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Start dev server
CMD ["npm", "start"]
```

### Backend Dockerfile

```dockerfile
# backend/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 5000

# Start dev server with hot reload
CMD ["npm", "run", "dev"]
```

### Docker Compose Setup

```yaml
# docker-compose.yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src
    environment:
      - REACT_APP_API_URL=http://backend:5000
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    volumes:
      - ./backend/src:/app/src
    environment:
      - DB_HOST=database
      - DB_PORT=3306
      - DB_USER=root
      - DB_PASSWORD=secret
    depends_on:
      - database

  database:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=myapp
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### Start Development Environment

```bash
# Start all services
docker compose up

# Start in detached mode
docker compose up -d

# View logs
docker compose logs -f

# Stop all services
docker compose down
```

---

## Development Workflow

### 1. Initial Setup

```bash
# Clone repository
git clone https://github.com/yourorg/project.git
cd project

# Start containers
docker compose up

# That's it! Everything is ready
```

### 2. Making Changes

**Edit files on host:**
```bash
# Edit frontend/src/App.js
# Changes automatically reflected in browser
# No rebuild needed!
```

**How it works:**
- Volumes mount host directories into containers
- File watchers in containers detect changes
- Hot reload triggers automatically

```yaml
volumes:
  - ./frontend/src:/app/src  # Host:Container mapping
```

### 3. Adding Dependencies

**Frontend:**
```bash
# Install new package
docker compose exec frontend npm install axios

# Or rebuild
docker compose up --build frontend
```

**Backend:**
```bash
# Install new package
docker compose exec backend npm install express-validator

# Or rebuild
docker compose up --build backend
```

### 4. Database Operations

```bash
# Access MySQL
docker compose exec database mysql -uroot -psecret myapp

# Run migrations
docker compose exec backend npm run migrate

# Seed data
docker compose exec backend npm run seed
```

### 5. Debugging

```bash
# View logs
docker compose logs backend

# Follow logs
docker compose logs -f frontend

# Execute commands
docker compose exec backend npm test

# Access shell
docker compose exec backend sh
```

---

## Container Networking

### The localhost Problem

**Why `localhost` doesn't work between containers:**

```javascript
// ❌ This FAILS in containerized frontend
fetch('http://localhost:5000/api/users')

// Why?
// Each container has its own network namespace
// localhost in frontend container = frontend container itself
// localhost in backend container = backend container itself
```

### Understanding Container Isolation

**Each container is like a separate machine:**

```
┌─────────────────┐
│  Frontend       │
│  localhost =    │
│  127.0.0.1 →    │ Points to itself
│  Container A    │
└─────────────────┘

┌─────────────────┐
│  Backend        │
│  localhost =    │
│  127.0.0.1 →    │ Points to itself
│  Container B    │
└─────────────────┘
```

### Solution: Docker DNS

**Docker provides automatic DNS resolution:**

```yaml
services:
  frontend:
    # Can access: http://backend:5000
  
  backend:
    # Can access: http://database:3306
  
  database:
    # No need to expose externally
```

**Frontend code:**
```javascript
// ✅ CORRECT - Use service name
fetch('http://backend:5000/api/users')

// Docker DNS resolves 'backend' to backend container's IP
```

**Backend code:**
```javascript
// ✅ CORRECT - Use service name
const connection = mysql.createConnection({
  host: 'database',  // Not 'localhost'
  port: 3306,
  user: 'root',
  password: 'secret'
});
```

### How Docker DNS Works

```yaml
services:
  frontend:
    networks:
      - app-network
  
  backend:
    networks:
      - app-network
  
  database:
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**Behind the scenes:**
1. Docker Compose creates custom network
2. All services join the network
3. Docker DNS maps service names to container IPs
4. Services can reach each other by name

### Environment Variables for URLs

**Best practice:**

```yaml
# docker-compose.yaml
services:
  frontend:
    environment:
      - REACT_APP_API_URL=http://backend:5000
      - REACT_APP_WS_URL=ws://backend:5000
```

**Frontend code:**
```javascript
// Use environment variable
const API_URL = process.env.REACT_APP_API_URL;

fetch(`${API_URL}/api/users`)
```

### Network Communication Example

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - BACKEND_URL=http://backend:5000
    networks:
      - app-network

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=database
      - DB_PORT=3306
      - REDIS_URL=redis://cache:6379
    networks:
      - app-network

  database:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=secret
    networks:
      - app-network

  cache:
    image: redis:alpine
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

**Communication flow:**
```
Frontend → backend:5000 → Database
                       → cache:6379
```

---

## Docker Compose for Development

### Dockerfile vs Compose File

**Dockerfile:**
- Instructions to build a **single image**
- Defines environment, dependencies, commands
- Portable and reusable

**Docker Compose:**
- Defines **multiple containers** (services)
- Orchestrates entire application stack
- Manages networking, volumes, dependencies

### Why Use Docker Compose?

**Single command for everything:**

```bash
# Instead of:
docker network create mynet
docker volume create db_data
docker run --network mynet -v db_data:/data mysql
docker run --network mynet -p 3000:3000 frontend
docker run --network mynet -p 5000:5000 backend

# Just:
docker compose up
```

**Benefits:**
- ✅ All services defined in one YAML file
- ✅ Automatic network creation
- ✅ Service dependency management
- ✅ Easy scaling
- ✅ Environment variable management

### Development Features

**Volume mounting for hot reload:**
```yaml
services:
  app:
    volumes:
      - ./src:/app/src        # Source code
      - ./public:/app/public  # Static files
      - /app/node_modules     # Don't overwrite
```

**Override for development:**
```yaml
# docker-compose.override.yaml
services:
  backend:
    command: npm run dev  # Use dev server
    volumes:
      - ./backend:/app    # Mount source
```

**Multiple compose files:**
```bash
# Base config
docker-compose.yaml

# Development overrides
docker-compose.override.yaml

# Production overrides
docker-compose.prod.yaml

# Use production config
docker compose -f docker-compose.yaml -f docker-compose.prod.yaml up
```

---

## Best Practices

### 1. Use .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.env
*.md
.vscode
.idea
```

### 2. Separate Dev and Prod

**Development:**
```dockerfile
FROM node:18

WORKDIR /app
COPY package*.json ./
RUN npm install  # All dependencies

COPY . .
CMD ["npm", "run", "dev"]
```

**Production:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production  # Prod dependencies only

COPY . .
CMD ["npm", "start"]
```

### 3. Health Checks

```yaml
services:
  backend:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
```

### 4. Resource Limits

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

### 5. Logging

```yaml
services:
  backend:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 6. Secrets Management

```yaml
# Don't commit .env files
# Use .env.example instead

# .env.example
DB_HOST=database
DB_PORT=3306
DB_USER=root
DB_PASSWORD=changeme

# .env (gitignored)
DB_HOST=database
DB_PORT=3306
DB_USER=root
DB_PASSWORD=actual_secret_password
```

---

## Complete Example

### Full-Stack App with Docker Compose

```yaml
# docker-compose.yaml
version: '3.8'

services:
  # React Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src
      - /app/node_modules
    environment:
      - REACT_APP_API_URL=http://localhost:5000
      - CHOKIDAR_USEPOLLING=true  # For file watching
    depends_on:
      - backend

  # Node.js Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
    volumes:
      - ./backend/src:/app/src
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DB_HOST=database
      - DB_PORT=3306
      - DB_NAME=myapp
      - DB_USER=root
      - DB_PASSWORD=secret
      - REDIS_URL=redis://cache:6379
    depends_on:
      - database
      - cache

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

  # Redis Cache
  cache:
    image: redis:alpine
    ports:
      - "6379:6379"

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - frontend
      - backend

volumes:
  db_data:
```

### Usage

```bash
# Start everything
docker compose up

# Start specific service
docker compose up frontend

# Rebuild and start
docker compose up --build

# Run in background
docker compose up -d

# View logs
docker compose logs -f backend

# Execute command
docker compose exec backend npm test

# Stop everything
docker compose down

# Stop and remove volumes
docker compose down -v
```

---

## Troubleshooting

### Common Issues

**Port already in use:**
```bash
# Change host port
ports:
  - "3001:3000"  # Use 3001 instead of 3000
```

**Changes not reflecting:**
```bash
# Ensure volumes are mounted
volumes:
  - ./src:/app/src

# Enable polling (Windows/Mac)
environment:
  - CHOKIDAR_USEPOLLING=true
```

**Container exits immediately:**
```bash
# Check logs
docker compose logs backend

# Common causes:
# - Syntax error in code
# - Missing dependencies
# - Wrong CMD/ENTRYPOINT
```

**Cannot connect to database:**
```bash
# Wait for database to be ready
depends_on:
  database:
    condition: service_healthy

healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
```

---

## Summary

Docker enables powerful containerized development workflows:

✅ **Zero installation effort** - Docker handles all dependencies
✅ **Instant changes** - Hot reload without rebuilds  
✅ **Consistent environments** - Dev = Prod
✅ **Easy sharing** - Team gets identical setup
✅ **Isolated projects** - No version conflicts

Once you start thinking with containers, you can create and share almost any development environment with your team.

---

This guide demonstrates production-quality containerized development workflows using Docker and Docker Compose.