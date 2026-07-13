# Docker Networking

Comprehensive guide to Docker networking concepts, network types, and container communication.

## Table of Contents
- [Docker Networking Fundamentals](#docker-networking-fundamentals)
- [Network Types](#network-types)
- [Container Communication](#container-communication)
- [Network Management](#network-management)
- [DNS and Service Discovery](#dns-and-service-discovery)
- [Port Publishing](#port-publishing)
- [Best Practices](#best-practices)

---

## Docker Networking Fundamentals

### Understanding Container Networking

Docker networking enables:
- **Container-to-container communication**
- **Container-to-external communication**
- **Network isolation and security**
- **Service discovery via DNS**

### Key Concepts

**Network drivers:** Define how containers connect  
**DNS resolution:** Containers resolve each other by name  
**Port publishing:** Expose container ports to host

---

## Network Types

### 1. Bridge Network (Default)

Private internal network on host. Containers on same bridge can communicate.

```bash
# Create custom bridge
docker network create my-bridge

# Run containers
docker run -d --name backend --network my-bridge node-api
docker run -d --name db --network my-bridge postgres
```

### 2. Host Network

Container uses host's network directly (no isolation).

```bash
docker run -d --network host nginx
```

### 3. None Network

No networking - complete isolation.

```bash
docker run -d --network none alpine
```

### 4. Overlay Network

Multi-host networking for Docker Swarm.

```bash
docker network create --driver overlay my-overlay
```

---

## Container Communication

Containers on same network communicate by name:

```yaml
services:
  frontend:
    environment:
      - API_URL=http://backend:5000
  backend:
    environment:
      - DB_HOST=database
  database:
    image: postgres
```

---

## Network Management

```bash
docker network create <name>
docker network ls
docker network inspect <name>
docker network rm <name>
docker network prune
```

---

## DNS and Service Discovery

Containers automatically resolve service names:

```bash
docker run -d --name web --network app-net nginx
docker run -d --name api --network app-net node

# From web: ping api (works!)
```

---

## Port Publishing

```bash
# Publish port
docker run -p 8080:80 nginx

# Multiple ports
docker run -p 8080:80 -p 8443:443 nginx
```

---

## Best Practices

1. Use custom networks (not default bridge)
2. Don't publish database ports
3. Use network isolation
4. Use environment variables for service URLs

---

This guide covers Docker networking for container communication and service discovery.