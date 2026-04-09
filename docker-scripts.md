# Docker Scripts & Commands

A curated collection of useful Docker commands and scripts for container management, debugging, and deployment.

---

## Table of Contents

- [Images](#images)
- [Containers](#containers)
- [Volumes](#volumes)
- [Networks](#networks)
- [Docker Compose](#docker-compose)
- [Cleanup](#cleanup)
- [Debugging & Inspection](#debugging--inspection)
- [Registry & Hub](#registry--hub)

---

## Images

### Build an Image

```bash
docker build -t my-app:latest .
```

**With a specific Dockerfile:**
```bash
docker build -f Dockerfile.prod -t my-app:prod .
```

**With build args:**
```bash
docker build --build-arg NODE_ENV=production -t my-app:latest .
```

---

### List Images

```bash
docker images
```

**Filter by name:**
```bash
docker images | grep my-app
```

---

### Remove an Image

```bash
docker rmi my-app:latest
```

**Force remove:**
```bash
docker rmi -f my-app:latest
```

---

### Tag an Image

```bash
docker tag my-app:latest my-app:1.0.0
```

---

### Pull an Image

```bash
docker pull node:20-alpine
```

---

### Inspect an Image

```bash
docker inspect my-app:latest
```

**Get just the environment variables:**
```bash
docker inspect my-app:latest | jq '.[0].Config.Env'
```

---

## Containers

### Run a Container

```bash
docker run -d --name my-container -p 3000:3000 my-app:latest
```

**Options:**
- `-d` = detached (background)
- `--name` = container name
- `-p host:container` = port mapping
- `--rm` = remove container when it exits
- `-e KEY=VALUE` = set environment variable

---

### Run Interactively

```bash
docker run -it --rm node:20-alpine sh
```

---

### List Running Containers

```bash
docker ps
```

**All containers (including stopped):**
```bash
docker ps -a
```

---

### Stop / Start / Restart

```bash
docker stop my-container
docker start my-container
docker restart my-container
```

---

### Execute Command in Running Container

```bash
docker exec -it my-container sh
```

**Run a one-off command:**
```bash
docker exec my-container ls /app
```

---

### View Container Logs

```bash
docker logs my-container
```

**Follow logs in real-time:**
```bash
docker logs -f my-container
```

**Last 100 lines:**
```bash
docker logs --tail 100 my-container
```

---

### Copy Files To/From Container

```bash
# Host → Container
docker cp ./file.txt my-container:/app/file.txt

# Container → Host
docker cp my-container:/app/file.txt ./file.txt
```

---

### View Container Resource Usage

```bash
docker stats
```

**Single container:**
```bash
docker stats my-container
```

---

## Volumes

### Create a Volume

```bash
docker volume create my-data
```

---

### List Volumes

```bash
docker volume ls
```

---

### Mount a Volume

```bash
docker run -d -v my-data:/app/data my-app:latest
```

**Bind mount (local directory):**
```bash
docker run -d -v $(pwd):/app my-app:latest
```

---

### Inspect a Volume

```bash
docker volume inspect my-data
```

---

### Remove a Volume

```bash
docker volume rm my-data
```

---

## Networks

### List Networks

```bash
docker network ls
```

---

### Create a Network

```bash
docker network create my-network
```

---

### Connect Container to Network

```bash
docker network connect my-network my-container
```

---

### Run Container on a Network

```bash
docker run -d --network my-network --name my-container my-app:latest
```

---

### Inspect a Network

```bash
docker network inspect my-network
```

---

## Docker Compose

### Start Services

```bash
docker compose up -d
```

**Rebuild images before starting:**
```bash
docker compose up -d --build
```

---

### Stop Services

```bash
docker compose down
```

**Also remove volumes:**
```bash
docker compose down -v
```

---

### View Logs

```bash
docker compose logs -f
```

**Specific service:**
```bash
docker compose logs -f app
```

---

### Scale a Service

```bash
docker compose up -d --scale worker=3
```

---

### Execute Command in a Service

```bash
docker compose exec app sh
```

---

### Pull Latest Images

```bash
docker compose pull
```

---

### View Running Services

```bash
docker compose ps
```

---

## Cleanup

### Remove Stopped Containers

```bash
docker container prune
```

---

### Remove Unused Images

```bash
docker image prune
```

**Remove all unused images (not just dangling):**
```bash
docker image prune -a
```

---

### Remove Unused Volumes

```bash
docker volume prune
```

---

### Remove Unused Networks

```bash
docker network prune
```

---

### Full System Cleanup

Remove all stopped containers, unused networks, dangling images, and build cache.

```bash
docker system prune
```

**Include unused images and volumes:**
```bash
docker system prune -a --volumes
```

> ⚠️ Use with caution, this removes a lot of data.

---

### Check Disk Usage

```bash
docker system df
```

---

## Debugging & Inspection

### Get Container IP Address

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container
```

---

### Check Container Environment Variables

```bash
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' my-container
```

---

### View Container Processes

```bash
docker top my-container
```

---

### Check Why Container Exited

```bash
docker inspect my-container --format='{{.State.ExitCode}} {{.State.Error}}'
```

---

### Run a Temporary Debug Container in Same Network

```bash
docker run --rm -it --network container:my-container nicolaka/netshoot
```

---

### Export Container Filesystem

```bash
docker export my-container > container.tar
```

---

## Registry & Hub

### Login to Docker Hub

```bash
docker login
```

**Login to private registry:**
```bash
docker login registry.example.com
```

---

### Push an Image

```bash
docker tag my-app:latest username/my-app:latest
docker push username/my-app:latest
```

---

### Search Docker Hub

```bash
docker search nginx
```

---

### Save Image to File

```bash
docker save -o my-app.tar my-app:latest
```

**Load it back:**
```bash
docker load -i my-app.tar
```

---

## Tips

### Multi-stage Build Example

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --omit=dev
CMD ["node", "dist/index.js"]
```
