# Docker Compose

## What is Docker Compose?

Docker Compose is a tool for defining and running a multi-container application.

Instead of starting containers one by one:

```text
docker run nginx ...
docker run myapp ...
docker run postgres ...
docker run redis ...
```

we describe the whole application in one file:

```text
compose.yaml
```

Then run:

```text
compose.yaml
     ↓
Docker Compose
     ↓
Docker Engine
     ↓
Containers
```

---

# Why Docker Compose?

A real application often consists of several services:

```text
Nginx
MyApp
PostgreSQL
Redis
```

Without Compose:

```text
Engineer
   ↓
many docker run commands
   ↓
manual ports
manual networks
manual volumes
manual environment
```

With Compose:

```text
compose.yaml
    ↓
complete application definition
```

The main idea:

> Describe the application stack as configuration.

---

# Docker vs Docker Compose

Docker works with individual containers:

```text
Dockerfile
    ↓
Image
    ↓
docker run
    ↓
Container
```

Docker Compose works with a group of related containers:

```text
compose.yaml
    ↓
Services
    ↓
Containers
```

So:

```text
Docker         = container engine
Docker Compose = application stack definition
```

Compose does not replace Docker.

It uses Docker.

---

# compose.yaml

The Compose file describes the application.

Example:

```yaml
services:
  app:
    image: myapp:v1
    ports:
      - "8001:8000"

  db:
    image: postgres:16
```

Conceptually:

```text
compose.yaml
├── app
└── db
```

Compose reads this desired configuration and creates the required Docker resources.

---

# Service vs Container

This distinction is important.

A Compose service is a configuration definition.

Example:

```yaml
services:
  app:
    image: myapp:v1
```

The service describes:

```text
which image
which ports
which environment
which volumes
which networks
```

Docker then creates a container from that service.

Think:

```text
Service
   ↓
configuration
   ↓
Container
```

So:

```text
Service   = desired configuration
Container = running instance
```

---

# Multi-Service Application

Example:

```text
Client
  ↓
Nginx
  ↓
MyApp
  ↓
PostgreSQL
```

Compose can define all of this:

```yaml
services:
  nginx:
    ...

  app:
    ...

  db:
    ...
```

This gives us one application stack instead of unrelated containers.

---

# Compose Networking

One of the most useful Compose features is automatic networking.

Suppose we have:

```yaml
services:
  app:
    image: myapp:v1

  db:
    image: postgres:16
```

Compose normally creates a project network:

```text
app
 │
 ├── Compose Network
 │
db
```

The services can reach each other by service name.

For example:

```text
db:5432
```

This is possible because Docker provides internal DNS.

---

# Service Name as DNS

Inside the Compose network:

```text
db
app
redis
nginx
```

can act as DNS names.

Example:

```text
App
 ↓
db:5432
```

or:

```text
Nginx
 ↓
app:8000
```

This is better than depending on container IP addresses, because container IPs can change.

Important:

```text
service name = stable logical address
container IP = implementation detail
```

---

# Why localhost Causes Confusion

Inside a container:

```text
localhost
```

means:

```text
this container
```

It does not mean another service.

Example:

```text
App container
```

If PostgreSQL runs in another container, this is wrong:

```text
localhost:5432
```

because PostgreSQL is not inside the App container.

Correct:

```text
db:5432
```

Mental model:

```text
App container
├── localhost = App container
└── db = PostgreSQL service
```

---

# Ports

Compose can publish container ports to the host.

Example:

```yaml
ports:
  - "8001:8000"
```

Meaning:

```text
Host :8001
   ↓
Docker
   ↓
Container :8000
```

But services communicating inside the Compose network usually do not need host-published ports.

Example:

```text
Nginx → app:8000
```

The application container does not necessarily need:

```text
8001:8000
```

unless the host or external clients must access it directly.

This is an important security principle:

> Publish only the ports that need external access.

---

# Volumes

Containers are replaceable.

Application data often is not.

Example:

```text
PostgreSQL container
        ↓
     db_data
```

Compose can define persistent volumes:

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

Now:

```text
Container
    ↓
Volume
    ↓
Data
```

The container may be recreated while the volume remains.

---

# Container Lifecycle vs Data Lifecycle

This separation is important:

```text
Container lifecycle
    ≠
Data lifecycle
```

Example:

```text
Postgres v16 container
        ↓
      removed

db_data volume
        ↓
      remains

new Postgres container
        ↓
uses same data
```

This is why destructive volume operations must be handled carefully.

---

# Environment Variables

Compose lets us separate application configuration from the image.

Example:

```yaml
services:
  app:
    image: myapp:v1
    environment:
      APP_ENV: production
```

Conceptually:

```text
Image
  +
Runtime configuration
  ↓
Container
```

The same image can therefore run with different configuration:

```text
myapp:v1
   ├── development
   ├── staging
   └── production
```

This supports the principle:

> Build once, configure at runtime.

---

# .env

Compose can read variables from `.env`.

Example:

```text
APP_ENV=production
APP_PORT=8001
```

Compose file:

```yaml
services:
  app:
    environment:
      APP_ENV: ${APP_ENV}
    ports:
      - "${APP_PORT}:8000"
```

This reduces repeated hard-coded values.

But `.env` should not automatically be treated as a secure secret store.

Secrets still require careful handling.

---

# Restart Policy

Containers can stop because:

```text
application crash
host reboot
manual stop
Docker restart
```

Compose can define restart behavior.

Example:

```yaml
restart: unless-stopped
```

Conceptually:

```text
Container stops
      ↓
Docker checks restart policy
      ↓
Container may restart
```

Restart policy provides basic recovery.

But this is not the same as Kubernetes orchestration.

---

# What Happens During `docker compose up -d`

Conceptually:

```text
docker compose up -d
        ↓
Read compose.yaml
        ↓
Compare required resources
        ↓
Create network if needed
        ↓
Create volumes if needed
        ↓
Create/recreate containers
        ↓
Start services
```

Compose is therefore more than a shortcut for one `docker run`.

It coordinates a complete local/single-host application stack.

---

# Recreate and Update

Suppose:

```yaml
image: myapp:v1
```

changes to:

```yaml
image: myapp:v2
```

Then:

```text
compose.yaml changes
       ↓
docker compose up -d
       ↓
old container replaced
       ↓
new container from v2
```

The important point:

```text
Image version changes
        ↓
Container is recreated
```

The existing running container does not magically transform into v2.

---

# Rollback

If both images exist:

```text
myapp:v1
myapp:v2
```

rollback can be:

```text
compose.yaml
myapp:v2
   ↓
myapp:v1
   ↓
docker compose up -d
```

This is one reason explicit image versions are useful.

You can clearly see:

```text
desired version = v1
```

instead of guessing what `latest` currently means.

---

# Pull vs Build

Compose can work in two common ways.

## Local Build

```yaml
services:
  app:
    build: .
```

Flow:

```text
Source Code
   ↓
Dockerfile
   ↓
docker compose build
   ↓
Image
   ↓
Container
```

## Registry Image

```yaml
services:
  app:
    image: ghcr.io/user/myapp:v1
```

Flow:

```text
Registry
   ↓
docker compose pull
   ↓
Image
   ↓
Container
```

This second model is common in CI/CD deployments.

---

# Compose in CI/CD

A simple deployment flow can be:

```text
Developer
   ↓
git push
   ↓
GitHub Actions
   ↓
Build Docker Image
   ↓
Registry
   ↓
VM
   ↓
docker compose pull
   ↓
docker compose up -d
```

Compose is therefore useful as a deployment mechanism for a single Docker host.

---

# Why Compose Feels Simple

Compose has relatively few layers:

```text
Linux VM
   ↓
Docker Engine
   ↓
Compose
   ↓
Containers
```

This gives good visibility.

If something breaks, there are fewer moving parts than in Kubernetes.

For many small or medium single-host applications this simplicity is valuable.

---

# Docker Compose vs Kubernetes

Compose:

```text
VM
 ↓
Docker
 ↓
Compose
 ↓
Containers
```

Kubernetes:

```text
Cluster
 ↓
Control Plane
 ↓
Deployment
 ↓
ReplicaSet
 ↓
Pods
 ↓
Containers
```

Compose focuses on running an application stack on Docker hosts, commonly one host.

Kubernetes adds orchestration mechanisms such as:

```text
desired state
replicas
scheduling
rescheduling
rolling updates
service abstraction
cluster networking
```

So Kubernetes adds capabilities, but also more layers and operational complexity.

---

# Restart Policy vs Self-Healing

Compose can use:

```yaml
restart: unless-stopped
```

If the container crashes:

```text
Container
   X
   ↓
Docker restarts it
```

But imagine the entire Docker host dies:

```text
VM
 X
```

Compose cannot move that container to another host by itself.

In a Kubernetes cluster:

```text
Node A dies
    ↓
scheduler/control plane
    ↓
Pod may run on Node B
```

This is one of the major architectural differences.

---

# Compose Is Not High Availability

Running:

```text
Nginx
App
Postgres
Redis
```

with Compose on one VM gives:

```text
one application stack
```

but the VM remains a failure domain.

```text
VM fails
   ↓
entire Compose stack fails
```

Compose makes deployment reproducible.

It does not automatically make the infrastructure highly available.

---

# Depends On

Compose can describe dependencies:

```yaml
services:
  app:
    depends_on:
      - db
```

This expresses that the app depends on the database.

But dependency ordering does not automatically guarantee that the database application is fully ready to serve traffic.

There is an important difference between:

```text
container started
```

and:

```text
service ready
```

For reliable applications, health checks and application retry behavior matter.

---

# Health Checks

A container can be running while the application inside it is broken.

Example:

```text
Container = Running
Application = unable to serve requests
```

Health checks help distinguish these states.

Conceptually:

```text
Container
   ↓
Health Check
   ↓
Is application actually healthy?
```

This gives better information than checking only whether the container process exists.

---

# Compose Project

Compose groups resources into a project.

If the directory/project is:

```text
myapp
```

Compose may create resources such as:

```text
myapp_default
myapp_db_data
myapp-app-1
myapp-db-1
```

This naming helps isolate multiple Compose applications on the same Docker host.

So one Docker host can run:

```text
Project A
├── containers
├── network
└── volumes

Project B
├── containers
├── network
└── volumes
```

---

# Security Model

Compose does not remove Docker security considerations.

Important areas remain:

```text
Docker socket
Container privileges
Published ports
Secrets
Volumes
Image trust
Host permissions
```

Avoid unnecessary:

```yaml
privileged: true
```

Avoid exposing ports that only internal services need.

Example:

```text
Internet
   ↓
Nginx :443
   ↓
App
   ↓
Database
```

Usually the database should not need:

```text
Host :5432
```

if only the application uses it.

---

# Troubleshooting Mental Model

If a Compose application fails:

```text
Linux Host
    ↓
Docker Engine
    ↓
Compose Project
    ↓
Service
    ↓
Container
    ↓
Network / Volume / Environment
    ↓
Application
```

Ask in order:

```text
Is Docker running?
      ↓
Did Compose parse the file?
      ↓
Which service failed?
      ↓
Is its container running?
      ↓
What do its logs say?
      ↓
Is the environment correct?
      ↓
Can services resolve each other?
      ↓
Are volumes mounted correctly?
      ↓
Are ports correct?
```

This is much better than immediately rebuilding everything.

---

# Mental Model

```text
compose.yaml
│
├── Service: nginx
│      ↓
│   Container
│
├── Service: app
│      ↓
│   Container
│
├── Service: db
│      ↓
│   Container
│
├── Networks
├── Volumes
├── Ports
└── Environment
        ↓
    Docker Engine
        ↓
      Linux
```

## Remember

```text
Dockerfile   = how to build an image
Image        = application artifact
Container    = running image
Service      = Compose configuration for a container
Compose      = application stack definition
Network      = service communication
Volume       = persistent data
```

Core idea:

> **Docker Compose turns multiple Docker containers, networks, volumes and configuration into one reproducible application stack.**