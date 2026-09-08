# Docker

## What is Docker?

Docker is a platform for building and running applications in containers.

Instead of installing the application and all its dependencies directly on the server:

```text
Server
├── Python
├── libraries
├── application
└── configuration
```

we package the application into an image:

```text
Dockerfile
    ↓
Docker Image
    ↓
Container
    ↓
Application
```

The same image can then be started on different Docker hosts.

---

# Why Docker?

Without Docker, deployment depends heavily on the server:

```text
Application
    ↓
Install runtime
    ↓
Install dependencies
    ↓
Configure environment
    ↓
Run process
```

This can create differences between environments.

With Docker:

```text
Application + Runtime + Dependencies
                ↓
             Image
                ↓
             Container
```

The deployment becomes more reproducible.

The important idea:

> Build the artifact once, then run that artifact.

---

# Docker Architecture

The basic architecture is:

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

## Docker CLI

When we run:

```bash
docker ps
docker build
docker run
```

the CLI does not create containers itself.

It sends requests to the Docker daemon.

---

## Docker Socket

On Linux, the Docker CLI normally communicates with the daemon through:

```text
/var/run/docker.sock
```

Conceptually:

```text
docker CLI
    ↓
docker.sock
    ↓
Docker daemon
```

This is why access to the Docker socket is security-sensitive.

A user with powerful access to the Docker daemon can effectively gain root-level control over the host.

---

## Docker Daemon — dockerd

`dockerd` is the main Docker daemon.

It manages things such as:

```text
Images
Containers
Networks
Volumes
```

It receives requests from the Docker client and coordinates the lower container runtime layers.

---

## containerd

Under Docker sits `containerd`.

Simplified:

```text
Docker
   ↓
containerd
   ↓
container runtime
```

It handles lower-level container lifecycle operations.

---

## runc

`runc` is the low-level runtime responsible for creating and starting containers.

The chain can be simplified as:

```text
docker run
    ↓
dockerd
    ↓
containerd
    ↓
runc
    ↓
Linux process
```

This leads to an important idea:

> A container is ultimately a Linux process, not a small virtual machine.

---

# Container vs Virtual Machine

A VM has its own operating system kernel:

```text
Physical Host
    ↓
Hypervisor
    ↓
VM
├── Linux Kernel
└── Application
```

Containers share the host Linux kernel:

```text
Linux Host
    ↓
Docker
    ↓
Container A
Container B
Container C
    ↓
shared Linux Kernel
```

Containers isolate processes, filesystem views, networking and resources, but they do not normally boot a separate kernel.

That is one reason containers can start very quickly.

---

# Dockerfile

A Dockerfile describes how to build an image.

Example:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

EXPOSE 8000

CMD ["python3", "app.py"]
```

Conceptually:

```text
Dockerfile
    ↓
instructions
    ↓
docker build
    ↓
Image
```

The Dockerfile is not the running application.

It is the recipe used to create the artifact.

---

# Image

A Docker image is a packaged, read-only template used to create containers.

Example:

```text
myapp:v1
```

It can contain:

```text
Application code
Runtime
Libraries
Dependencies
Default command
Filesystem
```

Think:

```text
Dockerfile
    ↓ build
Image
    ↓ run
Container
```

One image can create many containers:

```text
             ┌→ Container 1
Image:v1 ────┼→ Container 2
             └→ Container 3
```

---

# Image Tags

Images can have tags:

```text
myapp:v1
myapp:v2
myapp:1.4.2
myapp:latest
```

A tag is a human-readable reference to an image.

For deployment, explicit/versioned tags are usually easier to reason about:

```text
v1 → v2
```

and rollback becomes understandable:

```text
v2 → v1
```

Using only:

```text
latest
```

makes it harder to know exactly which application version is running.

---

# Container

A container is a running instance of an image.

```text
Image
  ↓
docker run
  ↓
Container
  ↓
Linux process
```

This distinction is fundamental:

```text
Image     = artifact
Container = runtime instance
```

Deleting a container does not necessarily mean deleting the image.

Likewise, rebuilding an image does not automatically replace an already running container.

---

# Container Lifecycle

Typical lifecycle:

```text
Image
 ↓
Create
 ↓
Start
 ↓
Running
 ↓
Stop
 ↓
Stopped
 ↓
Remove
```

A container normally lives as long as its main process.

If the main application process exits:

```text
PID 1 exits
    ↓
Container exits
```

This is why a container that immediately stops often means:

```text
Application crashed
Command finished
Configuration error
Missing dependency
```

The first useful place to investigate is usually the container logs.

---

# Port Mapping

Imagine the application listens inside the container on:

```text
8000
```

We run:

```text
-p 8001:8000
```

This means:

```text
Host :8001
    ↓
Docker
    ↓
Container :8000
    ↓
Application
```

So:

```text
8001 = host port
8000 = container port
```

These are two different network contexts.

A common troubleshooting mistake is checking only the application port and forgetting the host-to-container mapping.

---

# Docker Networking

Containers can communicate through Docker networks.

Simplified:

```text
Docker Host
│
├── Container A
│       │
│       └──────┐
│              │
├── Docker Network
│              │
│       ┌──────┘
│       │
└── Container B
```

A Docker network gives containers a controlled way to communicate.

This becomes especially useful with multiple services:

```text
Nginx
  ↓
web_network
  ↓
Application
  ↓
db_network
  ↓
PostgreSQL
```

Docker Compose often creates and manages these networks automatically.

---

# Container Data

The writable filesystem inside a container should not automatically be treated as permanent storage.

Think:

```text
Container
   ↓
temporary runtime
```

If important data exists only inside that container, removing the container can remove that data.

For persistent data we use volumes.

---

# Volumes

A Docker volume stores data separately from the container lifecycle.

```text
Container
    ↓
Volume
    ↓
Persistent Data
```

Example:

```text
PostgreSQL Container
        ↓
     db_data
        ↓
Database files
```

Now we can replace the container:

```text
old container
      ✕
      │
   Volume
      │
      ↓
new container
```

The application changes, while the persistent data remains.

This separation is especially important for:

```text
Databases
Uploads
Application state
Persistent files
```

---

# Environment Variables

Configuration can be passed to a container at runtime:

```text
Image
  +
Environment
  ↓
Container
```

For example:

```text
APP_ENV=production
DB_HOST=postgres
```

This helps separate:

```text
Application artifact
        from
Environment configuration
```

But secrets require additional care.

Passwords and tokens should not be baked into the Dockerfile or committed into a public repository.

---

# Container Isolation

Containers use Linux kernel mechanisms to isolate workloads.

At a high level:

```text
Namespaces
    ↓
process/network/filesystem isolation

cgroups
    ↓
resource control
```

This allows multiple containers to run on one Linux host while seeing different environments.

Conceptually:

```text
Container A
├── processes
├── network
└── filesystem view

Container B
├── processes
├── network
└── filesystem view

        ↓

shared Linux kernel
```

Isolation is not the same thing as having a completely separate VM.

---

# Docker and systemd

On a normal Linux Docker host:

```text
systemd
   ↓
docker.service
   ↓
dockerd
   ↓
containers
```

So there are two different layers to troubleshoot.

Docker itself:

```bash
systemctl status docker
```

Application container:

```bash
docker ps
docker logs myapp
```

If `dockerd` is down, all Docker operations are affected.

If only `myapp` is broken, Docker itself may be perfectly healthy.

---

# Docker and Docker Compose

Docker runs containers.

Docker Compose describes and manages a group of related containers.

Docker:

```text
docker run ...
```

Compose:

```text
compose.yaml
     ↓
Docker Compose
     ↓
Docker
     ↓
Containers
```

Example application:

```text
Nginx
  +
MyApp
  +
PostgreSQL
  +
Redis
```

Managing this with many individual `docker run` commands becomes inconvenient.

Compose describes the complete application stack as configuration.

Docker Compose is covered separately:

```text
30.Resources/20.Tools/Docker_Compose.md
```

---

# Docker in Our Deployment Path

Docker solves one specific part of the deployment problem:

```text
Source Code
    ↓
Dockerfile
    ↓
Docker Image
    ↓
Registry
    ↓
Deployment Platform
```

The deployment platform can then be different:

```text
Docker Compose
```

or:

```text
Kubernetes / K3s
```

This is important:

```text
Docker        = package/run containers
Docker Compose = manage a small container stack
Kubernetes    = orchestrate containers
GitHub Actions = automate CI/CD
Argo CD       = GitOps CD for Kubernetes
```

They are different layers, not competing names for the same thing.

---

# Troubleshooting Mental Model

When an application in Docker does not work, follow the layers:

```text
Linux Host
    ↓
Docker daemon
    ↓
Image
    ↓
Container
    ↓
Port / Network
    ↓
Volume / Configuration
    ↓
Application
```

Ask:

```text
Is Docker running?
        ↓
Does the image exist?
        ↓
Is the container running?
        ↓
Did the application start?
        ↓
Are ports correct?
        ↓
Can containers communicate?
        ↓
Is persistent data mounted?
        ↓
What do the logs say?
```

Do not troubleshoot every layer at the same time.

---

# Mental Model

```text
Dockerfile
    │
    │ docker build
    ▼
Image
    │
    │ docker run
    ▼
Container
    │
    ├── Network
    ├── Volume
    ├── Environment
    │
    ▼
Application Process
```

Underneath:

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

## Remember

```text
Dockerfile = build instructions
Image      = packaged artifact
Container  = running instance
Network    = communication
Volume     = persistent data
```

The most important idea:

> **Docker does not create a small VM. It packages an application into an image and runs it as an isolated process on the Linux host.**