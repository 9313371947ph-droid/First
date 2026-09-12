# Module 05: Containerization with Docker

## 🎯 Learning Objectives
By the end of this module, you will:
- Understand containerization concepts and why Docker revolutionized DevOps
- Master Docker architecture and components
- Create, build, and optimize Docker images
- Run and manage containers effectively
- Implement Docker networking and storage
- Deploy multi-container applications with Docker Compose
- Apply security best practices for containers
- Troubleshoot common Docker issues
- Prepare Docker images for production environments

## 📚 Table of Contents
1. [Introduction to Containerization](#1-introduction-to-containerization)
2. [Docker Architecture Deep Dive](#2-docker-architecture-deep-dive)
3. [Working with Docker Images](#3-working-with-docker-images)
4. [Container Management](#4-container-management)
5. [Docker Networking](#5-docker-networking)
6. [Docker Storage and Volumes](#6-docker-storage-and-volumes)
7. [Docker Compose](#7-docker-compose)
8. [Dockerfile Best Practices](#8-dockerfile-best-practices)
9. [Docker Security](#9-docker-security)
10. [Production Deployment Strategies](#10-production-deployment-strategies)
11. [Hands-on Labs](#11-hands-on-labs)
12. [Troubleshooting Guide](#12-troubleshooting-guide)
13. [Quiz & Knowledge Check](#13-quiz--knowledge-check)

---

## 1. Introduction to Containerization

### What is Containerization?
Containerization is a lightweight form of virtualization that packages an application with its dependencies, libraries, and configuration files into a single unit called a **container**. Unlike traditional virtual machines (VMs), containers share the host OS kernel, making them significantly faster and more resource-efficient.

### VMs vs Containers

| Aspect | Virtual Machines | Containers |
|--------|-----------------|------------|
| **Boot Time** | Minutes | Seconds |
| **Size** | GBs | MBs |
| **Performance** | Slower (hypervisor overhead) | Near-native |
| **Isolation** | Complete (separate OS) | Process-level |
| **Resource Usage** | High | Low |
| **Portability** | Moderate | Excellent |

### Why Docker Changed Everything
Before Docker (2013), containers existed but were difficult to use. Docker introduced:
- **Standardized image format** (Docker images)
- **Easy-to-use CLI** for developers
- **Docker Hub** for sharing images
- **Ecosystem tools** (Compose, Swarm, etc.)

### Key Benefits
✅ **Consistency**: "Works on my machine" problem solved
✅ **Isolation**: Applications don't interfere with each other
✅ **Efficiency**: Higher density than VMs
✅ **Portability**: Run anywhere Docker is installed
✅ **Scalability**: Easy to scale up/down
✅ **Version Control**: Images can be versioned and rolled back

---

## 2. Docker Architecture Deep Dive

### Core Components

```
┌─────────────────────────────────────────────────────────┐
│                     Docker Client                        │
│                  (docker CLI commands)                   │
└────────────────────┬────────────────────────────────────┘
                     │ REST API
                     ▼
┌─────────────────────────────────────────────────────────┐
│                      Docker Daemon                       │
│                    (dockerd service)                     │
│  ┌─────────────┬─────────────┬──────────────────────┐   │
│  │   Images    │  Containers │  Networks/Volumes    │   │
│  └─────────────┴─────────────┴──────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                    Host Operating System                 │
│              (Linux Kernel with namespaces/cgroups)      │
└─────────────────────────────────────────────────────────┘
```

### Docker Client
- Command-line interface (`docker` command)
- Sends commands to Docker Daemon via REST API
- Can communicate with remote daemons

### Docker Daemon (dockerd)
- Background service managing Docker objects
- Handles image builds, container runs, networking
- Listens for Docker API requests

### Docker Objects

#### Images
- Read-only templates used to create containers
- Built from layers (each instruction in Dockerfile creates a layer)
- Identified by ID or tag (e.g., `ubuntu:20.04`)

#### Containers
- Runnable instances of images
- Have a writable layer on top of image layers
- Can be started, stopped, moved, deleted

#### Networks
- Enable communication between containers
- Types: bridge, host, none, overlay, macvlan

#### Volumes
- Persistent storage for containers
- Survive container deletion
- Managed by Docker

### Under the Hood: Linux Kernel Features

#### Namespaces (Isolation)
- **PID**: Process isolation
- **NET**: Network interfaces and routing
- **MNT**: Filesystem mount points
- **UTS**: Hostname and domain name
- **IPC**: Inter-process communication
- **USER**: User and group IDs

#### Control Groups (cgroups) (Resource Limits)
- CPU usage limits
- Memory limits
- Block I/O limits
- Network bandwidth limits

#### Union File Systems (Layered Images)
- **Overlay2**: Default in modern Docker
- Allows images to be layered efficiently
- Copy-on-write mechanism

---

## 3. Working with Docker Images

### Image Structure
Docker images consist of multiple read-only layers stacked on top of each other:

```
┌─────────────────────┐
│  Writable Container │  ← Only this layer is writable
│       Layer         │
├─────────────────────┤
│   Application Layer │  ← Your app code
├─────────────────────┤
│  Dependency Layer   │  ← Libraries, packages
├─────────────────────┤
│    OS Layer         │  ← Base OS (Ubuntu, Alpine, etc.)
└─────────────────────┘
```

### Basic Image Commands

```bash
# Search for images on Docker Hub
docker search nginx

# Pull an image
docker pull nginx:1.21

# List local images
docker images
docker image ls

# Inspect image details
docker image inspect nginx:1.21

# View image history (layers)
docker image history nginx:1.21

# Remove an image
docker image rm nginx:1.21
docker rmi nginx:1.21

# Remove unused images
docker image prune
docker image prune -a  # Remove all unused images
```

### Building Images from Dockerfile

```bash
# Build an image from Dockerfile in current directory
docker build -t myapp:1.0 .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:1.0 .

# Build without cache
docker build --no-cache -t myapp:1.0 .

# Build specific target (multi-stage builds)
docker build --target production -t myapp:prod .
```

### Tagging and Pushing Images

```bash
# Tag an image
docker tag myapp:1.0 username/myapp:1.0
docker tag myapp:1.0 username/myapp:latest

# Login to Docker Hub
docker login

# Push to registry
docker push username/myapp:1.0
docker push username/myapp:latest

# Push to private registry
docker tag myapp:1.0 registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0
```

### Saving and Loading Images

```bash
# Save image to tar file
docker save -o myapp.tar myapp:1.0

# Load image from tar file
docker load -i myapp.tar

# Export container filesystem
docker export container_id > container.tar

# Import as image
docker import container.tar myimage:1.0
```

---

## 4. Container Management

### Running Containers

```bash
# Basic run
docker run nginx

# Run in detached mode (background)
docker run -d nginx

# Run with custom name
docker run -d --name web-server nginx

# Run with port mapping
docker run -d -p 8080:80 nginx

# Run with environment variables
docker run -d -e DB_HOST=localhost -e DB_PORT=5432 myapp

# Run with volume mount
docker run -d -v /host/data:/container/data myapp

# Run with resource limits
docker run -d --memory="512m" --cpus="1.0" myapp

# Run interactively (for debugging)
docker run -it ubuntu bash

# Run with restart policy
docker run -d --restart unless-stopped nginx
docker run -d --restart on-failure:3 myapp
docker run -d --restart always nginx
```

### Container Lifecycle Commands

```bash
# List running containers
docker ps
docker container ls

# List all containers (including stopped)
docker ps -a

# Stop a container (graceful shutdown)
docker stop container_name
docker stop container_id

# Start a stopped container
docker start container_name

# Restart a container
docker restart container_name

# Kill a container (forceful)
docker kill container_name

# Pause/Unpause a container
docker pause container_name
docker unpause container_name

# Remove a container
docker rm container_name
docker rm -f container_name  # Force remove running container

# Remove all stopped containers
docker container prune
```

### Executing Commands in Containers

```bash
# Execute command in running container
docker exec container_name ls -la
docker exec -it container_name bash

# Execute as specific user
docker exec -u www-data container_name whoami

# Run interactive session
docker exec -it container_name /bin/bash

# Execute and detach
docker exec -d container_name touch /tmp/file.txt
```

### Viewing Container Information

```bash
# View container logs
docker logs container_name
docker logs -f container_name  # Follow logs
docker logs --tail 100 container_name  # Last 100 lines
docker logs --since 2024-01-01 container_name

# Inspect container details
docker inspect container_name

# View resource usage
docker stats
docker stats container_name

# View top processes
docker top container_name

# View changes to filesystem
docker diff container_name
```

### Copying Files

```bash
# Copy from host to container
docker cp file.txt container_name:/path/in/container/

# Copy from container to host
docker cp container_name:/path/in/container/file.txt .

# Copy entire directory
docker cp ./dir container_name:/path/in/container/
```

---

## 5. Docker Networking

### Network Drivers

#### Bridge Network (Default)
- Default network for containers on same host
- Containers can communicate by IP or container name
- Isolated from host network

```bash
# Create custom bridge network
docker network create --driver bridge my-network

# Run container on specific network
docker run -d --network my-network --name web nginx
docker run -d --network my-network --name db postgres

# Containers can resolve each other by name
# web can reach db at http://db:5432
```

#### Host Network
- Container shares host's network namespace
- No network isolation
- Better performance (no NAT)

```bash
docker run -d --network host nginx
# Container uses host's IP and ports directly
```

#### None Network
- Complete network isolation
- Only loopback interface available

```bash
docker run -d --network none myapp
```

#### Overlay Network
- Multi-host networking (Docker Swarm)
- Enables communication across hosts

```bash
docker network create --driver overlay my-overlay
```

#### Macvlan Network
- Assign MAC address to container
- Container appears as physical device on network

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 macvlan-net
```

### Network Commands

```bash
# List networks
docker network ls

# Inspect network
docker network inspect my-network

# Create network
docker network create my-network
docker network create --driver bridge --subnet 172.20.0.0/16 my-network

# Connect container to network
docker network connect my-network container_name

# Disconnect container from network
docker network disconnect my-network container_name

# Remove network
docker network rm my-network

# Remove unused networks
docker network prune
```

### DNS and Service Discovery
- Docker provides embedded DNS server at 127.0.0.11
- Containers can resolve other containers by name on same network
- Custom DNS servers can be specified

```bash
docker run -d --dns 8.8.8.8 --dns 8.8.4.4 nginx
```

### Port Publishing

```bash
# Publish specific port
docker run -d -p 8080:80 nginx

# Publish all exposed ports to random ports
docker run -d -P nginx

# Publish to specific interface
docker run -d -p 127.0.0.1:8080:80 nginx

# Publish UDP ports
docker run -d -p 53:53/udp dns-server

# Publish multiple ports
docker run -d -p 80:80 -p 443:443 nginx
```

---

## 6. Docker Storage and Volumes

### Storage Options

| Type | Description | Use Case |
|------|-------------|----------|
| **Volumes** | Managed by Docker, stored in `/var/lib/docker/volumes/` | Production data, persistence |
| **Bind Mounts** | Map host filesystem path to container | Development, config files |
| **tmpfs** | In-memory storage (Linux only) | Sensitive data, temporary files |

### Working with Volumes

```bash
# Create a volume
docker volume create my-volume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-volume

# Use volume in container
docker run -d -v my-volume:/data nginx

# Use anonymous volume
docker run -d -v /data nginx

# Remove volume
docker volume rm my-volume

# Remove unused volumes
docker volume prune

# Backup volume
docker run --rm -v my-volume:/source -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /source .

# Restore volume
docker run --rm -v my-volume:/target -v $(pwd):/backup alpine tar xzf /backup/backup.tar.gz -C /target
```

### Bind Mounts

```bash
# Mount host directory
docker run -d -v /host/path:/container/path nginx

# Mount specific file
docker run -d -v /host/config.conf:/etc/nginx/nginx.conf nginx

# Read-only mount
docker run -d -v /host/data:/container/data:ro nginx

# Using --mount syntax (more explicit)
docker run -d --mount type=bind,source=/host/path,target=/container/path nginx
```

### tmpfs Mounts

```bash
# Create tmpfs mount
docker run -d --tmpfs /app/tmp nginx

# With size limit
docker run -d --tmpfs /app/tmp:size=100m nginx
```

### Storage Driver Information

```bash
# Check storage driver
docker info | grep "Storage Driver"

# Common drivers: overlay2 (recommended), aufs, devicemapper
```

### Best Practices for Data Persistence
1. Use volumes for database data
2. Use bind mounts for development code
3. Never store important data in container writable layer
4. Backup volumes regularly
5. Use named volumes for easier management

---

## 7. Docker Compose

### What is Docker Compose?
Docker Compose is a tool for defining and running multi-container Docker applications using a YAML file.

### Installation

```bash
# Docker Desktop includes Compose
# For Linux:
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker-compose --version
# Or with newer Docker versions:
docker compose version
```

### docker-compose.yml Structure

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    container_name: my-web
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - nginx-logs:/var/log/nginx
    networks:
      - app-network
    environment:
      - NGINX_HOST=example.com
    depends_on:
      - api
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      args:
        - VERSION=1.0
    container_name: my-api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - DB_PASSWORD=${DB_PASSWORD}
    volumes:
      - api-data:/app/data
    networks:
      - app-network
    depends_on:
      - db
    restart: always

  db:
    image: postgres:15-alpine
    container_name: my-db
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  nginx-logs:
  api-data:
  db-data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### Environment Variables

Create a `.env` file:
```bash
DB_PASSWORD=supersecret123
API_KEY=your-api-key
```

Reference in compose file:
```yaml
environment:
  - DB_PASSWORD=${DB_PASSWORD}
  - API_KEY=${API_KEY:-default_value}
```

### Compose Commands

```bash
# Start services
docker-compose up
docker-compose up -d  # Detached mode

# Build and start
docker-compose up --build

# Stop services
docker-compose down
docker-compose down -v  # Remove volumes too

# View logs
docker-compose logs
docker-compose logs -f api  # Follow specific service
docker-compose logs --tail=100

# List running containers
docker-compose ps

# Execute command in service
docker-compose exec api bash
docker-compose run --rm api npm test

# Scale services
docker-compose up -d --scale api=3

# View resource usage
docker-compose top

# Pause/Unpause services
docker-compose pause
docker-compose unpause

# Restart services
docker-compose restart

# Stop specific service
docker-compose stop api
docker-compose start api

# Remove stopped containers
docker-compose rm

# Validate compose file
docker-compose config

# Pull images
docker-compose pull

# Build images
docker-compose build
docker-compose build --no-cache
```

### Multiple Compose Files

```bash
# Override with additional file
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Merge configurations (later files override earlier)
```

Example override file (`docker-compose.prod.yml`):
```yaml
version: '3.8'

services:
  api:
    environment:
      - NODE_ENV=production
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

---

## 8. Dockerfile Best Practices

### Anatomy of a Dockerfile

```dockerfile
# Base image
FROM node:18-alpine

# Metadata
LABEL maintainer="your.email@example.com"
LABEL version="1.0"
LABEL description="My Node.js Application"

# Build arguments
ARG APP_VERSION=1.0
ENV APP_VERSION=${APP_VERSION}

# Set working directory
WORKDIR /app

# Copy package files first (leverages layer caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Change ownership
RUN chown -R nodejs:nodejs /app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Define entrypoint and command
ENTRYPOINT ["node"]
CMD ["server.js"]
```

### Best Practices

#### 1. Use Specific Base Image Tags
```dockerfile
# ❌ Bad
FROM node:latest

# ✅ Good
FROM node:18.17.0-alpine
```

#### 2. Minimize Layers
```dockerfile
# ❌ Bad (creates multiple layers)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good (single layer)
RUN apt-get update && \
    apt-get install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

#### 3. Leverage Layer Caching
```dockerfile
# ❌ Bad (copies everything before installing deps)
COPY . .
RUN npm install

# ✅ Good (copy package files first)
COPY package*.json ./
RUN npm install
COPY . .
```

#### 4. Use .dockerignore
Create `.dockerignore` file:
```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
Dockerfile
.dockerignore
*.log
coverage
.DS_Store
```

#### 5. Multi-Stage Builds
```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

#### 6. Don't Run as Root
```dockerfile
RUN adduser -D -u 1001 appuser
USER appuser
```

#### 7. Use COPY instead of ADD
```dockerfile
# ❌ ADD has extra features (URL, tar extraction) that may be unwanted
ADD app.tar.gz /app/

# ✅ COPY is more transparent
COPY app.tar.gz /app/
RUN tar -xzf app.tar.gz
```

#### 8. Handle SIGTERM Properly
Ensure your application handles graceful shutdown:
```javascript
// Node.js example
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    process.exit(0);
  });
});
```

#### 9. Optimize Image Size
```dockerfile
# Use Alpine-based images
FROM python:3.11-alpine

# Clean up in same RUN instruction
RUN pip install --no-cache-dir flask && \
    rm -rf /root/.cache

# Use distroless images for maximum security
FROM gcr.io/distroless/nodejs18
```

#### 10. Document with LABEL and comments
```dockerfile
LABEL version="1.0"
LABEL description="Production-ready Node.js API"
LABEL com.example.maintainer="team@example.com"
```

### Example: Complete Production Dockerfile

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder

WORKDIR /app

# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download

# Copy source and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Production
FROM alpine:3.18

# Install CA certificates for HTTPS
RUN apk --no-cache add ca-certificates

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/main .

# Change ownership
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["./main"]
```

---

## 9. Docker Security

### Security Best Practices

#### 1. Keep Images Updated
```bash
# Regularly update base images
docker pull alpine:latest

# Use automated tools
docker scout cves myapp:1.0
```

#### 2. Scan Images for Vulnerabilities
```bash
# Docker Scout (built-in)
docker scout cves myapp:1.0
docker scout recommendations myapp:1.0

# Trivy (open source)
trivy image myapp:1.0

# Clair
docker run -p 6060:6060 quay.io/coreos/clair
```

#### 3. Use Minimal Base Images
```dockerfile
# Instead of ubuntu/debian
FROM alpine:3.18

# Or even better - distroless
FROM gcr.io/distroless/static-debian11
```

#### 4. Don't Run as Root
```dockerfile
# Create user and switch
RUN useradd -r -u 1001 appuser
USER appuser
```

#### 5. Limit Capabilities
```bash
# Drop all capabilities, add only needed
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Common capabilities:
# NET_BIND_SERVICE - Bind to ports < 1024
# CHOWN - Change file ownership
# SETUID/SETGID - Change UID/GID
```

#### 6. Use Read-Only Filesystem
```bash
docker run --read-only -v /tmp nginx
docker run --read-only --tmpfs /app/tmp nginx
```

#### 7. Resource Limits
```bash
# Prevent DoS attacks
docker run -d --memory="512m" --cpus="1.0" --pids-limit=100 myapp

# Prevent fork bombs
docker run --pids-limit=50 myapp
```

#### 8. Secret Management
```bash
# ❌ Bad: Hardcoded secrets
docker run -e DB_PASSWORD=secret123 myapp

# ✅ Good: Use Docker secrets (Swarm)
echo "secret123" | docker secret create db_password -
docker service create --secret db_password myapp

# ✅ Good: Use environment files
docker run --env-file .env.myapp myapp

# ✅ Good: Use external secret managers
docker run -e DB_PASSWORD=$(aws secretsmanager get-secret-value ...) myapp
```

#### 9. Network Segmentation
```bash
# Create isolated networks
docker network create --driver bridge frontend
docker network create --driver bridge backend

# Web tier can access API, API can access DB, but web cannot access DB directly
docker run -d --network frontend --name web nginx
docker run -d --network frontend --network backend --name api myapi
docker run -d --network backend --name db postgres
```

#### 10. Enable Docker Content Trust
```bash
# Enable DCT
export DOCKER_CONTENT_TRUST=1

# Only pull signed images
docker pull nginx:1.21  # Will fail if not signed

# Sign your own images
docker trust sign username/myapp:1.0
```

#### 11. Audit Docker Daemon
```bash
# Run Docker in rootless mode
dockerd-rootless-setuptool.sh install

# Use user namespaces
# Edit /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

#### 12. Secure Docker Socket
```bash
# Never expose Docker socket to containers unless absolutely necessary
# If needed, use socket proxy
docker run -v /var/run/docker.sock:/var/run/docker.sock ...

# Better: Use TCP with TLS
# Edit /etc/docker/daemon.json
{
  "hosts": ["tcp://0.0.0.0:2376"],
  "tls": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "tlsverify": true
}
```

### Security Checklist
- [ ] Use minimal base images
- [ ] Scan images for vulnerabilities regularly
- [ ] Don't run containers as root
- [ ] Use read-only filesystems where possible
- [ ] Implement resource limits
- [ ] Manage secrets properly
- [ ] Segment networks
- [ ] Enable content trust
- [ ] Keep Docker updated
- [ ] Audit container activity
- [ ] Use security profiles (AppArmor, SELinux)

---

## 10. Production Deployment Strategies

### Strategy 1: Blue-Green Deployment

```bash
# Blue environment (current production)
docker run -d --name app-blue -p 80:80 myapp:v1

# Deploy green environment (new version)
docker run -d --name app-green -p 8080:80 myapp:v2

# Test green environment
curl http://localhost:8080/health

# Switch traffic (update load balancer)
# Then remove blue
docker stop app-blue
docker rm app-blue
docker rename app-green app-blue
```

### Strategy 2: Rolling Updates (Docker Swarm)

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: myapp:v1
    deploy:
      replicas: 5
      update_config:
        parallelism: 2
        delay: 10s
        failure_action: rollback
        monitor: 30s
        max_failure_ratio: 0.1
      rollback_config:
        parallelism: 2
        delay: 10s
```

```bash
# Deploy stack
docker stack deploy -c docker-compose.yml myapp

# Update image
docker service update --image myapp:v2 myapp_web

# Monitor rollout
docker service ps myapp_web

# Rollback if needed
docker service rollback myapp_web
```

### Strategy 3: Canary Deployment

```bash
# Run 90% traffic to v1, 10% to v2
docker run -d --name app-v1 -l version=v1 myapp:v1
docker run -d --name app-v2 -l version=v2 myapp:v2

# Use load balancer with weighted routing
# HAProxy example configuration:
# server v1 app-v1:80 weight 9
# server v2 app-v2:80 weight 1

# Gradually increase v2 traffic
# Monitor metrics, then complete rollout
```

### Health Checks in Production

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

```yaml
# In docker-compose.yml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Logging Strategy

```bash
# Configure logging driver
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Send to centralized logging
docker run -d \
  --log-driver syslog \
  --log-opt syslog-address=tcp://logs.example.com:514 \
  myapp
```

### Backup and Recovery

```bash
# Backup volumes
docker run --rm \
  -v myapp_data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/data-backup-$(date +%Y%m%d).tar.gz -C /source .

# Automated backup script
cat > backup.sh << 'EOF'
#!/bin/bash
VOLUME="myapp_data"
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

docker run --rm \
  -v ${VOLUME}:/source:ro \
  -v ${BACKUP_DIR}:/backup \
  alpine tar czf /backup/backup-${DATE}.tar.gz -C /source .

# Keep only last 7 backups
find ${BACKUP_DIR} -name "backup-*.tar.gz" -mtime +7 -delete
EOF
```

---

## 11. Hands-on Labs

### Lab 1: Containerize a Simple Web Application

**Objective**: Create a Docker image for a Python Flask app

**Steps**:
1. Create application directory
2. Write a simple Flask app
3. Create requirements.txt
4. Write Dockerfile following best practices
5. Build the image
6. Run the container
7. Test the application

**Solution**:

```bash
# Create project structure
mkdir -p flask-app && cd flask-app

# Create app.py
cat > app.py << 'EOF'
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return f"Hello from Flask! Version: {os.environ.get('APP_VERSION', '1.0')}"

@app.route('/health')
def health():
    return {'status': 'healthy'}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
flask==3.0.0
gunicorn==21.2.0
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-alpine

LABEL maintainer="devops-learner@example.com"

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app.py .

# Create non-root user
RUN adduser -D appuser
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -q --spider http://localhost:5000/health || exit 1

ENTRYPOINT ["gunicorn"]
CMD ["--bind", "0.0.0.0:5000", "app:app"]
EOF

# Build image
docker build -t flask-app:1.0 .

# Run container
docker run -d -p 5000:5000 --name flask-demo flask-app:1.0

# Test
curl http://localhost:5000/
curl http://localhost:5000/health

# View logs
docker logs flask-demo
```

### Lab 2: Multi-Container Application with Docker Compose

**Objective**: Deploy a WordPress site with MySQL database

**Steps**:
1. Create docker-compose.yml
2. Configure environment variables
3. Start services
4. Access WordPress
5. Verify database connection
6. Test persistence

**Solution**:

```bash
mkdir -p wordpress-project && cd wordpress-project

# Create .env file
cat > .env << 'EOF'
MYSQL_ROOT_PASSWORD=secure_root_password_123
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser
MYSQL_PASSWORD=secure_wp_password_456
WORDPRESS_DB_HOST=db:3306
EOF

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: wordpress-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wordpress-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

  wordpress:
    image: wordpress:latest
    container_name: wordpress-app
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: ${WORDPRESS_DB_HOST}
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
    volumes:
      - wp_data:/var/www/html
    networks:
      - wordpress-net
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/wp-login.php"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  db_data:
  wp_data:

networks:
  wordpress-net:
    driver: bridge
EOF

# Start services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f

# Access WordPress at http://localhost:8080
# Complete installation wizard

# Verify database connection
docker-compose exec db mysql -u wpuser -p${MYSQL_PASSWORD} wordpress -e "SHOW TABLES;"

# Test persistence
docker-compose down
docker-compose up -d
# WordPress data should still be there
```

### Lab 3: Docker Networking Deep Dive

**Objective**: Understand container networking and service discovery

**Steps**:
1. Create custom networks
2. Deploy containers on different networks
3. Test connectivity
4. Configure DNS resolution
5. Implement network segmentation

**Solution**:

```bash
# Create networks
docker network create frontend-net
docker network create backend-net

# Deploy database (backend only)
docker run -d \
  --name db \
  --network backend-net \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# Deploy API (both networks)
docker run -d \
  --name api \
  --network frontend-net \
  --network backend-net \
  -e DB_HOST=db \
  nginx:alpine

# Deploy web frontend (frontend only)
docker run -d \
  --name web \
  --network frontend-net \
  -p 8080:80 \
  nginx:alpine

# Test connectivity
# From web, can reach api
docker exec web ping -c 3 api

# From api, can reach db
docker exec api ping -c 3 db

# From web, cannot reach db directly (different network)
docker exec web ping -c 3 db  # Should fail

# Connect web to backend temporarily
docker network connect backend-net web
docker exec web ping -c 3 db  # Now works

# Disconnect
docker network disconnect backend-net web
```

### Lab 4: Multi-Stage Build Optimization

**Objective**: Create optimized production image for a React app

**Solution**:

```bash
mkdir -p react-app && cd react-app

# Create Dockerfile with multi-stage build
cat > Dockerfile << 'EOF'
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (including devDependencies for build)
RUN npm ci

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Stage 2: Production
FROM nginx:alpine

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built assets from builder stage
COPY --from=builder /app/build /usr/share/nginx/html

# Expose port
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost/ || exit 1

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
EOF

# Create nginx.conf
cat > nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Disable caching for index.html
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }
}
EOF

# Build image
docker build -t react-app:optimized .

# Compare sizes
docker images | grep react-app

# Typical results:
# Builder stage: ~900MB
# Production image: ~25MB (96% reduction!)
```

### Lab 5: Docker Security Scanning

**Objective**: Learn to scan and secure Docker images

**Solution**:

```bash
# Pull a vulnerable image for testing
docker pull node:14-alpine

# Scan with Docker Scout
docker scout cves node:14-alpine

# Get recommendations
docker scout recommendations node:14-alpine

# Compare with newer version
docker scout cves node:18-alpine

# Install Trivy (alternative scanner)
wget https://github.com/aquasecurity/trivy/releases/download/v0.45.0/trivy_0.45.0_Linux-64bit.deb
sudo dpkg -i trivy_0.45.0_Linux-64bit.deb

# Scan with Trivy
trivy image node:14-alpine

# Generate report
trivy image --format table --output report.html node:14-alpine

# Scan for secrets
trivy image --secret-scanners node:14-alpine

# Create secure Dockerfile
cat > Dockerfile.secure << 'EOF'
FROM node:18-alpine

# Add metadata
LABEL maintainer="security@example.com"

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy and install dependencies
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application
COPY . .

# Change ownership
RUN chown -R nodejs:nodejs /app

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD node healthcheck.js || exit 1

CMD ["node", "server.js"]
EOF

# Build secure image
docker build -f Dockerfile.secure -t secure-app:1.0 .

# Scan secure image
docker scout cves secure-app:1.0
trivy image secure-app:1.0

# Run with security options
docker run -d \
  --name secure-container \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --memory="256m" \
  --cpus="0.5" \
  --pids-limit=50 \
  secure-app:1.0
```

---

## 12. Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: Container Exits Immediately

**Symptoms**: Container starts then stops instantly

**Diagnosis**:
```bash
docker ps -a  # Check exit code
docker logs container_name
docker inspect container_name | grep -A 10 State
```

**Common Causes**:
- Application crashed on startup
- Missing environment variables
- Incorrect command/entrypoint
- Port already in use

**Solutions**:
```bash
# Run interactively to see error
docker run -it --entrypoint /bin/sh myapp

# Check environment variables
docker run -e DEBUG=true myapp

# Override command for debugging
docker run myapp npm run debug
```

#### Issue 2: Cannot Connect to Container

**Symptoms**: Port mapping doesn't work

**Diagnosis**:
```bash
# Check if port is mapped
docker port container_name

# Check if container is listening
docker exec container_name netstat -tlnp

# Check firewall
sudo ufw status
```

**Solutions**:
```bash
# Ensure container is running
docker ps

# Check port mapping syntax
docker run -p HOST_PORT:CONTAINER_PORT myapp

# Check if host port is already used
sudo lsof -i :8080

# Try different port
docker run -p 8081:80 myapp
```

#### Issue 3: Out of Disk Space

**Symptoms**: Docker operations fail with "no space left on device"

**Diagnosis**:
```bash
df -h
docker system df
```

**Solutions**:
```bash
# Remove unused data
docker system prune -a

# Remove unused volumes
docker volume prune

# Remove old images
docker image prune -a

# Find large images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -hr

# Configure log rotation
# Edit /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

#### Issue 4: Slow Image Builds

**Symptoms**: docker build takes too long

**Diagnosis**:
```bash
# Check build output for slow steps
docker build -t myapp . 2>&1 | tee build.log

# Check layer caching
docker history myapp
```

**Solutions**:
```dockerfile
# Order instructions by change frequency
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./  # Changes less frequently
RUN npm ci
COPY . .  # Changes frequently
RUN npm run build
```

```bash
# Use build cache
docker build --cache-from myapp:latest -t myapp:new .

# Use multi-stage builds
# See Lab 4 example

# Use .dockerignore
# Exclude node_modules, .git, etc.
```

#### Issue 5: Container Cannot Resolve DNS

**Symptoms**: "Temporary failure in name resolution"

**Diagnosis**:
```bash
docker exec container_name nslookup google.com
docker exec container_name cat /etc/resolv.conf
```

**Solutions**:
```bash
# Specify DNS servers
docker run --dns 8.8.8.8 --dns 8.8.4.4 myapp

# Configure daemon-wide DNS
# Edit /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}

# Restart Docker
sudo systemctl restart docker
```

#### Issue 6: Permission Denied Errors

**Symptoms**: "Permission denied" when accessing files

**Diagnosis**:
```bash
docker exec container_name whoami
docker exec container_name ls -la /problematic/path
```

**Solutions**:
```dockerfile
# Don't run as root
RUN useradd -r -u 1001 appuser
USER appuser
```

```bash
# Fix volume permissions
docker run -v myvolume:/data myapp
docker exec container_name chown -R appuser:appgroup /data

# Or set permissions on host
sudo chown -R 1001:1001 /host/path
```

#### Issue 7: High Memory Usage

**Symptoms**: Container using too much memory, OOM kills

**Diagnosis**:
```bash
docker stats
docker inspect container_name | grep -A 10 HostConfig
```

**Solutions**:
```bash
# Set memory limits
docker run -m 512m --memory-swap=512m myapp

# Investigate memory leaks
docker exec container_name top
docker exec container_name free -m

# Optimize application
# Check for memory leaks in code
```

#### Issue 8: Docker Daemon Not Responding

**Symptoms**: "Cannot connect to the Docker daemon"

**Diagnosis**:
```bash
systemctl status docker
journalctl -u docker -n 50
```

**Solutions**:
```bash
# Restart Docker
sudo systemctl restart docker

# Check disk space
df -h /var/lib/docker

# Check logs
sudo journalctl -u docker -f

# Reinstall if necessary
sudo apt-get remove docker docker-engine
sudo apt-get install docker-ce
```

### Debugging Tools

```bash
# Enter running container
docker exec -it container_name bash

# Inspect container metadata
docker inspect container_name

# View live resource usage
docker stats container_name

# View process tree
docker top container_name

# View filesystem changes
docker diff container_name

# Export container for analysis
docker export container_name > container.tar
tar -tvf container.tar

# Network debugging
docker exec container_name ping google.com
docker exec container_name curl -v http://api:3000
docker exec container_name netstat -tlnp

# Log analysis
docker logs --tail 1000 container_name
docker logs --since 2024-01-01T00:00:00 container_name
docker logs -f container_name 2>&1 | grep ERROR
```

---

## 13. Quiz & Knowledge Check

### Questions

**Q1**: What is the main difference between a Docker image and a container?
- A) Images are writable, containers are read-only
- B) Images are read-only templates, containers are runnable instances
- C) Images run on Linux, containers run on Windows
- D) There is no difference

**Q2**: Which Dockerfile instruction creates a new layer in the image?
- A) Only RUN
- B) Only COPY and ADD
- C) Every instruction except ARG and ENV
- D) Only FROM and RUN

**Q3**: What happens when you run `docker run -d -p 8080:80 nginx`?
- A) Nginx runs on port 8080 inside the container
- B) Port 8080 on host maps to port 80 in container
- C) Port 80 on host maps to port 8080 in container
- D) Both ports 80 and 8080 are exposed on the host

**Q4**: Which network driver allows containers to communicate across multiple Docker hosts?
- A) bridge
- B) host
- C) overlay
- D) macvlan

**Q5**: What is the purpose of a multi-stage build?
- A) To create multiple containers from one Dockerfile
- B) To reduce final image size by separating build and runtime environments
- C) To build images for multiple architectures
- D) To run multiple applications in one container

**Q6**: Which command removes all stopped containers, unused networks, and dangling images?
- A) docker clean
- B) docker system prune
- C) docker remove all
- D) docker cleanup

**Q7**: How do you ensure a container doesn't run as root?
- A) Use --non-root flag
- B) Add USER instruction in Dockerfile
- C) It's automatic in Docker
- D) Use --user=1000 in docker run

**Q8**: What does the HEALTHCHECK instruction do?
- A) Automatically restarts unhealthy containers
- B) Runs a command to check container health and updates status
- C) Monitors resource usage
- D) Validates the Dockerfile syntax

**Q9**: Which volume type is managed by Docker and persists after container removal?
- A) Bind mount
- B) tmpfs
- C) Volume
- D) Anonymous mount

**Q10**: In Docker Compose, how do you specify that one service must start before another?
- A) using order:
- B) using requires:
- C) using depends_on:
- D) using priority:

**Q11**: What is the default network driver in Docker?
- A) host
- B) none
- C) bridge
- D) overlay

**Q12**: Which command shows the layers of a Docker image?
- A) docker image layers
- B) docker image history
- C) docker image inspect
- D) docker image tree

### Answers

1. **B** - Images are read-only templates, containers are runnable instances with a writable layer
2. **C** - Every instruction (except ARG, ENV, LABEL, FROM) creates a new layer
3. **B** - Host port 8080 maps to container port 80
4. **C** - Overlay network enables multi-host communication
5. **B** - Multi-stage builds reduce image size by separating build and runtime
6. **B** - docker system prune removes unused data
7. **B** - USER instruction in Dockerfile switches to non-root user
8. **B** - HEALTHCHECK runs a command periodically to check health status
9. **C** - Volumes are managed by Docker and persist independently
10. **C** - depends_on: ensures service startup order
11. **C** - Bridge is the default network driver
12. **B** - docker image history shows layer information

### Score Interpretation
- **12/12**: Docker Master! 🏆
- **10-11**: Excellent understanding! 🎯
- **7-9**: Good foundation, review weak areas 📚
- **4-6**: Need more practice with concepts 🔄
- **0-3**: Re-read the module and retry labs 📖

---

## 🎓 Next Steps

Congratulations on completing the Docker module! You now have solid containerization skills.

### What You've Learned:
✅ Container fundamentals and Docker architecture
✅ Creating and optimizing Docker images
✅ Managing containers and their lifecycle
✅ Docker networking and storage
✅ Multi-container applications with Compose
✅ Security best practices
✅ Production deployment strategies
✅ Troubleshooting techniques

### Practice Projects:
1. Containerize your own application
2. Create a CI/CD pipeline that builds Docker images
3. Migrate a legacy app to containers
4. Implement container security scanning in your workflow
5. Build a microservices demo with Docker Compose

### Ready for the Next Module?
Proceed to **Module 06: Kubernetes Orchestration** to learn:
- Container orchestration at scale
- Pods, Deployments, and Services
- Auto-scaling and self-healing
- Helm charts for package management
- Production Kubernetes clusters

---

## 📚 Additional Resources

### Books
- "Docker Deep Dive" by Nigel Poulton
- "The Docker Book" by James Turnbull
- "Docker in Action" by Jeff Nickoloff

### Documentation
- [Official Docker Docs](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)

### Practice Platforms
- [Play with Docker](https://labs.play-with-docker.com/)
- [Katacoda Docker Scenarios](https://www.katacoda.com/courses/docker)

### Tools to Explore
- **Portainer**: Docker GUI management
- **Watchtower**: Automatic container updates
- **Dive**: Analyze Docker image layers
- **Trivy**: Vulnerability scanner
- **Hadolint**: Dockerfile linter

### Communities
- r/docker on Reddit
- Docker Community Forums
- Docker Slack workspace
- Local Docker meetups

---

*"Containers are the building blocks of modern applications. Master them, and you master the future of software deployment."*
