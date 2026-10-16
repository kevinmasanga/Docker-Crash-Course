# Docker Learning Guide: From Basics to Advanced

A comprehensive, structured reference and handbook for mastering Docker. This guide starts with core mental models and takes you step-by-step through fundamental container operations, image creation, storage, networking, multi-container orchestration, and system maintenance.

---

## Table of Contents
1. [Core Concepts & Mental Model](#1-core-concepts--mental-model)
2. [System Information & Verification](#2-system-information--verification)
3. [Container Lifecycle & Basic Commands](#3-container-lifecycle--basic-commands)
4. [Inspecting & Interacting with Containers](#4-inspecting--interacting-with-containers)
5. [Working with Images](#5-working-with-images)
6. [Building Custom Images with Dockerfile](#6-building-custom-images-with-dockerfile)
7. [Storage & Data Persistence (Volumes & Bind Mounts)](#7-storage--data-persistence-volumes--bind-mounts)
8. [Docker Networking](#8-docker-networking)
9. [Registries & Repositories](#9-registries--repositories)
10. [Docker Compose (Multi-Container Apps)](#10-docker-compose-multi-container-apps)
11. [System Cleanup & Disk Maintenance](#11-system-cleanup--disk-maintenance)
12. [Quick Command Cheatsheet](#12-quick-command-cheatsheet)

---

## 1. Core Concepts & Mental Model

Before running commands, it is crucial to understand Docker's four foundational building blocks:

```
+-------------------------------------------------------------+
| Registry (e.g., Docker Hub, GitHub Container Registry)       |
|   └── Repository (e.g., "masangakevin/my-express-app")      |
|         ├── Tag: 1.0 (Image)                                |
|         └── Tag: latest (Image)                             |
+-------------------------------------------------------------+
                                │
                        docker pull / push
                                ▼
+─────────────────────────────────────────────────────────────+
| Local Docker Host                                           |
|   ├── Image: Read-only blueprint / template                 |
|   │     │                                                   |
|   │     ▼ docker run                                        |
|   └── Container: Running, writable instance of an Image     |
+─────────────────────────────────────────────────────────────+
```

### Key Distinctions
* **Image vs. Container**:
  * **Image**: An immutable (read-only) snapshot containing the application code, libraries, dependencies, and environment configuration. Think of it like a class in programming or a cookie cutter.
  * **Container**: A running, isolated process instantiated from an image with a thin, writable layer on top. Think of it like an object instance or the actual cookie.
* **Registry vs. Repository**:
  * **Registry**: A hosted service storing and distributing container images (e.g., Docker Hub, GitHub Packages `ghcr.io`, AWS ECR).
  * **Repository**: A collection of related container images within a registry sharing the same name but differentiated by tags (e.g., `node:18-alpine`, `node:20-alpine`, `node:latest`).
* **Container vs. Virtual Machine**:
  * VMs virtualize the hardware and include a full Guest OS (heavy, slow boot, gigabytes in size).
  * Containers share the host kernel and isolate processes using Linux namespaces and cgroups (lightweight, boots in milliseconds, megabytes in size).

---

## 2. System Information & Verification

Start by checking your Docker daemon and client installation status:

```bash
# Check installed Docker version
docker --version

# View comprehensive system-wide information (containers running/paused/stopped, storage driver, CPU/RAM limits)
docker info

# Run canonical test image to confirm your engine can pull and run containers
docker run hello-world
```

---

## 3. Container Lifecycle & Basic Commands

### `docker run` vs `docker start`
> **Important Distinction**:
> * `docker run` creates a **brand-new** container from an image and starts it.
> * `docker start` restarts an **existing, stopped** container without creating a new one.

### Running Containers (`docker run`)

```bash
# Basic run: pulls 'nginx' if not local, creates container, runs in foreground
docker run nginx

# Run in background (detached mode)
docker run -d nginx

# Assign a custom, memorable name to your container
docker run -d --name my-web-server nginx

# Map ports: -p <HOST_PORT>:<CONTAINER_PORT>
# Forward host port 8080 to container's internal port 80
docker run -d --name my-web-server -p 8080:80 nginx

# Run interactively with a pseudo-TTY (ideal for shells/terminals)
docker run -it ubuntu bash

# Automatically remove container when it exits (prevents stopped container clutter)
docker run --rm alpine echo "Hello from temporary container"

# Pass environment variables
docker run -d --name my-db -e MYSQL_ROOT_PASSWORD=secret mysql:8

# Combining flags (common real-world pattern):
docker run -d \
  --name webapp \
  -p 3000:3000 \
  --restart unless-stopped \
  -e NODE_ENV=production \
  node:19-alpine
```

### Listing Containers (`docker ps`)

```bash
# List only running containers
docker ps

# List ALL containers (both running and stopped)
docker ps -a

# Show only container IDs (very useful for piping into other commands)
docker ps -q
docker ps -aq

# View disk usage and size of running containers
docker ps -s

# Filter containers (e.g., by status)
docker ps -a --filter "status=exited"
```

### Starting, Stopping, and Restarting

```bash
# Start an existing stopped container by Name or ID
docker start my-web-server

# Gracefully stop a running container (sends SIGTERM, then SIGKILL after 10s grace period)
docker stop my-web-server

# Immediately kill a container (sends SIGKILL)
docker kill my-web-server

# Restart a container
docker restart my-web-server

# Pause all processes within a container (freezes execution without stopping)
docker pause my-web-server

# Unpause frozen container
docker unpause my-web-server
```

### Removing Containers (`docker rm`)

```bash
# Remove a stopped container
docker rm my-web-server

# Force remove a running container (stops and deletes in one command)
docker rm -f my-web-server

# Remove all stopped containers at once
docker container prune -f

# Advanced bash one-liner: stop and remove all containers on the system
docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
```

---

## 4. Inspecting & Interacting with Containers

Once a container is running, use these commands to inspect what is happening inside.

### Viewing Logs (`docker logs`)

```bash
# View standard output and error logs
docker logs my-web-server

# Follow logs live (stream logs like 'tail -f')
docker logs -f my-web-server

# Show only the last 50 lines
docker logs --tail 50 my-web-server

# Follow logs with timestamps
docker logs -f --timestamps --tail 20 my-web-server
```

### Executing Commands Inside a Running Container (`docker exec`)

```bash
# Open an interactive shell inside a running container
# Use bash if available:
docker exec -it my-web-server bash

# For Alpine-based containers (which do not ship with bash by default), use sh:
docker exec -it my-web-server sh

# Run a one-off command without opening a persistent shell
docker exec my-web-server ls -la /app
docker exec my-web-server cat /etc/hosts
```

### Monitoring Processes & Resource Usage

```bash
# Stream live resource usage (CPU %, Memory, Network I/O, Block I/O) for all containers
docker stats

# Stream resource usage for a specific container without continuous loop
docker stats --no-stream my-web-server

# Display running processes inside a container (like `ps aux`)
docker top my-web-server

# Inspect detailed metadata (JSON) such as IP address, mounts, environment variables
docker inspect my-web-server

# Extract specific JSON field using Go template formatting (e.g., get container IP)
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-web-server
```

### Copying Files Between Host and Container (`docker cp`)

```bash
# Copy a file from host machine into a container
docker cp ./package.json my-web-server:/app/package.json

# Copy a file/folder from container back to host machine
docker cp my-web-server:/app/logs ./container-logs
```

---

## 5. Working with Images

Images are the blueprints of your containers.

```bash
# Search for official images on Docker Hub
docker search nginx

# Pull an image from Docker Hub to your local cache
docker pull node:19-alpine

# List locally cached images
docker images
# or modern syntax:
docker image ls

# Inspect image details and layers
docker image inspect node:19-alpine

# View layer history of an image and commands that created them
docker history node:19-alpine

# Tag an existing image with a new name or repository target
# Format: docker tag <SOURCE_IMAGE>:<TAG> <TARGET_REPOSITORY>:<NEW_TAG>
docker tag node:19-alpine my-registry-user/my-node:v1

# Remove an image from local machine (must not be used by any existing container)
docker rmi node:19-alpine
# or:
docker image rm node:19-alpine

# Remove unused/dangling images
docker image prune
```

---

## 6. Building Custom Images with Dockerfile

A **Dockerfile** is a text script containing sequential instructions that Docker executes to assemble an image.

### Essential Dockerfile Instructions

| Instruction | Purpose | Best Practice Example |
| :--- | :--- | :--- |
| `FROM` | Sets the base image (first instruction) | `FROM node:19-alpine` |
| `WORKDIR` | Sets the working directory for subsequent instructions | `WORKDIR /app` |
| `COPY` | Copies files/directories from host to image filesystem | `COPY package.json ./` |
| `ADD` | Like COPY, but unpacks tarballs and supports remote URLs | Use `COPY` unless auto-extracting `.tar.gz` |
| `RUN` | Executes commands during image build time (installs deps) | `RUN npm install --production` |
| `ENV` | Sets persistent environment variables | `ENV PORT=3000` |
| `EXPOSE` | Documents the port the container intends to listen on | `EXPOSE 3000` |
| `CMD` | Default command when container starts (can be overridden) | `CMD ["node", "src/server.js"]` |
| `ENTRYPOINT` | Configures a container that will run as an executable | `ENTRYPOINT ["node"]` |

### `CMD` vs `ENTRYPOINT`
* `CMD`: Provides default arguments. Can be completely replaced when running `docker run <image> <new_command>`.
* `ENTRYPOINT`: Fixed binary that always runs. Arguments passed via `docker run` are appended to it.
* **Best practice**: Use `CMD ["node", "src/server.js"]` for standard web servers.

---

### Hands-on Example: Dockerizing This Express Project

Here is how to properly structure your `Dockerfile` for the Node/Express app in this directory:

#### 1. `.dockerignore`
Always create a `.dockerignore` file next to your Dockerfile to avoid copying unnecessary files into the image:
```text
node_modules
npm-debug.log
.git
.gitignore
.env
Readme.md
```

#### 2. `Dockerfile`
Optimize layer caching by copying dependency manifests before application code:

```dockerfile
# 1. Choose a minimal, stable base image
FROM node:19-alpine

# 2. Set the working directory inside the container
WORKDIR /app

# 3. Copy package manifests first (leverages Docker layer cache)
COPY package.json pnpm-lock.yaml* ./

# 4. Install dependencies
RUN npm install

# 5. Copy the rest of the application source code
COPY src/ ./src/

# 6. Expose the port your server listens on (documentary)
EXPOSE 3000

# 7. Define the default startup command
CMD ["node", "src/server.js"]
```

#### 3. Building and Running the Image

```bash
# Build the image and tag it as 'my-express-app:1.0'
# The '.' at the end specifies the build context (current directory)
docker build -t my-express-app:1.0 .

# Build without using cached layers (forces fresh installation)
docker build --no-cache -t my-express-app:1.0 .

# Run your new custom container
docker run -d --name express-api -p 3000:3000 my-express-app:1.0

# Verify it works
curl http://localhost:3000
```

---

## 7. Storage & Data Persistence (Volumes & Bind Mounts)

By default, container filesystems are **ephemeral**: all data written inside a container is lost when the container is deleted.

Docker provides two primary ways to persist data:
1. **Named Volumes**: Managed by Docker in a dedicated directory (`/var/lib/docker/volumes/`). Best for production, databases, and performance.
2. **Bind Mounts**: Maps a specific directory on your host machine into the container. Best for local development and live code reloading.

```
       Host Machine                      Container
┌───────────────────────────┐      ┌───────────────────┐
│ Named Volume:             │      │                   │
│ /var/lib/docker/volumes/  ├─────►│ /var/lib/mysql    │
│                           │      │                   │
│ Bind Mount:               │      │                   │
│ /home/user/project/src    ├─────►│ /app/src          │
└───────────────────────────┘      └───────────────────┘
```

### Named Volumes

```bash
# Create a volume
docker volume create db_data

# List volumes
docker volume ls

# Inspect volume location and metadata
docker volume inspect db_data

# Mount volume to container: -v <VOLUME_NAME>:<CONTAINER_PATH>
docker run -d --name database -v db_data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=secret mysql:8

# Remove an unused volume
docker volume rm db_data

# Delete all unused volumes
docker volume prune -f
```

### Bind Mounts (Hot-Reloading in Development)

Mount your current source directory directly into the container so file edits immediately reflect inside:

```bash
# Mount host directory into container: -v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH>
# Using $(pwd) grabs your current working directory
docker run -d \
  --name dev-server \
  -p 3000:3000 \
  -v $(pwd)/src:/app/src \
  my-express-app:1.0
```

---

## 8. Docker Networking

Docker containers can talk to each other and to the outside world through virtual networks.

### Built-in Network Drivers
* **`bridge`**: Default network driver for standalone containers on the same host.
* **`host`**: Removes network isolation; container shares host's network stack directly.
* **`none`**: Completely isolates container from all networking.
* **`overlay`**: Multi-host networking across Docker Swarm nodes.

### Managing Networks

```bash
# List available networks
docker network ls

# Create a custom user-defined bridge network
docker network create app-network

# Run containers on the custom network
# Note: Containers on the same custom network can resolve each other by NAME!
docker run -d --name backend-db --network app-network mongo:6
docker run -d --name backend-api --network app-network -p 3000:3000 my-express-app:1.0

# Inside backend-api, you can connect to the database directly using hostname 'backend-db':
# mongodb://backend-db:27017/mydb

# Connect a running container to an additional network
docker network connect app-network my-web-server

# Disconnect container from network
docker network disconnect app-network my-web-server

# Inspect network to see attached containers and their assigned IPs
docker network inspect app-network

# Remove a network
docker network rm app-network
```

---

## 9. Registries & Repositories

### Definitions Review
* **Registry**: The service hosting images (e.g., `docker.io`, `ghcr.io`).
* **Repository**: A named storage group within a registry for versions of a specific image.

### Authenticating and Pushing Images

```bash
# Log in to Docker Hub (or a private registry)
docker login
# To log in to a custom registry:
# docker login ghcr.io

# 1. Tag the local image with your registry username and repository name
# Format: docker tag <LOCAL_IMAGE> <USERNAME>/<REPO_NAME>:<TAG>
docker tag my-express-app:1.0 masangakevin/my-express-app:1.0
docker tag my-express-app:1.0 masangakevin/my-express-app:latest

# 2. Push image to registry
docker push masangakevin/my-express-app:1.0
docker push masangakevin/my-express-app:latest

# 3. Anyone can now pull and run your image anywhere
docker pull masangakevin/my-express-app:1.0

# Log out when finished
docker logout
```

---

## 10. Docker Compose (Multi-Container Apps)

Docker Compose allows you to define and manage multi-container applications in a single YAML configuration file (`compose.yaml` or `docker-compose.yml`).

### Example `compose.yaml`

```yaml
services:
  api:
    build: .
    container_name: express_api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - PORT=3000
      - MONGO_URL=mongodb://db:27017/app
    volumes:
      - ./src:/app/src
    depends_on:
      - db
    restart: always

  db:
    image: mongo:6
    container_name: mongo_db
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

### Essential Compose Commands

```bash
# Start all services in the background (builds if not yet built)
docker compose up -d

# Force rebuild of images before starting
docker compose up -d --build

# View aggregated logs from all services in compose file
docker compose logs -f

# View logs for a specific service only
docker compose logs -f api

# List status of all compose services
docker compose ps

# Execute a command inside a compose service
docker compose exec api sh

# Stop and remove containers, networks, and internal resources created by up
docker compose down

# Stop and remove everything INCLUDING named volumes (resets databases)
docker compose down -v
```

---

## 11. System Cleanup & Disk Maintenance

Docker caches images, containers, volumes, and build layers. Over time, these can consume significant disk space.

```bash
# Check how much disk space Docker is consuming
docker system df

# Check detailed breakdown of items taking up space
docker system df -v

# Remove stopped containers, dangling images, unused networks, and build caches
docker system prune

# Comprehensive cleanup (removes all unused images, stopped containers, and unused volumes)
# CAUTION: This deletes all images not currently attached to a running container!
docker system prune -a --volumes -f
```

---

## 12. Quick Command Cheatsheet

### Container Lifecycle
| Command | Description |
| :--- | :--- |
| `docker run -d -p 80:80 --name web nginx` | Run container in background with port forwarding |
| `docker run -it ubuntu bash` | Run interactive container attached to terminal |
| `docker ps` | List active running containers |
| `docker ps -a` | List all containers (active & stopped) |
| `docker start <name/id>` | Start an already created, stopped container |
| `docker stop <name/id>` | Gracefully stop running container |
| `docker restart <name/id>` | Restart container |
| `docker rm <name/id>` | Delete stopped container |
| `docker rm -f <name/id>` | Force stop and delete running container |

### Diagnostics & Debugging
| Command | Description |
| :--- | :--- |
| `docker logs -f <name/id>` | Follow real-time log output |
| `docker exec -it <name/id> sh` | Open terminal inside running container |
| `docker top <name/id>` | Show active processes in container |
| `docker stats` | Live CPU, memory, and network usage monitor |
| `docker inspect <name/id>` | Low-level configuration JSON details |
| `docker diff <name/id>` | Inspect changes to files on container filesystem |

### Image Management
| Command | Description |
| :--- | :--- |
| `docker images` | List local images |
| `docker pull <image>:<tag>` | Download image from registry |
| `docker build -t <name>:<tag> .` | Build image from Dockerfile in current directory |
| `docker tag <old> <new>` | Create a new tag pointing to an image |
| `docker rmi <image>` | Delete local image |
| `docker history <image>` | Show build steps and layer sizes |

### Volumes & Networks
| Command | Description |
| :--- | :--- |
| `docker volume create <name>` | Create persistent volume |
| `docker volume ls` | List all persistent volumes |
| `docker network create <name>` | Create custom bridge network |
| `docker network ls` | List virtual networks |
| `docker network connect <net> <c>` | Connect container to network |

### Maintenance
| Command | Description |
| :--- | :--- |
| `docker system df` | View disk usage |
| `docker container prune` | Remove all stopped containers |
| `docker image prune` | Remove unused/dangling images |
| `docker system prune -a` | Deep clean all unused resources |
