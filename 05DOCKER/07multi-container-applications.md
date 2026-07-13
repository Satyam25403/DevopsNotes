# Multi-Container Docker Applications

Comprehensive guide to building, orchestrating, and deploying multi-container applications with Docker and Docker Compose.

## Table of Contents
- [Introduction](#introduction)
- [Why Multi-Container Applications](#why-multi-container-applications)
- [Application Architecture](#application-architecture)
- [Docker Compose for Multi-Container Apps](#docker-compose-for-multi-container-apps)
- [Service Dependencies](#service-dependencies)
- [Complete Examples](#complete-examples)
- [Best Practices](#best-practices)

---

## Introduction

### What is a Multi-Container Application?

A **multi-container application** consists of multiple interconnected services running in separate containers:
- Frontend (React, Vue, Angular)
- Backend/API (Node.js, Python, Java)
- Database (MySQL, PostgreSQL, MongoDB)
- Cache (Redis)
- Message Queue (RabbitMQ)

---

## Why Multi-Container Applications

### Benefits

**1. Separation of Concerns** - Each service has single responsibility  
**2. Scalability** - Scale services independently  
**3. Maintainability** - Easier to update individual services  
**4. Development Workflow** - Teams work on different services

---

## Application Architecture

### Three-Tier Architecture

```
Frontend (React/Vue)
       ↓
Backend/API (Node/Python)
       ↓
Database (PostgreSQL/MongoDB)
```

---

## Docker Compose for Multi-Container Apps

### Basic Setup

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=database
    depends_on:
      - database

  database:
    image: postgres:15
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

---

## Service Dependencies

### With Health Checks

```yaml
services:
  database:
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 5s

  backend:
    depends_on:
      database:
        condition: service_healthy
```

---

## Complete Examples

### MERN Stack

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://backend:5000

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://database:27017/myapp

  database:
    image: mongo:6
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

---

## Best Practices

1. Use health checks
2. Environment variables for configuration
3. Resource limits
4. Don't publish database ports
5. Use named volumes for data

---

This guide covers multi-container Docker applications for production deployments.