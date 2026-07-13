# Docker Images, Layers, and Build Cache

Comprehensive guide to understanding Docker image architecture, layering system, and build cache optimization.

## Table of Contents
- [Dockerfile Basics](#dockerfile-basics)
- [Image Layers](#image-layers)
- [Build Cache](#build-cache)
- [Production-Ready Images](#production-ready-images)
- [Best Practices](#best-practices)

---

## Dockerfile Basics

### What is a Dockerfile?

A **Dockerfile** is a text-based document providing instructions to build container images.

### Core Instructions

```dockerfile
FROM <image>
# Specifies the base image
# Example: FROM ubuntu:22.04, FROM node:18-alpine

WORKDIR <path>
# Sets working directory for subsequent instructions
# Example: WORKDIR /app

COPY <host-path> <image-path>
# Copies files from build context into image
# Example: COPY package.json /app/

RUN <command>
# Executes command during build (creates new layer)
# Example: RUN npm install

ENV <name> <value>
# Sets environment variable (available at build and runtime)
# Example: ENV NODE_ENV=production

EXPOSE <port>
# Documents which port application listens on
# Example: EXPOSE 8080

USER <user-or-uid>
# Sets user for subsequent operations
# Example: USER appuser

CMD ["<command>", "<arg1>"]
# Default command when container starts
# Example: CMD ["node", "server.js"]
```

### Example Dockerfile

```dockerfile
# Base image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Set environment variable
ENV PORT=3000

# Expose port
EXPOSE 3000

# Run as non-root user
USER node

# Start application
CMD ["npm", "start"]
```

### Building Images

```bash
# Build image
docker build -t myapp:v1 .
docker build -t myapp:latest /path/to/dockerfile

# Build with custom Dockerfile name
docker build -t myapp:v1 -f Dockerfile.prod .

# Build with arguments
docker build --build-arg VERSION=1.0 -t myapp .
```

---

## Image Layers

### Understanding Layers

**Images are composed of layers:**
- Each instruction in Dockerfile creates a new layer
- Layers are **immutable** once created
- Layers stack to create the complete filesystem

### What is a Layer?

A layer is a **snapshot of filesystem changes:**
- What got added
- What got removed  
- What got modified

**Example:**
```dockerfile
FROM ubuntu:22.04        # Layer 1: Ubuntu base
RUN apt update           # Layer 2: Package index
RUN apt install nodejs   # Layer 3: Node.js
COPY app.js /app/        # Layer 4: Application code
```

### Layer Visualization

```
┌─────────────────────────┐
│  COPY app.js /app/      │ ← Layer 4
├─────────────────────────┤
│  RUN apt install nodejs │ ← Layer 3
├─────────────────────────┤
│  RUN apt update         │ ← Layer 2
├─────────────────────────┤
│  FROM ubuntu:22.04      │ ← Layer 1 (Base)
└─────────────────────────┘
```

### Theoretical Image Example

**Layer-by-layer breakdown:**

1. **Layer 1:** Adds basic commands and package manager (apt)
2. **Layer 2:** Installs Python runtime and pip
3. **Layer 3:** Copies requirements.txt
4. **Layer 4:** Installs application dependencies
5. **Layer 5:** Copies actual source code

### Layer Reuse

**Key benefit:** Layers can be reused between images

**Example:**
```
Image A:                    Image B:
├─ Python base (Layer 1-2)  ├─ Python base (Layer 1-2) ← Reused!
├─ App A deps (Layer 3-4)   ├─ App B deps (Layer 3-4)
└─ App A code (Layer 5)     └─ App B code (Layer 5)
```

**Benefits:**
- ✅ Faster builds (shared layers cached)
- ✅ Reduced storage (layers stored once)
- ✅ Faster distribution (shared layers not re-downloaded)

### Viewing Image Layers

```bash
# View image history (shows layers)
docker image history <image-name>

# Example output:
IMAGE          CREATED BY                                      SIZE
abc123         CMD ["npm" "start"]                             0B
def456         COPY . .                                        1.2MB
ghi789         RUN npm install                                 45MB
jkl012         COPY package*.json ./                           2KB
mno345         WORKDIR /app                                    0B
pqr678         FROM node:18-alpine                             40MB
```

### Layer Storage Location

Layers live on the host machine:
```bash
/var/lib/docker/overlay2/
```

**Not inside containers** - containers use layers to create filesystem.

---

## Build Cache

### Understanding Build Cache

Docker caches the result of each build instruction:
- Checks if instruction was executed before
- If identical, reuses cached layer
- If different, rebuilds from that point

**Benefits:**
- ✅ Faster builds
- ✅ More efficient development
- ✅ Reduced network usage

### How Cache Works

**First build:**
```dockerfile
FROM node:18-alpine      # Execute (no cache)
WORKDIR /app             # Execute (no cache)
COPY package*.json ./    # Execute (no cache)
RUN npm install          # Execute (no cache)
COPY . .                 # Execute (no cache)
```

**Second build (no changes):**
```dockerfile
FROM node:18-alpine      # ✅ Cached
WORKDIR /app             # ✅ Cached
COPY package*.json ./    # ✅ Cached
RUN npm install          # ✅ Cached
COPY . .                 # ✅ Cached
```

### Cache Invalidation

**When one layer changes, all subsequent layers rebuild:**

```dockerfile
FROM node:18-alpine      # ✅ Cached
WORKDIR /app             # ❌ Changed (new path)
COPY package*.json ./    # ❌ Rebuilt (dependency)
RUN npm install          # ❌ Rebuilt (dependency)
COPY . .                 # ❌ Rebuilt (dependency)
```

**Why subsequent layers rebuild:**
- They depend on filesystem state from earlier layers
- Must maintain consistency
- Cannot reuse if base changed

### Cache Invalidation Example

**Small change, big impact:**

```dockerfile
# Original
RUN apt install -y nodejs

# Changed
RUN apt install -y nodejs npm

# Result:
# ✅ All layers before: Cached
# ❌ This layer: Rebuilt
# ❌ All layers after: Rebuilt
```

### What Invalidates Cache?

✅ **Invalidates cache:**
- Changes in `RUN` instruction
- Changes in files copied by `COPY` or `ADD`
- Changes in Dockerfile instruction

❌ **Does NOT invalidate cache:**
- Container runtime changes
- `CMD` execution
- Container logs
- Environment at `docker run` time
- Volume mounts

**Only build-time inputs matter!**

### Bad vs Good Dockerfile (Cache Perspective)

**❌ Bad (cache-unfriendly):**
```dockerfile
FROM node:18-alpine
WORKDIR /app

# Copies everything first
COPY . .

# Installs dependencies
RUN npm install

# Any code change → npm install reruns 😭
```

**✅ Good (cache-friendly):**
```dockerfile
FROM node:18-alpine
WORKDIR /app

# Copy dependency files first
COPY package.json package-lock.json ./

# Install dependencies (cached unless package.json changes)
RUN npm install

# Copy source code last
COPY . .

# Code changes → npm install stays cached 😊
```

### Optimization Strategy

**Order instructions from least to most frequently changed:**

```dockerfile
# 1. Base image (rarely changes)
FROM node:18-alpine

# 2. System dependencies (rarely change)
RUN apk add --no-cache python3 make g++

# 3. Application dependencies (change occasionally)
COPY package*.json ./
RUN npm install

# 4. Application code (changes frequently)
COPY . .
```

### Practical Example

**Before optimization:**
```dockerfile
FROM node:18

WORKDIR /app

# ❌ Copies everything
COPY . .

# Dependencies reinstall on every code change
RUN npm install

EXPOSE 3000
CMD ["npm", "start"]
```

**After optimization:**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# ✅ Copy dependencies first
COPY package*.json ./

# Install (cached unless package.json changes)
RUN npm install

# Copy source code last
COPY src/ ./src/
COPY public/ ./public/

EXPOSE 3000
CMD ["npm", "start"]
```

**Build comparison:**

| Scenario | Bad Dockerfile | Good Dockerfile |
|----------|---------------|-----------------|
| First build | 2 min | 2 min |
| Code change | 2 min (reinstalls) | 5 sec (cached deps) |
| New dependency | 2 min | 2 min |

---

## Production-Ready Images

### Key Characteristics

Production images should have:

1. **Maximized build cache** - Fast rebuilds
2. **Small size** - Faster deployment
3. **Secure** - Minimal attack surface
4. **Non-root user** - Security best practice
5. **Multi-stage builds** - Separation of concerns

### Example: Production-Ready Node.js

```dockerfile
# Use Alpine for smaller size
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy dependency files (better caching)
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production

# Copy source code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Change ownership
RUN chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

# Start application
CMD ["node", "server.js"]
```

### Size Optimization

**Choose appropriate base image:**

```dockerfile
# ❌ Large (900MB+)
FROM node:18

# ✅ Medium (200MB)
FROM node:18-slim

# ✅ Small (40MB)
FROM node:18-alpine
```

**Multi-stage builds** (covered in detail in next section):

```dockerfile
# Build stage (large)
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Runtime stage (small)
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

### Security Best Practices

**1. Run as non-root:**
```dockerfile
USER nodejs
```

**2. Use specific versions:**
```dockerfile
# ❌ Bad
FROM node

# ✅ Good
FROM node:18.17.0-alpine3.18
```

**3. Minimize layers:**
```dockerfile
# ❌ Bad (3 layers)
RUN apt update
RUN apt install -y curl
RUN apt install -y git

# ✅ Good (1 layer)
RUN apt update && \
    apt install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

**4. Don't include secrets:**
```dockerfile
# ❌ NEVER do this
ENV API_KEY=secret123

# ✅ Use build args or runtime env vars
ARG API_KEY
RUN fetch_config.sh

# Or at runtime:
docker run -e API_KEY=secret123 myapp
```

---

## Best Practices

### 1. Order Instructions Strategically

```dockerfile
# Least frequently changed → Most frequently changed

FROM node:18-alpine          # 1. Base (rarely)
RUN apk add --no-cache git   # 2. System deps (rarely)
COPY package*.json ./        # 3. App deps (sometimes)
RUN npm install              # 4. Install deps (sometimes)
COPY . .                     # 5. Source code (often)
```

### 2. Use .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.vscode
.idea
*.log
coverage/
dist/
build/
```

### 3. Combine RUN Commands

```dockerfile
# ❌ Multiple layers
RUN apt update
RUN apt install -y curl
RUN apt install -y git
RUN rm -rf /var/lib/apt/lists/*

# ✅ Single layer
RUN apt update && \
    apt install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

### 4. Use Specific Tags

```dockerfile
# ❌ Unpredictable
FROM node:latest

# ✅ Predictable
FROM node:18.17.0-alpine3.18
```

### 5. Clean Up in Same Layer

```dockerfile
# ❌ Cleanup in separate layer (doesn't reduce size)
RUN apt install -y build-tools
RUN make build
RUN apt remove -y build-tools

# ✅ Cleanup in same layer
RUN apt install -y build-tools && \
    make build && \
    apt remove -y build-tools && \
    rm -rf /var/lib/apt/lists/*
```

### 6. Copy Only What's Needed

```dockerfile
# ❌ Copies everything
COPY . .

# ✅ Selective copying
COPY package*.json ./
COPY src/ ./src/
COPY public/ ./public/
# Excludes tests, docs, etc.
```

---

## Layer Best Practices Summary

### Cache Optimization

```dockerfile
# ✅ Perfect cache usage
FROM python:3.11-slim

# System dependencies (rarely change)
RUN apt update && apt install -y gcc

# Python dependencies (change occasionally)
COPY requirements.txt .
RUN pip install -r requirements.txt

# Application code (changes frequently)
COPY app/ ./app/
```

### Build Efficiency

**Check cache usage:**
```bash
# Build with cache
docker build -t myapp .

# Build without cache (force rebuild)
docker build --no-cache -t myapp .

# Build with inline cache
docker build --cache-from myapp:latest -t myapp:v2 .
```

---

## Quick Reference

### Dockerfile Instructions

| Instruction | Purpose | Layer Created |
|-------------|---------|---------------|
| `FROM` | Base image | Yes |
| `WORKDIR` | Set working dir | Yes |
| `COPY` | Copy files | Yes |
| `ADD` | Copy + extract | Yes |
| `RUN` | Execute command | Yes |
| `ENV` | Set env var | Yes |
| `EXPOSE` | Document port | No |
| `USER` | Set user | No |
| `CMD` | Default command | No |
| `ENTRYPOINT` | Main executable | No |

### Cache Behavior

| Change | Cache Status |
|--------|-------------|
| No changes | All cached |
| Dockerfile edit | From change onward: rebuilt |
| File in COPY changed | From COPY onward: rebuilt |
| New dependency | From dependency install: rebuilt |
| Code change (optimized) | Only code layer: rebuilt |

---

This guide covers Docker image architecture, layering, and build cache optimization for efficient, production-ready containerization.