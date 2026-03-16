# 🐳 Multi-Stage Docker Builds
Multi-stage builds use one container to build the application and another clean container to run it, copying only the final artifact (JAR/binary).
A complete guide to understanding multi-stage Docker builds — what they are, why they matter, how to minimize image sizes, and how to apply them across different languages like **Go**, **Java**, and **Python**, including the use of **Distroless images**.

---

## 📋 Table of Contents

- [What is a Multi-Stage Docker Build?](#what-is-a-multi-stage-docker-build)
- [Why Does Image Size Matter?](#why-does-image-size-matter)
- [Single-Stage vs Multi-Stage: A Comparison](#single-stage-vs-multi-stage-a-comparison)
- [This Project: Go Calculator Example](#this-project-go-calculator-example)
- [Techniques to Minimize Docker Image Size](#techniques-to-minimize-docker-image-size)
- [Multi-Stage Builds by Language](#multi-stage-builds-by-language)
  - [Go](#go)
  - [Java (Spring Boot)](#java-spring-boot)
  - [Python (Flask)](#python-flask)
- [What Are Distroless Images?](#what-are-distroless-images)
- [Distroless vs scratch vs Alpine](#distroless-vs-scratch-vs-alpine)
- [When to Use Each Approach](#when-to-use-each-approach)
- [Quick Reference: Image Size Comparison](#quick-reference-image-size-comparison)

---

## What is a Multi-Stage Docker Build?

A **multi-stage Docker build** lets you use **multiple `FROM` instructions** in a single `Dockerfile`. Each `FROM` starts a new **build stage**, and you can selectively copy only the artifacts you need from one stage into the next.

The key idea: **separate your build environment from your runtime environment.**

```
┌─────────────────────────────────────────────────────────┐
│                   TRADITIONAL BUILD                      │
│                                                         │
│  Base OS + Build Tools + Source Code + Binary           │
│  → One huge image shipped to production (~1GB+)         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  MULTI-STAGE BUILD                       │
│                                                         │
│  Stage 1 (Build):  Base OS + Build Tools + Source Code  │
│         │                                               │
│         │  COPY only the compiled binary                │
│         ▼                                               │
│  Stage 2 (Runtime): Tiny OS + Binary only               │
│  → Lean image shipped to production (~5–30MB)           │
└─────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description |
|---|---|
| 🔒 **Security** | No build tools (compilers, package managers) in production image = smaller attack surface |
| 📦 **Size** | Drop image size from ~1 GB to as low as a few MB |
| 🚀 **Speed** | Faster push/pull from container registries |
| 🧹 **Cleanliness** | Source code, test files, and temporary build artifacts never reach production |

---

## Why Does Image Size Matter?

- **Faster deployments**: Smaller images pull faster from Docker Hub / ECR / GCR
- **Lower storage costs**: Fewer gigabytes stored in your container registry
- **Reduced attack surface**: Every extra tool (bash, curl, apt) is a potential vulnerability
- **Faster CI/CD**: Less data to move through pipelines = quicker feedback loops

---

## Single-Stage vs Multi-Stage: A Comparison

### ❌ Single-Stage Dockerfile (Bad Practice)

```dockerfile
FROM ubuntu AS build

RUN apt-get update && apt-get install -y golang-go
ENV GO111MODULE=off

COPY . .
RUN CGO_ENABLED=0 go build -o /app .

# The final image contains the entire Ubuntu OS AND Go toolchain
ENTRYPOINT ["/app"]
```

**Result:** ~700 MB image — contains Ubuntu, Go compiler, source code, build cache, and the binary.

### ✅ Multi-Stage Dockerfile (Best Practice)

```dockerfile
FROM ubuntu AS build

RUN apt-get update && apt-get install -y golang-go
ENV GO111MODULE=off

COPY . .
RUN CGO_ENABLED=0 go build -o /app .

# ---- New stage: only the binary ----
FROM scratch

COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

**Result:** ~2 MB image — contains **only** the statically compiled binary. Nothing else.

---

## This Project: Go Calculator Example

This repository demonstrates multi-stage Docker builds using a simple Go calculator CLI app.

### Project Structure

```
multi-stage-docker-build/
├── Dockerfile                        # ✅ Multi-stage build (ubuntu → scratch)
├── caluculator.go                    # Go source code
└── dockerfile-without-multistage/
    └── Dockerfile                    # ❌ Single-stage build (for comparison)
```

### How to Build & Run

```bash
# Build the multi-stage image
docker build -t calculator-multistage .

# Build the single-stage image (for size comparison)
docker build -t calculator-singlestage ./dockerfile-without-multistage

# Compare sizes
docker images | grep calculator

# Run the multi-stage calculator
docker run -it calculator-multistage
```

### Size Comparison (Expected)

```
REPOSITORY                    SIZE
calculator-singlestage        ~700 MB
calculator-multistage         ~2 MB     ✅ 99.7% smaller!
```

---

## Techniques to Minimize Docker Image Size

### 1. Use Multi-Stage Builds
Separate build-time and runtime dependencies. Only copy what's needed into the final stage.

### 2. Choose a Minimal Base Image

| Base Image | Size | Use Case |
|---|---|---|
| `ubuntu` / `debian` | ~80–130 MB | Development, debugging |
| `alpine` | ~5 MB | General purpose, shell access needed |
| `distroless` | ~2–20 MB | Production, security-focused |
| `scratch` | 0 MB | Statically compiled binaries (Go, Rust) |

### 3. Combine RUN Commands

```dockerfile
# ❌ Bad — creates 3 separate layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good — single layer
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

### 4. Use `.dockerignore`

Create a `.dockerignore` file to prevent unnecessary files from being copied:

```
.git
*.md
tests/
*.log
node_modules/
__pycache__/
.env
```

### 5. Avoid Installing Unnecessary Packages

```dockerfile
# Use --no-install-recommends to skip optional packages
RUN apt-get install -y --no-install-recommends curl
```

### 6. Use Specific Package Versions

Pin dependency versions to avoid unexpected bloat during cache-busted builds.

### 7. Remove Build Artefacts in the Same Layer

```dockerfile
RUN go build -o /app . && rm -rf /root/go/pkg /root/.cache
```

---

## Multi-Stage Builds by Language

---

### Go

Go is ideal for multi-stage builds because it can produce **statically linked binaries** with zero external dependencies — perfect for `scratch` images.

```dockerfile
# ---- Stage 1: Build ----
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

# ---- Stage 2: Run ----
FROM scratch

COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

**Why `CGO_ENABLED=0`?** Disables C bindings so the binary is fully static and can run inside a `scratch` container with no C runtime libraries.

**Why `-ldflags="-s -w"`?** Strips debug symbols and DWARF info, reducing binary size by ~25%.

---

### Java (Spring Boot)

Java applications need the JVM at runtime, so `scratch` isn't suitable. Instead, use the **JRE** (runtime) rather than the full **JDK** (development kit).

```dockerfile
# ---- Stage 1: Build ----
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app
COPY pom.xml .
# Download dependencies first (layer caching optimization)
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn clean package -DskipTests

# ---- Stage 2: Run ----
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app
# Copy only the built JAR, not the entire Maven project
COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Size Comparison:**

| Approach | Approximate Size |
|---|---|
| `FROM maven:3.9` (single-stage) | ~500 MB |
| `FROM eclipse-temurin:17-jre-alpine` (multi-stage) | ~180 MB |

#### Java with Distroless

For even better security in production Java apps:

```dockerfile
# ---- Stage 2: Distroless Runtime ----
FROM gcr.io/distroless/java17-debian12

WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Resulting size:** ~220 MB (includes only JRE + OS libraries, no shell or package manager)

---

### Python (Flask)

Python applications can use **slim** or **alpine** variants and install only production dependencies.

```dockerfile
# ---- Stage 1: Build / Install Dependencies ----
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .

# Install dependencies into a virtual environment
RUN python -m venv /opt/venv && \
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# ---- Stage 2: Run ----
FROM python:3.12-slim

WORKDIR /app

# Copy the virtual environment from the build stage
COPY --from=builder /opt/venv /opt/venv

# Copy application code
COPY . .

# Activate venv by updating PATH
ENV PATH="/opt/venv/bin:$PATH"

EXPOSE 5000
CMD ["python", "app.py"]
```

**Why use a virtual environment in Docker?** It isolates the installed packages into `/opt/venv`, making it easy to copy just the dependency layer cleanly into the next stage without contaminating the system Python.

**Size Comparison:**

| Base Image | Approximate Size |
|---|---|
| `python:3.12` (full) | ~1 GB |
| `python:3.12-slim` | ~130 MB |
| `python:3.12-alpine` | ~50 MB |

#### Python with Distroless

```dockerfile
FROM gcr.io/distroless/python3-debian12

WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY . .

ENV PATH="/opt/venv/bin:$PATH"
CMD ["app.py"]
```

> **Note:** Distroless Python images have limitations — they include a specific Python version and no `pip`. All dependencies must be pre-installed in an earlier stage.

---

## What Are Distroless Images?

**Distroless images**, maintained by Google, contain **only your application and its runtime dependencies** — no shell (`bash`/`sh`), no package manager (`apt`/`yum`), no system utilities.

> "Distroless" means the image strips out the Linux **distribution** layer, keeping only the bare minimum needed to run your app.

### What's Included vs Excluded

| Included ✅ | Excluded ❌ |
|---|---|
| Language runtime (Python, JRE, Node.js) | Shell (`bash`, `sh`) |
| `glibc` / system libraries | Package managers (`apt`, `yum`) |
| SSL/TLS certificates (`ca-certificates`) | Build tools (`gcc`, `make`) |
| Timezone data | `curl`, `wget`, `ls`, `grep` |

### Available Distroless Images

```bash
# Static (for Go/Rust compiled binaries)
gcr.io/distroless/static-debian12

# C/C++ runtime
gcr.io/distroless/base-debian12

# Java
gcr.io/distroless/java17-debian12
gcr.io/distroless/java21-debian12

# Python
gcr.io/distroless/python3-debian12

# Node.js
gcr.io/distroless/nodejs20-debian12
```

### Distroless Example (Go)

```dockerfile
# ---- Stage 1: Build ----
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

# ---- Stage 2: Distroless Runtime ----
FROM gcr.io/distroless/static-debian12

COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

### Security Advantages of Distroless

1. **No shell = no shell injection attacks** — attackers cannot execute arbitrary commands even if they gain container access
2. **Smaller CVE footprint** — fewer packages = fewer known vulnerabilities reported by scanners like Trivy or Snyk
3. **Compliance** — many security standards (CIS, NIST) encourage minimal-privilege, minimal-package containers
4. **Immutable runtime** — nothing can be installed at runtime

### Debugging Distroless Containers

Since there's no shell, use Docker's `--target` flag to build a debug variant:

```bash
# Google provides debug tags with busybox shell for inspection
FROM gcr.io/distroless/static-debian12:debug

# Then exec into it:
docker run --entrypoint=sh -it your-distroless-image
```

---

## Distroless vs scratch vs Alpine

| Feature | `scratch` | `distroless` | `alpine` |
|---|---|---|---|
| **Size** | 0 MB (empty) | ~2–20 MB | ~5 MB |
| **Shell** | ❌ None | ❌ None (debug tag has busybox) | ✅ `sh` (busybox) |
| **Package manager** | ❌ | ❌ | ✅ `apk` |
| **SSL certs** | ❌ (must copy manually) | ✅ Included | ✅ Included |
| **`glibc`** | ❌ | ✅ Included | ❌ (uses `musl`) |
| **Best for** | Statically compiled binaries (Go, Rust) | Production JVM/Python/Node apps | Apps needing shell access or `apk` |
| **Security** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Debugging ease** | Hard | Hard | Easy |

---

## When to Use Each Approach

| Scenario | Recommended Approach |
|---|---|
| Go/Rust statically compiled apps | `FROM scratch` |
| Java/Python/Node in production | `FROM gcr.io/distroless/<runtime>` |
| Need shell access for debugging | `FROM alpine` |
| General purpose small images | `FROM alpine` or `FROM debian:slim` |
| Dev/test environments | Full images (`ubuntu`, `python:3.12`) |
| Security-critical production deployments | Distroless |

---

## Quick Reference: Image Size Comparison

```
<img width="1123" height="540" alt="image" src="https://github.com/user-attachments/assets/0a9a1181-a424-4d77-bf6e-03640c36c9e1" />

Language   │ Single-Stage        │ Multi-Stage (Alpine) │ Multi-Stage (Distroless/Scratch)
───────────┼─────────────────────┼──────────────────────┼─────────────────────────────────
Go         │ ~700 MB (ubuntu)    │ ~12 MB               │ ~2 MB (scratch)
Java       │ ~500 MB (maven)     │ ~180 MB (jre-alpine) │ ~220 MB (distroless/java17)
Python     │ ~1 GB (python:3.12) │ ~130 MB (slim)       │ ~60 MB (distroless/python3)
Node.js    │ ~900 MB (node)      │ ~80 MB (alpine)      │ ~100 MB (distroless/nodejs)
```

> **Rule of thumb:** Use multi-stage builds always. For the final stage, prefer `scratch` for compiled languages, `distroless` for runtimes, and `alpine` when shell access is genuinely required.

---

## 📚 Further Reading

- [Docker Official: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Google Distroless GitHub](https://github.com/GoogleContainerTools/distroless)
- [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Trivy - Container vulnerability scanner](https://github.com/aquasecurity/trivy)

---

*This project is part of a hands-on demonstration of Docker image optimization using Go as the primary language.*
