# Docker Practice

## Goal

Learn Docker by building, running, changing, breaking and troubleshooting one application.

Final flow:

```text
app.py
   ↓
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
Port
   ↓
Application
```

Then:

```text
v1 → v2 → rollback → troubleshooting
```

---

# Prerequisites

You need:

```text
Linux VM
Docker installed
Docker Compose plugin installed
Git
curl
```

Check:

```bash
docker --version
docker compose version
docker ps
```

If Docker permissions were just changed:

```bash
groups
```

You may need to log out and reconnect.

---

# Level 1 — Verify Docker

Check Docker service:

```bash
systemctl status docker
```

Run test container:

```bash
docker run hello-world
```

Check containers:

```bash
docker ps
docker ps -a
```

### Checkpoint

- [ ] Docker service is active
- [ ] `hello-world` runs
- [ ] `docker ps` works without sudo

---

# Level 2 — Create Application

Create directory:

```bash
mkdir -p ~/docker-lab
cd ~/docker-lab
```

Create:

```text
app.py
```

Example:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(b"<h1>MyApp Docker v1</h1>")

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

Run directly:

```bash
python3 app.py
```

From another terminal:

```bash
curl http://localhost:8000
```

Expected:

```html
<h1>MyApp Docker v1</h1>
```

Stop the application.

### Understand

Before Docker:

```text
Linux
  ↓
Python
  ↓
app.py
```

We know the application works before containerizing it.

---

# Level 3 — Create Dockerfile

Create:

```text
Dockerfile
```

Content:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

EXPOSE 8000

CMD ["python3", "app.py"]
```

Check:

```bash
cat Dockerfile
```

### Explain

Be able to explain:

```text
FROM
WORKDIR
COPY
EXPOSE
CMD
```

---

# Level 4 — Build Image

Build:

```bash
docker build -t myapp:v1 .
```

Check:

```bash
docker images
```

Inspect:

```bash
docker inspect myapp:v1
```

Image history:

```bash
docker history myapp:v1
```

### Checkpoint

- [ ] `myapp:v1` exists
- [ ] You can explain image vs container

Mental model:

```text
Dockerfile
    ↓
docker build
    ↓
myapp:v1
```

---

# Level 5 — Run Container

Run:

```bash
docker run -d \
  --name myapp \
  -p 8001:8000 \
  myapp:v1
```

Check:

```bash
docker ps
```

Test:

```bash
curl http://localhost:8001
```

Expected:

```html
<h1>MyApp Docker v1</h1>
```

### Understand Port Mapping

```text
Host :8001
   ↓
Docker
   ↓
Container :8000
   ↓
Python App
```

Explain:

```text
8001 = host port
8000 = container port
```

---

# Level 6 — Logs

Check logs:

```bash
docker logs myapp
```

Follow:

```bash
docker logs -f myapp
```

Generate requests:

```bash
curl http://localhost:8001
curl http://localhost:8001
```

Observe logs.

### Checkpoint

- [ ] You can find application logs
- [ ] You understand why logs are useful before entering the container

---

# Level 7 — Enter Container

Open shell:

```bash
docker exec -it myapp /bin/sh
```

Inside:

```bash
pwd
ls -la
ps
env
```

Check application:

```bash
cat /app/app.py
```

Exit:

```bash
exit
```

### Understand

You are not entering a VM.

You are entering the filesystem/process environment of the container.

---

# Level 8 — Container Lifecycle

Stop:

```bash
docker stop myapp
```

Check:

```bash
docker ps
docker ps -a
```

Start:

```bash
docker start myapp
```

Test:

```bash
curl http://localhost:8001
```

Restart:

```bash
docker restart myapp
```

Remove:

```bash
docker stop myapp
docker rm myapp
```

Check:

```bash
docker ps -a
docker images
```

### Question

After removing the container:

```text
Does myapp:v1 still exist?
```

You should understand:

```text
Image != Container
```

---

# Level 9 — Create v2

Change:

```text
MyApp Docker v1
```

to:

```text
MyApp Docker v2
```

Build:

```bash
docker build -t myapp:v2 .
```

Check:

```bash
docker images
```

Now you should have:

```text
myapp:v1
myapp:v2
```

Run v2:

```bash
docker run -d \
  --name myapp \
  -p 8001:8000 \
  myapp:v2
```

Test:

```bash
curl http://localhost:8001
```

Expected:

```html
<h1>MyApp Docker v2</h1>
```

---

# Level 10 — Rollback

Remove v2 container:

```bash
docker rm -f myapp
```

Run v1 again:

```bash
docker run -d \
  --name myapp \
  -p 8001:8000 \
  myapp:v1
```

Verify:

```bash
curl http://localhost:8001
```

Expected:

```html
<h1>MyApp Docker v1</h1>
```

### Understand

Rollback here is:

```text
myapp:v2
   ↓
myapp:v1
```

Versioned images make rollback clear.

---

# Level 11 — Docker Networks

List networks:

```bash
docker network ls
```

Inspect default bridge:

```bash
docker network inspect bridge
```

Create:

```bash
docker network create web_network
```

Check:

```bash
docker network ls
```

Run container in this network:

```bash
docker rm -f myapp

docker run -d \
  --name myapp \
  --network web_network \
  -p 8001:8000 \
  myapp:v1
```

Inspect:

```bash
docker network inspect web_network
```

Find:

```text
container name
container IP
network subnet
```

### Checkpoint

- [ ] You can create a Docker network
- [ ] You can see which containers belong to it

---

# Level 12 — Volumes

Create volume:

```bash
docker volume create mydata
```

Check:

```bash
docker volume ls
```

Inspect:

```bash
docker volume inspect mydata
```

Run a temporary container:

```bash
docker run --rm \
  -v mydata:/data \
  alpine \
  sh -c 'echo "persistent-data" > /data/test.txt'
```

Start another container:

```bash
docker run --rm \
  -v mydata:/data \
  alpine \
  cat /data/test.txt
```

Expected:

```text
persistent-data
```

### Understand

First container is gone.

Data remains.

```text
Container 1
    X
    │
 Volume
    │
    ↓
Container 2
```

---

# Level 13 — Environment Variables

Run:

```bash
docker run --rm \
  -e APP_ENV=production \
  alpine \
  env
```

Find:

```text
APP_ENV=production
```

### Question

What belongs in:

```text
Image
```

and what belongs in:

```text
Runtime configuration
```

Do not put secrets directly into Dockerfiles.

---

# Level 14 — Resource Inspection

Run MyApp again.

Check:

```bash
docker stats
```

Processes:

```bash
docker top myapp
```

Disk usage:

```bash
docker system df
```

Full metadata:

```bash
docker inspect myapp
```

### Checkpoint

You should know where to look for:

```text
CPU / RAM
processes
ports
network
mounts
image
container state
```

---

# Level 15 — Break the Container

Now intentionally create a failure.

Change Dockerfile:

```dockerfile
CMD ["python3", "wrong.py"]
```

Build:

```bash
docker build -t myapp:broken .
```

Run:

```bash
docker run -d \
  --name broken-app \
  myapp:broken
```

Check:

```bash
docker ps
```

The container may disappear from running containers.

Now troubleshoot:

```bash
docker ps -a
```

Then:

```bash
docker logs broken-app
```

Check exit code:

```bash
docker inspect broken-app \
  --format='{{.State.ExitCode}}'
```

### Lesson

When container exits:

```text
docker ps -a
    ↓
docker logs
    ↓
docker inspect
```

---

# Level 16 — Break Port Mapping

Run MyApp with:

```bash
docker run -d \
  --name myapp \
  -p 9000:8000 \
  myapp:v1
```

Try:

```bash
curl http://localhost:8001
```

Then:

```bash
curl http://localhost:9000
```

Check:

```bash
docker port myapp
```

Also:

```bash
ss -lntp
```

### Lesson

Troubleshoot:

```text
Host Port
   ↓
Docker Mapping
   ↓
Container Port
   ↓
Application
```

---

# Level 17 — Docker Architecture Check

Check Docker process:

```bash
ps aux | grep dockerd
```

Check socket:

```bash
ls -l /var/run/docker.sock
```

Check service:

```bash
systemctl status docker
```

Architecture:

```text
docker CLI
    ↓
docker.sock
    ↓
dockerd
    ↓
containerd
    ↓
runc
    ↓
Linux Kernel
```

### Explain

Be able to explain:

```text
docker CLI != Docker daemon
```

and:

```text
Container = Linux process
```

---

# Level 18 — Docker Permissions

Check:

```bash
groups
```

Check socket:

```bash
ls -l /var/run/docker.sock
```

Understand why this works:

```bash
sudo usermod -aG docker ops
```

Important:

```text
docker group
    ≈
root-level host control
```

This is a security decision, not only convenience.

---

# Level 19 — Cleanup

Check first:

```bash
docker ps -a
docker images
docker volume ls
docker network ls
docker system df
```

Remove lab containers:

```bash
docker rm -f myapp broken-app
```

Remove lab images:

```bash
docker rmi myapp:v1
docker rmi myapp:v2
docker rmi myapp:broken
```

Remove network:

```bash
docker network rm web_network
```

Remove volume:

```bash
docker volume rm mydata
```

Check:

```bash
docker system df
```

Do not blindly use aggressive cleanup on a production host.

---

# Troubleshooting Kata

You receive this ticket:

```text
MyApp does not respond on port 8001.
```

Do not rebuild immediately.

Follow the layers:

```text
1. Docker daemon
2. Container state
3. Logs
4. Port mapping
5. Application inside container
6. Network
7. Host firewall / port
```

Useful commands:

```bash
systemctl status docker

docker ps
docker ps -a

docker logs myapp

docker port myapp

docker inspect myapp

docker exec -it myapp /bin/sh

ss -lntp
```

Mental flow:

```text
Host
 ↓
Docker
 ↓
Container
 ↓
Port
 ↓
Application
```

---

# Final Challenge 🥷

Delete the lab and rebuild it without following the commands line by line.

You should be able to:

- [ ] Create a simple application
- [ ] Write a Dockerfile
- [ ] Build `myapp:v1`
- [ ] Run it on `8001:8000`
- [ ] Read logs
- [ ] Enter the container
- [ ] Stop/start/remove containers
- [ ] Build `myapp:v2`
- [ ] Deploy v2
- [ ] Roll back to v1
- [ ] Create and inspect a Docker network
- [ ] Create and verify a volume
- [ ] Pass environment variables
- [ ] Troubleshoot a crashed container
- [ ] Troubleshoot wrong port mapping
- [ ] Explain Docker architecture
- [ ] Explain why Docker socket access is sensitive

---

# I Understand Docker When I Can Explain

```text
Dockerfile → Image → Container

Image != Container

Container = isolated Linux process

Host port != Container port

Container data != Persistent data

Docker CLI != dockerd

docker.sock = powerful host access
```

And when something breaks, I know to follow:

```text
Linux
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