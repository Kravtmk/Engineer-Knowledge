# Docker

Quick reference for everyday Docker work.

---

## Check Docker

```bash
docker --version
docker info
systemctl status docker
```

Check Docker Compose:

```bash
docker compose version
```

---

# Images

List images:

```bash
docker images
```

Build image:

```bash
docker build -t myapp:v1 .
```

Build without cache:

```bash
docker build --no-cache -t myapp:v1 .
```

Remove image:

```bash
docker rmi myapp:v1
```

Inspect image:

```bash
docker inspect myapp:v1
```

Image history:

```bash
docker history myapp:v1
```

---

# Containers

Running containers:

```bash
docker ps
```

All containers:

```bash
docker ps -a
```

Run container:

```bash
docker run -d \
  --name myapp \
  -p 8001:8000 \
  myapp:v1
```

Stop:

```bash
docker stop myapp
```

Start:

```bash
docker start myapp
```

Restart:

```bash
docker restart myapp
```

Remove:

```bash
docker rm myapp
```

Force remove:

```bash
docker rm -f myapp
```

---

# Logs

Show logs:

```bash
docker logs myapp
```

Follow logs:

```bash
docker logs -f myapp
```

Last 100 lines:

```bash
docker logs --tail 100 myapp
```

---

# Enter Container

Open shell:

```bash
docker exec -it myapp /bin/bash
```

For minimal images:

```bash
docker exec -it myapp /bin/sh
```

Run one command:

```bash
docker exec myapp env
```

---

# Ports

Show published ports:

```bash
docker port myapp
```

Example:

```text
VM :8001
   ↓
Container :8000
```

Run with port mapping:

```bash
docker run -d \
  -p 8001:8000 \
  myapp:v1
```

Check host ports:

```bash
ss -lntp
```

---

# Networks

List networks:

```bash
docker network ls
```

Inspect network:

```bash
docker network inspect bridge
```

Create network:

```bash
docker network create web_network
```

Inspect:

```bash
docker network inspect web_network
```

Remove:

```bash
docker network rm web_network
```

Connect container:

```bash
docker network connect web_network myapp
```

---

# Volumes

List volumes:

```bash
docker volume ls
```

Create:

```bash
docker volume create mydata
```

Inspect:

```bash
docker volume inspect mydata
```

Run container with volume:

```bash
docker run -d \
  --name myapp \
  -v mydata:/data \
  myapp:v1
```

Remove volume:

```bash
docker volume rm mydata
```

Be careful:

> Container removal does not automatically mean application data should be removed.

---

# Environment Variables

Pass variable:

```bash
docker run -d \
  -e APP_ENV=production \
  myapp:v1
```

Check:

```bash
docker exec myapp env
```

Do not put passwords or tokens directly into Dockerfiles or public repositories.

---

# Inspect

Full container information:

```bash
docker inspect myapp
```

Container processes:

```bash
docker top myapp
```

Resource usage:

```bash
docker stats
```

Disk usage:

```bash
docker system df
```

---

# Copy Files

Host → container:

```bash
docker cp config.txt myapp:/app/config.txt
```

Container → host:

```bash
docker cp myapp:/app/log.txt .
```

---

# Cleanup

Stopped containers:

```bash
docker container prune
```

Unused images:

```bash
docker image prune
```

Unused networks:

```bash
docker network prune
```

Unused resources:

```bash
docker system prune
```

Check before using:

```bash
docker system df
```

⚠️ Be careful with cleanup commands on production hosts.

---

# Troubleshooting

## Container is not running

```bash
docker ps -a
```

Then:

```bash
docker logs myapp
```

Inspect:

```bash
docker inspect myapp
```

---

## Application is not reachable

Check container:

```bash
docker ps
```

Check port mapping:

```bash
docker port myapp
```

Check inside container:

```bash
docker exec -it myapp /bin/sh
```

Check host:

```bash
ss -lntp
```

Think layer by layer:

```text
Client
  ↓
Host Port
  ↓
Docker Port Mapping
  ↓
Container Port
  ↓
Application
```

---

## Container exits immediately

```bash
docker ps -a
docker logs myapp
```

Check exit code:

```bash
docker inspect myapp --format='{{.State.ExitCode}}'
```

---

## Permission denied

Check Docker socket:

```bash
ls -l /var/run/docker.sock
```

Check groups:

```bash
groups
```

Docker group membership normally requires a new login session after:

```bash
sudo usermod -aG docker ops
```

⚠️ Membership in the `docker` group effectively gives root-level control over the host.

---

# Docker Architecture

```text
docker CLI
    ↓
/var/run/docker.sock
    ↓
dockerd
    ↓
containerd
    ↓
runc
    ↓
Linux Kernel
```

For daily troubleshooting, think:

```text
CLI
 ↓
Docker daemon
 ↓
Image
 ↓
Container
 ↓
Network / Volume
 ↓
Application
```

---

# Install — Ubuntu

Remove conflicting packages if necessary:

```bash
sudo apt remove -y \
  docker.io \
  docker-compose \
  docker-compose-v2 \
  docker-doc \
  podman-docker \
  containerd \
  runc
```

Install prerequisites:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Add Docker's official repository and install:

```text
docker-ce
docker-ce-cli
containerd.io
docker-buildx-plugin
docker-compose-plugin
```

After installation:

```bash
docker --version
docker compose version
```

Add user to Docker group if required:

```bash
sudo usermod -aG docker ops
```

Log out and reconnect:

```bash
exit
ssh vm1
```

Verify:

```bash
groups
docker ps
docker run hello-world
```

> For the exact current repository installation commands, use the official Docker documentation rather than copying an old repository key configuration.

---

# Daily Docker Flow

```text
Dockerfile
    ↓
docker build
    ↓
Image
    ↓
docker run
    ↓
Container
    ↓
docker ps
    ↓
docker logs
    ↓
docker exec
```

## Remember

```text
Dockerfile = how to BUILD
Image      = packaged artifact
Container  = running instance of image
Volume     = persistent data
Network    = container communication
```

Docker Compose has its own note:

```text
20.Areas/20.Tools/03.Docker_Compose.md
```