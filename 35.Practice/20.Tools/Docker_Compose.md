# Docker Compose Practice

## Goal

Build and operate a small multi-container application.

Final architecture:

```text
Client
  ↓
Nginx
  ↓
MyApp
  ↓
PostgreSQL
```

Then practice:

```text
deploy v1
   ↓
deploy v2
   ↓
rollback
   ↓
break
   ↓
troubleshoot
```

---

# Prerequisites

You need:

```text
Linux VM
Docker
Docker Compose
curl
git
```

Check:

```bash
docker --version
docker compose version
docker ps
```

---

# Level 1 — Create Project

Create:

```bash
mkdir -p ~/compose-lab
cd ~/compose-lab
```

Project:

```text
compose-lab/
├── app.py
├── Dockerfile
├── nginx.conf
└── compose.yaml
```

---

# Level 2 — Create MyApp

Create:

```text
app.py
```

Example:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import os

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        version = os.getenv("APP_VERSION", "unknown")

        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()

        self.wfile.write(
            f"<h1>MyApp {version}</h1>".encode()
        )

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

Test locally:

```bash
APP_VERSION=v1 python3 app.py
```

From another terminal:

```bash
curl http://localhost:8000
```

Expected:

```html
<h1>MyApp v1</h1>
```

Stop application.

---

# Level 3 — Dockerfile

Create:

```text
Dockerfile
```

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

EXPOSE 8000

CMD ["python3", "app.py"]
```

Build:

```bash
docker build -t myapp:v1 .
```

Check:

```bash
docker images
```

---

# Level 4 — First Compose Service

Create:

```text
compose.yaml
```

```yaml
services:
  app:
    image: myapp:v1
    environment:
      APP_VERSION: v1
    ports:
      - "8001:8000"
    restart: unless-stopped
```

Validate:

```bash
docker compose config
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Test:

```bash
curl http://localhost:8001
```

Expected:

```html
<h1>MyApp v1</h1>
```

### Checkpoint

- [ ] Compose project starts
- [ ] App service is Running
- [ ] Port mapping works

---

# Level 5 — Logs and Exec

Logs:

```bash
docker compose logs
```

One service:

```bash
docker compose logs app
```

Follow:

```bash
docker compose logs -f app
```

Enter container:

```bash
docker compose exec app /bin/sh
```

Inside:

```bash
pwd
ls -la
env
```

Exit:

```bash
exit
```

---

# Level 6 — Add Nginx

Create:

```text
nginx.conf
```

```nginx
events {}

http {
    server {
        listen 80;

        location / {
            proxy_pass http://app:8000;
        }
    }
}
```

Update `compose.yaml`:

```yaml
services:
  app:
    image: myapp:v1
    environment:
      APP_VERSION: v1
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    restart: unless-stopped
```

Apply:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Test:

```bash
curl http://localhost:8080
```

Expected:

```html
<h1>MyApp v1</h1>
```

---

# Level 7 — Understand Service DNS

Important flow:

```text
Client
 ↓
Host :8080
 ↓
Nginx Container
 ↓
app:8000
 ↓
MyApp Container
```

Inside Compose:

```text
app
```

is a DNS name.

Check network:

```bash
docker network ls
```

Inspect project network:

```bash
docker network inspect compose-lab_default
```

You should find:

```text
nginx
app
```

### Understand

This works:

```text
nginx → app:8000
```

This would be wrong inside Nginx:

```text
localhost:8000
```

because:

```text
localhost = Nginx container
```

---

# Level 8 — Add PostgreSQL

Update `compose.yaml`:

```yaml
services:
  app:
    image: myapp:v1
    environment:
      APP_VERSION: v1
      DB_HOST: db
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: labpassword
    volumes:
      - db_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  db_data:
```

Apply:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Check DB logs:

```bash
docker compose logs db
```

---

# Level 9 — Inspect Service Communication

Enter App container:

```bash
docker compose exec app /bin/sh
```

Check environment:

```bash
env | grep DB
```

Expected:

```text
DB_HOST=db
```

Understand:

```text
App
 ↓
db
 ↓
PostgreSQL
```

The service name:

```text
db
```

is used instead of a container IP.

---

# Level 10 — Persistent Volume

Check volumes:

```bash
docker volume ls
```

Find the Compose volume.

Inspect:

```bash
docker volume inspect compose-lab_db_data
```

Stop and remove containers:

```bash
docker compose down
```

Check volume:

```bash
docker volume ls
```

It should still exist.

Start again:

```bash
docker compose up -d
```

### Lesson

```text
Container lifecycle
        ≠
Volume lifecycle
```

---

# Level 11 — Environment Variables

Create:

```text
.env
```

Example:

```text
APP_VERSION=v1
POSTGRES_DB=myapp
POSTGRES_USER=myapp
POSTGRES_PASSWORD=labpassword
```

Update Compose:

```yaml
services:
  app:
    image: myapp:v1
    environment:
      APP_VERSION: ${APP_VERSION}

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Check rendered config:

```bash
docker compose config
```

### Security

Do not commit real secrets:

```bash
echo ".env" >> .gitignore
```

For this lab a simple password is acceptable.

For real environments use proper secret management.

---

# Level 12 — Deploy v2

Change application:

```text
MyApp v1
```

to:

```text
MyApp v2
```

Build:

```bash
docker build -t myapp:v2 .
```

Update:

```yaml
app:
  image: myapp:v2
```

And:

```text
APP_VERSION=v2
```

Apply:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Test:

```bash
curl http://localhost:8080
```

Expected:

```html
<h1>MyApp v2</h1>
```

### Observe

Compose recreates the App container.

Check:

```bash
docker ps
```

---

# Level 13 — Rollback

Change:

```yaml
image: myapp:v2
```

back to:

```yaml
image: myapp:v1
```

Also:

```text
APP_VERSION=v1
```

Apply:

```bash
docker compose up -d
```

Verify:

```bash
curl http://localhost:8080
```

Expected:

```html
<h1>MyApp v1</h1>
```

### Understand

Rollback:

```text
compose.yaml
   ↓
desired image version
   ↓
docker compose up -d
   ↓
recreate container
```

---

# Level 14 — Break the Application

Intentionally use a bad image tag:

```yaml
image: myapp:v99
```

Run:

```bash
docker compose up -d
```

Troubleshoot:

```bash
docker compose ps
docker compose logs app
docker images
```

Question:

```text
Does image myapp:v99 exist?
```

Fix the tag.

Apply again:

```bash
docker compose up -d
```

---

# Level 15 — Break Nginx

Change:

```nginx
proxy_pass http://wrong-service:8000;
```

Apply:

```bash
docker compose up -d --force-recreate nginx
```

Test:

```bash
curl http://localhost:8080
```

Troubleshoot:

```bash
docker compose ps
docker compose logs nginx
```

Check network:

```bash
docker network inspect compose-lab_default
```

Fix:

```nginx
proxy_pass http://app:8000;
```

Recreate:

```bash
docker compose up -d --force-recreate nginx
```

---

# Level 16 — Break Port Mapping

Change:

```yaml
ports:
  - "9000:80"
```

Apply:

```bash
docker compose up -d
```

Try:

```bash
curl http://localhost:8080
```

Then:

```bash
curl http://localhost:9000
```

Check:

```bash
docker compose ps
ss -lntp
```

### Lesson

```text
Client
 ↓
Host Port
 ↓
Nginx Container Port
 ↓
App Service
```

---

# Level 17 — Break Environment

Remove:

```text
APP_VERSION
```

Run:

```bash
docker compose up -d
```

Check:

```bash
curl http://localhost:8080
```

Then:

```bash
docker compose exec app env
```

Understand:

```text
Image can be correct
Container can be Running
But runtime configuration can still be wrong
```

---

# Level 18 — Health Check

Add a health check to the app:

```yaml
services:
  app:
    image: myapp:v1
    healthcheck:
      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000')"]
      interval: 10s
      timeout: 3s
      retries: 3
```

Apply:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Observe:

```text
running
healthy
```

### Understand

```text
Container Running
       ≠
Application Healthy
```

---

# Level 19 — Restart Policy

Check current policy:

```yaml
restart: unless-stopped
```

Find App container:

```bash
docker ps
```

Kill application process from inside:

```bash
docker compose exec app /bin/sh
```

Inspect processes.

Understand conceptually:

```text
main process exits
      ↓
container exits
      ↓
restart policy
      ↓
container starts again
```

Check:

```bash
docker compose ps
```

---

# Level 20 — Update From Registry

Production-style Compose often uses:

```yaml
image: ghcr.io/user/myapp:v1
```

Flow:

```text
Registry
   ↓
docker compose pull
   ↓
docker compose up -d
```

Commands:

```bash
docker compose pull
docker compose up -d
```

Then:

```bash
docker compose ps
docker compose logs
```

Understand:

```text
CI builds image
      ↓
Registry
      ↓
Compose host pulls image
      ↓
Container recreated
```

---

# Level 21 — Inspect Full Project

Check services:

```bash
docker compose config --services
```

Rendered configuration:

```bash
docker compose config
```

Status:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs
```

Networks:

```bash
docker network ls
```

Volumes:

```bash
docker volume ls
```

Images:

```bash
docker images
```

Resources:

```bash
docker stats
```

---

# Level 22 — Safe Cleanup

Check first:

```bash
docker compose ps
docker volume ls
docker network ls
```

Stop project:

```bash
docker compose down
```

This normally removes:

```text
containers
project network
```

but keeps named volumes.

Check:

```bash
docker volume ls
```

Only if you intentionally want to delete lab data:

```bash
docker compose down -v
```

### Important

```text
down
```

and:

```text
down -v
```

are very different for stateful applications.

---

# Troubleshooting Kata

Ticket:

```text
Website does not open.
```

Do not immediately restart everything.

Follow:

```text
Linux Host
    ↓
Docker
    ↓
Compose
    ↓
Nginx
    ↓
App
    ↓
Database
```

## 1. Docker

```bash
systemctl status docker
```

## 2. Compose

```bash
docker compose ps
```

## 3. Logs

```bash
docker compose logs nginx
docker compose logs app
docker compose logs db
```

## 4. Configuration

```bash
docker compose config
```

## 5. Network

```bash
docker network ls
docker network inspect compose-lab_default
```

## 6. App

```bash
docker compose exec app /bin/sh
```

## 7. Host Port

```bash
ss -lntp
```

Follow the path of the request:

```text
Client
 ↓
Host Port
 ↓
Nginx
 ↓
Compose Network
 ↓
App
 ↓
DB
```

---

# Final Challenge 🥷

Delete the lab and rebuild it without following the commands line by line.

Build:

```text
Nginx
  ↓
MyApp
  ↓
PostgreSQL
```

You should be able to:

- [ ] Write `compose.yaml`
- [ ] Start the full stack
- [ ] Explain service vs container
- [ ] Explain Compose networking
- [ ] Use service names as DNS
- [ ] Explain why `localhost` can be wrong
- [ ] Add persistent volume
- [ ] Use environment variables
- [ ] Validate with `docker compose config`
- [ ] Read service logs
- [ ] Enter a service container
- [ ] Deploy v1
- [ ] Deploy v2
- [ ] Roll back to v1
- [ ] Diagnose wrong image
- [ ] Diagnose wrong DNS/service name
- [ ] Diagnose wrong host port
- [ ] Explain `down` vs `down -v`
- [ ] Explain restart policy
- [ ] Explain Compose vs Kubernetes

---

# I Understand Docker Compose When I Can Explain

```text
compose.yaml
    ↓
Services
    ↓
Containers
```

and:

```text
Service      = desired container configuration
Container    = running instance
Network      = service communication
Volume       = persistent data
Environment  = runtime configuration
```

and the request path:

```text
Client
 ↓
Host Port
 ↓
Nginx
 ↓
Compose Network
 ↓
App
 ↓
Database
```

and the deployment path:

```text
Image v1
  ↓
Compose
  ↓
Container v1

Image v2
  ↓
change compose.yaml
  ↓
docker compose up -d
  ↓
Container v2
```

Core idea:

> Build the stack, break the stack, understand the stack, restore the stack.