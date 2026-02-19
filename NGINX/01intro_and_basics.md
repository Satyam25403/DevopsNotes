# Nginx - Introduction and Architecture

Complete guide to Nginx web server, reverse proxy, load balancer, and HTTP cache for high-performance web applications.

## Table of Contents
- [What is Nginx](#what-is-nginx)
- [Installation](#installation)
- [Configuration Basics](#configuration-basics)
- [Nginx as Web Server](#nginx-as-web-server)
- [Nginx as Reverse Proxy](#nginx-as-reverse-proxy)
- [Load Balancing](#load-balancing)
- [SSL/TLS and HTTPS](#ssltls-and-https)
- [Caching](#caching)
- [Kubernetes Ingress Controller](#kubernetes-ingress-controller)

---

## What is Nginx?

**Nginx** (pronounced "engine-x") is a high-performance, open-source web server that functions as:
- Web server
- Reverse proxy
- Load balancer
- HTTP cache
- API gateway
- SSL terminator

### Common Use Cases

**Static Content Serving:**
- HTML, CSS, JavaScript files
- Images, videos, documents
- Single-page applications (SPAs)

**Reverse Proxy:**
- Route traffic to backend applications
- Flask, Express, Django, Spring Boot
- Microservices architecture

**Load Balancing:**
- Distribute traffic across multiple servers
- Fault tolerance and high availability
- Horizontal scaling

**SSL Termination:**
- Handle HTTPS connections
- Decrypt SSL traffic before backend
- Offload encryption from application servers

**HTTP Cache:**
- Cache responses from backend
- Reduce backend load
- Faster content delivery

**Docker/Kubernetes:**
- Container orchestration
- Ingress controller
- Service mesh integration

### Why Nginx?

**Performance:**
- Event-driven architecture
- Handles thousands of concurrent connections
- Low memory footprint
- Non-blocking I/O

**Scalability:**
- Horizontal scaling support
- Load balancing built-in
- Efficient resource utilization

**Flexibility:**
- Highly configurable
- Module system
- Extensive ecosystem

**Reliability:**
- Battle-tested in production
- Used by top companies (Netflix, Cloudflare, Dropbox)
- Active community support

---

## Installation

### Ubuntu/Debian

```bash
# Update package list
sudo apt update

# Install Nginx
sudo apt install nginx

# Start Nginx
sudo systemctl start nginx

# Enable auto-start on boot
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

### Verify Installation

```bash
# Check version
nginx -v
# Output: nginx version: nginx/1.18.0

# Detailed version info
nginx -V

# Test configuration
sudo nginx -t

# Check help
nginx -h
```

**Access Nginx:**
```bash
# Open browser
http://localhost
# Should see "Welcome to nginx!" page
```

### CentOS/RHEL

```bash
sudo yum install epel-release
sudo yum install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Docker

```bash
# Pull Nginx image
docker pull nginx

# Run Nginx container
docker run -d -p 80:80 --name nginx nginx

# With custom config
docker run -d -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf \
  --name nginx nginx
```

---

## Configuration Basics

### Configuration File Locations

**Main configuration:**
```bash
/etc/nginx/nginx.conf
```

**Site configurations:**
```bash
/etc/nginx/sites-available/  # Available sites
/etc/nginx/sites-enabled/    # Enabled sites (symlinks)
/etc/nginx/conf.d/           # Additional configs
```

**Logs:**
```bash
/var/log/nginx/access.log    # Access logs
/var/log/nginx/error.log     # Error logs
```

**Web root:**
```bash
/var/www/html/               # Default document root
/usr/share/nginx/html/       # Alternative on some systems
```

### Find Nginx Locations

```bash
# Locate Nginx binary
whereis nginx

# Find configuration
nginx -V 2>&1 | grep -o '\-\-conf-path=\S*'

# Find logs
nginx -V 2>&1 | grep -o '\-\-error-log-path=\S*'
```

### Basic nginx.conf Structure

```nginx
# Main context - global settings
user www-data;
worker_processes auto;
pid /run/nginx.pid;

# Events context - connection processing
events {
    worker_connections 1024;
}

# HTTP context - web server settings
http {
    # Include MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Server context - virtual host
    server {
        listen 80;
        server_name localhost;

        # Location context - request routing
        location / {
            root /var/www/html;
            index index.html;
        }
    }
}
```

### Configuration Contexts

**Main Context:**
- Global settings
- Worker processes
- User permissions

**Events Context:**
- Connection processing
- Worker connections
- Event model

**HTTP Context:**
- Web server settings
- MIME types
- Logging formats

**Server Context:**
- Virtual host definition
- Listen ports
- Server names

**Location Context:**
- Request routing
- Proxy settings
- Cache configuration

### Testing and Reloading

```bash
# Test configuration syntax
sudo nginx -t

# Reload configuration (graceful)
sudo nginx -s reload

# Restart Nginx
sudo systemctl restart nginx

# Stop Nginx
sudo nginx -s stop
sudo systemctl stop nginx
```

---

## Nginx as Web Server

### Serving Static Files

**Basic static file server:**
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}
```

**What this does:**
- Listens on port 80 (HTTP)
- Serves files from `/var/www/html`
- Default file: `index.html`
- Access: `http://localhost`

### Multiple Locations

```nginx
server {
    listen 80;
    server_name example.com;

    # Root path
    location / {
        root /var/www/html;
        index index.html;
    }

    # Images
    location /images/ {
        root /var/www;
        autoindex on;  # Directory listing
    }

    # Downloads
    location /downloads/ {
        alias /var/www/files/;
        autoindex on;
    }
}
```

**Difference between `root` and `alias`:**

**root:**
```nginx
location /images/ {
    root /var/www;
}
# Request: /images/pic.jpg
# Serves: /var/www/images/pic.jpg
```

**alias:**
```nginx
location /images/ {
    alias /var/www/pics/;
}
# Request: /images/pic.jpg
# Serves: /var/www/pics/pic.jpg
```

### File Type Handling

```nginx
server {
    listen 80;
    
    # HTML files
    location ~ \.html$ {
        root /var/www/html;
    }

    # Images with caching
    location ~* \.(jpg|jpeg|png|gif|ico|svg)$ {
        root /var/www/images;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # CSS and JavaScript
    location ~* \.(css|js)$ {
        root /var/www/static;
        expires 7d;
    }

    # Download PDFs
    location ~* \.pdf$ {
        root /var/www/documents;
        add_header Content-Disposition "attachment";
    }
}
```

### Custom Error Pages

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/html;

    # Custom error pages
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html {
        internal;
    }

    location = /50x.html {
        internal;
    }
}
```

### Directory Listing

```nginx
location /downloads/ {
    root /var/www;
    autoindex on;               # Enable directory listing
    autoindex_exact_size off;   # Human-readable sizes
    autoindex_localtime on;     # Local timezone
}
```

---

## Nginx as Reverse Proxy

### Basic Reverse Proxy

**Proxy to backend application:**
```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**What this does:**
- Forwards all requests to Node.js app on port 3000
- Preserves client information in headers
- Nginx acts as gateway

### Multiple Backend Services

```nginx
server {
    listen 80;
    server_name example.com;

    # Frontend
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
    }

    # Backend API
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Admin panel
    location /admin/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
}
```

### Essential Proxy Headers

```nginx
location / {
    proxy_pass http://backend;
    
    # Essential headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
}
```

**Header explanations:**

| Header | Purpose |
|--------|---------|
| `Host` | Original domain name |
| `X-Real-IP` | Client's real IP address |
| `X-Forwarded-For` | Client IP through proxy chain |
| `X-Forwarded-Proto` | Original protocol (http/https) |
| `X-Forwarded-Host` | Original host header |
| `X-Forwarded-Port` | Original port |

### WebSocket Support

```nginx
location /ws/ {
    proxy_pass http://websocket_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

### Access Headers in Node.js

```javascript
// Express example
app.get('/', (req, res) => {
    const ip = req.headers['x-forwarded-for'] || req.connection.remoteAddress;
    const proto = req.headers['x-forwarded-proto'];
    const host = req.headers['host'];
    
    res.send(`
        Client IP: ${ip}
        Protocol: ${proto}
        Host: ${host}
    `);
});
```

---

## Load Balancing

### Basic Load Balancing (Round Robin)

```nginx
http {
    upstream backend_servers {
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
        server 127.0.0.1:3003;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
        }
    }
}
```

**How it works:**
- Requests distributed evenly
- Each server gets turn in sequence
- Default algorithm

### Load Balancing Methods

**1. Round Robin (default):**
```nginx
upstream backend {
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}
```

**2. Least Connections:**
```nginx
upstream backend {
    least_conn;  # Send to server with fewest connections
    server 192.168.1.101;
    server 192.168.1.102;
    server 192.168.1.103;
}
```

**3. IP Hash (Session Persistence):**
```nginx
upstream backend {
    ip_hash;  # Same client always goes to same server
    server 192.168.1.101;
    server 192.168.1.102;
}
```

**4. Weighted Round Robin:**
```nginx
upstream backend {
    server 192.168.1.101 weight=3;  # Gets 3x more requests
    server 192.168.1.102 weight=1;
    server 192.168.1.103 weight=1;
}
```

### Server Parameters

```nginx
upstream backend {
    server 192.168.1.101 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.102 weight=1;
    server 192.168.1.103 backup;  # Only used if others fail
    server 192.168.1.104 down;    # Temporarily removed
}
```

**Parameters:**
- `weight=N` - Server weight (default: 1)
- `max_fails=N` - Failed attempts before marking down (default: 1)
- `fail_timeout=T` - Time to mark server down (default: 10s)
- `backup` - Backup server (used when others fail)
- `down` - Permanently mark server as unavailable

### Health Checks (Passive)

```nginx
upstream backend {
    server 192.168.1.101 max_fails=3 fail_timeout=30s;
    server 192.168.1.102 max_fails=3 fail_timeout=30s;
}
```

**How it works:**
- Nginx monitors responses
- After 3 failed attempts, marks server down
- Retries after 30 seconds
- No active probing (use Nginx Plus for active checks)

---

## SSL/TLS and HTTPS

### Generate Self-Signed Certificate

**For development/testing:**
```bash
# Create directory
mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

# Generate certificate and key
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx-selfsigned.key \
  -out nginx-selfsigned.crt \
  -subj "/CN=localhost"
```

**Parameters explained:**
- `-x509` - Self-signed certificate (not CSR)
- `-nodes` - No password on private key
- `-days 365` - Valid for 1 year
- `-newkey rsa:2048` - 2048-bit RSA key
- `-keyout` - Private key output file
- `-out` - Certificate output file

**Files created:**
- `nginx-selfsigned.key` - Private key (keep secret!)
- `nginx-selfsigned.crt` - Public certificate

### HTTPS Server Configuration

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    # SSL certificate files
    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

### HTTP to HTTPS Redirection

```nginx
# Redirect all HTTP to HTTPS
server {
    listen 80;
    server_name example.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

**What `301` means:**
- Permanent redirect
- Browsers cache the redirect
- SEO-friendly

### Complete HTTPS Load Balancer

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name localhost;

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Load Balancer
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://node_backends;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Upstream servers
upstream node_backends {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

### Production SSL Settings

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # Certificates
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Security settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    
    # HSTS (force HTTPS for 1 year)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # SSL session
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        # Your configuration
    }
}
```

---

## Caching

### Basic Proxy Cache

```nginx
http {
    # Define cache path
    proxy_cache_path /var/cache/nginx 
        levels=1:2 
        keys_zone=my_cache:10m 
        max_size=1g 
        inactive=60m 
        use_temp_path=off;

    server {
        listen 80;

        location / {
            proxy_cache my_cache;
            proxy_pass http://backend:5000;
            
            # Cache valid responses
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
            
            # Add cache status header
            add_header X-Cache-Status $upstream_cache_status;
            
            # Proxy headers
            proxy_set_header Host $host;
        }
    }
}
```

### Cache Directives Explained

| Directive | Description |
|-----------|-------------|
| `proxy_cache_path` | Cache storage location and settings |
| `levels=1:2` | Directory hierarchy (optimization) |
| `keys_zone=name:size` | Shared memory for cache keys (10MB) |
| `max_size=1g` | Maximum cache size (1GB) |
| `inactive=60m` | Remove if not accessed for 60 min |
| `proxy_cache` | Enable cache for location |
| `proxy_cache_valid` | Cache duration per status code |

### Cache Status Values

**Check `X-Cache-Status` header:**
- `MISS` - Not cached, fetched from backend
- `HIT` - Served from cache
- `EXPIRED` - Cache expired, refreshed
- `BYPASS` - Caching bypassed
- `UPDATING` - Updating stale cache
- `STALE` - Serving stale while updating

### Cache Control

```nginx
location / {
    proxy_cache my_cache;
    proxy_pass http://backend;
    
    # Bypass cache for certain requests
    proxy_cache_bypass $cookie_nocache $arg_nocache;
    
    # Don't cache these
    proxy_no_cache $cookie_nocache $arg_nocache;
    
    # Cache based on URI and args
    proxy_cache_key "$scheme$request_method$host$request_uri";
    
    # Cache only GET and HEAD
    proxy_cache_methods GET HEAD;
}
```

### Static File Caching

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    root /var/www/static;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

---

## Kubernetes Ingress Controller

### What is Ingress Controller?

**In Kubernetes:**
- **Ingress** - Rules for routing external traffic
- **Ingress Controller** - Implementation that enforces rules
- **Nginx Ingress** - Most popular choice

### Install Nginx Ingress (Helm)

```bash
# Add Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install
helm install nginx-ingress ingress-nginx/ingress-nginx

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Basic Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 5000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

### TLS Termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    # ... paths
```

**Create TLS secret:**
```bash
kubectl create secret tls myapp-tls \
  --cert=cert.pem \
  --key=key.pem
```

---

## Quick Reference

### Essential Commands

```bash
# Test config
sudo nginx -t

# Reload
sudo nginx -s reload

# Stop
sudo nginx -s stop

# Start
sudo systemctl start nginx

# Status
sudo systemctl status nginx
```

### Common Locations

```bash
/etc/nginx/nginx.conf           # Main config
/etc/nginx/sites-available/     # Available sites
/etc/nginx/sites-enabled/       # Enabled sites
/var/log/nginx/                 # Logs
/var/www/html/                  # Web root
```

---

This comprehensive guide covers Nginx fundamentals, web server configuration, reverse proxy, load balancing, SSL/TLS, caching, and Kubernetes integration for production-ready deployments.