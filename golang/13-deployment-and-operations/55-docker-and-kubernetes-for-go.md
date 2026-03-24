# Chapter 55: Docker and Kubernetes for Go Applications

## Table of Contents

1. [Introduction](#introduction)
2. [Building Docker Images for Go](#building-docker-images-for-go)
3. [Docker Compose for Local Development](#docker-compose-for-local-development)
4. [Health Check Endpoints for Containers](#health-check-endpoints-for-containers)
5. [Kubernetes Fundamentals for Go Developers](#kubernetes-fundamentals-for-go-developers)
6. [Graceful Shutdown in Kubernetes](#graceful-shutdown-in-kubernetes)
7. [Go Application Configuration in Kubernetes](#go-application-configuration-in-kubernetes)
8. [Building Kubernetes Operators in Go](#building-kubernetes-operators-in-go)
9. [client-go for Interacting with the Kubernetes API](#client-go-for-interacting-with-the-kubernetes-api)
10. [Helm Charts for Go Services](#helm-charts-for-go-services)
11. [Debugging Go in Containers](#debugging-go-in-containers)
12. [CI/CD Pipeline Example](#cicd-pipeline-example)
13. [Production Checklist](#production-checklist)
14. [Summary](#summary)

---

## Introduction

Go and containers are a natural pairing. Go compiles to a single static binary with no
runtime dependencies, making it ideal for minimal container images. Kubernetes, itself
written in Go, provides a powerful orchestration platform with first-class Go client
libraries.

This chapter walks through the full lifecycle of containerizing and deploying Go
applications: from writing production-grade Dockerfiles, through Kubernetes manifests and
operators, to CI/CD pipelines and debugging techniques.

**Prerequisites:**
- Familiarity with Go modules and building Go binaries
- Basic understanding of containers (what an image and container are)
- Docker and kubectl installed locally
- A Kubernetes cluster (minikube, kind, or a cloud provider)

---

## Building Docker Images for Go

### Why Containers for Go?

Go produces self-contained binaries, but containers provide:
- Reproducible builds across environments
- Consistent deployment artifacts
- Isolation and resource control
- Integration with orchestrators like Kubernetes

### The Naive Dockerfile (Don't Do This)

```dockerfile
# BAD EXAMPLE - results in a ~1 GB image
FROM golang:1.22

WORKDIR /app
COPY . .
RUN go build -o server .

EXPOSE 8080
CMD ["./server"]
```

**Problems with this approach:**

| Issue | Impact |
|-------|--------|
| Full Go toolchain in final image | Image is ~1 GB instead of ~10 MB |
| Source code shipped in image | Security risk, intellectual property exposure |
| No layer caching for dependencies | Every code change re-downloads all modules |
| Runs as root by default | Container escape vulnerability |

### Multi-Stage Builds: The Gold Standard

Multi-stage builds use one stage to compile and another to run. The final image contains
only the binary.

#### Pattern 1: Builder + Scratch

```dockerfile
# ---- Build Stage ----
FROM golang:1.22-alpine AS builder

# Install git and ca-certificates (needed for HTTPS and private repos)
RUN apk add --no-cache git ca-certificates tzdata

# Create a non-root user for the final image
RUN adduser -D -g '' appuser

WORKDIR /build

# Cache dependencies - copy go.mod and go.sum first
COPY go.mod go.sum ./
RUN go mod download
RUN go mod verify

# Copy source code
COPY . .

# Build a fully static binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static"' \
    -tags netgo \
    -o /build/server \
    ./cmd/server

# ---- Final Stage ----
FROM scratch

# Import CA certificates for HTTPS calls
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Import timezone data
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Import the non-root user
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group

# Copy the binary
COPY --from=builder /build/server /server

# Use the non-root user
USER appuser

EXPOSE 8080

ENTRYPOINT ["/server"]
```

> **Tip:** `scratch` is an empty image with literally nothing in it -- no shell, no
> package manager, no libc. This is the most secure option because there is nothing an
> attacker can exploit if they gain access to the container.

#### Pattern 2: Builder + Distroless

Google's distroless images are a middle ground: no shell or package manager, but they
include CA certificates, timezone data, and a non-root user out of the box.

```dockerfile
# ---- Build Stage ----
FROM golang:1.22 AS builder

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s' \
    -o /build/server \
    ./cmd/server

# ---- Final Stage ----
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /build/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

#### Comparison of Base Images

| Base Image | Size (approx.) | Shell | CA Certs | Non-root User | Debug Tools |
|------------|----------------|-------|----------|---------------|-------------|
| `golang:1.22` | ~850 MB | Yes | Yes | No | Yes |
| `golang:1.22-alpine` | ~250 MB | Yes | Yes | No | Minimal |
| `alpine:3.19` | ~7 MB | Yes | Yes | No | Minimal |
| `gcr.io/distroless/static` | ~2 MB | No | Yes | Available | No |
| `scratch` | 0 MB | No | No | No | No |

### Building Static Binaries: CGO_ENABLED=0

Go can link against C libraries via cgo. When `CGO_ENABLED=1` (the default on most
systems), the binary depends on libc and other shared libraries. Setting
`CGO_ENABLED=0` forces pure Go implementations.

```bash
# Standard build (may link against C libraries)
go build -o server ./cmd/server

# Fully static build (pure Go, no C dependencies)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static"' \
    -o server ./cmd/server
```

**Explanation of linker flags:**

| Flag | Purpose |
|------|---------|
| `-w` | Omit DWARF debug information (reduces binary size) |
| `-s` | Omit symbol table (reduces binary size further) |
| `-extldflags "-static"` | Pass `-static` to the external linker |

> **Warning:** Some packages require cgo (e.g., `github.com/mattn/go-sqlite3`). If you
> need cgo, use `alpine` with `musl-dev` installed instead of `scratch`:
>
> ```dockerfile
> FROM golang:1.22-alpine AS builder
> RUN apk add --no-cache gcc musl-dev
> RUN CGO_ENABLED=1 go build -o server ./cmd/server
>
> FROM alpine:3.19
> RUN apk add --no-cache ca-certificates
> COPY --from=builder /build/server /server
> CMD ["/server"]
> ```

### Injecting Build Information via ldflags

Embed version, commit hash, and build time into the binary at compile time:

```go
// cmd/server/main.go
package main

import (
    "fmt"
    "runtime/debug"
)

// Set via -ldflags at build time
var (
    version   = "dev"
    commit    = "unknown"
    buildTime = "unknown"
)

func main() {
    fmt.Printf("Version:    %s\n", version)
    fmt.Printf("Commit:     %s\n", commit)
    fmt.Printf("Build Time: %s\n", buildTime)

    // You can also read build info embedded by the Go toolchain
    if info, ok := debug.ReadBuildInfo(); ok {
        fmt.Printf("Go Version: %s\n", info.GoVersion)
        for _, setting := range info.Settings {
            if setting.Key == "vcs.revision" {
                fmt.Printf("VCS Revision: %s\n", setting.Value)
            }
        }
    }
}
```

```dockerfile
# In the build stage:
ARG VERSION=dev
RUN CGO_ENABLED=0 go build \
    -ldflags="-w -s \
      -X main.version=${VERSION} \
      -X main.commit=$(git rev-parse --short HEAD) \
      -X main.buildTime=$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
    -o /build/server ./cmd/server
```

Build with:

```bash
docker build --build-arg VERSION=1.2.3 -t myapp:1.2.3 .
```

### Caching Go Modules Layer

Docker builds layers top-to-bottom. If a layer changes, all subsequent layers are
rebuilt. By copying `go.mod` and `go.sum` before the source code, module downloads are
cached until dependencies actually change.

```dockerfile
# Step 1: Copy only dependency files
COPY go.mod go.sum ./

# Step 2: Download dependencies (cached unless go.mod/go.sum change)
RUN go mod download

# Step 3: Copy source code (changes frequently)
COPY . .

# Step 4: Build (runs every time source changes, but deps are cached)
RUN CGO_ENABLED=0 go build -o /server ./cmd/server
```

**Layer cache behavior:**

```
go.mod unchanged  --> reuse cached go mod download layer (fast)
go.mod changed    --> re-download all modules (slow, but necessary)
source changed    --> only rebuild the binary (fast, deps cached)
```

For monorepos with multiple `go.mod` files:

```dockerfile
# Copy all go.mod and go.sum files first
COPY go.work go.work.sum ./
COPY services/api/go.mod services/api/go.sum ./services/api/
COPY services/worker/go.mod services/worker/go.sum ./services/worker/
COPY pkg/shared/go.mod pkg/shared/go.sum ./pkg/shared/

RUN go mod download
```

### Using Docker BuildKit Cache Mounts

Docker BuildKit provides cache mounts that persist across builds, even when layers
are invalidated:

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.22 AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /server ./cmd/server

FROM scratch
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

> **Tip:** Enable BuildKit with `DOCKER_BUILDKIT=1 docker build ...` or set it as the
> default in Docker Desktop settings.

### .dockerignore Best Practices

A `.dockerignore` file prevents unnecessary files from being sent to the Docker daemon,
speeding up builds and reducing image size.

```dockerignore
# Version control
.git
.gitignore

# IDE and editor files
.vscode/
.idea/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Build artifacts
bin/
dist/
*.exe

# Test and development files
*_test.go
testdata/
coverage.out
coverage.html

# Documentation
*.md
docs/
LICENSE

# Docker files (prevent recursive builds)
Dockerfile*
docker-compose*.yml
.dockerignore

# CI/CD
.github/
.gitlab-ci.yml
Jenkinsfile

# Kubernetes manifests (not needed in image)
k8s/
helm/
*.yaml
!config.yaml

# Environment and secrets
.env
.env.*
*.pem
*.key
```

> **Warning:** Be careful excluding `*.yaml` -- if your application reads YAML config
> files at runtime, you need to whitelist them with `!config.yaml`.

### Multi-Architecture Builds

Build images for both `amd64` and `arm64` (e.g., for AWS Graviton or Apple Silicon):

```bash
# Create a builder that supports multiple platforms
docker buildx create --name multiarch --use

# Build and push for multiple architectures
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --tag myregistry/myapp:1.0.0 \
    --push \
    .
```

The Dockerfile handles this automatically when using Go's cross-compilation:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22 AS builder

ARG TARGETOS
ARG TARGETARCH

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .

RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags='-w -s' -o /server ./cmd/server

FROM --platform=$TARGETPLATFORM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

### Image Security Scanning

Always scan your images for vulnerabilities before deploying:

```bash
# Using Docker Scout (built into Docker Desktop)
docker scout cves myapp:latest

# Using Trivy (open source)
trivy image myapp:latest

# Using Snyk
snyk container test myapp:latest
```

---

## Docker Compose for Local Development

Docker Compose orchestrates multiple containers locally, simulating a production-like
environment with databases, caches, and message queues.

### Basic docker-compose.yml

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder          # Stop at the builder stage for hot reload
    ports:
      - "8080:8080"
      - "2345:2345"            # Delve debugger port
    environment:
      - DATABASE_URL=postgres://app:secret@postgres:5432/myapp?sslmode=disable
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=debug
    volumes:
      - .:/build               # Mount source for hot reload
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - backend

  migrate:
    image: migrate/migrate:v4.17.0
    volumes:
      - ./migrations:/migrations
    command: [
      "-path", "/migrations",
      "-database", "postgres://app:secret@postgres:5432/myapp?sslmode=disable",
      "up"
    ]
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend

volumes:
  postgres_data:
  redis_data:

networks:
  backend:
    driver: bridge
```

### Development Dockerfile with Hot Reload

Use [Air](https://github.com/air-verse/air) for live reloading during development:

```dockerfile
# Dockerfile.dev
FROM golang:1.22-alpine

RUN go install github.com/air-verse/air@latest
RUN go install github.com/go-delve/delve/cmd/dlv@latest

WORKDIR /build

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Air watches for file changes and rebuilds
CMD ["air", "-c", ".air.toml"]
```

The Air configuration:

```toml
# .air.toml
root = "."
tmp_dir = "tmp"

[build]
  bin = "./tmp/server"
  cmd = "go build -gcflags='all=-N -l' -o ./tmp/server ./cmd/server"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata", "node_modules"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html", "yaml", "toml"]
  kill_delay = "2s"
  log = "build-errors.log"
  send_interrupt = true
  stop_on_error = true

[log]
  time = false

[color]
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"
```

Updated compose file for development:

```yaml
# docker-compose.dev.yml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
      - "2345:2345"
    environment:
      - DATABASE_URL=postgres://app:secret@postgres:5432/myapp?sslmode=disable
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=debug
    volumes:
      - .:/build
      - go_modules:/go/pkg/mod       # Cache modules across rebuilds
      - go_build_cache:/root/.cache   # Cache build artifacts
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

volumes:
  go_modules:
  go_build_cache:
```

Run with:

```bash
docker compose -f docker-compose.dev.yml up --build
```

### Useful Compose Commands

```bash
# Start all services
docker compose up -d

# Rebuild and start
docker compose up --build -d

# View logs
docker compose logs -f api

# Run a one-off command
docker compose exec api go test ./...

# Stop everything
docker compose down

# Stop and remove volumes (clean slate)
docker compose down -v
```

---

## Health Check Endpoints for Containers

Health checks let orchestrators (Docker, Kubernetes) know whether your application is
functioning correctly. A well-designed Go service exposes multiple health endpoints.

### Comprehensive Health Check Implementation

```go
// internal/health/health.go
package health

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
)

// Status represents the health status of a component.
type Status string

const (
    StatusUp       Status = "up"
    StatusDown     Status = "down"
    StatusDegraded Status = "degraded"
)

// ComponentHealth represents the health of a single dependency.
type ComponentHealth struct {
    Status   Status        `json:"status"`
    Latency  string        `json:"latency,omitempty"`
    Error    string        `json:"error,omitempty"`
    Details  interface{}   `json:"details,omitempty"`
}

// HealthResponse is the full health check response.
type HealthResponse struct {
    Status     Status                      `json:"status"`
    Version    string                      `json:"version"`
    Uptime     string                      `json:"uptime"`
    Components map[string]ComponentHealth  `json:"components"`
}

// Checker performs health checks on application dependencies.
type Checker struct {
    db        *sql.DB
    redis     *redis.Client
    version   string
    startTime time.Time
}

// NewChecker creates a new health checker.
func NewChecker(db *sql.DB, redis *redis.Client, version string) *Checker {
    return &Checker{
        db:        db,
        redis:     redis,
        version:   version,
        startTime: time.Now(),
    }
}

// checkDatabase verifies database connectivity.
func (c *Checker) checkDatabase(ctx context.Context) ComponentHealth {
    start := time.Now()

    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    if err := c.db.PingContext(ctx); err != nil {
        return ComponentHealth{
            Status:  StatusDown,
            Latency: time.Since(start).String(),
            Error:   fmt.Sprintf("ping failed: %v", err),
        }
    }

    // Check connection pool stats
    stats := c.db.Stats()
    status := StatusUp
    if stats.OpenConnections >= stats.MaxOpenConnections {
        status = StatusDegraded
    }

    return ComponentHealth{
        Status:  status,
        Latency: time.Since(start).String(),
        Details: map[string]interface{}{
            "open_connections": stats.OpenConnections,
            "max_connections":  stats.MaxOpenConnections,
            "in_use":           stats.InUse,
            "idle":             stats.Idle,
        },
    }
}

// checkRedis verifies Redis connectivity.
func (c *Checker) checkRedis(ctx context.Context) ComponentHealth {
    start := time.Now()

    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    if err := c.redis.Ping(ctx).Err(); err != nil {
        return ComponentHealth{
            Status:  StatusDown,
            Latency: time.Since(start).String(),
            Error:   fmt.Sprintf("ping failed: %v", err),
        }
    }

    return ComponentHealth{
        Status:  StatusUp,
        Latency: time.Since(start).String(),
    }
}

// LivenessHandler is a minimal check: "Is the process alive?"
// This should NOT check external dependencies.
func (c *Checker) LivenessHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "alive",
        })
    }
}

// ReadinessHandler checks if the app is ready to receive traffic.
// This SHOULD check external dependencies.
func (c *Checker) ReadinessHandler() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        response := HealthResponse{
            Version:    c.version,
            Uptime:     time.Since(c.startTime).String(),
            Components: make(map[string]ComponentHealth),
        }

        // Run checks concurrently
        var wg sync.WaitGroup
        var mu sync.Mutex

        checks := map[string]func(context.Context) ComponentHealth{
            "database": c.checkDatabase,
            "redis":    c.checkRedis,
        }

        for name, check := range checks {
            wg.Add(1)
            go func(name string, check func(context.Context) ComponentHealth) {
                defer wg.Done()
                result := check(ctx)
                mu.Lock()
                response.Components[name] = result
                mu.Unlock()
            }(name, check)
        }

        wg.Wait()

        // Determine overall status
        response.Status = StatusUp
        for _, comp := range response.Components {
            if comp.Status == StatusDown {
                response.Status = StatusDown
                break
            }
            if comp.Status == StatusDegraded {
                response.Status = StatusDegraded
            }
        }

        w.Header().Set("Content-Type", "application/json")
        if response.Status == StatusDown {
            w.WriteHeader(http.StatusServiceUnavailable)
        } else {
            w.WriteHeader(http.StatusOK)
        }
        json.NewEncoder(w).Encode(response)
    }
}

// StartupHandler checks if the app has finished initializing.
func (c *Checker) StartupHandler() http.HandlerFunc {
    var (
        ready     bool
        readyOnce sync.Once
    )

    return func(w http.ResponseWriter, r *http.Request) {
        readyOnce.Do(func() {
            ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
            defer cancel()

            // Verify initial connectivity
            if err := c.db.PingContext(ctx); err == nil {
                if err := c.redis.Ping(ctx).Err(); err == nil {
                    ready = true
                }
            }
        })

        w.Header().Set("Content-Type", "application/json")
        if ready {
            w.WriteHeader(http.StatusOK)
            json.NewEncoder(w).Encode(map[string]string{"status": "started"})
        } else {
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]string{"status": "starting"})
        }
    }
}
```

### Registering Health Endpoints

```go
// cmd/server/main.go
package main

import (
    "database/sql"
    "log/slog"
    "net/http"
    "os"

    "myapp/internal/health"
    "github.com/redis/go-redis/v9"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // ... initialize db and redis ...

    checker := health.NewChecker(db, redisClient, version)

    // Health check routes (separate from main API mux)
    healthMux := http.NewServeMux()
    healthMux.HandleFunc("GET /healthz",  checker.LivenessHandler())
    healthMux.HandleFunc("GET /readyz",   checker.ReadinessHandler())
    healthMux.HandleFunc("GET /startupz", checker.StartupHandler())

    // Run health server on a separate port
    go func() {
        logger.Info("health server starting", "port", 8081)
        if err := http.ListenAndServe(":8081", healthMux); err != nil {
            logger.Error("health server failed", "error", err)
        }
    }()

    // Main application server
    apiMux := http.NewServeMux()
    // ... register API routes ...

    logger.Info("api server starting", "port", 8080)
    if err := http.ListenAndServe(":8080", apiMux); err != nil {
        logger.Error("api server failed", "error", err)
    }
}
```

> **Tip:** Running health checks on a separate port (e.g., 8081) prevents health check
> traffic from competing with application traffic and allows different network policies.

### Docker Health Check

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]
```

Implement a self-check mode in your binary:

```go
func main() {
    if len(os.Args) > 1 && os.Args[1] == "-health-check" {
        resp, err := http.Get("http://localhost:8081/healthz")
        if err != nil || resp.StatusCode != http.StatusOK {
            os.Exit(1)
        }
        os.Exit(0)
    }

    // ... normal server startup ...
}
```

---

## Kubernetes Fundamentals for Go Developers

### Core Resource Overview

```
              Ingress
                |
            Service (ClusterIP / LoadBalancer)
                |
          Deployment (manages ReplicaSets)
                |
        ReplicaSet (manages Pods)
                |
    Pod [Container(s)] --- ConfigMap, Secret (config)
                           PersistentVolumeClaim (storage)
```

### Deployment

A Deployment declares the desired state for your application Pods.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: production
  labels:
    app: myapp
    component: api
    version: v1.2.3
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: myapp
      component: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Allow 1 extra pod during update
      maxUnavailable: 0     # Never have fewer than desired replicas
  template:
    metadata:
      labels:
        app: myapp
        component: api
        version: v1.2.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: myapp-api
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534     # nobody
        fsGroup: 65534
      containers:
        - name: api
          image: myregistry/myapp-api:v1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: health
              containerPort: 8081
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: myapp-config
                  key: log_level
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: database_url
          envFrom:
            - configMapRef:
                name: myapp-env-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          # Probes (detailed below)
          startupProbe:
            httpGet:
              path: /startupz
              port: health
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 12    # 5 + (5 * 12) = 65 seconds max startup
          livenessProbe:
            httpGet:
              path: /healthz
              port: health
            periodSeconds: 15
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: health
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
              component: api
```

### Service

A Service provides stable networking for Pods behind a Deployment.

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
  namespace: production
  labels:
    app: myapp
    component: api
spec:
  type: ClusterIP
  selector:
    app: myapp
    component: api
  ports:
    - name: http
      port: 80
      targetPort: http    # References the named port on the container
      protocol: TCP
    - name: health
      port: 8081
      targetPort: health
      protocol: TCP
---
# External access via LoadBalancer (cloud environments)
apiVersion: v1
kind: Service
metadata:
  name: myapp-api-external
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: myapp
    component: api
  ports:
    - name: http
      port: 80
      targetPort: http
```

### ConfigMap

ConfigMaps hold non-sensitive configuration data.

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  log_level: "info"
  max_connections: "100"
  cache_ttl: "300s"
  feature_flags: |
    enable_new_ui=true
    enable_experimental_api=false
---
# ConfigMap with environment variables
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-env-config
  namespace: production
data:
  APP_ENV: production
  APP_PORT: "8080"
  HEALTH_PORT: "8081"
  METRICS_PORT: "9090"
---
# ConfigMap from a config file
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-file-config
  namespace: production
data:
  config.yaml: |
    server:
      port: 8080
      read_timeout: 10s
      write_timeout: 30s
      idle_timeout: 120s
    database:
      max_open_conns: 25
      max_idle_conns: 5
      conn_max_lifetime: 5m
    cache:
      ttl: 5m
      max_size: 1000
```

Mount the config file into the container:

```yaml
# In the deployment spec:
containers:
  - name: api
    volumeMounts:
      - name: config
        mountPath: /etc/myapp
        readOnly: true
volumes:
  - name: config
    configMap:
      name: myapp-file-config
```

### Secrets

Secrets hold sensitive data like passwords, tokens, and certificates.

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
stringData:                          # stringData accepts plain text
  database_url: "postgres://user:pass@db:5432/myapp?sslmode=require"
  redis_url: "redis://:authpass@redis:6379"
  jwt_signing_key: "super-secret-key-change-in-production"
```

> **Warning:** Kubernetes Secrets are only base64-encoded, not encrypted. For
> production, use:
> - **Sealed Secrets** (Bitnami) -- encrypts secrets in Git
> - **External Secrets Operator** -- syncs from AWS Secrets Manager, HashiCorp Vault, etc.
> - **SOPS** -- encrypts secret files with cloud KMS

### Liveness, Readiness, and Startup Probes

Understanding the three probe types is critical for Go applications in Kubernetes.

| Probe | Question It Answers | Failure Action | Check External Deps? |
|-------|---------------------|----------------|----------------------|
| **Startup** | Has the app finished initializing? | Keep waiting (don't kill) | Yes |
| **Liveness** | Is the process healthy? | Restart the pod | **No** |
| **Readiness** | Can it handle traffic right now? | Remove from Service endpoints | Yes |

**Probe configuration guidelines for Go apps:**

```yaml
# Startup probe: allows slow initialization
startupProbe:
  httpGet:
    path: /startupz
    port: 8081
  initialDelaySeconds: 2
  periodSeconds: 5
  failureThreshold: 12       # Total: 2 + (5 * 12) = 62s max

# Liveness probe: detects deadlocks and hangs
# KEEP THIS SIMPLE -- only check if the Go process can respond
livenessProbe:
  httpGet:
    path: /healthz
    port: 8081
  initialDelaySeconds: 0     # Runs after startup probe succeeds
  periodSeconds: 15
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3        # Restart after 3 consecutive failures

# Readiness probe: checks if dependencies are available
readinessProbe:
  httpGet:
    path: /readyz
    port: 8081
  initialDelaySeconds: 0
  periodSeconds: 10
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 2        # Remove from LB after 2 failures
```

> **Warning:** A common mistake is making the liveness probe check database connectivity.
> If the database goes down, Kubernetes restarts all your pods -- which makes the
> situation worse. Liveness should only verify the Go process itself is healthy (not
> deadlocked, not out of memory).

### Resource Requests and Limits for Go Apps

Go's garbage collector and goroutine scheduler interact with container resource limits
in important ways.

```yaml
resources:
  requests:
    cpu: 100m          # 0.1 CPU core
    memory: 128Mi      # 128 MiB
  limits:
    cpu: 500m          # 0.5 CPU core (consider omitting -- see below)
    memory: 256Mi      # 256 MiB (ALWAYS set this)
```

**Memory considerations:**

Go's runtime reads `GOMEMLIMIT` (Go 1.19+) to understand the memory budget. Without it,
the GC does not know about the container memory limit.

```yaml
env:
  - name: GOMAXPROCS
    value: "2"              # Match CPU request/limit
  - name: GOMEMLIMIT
    value: "230MiB"         # ~90% of memory limit (leave room for non-heap)
```

Alternatively, use the `automaxprocs` and `automemlimit` libraries:

```go
package main

import (
    _ "go.uber.org/automaxprocs"          // Automatically set GOMAXPROCS
    _ "github.com/KimMachineGun/automemlimit" // Automatically set GOMEMLIMIT
)

func main() {
    // GOMAXPROCS and GOMEMLIMIT are now set based on cgroup limits
    // ...
}
```

**CPU considerations:**

| Setting | Behavior |
|---------|----------|
| **Requests only** (no CPU limit) | Pod gets guaranteed minimum, can burst higher |
| **Requests + Limits** | Pod is throttled when it exceeds the limit |
| **`GOMAXPROCS` not set** | Go sees all host CPUs, creates too many OS threads |

> **Tip:** Many teams omit CPU limits and set only CPU requests. CPU limits cause
> throttling (CFS quota), which can add unpredictable latency to Go applications. See
> the "CPU limits are harmful" debate in the Kubernetes community.

### Horizontal Pod Autoscaler

HPA automatically scales the number of replicas based on metrics.

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  minReplicas: 3
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60      # Add up to 4 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60      # Remove up to 25% of pods per minute
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metric from Prometheus (e.g., requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

Expose custom metrics from your Go app for HPA:

```go
// internal/metrics/metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    HTTPRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    HTTPRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Duration of HTTP requests",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )

    ActiveConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)
```

---

## Graceful Shutdown in Kubernetes

When Kubernetes terminates a pod (during rolling updates, scale-down, or node
maintenance), it follows a specific sequence. Your Go application must handle this
correctly to avoid dropped requests.

### The Kubernetes Pod Termination Sequence

```
1. Pod is marked for termination
2. Pod is removed from Service endpoints (async)
3. preStop hook runs (if configured)
4. SIGTERM is sent to PID 1
5. App has terminationGracePeriodSeconds to shut down (default: 30s)
6. SIGKILL is sent if app hasn't exited
```

> **Warning:** Steps 2 and 3-4 happen concurrently. There is a race condition: the
> SIGTERM may arrive before the pod is removed from the load balancer. The `preStop`
> hook with a small sleep (3-5 seconds) gives the endpoints controller time to update.

### Production-Grade Graceful Shutdown in Go

```go
// cmd/server/main.go
package main

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))

    // Initialize dependencies
    db, err := initDB()
    if err != nil {
        logger.Error("failed to initialize database", "error", err)
        os.Exit(1)
    }

    // Create servers
    apiServer := &http.Server{
        Addr:         ":8080",
        Handler:      newAPIRouter(db, logger),
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    healthServer := &http.Server{
        Addr:    ":8081",
        Handler: newHealthRouter(db, logger),
    }

    metricsServer := &http.Server{
        Addr:    ":9090",
        Handler: newMetricsRouter(),
    }

    // Channel to signal when servers are ready
    serverErrors := make(chan error, 3)

    // Start servers
    go func() {
        logger.Info("starting API server", "addr", apiServer.Addr)
        if err := apiServer.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            serverErrors <- fmt.Errorf("API server error: %w", err)
        }
    }()

    go func() {
        logger.Info("starting health server", "addr", healthServer.Addr)
        if err := healthServer.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            serverErrors <- fmt.Errorf("health server error: %w", err)
        }
    }()

    go func() {
        logger.Info("starting metrics server", "addr", metricsServer.Addr)
        if err := metricsServer.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            serverErrors <- fmt.Errorf("metrics server error: %w", err)
        }
    }()

    // Wait for shutdown signal or server error
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    select {
    case sig := <-quit:
        logger.Info("shutdown signal received", "signal", sig.String())
    case err := <-serverErrors:
        logger.Error("server error, initiating shutdown", "error", err)
    }

    // Begin graceful shutdown
    // Give Kubernetes time to update endpoints (matches preStop sleep)
    logger.Info("waiting for in-flight requests to be routed elsewhere")

    // Create a deadline for the shutdown process
    shutdownTimeout := 25 * time.Second // Less than terminationGracePeriodSeconds
    ctx, cancel := context.WithTimeout(context.Background(), shutdownTimeout)
    defer cancel()

    // Shutdown servers concurrently
    var wg sync.WaitGroup

    // First: stop accepting new requests on the health endpoint
    // This causes readiness probe to fail, removing pod from LB
    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := healthServer.Shutdown(ctx); err != nil {
            logger.Error("health server shutdown error", "error", err)
        }
        logger.Info("health server stopped")
    }()

    // Brief pause for load balancer to update
    time.Sleep(3 * time.Second)

    // Then: stop the API server (waits for in-flight requests)
    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := apiServer.Shutdown(ctx); err != nil {
            logger.Error("API server shutdown error", "error", err)
        }
        logger.Info("API server stopped")
    }()

    // Stop metrics server
    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := metricsServer.Shutdown(ctx); err != nil {
            logger.Error("metrics server shutdown error", "error", err)
        }
        logger.Info("metrics server stopped")
    }()

    wg.Wait()

    // Close database connections
    if err := db.Close(); err != nil {
        logger.Error("database close error", "error", err)
    }
    logger.Info("database connections closed")

    logger.Info("shutdown complete")
}

func initDB() (*sql.DB, error) {
    dsn := os.Getenv("DATABASE_URL")
    if dsn == "" {
        return nil, fmt.Errorf("DATABASE_URL is required")
    }

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }

    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(1 * time.Minute)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("database ping failed: %w", err)
    }

    return db, nil
}
```

### Connection Draining Middleware

```go
// internal/middleware/draining.go
package middleware

import (
    "net/http"
    "sync/atomic"
)

// Draining is middleware that tracks in-flight requests
// and can reject new ones during shutdown.
type Draining struct {
    inFlight  atomic.Int64
    draining  atomic.Bool
}

func NewDraining() *Draining {
    return &Draining{}
}

// Middleware wraps an http.Handler with connection draining support.
func (d *Draining) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if d.draining.Load() {
            w.Header().Set("Connection", "close")
            http.Error(w, "server is shutting down", http.StatusServiceUnavailable)
            return
        }

        d.inFlight.Add(1)
        defer d.inFlight.Add(-1)

        next.ServeHTTP(w, r)
    })
}

// StartDraining marks the server as draining.
func (d *Draining) StartDraining() {
    d.draining.Store(true)
}

// InFlight returns the number of in-flight requests.
func (d *Draining) InFlight() int64 {
    return d.inFlight.Load()
}
```

### Kubernetes Manifest for Graceful Shutdown

```yaml
spec:
  terminationGracePeriodSeconds: 30     # Total time allowed
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            # Sleep to let endpoints controller remove pod from Service
            command: ["/bin/sh", "-c", "sleep 5"]
```

**Timeline of a graceful shutdown:**

```
T+0s   SIGTERM received, preStop starts (sleep 5)
T+0s   Endpoints controller begins removing pod from Service
T+5s   preStop finishes, app begins draining
T+5s   Health server shuts down (readiness fails)
T+8s   API server stops accepting new connections
T+8s   In-flight requests complete
T+10s  Database connections close
T+10s  Process exits cleanly
T+30s  SIGKILL would fire (but we exited at T+10s)
```

---

## Go Application Configuration in Kubernetes

### The Configuration Hierarchy

Go applications in Kubernetes typically load configuration from multiple sources,
with later sources overriding earlier ones:

```
1. Compiled defaults (in Go code)
2. Config file (mounted from ConfigMap)
3. Environment variables (from ConfigMap/Secret/Pod spec)
4. Command-line flags
5. Remote config (Consul, etcd -- optional)
```

### A Flexible Configuration Package

```go
// internal/config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"

    "gopkg.in/yaml.v3"
)

// Config holds all application configuration.
type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Redis    RedisConfig    `yaml:"redis"`
    Auth     AuthConfig     `yaml:"auth"`
    Feature  FeatureConfig  `yaml:"features"`
}

type ServerConfig struct {
    Port            int           `yaml:"port"`
    HealthPort      int           `yaml:"health_port"`
    MetricsPort     int           `yaml:"metrics_port"`
    ReadTimeout     time.Duration `yaml:"read_timeout"`
    WriteTimeout    time.Duration `yaml:"write_timeout"`
    IdleTimeout     time.Duration `yaml:"idle_timeout"`
    ShutdownTimeout time.Duration `yaml:"shutdown_timeout"`
}

type DatabaseConfig struct {
    URL             string        `yaml:"url"`
    MaxOpenConns    int           `yaml:"max_open_conns"`
    MaxIdleConns    int           `yaml:"max_idle_conns"`
    ConnMaxLifetime time.Duration `yaml:"conn_max_lifetime"`
    ConnMaxIdleTime time.Duration `yaml:"conn_max_idle_time"`
}

type RedisConfig struct {
    URL          string        `yaml:"url"`
    PoolSize     int           `yaml:"pool_size"`
    ReadTimeout  time.Duration `yaml:"read_timeout"`
    WriteTimeout time.Duration `yaml:"write_timeout"`
}

type AuthConfig struct {
    JWTSigningKey string        `yaml:"jwt_signing_key"`
    TokenTTL      time.Duration `yaml:"token_ttl"`
}

type FeatureConfig struct {
    EnableNewUI          bool `yaml:"enable_new_ui"`
    EnableExperimentalAPI bool `yaml:"enable_experimental_api"`
}

// DefaultConfig returns the compiled-in defaults.
func DefaultConfig() Config {
    return Config{
        Server: ServerConfig{
            Port:            8080,
            HealthPort:      8081,
            MetricsPort:     9090,
            ReadTimeout:     10 * time.Second,
            WriteTimeout:    30 * time.Second,
            IdleTimeout:     120 * time.Second,
            ShutdownTimeout: 25 * time.Second,
        },
        Database: DatabaseConfig{
            MaxOpenConns:    25,
            MaxIdleConns:    5,
            ConnMaxLifetime: 5 * time.Minute,
            ConnMaxIdleTime: 1 * time.Minute,
        },
        Redis: RedisConfig{
            PoolSize:     10,
            ReadTimeout:  3 * time.Second,
            WriteTimeout: 3 * time.Second,
        },
        Auth: AuthConfig{
            TokenTTL: 24 * time.Hour,
        },
    }
}

// Load reads configuration from a YAML file and then overlays
// environment variables.
func Load(configPath string) (Config, error) {
    cfg := DefaultConfig()

    // Layer 1: Load from YAML file if it exists
    if configPath != "" {
        data, err := os.ReadFile(configPath)
        if err != nil {
            return cfg, fmt.Errorf("reading config file: %w", err)
        }
        if err := yaml.Unmarshal(data, &cfg); err != nil {
            return cfg, fmt.Errorf("parsing config file: %w", err)
        }
    }

    // Layer 2: Override with environment variables
    // Sensitive values should ONLY come from env vars / secrets
    if v := os.Getenv("SERVER_PORT"); v != "" {
        if port, err := strconv.Atoi(v); err == nil {
            cfg.Server.Port = port
        }
    }
    if v := os.Getenv("DATABASE_URL"); v != "" {
        cfg.Database.URL = v
    }
    if v := os.Getenv("REDIS_URL"); v != "" {
        cfg.Redis.URL = v
    }
    if v := os.Getenv("JWT_SIGNING_KEY"); v != "" {
        cfg.Auth.JWTSigningKey = v
    }
    if v := os.Getenv("DATABASE_MAX_OPEN_CONNS"); v != "" {
        if n, err := strconv.Atoi(v); err == nil {
            cfg.Database.MaxOpenConns = n
        }
    }

    // Validate required fields
    if cfg.Database.URL == "" {
        return cfg, fmt.Errorf("DATABASE_URL is required")
    }

    return cfg, nil
}
```

### Using the Config in Kubernetes

```yaml
# Non-sensitive config in a ConfigMap (mounted as a file)
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-file-config
data:
  config.yaml: |
    server:
      port: 8080
      read_timeout: 10s
      write_timeout: 30s
    database:
      max_open_conns: 50
      max_idle_conns: 10
    features:
      enable_new_ui: true

---
# Sensitive config in a Secret (injected as env vars)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
stringData:
  DATABASE_URL: "postgres://user:pass@db:5432/app"
  JWT_SIGNING_KEY: "my-secret-key"

---
# Deployment excerpt
spec:
  containers:
    - name: api
      args: ["--config", "/etc/myapp/config.yaml"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: DATABASE_URL
        - name: JWT_SIGNING_KEY
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: JWT_SIGNING_KEY
      volumeMounts:
        - name: config
          mountPath: /etc/myapp
          readOnly: true
  volumes:
    - name: config
      configMap:
        name: myapp-file-config
```

### Watching ConfigMap Changes at Runtime

When a ConfigMap mounted as a volume is updated, Kubernetes eventually updates the
file in the pod (with a delay of up to 60 seconds by default). Your Go app can watch
for changes:

```go
// internal/config/watcher.go
package config

import (
    "log/slog"
    "os"
    "sync"
    "time"

    "gopkg.in/yaml.v3"
)

// Watcher watches a config file for changes and reloads it.
type Watcher struct {
    path     string
    mu       sync.RWMutex
    current  Config
    onChange []func(Config)
    logger   *slog.Logger
}

func NewWatcher(path string, initial Config, logger *slog.Logger) *Watcher {
    return &Watcher{
        path:    path,
        current: initial,
        logger:  logger,
    }
}

// OnChange registers a callback for config changes.
func (w *Watcher) OnChange(fn func(Config)) {
    w.onChange = append(w.onChange, fn)
}

// Current returns the current configuration.
func (w *Watcher) Current() Config {
    w.mu.RLock()
    defer w.mu.RUnlock()
    return w.current
}

// Start begins watching the config file. It blocks until the
// stop channel is closed.
func (w *Watcher) Start(stop <-chan struct{}) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    var lastModTime time.Time

    for {
        select {
        case <-stop:
            w.logger.Info("config watcher stopped")
            return
        case <-ticker.C:
            info, err := os.Stat(w.path)
            if err != nil {
                w.logger.Error("failed to stat config file", "error", err)
                continue
            }

            if info.ModTime().Equal(lastModTime) {
                continue // No change
            }

            lastModTime = info.ModTime()

            data, err := os.ReadFile(w.path)
            if err != nil {
                w.logger.Error("failed to read config file", "error", err)
                continue
            }

            var newCfg Config
            if err := yaml.Unmarshal(data, &newCfg); err != nil {
                w.logger.Error("failed to parse config file", "error", err)
                continue
            }

            w.mu.Lock()
            w.current = newCfg
            w.mu.Unlock()

            w.logger.Info("config reloaded", "mod_time", lastModTime)

            for _, fn := range w.onChange {
                fn(newCfg)
            }
        }
    }
}
```

---

## Building Kubernetes Operators in Go

Kubernetes operators extend the Kubernetes API with custom resources and controllers.
Go is the primary language for building operators, using the `controller-runtime`
library and the `kubebuilder` scaffolding tool.

### What Is an Operator?

An operator encodes operational knowledge into software. Instead of a human running
kubectl commands to manage a service, an operator watches for custom resources and
reconciles them to the desired state.

```
Custom Resource (CR)      -->  Controller  -->  Kubernetes Resources
(what the user wants)          (your Go code)   (what gets created)

Example:
MyDatabase CR             -->  Database     -->  StatefulSet, Service,
  name: prod-db                Controller        PVC, ConfigMap, Secret
  replicas: 3
  version: 16
```

### Setting Up with Kubebuilder

```bash
# Install kubebuilder
go install sigs.k8s.io/kubebuilder/v4/cmd/kubebuilder@latest

# Initialize a new project
mkdir myapp-operator && cd myapp-operator
kubebuilder init --domain example.com --repo github.com/myorg/myapp-operator

# Create an API (custom resource + controller)
kubebuilder create api --group apps --version v1alpha1 --kind MyApp
```

This generates the following structure:

```
myapp-operator/
  api/
    v1alpha1/
      myapp_types.go          # Custom Resource Definition (CRD) types
      groupversion_info.go
      zz_generated.deepcopy.go
  cmd/
    main.go                   # Entry point
  internal/
    controller/
      myapp_controller.go     # Reconciliation logic
  config/
    crd/                      # Generated CRD manifests
    manager/                  # Manager deployment manifests
    rbac/                     # RBAC rules
```

### Defining the Custom Resource

```go
// api/v1alpha1/myapp_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MyAppSpec defines the desired state of MyApp.
type MyAppSpec struct {
    // Replicas is the desired number of application instances.
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=20
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas"`

    // Image is the container image to deploy.
    // +kubebuilder:validation:MinLength=1
    Image string `json:"image"`

    // Port is the container port to expose.
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=65535
    // +kubebuilder:default=8080
    Port int32 `json:"port,omitempty"`

    // Resources defines the compute resources for each replica.
    // +optional
    Resources *ResourceSpec `json:"resources,omitempty"`

    // Database configures an optional database connection.
    // +optional
    Database *DatabaseSpec `json:"database,omitempty"`
}

type ResourceSpec struct {
    CPURequest    string `json:"cpuRequest,omitempty"`
    MemoryRequest string `json:"memoryRequest,omitempty"`
    CPULimit      string `json:"cpuLimit,omitempty"`
    MemoryLimit   string `json:"memoryLimit,omitempty"`
}

type DatabaseSpec struct {
    // SecretName is the name of the Secret containing DATABASE_URL.
    SecretName string `json:"secretName"`
    // Key is the key within the Secret. Defaults to "database_url".
    // +kubebuilder:default="database_url"
    Key string `json:"key,omitempty"`
}

// MyAppStatus defines the observed state of MyApp.
type MyAppStatus struct {
    // ReadyReplicas is the number of replicas that are ready.
    ReadyReplicas int32 `json:"readyReplicas"`

    // Conditions represent the latest observations of the resource's state.
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // ObservedGeneration is the most recent generation observed.
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`

// MyApp is the Schema for the myapps API.
type MyApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MyAppSpec   `json:"spec,omitempty"`
    Status MyAppStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// MyAppList contains a list of MyApp.
type MyAppList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []MyApp `json:"items"`
}

func init() {
    SchemeBuilder.Register(&MyApp{}, &MyAppList{})
}
```

### Writing the Controller (Reconciliation Loop)

```go
// internal/controller/myapp_controller.go
package controller

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/equality"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/apimachinery/pkg/util/intstr"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    appsv1alpha1 "github.com/myorg/myapp-operator/api/v1alpha1"
)

const myAppFinalizer = "apps.example.com/finalizer"

// MyAppReconciler reconciles a MyApp object.
type MyAppReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=apps.example.com,resources=myapps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps.example.com,resources=myapps/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete

// Reconcile is the main reconciliation loop.
func (r *MyAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Fetch the MyApp resource
    var myapp appsv1alpha1.MyApp
    if err := r.Get(ctx, req.NamespacedName, &myapp); err != nil {
        if apierrors.IsNotFound(err) {
            logger.Info("MyApp resource not found, likely deleted")
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, fmt.Errorf("fetching MyApp: %w", err)
    }

    // Handle deletion with finalizer
    if !myapp.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&myapp, myAppFinalizer) {
            // Perform cleanup logic here (e.g., delete external resources)
            logger.Info("performing cleanup for MyApp")

            controllerutil.RemoveFinalizer(&myapp, myAppFinalizer)
            if err := r.Update(ctx, &myapp); err != nil {
                return ctrl.Result{}, fmt.Errorf("removing finalizer: %w", err)
            }
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(&myapp, myAppFinalizer) {
        controllerutil.AddFinalizer(&myapp, myAppFinalizer)
        if err := r.Update(ctx, &myapp); err != nil {
            return ctrl.Result{}, fmt.Errorf("adding finalizer: %w", err)
        }
    }

    // Reconcile the Deployment
    if err := r.reconcileDeployment(ctx, &myapp); err != nil {
        return ctrl.Result{}, fmt.Errorf("reconciling deployment: %w", err)
    }

    // Reconcile the Service
    if err := r.reconcileService(ctx, &myapp); err != nil {
        return ctrl.Result{}, fmt.Errorf("reconciling service: %w", err)
    }

    // Update status
    if err := r.updateStatus(ctx, &myapp); err != nil {
        return ctrl.Result{}, fmt.Errorf("updating status: %w", err)
    }

    // Requeue after 30 seconds for periodic reconciliation
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *MyAppReconciler) reconcileDeployment(
    ctx context.Context,
    myapp *appsv1alpha1.MyApp,
) error {
    logger := log.FromContext(ctx)

    desired := r.desiredDeployment(myapp)

    // Set the owner reference so the Deployment is garbage-collected
    // when the MyApp resource is deleted
    if err := controllerutil.SetControllerReference(myapp, desired, r.Scheme); err != nil {
        return fmt.Errorf("setting owner reference: %w", err)
    }

    // Check if the Deployment already exists
    var existing appsv1.Deployment
    err := r.Get(ctx, types.NamespacedName{
        Name:      desired.Name,
        Namespace: desired.Namespace,
    }, &existing)

    if apierrors.IsNotFound(err) {
        logger.Info("creating Deployment", "name", desired.Name)
        return r.Create(ctx, desired)
    }
    if err != nil {
        return fmt.Errorf("getting deployment: %w", err)
    }

    // Update if the spec has changed
    if !equality.Semantic.DeepEqual(existing.Spec, desired.Spec) {
        existing.Spec = desired.Spec
        logger.Info("updating Deployment", "name", desired.Name)
        return r.Update(ctx, &existing)
    }

    return nil
}

func (r *MyAppReconciler) desiredDeployment(myapp *appsv1alpha1.MyApp) *appsv1.Deployment {
    labels := map[string]string{
        "app.kubernetes.io/name":       "myapp",
        "app.kubernetes.io/instance":   myapp.Name,
        "app.kubernetes.io/managed-by": "myapp-operator",
    }

    replicas := myapp.Spec.Replicas

    container := corev1.Container{
        Name:  "app",
        Image: myapp.Spec.Image,
        Ports: []corev1.ContainerPort{
            {
                Name:          "http",
                ContainerPort: myapp.Spec.Port,
                Protocol:      corev1.ProtocolTCP,
            },
        },
        LivenessProbe: &corev1.Probe{
            ProbeHandler: corev1.ProbeHandler{
                HTTPGet: &corev1.HTTPGetAction{
                    Path: "/healthz",
                    Port: intstr.FromString("http"),
                },
            },
            PeriodSeconds:    15,
            FailureThreshold: 3,
        },
        ReadinessProbe: &corev1.Probe{
            ProbeHandler: corev1.ProbeHandler{
                HTTPGet: &corev1.HTTPGetAction{
                    Path: "/readyz",
                    Port: intstr.FromString("http"),
                },
            },
            PeriodSeconds:    10,
            FailureThreshold: 2,
        },
    }

    // Set resource requests/limits if specified
    if myapp.Spec.Resources != nil {
        container.Resources = corev1.ResourceRequirements{
            Requests: corev1.ResourceList{},
            Limits:   corev1.ResourceList{},
        }
        if myapp.Spec.Resources.CPURequest != "" {
            container.Resources.Requests[corev1.ResourceCPU] =
                resource.MustParse(myapp.Spec.Resources.CPURequest)
        }
        if myapp.Spec.Resources.MemoryRequest != "" {
            container.Resources.Requests[corev1.ResourceMemory] =
                resource.MustParse(myapp.Spec.Resources.MemoryRequest)
        }
        if myapp.Spec.Resources.CPULimit != "" {
            container.Resources.Limits[corev1.ResourceCPU] =
                resource.MustParse(myapp.Spec.Resources.CPULimit)
        }
        if myapp.Spec.Resources.MemoryLimit != "" {
            container.Resources.Limits[corev1.ResourceMemory] =
                resource.MustParse(myapp.Spec.Resources.MemoryLimit)
        }
    }

    // Add database env var from secret if configured
    if myapp.Spec.Database != nil {
        container.Env = append(container.Env, corev1.EnvVar{
            Name: "DATABASE_URL",
            ValueFrom: &corev1.EnvVarSource{
                SecretKeyRef: &corev1.SecretKeySelector{
                    LocalObjectReference: corev1.LocalObjectReference{
                        Name: myapp.Spec.Database.SecretName,
                    },
                    Key: myapp.Spec.Database.Key,
                },
            },
        })
    }

    return &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      myapp.Name,
            Namespace: myapp.Namespace,
            Labels:    labels,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{container},
                },
            },
        },
    }
}

func (r *MyAppReconciler) reconcileService(
    ctx context.Context,
    myapp *appsv1alpha1.MyApp,
) error {
    logger := log.FromContext(ctx)

    labels := map[string]string{
        "app.kubernetes.io/name":     "myapp",
        "app.kubernetes.io/instance": myapp.Name,
    }

    desired := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      myapp.Name,
            Namespace: myapp.Namespace,
            Labels:    labels,
        },
        Spec: corev1.ServiceSpec{
            Selector: labels,
            Ports: []corev1.ServicePort{
                {
                    Name:       "http",
                    Port:       80,
                    TargetPort: intstr.FromString("http"),
                    Protocol:   corev1.ProtocolTCP,
                },
            },
        },
    }

    if err := controllerutil.SetControllerReference(myapp, desired, r.Scheme); err != nil {
        return err
    }

    var existing corev1.Service
    err := r.Get(ctx, types.NamespacedName{
        Name:      desired.Name,
        Namespace: desired.Namespace,
    }, &existing)

    if apierrors.IsNotFound(err) {
        logger.Info("creating Service", "name", desired.Name)
        return r.Create(ctx, desired)
    }
    if err != nil {
        return err
    }

    // Service already exists, update if needed
    existing.Spec.Selector = desired.Spec.Selector
    existing.Spec.Ports = desired.Spec.Ports
    return r.Update(ctx, &existing)
}

func (r *MyAppReconciler) updateStatus(
    ctx context.Context,
    myapp *appsv1alpha1.MyApp,
) error {
    var deployment appsv1.Deployment
    if err := r.Get(ctx, types.NamespacedName{
        Name:      myapp.Name,
        Namespace: myapp.Namespace,
    }, &deployment); err != nil {
        return err
    }

    myapp.Status.ReadyReplicas = deployment.Status.ReadyReplicas
    myapp.Status.ObservedGeneration = myapp.Generation

    return r.Status().Update(ctx, myapp)
}

// SetupWithManager sets up the controller with the Manager.
func (r *MyAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appsv1alpha1.MyApp{}).
        Owns(&appsv1.Deployment{}).   // Watch Deployments owned by MyApp
        Owns(&corev1.Service{}).      // Watch Services owned by MyApp
        Complete(r)
}
```

### Using the Custom Resource

```yaml
# example-myapp.yaml
apiVersion: apps.example.com/v1alpha1
kind: MyApp
metadata:
  name: my-web-service
  namespace: default
spec:
  replicas: 3
  image: myregistry/web-service:v1.0.0
  port: 8080
  resources:
    cpuRequest: "100m"
    memoryRequest: "128Mi"
    memoryLimit: "256Mi"
  database:
    secretName: web-service-db
    key: database_url
```

```bash
kubectl apply -f example-myapp.yaml
kubectl get myapps
kubectl describe myapp my-web-service
```

---

## client-go for Interacting with the Kubernetes API

`client-go` is the official Go client library for the Kubernetes API. It is used by
kubectl, operators, and any Go program that needs to interact with Kubernetes.

### Setting Up the Client

```go
// internal/k8s/client.go
package k8s

import (
    "fmt"
    "os"
    "path/filepath"

    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
)

// NewClient creates a Kubernetes clientset.
// It tries in-cluster config first, then falls back to kubeconfig.
func NewClient() (*kubernetes.Clientset, error) {
    // Try in-cluster config first (when running inside a pod)
    config, err := rest.InClusterConfig()
    if err != nil {
        // Fall back to kubeconfig (for local development)
        kubeconfig := os.Getenv("KUBECONFIG")
        if kubeconfig == "" {
            home, _ := os.UserHomeDir()
            kubeconfig = filepath.Join(home, ".kube", "config")
        }

        config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
        if err != nil {
            return nil, fmt.Errorf("building kubeconfig: %w", err)
        }
    }

    // Tune client settings
    config.QPS = 50
    config.Burst = 100

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        return nil, fmt.Errorf("creating clientset: %w", err)
    }

    return clientset, nil
}
```

### Listing and Watching Resources

```go
// internal/k8s/pods.go
package k8s

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/watch"
    "k8s.io/client-go/kubernetes"
)

// ListPods lists all pods in a namespace matching a label selector.
func ListPods(
    ctx context.Context,
    clientset *kubernetes.Clientset,
    namespace string,
    labelSelector string,
) ([]corev1.Pod, error) {
    podList, err := clientset.CoreV1().Pods(namespace).List(ctx, metav1.ListOptions{
        LabelSelector: labelSelector,
    })
    if err != nil {
        return nil, fmt.Errorf("listing pods: %w", err)
    }
    return podList.Items, nil
}

// WatchPods watches for pod events in a namespace.
func WatchPods(
    ctx context.Context,
    clientset *kubernetes.Clientset,
    namespace string,
    logger *slog.Logger,
) error {
    watcher, err := clientset.CoreV1().Pods(namespace).Watch(ctx, metav1.ListOptions{})
    if err != nil {
        return fmt.Errorf("creating watcher: %w", err)
    }
    defer watcher.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case event, ok := <-watcher.ResultChan():
            if !ok {
                return fmt.Errorf("watcher channel closed")
            }

            pod, ok := event.Object.(*corev1.Pod)
            if !ok {
                continue
            }

            switch event.Type {
            case watch.Added:
                logger.Info("pod added",
                    "name", pod.Name,
                    "phase", pod.Status.Phase,
                )
            case watch.Modified:
                logger.Info("pod modified",
                    "name", pod.Name,
                    "phase", pod.Status.Phase,
                )
            case watch.Deleted:
                logger.Info("pod deleted", "name", pod.Name)
            }
        }
    }
}
```

### Creating and Managing Resources

```go
// internal/k8s/deploy.go
package k8s

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/util/intstr"
    "k8s.io/client-go/kubernetes"
    "k8s.io/utils/ptr"
)

// DeployApp creates or updates a Deployment for an application.
func DeployApp(
    ctx context.Context,
    clientset *kubernetes.Clientset,
    namespace string,
    name string,
    image string,
    replicas int32,
) error {
    deploymentsClient := clientset.AppsV1().Deployments(namespace)

    labels := map[string]string{
        "app": name,
    }

    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      name,
            Namespace: namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: ptr.To(replicas),
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  name,
                            Image: image,
                            Ports: []corev1.ContainerPort{
                                {
                                    ContainerPort: 8080,
                                    Protocol:      corev1.ProtocolTCP,
                                },
                            },
                            Resources: corev1.ResourceRequirements{
                                Requests: corev1.ResourceList{
                                    corev1.ResourceCPU:    resource.MustParse("100m"),
                                    corev1.ResourceMemory: resource.MustParse("128Mi"),
                                },
                                Limits: corev1.ResourceList{
                                    corev1.ResourceMemory: resource.MustParse("256Mi"),
                                },
                            },
                            LivenessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    HTTPGet: &corev1.HTTPGetAction{
                                        Path: "/healthz",
                                        Port: intstr.FromInt32(8080),
                                    },
                                },
                                PeriodSeconds:    15,
                                FailureThreshold: 3,
                            },
                        },
                    },
                },
            },
        },
    }

    // Try to get existing deployment
    _, err := deploymentsClient.Get(ctx, name, metav1.GetOptions{})
    if err != nil {
        // Create new deployment
        _, err = deploymentsClient.Create(ctx, deployment, metav1.CreateOptions{})
        if err != nil {
            return fmt.Errorf("creating deployment: %w", err)
        }
        return nil
    }

    // Update existing deployment
    _, err = deploymentsClient.Update(ctx, deployment, metav1.UpdateOptions{})
    if err != nil {
        return fmt.Errorf("updating deployment: %w", err)
    }

    return nil
}

// ScaleDeployment adjusts the replica count of a Deployment.
func ScaleDeployment(
    ctx context.Context,
    clientset *kubernetes.Clientset,
    namespace string,
    name string,
    replicas int32,
) error {
    scale, err := clientset.AppsV1().Deployments(namespace).
        GetScale(ctx, name, metav1.GetOptions{})
    if err != nil {
        return fmt.Errorf("getting scale: %w", err)
    }

    scale.Spec.Replicas = replicas

    _, err = clientset.AppsV1().Deployments(namespace).
        UpdateScale(ctx, name, scale, metav1.UpdateOptions{})
    if err != nil {
        return fmt.Errorf("updating scale: %w", err)
    }

    return nil
}
```

### Using Informers for Efficient Watching

Informers provide a cached, event-driven way to watch resources -- much more efficient
than polling.

```go
// internal/k8s/informer.go
package k8s

import (
    "context"
    "log/slog"
    "time"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
)

// StartPodInformer watches for pod events using an informer.
func StartPodInformer(
    ctx context.Context,
    clientset *kubernetes.Clientset,
    namespace string,
    logger *slog.Logger,
) error {
    // Create a shared informer factory
    factory := informers.NewSharedInformerFactoryWithOptions(
        clientset,
        30*time.Second, // Resync period
        informers.WithNamespace(namespace),
    )

    // Get the pod informer
    podInformer := factory.Core().V1().Pods().Informer()

    // Register event handlers
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            logger.Info("pod added",
                "name", pod.Name,
                "namespace", pod.Namespace,
                "phase", pod.Status.Phase,
            )
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldPod := oldObj.(*corev1.Pod)
            newPod := newObj.(*corev1.Pod)
            if oldPod.Status.Phase != newPod.Status.Phase {
                logger.Info("pod phase changed",
                    "name", newPod.Name,
                    "old_phase", oldPod.Status.Phase,
                    "new_phase", newPod.Status.Phase,
                )
            }
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            logger.Info("pod deleted",
                "name", pod.Name,
                "namespace", pod.Namespace,
            )
        },
    })

    // Start the informer
    stopCh := make(chan struct{})
    go func() {
        <-ctx.Done()
        close(stopCh)
    }()

    factory.Start(stopCh)

    // Wait for the cache to sync
    if !cache.WaitForCacheSync(stopCh, podInformer.HasSynced) {
        return fmt.Errorf("timed out waiting for cache sync")
    }

    logger.Info("pod informer started and synced")

    <-ctx.Done()
    return nil
}
```

---

## Helm Charts for Go Services

Helm is the package manager for Kubernetes. It templates Kubernetes manifests and
manages releases.

### Chart Structure

```
helm/myapp/
  Chart.yaml
  values.yaml
  values-staging.yaml
  values-production.yaml
  templates/
    _helpers.tpl
    deployment.yaml
    service.yaml
    ingress.yaml
    hpa.yaml
    configmap.yaml
    secret.yaml
    serviceaccount.yaml
    pdb.yaml
    NOTES.txt
```

### Chart.yaml

```yaml
# helm/myapp/Chart.yaml
apiVersion: v2
name: myapp
description: A Go microservice deployed with Helm
type: application
version: 0.1.0       # Chart version
appVersion: "1.0.0"  # Application version

keywords:
  - go
  - api
  - microservice

maintainers:
  - name: Platform Team
    email: platform@example.com
```

### values.yaml

```yaml
# helm/myapp/values.yaml

# -- Number of replicas
replicaCount: 2

image:
  # -- Container image repository
  repository: myregistry/myapp-api
  # -- Image pull policy
  pullPolicy: IfNotPresent
  # -- Image tag (defaults to appVersion)
  tag: ""

# -- Image pull secrets
imagePullSecrets: []

# -- Override the release name
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # -- Create a service account
  create: true
  # -- Annotations for the service account
  annotations: {}
  # -- Service account name (auto-generated if not set)
  name: ""

# -- Pod annotations
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"

# -- Pod security context
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 65534
  fsGroup: 65534

# -- Container security context
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

service:
  # -- Service type
  type: ClusterIP
  # -- Service port
  port: 80

ingress:
  # -- Enable ingress
  enabled: false
  # -- Ingress class name
  className: nginx
  # -- Ingress annotations
  annotations: {}
  # -- Ingress hosts
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  # -- TLS configuration
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi

autoscaling:
  # -- Enable HPA
  enabled: false
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# -- Node selector
nodeSelector: {}

# -- Tolerations
tolerations: []

# -- Affinity rules
affinity: {}

# -- Topology spread constraints
topologySpreadConstraints: []

# -- Pod disruption budget
pdb:
  enabled: true
  minAvailable: 1

# Application-specific configuration
app:
  # -- Log level
  logLevel: info
  # -- Server port
  port: 8080
  # -- Health check port
  healthPort: 8081
  # -- Metrics port
  metricsPort: 9090

  # -- Database configuration
  database:
    # -- Name of the Secret containing DATABASE_URL
    existingSecret: ""
    secretKey: database_url

  # -- Environment variables from ConfigMaps/Secrets
  extraEnv: []
  extraEnvFrom: []

  # -- Go runtime settings
  goMaxProcs: ""
  goMemLimit: ""
```

### Template Helpers

```yaml
{{/* helm/myapp/templates/_helpers.tpl */}}

{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
app.kubernetes.io/version: {{ .Values.image.tag | default .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Deployment Template

```yaml
# helm/myapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      terminationGracePeriodSeconds: 30
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          ports:
            - name: http
              containerPort: {{ .Values.app.port }}
              protocol: TCP
            - name: health
              containerPort: {{ .Values.app.healthPort }}
              protocol: TCP
            - name: metrics
              containerPort: {{ .Values.app.metricsPort }}
              protocol: TCP
          env:
            - name: APP_PORT
              value: {{ .Values.app.port | quote }}
            - name: HEALTH_PORT
              value: {{ .Values.app.healthPort | quote }}
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: {{ include "myapp.fullname" . }}
                  key: log_level
            {{- if .Values.app.database.existingSecret }}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.app.database.existingSecret }}
                  key: {{ .Values.app.database.secretKey }}
            {{- end }}
            {{- if .Values.app.goMaxProcs }}
            - name: GOMAXPROCS
              value: {{ .Values.app.goMaxProcs | quote }}
            {{- end }}
            {{- if .Values.app.goMemLimit }}
            - name: GOMEMLIMIT
              value: {{ .Values.app.goMemLimit | quote }}
            {{- end }}
            {{- with .Values.app.extraEnv }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          {{- with .Values.app.extraEnvFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          startupProbe:
            httpGet:
              path: /startupz
              port: health
            initialDelaySeconds: 2
            periodSeconds: 5
            failureThreshold: 12
          livenessProbe:
            httpGet:
              path: /healthz
              port: health
            periodSeconds: 15
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: health
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Environment-Specific Values

```yaml
# helm/myapp/values-production.yaml
replicaCount: 5

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 60

pdb:
  enabled: true
  minAvailable: 3

app:
  logLevel: warn
  goMaxProcs: "4"
  goMemLimit: "460MiB"
  database:
    existingSecret: myapp-db-prod
    secretKey: database_url

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-example-tls
      hosts:
        - api.example.com

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: myapp
```

### Helm Commands

```bash
# Lint the chart
helm lint helm/myapp/

# Template locally (see what would be deployed)
helm template myapp helm/myapp/ -f helm/myapp/values-production.yaml

# Install
helm install myapp helm/myapp/ \
    -n production \
    -f helm/myapp/values-production.yaml

# Upgrade
helm upgrade myapp helm/myapp/ \
    -n production \
    -f helm/myapp/values-production.yaml \
    --set image.tag=v1.2.3

# Rollback
helm rollback myapp 1 -n production

# Uninstall
helm uninstall myapp -n production
```

---

## Debugging Go in Containers

### Remote Debugging with Delve

Delve is Go's debugger. It supports remote debugging over a TCP connection, which is
essential for debugging containerized applications.

#### Debug Dockerfile

```dockerfile
# Dockerfile.debug
FROM golang:1.22 AS builder

# Install Delve
RUN go install github.com/go-delve/delve/cmd/dlv@latest

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .

# Build WITHOUT optimizations so Delve can inspect variables
RUN CGO_ENABLED=0 go build \
    -gcflags="all=-N -l" \
    -o /server \
    ./cmd/server

# Use a base image with shell for debugging
FROM alpine:3.19

RUN apk add --no-cache ca-certificates

COPY --from=builder /go/bin/dlv /dlv
COPY --from=builder /server /server

EXPOSE 8080 2345

# Start Delve headless server
# --listen: address to listen on
# --headless: run in headless mode (no terminal UI)
# --api-version=2: use DAP protocol
# --accept-multiclient: allow multiple debugger connections
# --continue: start the process immediately (don't wait for debugger)
CMD ["/dlv", "exec", "/server", \
     "--listen=:2345", \
     "--headless=true", \
     "--api-version=2", \
     "--accept-multiclient", \
     "--continue"]
```

#### Docker Compose for Debugging

```yaml
# docker-compose.debug.yml
version: "3.9"

services:
  api-debug:
    build:
      context: .
      dockerfile: Dockerfile.debug
    ports:
      - "8080:8080"
      - "2345:2345"     # Delve debug port
    environment:
      - DATABASE_URL=postgres://app:secret@postgres:5432/myapp?sslmode=disable
    security_opt:
      - "seccomp:unconfined"   # Required for Delve's ptrace
    cap_add:
      - SYS_PTRACE             # Required for Delve
    depends_on:
      postgres:
        condition: service_healthy
```

#### VS Code Launch Configuration

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to Container",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "remotePath": "/build",
            "port": 2345,
            "host": "127.0.0.1",
            "showLog": true,
            "trace": "log",
            "logOutput": "rpc"
        }
    ]
}
```

#### GoLand / IntelliJ Configuration

1. Go to **Run > Edit Configurations**
2. Add a **Go Remote** configuration
3. Set Host to `localhost` and Port to `2345`
4. Click **Debug**

### Debugging in Kubernetes

For debugging pods in a Kubernetes cluster:

```bash
# Port-forward the debug port from a pod
kubectl port-forward pod/myapp-api-xxxxx 2345:2345

# Then connect your IDE to localhost:2345
```

#### Ephemeral Debug Containers

Kubernetes 1.25+ supports ephemeral debug containers:

```bash
# Attach a debug container to a running pod
kubectl debug -it myapp-api-xxxxx \
    --image=golang:1.22 \
    --target=api \
    -- bash

# Inside the debug container, you can:
# - Inspect the filesystem
# - Run network diagnostics
# - Attach strace, tcpdump, etc.
```

### Capturing Profiles from Running Containers

```go
// Enable pprof in your application
import _ "net/http/pprof"

func main() {
    // Expose pprof on a separate port
    go func() {
        http.ListenAndServe(":6060", nil) // default mux has pprof handlers
    }()
    // ...
}
```

```bash
# Port-forward and capture a profile
kubectl port-forward pod/myapp-api-xxxxx 6060:6060

# CPU profile (30 seconds)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine dump
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Trace (5 seconds)
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5
go tool trace trace.out
```

### Logging Best Practices in Containers

```go
// Use structured JSON logging for container environments
package main

import (
    "log/slog"
    "os"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     parseLogLevel(os.Getenv("LOG_LEVEL")),
        AddSource: true,
    }))

    // Set as the default logger
    slog.SetDefault(logger)

    logger.Info("server starting",
        "version", version,
        "port", 8080,
        "pod", os.Getenv("POD_NAME"),
        "namespace", os.Getenv("POD_NAMESPACE"),
    )
}

func parseLogLevel(level string) slog.Level {
    switch level {
    case "debug":
        return slog.LevelDebug
    case "warn", "warning":
        return slog.LevelWarn
    case "error":
        return slog.LevelError
    default:
        return slog.LevelInfo
    }
}
```

> **Tip:** Always log to stdout/stderr in containers. The container runtime (Docker,
> containerd) captures stdout/stderr and forwards it to the logging subsystem
> (Fluentd, Loki, CloudWatch, etc.). Never write to log files inside the container.

---

## CI/CD Pipeline Example

### GitHub Actions: Build, Test, Deploy to Kubernetes

```yaml
# .github/workflows/deploy.yml
name: Build, Test, and Deploy

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: read
  packages: write
  id-token: write   # For OIDC authentication to cloud providers

jobs:
  # ──────────────────────────────────────────
  # Job 1: Lint and Test
  # ──────────────────────────────────────────
  test:
    name: Lint and Test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true

      - name: Install dependencies
        run: go mod download

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m

      - name: Run tests
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb?sslmode=disable
        run: |
          go test -v -race -coverprofile=coverage.out ./...
          go tool cover -func=coverage.out

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: coverage.out

  # ──────────────────────────────────────────
  # Job 2: Security Scanning
  # ──────────────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner (filesystem)
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Run govulncheck
        uses: golang/govulncheck-action@v1
        with:
          go-version-input: '1.22'

  # ──────────────────────────────────────────
  # Job 3: Build and Push Docker Image
  # ──────────────────────────────────────────
  build:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: [test, security]
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=,format=short

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ steps.meta.outputs.version }}

      - name: Scan built image
        if: github.event_name != 'pull_request'
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  # ──────────────────────────────────────────
  # Job 4: Deploy to Staging
  # ──────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging-api.example.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: 'v3.14.0'

      - name: Configure kubectl
        uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp helm/myapp/ \
            --namespace staging \
            --create-namespace \
            -f helm/myapp/values-staging.yaml \
            --set image.tag=${{ needs.build.outputs.image-tag }} \
            --wait \
            --timeout 5m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n staging --timeout=120s
          kubectl get pods -n staging -l app.kubernetes.io/name=myapp

      - name: Run smoke tests
        run: |
          # Wait for service to be reachable
          for i in {1..30}; do
            if curl -sf https://staging-api.example.com/healthz; then
              echo "Service is healthy"
              break
            fi
            echo "Waiting for service... ($i/30)"
            sleep 5
          done

          # Run integration tests against staging
          API_URL=https://staging-api.example.com go test ./tests/integration/ -v

  # ──────────────────────────────────────────
  # Job 5: Deploy to Production
  # ──────────────────────────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://api.example.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: 'v3.14.0'

      - name: Configure kubectl
        uses: azure/k8s-set-context@v4
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.PRODUCTION_KUBECONFIG }}

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp helm/myapp/ \
            --namespace production \
            --create-namespace \
            -f helm/myapp/values-production.yaml \
            --set image.tag=${{ needs.build.outputs.image-tag }} \
            --wait \
            --timeout 10m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n production --timeout=300s

      - name: Run smoke tests
        run: |
          for i in {1..30}; do
            if curl -sf https://api.example.com/healthz; then
              echo "Production service is healthy"
              exit 0
            fi
            echo "Waiting... ($i/30)"
            sleep 10
          done
          echo "Production health check failed"
          exit 1
```

### Makefile for Local Development

A Makefile ties together all the build, test, and deploy commands:

```makefile
# Makefile
.PHONY: help build test lint docker-build docker-push deploy-staging deploy-prod

# Variables
APP_NAME     := myapp-api
VERSION      ?= $(shell git describe --tags --always --dirty)
REGISTRY     ?= ghcr.io/myorg
IMAGE        := $(REGISTRY)/$(APP_NAME)
GO_FILES     := $(shell find . -name '*.go' -not -path './vendor/*')

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

## ── Build ──────────────────────────────────

build: ## Build the Go binary
	CGO_ENABLED=0 go build -ldflags="-w -s -X main.version=$(VERSION)" \
		-o bin/$(APP_NAME) ./cmd/server

build-debug: ## Build with debug symbols
	go build -gcflags="all=-N -l" -o bin/$(APP_NAME) ./cmd/server

## ── Test ───────────────────────────────────

test: ## Run unit tests
	go test -race -count=1 ./...

test-coverage: ## Run tests with coverage report
	go test -race -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html
	@echo "Coverage report: coverage.html"

test-integration: ## Run integration tests (requires Docker)
	docker compose -f docker-compose.test.yml up -d
	DATABASE_URL=postgres://test:test@localhost:5432/testdb?sslmode=disable \
		go test -tags=integration -v ./tests/integration/
	docker compose -f docker-compose.test.yml down -v

## ── Lint ───────────────────────────────────

lint: ## Run linters
	golangci-lint run --timeout 5m

lint-fix: ## Run linters and fix issues
	golangci-lint run --fix --timeout 5m

vuln: ## Run vulnerability check
	govulncheck ./...

## ── Docker ─────────────────────────────────

docker-build: ## Build Docker image
	DOCKER_BUILDKIT=1 docker build \
		--build-arg VERSION=$(VERSION) \
		-t $(IMAGE):$(VERSION) \
		-t $(IMAGE):latest \
		.

docker-push: ## Push Docker image
	docker push $(IMAGE):$(VERSION)
	docker push $(IMAGE):latest

docker-scan: ## Scan Docker image for vulnerabilities
	trivy image $(IMAGE):$(VERSION)

## ── Local Development ──────────────────────

dev: ## Start local development environment
	docker compose -f docker-compose.dev.yml up --build

dev-down: ## Stop local development environment
	docker compose -f docker-compose.dev.yml down -v

## ── Kubernetes ─────────────────────────────

deploy-staging: docker-build docker-push ## Deploy to staging
	helm upgrade --install $(APP_NAME) helm/myapp/ \
		-n staging --create-namespace \
		-f helm/myapp/values-staging.yaml \
		--set image.tag=$(VERSION) \
		--wait

deploy-prod: docker-build docker-push ## Deploy to production
	helm upgrade --install $(APP_NAME) helm/myapp/ \
		-n production \
		-f helm/myapp/values-production.yaml \
		--set image.tag=$(VERSION) \
		--wait --timeout 10m

rollback: ## Rollback to previous release
	helm rollback $(APP_NAME) -n $(NAMESPACE)

## ── Utilities ──────────────────────────────

generate: ## Run code generators
	go generate ./...

migrate-up: ## Run database migrations
	migrate -path ./migrations -database "$(DATABASE_URL)" up

migrate-down: ## Rollback last migration
	migrate -path ./migrations -database "$(DATABASE_URL)" down 1

clean: ## Remove build artifacts
	rm -rf bin/ tmp/ coverage.out coverage.html
```

---

## Production Checklist

Use this checklist before deploying a Go application to production on Kubernetes.

### Docker Image

| Item | Status | Notes |
|------|--------|-------|
| Multi-stage build | Required | Builder stage + scratch/distroless |
| Static binary (CGO_ENABLED=0) | Required | Unless cgo is needed |
| Non-root user | Required | Use `USER` directive or distroless `:nonroot` |
| Minimal base image | Required | scratch, distroless, or alpine |
| .dockerignore configured | Required | Exclude .git, tests, docs, secrets |
| Image scanned for vulnerabilities | Required | Trivy, Snyk, or Docker Scout |
| Version/commit baked into binary | Recommended | Via `-ldflags -X` |
| Multi-architecture support | Recommended | amd64 + arm64 |
| Build cache optimized | Recommended | Separate `go mod download` layer |
| Pinned base image tags | Recommended | `golang:1.22.1` not `golang:latest` |

### Application Code

| Item | Status | Notes |
|------|--------|-------|
| Graceful shutdown (SIGTERM) | Required | `http.Server.Shutdown()` |
| Separate health check port | Required | Keep health probes off the main port |
| Liveness endpoint (/healthz) | Required | Do NOT check dependencies |
| Readiness endpoint (/readyz) | Required | DO check dependencies |
| Startup endpoint (/startupz) | Recommended | For slow-starting apps |
| Structured JSON logging to stdout | Required | `slog` with JSON handler |
| Prometheus metrics | Recommended | `/metrics` endpoint |
| pprof endpoint (disabled in prod) | Recommended | Behind a flag or separate port |
| Connection draining | Required | Complete in-flight requests before exit |
| Context propagation | Required | Pass `context.Context` everywhere |
| Timeouts on all external calls | Required | HTTP, DB, Redis, gRPC |
| Retry with backoff | Recommended | For transient failures |

### Kubernetes Manifests

| Item | Status | Notes |
|------|--------|-------|
| Resource requests set | Required | CPU and memory |
| Memory limit set | Required | Prevents OOM killing other pods |
| GOMAXPROCS set | Required | Match to CPU request/limit |
| GOMEMLIMIT set | Recommended | ~90% of memory limit |
| Startup probe configured | Recommended | Prevents premature liveness kills |
| Liveness probe configured | Required | Detects deadlocked processes |
| Readiness probe configured | Required | Removes unhealthy pods from LB |
| preStop hook with sleep | Required | Allows endpoint propagation |
| terminationGracePeriodSeconds | Required | >= preStop + shutdown time |
| Pod Disruption Budget | Required | Prevents all pods going down at once |
| Topology spread constraints | Recommended | Spread across zones/nodes |
| Security context (non-root, read-only FS) | Required | Principle of least privilege |
| Network policies | Recommended | Restrict pod-to-pod traffic |
| Service account with minimal RBAC | Required | Do not use default SA |
| Secrets from external store | Recommended | Not plain K8s Secrets in Git |

### Observability

| Item | Status | Notes |
|------|--------|-------|
| Structured logging (JSON) | Required | With correlation IDs |
| Metrics (Prometheus) | Required | RED metrics at minimum |
| Distributed tracing (OpenTelemetry) | Recommended | Trace across services |
| Alerting rules | Required | On error rate, latency, saturation |
| Dashboard (Grafana) | Recommended | For the on-call team |
| Log aggregation (Loki/ELK/CloudWatch) | Required | Centralized log search |

### CI/CD

| Item | Status | Notes |
|------|--------|-------|
| Automated tests in pipeline | Required | Unit + integration |
| Linting in pipeline | Required | golangci-lint |
| Vulnerability scanning | Required | govulncheck + image scanning |
| Image signed (cosign/notation) | Recommended | Supply chain security |
| Automated staging deployment | Required | On merge to main |
| Manual production approval | Recommended | GitHub Environments |
| Rollback strategy documented | Required | `helm rollback` or `kubectl rollout undo` |
| Canary or blue-green deployment | Recommended | For zero-downtime |

### Go Runtime Tuning Quick Reference

```
┌─────────────────────────────────────────────────────────────┐
│  Container: memory limit = 256Mi, cpu limit = 500m          │
│                                                             │
│  GOMAXPROCS = 2         (match available CPUs)              │
│  GOMEMLIMIT = 230MiB    (90% of memory limit)              │
│                                                             │
│  Or use:                                                    │
│    go.uber.org/automaxprocs                                 │
│    github.com/KimMachineGun/automemlimit                    │
│                                                             │
│  Database pool:                                             │
│    MaxOpenConns = 25     (tune per workload)                │
│    MaxIdleConns = 5                                         │
│    ConnMaxLifetime = 5m  (< server wait_timeout)            │
│                                                             │
│  HTTP server:                                               │
│    ReadTimeout = 10s                                        │
│    WriteTimeout = 30s                                       │
│    IdleTimeout = 120s    (match LB idle timeout)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

This chapter covered the complete journey of deploying Go applications with Docker and
Kubernetes:

1. **Docker Images**: Multi-stage builds produce minimal, secure images. Use `scratch`
   or distroless as the final stage, build static binaries with `CGO_ENABLED=0`, and
   cache Go module downloads in a separate layer.

2. **Docker Compose**: Provides a local development environment that mirrors production
   topology. Use volume mounts and Air for hot reload.

3. **Health Checks**: Three endpoints serve different purposes -- liveness (process
   alive), readiness (can handle traffic), and startup (finished initializing). Never
   check external dependencies in liveness probes.

4. **Kubernetes Resources**: Deployments, Services, ConfigMaps, and Secrets form the
   foundation. Set resource requests/limits, configure probes, and use topology spread
   constraints.

5. **Graceful Shutdown**: Handle SIGTERM, use preStop hooks to allow endpoint
   propagation, drain connections, and close resources in order.

6. **Configuration**: Layer defaults, config files (ConfigMaps), environment variables
   (Secrets), and flags. Watch for config file changes at runtime.

7. **Operators**: Use kubebuilder and controller-runtime to extend Kubernetes with
   custom resources and reconciliation logic written in Go.

8. **client-go**: Interact with the Kubernetes API programmatically using clientsets,
   watchers, and informers.

9. **Helm**: Template and manage Kubernetes manifests with environment-specific values
   files. Use chart versioning for reproducible deployments.

10. **Debugging**: Use Delve for remote debugging, pprof for profiling, and ephemeral
    containers for live investigation.

11. **CI/CD**: Automate linting, testing, building, scanning, and deploying with GitHub
    Actions. Use separate environments for staging and production.

12. **Production Checklist**: A systematic review of security, reliability, and
    observability requirements before going live.

The combination of Go's static binaries and Kubernetes' orchestration capabilities
creates a powerful, efficient deployment platform. Master these patterns and you will
be well-equipped to run Go services at any scale.

---

## Further Reading

- [Docker Multi-Stage Build Documentation](https://docs.docker.com/build/building/multi-stage/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Kubebuilder Book](https://book.kubebuilder.io/)
- [client-go Examples](https://github.com/kubernetes/client-go/tree/master/examples)
- [Helm Documentation](https://helm.sh/docs/)
- [Delve Debugger](https://github.com/go-delve/delve)
- [Google Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [Go Runtime Memory Guide](https://go.dev/doc/gc-guide)
