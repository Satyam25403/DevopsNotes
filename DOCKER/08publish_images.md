# Publishing Docker Images to Docker Hub

Comprehensive guide to building, tagging, and publishing Docker images to Docker Hub registry.

## Table of Contents
- [Docker Hub Overview](#docker-hub-overview)
- [Image Naming Convention](#image-naming-convention)
- [Building Images](#building-images)
- [Tagging Images](#tagging-images)
- [Pushing to Docker Hub](#pushing-to-docker-hub)
- [Complete Workflow](#complete-workflow)
- [Best Practices](#best-practices)

---

## Docker Hub Overview

### What is Docker Hub?

**Docker Hub (hub.docker.com)** is a cloud-based registry for storing and sharing Docker images.

**Features:**
- Public and private repositories
- Automated builds
- Team collaboration
- Official images
- Free for public repos

### Getting Started

1. **Create account:** Visit hub.docker.com
2. **Sign up:** Free account for public repos
3. **Verify email**
4. **Optional:** Create access token for security

---

## Image Naming Convention

### Image Tag Format

```
[HOST[:PORT]/]PATH[:TAG]
```

**Components:**

| Component | Description | Example |
|-----------|-------------|---------|
| **HOST** | Registry hostname (optional) | `docker.io`, `ghcr.io` |
| **PORT** | Registry port (if non-standard) | `5000` |
| **PATH** | Image path | `username/imagename` |
| **TAG** | Version identifier | `v1.0`, `latest` |

### Common Examples

```bash
# Official nginx image
nginx
# Equivalent to: docker.io/library/nginx:latest

# User image
john/myapp
# Equivalent to: docker.io/john/myapp:latest

# With tag
john/myapp:v1.0
# Equivalent to: docker.io/john/myapp:v1.0

# Different registry (GitHub)
ghcr.io/username/myapp:latest

# Private registry
registry.company.com:5000/project/app:v2.0
```

### Path Structure

**For Docker Hub:**
```
[NAMESPACE]/REPOSITORY[:TAG]
```

**NAMESPACE:**
- User's or organization's name
- `library` for official images (implicit)

**REPOSITORY:**
- Image name

**TAG:**
- Version identifier
- `latest` is default

---

## Building Images

### 1. Create Dockerfile

```dockerfile
# Example: Node.js application
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
```

### 2. Build Image

**Basic build (generates SHA256 ID):**
```bash
docker build .
# Output: Successfully built abc123...
```

**Build with name:**
```bash
# Build with custom name
docker build -t myapp .

# Build with username/name format (for Docker Hub)
docker build -t username/myapp .

# Build with tag
docker build -t username/myapp:v1.0 .
```

**Build context:**
```bash
# Current directory
docker build -t myapp .

# Specific path
docker build -t myapp /path/to/dockerfile

# Custom Dockerfile name
docker build -t myapp -f Dockerfile.prod .

# Build from Git repo
docker build -t myapp https://github.com/user/repo.git
```

### 3. Verify Build

```bash
# List images
docker images

# Output:
# REPOSITORY        TAG       IMAGE ID       SIZE
# username/myapp    v1.0      abc123def      50MB
# username/myapp    latest    abc123def      50MB
```

---

## Tagging Images

### Understanding Tags

**Tags are aliases for image IDs:**
- Multiple tags can point to same image
- Used for versioning
- `latest` is convention (not automatically updated!)

### Tagging Existing Images

```bash
# Tag existing image
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
```

**Examples:**

```bash
# Add version tag to existing image
docker tag myapp username/myapp:v1.0

# Add multiple tags
docker tag myapp username/myapp:v1.0
docker tag myapp username/myapp:latest
docker tag myapp username/myapp:stable

# Tag for different registry
docker tag myapp ghcr.io/username/myapp:v1.0

# Tag for different user/organization
docker tag username/myapp anotheruser/myapp:v1.0
```

### Tag During Build

**Combine build and tag:**
```bash
# Build with Docker Hub format
docker build -t dockerhub_username/imagename:tag .

# Examples
docker build -t john/myapp:v1.0 .
docker build -t john/myapp:latest .
docker build -t john/myapp:dev .
```

### Semantic Versioning

```bash
# Version tags
docker build -t username/myapp:1.0.0 .
docker build -t username/myapp:1.0 .
docker build -t username/myapp:1 .
docker build -t username/myapp:latest .

# All point to same image
```

---

## Pushing to Docker Hub

### 1. Login to Docker Hub

**Interactive login:**
```bash
docker login

# Prompts for:
# Username:
# Password:
```

**Non-interactive login:**
```bash
docker login -u username

# Or with password (not recommended)
docker login -u username -p password
```

**Using access token (recommended):**
```bash
# Create access token at hub.docker.com
# Account Settings → Security → New Access Token

docker login -u username
# Password: [paste access token]
```

### 2. Push Image

```bash
# Push specific tag
docker push username/imagename:tag

# Examples
docker push john/myapp:v1.0
docker push john/myapp:latest
```

**Push all tags:**
```bash
# Push all tags of an image
docker push username/imagename --all-tags
```

### 3. Verify on Docker Hub

Visit hub.docker.com/r/username/imagename

---

## Complete Workflow

### End-to-End Example

**Step 1: Prepare project**
```bash
# Project structure
myapp/
├── Dockerfile
├── package.json
├── src/
└── README.md
```

**Step 2: Create Dockerfile**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

**Step 3: Build and tag**
```bash
# Build with Docker Hub naming
docker build -t john/myapp:v1.0 .
docker build -t john/myapp:latest .

# Verify
docker images | grep myapp
```

**Step 4: Test locally**
```bash
# Test the image
docker run -p 3000:3000 john/myapp:v1.0

# Verify it works
curl http://localhost:3000
```

**Step 5: Login**
```bash
docker login -u john
```

**Step 6: Push**
```bash
# Push both tags
docker push john/myapp:v1.0
docker push john/myapp:latest
```

**Step 7: Pull and run anywhere**
```bash
# On any machine with Docker
docker pull john/myapp:v1.0
docker run -d -p 3000:3000 john/myapp:v1.0
```

---

## Credential Management

### Storing Credentials

**Docker stores credentials in:**

**Linux:**
```bash
$HOME/.docker/config.json
```

**Windows:**
```
%USERPROFILE%/.docker/config.json
```

**Example config.json:**
```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "base64_encoded_credentials"
    }
  }
}
```

### Using Credential Helpers

**Configure credential store:**
```bash
# Install credential helper (Ubuntu)
sudo apt install pass gnupg2

# Configure Docker
docker-credential-pass

# Update config.json
{
  "credsStore": "pass"
}
```

### Logout

```bash
# Logout from Docker Hub
docker logout

# Logout from specific registry
docker logout registry.example.com
```

---

## Multi-Registry Workflow

### GitHub Container Registry

```bash
# Login to GitHub registry
echo $GITHUB_TOKEN | docker login ghcr.io -u username --password-stdin

# Build with GitHub registry format
docker build -t ghcr.io/username/myapp:v1.0 .

# Push to GitHub
docker push ghcr.io/username/myapp:v1.0
```

### Private Registry

```bash
# Login to private registry
docker login registry.company.com:5000

# Build with private registry format
docker build -t registry.company.com:5000/project/app:v1.0 .

# Push
docker push registry.company.com:5000/project/app:v1.0
```

### AWS ECR

```bash
# Get login token
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Build
docker build -t 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0 .

# Push
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
```

---

## Best Practices

### 1. Use Semantic Versioning

```bash
# Tag with multiple levels
docker tag myapp username/myapp:1.2.3
docker tag myapp username/myapp:1.2
docker tag myapp username/myapp:1
docker tag myapp username/myapp:latest

# Push all
docker push username/myapp --all-tags
```

### 2. Don't Rely on Latest

```bash
# ❌ Bad - Unpredictable
docker pull username/myapp:latest

# ✅ Good - Specific version
docker pull username/myapp:v1.2.3
```

### 3. Create Meaningful Tags

```bash
# Environment tags
docker build -t username/myapp:dev .
docker build -t username/myapp:staging .
docker build -t username/myapp:prod .

# Git commit tags
docker build -t username/myapp:$(git rev-parse --short HEAD) .

# Date tags
docker build -t username/myapp:$(date +%Y%m%d) .
```

### 4. Use Multi-Stage Builds

```dockerfile
# Small production images
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

### 5. Include Metadata

```dockerfile
# Add labels
LABEL maintainer="john@example.com"
LABEL version="1.0.0"
LABEL description="My awesome application"
```

### 6. Security

```bash
# Use access tokens (not passwords)
docker login -u username
# Paste access token when prompted

# Scan images for vulnerabilities
docker scan username/myapp:v1.0

# Sign images (Docker Content Trust)
export DOCKER_CONTENT_TRUST=1
docker push username/myapp:v1.0
```

---

## Automated Builds

### GitHub Actions

```yaml
# .github/workflows/docker-publish.yml
name: Docker Publish

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: |
            username/myapp:${{ github.ref_name }}
            username/myapp:latest
```

---

## Troubleshooting

### Common Issues

**"denied: requested access to the resource is denied"**
```bash
# Check you're logged in
docker login

# Check image name format
docker tag myapp username/myapp:v1.0
```

**"unauthorized: authentication required"**
```bash
# Login first
docker login -u username
```

**"name unknown: repository not found"**
```bash
# Repository must exist or be created on first push
# Check spelling and username
```

---

## Quick Reference

### Build and Push Workflow

```bash
# 1. Build
docker build -t username/imagename:tag .

# 2. Test
docker run username/imagename:tag

# 3. Login
docker login

# 4. Push
docker push username/imagename:tag
```

### Common Commands

```bash
# Build with tag
docker build -t user/app:v1 .

# Tag existing image
docker tag app user/app:v1

# Login
docker login -u username

# Push
docker push user/app:v1

# Pull
docker pull user/app:v1

# Logout
docker logout
```

---

This comprehensive guide covers building, tagging, and publishing Docker images to Docker Hub and other registries.