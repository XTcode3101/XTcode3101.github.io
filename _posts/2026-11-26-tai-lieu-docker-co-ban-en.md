---
title: "Docker Basics Documentation"
description: "Core Docker fundamentals: comparing Containers vs. VMs, key components, cache-optimized Dockerfiles, Volumes, and Docker Compose."
date: 2026-11-26 17:52:00 +0700
categories: [DevOps, Docker & K8s]
tags: [docker, container, dockerfile, docker-compose, devops]
pin: false
toc: true
---

## 1. Overview

Before containers became widespread, moving code from a developer's laptop to a server or a teammate's machine often meant manually installing libraries, tools, and system configurations step by step. Even minor differences in package versions or operating systems could trigger the classic *"it works on my machine"* problem.

**Docker** (launched in 2013) solves this by providing a complete toolkit to package an application together with all its dependencies and configuration into a standardized unit called a **Container**. This ensures the application runs consistently across any environment, from local development to production servers. Docker manages the entire lifecycle: packaging code into an **Image**, creating and running **Containers**, connecting them via **Networks**, and attaching persistent storage with **Volumes**.

### Docker Containers vs. Virtual Machines (VMs)

To see why Docker is much lighter than traditional virtualization, consider the three-layer system model: `Application` → `OS Kernel` → `Hardware`.

| Aspect | Virtual Machine (VM) | Docker Container |
| :--- | :--- | :--- |
| **Isolation Mechanism** | Uses a **Hypervisor** to virtualize hardware | Isolates only the **Application** layer (processes and userland) via `namespaces` and `cgroups` |
| **Operating System (Kernel)** | Each VM bundles its own **OS Kernel** | Shares the host's **OS Kernel** and hardware |
| **Speed & Footprint** | Heavyweight; boots in minutes | Lightweight; starts in seconds |
| **OS Compatibility** | Can run an OS with a different kernel from the host | Tied to the host's OS kernel |

> **Note for Windows and macOS:** Because Linux containers always require a Linux kernel, Docker Desktop runs a lightweight Linux VM in the background on Windows and macOS to provide that kernel.

---

## 2. Main Components

The Docker ecosystem revolves around a few core building blocks working together:

- **Docker Image**: A static, `read-only`, and `immutable` package containing the application code along with all required libraries and runtime dependencies. Images guarantee consistent execution across machines and are the artifacts stored, shared, pushed, and pulled on **Container Registries** (such as Docker Hub).
- **Docker Container**: A live, running instance created from an Image. Each container runs as an isolated process (`namespaces`, `cgroups`) with its own environment variables (`env`), ports, configuration, isolated filesystem, and a temporary `writable layer` on top. Any data that needs to persist beyond the container's lifecycle (such as database files) should be stored outside this writable layer using a **Volume**.
- **Dockerfile**: A text file containing step-by-step instructions that tell Docker how to automatically build an application into an Image.
- **Docker Compose**: A tool for defining and orchestrating multi-container applications in a single YAML file (`compose.yaml` or `docker-compose.yaml`), replacing long and repetitive `docker run` commands.
- **Volume & Bind Mount**: Storage mechanisms that persist data independently of a container's lifecycle or share live source code directories between the host and a container.
- **Docker Network**: Virtual networking that lets containers communicate securely with one another (including by container/service name) and with the outside world.

---

## 3. Usage

### Core Workflow

In day-to-day software development, working with Docker typically follows a three-stage workflow from local machine to server:

1. **Local Development (Dev)**: Write application code and pull supporting images (such as databases or caches) to run as containers locally, commonly orchestrated with `docker compose up`.
2. **Build & Push**: Write a `Dockerfile`, build the application into an Image with `docker build`, log in to a Registry using `docker login`, tag the image in the `<username>/<image-name>:<tag>` format, and push it with `docker push`.
3. **Deployment (Server)**: On a server running Docker Engine, pull the Image from the Registry using `docker pull` and start it as a container with `docker run` or `docker compose up -d`.

### Essential CLI Commands

The table below summarizes the core `docker` commands for managing Images, Containers, Networks, and cleanup:

| Command | Purpose & When to Use |
| :--- | :--- |
| `docker pull <image>` | Downloads an Image from a Registry (Docker Hub by default) to your local machine. |
| `docker images` *(or `docker image ls`)* | Lists all Images currently stored locally to check versions or disk usage. |
| `docker run -d <image>` | Creates and starts a new Container from an Image; add `-d` (`--detach`) to run it in the background. |
| `docker ps` | Lists running Containers; add `-a` to include stopped containers. |
| `docker stop <id\|name>` / `docker start <id\|name>` | Stops a running Container or restarts a stopped one. |
| `docker logs <id\|name>` | Displays the output logs of a Container, useful for monitoring or debugging. |
| `docker exec -it <id\|name> bash` | Opens an interactive shell inside a running Container (use `sh` instead of `bash` for minimal images like `alpine`). |
| `docker network create <network-name>` | Creates a `user-defined bridge` network so attached Containers can connect and resolve each other by name. |
| `docker push <user/image:tag>` | Uploads a tagged Image to Docker Hub or a Private Registry. |
| `docker rm <id>` / `docker rmi <image>` | Removes a stopped Container or deletes an unused Image. |
| `docker system prune -a` | Quickly removes all stopped containers, unused networks, and unreferenced images to free up disk space. |

### Building Images with Dockerfile & Optimizing Build Cache

A standard `Dockerfile` brings together the following core instructions:

```dockerfile
# Base image (many official images like node or python default to Debian; alpine is a popular lightweight variant)
FROM node:20-alpine

# Set environment variables inside the image and container
ENV NODE_ENV=production

# Copy dependency manifests first to leverage build cache
COPY package*.json ./

# Execute commands during IMAGE BUILD to install dependencies/tools
RUN npm install

# Copy the rest of the application source code from the host into the image
COPY . .

# Default command executed when STARTING the container (not during build)
CMD ["node", "index.js"]
```

To keep builds fast, small, and secure, apply these two practices:

- **Optimize Build Cache**: Place instructions that change infrequently (such as `COPY package*.json ./` and `RUN npm install`) **before** `COPY . .`. When you only edit application code, Docker reuses the cached dependency layer instead of reinstalling everything.
- **Use `.dockerignore`**: List heavy directories and sensitive files (such as `node_modules`, `.git`, `.env`) in `.dockerignore` so `COPY . .` never copies unnecessary files or secrets into the Image.

Build the Image with `docker build`, making sure to include the build context path at the end (typically `.` for the current directory):

```bash
docker build -t my-app:1.0 .
```

### Data Persistence: Named Volumes vs. Bind Mounts

Because a container's writable layer is lost when the container is removed, Docker offers two primary ways to mount external storage:

- **Named Volume** (e.g., `-v db_data:/var/lib/postgresql/data`): Managed directly by Docker and decoupled from the host's directory layout. This is the recommended choice for persistent data like production databases.
- **Bind Mount** (e.g., `-v ./src:/app/src`): Maps a specific host directory directly into the container. This is ideal during local development so code edits appear immediately inside the container (*hot-reload*).

### Multi-Container Setup with Docker Compose

Instead of manually running individual containers and wiring up flags, you can declare your entire stack in a `compose.yaml` file (note that the `version:` key is deprecated in Compose v2+):

```yaml
services:
  web:
    image: my-app:1.0
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
  db:
    image: postgres:16
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

- Start all services in the background with `docker compose up -d`. If your Compose file uses a custom name, specify it with `-f`:
  ```bash
  docker compose -f custom-compose.yaml up -d
  ```
- Stop and remove all containers and networks created by Compose with `docker compose down` (Volumes are preserved by default). Add `-v` (`docker compose down -v`) only when you want to delete Volume data as well.

### Troubleshooting Common Issues

- **Port conflict (`port is already allocated`)**: In `-p <host>:<container>`, this error means the host port on the left is already in use by another process. Switch the left-hand port to an available one (for example, `-p 8081:80`).
- **Container exits immediately (`Exited`)**: Run `docker logs <id|name>` to inspect the application's error output, or run `docker inspect <id|name>` to check the `ExitCode` and container configuration.

---

## 4. References

- [Docker Tutorial for Beginners [FULL COURSE in 3 Hours]](https://www.youtube.com/watch?v=3c-iBn73dDE&t=136s) — Full Docker tutorial course by TechWorld with Nana.
- [What is a container? (Docker Learn)](https://docs.docker.com/learn/content/what-is-a-container/) — Core concepts covering containers, images, and how they differ from VMs.
- [Docker Engine Overview](https://docs.docker.com/engine/) — Overview of Docker Engine's client-server architecture (`docker` CLI, REST API, `dockerd`).
