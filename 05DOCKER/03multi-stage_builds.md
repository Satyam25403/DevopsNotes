# Docker Multi-Stage Builds

Comprehensive guide to multi-stage builds for creating optimized, production-ready Docker images.

## Table of Contents
- [Understanding Multi-Stage Builds](#understanding-multi-stage-builds)
- [Why Multi-Stage Builds](#why-multi-stage-builds)
- [Basic Multi-Stage Build](#basic-multi-stage-build)
- [Language-Specific Examples](#language-specific-examples)
- [Advanced Techniques](#advanced-techniques)
- [Best Practices](#best-practices)

---

## Understanding Multi-Stage Builds

### The Problem with Traditional Builds

**Traditional single-stage build:**

```dockerfile
FROM node:18

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source
COPY . .

# Build application
RUN npm run build

# Start app
CMD ["node", "dist/server.js"]
```

**Problems:**
- ❌ Final image contains build tools (npm, compilers)
- ❌ Final image contains source code
- ❌ Final image contains dev dependencies
- ❌ Large image size (500MB+)
- ❌ Larger attack surface

### The Multi-Stage Solution

**Multi-stage build separates:**
1. **Build environment** (heavy, has build tools)
2. **Runtime environment** (lightweight, production-ready)

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

**Benefits:**
- ✅ Small final image (~50MB vs 500MB)
- ✅ No build tools in production
- ✅ No source code in production
- ✅ Only runtime dependencies
- ✅ Reduced attack surface

---

## Why Multi-Stage Builds

### Separation of Concerns

```
┌─────────────────────────────┐
│   Build Stage (Large)       │
│  - Build tools              │
│  - Compilers                │
│  - Source code              │
│  - Dev dependencies         │
│  - Build artifacts          │
└─────────────────────────────┘
           ↓
    COPY only artifacts
           ↓
┌─────────────────────────────┐
│  Runtime Stage (Small)      │
│  - Runtime only             │
│  - Compiled binaries        │
│  - Prod dependencies        │
│  - No build tools           │
└─────────────────────────────┘
```

### For Interpreted Languages

**JavaScript, Python, Ruby:**

Multi-stage builds help:
- Build and minify code in one stage
- Copy production files to runtime image
- Exclude dev dependencies
- Remove source maps, tests, docs

**Example (React):**
```dockerfile
# Build stage: Compile React app
FROM node:18 AS builder
RUN npm run build  # Creates optimized /dist

# Runtime stage: Serve with nginx
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### For Compiled Languages

**C, Go, Rust, Java:**

Multi-stage builds help:
- Compile in one stage with full compiler
- Copy compiled binary to minimal runtime
- No compiler in final image

**Example (Go):**
```dockerfile
# Build stage: Compile Go binary
FROM golang:1.21 AS builder
RUN go build -o app

# Runtime stage: Minimal image with just binary
FROM scratch
COPY --from=builder /app/app /app
CMD ["/app"]
```

---

## Basic Multi-Stage Build

### Anatomy of Multi-Stage Build

```dockerfile
# ============ Stage 1: Build ============
FROM <builder-image> AS <stage-name>
WORKDIR /app

# Install build tools
RUN install-build-tools

# Copy source
COPY . .

# Build
RUN build-command

# ============ Stage 2: Runtime ============
FROM <runtime-image> AS <stage-name>
WORKDIR /app

# Copy ONLY built artifacts
COPY --from=<previous-stage> /path/in/build /path/in/runtime

# Runtime configuration
CMD ["run-command"]
```

### Key Concepts

**1. Stage naming:**
```dockerfile
FROM node:18 AS build-stage
FROM node:18-alpine AS runtime-stage
```

**2. Copying between stages:**
```dockerfile
COPY --from=build-stage /app/dist ./dist
```

**3. Only final stage matters:**
- Previous stages discarded
- Only last `FROM` creates final image

---

## Language-Specific Examples

### Node.js / TypeScript

```dockerfile
# ============ Build Stage ============
FROM node:18 AS build

WORKDIR /app

# Copy dependency files (better caching)
COPY package*.json ./

# Install ALL dependencies (including dev)
RUN npm install

# Copy source code
COPY . .

# Build TypeScript to JavaScript
RUN npm run build

# ============ Runtime Stage ============
FROM node:18-slim AS runtime

WORKDIR /app

# Copy built files
COPY --from=build /app/dist ./dist

# Copy package files
COPY --from=build /app/package*.json ./

# Install ONLY production dependencies
RUN npm install --omit=dev

# Expose port
EXPOSE 3000

# Start app
CMD ["node", "dist/index.js"]
```

**Size comparison:**
- Single stage: ~900MB
- Multi-stage: ~200MB

### React / Frontend

```dockerfile
# ============ Build Stage ============
FROM node:18 AS build

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source and build
COPY . .
RUN npm run build
# Creates optimized production build in /app/build

# ============ Runtime Stage ============
FROM nginx:alpine AS runtime

# Copy built static files to nginx
COPY --from=build /app/build /usr/share/nginx/html

# Copy custom nginx config (optional)
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Benefits:**
- Build stage: Full Node environment
- Runtime: Just nginx serving static files
- Size: ~25MB (vs ~900MB with Node)

### Python

```dockerfile
# ============ Build Stage ============
FROM python:3.11 AS build

WORKDIR /app

# Install build dependencies
RUN apt-get update && \
    apt-get install -y gcc

# Copy requirements
COPY requirements.txt .

# Install Python packages
RUN pip install --user -r requirements.txt

# ============ Runtime Stage ============
FROM python:3.11-slim AS runtime

WORKDIR /app

# Copy installed packages from build stage
COPY --from=build /root/.local /root/.local

# Copy application code
COPY . .

# Add local packages to PATH
ENV PATH=/root/.local/bin:$PATH

# Run application
CMD ["python", "app.py"]
```

### Go (Compiled Language)

```dockerfile
# ============ Build Stage ============
FROM golang:1.21 AS build

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server

# ============ Runtime Stage ============
FROM scratch

# Copy ONLY the binary
COPY --from=build /app/server /server

# Copy SSL certificates (if needed)
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Expose port
EXPOSE 8080

# Run binary
CMD ["/server"]
```

**Benefits:**
- Build stage: Full Go toolchain (~800MB)
- Runtime: Scratch (minimal) + binary (~10MB)
- Massive size reduction!

### Java / Maven

```dockerfile
# ============ Build Stage ============
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app

# Copy pom.xml (dependency definition)
COPY pom.xml .

# Download dependencies (cached)
RUN mvn dependency:go-offline

# Copy source code
COPY src ./src

# Build JAR
RUN mvn package -DskipTests

# ============ Runtime Stage ============
FROM eclipse-temurin:17-jre-alpine AS runtime

WORKDIR /app

# Copy JAR from build stage
COPY --from=build /app/target/*.jar app.jar

# Expose port
EXPOSE 8080

# Run application
CMD ["java", "-jar", "app.jar"]
```

**Benefits:**
- Build stage: Maven + JDK (~700MB)
- Runtime: JRE only (~150MB)

---

## Advanced Techniques

### Multiple Build Stages

```dockerfile
# ============ Dependencies Stage ============
FROM node:18 AS dependencies

WORKDIR /app
COPY package*.json ./
RUN npm install

# ============ Build Stage ============
FROM node:18 AS build

WORKDIR /app

# Copy deps from previous stage
COPY --from=dependencies /app/node_modules ./node_modules

# Copy source
COPY . .

# Build
RUN npm run build

# ============ Test Stage (optional) ============
FROM node:18 AS test

WORKDIR /app

COPY --from=dependencies /app/node_modules ./node_modules
COPY . .

# Run tests
RUN npm test

# ============ Runtime Stage ============
FROM node:18-alpine AS runtime

WORKDIR /app

COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./

RUN npm install --omit=dev

CMD ["node", "dist/server.js"]
```

### Building Specific Stage

```bash
# Build only test stage
docker build --target test -t myapp:test .

# Build only build stage
docker build --target build -t myapp:build .

# Build final runtime stage (default)
docker build -t myapp:latest .
```

### Copy from External Image

```dockerfile
FROM node:18 AS build

# Copy from another image
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf

# Rest of build...
```

### Use Build Arguments

```dockerfile
# ============ Build Stage ============
FROM node:18 AS build

ARG BUILD_ENV=production

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .

# Use build arg
RUN npm run build:${BUILD_ENV}

# ============ Runtime Stage ============
FROM node:18-alpine AS runtime

# Inherit build arg
ARG BUILD_ENV=production

WORKDIR /app
COPY --from=build /app/dist ./dist

CMD ["node", "dist/server.js"]
```

```bash
# Build for different environments
docker build --build-arg BUILD_ENV=staging -t myapp:staging .
docker build --build-arg BUILD_ENV=production -t myapp:prod .
```

---

## Best Practices

### 1. Name Your Stages

```dockerfile
# ✅ Good - Named stages
FROM node:18 AS dependencies
FROM node:18 AS builder
FROM node:18-alpine AS runtime

# ❌ Bad - Unnamed stages
FROM node:18
FROM node:18
FROM node:18-alpine
```

### 2. Order Stages Logically

```dockerfile
# Dependencies → Build → Test → Runtime
FROM ... AS deps
FROM ... AS build
FROM ... AS test
FROM ... AS runtime
```

### 3. Use Appropriate Base Images

```dockerfile
# Build stage: Full image
FROM node:18 AS build

# Runtime stage: Slim/Alpine
FROM node:18-alpine AS runtime
```

### 4. Copy Only What's Needed

```dockerfile
# ❌ Bad - Copies everything
COPY --from=build /app /app

# ✅ Good - Copies only dist
COPY --from=build /app/dist ./dist
```

### 5. Leverage Build Cache

```dockerfile
# Copy deps first (cached unless changed)
COPY package*.json ./
RUN npm install

# Copy source last (changes frequently)
COPY . .
```

### 6. Security: Run as Non-Root

```dockerfile
FROM node:18-alpine AS runtime

# Create user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy files
COPY --from=build --chown=appuser:appgroup /app/dist ./dist

# Switch user
USER appuser

CMD ["node", "dist/server.js"]
```

### 7. Health Checks

```dockerfile
FROM node:18-alpine AS runtime

COPY --from=build /app/dist ./dist

HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

CMD ["node", "dist/server.js"]
```

---

## Complete Example: Production-Grade Multi-Stage Build

```dockerfile
# ============================================
# Stage 1: Dependencies
# ============================================
FROM node:18-alpine AS dependencies

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies
RUN npm ci

# ============================================
# Stage 2: Build
# ============================================
FROM node:18-alpine AS build

WORKDIR /app

# Copy dependencies from previous stage
COPY --from=dependencies /app/node_modules ./node_modules

# Copy source code
COPY . .

# Build application
RUN npm run build

# ============================================
# Stage 3: Test (Optional)
# ============================================
FROM node:18-alpine AS test

WORKDIR /app

# Copy dependencies and code
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .

# Run tests
RUN npm test

# ============================================
# Stage 4: Runtime
# ============================================
FROM node:18-alpine AS runtime

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy built application
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist

# Copy package files
COPY --chown=nodejs:nodejs package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node dist/healthcheck.js || exit 1

# Start application
CMD ["node", "dist/server.js"]
```

**Build and test:**
```bash
# Build and run tests
docker build --target test -t myapp:test .

# Build production image
docker build -t myapp:latest .

# Check image size
docker images myapp
```

---

## Size Comparison Examples

### Node.js Application

| Approach | Size | Contents |
|----------|------|----------|
| Single-stage (node:18) | ~900MB | Node + npm + source + deps + build tools |
| Single-stage (node:18-slim) | ~200MB | Node + source + deps |
| Multi-stage (18-alpine) | ~50MB | Node + dist + prod deps only |

### Go Application

| Approach | Size | Contents |
|----------|------|----------|
| Single-stage | ~800MB | Go toolchain + binary |
| Multi-stage (alpine) | ~15MB | Binary + minimal OS |
| Multi-stage (scratch) | ~10MB | Binary only |

### React Application

| Approach | Size | Contents |
|----------|------|----------|
| Single-stage (node) | ~900MB | Node + source + build |
| Multi-stage (nginx) | ~25MB | nginx + static files only |

---

## Quick Reference

### Multi-Stage Build Template

```dockerfile
# Build stage
FROM <builder-image> AS build
WORKDIR /app
COPY <dependencies> .
RUN <install-deps>
COPY . .
RUN <build-command>

# Runtime stage
FROM <runtime-image> AS runtime
WORKDIR /app
COPY --from=build <built-artifacts> .
CMD [<run-command>]
```

### Common Patterns

```bash
# Build specific stage
docker build --target <stage-name> .

# Build with args
docker build --build-arg ENV=prod .

# View size savings
docker images <image-name>
```

---

This guide covers multi-stage builds for creating optimized, secure, production-ready Docker images across different programming languages and frameworks.