# YAML — DevOps Reference Notes

> **YAML Ain't Markup Language** — a human-friendly data serialization standard used heavily across DevOps tooling: Kubernetes, Ansible, GitHub Actions, Docker Compose, Helm, ArgoCD, and more.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Data Types](#2-data-types)
3. [Collections — Mappings & Sequences](#3-collections--mappings--sequences)
4. [Multi-line Strings](#4-multi-line-strings)
5. [Anchors & Aliases (DRY in YAML)](#5-anchors--aliases-dry-in-yaml)
6. [Multiple Documents in One File](#6-multiple-documents-in-one-file)
7. [Comments & Style Best Practices](#7-comments--style-best-practices)
8. [Common Gotchas & Bugs](#8-common-gotchas--bugs)
9. [YAML in Kubernetes](#9-yaml-in-kubernetes)
10. [YAML in Ansible](#10-yaml-in-ansible)
11. [YAML in GitHub Actions](#11-yaml-in-github-actions)
12. [YAML in Docker Compose](#12-yaml-in-docker-compose)
13. [YAML in Helm Charts](#13-yaml-in-helm-charts)
14. [Validation & Debugging Tools](#14-validation--debugging-tools)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. Core Concepts

- YAML is a **superset of JSON** — all valid JSON is valid YAML
- Relies on **indentation** (spaces only, never tabs) to define structure
- **Case-sensitive** — `Name` and `name` are different keys
- File extensions: `.yaml` or `.yml` (both are equally valid)
- Encoding: UTF-8 (default), UTF-16, or UTF-32

### How YAML parsers read files

```
Raw YAML text
    → Lexer (tokenizes)
    → Parser (builds events)
    → Composer (builds nodes)
    → Constructor (builds native objects)
```

---

## 2. Data Types

YAML auto-detects types. You can override with explicit tags.

### Strings

```yaml
# Unquoted (implicit string)
name: priya

# Single-quoted (literal — no escape sequences)
message: 'It''s a DevOps world'    # double '' to escape single quote

# Double-quoted (supports escape sequences)
path: "C:\\Users\\priya\\Desktop"
tab_example: "column1\tcolumn2"
newline: "line1\nline2"

# Strings that look like other types MUST be quoted
version: "1.0"          # without quotes → float 1.0
enabled: "true"         # without quotes → boolean true
port: "8080"            # without quotes → integer 8080
null_str: "null"        # without quotes → null
date_str: "2024-01-01"  # without quotes → date object in some parsers
```

### Integers

```yaml
count: 42
negative: -7
octal: 0o17          # octal 17 = decimal 15
hex: 0xFF            # hex FF = decimal 255
binary: 0b1010       # binary 1010 = decimal 10
thousand: 1_000_000  # underscores as visual separator (YAML 1.2)
```

### Floats

```yaml
pi: 3.14159
scientific: 1.5e10
negative_float: -2.7
infinity: .inf
neg_infinity: -.inf
not_a_number: .nan
```

### Booleans

```yaml
# YAML 1.1 (older — used by many tools like PyYAML, Ansible)
# All of these are TRUE:
yes, Yes, YES, true, True, TRUE, on, On, ON

# All of these are FALSE:
no, No, NO, false, False, FALSE, off, Off, OFF

# YAML 1.2 (strict — used by newer tools)
# Only these are boolean:
true, false    # case-insensitive: True, False, TRUE, FALSE

# Best practice: always use lowercase true/false to be safe
enabled: true
disabled: false
```

### Null

```yaml
nothing: null
also_null: ~
empty_key:          # empty value = null
```

### Dates and Timestamps

```yaml
date: 2024-01-15                   # date (ISO 8601)
datetime: 2024-01-15T10:30:00Z     # UTC datetime
datetime_tz: 2024-01-15T10:30:00+05:30  # with timezone offset
```

### Explicit Type Tags

```yaml
# Override auto-detection using !! tags
force_string: !!str 123
force_int: !!int "42"
force_float: !!float "3"
force_bool: !!bool "yes"
force_null: !!null ""
binary_data: !!binary |
  SGVsbG8gV29ybGQ=
```

---

## 3. Collections — Mappings & Sequences

### Mappings (key-value pairs / dictionaries)

```yaml
# Block style (most readable)
server:
  host: localhost
  port: 8080
  debug: true

# Flow style (inline — good for short maps)
server: {host: localhost, port: 8080, debug: true}

# Nested mappings
database:
  primary:
    host: db1.example.com
    port: 5432
  replica:
    host: db2.example.com
    port: 5432
```

### Sequences (lists / arrays)

```yaml
# Block style with dash
fruits:
  - apple
  - banana
  - cherry

# Flow style (inline)
fruits: [apple, banana, cherry]

# Sequence of mappings (most common in K8s/Docker)
containers:
  - name: web
    image: nginx:latest
    port: 80
  - name: db
    image: postgres:15
    port: 5432

# Nested sequences
matrix:
  - [1, 2, 3]
  - [4, 5, 6]
  - [7, 8, 9]
```

### Mappings with sequence values

```yaml
# Common pattern: key → list of items
env_vars:
  - name: APP_ENV
    value: production
  - name: LOG_LEVEL
    value: info

ports:
  - "80:80"
  - "443:443"

tags:
  - devops
  - kubernetes
  - cicd
```

### Complex nested structures

```yaml
application:
  name: my-app
  version: "2.1.0"
  replicas: 3
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
  env:
    - name: DB_HOST
      value: postgres-service
    - name: DB_PORT
      value: "5432"
  labels:
    app: my-app
    tier: backend
    environment: production
```

---

## 4. Multi-line Strings

This is one of the most important and tricky parts of YAML.

### Literal Block Scalar `|` — preserves newlines

```yaml
# Every newline in the text is preserved as-is
script: |
  #!/bin/bash
  echo "Starting deployment"
  kubectl apply -f deployment.yaml
  echo "Done"

# Result: "#!/bin/bash\necho \"Starting deployment\"\nkubectl apply...\n"
```

### Folded Block Scalar `>` — newlines become spaces

```yaml
# Single newlines → spaces (paragraphs separated by blank lines stay)
description: >
  This is a very long description
  that spans multiple lines but
  will be joined into one paragraph.

# Result: "This is a very long description that spans multiple lines but will be joined into one paragraph.\n"

# Blank line = actual newline
description: >
  First paragraph that
  gets folded into one line.

  Second paragraph after
  the blank line.
```

### Chomping indicators — control trailing newlines

```yaml
# Default (clip) — keeps exactly one trailing newline
default: |
  hello

# Strip (-)  — removes ALL trailing newlines
strip: |-
  hello

# Keep (+)  — preserves all trailing newlines
keep: |+
  hello


# With folded:
folded_strip: >-
  no trailing newline at end

folded_keep: >+
  keeps trailing newlines
```

### Indentation indicator

```yaml
# Specify indentation explicitly (useful when content starts with spaces)
indented: |2
  two-space indented content
  that is preserved as-is
```

### Real-world examples

```yaml
# Kubernetes ConfigMap with script
apiVersion: v1
kind: ConfigMap
metadata:
  name: startup-script
data:
  init.sh: |
    #!/bin/bash
    set -euo pipefail
    echo "Initializing..."
    /app/migrate.sh
    /app/seed.sh

# GitHub Actions run step
- name: Deploy
  run: |
    echo "Deploying to $ENVIRONMENT"
    helm upgrade --install myapp ./chart \
      --set image.tag=$IMAGE_TAG \
      --namespace production

# Long command with folded (joins lines with spaces)
- name: Long curl command
  run: >
    curl -X POST
    -H "Content-Type: application/json"
    -H "Authorization: Bearer $TOKEN"
    -d '{"key": "value"}'
    https://api.example.com/deploy
```

---

## 5. Anchors & Aliases (DRY in YAML)

Anchors let you define a value once and reuse it, avoiding repetition.

### Basic syntax

```yaml
# & defines an anchor
# * references it (alias)

defaults: &defaults
  retries: 3
  timeout: 30
  log_level: info

production:
  <<: *defaults        # merge key — inlines all keys from defaults
  environment: prod
  log_level: warn      # override a merged key

staging:
  <<: *defaults
  environment: staging
```

### Merge key `<<`

```yaml
# Merge multiple anchors
base_config: &base
  image: ubuntu:22.04
  restart: always

monitoring_labels: &monitoring
  labels:
    prometheus: "true"
    grafana: "true"

app:
  <<: [*base, *monitoring]   # merge from multiple anchors
  name: my-app
```

### Anchoring scalar values

```yaml
# Anchor a single value
registry: &registry "registry.example.com"

services:
  web:
    image: *registry/web:latest
  api:
    image: *registry/api:latest
```

### Real-world: Ansible with anchors

```yaml
# Define common task options
common_task: &common
  become: yes
  tags:
    - setup

- name: Install nginx
  <<: *common
  apt:
    name: nginx
    state: present

- name: Install postgres
  <<: *common
  apt:
    name: postgresql
    state: present
```

### Real-world: GitHub Actions job matrix anchors

```yaml
env:
  NODE_VERSION: &node_version "20"

jobs:
  build:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: *node_version
  test:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: *node_version
```

> **Note:** Anchors only work within the same YAML document. They do NOT work across files or across `---` document separators.

---

## 6. Multiple Documents in One File

```yaml
# --- separates documents within one file
# ... optionally marks end of a document

---
# Document 1 — Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3

---
# Document 2 — Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
    - port: 80
...
```

Applying multi-document YAML in Kubernetes:

```bash
kubectl apply -f all-resources.yaml   # applies all documents in one file
```

---

## 7. Comments & Style Best Practices

### Comments

```yaml
# This is a comment — only full-line or end-of-line comments are supported
# YAML has NO multi-line comment syntax

name: priya      # inline comment
port: 8080       # this port must match the Dockerfile EXPOSE
```

### Indentation rules

```yaml
# ALWAYS use spaces — NEVER tabs
# 2 spaces per level is the universal convention in DevOps tooling

parent:          # level 0
  child:         # level 1 (2 spaces)
    grandchild:  # level 2 (4 spaces)
      value: 42  # level 3 (6 spaces)
```

### Style consistency

```yaml
# Choose block OR flow style per context and stick with it
# Block = multi-line, readable (preferred for config files)
# Flow = inline, compact (OK for short lists/maps in args)

# Good — block style for main config
resources:
  limits:
    cpu: "500m"
    memory: "256Mi"

# OK — flow style for short inline args
args: ["--config", "/etc/app/config.yaml", "--verbose"]
```

### Key naming conventions

```yaml
# snake_case is most common in Ansible/Python-based tools
max_retries: 3
log_level: info

# camelCase is used in Kubernetes
apiVersion: apps/v1
containerPort: 8080

# kebab-case in some tools (avoid — hard to use in code)
max-retries: 3   # can't access as dict key in some languages
```

---

## 8. Common Gotchas & Bugs

### The Norway problem (YAML 1.1 booleans)

```yaml
# YAML 1.1 treats these country codes as booleans!
country: NO     # → false (boolean!) not "NO"
flag: YES       # → true (boolean!) not "YES"

# Fix: always quote ambiguous values
country: "NO"
flag: "YES"

# Other YAML 1.1 boolean traps:
- on: "on"      # quote it
- off: "off"    # quote it
- y: "y"        # quote it
- n: "n"        # quote it
```

### Colon inside strings

```yaml
# Colons in values cause parse errors without quotes
title: Hello: World       # ERROR — parser sees key-value pair
title: "Hello: World"     # CORRECT

url: https://example.com  # ERROR in some parsers
url: "https://example.com" # CORRECT — always quote URLs
```

### Tab characters

```yaml
# NEVER use tabs — always spaces
# This causes cryptic parse errors that are hard to debug

# WRONG (with tab)
name:
[TAB]value: test   # ParseError: found character '\t' not allowed

# RIGHT (with 2 spaces)
name:
  value: test
```

### Numbers that should be strings

```yaml
# Without quotes these become numbers, not strings
version: 1.0        # float → loses the .0 in some serializers
zip_code: 01234     # may lose leading zero
port: 8080          # integer → fine for most uses but be explicit
image_tag: latest   # string (fine, no ambiguity)
image_tag: "1.2.3"  # CORRECT — quoted to ensure string
```

### Indentation inconsistency

```yaml
# Mixing indentation levels causes unexpected structure

# WRONG
items:
  - first
   - second    # 3 spaces instead of 2 — parse error or wrong structure

# CORRECT
items:
  - first
  - second
```

### Empty values vs null

```yaml
# These are equivalent (both null):
key1:
key2: null
key3: ~

# NOT the same as empty string:
key4: ""    # empty string
key5: ''    # empty string
```

### Special characters in keys

```yaml
# Keys with special chars must be quoted
"app.kubernetes.io/name": my-app
"prometheus.io/scrape": "true"

# Unquoted will fail or misparse:
app.kubernetes.io/name: my-app  # ERROR
```

### Duplicate keys

```yaml
# YAML spec says duplicate keys are an error — but many parsers allow it
# Last value wins (silently!), which causes hard-to-find bugs

config:
  port: 8080
  port: 9090   # silently overrides — port will be 9090
```

---

## 9. YAML in Kubernetes

### Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: default
  labels:
    app: my-app
    version: "1.0"
  annotations:
    description: "Example pod"
spec:
  containers:
    - name: web
      image: nginx:1.25
      ports:
        - containerPort: 80
          protocol: TCP
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
      env:
        - name: APP_ENV
          value: production
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app
  volumes:
    - name: config-vol
      configMap:
        name: app-config
  restartPolicy: Always
```

### Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

### ConfigMap and Secret

```yaml
# ConfigMap — non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
      timeout: 30

---
# Secret — base64-encoded sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: cHJpeWE=         # base64 of "priya"
  password: c3VwZXJzZWNyZXQ= # base64 of "supersecret"
stringData:
  # stringData accepts plain text (auto base64-encoded)
  api_key: "my-plain-text-api-key"
```

### Service and Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  type: ClusterIP
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

---

## 10. YAML in Ansible

### Playbook structure

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  vars:
    http_port: 80
    app_user: deploy
    packages:
      - nginx
      - git
      - curl

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

  tasks:
    - name: Install packages
      apt:
        name: "{{ packages }}"
        state: present

    - name: Create app directory
      file:
        path: /opt/myapp
        state: directory
        owner: "{{ app_user }}"
        mode: "0755"

    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/sites-available/myapp
      notify: Reload nginx

    - name: Enable nginx site
      file:
        src: /etc/nginx/sites-available/myapp
        dest: /etc/nginx/sites-enabled/myapp
        state: link

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

### Inventory YAML

```yaml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
          ansible_user: ubuntu
        web2:
          ansible_host: 192.168.1.11
          ansible_user: ubuntu
      vars:
        http_port: 80

    dbservers:
      hosts:
        db1:
          ansible_host: 192.168.1.20
      vars:
        db_port: 5432

  vars:
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ansible_python_interpreter: /usr/bin/python3
```

### Vars files and conditionals

```yaml
---
# group_vars/all.yml
environment: production
log_retention_days: 30
enable_monitoring: true

---
# tasks with conditionals
- name: Install dev tools
  apt:
    name: vim
    state: present
  when: environment == "development"

- name: Configure log rotation
  template:
    src: logrotate.j2
    dest: /etc/logrotate.d/app
  vars:
    days: "{{ log_retention_days }}"

- name: Run only on debian systems
  debug:
    msg: "This is Debian"
  when:
    - ansible_os_family == "Debian"
    - ansible_distribution_major_version | int >= 11
```

---

## 11. YAML in GitHub Actions

### Workflow structure

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - "release/**"
  pull_request:
    branches:
      - main
  schedule:
    - cron: "0 2 * * 1"    # every Monday at 2 AM UTC
  workflow_dispatch:         # manual trigger
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: staging
        type: choice
        options:
          - staging
          - production

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: ["18", "20", "22"]
        os: [ubuntu-latest, windows-latest]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.node-version == '20' && matrix.os == 'ubuntu-latest'

  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-and-push
    environment:
      name: production
      url: https://myapp.example.com
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Deploy
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          kubectl set image deployment/web-app \
            web=${{ needs.build-and-push.outputs.image_tag }}
          kubectl rollout status deployment/web-app
```

### Reusable workflow

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
    secrets:
      KUBECONFIG:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy ${{ inputs.image_tag }} to ${{ inputs.environment }}
        run: |
          echo "Deploying..."
```

---

## 12. YAML in Docker Compose

### Full docker-compose.yaml

```yaml
version: "3.9"

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    image: myapp:latest
    container_name: myapp_web
    restart: unless-stopped
    ports:
      - "80:3000"
      - "443:3443"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./uploads:/app/uploads
      - app_logs:/var/log/app
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M

  db:
    image: postgres:15-alpine
    container_name: myapp_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: myapp_redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend

  nginx:
    image: nginx:alpine
    container_name: myapp_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - web
    networks:
      - frontend

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  app_logs:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /var/log/myapp

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true   # no external internet access
```

---

## 13. YAML in Helm Charts

### values.yaml

```yaml
# Default values for my-chart

replicaCount: 2

image:
  repository: registry.example.com/myapp
  pullPolicy: IfNotPresent
  tag: ""   # Overridden by Chart.appVersion if empty

imagePullSecrets:
  - name: registry-secret

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

env:
  - name: APP_ENV
    value: production

config:
  logLevel: info
  timeout: 30
  features:
    darkMode: true
    betaAPI: false
```

### Template using values (templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        {{- toYaml .Values.podAnnotations | nindent 8 }}
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## 14. Validation & Debugging Tools

### yamllint — lint YAML files

```bash
# Install
pip install yamllint

# Lint a file
yamllint deployment.yaml

# Lint with config
yamllint -d "{extends: default, rules: {line-length: {max: 120}}}" file.yaml

# .yamllint config file
extends: default
rules:
  line-length:
    max: 120
  truthy:
    allowed-values: ["true", "false"]
  comments:
    min-spaces-from-content: 2
```

### yq — query and transform YAML (like jq for YAML)

```bash
# Install
brew install yq        # macOS
snap install yq        # Linux

# Read a value
yq '.spec.replicas' deployment.yaml

# Update a value
yq '.spec.replicas = 5' deployment.yaml

# In-place edit
yq -i '.spec.replicas = 5' deployment.yaml

# Read nested value
yq '.spec.template.spec.containers[0].image' deployment.yaml

# Filter and output
yq '.metadata.labels' deployment.yaml

# Merge two YAML files
yq ea '. as $item ireduce ({}; . * $item)' file1.yaml file2.yaml

# Convert YAML to JSON
yq -o=json deployment.yaml

# Convert JSON to YAML
cat data.json | yq -P '.'
```

### kubectl dry-run validation

```bash
# Validate without applying
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

# Validate all files in directory
kubectl apply -f ./k8s/ --dry-run=client
```

### kubeconform — validate against K8s schemas

```bash
# Install
brew install kubeconform

# Validate
kubeconform deployment.yaml
kubeconform -summary -output json ./k8s/
```

### Python — parse and validate

```python
import yaml

# Load YAML (safe_load prevents code execution)
with open("config.yaml") as f:
    config = yaml.safe_load(f)

# Load multiple documents
with open("multi.yaml") as f:
    docs = list(yaml.safe_load_all(f))

# Dump Python object to YAML
data = {"name": "priya", "port": 8080}
print(yaml.dump(data, default_flow_style=False))
```

### Online tools

| Tool | Purpose |
|------|---------|
| `yaml.org/spec` | YAML specification |
| `yamllint.com` | Online YAML linter |
| `onlineyamltools.com` | YAML → JSON converter |
| `kubeeval.io` | Kubernetes YAML validator online |

---

## 15. Quick Reference Cheat Sheet

```yaml
# ─── SCALARS ───────────────────────────────────────────────
string_bare: hello
string_single: 'no escape \n here'
string_double: "escape works \n here"
integer: 42
float: 3.14
bool_true: true
bool_false: false
null_value: null
null_tilde: ~

# ─── SEQUENCES ─────────────────────────────────────────────
list_block:
  - item1
  - item2
  - item3

list_flow: [item1, item2, item3]

# ─── MAPPINGS ──────────────────────────────────────────────
map_block:
  key1: value1
  key2: value2

map_flow: {key1: value1, key2: value2}

# ─── NESTED ────────────────────────────────────────────────
nested:
  list_of_maps:
    - name: alice
      role: admin
    - name: bob
      role: viewer

# ─── MULTI-LINE ────────────────────────────────────────────
literal_block: |
  line 1
  line 2
  line 3

folded_block: >
  these lines
  get joined
  into one

no_trailing_newline: |-
  content here

# ─── ANCHORS & ALIASES ─────────────────────────────────────
base: &base
  env: production
  retries: 3

derived:
  <<: *base
  retries: 5    # override

# ─── EXPLICIT TYPES ────────────────────────────────────────
force_string: !!str 123
force_int: !!int "42"
no_auto_bool: "true"
no_auto_null: "null"

# ─── MULTI-DOC ─────────────────────────────────────────────
---
document: one
---
document: two
```

### Key rules at a glance

| Rule | Detail |
|------|--------|
| Indentation | Spaces only — never tabs |
| Indent size | 2 spaces (universal convention) |
| Boolean safe | Use `true` / `false` (lowercase) |
| Quote URLs | Always quote `https://...` |
| Quote versions | Always quote `"1.0"`, `"2.3.1"` |
| Quote boolish strings | Quote `"yes"`, `"no"`, `"on"`, `"off"` |
| Anchors scope | Per-document only, not cross-file |
| Comments | `#` only — no block comments |
| Null | `null`, `~`, or empty value |
| Duplicate keys | Avoid — last value silently wins |

---

*Part of DevOpsNotes / LANGUAGES — see also `01_JSON.md`, `02_Groovy_Jenkins.md`*