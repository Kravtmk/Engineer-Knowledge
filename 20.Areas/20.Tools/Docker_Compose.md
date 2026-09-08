# Docker Compose

Quick reference for everyday Docker Compose work.

---

## Check Compose

```bash
docker compose version
```

Check current project:

```bash
docker compose ps
```

---

# Basic Workflow

Start services:

```bash
docker compose up -d
```

Stop services:

```bash
docker compose stop
```

Start stopped services:

```bash
docker compose start
```

Restart services:

```bash
docker compose restart
```

Stop and remove containers/network:

```bash
docker compose down
```

---

# Build

Build images:

```bash
docker compose build
```

Build without cache:

```bash
docker compose build --no-cache
```

Build and start:

```bash
docker compose up -d --build
```

---

# Pull Images

Pull latest configured images:

```bash
docker compose pull
```

Then recreate services:

```bash
docker compose up -d
```

Typical update flow:

```text
docker compose pull
        ↓
docker compose up -d
```

---

# Services

List services:

```bash
docker compose config --services
```

Start one service:

```bash
docker compose up -d myapp
```

Restart one service:

```bash
docker compose restart myapp
```

Stop one service:

```bash
docker compose stop myapp
```

---

# Logs

All services:

```bash
docker compose logs
```

Follow logs:

```bash
docker compose logs -f
```

One service:

```bash
docker compose logs myapp
```

Follow one service:

```bash
docker compose logs -f myapp
```

Last 100 lines:

```bash
docker compose logs --tail 100 myapp
```

---

# Enter Container

Open shell:

```bash
docker compose exec myapp /bin/bash
```

For minimal images:

```bash
docker compose exec myapp /bin/sh
```

Run one command:

```bash
docker compose exec myapp env
```

---

# Status

Check services:

```bash
docker compose ps
```

Check containers directly:

```bash
docker ps
```

Detailed config:

```bash
docker compose config
```

Validate compose file:

```bash
docker compose config >/dev/null
```

---

# compose.yaml

Basic example:

```yaml
services:
  myapp:
    image: myapp:v1
    container_name: myapp
    ports:
      - "8001:8000"
    restart: unless-stopped
```

Run:

```bash
docker compose up -d
```

---

# Build From Dockerfile

```yaml
services:
  myapp:
    build: .
    image: myapp:v1
    ports:
      - "8001:8000"
```

Build and run:

```bash
docker compose up -d --build
```

---

# Environment Variables

Inline:

```yaml
services:
  myapp:
    environment:
      APP_ENV: production
```

Using variable:

```yaml
services:
  myapp:
    environment:
      APP_ENV: ${APP_ENV}
```

`.env`:

```text
APP_ENV=production
```

Check rendered configuration:

```bash
docker compose config
```

Do not commit secrets into public repositories.

---

# Volumes

Named volume:

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

Check:

```bash
docker volume ls
```

Important:

```bash
docker compose down
```

does NOT remove named volumes by default.

But:

```bash
docker compose down -v
```

removes volumes.

⚠️ Be careful with `-v`.

---

# Networks

Example:

```yaml
services:
  myapp:
    networks:
      - backend

  db:
    networks:
      - backend

networks:
  backend:
```

Check:

```bash
docker network ls
```

Inspect:

```bash
docker network inspect <network_name>
```

Compose normally creates project networks automatically.

---

# Service Communication

Inside the Compose network, use service names.

Example:

```yaml
services:
  app:
    image: myapp:v1

  db:
    image: postgres:16
```

Application should connect to:

```text
db:5432
```

Not:

```text
localhost:5432
```

Inside a container:

```text
localhost = this container
```

---

# Restart Policy

Example:

```yaml
restart: unless-stopped
```

Common values:

```text
no
always
unless-stopped
on-failure
```

---

# Recreate

Force recreate containers:

```bash
docker compose up -d --force-recreate
```

Recreate one service:

```bash
docker compose up -d --force-recreate myapp
```

---

# Version Update

Example:

```yaml
image: myapp:v1
```

Change to:

```yaml
image: myapp:v2
```

Apply:

```bash
docker compose up -d
```

Verify:

```bash
docker compose ps
docker compose logs myapp
```

---

# Rollback

Change:

```yaml
image: myapp:v2
```

back to:

```yaml
image: myapp:v1
```

Then:

```bash
docker compose up -d
```

Verify:

```bash
docker compose ps
curl http://localhost:8001
```

---

# Remove Project

Stop and remove containers/network:

```bash
docker compose down
```

Also remove volumes:

```bash
docker compose down -v
```

Also remove images built by Compose:

```bash
docker compose down --rmi local
```

Use destructive options carefully.

---

# Troubleshooting

## Service not running

```bash
docker compose ps
```

Then:

```bash
docker compose logs myapp
```

---

## Config problem

```bash
docker compose config
```

Useful for:

```text
YAML errors
environment variable problems
final rendered configuration
```

---

## App not reachable

Check:

```bash
docker compose ps
```

Then:

```bash
docker port myapp
```

Check host:

```bash
ss -lntp
```

Think:

```text
Client
  ↓
Host Port
  ↓
Compose Port Mapping
  ↓
Container Port
  ↓
Application
```

---

## Service cannot reach another service

Check network:

```bash
docker network ls
```

Inspect:

```bash
docker network inspect <network_name>
```

Test from container:

```bash
docker compose exec myapp /bin/sh
```

Then:

```bash
ping db
```

or:

```bash
curl http://backend:8000
```

Use the Compose service name as DNS name.

---

# Useful Daily Flow

Deploy:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f
```

Update:

```bash
docker compose pull
docker compose up -d
```

Rollback:

```text
change image tag
      ↓
docker compose up -d
```

Stop:

```bash
docker compose down
```

---

# Mental Model

```text
compose.yaml
     ↓
Docker Compose
     ↓
Docker Engine
     ↓
Containers
 ├── myapp
 ├── nginx
 ├── postgres
 └── redis
```

Compose manages:

```text
Services
Networks
Volumes
Environment
Ports
```

---

# Remember

```text
Dockerfile   = build one image
Docker       = run one container
Compose      = describe and run a multi-container application
```

Inside Compose networking:

```text
service name = DNS name
```

Normal production-style update:

```text
change/pull image
      ↓
docker compose up -d
      ↓
verify
```

Docker theory:

```text
30.Resources/20.Tools/Docker.md
```

Docker Compose theory:

```text
30.Resources/20.Tools/Docker_Compose.md
```

Docker Compose practice:

```text
35.Practice/20.Tools/Docker_Compose.md
```