# Chapter 56: CI/CD Pipelines for Go Projects

## Table of Contents

1. [CI/CD Fundamentals for Go](#1-cicd-fundamentals-for-go)
2. [GitHub Actions for Go Projects](#2-github-actions-for-go-projects)
3. [GitLab CI for Go](#3-gitlab-ci-for-go)
4. [Makefile Patterns for Go Projects](#4-makefile-patterns-for-go-projects)
5. [Go Build Flags and Version Injection](#5-go-build-flags-and-version-injection)
6. [Code Quality Gates](#6-code-quality-gates)
7. [Release Automation](#7-release-automation)
8. [Feature Flags and Gradual Rollouts](#8-feature-flags-and-gradual-rollouts)
9. [Database Migrations in CI/CD](#9-database-migrations-in-cicd)
10. [Integration Testing in Pipelines](#10-integration-testing-in-pipelines)
11. [Secrets Management in CI](#11-secrets-management-in-ci)
12. [Monitoring Deployments](#12-monitoring-deployments)
13. [Complete Pipeline: Commit to Production](#13-complete-pipeline-commit-to-production)

---

## 1. CI/CD Fundamentals for Go

### What CI/CD Means for Go

Continuous Integration (CI) and Continuous Delivery/Deployment (CD) form the backbone of
modern software development. For Go projects, CI/CD pipelines automate the process of
building, testing, vetting, and shipping your code.

Go is particularly well-suited for CI/CD because:

- **Fast compilation**: Go compiles to a single static binary in seconds.
- **Built-in tooling**: `go test`, `go vet`, `go build`, and `go fmt` are first-class citizens.
- **Cross-compilation**: A single machine can produce binaries for Linux, macOS, Windows, ARM, and more.
- **Minimal runtime dependencies**: Static binaries mean tiny Docker images and simple deployments.

### Anatomy of a Go CI/CD Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Commit /   │───>│    Build &   │───>│   Quality    │───>│   Release &  │
│  Pull Request│    │     Test     │    │    Gates     │    │    Deploy    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       │                   │                   │                   │
       │              - go build          - go vet            - goreleaser
       │              - go test           - golangci-lint      - Docker push
       │              - race detector     - coverage check     - Helm deploy
       │              - go mod tidy       - govulncheck        - Canary/B-G
       │                                 - gosec
```

### Pipeline Stages Overview

| Stage | Purpose | Tools |
|-------|---------|-------|
| **Checkout** | Clone repository | `actions/checkout` |
| **Setup** | Install Go, restore caches | `actions/setup-go` |
| **Dependencies** | Download and verify modules | `go mod download`, `go mod verify` |
| **Build** | Compile the project | `go build` |
| **Test** | Run unit and integration tests | `go test`, `-race`, `-cover` |
| **Lint** | Static analysis and style | `golangci-lint`, `go vet` |
| **Security** | Vulnerability scanning | `govulncheck`, `gosec`, `trivy` |
| **Package** | Build Docker image / binary | `docker build`, `goreleaser` |
| **Deploy** | Ship to staging/production | `kubectl`, `helm`, cloud CLI |

### Key Principles

1. **Fail fast**: Put the cheapest checks first (formatting, vetting) so developers get quick feedback.
2. **Reproducibility**: Pin Go versions, tool versions, and module checksums.
3. **Parallelism**: Run independent jobs concurrently (lint and test don't depend on each other).
4. **Caching**: Cache `~/go/pkg/mod` and `~/.cache/go-build` to speed up builds.
5. **Idempotency**: Every pipeline run should produce the same result given the same input.

---

## 2. GitHub Actions for Go Projects

### 2.1 Basic Go Setup

The foundation of any GitHub Actions Go workflow starts with checking out the code and
setting up the Go toolchain.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          # Built-in caching (caches ~/go/pkg/mod and ~/.cache/go-build)
          cache: true

      - name: Download dependencies
        run: go mod download

      - name: Verify dependencies
        run: go mod verify

      - name: Build
        run: go build -v ./...
```

> **Tip**: `actions/setup-go@v5` has built-in module caching. You typically do not need a
> separate `actions/cache` step unless you have custom caching requirements.

### 2.2 Module Caching (Advanced)

When you need fine-grained control over caching, use the cache action directly.

```yaml
      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: |
            ~/go/pkg/mod
            ~/.cache/go-build
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-

      - name: Cache tool binaries
        uses: actions/cache@v4
        with:
          path: ~/go/bin
          key: ${{ runner.os }}-go-tools-${{ hashFiles('tools/go.sum') }}
```

### 2.3 Matrix Builds

Test across multiple Go versions and operating systems simultaneously.

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        go-version: ['1.22', '1.23']
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true

      - name: Run tests
        run: go test -v ./...
```

> **Warning**: Matrix builds multiply runner minutes quickly. For open-source projects on
> GitHub's free tier, be selective. A common pattern is to run the full matrix only on PRs
> to `main` and nightly, while feature branch pushes test only the latest Go on Linux.

### 2.4 Running Tests with Race Detector and Coverage

```yaml
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Run tests with race detector
        run: go test -race -v ./...

      - name: Run tests with coverage
        run: |
          go test -race -coverprofile=coverage.out -covermode=atomic ./...
          go tool cover -func=coverage.out

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

      - name: Check coverage threshold
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          echo "Total coverage: ${COVERAGE}%"
          THRESHOLD=80.0
          if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below threshold ${THRESHOLD}%"
            exit 1
          fi
```

**Understanding the flags:**

| Flag | Purpose |
|------|---------|
| `-race` | Enables the Go race detector; catches data races at test time |
| `-coverprofile=coverage.out` | Writes coverage data to a file |
| `-covermode=atomic` | Required when using `-race`; counts how many times each statement runs |
| `-count=1` | Disables test caching (useful in CI) |
| `-timeout=10m` | Sets a timeout for the entire test run |
| `-short` | Skips long-running tests (when `testing.Short()` is checked) |

### 2.5 Linting with golangci-lint

`golangci-lint` is the de facto standard meta-linter for Go. It bundles dozens of linters
into one fast tool.

#### Basic Setup

```yaml
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: v1.62
          args: --timeout=5m
```

#### Configuration File

Create `.golangci.yml` at the root of your repository:

```yaml
# .golangci.yml
run:
  timeout: 5m
  modules-download-mode: readonly

linters:
  enable:
    - errcheck        # Check that errors are handled
    - govet           # Reports suspicious constructs
    - staticcheck     # Advanced static analysis
    - unused          # Find unused code
    - gosimple        # Simplify code
    - ineffassign     # Detect ineffectual assignments
    - typecheck       # Type-checks Go code
    - bodyclose       # Ensure HTTP response bodies are closed
    - dupl            # Find duplicate code
    - errorlint       # Find errors not conforming to Go 1.13+ error patterns
    - exhaustive      # Check exhaustiveness of enum switch statements
    - exportloopref   # Check for pointers to loop variables
    - gocognit        # Cognitive complexity analysis
    - goconst         # Find repeated strings that could be constants
    - gocritic        # Opinionated Go linter
    - gocyclo         # Cyclomatic complexity
    - godot           # Check that comments end with a period
    - gofmt           # Check code is gofmt-ed
    - goimports       # Check imports ordering
    - misspell        # Find common misspellings
    - nilerr          # Find code that returns nil even if error is not nil
    - noctx           # Find HTTP requests without context
    - prealloc        # Find slice declarations that could be preallocated
    - predeclared     # Find shadowed predeclared identifiers
    - revive          # Fast, configurable linter (replacement for golint)
    - unconvert       # Find unnecessary type conversions
    - unparam         # Find unused function parameters
    - whitespace      # Check for unnecessary whitespace

linters-settings:
  gocyclo:
    min-complexity: 15
  gocognit:
    min-complexity: 20
  dupl:
    threshold: 100
  goconst:
    min-len: 3
    min-occurrences: 3
  gocritic:
    enabled-tags:
      - diagnostic
      - experimental
      - opinionated
      - performance
      - style
  revive:
    rules:
      - name: blank-imports
      - name: context-as-argument
      - name: context-keys-type
      - name: dot-imports
      - name: error-naming
      - name: error-return
      - name: error-strings
      - name: exported
      - name: if-return
      - name: increment-decrement
      - name: package-comments
      - name: range
      - name: receiver-naming
      - name: time-naming
      - name: unexported-return
      - name: var-declaration
      - name: var-naming
  errorlint:
    errorf: true
    asserts: true
    comparison: true

issues:
  exclude-rules:
    # Ignore long lines in generated files
    - path: _test\.go
      linters:
        - dupl
        - gocyclo
        - gocognit
    # Ignore certain patterns in test files
    - path: _test\.go
      text: "unnecessaryBlock"
    # Allow fmt.Print in main packages
    - path: cmd/
      linters:
        - forbidigo

  max-issues-per-linter: 0
  max-same-issues: 0

severity:
  default-severity: warning
  rules:
    - linters:
        - govet
        - errcheck
      severity: error
```

#### Custom Linter Rules with revive

```yaml
# .golangci.yml (revive section)
linters-settings:
  revive:
    confidence: 0.8
    severity: warning
    rules:
      - name: blank-imports
      - name: context-as-argument
        arguments:
          - allowTypesBefore: "*testing.T"
      - name: context-keys-type
      - name: early-return
      - name: error-naming
      - name: error-return
      - name: exported
        arguments:
          - "checkPrivateReceivers"
          - "sayRepetitiveInsteadOfStutters"
      - name: function-result-limit
        arguments: [3]
      - name: import-shadowing
      - name: line-length-limit
        arguments: [120]
      - name: nested-structs
      - name: optimize-operands-order
      - name: string-format
        arguments:
          - - 'fmt.Errorf[0]'
            - '/^[^A-Z].*[^.]$/'
            - 'error strings must not be capitalized or end with punctuation'
          - - 'log.Printf[0]'
            - '/^[^A-Z]/'
            - 'log messages must not be capitalized'
      - name: unhandled-error
        arguments:
          - "fmt.Fprintf"
          - "fmt.Fprintln"
          - "fmt.Fprint"
```

### 2.6 Security Scanning

#### govulncheck - Official Go Vulnerability Scanner

```yaml
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Run govulncheck
        run: govulncheck ./...
```

#### gosec - Go Security Checker

```yaml
      - name: Run gosec
        uses: securego/gosec@master
        with:
          args: '-no-fail -fmt sarif -out gosec-results.sarif ./...'

      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: gosec-results.sarif
```

**gosec configuration file** (`.gosec.yml`):

```yaml
# .gosec.yml
global:
  audit: enabled
rules:
  G101:  # Look for hard-coded credentials
    pattern: "(?i)(password|secret|token|key|api_key)\\s*=\\s*\"[^\"]+\""
  G104:  # Audit errors not checked
    enabled: true
  G201:  # SQL string formatting
    enabled: true
  G202:  # SQL string concatenation
    enabled: true
  G301:  # Poor file permissions
    enabled: true
  G304:  # File path provided as taint input
    enabled: true
  G401:  # Detect use of DES, RC4, MD5, or SHA1
    enabled: true
  G501:  # Blocklisted import crypto/md5
    enabled: true
```

#### Trivy - Container and Filesystem Vulnerability Scanner

```yaml
      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Run Trivy on Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true
```

### 2.7 Building and Pushing Docker Images

#### Multi-stage Dockerfile for Go

```dockerfile
# Build stage
FROM golang:1.23-alpine AS builder

RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

# Copy go.mod and go.sum first for better caching
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copy source code
COPY . .

# Build with all optimizations
ARG VERSION=dev
ARG COMMIT=unknown
ARG BUILD_TIME=unknown

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" \
    -trimpath \
    -o /app/server \
    ./cmd/server

# Runtime stage
FROM scratch

COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

#### GitHub Actions Docker Build and Push

```yaml
  docker:
    runs-on: ubuntu-latest
    needs: [test, lint, security]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.ref_name }}
            COMMIT=${{ github.sha }}
            BUILD_TIME=${{ github.event.head_commit.timestamp }}
          platforms: linux/amd64,linux/arm64
```

### 2.8 Deployment Workflows

#### Deploy to Kubernetes

```yaml
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [docker]
    environment:
      name: staging
      url: https://staging.myapp.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.30.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp ./deploy/helm/myapp \
            --namespace myapp-staging \
            --set image.tag=${{ github.sha }} \
            --set replicas=2 \
            --wait \
            --timeout 5m

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n myapp-staging --timeout=300s
          kubectl get pods -n myapp-staging -l app=myapp

      - name: Run smoke tests
        run: |
          ENDPOINT=$(kubectl get svc myapp -n myapp-staging -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -sf "http://${ENDPOINT}/healthz" || exit 1

  deploy-production:
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    environment:
      name: production
      url: https://myapp.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to production (canary)
        run: |
          helm upgrade --install myapp ./deploy/helm/myapp \
            --namespace myapp-production \
            --set image.tag=${{ github.sha }} \
            --set canary.enabled=true \
            --set canary.weight=10 \
            --wait \
            --timeout 5m

      - name: Monitor canary (5 minutes)
        run: |
          for i in $(seq 1 30); do
            ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~'5..', deployment='myapp-canary'}[1m])" | jq '.data.result[0].value[1] // "0"' -r)
            if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
              echo "::error::Canary error rate too high: ${ERROR_RATE}"
              helm rollback myapp -n myapp-production
              exit 1
            fi
            echo "Canary check $i/30: error rate = ${ERROR_RATE}"
            sleep 10
          done

      - name: Promote canary to full deployment
        run: |
          helm upgrade --install myapp ./deploy/helm/myapp \
            --namespace myapp-production \
            --set image.tag=${{ github.sha }} \
            --set canary.enabled=false \
            --set replicas=5 \
            --wait \
            --timeout 10m
```

---

## 3. GitLab CI for Go

GitLab CI uses `.gitlab-ci.yml` at the repository root. Here is a comprehensive pipeline.

```yaml
# .gitlab-ci.yml
image: golang:1.23-alpine

variables:
  GOPATH: $CI_PROJECT_DIR/.go
  CGO_ENABLED: "0"
  GOFLAGS: "-mod=readonly"

cache:
  key:
    files:
      - go.sum
  paths:
    - .go/pkg/mod/
    - .cache/go-build/

stages:
  - validate
  - test
  - security
  - build
  - deploy

# ── Validate ──────────────────────────────────────────────────

fmt:
  stage: validate
  script:
    - test -z "$(gofmt -l .)" || (echo "Files not formatted:" && gofmt -l . && exit 1)

vet:
  stage: validate
  script:
    - go vet ./...

mod-tidy:
  stage: validate
  script:
    - go mod tidy
    - git diff --exit-code go.mod go.sum

lint:
  stage: validate
  image: golangci/golangci-lint:v1.62-alpine
  script:
    - golangci-lint run --timeout 5m
  allow_failure: false

# ── Test ──────────────────────────────────────────────────────

unit-test:
  stage: test
  script:
    - go test -race -coverprofile=coverage.out -covermode=atomic ./...
    - go tool cover -func=coverage.out
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.out
  coverage: '/total:\s+\(statements\)\s+(\d+.\d+)%/'

integration-test:
  stage: test
  services:
    - postgres:16-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgres://testuser:testpass@postgres:5432/testdb?sslmode=disable"
    REDIS_URL: "redis://redis:6379"
  script:
    - go test -tags=integration -v ./...

# ── Security ──────────────────────────────────────────────────

govulncheck:
  stage: security
  script:
    - go install golang.org/x/vuln/cmd/govulncheck@latest
    - govulncheck ./...

gosec:
  stage: security
  image: securego/gosec:latest
  script:
    - gosec -fmt=junit-xml -out=gosec-report.xml ./...
  artifacts:
    reports:
      junit: gosec-report.xml
  allow_failure: true

trivy:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --exit-code 1 --severity HIGH,CRITICAL .
  allow_failure: true

# ── Build ─────────────────────────────────────────────────────

build:
  stage: build
  script:
    - >
      go build
      -ldflags="-w -s -X main.version=${CI_COMMIT_TAG:-dev} -X main.commit=${CI_COMMIT_SHORT_SHA}"
      -trimpath
      -o myapp
      ./cmd/server
  artifacts:
    paths:
      - myapp
    expire_in: 1 week

docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - >
      docker build
      --build-arg VERSION=${CI_COMMIT_TAG:-dev}
      --build-arg COMMIT=${CI_COMMIT_SHORT_SHA}
      -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
      -t $CI_REGISTRY_IMAGE:latest
      .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
    - tags

# ── Deploy ────────────────────────────────────────────────────

deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://staging.myapp.example.com
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n staging
    - kubectl rollout status deployment/myapp -n staging --timeout=300s
  only:
    - main

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://myapp.example.com
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n production
    - kubectl rollout status deployment/myapp -n production --timeout=300s
  only:
    - tags
  when: manual
```

---

## 4. Makefile Patterns for Go Projects

A well-structured Makefile is the glue between your local development workflow and CI. It
ensures that what runs locally is exactly what runs in the pipeline.

```makefile
# Makefile

# ── Variables ─────────────────────────────────────────────────

APP_NAME     := myapp
VERSION      ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
COMMIT       := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_TIME   := $(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
GO_VERSION   := $(shell go version | awk '{print $$3}')

# Build flags
LDFLAGS := -w -s \
  -X main.version=$(VERSION) \
  -X main.commit=$(COMMIT) \
  -X main.buildTime=$(BUILD_TIME) \
  -X main.goVersion=$(GO_VERSION)

# Directories
BUILD_DIR    := ./build
CMD_DIR      := ./cmd/server
COVERAGE_DIR := ./coverage

# Tools (pinned versions)
GOLANGCI_LINT_VERSION := v1.62.0
GOVULNCHECK_VERSION   := latest
GORELEASER_VERSION    := v2.4.8

# ── Default Target ────────────────────────────────────────────

.DEFAULT_GOAL := help

# ── Help ──────────────────────────────────────────────────────

.PHONY: help
help: ## Display this help message
	@echo "Usage: make [target]"
	@echo ""
	@echo "Targets:"
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ── Dependencies ──────────────────────────────────────────────

.PHONY: deps
deps: ## Download and verify dependencies
	go mod download
	go mod verify

.PHONY: tidy
tidy: ## Run go mod tidy and verify no changes
	go mod tidy
	@git diff --exit-code go.mod go.sum || \
		(echo "ERROR: go.mod or go.sum changed after 'go mod tidy'. Please commit the changes." && exit 1)

# ── Building ──────────────────────────────────────────────────

.PHONY: build
build: ## Build the application
	@mkdir -p $(BUILD_DIR)
	CGO_ENABLED=0 go build -ldflags="$(LDFLAGS)" -trimpath -o $(BUILD_DIR)/$(APP_NAME) $(CMD_DIR)
	@echo "Built $(BUILD_DIR)/$(APP_NAME) ($(VERSION))"

.PHONY: build-all
build-all: ## Cross-compile for all platforms
	@mkdir -p $(BUILD_DIR)
	GOOS=linux   GOARCH=amd64 go build -ldflags="$(LDFLAGS)" -trimpath -o $(BUILD_DIR)/$(APP_NAME)-linux-amd64 $(CMD_DIR)
	GOOS=linux   GOARCH=arm64 go build -ldflags="$(LDFLAGS)" -trimpath -o $(BUILD_DIR)/$(APP_NAME)-linux-arm64 $(CMD_DIR)
	GOOS=darwin  GOARCH=amd64 go build -ldflags="$(LDFLAGS)" -trimpath -o $(BUILD_DIR)/$(APP_NAME)-darwin-amd64 $(CMD_DIR)
	GOOS=darwin  GOARCH=arm64 go build -ldflags="$(LDFLAGS)" -trimpath -o $(BUILD_DIR)/$(APP_NAME)-darwin-arm64 $(CMD_DIR)
	GOOS=windows GOARCH=amd64 go build -ldflags="$(LDFLAGS)" -trimpath -o $(BUILD_DIR)/$(APP_NAME)-windows-amd64.exe $(CMD_DIR)

# ── Testing ───────────────────────────────────────────────────

.PHONY: test
test: ## Run unit tests
	go test -race -count=1 ./...

.PHONY: test-v
test-v: ## Run unit tests with verbose output
	go test -race -count=1 -v ./...

.PHONY: test-short
test-short: ## Run only short tests
	go test -race -count=1 -short ./...

.PHONY: test-integration
test-integration: ## Run integration tests
	go test -race -count=1 -tags=integration -v ./...

.PHONY: test-coverage
test-coverage: ## Run tests with coverage report
	@mkdir -p $(COVERAGE_DIR)
	go test -race -coverprofile=$(COVERAGE_DIR)/coverage.out -covermode=atomic ./...
	go tool cover -func=$(COVERAGE_DIR)/coverage.out
	go tool cover -html=$(COVERAGE_DIR)/coverage.out -o $(COVERAGE_DIR)/coverage.html
	@echo "Coverage report: $(COVERAGE_DIR)/coverage.html"

.PHONY: test-coverage-check
test-coverage-check: test-coverage ## Check coverage meets threshold
	@COVERAGE=$$(go tool cover -func=$(COVERAGE_DIR)/coverage.out | grep total | awk '{print $$3}' | sed 's/%//'); \
	THRESHOLD=80.0; \
	echo "Total coverage: $${COVERAGE}%"; \
	if [ $$(echo "$${COVERAGE} < $${THRESHOLD}" | bc -l) -eq 1 ]; then \
		echo "FAIL: Coverage $${COVERAGE}% is below threshold $${THRESHOLD}%"; \
		exit 1; \
	fi

.PHONY: bench
bench: ## Run benchmarks
	go test -bench=. -benchmem -run=^$$ ./...

# ── Code Quality ──────────────────────────────────────────────

.PHONY: fmt
fmt: ## Format code
	gofmt -w -s .
	goimports -w -local github.com/myorg/myapp .

.PHONY: fmt-check
fmt-check: ## Check code formatting (CI)
	@test -z "$$(gofmt -l .)" || (echo "Files not formatted:" && gofmt -l . && exit 1)
	@test -z "$$(goimports -l -local github.com/myorg/myapp .)" || \
		(echo "Imports not organized:" && goimports -l -local github.com/myorg/myapp . && exit 1)

.PHONY: vet
vet: ## Run go vet
	go vet ./...

.PHONY: lint
lint: ## Run golangci-lint
	golangci-lint run --timeout 5m

.PHONY: lint-fix
lint-fix: ## Run golangci-lint with auto-fix
	golangci-lint run --fix --timeout 5m

.PHONY: staticcheck
staticcheck: ## Run staticcheck
	staticcheck ./...

# ── Security ──────────────────────────────────────────────────

.PHONY: vuln
vuln: ## Run govulncheck
	govulncheck ./...

.PHONY: gosec
gosec: ## Run gosec security scanner
	gosec ./...

# ── Docker ────────────────────────────────────────────────────

.PHONY: docker-build
docker-build: ## Build Docker image
	docker build \
		--build-arg VERSION=$(VERSION) \
		--build-arg COMMIT=$(COMMIT) \
		--build-arg BUILD_TIME=$(BUILD_TIME) \
		-t $(APP_NAME):$(VERSION) \
		-t $(APP_NAME):latest \
		.

.PHONY: docker-push
docker-push: ## Push Docker image
	docker push $(APP_NAME):$(VERSION)
	docker push $(APP_NAME):latest

# ── Tools ─────────────────────────────────────────────────────

.PHONY: tools
tools: ## Install development tools
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@$(GOLANGCI_LINT_VERSION)
	go install golang.org/x/vuln/cmd/govulncheck@$(GOVULNCHECK_VERSION)
	go install golang.org/x/tools/cmd/goimports@latest
	go install honnef.co/go/tools/cmd/staticcheck@latest
	go install github.com/securego/gosec/v2/cmd/gosec@latest
	go install github.com/goreleaser/goreleaser/v2@$(GORELEASER_VERSION)

# ── CI ────────────────────────────────────────────────────────

.PHONY: ci
ci: deps fmt-check vet lint test-coverage-check vuln build ## Run full CI pipeline locally
	@echo "CI pipeline passed!"

# ── Release ───────────────────────────────────────────────────

.PHONY: release-snapshot
release-snapshot: ## Build a snapshot release (no publish)
	goreleaser release --snapshot --clean

.PHONY: release
release: ## Create a full release
	goreleaser release --clean

# ── Cleanup ───────────────────────────────────────────────────

.PHONY: clean
clean: ## Remove build artifacts
	rm -rf $(BUILD_DIR) $(COVERAGE_DIR)
	go clean -cache -testcache
```

> **Tip**: The `ci` target lets developers run the full pipeline on their machine before
> pushing. This catches most issues before CI even starts.

---

## 5. Go Build Flags and Version Injection

### 5.1 Linker Flags (-ldflags)

Go's linker flags allow you to inject values into variables at compile time. This is the
standard way to embed version information into your binaries.

```go
// cmd/server/main.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"runtime"
)

// These variables are set at build time via -ldflags.
var (
	version   = "dev"
	commit    = "unknown"
	buildTime = "unknown"
	goVersion = runtime.Version()
)

// BuildInfo holds build metadata.
type BuildInfo struct {
	Version   string `json:"version"`
	Commit    string `json:"commit"`
	BuildTime string `json:"build_time"`
	GoVersion string `json:"go_version"`
	OS        string `json:"os"`
	Arch      string `json:"arch"`
}

func getBuildInfo() BuildInfo {
	return BuildInfo{
		Version:   version,
		Commit:    commit,
		BuildTime: buildTime,
		GoVersion: goVersion,
		OS:        runtime.GOOS,
		Arch:      runtime.GOARCH,
	}
}

func main() {
	// Version subcommand
	if len(os.Args) > 1 && os.Args[1] == "version" {
		info := getBuildInfo()
		data, _ := json.MarshalIndent(info, "", "  ")
		fmt.Println(string(data))
		os.Exit(0)
	}

	// Health endpoint that includes version info
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(getBuildInfo())
	})

	fmt.Printf("Starting server version=%s commit=%s\n", version, commit)
	http.ListenAndServe(":8080", nil)
}
```

**Building with ldflags:**

```bash
go build \
  -ldflags="-w -s \
    -X main.version=v1.2.3 \
    -X main.commit=$(git rev-parse --short HEAD) \
    -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -trimpath \
  -o myapp \
  ./cmd/server
```

**ldflags reference:**

| Flag | Purpose |
|------|---------|
| `-w` | Omit DWARF debugging information (reduces binary size) |
| `-s` | Omit the symbol table (reduces binary size further) |
| `-X importpath.name=value` | Set the value of a string variable at link time |
| `-trimpath` | Remove file system paths from the binary (reproducible builds) |

### 5.2 Using a Dedicated Version Package

For larger projects, centralize version info in an internal package.

```go
// internal/version/version.go
package version

import (
	"fmt"
	"runtime"
	"runtime/debug"
)

var (
	// Set via ldflags.
	Version   = "dev"
	Commit    = "unknown"
	BuildTime = "unknown"
)

// Info returns a human-readable version string.
func Info() string {
	return fmt.Sprintf("%s (commit: %s, built: %s, go: %s)",
		Version, Commit, BuildTime, runtime.Version())
}

// DetailedInfo returns version info including module details from the binary.
func DetailedInfo() string {
	info, ok := debug.ReadBuildInfo()
	if !ok {
		return Info()
	}

	var vcsRevision, vcsTime, vcsModified string
	for _, setting := range info.Settings {
		switch setting.Key {
		case "vcs.revision":
			vcsRevision = setting.Value
		case "vcs.time":
			vcsTime = setting.Value
		case "vcs.modified":
			vcsModified = setting.Value
		}
	}

	return fmt.Sprintf(
		"Version:     %s\nCommit:      %s\nBuild Time:  %s\nGo Version:  %s\nVCS Rev:     %s\nVCS Time:    %s\nVCS Modified:%s\n",
		Version, Commit, BuildTime, runtime.Version(),
		vcsRevision, vcsTime, vcsModified,
	)
}
```

Build command using the internal package:

```bash
go build -ldflags="-X github.com/myorg/myapp/internal/version.Version=v1.2.3 \
  -X github.com/myorg/myapp/internal/version.Commit=$(git rev-parse --short HEAD) \
  -X github.com/myorg/myapp/internal/version.BuildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  ./cmd/server
```

### 5.3 Build Tags

Build tags control which files are included in a build. They are essential for separating
integration tests, feature-gating platform-specific code, and more.

```go
// store_postgres.go
//go:build postgres

package store

import "database/sql"

func NewStore(dsn string) (*sql.DB, error) {
    return sql.Open("postgres", dsn)
}
```

```go
// store_sqlite.go
//go:build sqlite

package store

import "database/sql"

func NewStore(dsn string) (*sql.DB, error) {
    return sql.Open("sqlite3", dsn)
}
```

```go
// integration_test.go
//go:build integration

package myapp_test

import "testing"

func TestDatabaseIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    // ... actual integration tests
}
```

**Using build tags in CI:**

```yaml
# Run only unit tests (fast, no external deps)
- run: go test ./...

# Run integration tests (requires services)
- run: go test -tags=integration ./...

# Build with specific store backend
- run: go build -tags=postgres ./cmd/server
```

### 5.4 Build Reproducibility

For reproducible builds, combine several techniques:

```bash
# Fully reproducible build
CGO_ENABLED=0 \
GOFLAGS="-trimpath" \
GOAMD64=v1 \
go build \
  -ldflags="-w -s -buildid=" \
  -o myapp \
  ./cmd/server
```

| Technique | Purpose |
|-----------|---------|
| `CGO_ENABLED=0` | No dependency on system C libraries |
| `-trimpath` | Removes local filesystem paths from the binary |
| `-buildid=` | Removes the build ID for byte-for-byte reproducibility |
| `GOAMD64=v1` | Uses baseline x86-64 instructions (broadest compatibility) |

---

## 6. Code Quality Gates

Quality gates are automated checks that prevent substandard code from being merged.

### 6.1 Test Coverage Thresholds

#### Per-Package Coverage Script

```bash
#!/usr/bin/env bash
# scripts/check-coverage.sh
set -euo pipefail

THRESHOLD=${1:-80}
FAILED=0

echo "Checking coverage threshold: ${THRESHOLD}%"
echo "==========================================="

go test -coverprofile=coverage.out -covermode=atomic ./... 2>/dev/null

while IFS= read -r line; do
    PKG=$(echo "$line" | awk '{print $1}')
    COVERAGE=$(echo "$line" | awk '{print $3}' | sed 's/%//')

    if [ "$PKG" = "total:" ]; then
        continue
    fi

    if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
        echo "FAIL: ${PKG} coverage=${COVERAGE}% (below ${THRESHOLD}%)"
        FAILED=1
    else
        echo "PASS: ${PKG} coverage=${COVERAGE}%"
    fi
done < <(go tool cover -func=coverage.out | grep -v "total:")

# Check total
TOTAL=$(go tool cover -func=coverage.out | grep "total:" | awk '{print $3}' | sed 's/%//')
echo "==========================================="
echo "Total coverage: ${TOTAL}%"

if (( $(echo "$TOTAL < $THRESHOLD" | bc -l) )); then
    echo "FAIL: Total coverage ${TOTAL}% is below threshold ${THRESHOLD}%"
    exit 1
fi

if [ "$FAILED" -eq 1 ]; then
    echo "FAIL: Some packages are below the coverage threshold"
    exit 1
fi

echo "PASS: All packages meet the coverage threshold"
```

#### Go Test Coverage with Exclusions

```go
// coverage_test.go (test helper)
package testutil

import (
	"os"
	"os/exec"
	"strconv"
	"strings"
	"testing"
)

// TestCoverageThreshold ensures minimum coverage is maintained.
// Run with: go test -run TestCoverageThreshold -coverprofile=coverage.out ./...
func TestCoverageThreshold(t *testing.T) {
	threshold := 80.0
	if v := os.Getenv("COVERAGE_THRESHOLD"); v != "" {
		parsed, err := strconv.ParseFloat(v, 64)
		if err == nil {
			threshold = parsed
		}
	}

	out, err := exec.Command("go", "tool", "cover", "-func=coverage.out").Output()
	if err != nil {
		t.Skipf("coverage.out not found: %v", err)
	}

	lines := strings.Split(string(out), "\n")
	for _, line := range lines {
		if strings.Contains(line, "total:") {
			fields := strings.Fields(line)
			if len(fields) >= 3 {
				coverStr := strings.TrimSuffix(fields[len(fields)-1], "%")
				coverage, err := strconv.ParseFloat(coverStr, 64)
				if err != nil {
					t.Fatalf("parsing coverage: %v", err)
				}
				if coverage < threshold {
					t.Errorf("total coverage %.1f%% is below threshold %.1f%%", coverage, threshold)
				} else {
					t.Logf("total coverage %.1f%% meets threshold %.1f%%", coverage, threshold)
				}
			}
		}
	}
}
```

### 6.2 Static Analysis

#### go vet

`go vet` examines Go source code and reports suspicious constructs. It catches things like:

- Printf format string mismatches
- Unreachable code
- Copying locks
- Incorrect struct tags

```yaml
# In CI
- name: Run go vet
  run: go vet ./...
```

#### staticcheck

`staticcheck` is the most advanced standalone Go linter. It includes checks from several
categories.

```yaml
- name: Install staticcheck
  run: go install honnef.co/go/tools/cmd/staticcheck@latest

- name: Run staticcheck
  run: staticcheck ./...
```

**staticcheck configuration** (`staticcheck.conf`):

```toml
# staticcheck.conf
checks = [
    "all",       # Enable all checks
    "-ST1000",   # Don't require package comments
    "-ST1003",   # Don't enforce naming conventions for generated code
    "-SA1029",   # Allow using strings as context keys (sometimes necessary)
]

# Use the same Go version as the project
go = "1.23"
```

**Common staticcheck categories:**

| Category | Description | Example |
|----------|-------------|---------|
| SA1 | Various misuses of the standard library | `SA1012`: Nil context passed |
| SA2 | Concurrency issues | `SA2001`: Empty critical section |
| SA4 | Useless code | `SA4006`: Assigned but never used |
| SA5 | Correctness issues | `SA5001`: Defer in loop |
| SA6 | Performance issues | `SA6005`: Inefficient string comparison |
| S1 | Code simplifications | `S1000`: Use plain channel send |
| ST1 | Style issues | `ST1005`: Error string format |
| QF1 | Quick fixes | `QF1001`: Apply De Morgan's law |

### 6.3 Formatting Enforcement

#### gofmt

```bash
# Check if code is formatted (returns non-zero if not)
test -z "$(gofmt -l .)"

# Auto-format all Go files
gofmt -w -s .
```

The `-s` flag applies simplification rules:

```go
// Before -s
s[a:len(s)]      // becomes s[a:]
for x, _ = range v  // becomes for x = range v
[]T{T{}}         // becomes []T{{}}
```

#### goimports

`goimports` does everything `gofmt` does plus manages imports (adds missing ones, removes
unused ones, and groups them properly).

```bash
# Check imports
goimports -l -local github.com/myorg/myapp .

# Fix imports
goimports -w -local github.com/myorg/myapp .
```

The `-local` flag ensures your project's imports are grouped separately:

```go
import (
    // Standard library
    "context"
    "fmt"
    "net/http"

    // Third-party
    "github.com/gin-gonic/gin"
    "go.uber.org/zap"

    // Local (your project)
    "github.com/myorg/myapp/internal/config"
    "github.com/myorg/myapp/internal/store"
)
```

#### Combined Formatting Check in CI

```yaml
  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Install goimports
        run: go install golang.org/x/tools/cmd/goimports@latest

      - name: Check gofmt
        run: |
          UNFORMATTED=$(gofmt -l .)
          if [ -n "$UNFORMATTED" ]; then
            echo "::error::The following files are not formatted with gofmt:"
            echo "$UNFORMATTED"
            echo ""
            echo "Run 'gofmt -w -s .' to fix."
            exit 1
          fi

      - name: Check goimports
        run: |
          UNIMPORTED=$(goimports -l -local github.com/myorg/myapp .)
          if [ -n "$UNIMPORTED" ]; then
            echo "::error::The following files have incorrect imports:"
            echo "$UNIMPORTED"
            echo ""
            echo "Run 'goimports -w -local github.com/myorg/myapp .' to fix."
            exit 1
          fi

      - name: Check go mod tidy
        run: |
          go mod tidy
          if ! git diff --exit-code go.mod go.sum; then
            echo "::error::go.mod or go.sum not tidy. Run 'go mod tidy' and commit."
            exit 1
          fi
```

---

## 7. Release Automation

### 7.1 Semantic Versioning

Go modules use [Semantic Versioning](https://semver.org/) (SemVer): `vMAJOR.MINOR.PATCH`.

| Component | When to Increment | Example |
|-----------|-------------------|---------|
| **MAJOR** | Breaking API changes | v1.0.0 -> v2.0.0 |
| **MINOR** | New features, backward-compatible | v1.0.0 -> v1.1.0 |
| **PATCH** | Bug fixes, backward-compatible | v1.0.0 -> v1.0.1 |

**Pre-release versions**: `v1.0.0-alpha.1`, `v1.0.0-beta.2`, `v1.0.0-rc.1`

#### Automated Version Bumping Script

```bash
#!/usr/bin/env bash
# scripts/bump-version.sh
set -euo pipefail

BUMP_TYPE=${1:-patch}  # major, minor, or patch

# Get the latest tag
LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
echo "Current version: $LATEST_TAG"

# Parse version components
MAJOR=$(echo "$LATEST_TAG" | sed 's/v//' | cut -d. -f1)
MINOR=$(echo "$LATEST_TAG" | sed 's/v//' | cut -d. -f2)
PATCH=$(echo "$LATEST_TAG" | sed 's/v//' | cut -d. -f3)

case $BUMP_TYPE in
    major)
        MAJOR=$((MAJOR + 1))
        MINOR=0
        PATCH=0
        ;;
    minor)
        MINOR=$((MINOR + 1))
        PATCH=0
        ;;
    patch)
        PATCH=$((PATCH + 1))
        ;;
    *)
        echo "Usage: $0 [major|minor|patch]"
        exit 1
        ;;
esac

NEW_TAG="v${MAJOR}.${MINOR}.${PATCH}"
echo "New version: $NEW_TAG"

# Create annotated tag
git tag -a "$NEW_TAG" -m "Release $NEW_TAG"
echo "Created tag $NEW_TAG"
echo ""
echo "To push: git push origin $NEW_TAG"
```

### 7.2 goreleaser

`goreleaser` is the standard tool for automating Go releases. It handles cross-compilation,
changelogs, Docker images, Homebrew taps, and more.

#### goreleaser Configuration

```yaml
# .goreleaser.yml
version: 2

project_name: myapp

before:
  hooks:
    - go mod tidy
    - go generate ./...
    - go test ./...

builds:
  - id: myapp
    main: ./cmd/server
    binary: myapp
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    goarm:
      - "7"
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.commit={{.Commit}}
      - -X main.buildTime={{.Date}}
    mod_timestamp: '{{ .CommitTimestamp }}'
    flags:
      - -trimpath

  - id: myapp-cli
    main: ./cmd/cli
    binary: myapp-cli
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64

archives:
  - id: default
    builds:
      - myapp
      - myapp-cli
    format: tar.gz
    name_template: >-
      {{ .ProjectName }}_
      {{- title .Os }}_
      {{- if eq .Arch "amd64" }}x86_64
      {{- else if eq .Arch "386" }}i386
      {{- else }}{{ .Arch }}{{ end }}
    format_overrides:
      - goos: windows
        format: zip
    files:
      - LICENSE
      - README.md
      - CHANGELOG.md

checksum:
  name_template: 'checksums.txt'
  algorithm: sha256

snapshot:
  version_template: "{{ incpatch .Version }}-next"

changelog:
  sort: asc
  use: github
  groups:
    - title: "New Features"
      regexp: '^.*?feat(\([[:word:]]+\))??!?:.+$'
      order: 0
    - title: "Bug Fixes"
      regexp: '^.*?fix(\([[:word:]]+\))??!?:.+$'
      order: 1
    - title: "Performance"
      regexp: '^.*?perf(\([[:word:]]+\))??!?:.+$'
      order: 2
    - title: "Documentation"
      regexp: '^.*?docs(\([[:word:]]+\))??!?:.+$'
      order: 3
    - title: "Other"
      order: 999
  filters:
    exclude:
      - '^docs:'
      - '^test:'
      - '^chore:'
      - '^ci:'
      - Merge pull request
      - Merge branch

# Docker images
dockers:
  - image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}"
      - "ghcr.io/myorg/myapp:v{{ .Major }}"
      - "ghcr.io/myorg/myapp:v{{ .Major }}.{{ .Minor }}"
      - "ghcr.io/myorg/myapp:latest"
    dockerfile: Dockerfile
    use: buildx
    build_flag_templates:
      - "--platform=linux/amd64"
      - "--label=org.opencontainers.image.title={{ .ProjectName }}"
      - "--label=org.opencontainers.image.version={{ .Version }}"
      - "--label=org.opencontainers.image.created={{ .Date }}"
      - "--label=org.opencontainers.image.revision={{ .FullCommit }}"
      - "--build-arg=VERSION={{ .Version }}"
      - "--build-arg=COMMIT={{ .Commit }}"

  - image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}-arm64"
    dockerfile: Dockerfile
    use: buildx
    build_flag_templates:
      - "--platform=linux/arm64"
    goarch: arm64

docker_manifests:
  - name_template: "ghcr.io/myorg/myapp:{{ .Tag }}"
    image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}"
      - "ghcr.io/myorg/myapp:{{ .Tag }}-arm64"
  - name_template: "ghcr.io/myorg/myapp:latest"
    image_templates:
      - "ghcr.io/myorg/myapp:{{ .Tag }}"
      - "ghcr.io/myorg/myapp:{{ .Tag }}-arm64"

# Homebrew tap
brews:
  - name: myapp
    repository:
      owner: myorg
      name: homebrew-tap
      token: "{{ .Env.HOMEBREW_TAP_TOKEN }}"
    directory: Formula
    homepage: "https://github.com/myorg/myapp"
    description: "My application description"
    license: "MIT"
    test: |
      system "#{bin}/myapp", "version"
    install: |
      bin.install "myapp"
      bin.install "myapp-cli"

# Sign releases
signs:
  - artifacts: checksum
    args:
      - "--batch"
      - "--local-user"
      - "{{ .Env.GPG_FINGERPRINT }}"
      - "--output"
      - "${signature}"
      - "--detach-sign"
      - "${artifact}"

# GitHub release
release:
  github:
    owner: myorg
    name: myapp
  draft: false
  prerelease: auto
  name_template: "{{ .Tag }}"
  header: |
    ## What's Changed

    See the [changelog](https://github.com/myorg/myapp/blob/main/CHANGELOG.md) for details.
  footer: |
    **Full Changelog**: https://github.com/myorg/myapp/compare/{{ .PreviousTag }}...{{ .Tag }}

# Announce on social media (optional)
announce:
  slack:
    enabled: true
    channel: '#releases'
    message_template: 'myapp {{ .Tag }} is out! Check it out: {{ .ReleaseURL }}'
```

#### GitHub Actions Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for changelog generation

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: '~> v2'
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
```

### 7.3 Container Image Tagging Strategies

Consistent image tagging is critical for traceability and rollbacks.

| Strategy | Tag Example | Use Case |
|----------|-------------|----------|
| **Git SHA** | `abc1234` | Every commit, perfect traceability |
| **SemVer** | `v1.2.3` | Release tags, human-readable |
| **Major.Minor** | `v1.2` | Float tag, always points to latest patch |
| **Major** | `v1` | Float tag, latest minor of major version |
| **latest** | `latest` | Most recent stable release |
| **Branch** | `main`, `develop` | Track branch HEAD |
| **PR** | `pr-42` | Pull request builds |
| **Timestamp** | `20260324-143022` | When you need ordering |

**Multi-tag strategy in CI:**

```yaml
      - name: Generate tags
        id: tags
        run: |
          IMAGE="ghcr.io/myorg/myapp"
          SHA="${{ github.sha }}"
          SHORT_SHA="${SHA:0:7}"
          BRANCH="${GITHUB_REF##*/}"

          TAGS="${IMAGE}:${SHORT_SHA}"

          if [[ "$GITHUB_REF" == refs/tags/v* ]]; then
            VERSION="${GITHUB_REF#refs/tags/}"
            MAJOR=$(echo "$VERSION" | cut -d. -f1)
            MINOR=$(echo "$VERSION" | cut -d. -f1-2)
            TAGS="${TAGS},${IMAGE}:${VERSION},${IMAGE}:${MINOR},${IMAGE}:${MAJOR},${IMAGE}:latest"
          elif [[ "$BRANCH" == "main" ]]; then
            TAGS="${TAGS},${IMAGE}:main"
          fi

          echo "tags=${TAGS}" >> "$GITHUB_OUTPUT"
```

---

## 8. Feature Flags and Gradual Rollouts

### 8.1 Simple Feature Flags in Go

For small projects, environment-based feature flags are sufficient.

```go
// internal/features/flags.go
package features

import (
	"os"
	"strconv"
	"strings"
	"sync"
)

// Flag represents a feature flag.
type Flag struct {
	Name         string
	Description  string
	DefaultValue bool
}

// Registry holds all feature flags.
type Registry struct {
	mu    sync.RWMutex
	flags map[string]*flagState
}

type flagState struct {
	flag    Flag
	enabled bool
}

// NewRegistry creates a new feature flag registry.
func NewRegistry() *Registry {
	return &Registry{
		flags: make(map[string]*flagState),
	}
}

// Register adds a feature flag.
func (r *Registry) Register(f Flag) {
	r.mu.Lock()
	defer r.mu.Unlock()

	enabled := f.DefaultValue

	// Check environment variable: FEATURE_<UPPER_NAME>=true|false
	envKey := "FEATURE_" + strings.ToUpper(strings.ReplaceAll(f.Name, "-", "_"))
	if val, ok := os.LookupEnv(envKey); ok {
		if b, err := strconv.ParseBool(val); err == nil {
			enabled = b
		}
	}

	r.flags[f.Name] = &flagState{flag: f, enabled: enabled}
}

// IsEnabled checks if a feature flag is enabled.
func (r *Registry) IsEnabled(name string) bool {
	r.mu.RLock()
	defer r.mu.RUnlock()

	if state, ok := r.flags[name]; ok {
		return state.enabled
	}
	return false
}

// SetEnabled dynamically toggles a flag (useful for remote config).
func (r *Registry) SetEnabled(name string, enabled bool) {
	r.mu.Lock()
	defer r.mu.Unlock()

	if state, ok := r.flags[name]; ok {
		state.enabled = enabled
	}
}

// All returns all registered flags and their states.
func (r *Registry) All() map[string]bool {
	r.mu.RLock()
	defer r.mu.RUnlock()

	result := make(map[string]bool, len(r.flags))
	for name, state := range r.flags {
		result[name] = state.enabled
	}
	return result
}
```

#### Usage in Application Code

```go
// main.go
package main

import (
	"log"
	"net/http"

	"github.com/myorg/myapp/internal/features"
)

var flags *features.Registry

func init() {
	flags = features.NewRegistry()

	flags.Register(features.Flag{
		Name:         "new-dashboard",
		Description:  "Enable the redesigned dashboard UI",
		DefaultValue: false,
	})
	flags.Register(features.Flag{
		Name:         "async-notifications",
		Description:  "Process notifications asynchronously",
		DefaultValue: true,
	})
	flags.Register(features.Flag{
		Name:         "v2-api",
		Description:  "Enable v2 API endpoints",
		DefaultValue: false,
	})
}

func handleDashboard(w http.ResponseWriter, r *http.Request) {
	if flags.IsEnabled("new-dashboard") {
		// Serve new dashboard
		serveNewDashboard(w, r)
		return
	}
	// Serve legacy dashboard
	serveLegacyDashboard(w, r)
}

func serveNewDashboard(w http.ResponseWriter, r *http.Request)    { /* ... */ }
func serveLegacyDashboard(w http.ResponseWriter, r *http.Request) { /* ... */ }

func main() {
	log.Printf("Feature flags: %v", flags.All())
	http.HandleFunc("/dashboard", handleDashboard)
	http.ListenAndServe(":8080", nil)
}
```

### 8.2 Percentage-Based Rollouts

```go
// internal/features/rollout.go
package features

import (
	"hash/fnv"
	"sync"
)

// RolloutFlag supports gradual rollout by percentage.
type RolloutFlag struct {
	Name       string
	Percentage int // 0-100
}

// RolloutRegistry manages percentage-based feature flags.
type RolloutRegistry struct {
	mu    sync.RWMutex
	flags map[string]*RolloutFlag
}

// NewRolloutRegistry creates a new rollout registry.
func NewRolloutRegistry() *RolloutRegistry {
	return &RolloutRegistry{
		flags: make(map[string]*RolloutFlag),
	}
}

// Register adds a rollout flag.
func (r *RolloutRegistry) Register(f RolloutFlag) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.flags[f.Name] = &f
}

// SetPercentage updates the rollout percentage.
func (r *RolloutRegistry) SetPercentage(name string, pct int) {
	r.mu.Lock()
	defer r.mu.Unlock()

	if pct < 0 {
		pct = 0
	}
	if pct > 100 {
		pct = 100
	}

	if f, ok := r.flags[name]; ok {
		f.Percentage = pct
	}
}

// IsEnabledFor checks if a feature is enabled for a specific user.
// Uses consistent hashing so the same user always gets the same result
// for a given percentage.
func (r *RolloutRegistry) IsEnabledFor(name, userID string) bool {
	r.mu.RLock()
	defer r.mu.RUnlock()

	flag, ok := r.flags[name]
	if !ok {
		return false
	}

	if flag.Percentage >= 100 {
		return true
	}
	if flag.Percentage <= 0 {
		return false
	}

	// Consistent hash: same user + flag always produces the same bucket
	h := fnv.New32a()
	h.Write([]byte(name + ":" + userID))
	bucket := int(h.Sum32() % 100)

	return bucket < flag.Percentage
}
```

#### Deploying with Gradual Rollout in CI

```yaml
  gradual-rollout:
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    steps:
      - name: Enable feature for 10%
        run: |
          curl -X PUT "https://config.internal/api/flags/new-dashboard" \
            -H "Authorization: Bearer ${{ secrets.CONFIG_TOKEN }}" \
            -d '{"percentage": 10}'

      - name: Wait and monitor (5 minutes)
        run: |
          sleep 300
          ERROR_RATE=$(curl -s "http://metrics.internal/api/v1/query?query=rate(http_errors_total{feature='new-dashboard'}[5m])" | jq -r '.data.result[0].value[1] // "0"')
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Rolling back feature flag"
            curl -X PUT "https://config.internal/api/flags/new-dashboard" \
              -H "Authorization: Bearer ${{ secrets.CONFIG_TOKEN }}" \
              -d '{"percentage": 0}'
            exit 1
          fi

      - name: Increase to 50%
        run: |
          curl -X PUT "https://config.internal/api/flags/new-dashboard" \
            -H "Authorization: Bearer ${{ secrets.CONFIG_TOKEN }}" \
            -d '{"percentage": 50}'

      - name: Wait and monitor (10 minutes)
        run: sleep 600

      - name: Full rollout
        run: |
          curl -X PUT "https://config.internal/api/flags/new-dashboard" \
            -H "Authorization: Bearer ${{ secrets.CONFIG_TOKEN }}" \
            -d '{"percentage": 100}'
```

---

## 9. Database Migrations in CI/CD

### 9.1 Migration Tooling with golang-migrate

```go
// internal/database/migrate.go
package database

import (
	"database/sql"
	"embed"
	"fmt"

	"github.com/golang-migrate/migrate/v4"
	"github.com/golang-migrate/migrate/v4/database/postgres"
	"github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

// RunMigrations applies all pending database migrations.
func RunMigrations(db *sql.DB, dbName string) error {
	// Create migration source from embedded files
	source, err := iofs.New(migrationsFS, "migrations")
	if err != nil {
		return fmt.Errorf("creating migration source: %w", err)
	}

	// Create migration driver
	driver, err := postgres.WithInstance(db, &postgres.Config{})
	if err != nil {
		return fmt.Errorf("creating migration driver: %w", err)
	}

	// Create migrate instance
	m, err := migrate.NewWithInstance("iofs", source, dbName, driver)
	if err != nil {
		return fmt.Errorf("creating migrate instance: %w", err)
	}

	// Apply all up migrations
	if err := m.Up(); err != nil && err != migrate.ErrNoChange {
		return fmt.Errorf("running migrations: %w", err)
	}

	version, dirty, err := m.Version()
	if err != nil && err != migrate.ErrNoChange {
		return fmt.Errorf("getting migration version: %w", err)
	}

	fmt.Printf("Database migrated to version %d (dirty: %v)\n", version, dirty)
	return nil
}

// RollbackMigration rolls back the last migration.
func RollbackMigration(db *sql.DB, dbName string) error {
	source, err := iofs.New(migrationsFS, "migrations")
	if err != nil {
		return fmt.Errorf("creating migration source: %w", err)
	}

	driver, err := postgres.WithInstance(db, &postgres.Config{})
	if err != nil {
		return fmt.Errorf("creating migration driver: %w", err)
	}

	m, err := migrate.NewWithInstance("iofs", source, dbName, driver)
	if err != nil {
		return fmt.Errorf("creating migrate instance: %w", err)
	}

	if err := m.Steps(-1); err != nil {
		return fmt.Errorf("rolling back migration: %w", err)
	}

	return nil
}
```

#### Migration Files

```sql
-- internal/database/migrations/000001_create_users.up.sql
CREATE TABLE IF NOT EXISTS users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL UNIQUE,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

```sql
-- internal/database/migrations/000001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

```sql
-- internal/database/migrations/000002_add_user_status.up.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';
CREATE INDEX idx_users_status ON users(status);
```

```sql
-- internal/database/migrations/000002_add_user_status.down.sql
DROP INDEX IF EXISTS idx_users_status;
ALTER TABLE users DROP COLUMN IF EXISTS status;
```

### 9.2 Migration CI Pipeline

```yaml
  migrate:
    runs-on: ubuntu-latest
    needs: [test]
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Install migrate CLI
        run: |
          go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

      - name: Validate migrations (up then down)
        env:
          DATABASE_URL: "postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable"
        run: |
          # Apply all migrations
          migrate -database "$DATABASE_URL" -path internal/database/migrations up
          echo "All migrations applied successfully"

          # Verify we can roll back all migrations
          migrate -database "$DATABASE_URL" -path internal/database/migrations down -all
          echo "All migrations rolled back successfully"

          # Re-apply to verify idempotency
          migrate -database "$DATABASE_URL" -path internal/database/migrations up
          echo "Re-applied migrations successfully"

      - name: Run migration tests
        env:
          DATABASE_URL: "postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable"
        run: go test -tags=integration -v ./internal/database/...
```

### 9.3 Safe Migration Practices in CI

```go
// internal/database/migrate_safe.go
package database

import (
	"context"
	"database/sql"
	"fmt"
	"time"
)

// MigrationLock prevents concurrent migrations using an advisory lock.
type MigrationLock struct {
	db     *sql.DB
	lockID int64
}

// NewMigrationLock creates a new migration lock.
func NewMigrationLock(db *sql.DB) *MigrationLock {
	return &MigrationLock{
		db:     db,
		lockID: 123456789, // Unique advisory lock ID for migrations
	}
}

// Acquire obtains an advisory lock. Returns an unlock function.
func (ml *MigrationLock) Acquire(ctx context.Context) (func(), error) {
	ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()

	// Try to acquire advisory lock
	var acquired bool
	err := ml.db.QueryRowContext(ctx,
		"SELECT pg_try_advisory_lock($1)", ml.lockID).Scan(&acquired)
	if err != nil {
		return nil, fmt.Errorf("acquiring advisory lock: %w", err)
	}

	if !acquired {
		return nil, fmt.Errorf("migration lock already held by another process")
	}

	unlock := func() {
		ml.db.Exec("SELECT pg_advisory_unlock($1)", ml.lockID)
	}

	return unlock, nil
}

// SafeMigrate runs migrations with locking and timeout.
func SafeMigrate(ctx context.Context, db *sql.DB, dbName string) error {
	lock := NewMigrationLock(db)

	unlock, err := lock.Acquire(ctx)
	if err != nil {
		return fmt.Errorf("acquiring migration lock: %w", err)
	}
	defer unlock()

	// Set statement timeout to prevent long-running DDL
	_, err = db.ExecContext(ctx, "SET statement_timeout = '60s'")
	if err != nil {
		return fmt.Errorf("setting statement timeout: %w", err)
	}

	return RunMigrations(db, dbName)
}
```

> **Warning**: Never run destructive migrations (DROP TABLE, DROP COLUMN) in a single step
> in production. Use a multi-phase approach:
> 1. Deploy code that stops reading the column
> 2. Run the migration to drop the column
> 3. Deploy code that removes the column from models
>
> This prevents downtime during rolling deployments where old and new code coexist.

---

## 10. Integration Testing in Pipelines

### 10.1 Testcontainers for Go

Testcontainers lets you spin up real Docker containers in your tests, giving you true
integration testing without mocks.

```go
// internal/store/user_store_integration_test.go
//go:build integration

package store_test

import (
	"context"
	"database/sql"
	"fmt"
	"testing"
	"time"

	_ "github.com/lib/pq"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/modules/postgres"
	"github.com/testcontainers/testcontainers-go/wait"

	"github.com/myorg/myapp/internal/store"
)

func setupPostgresContainer(t *testing.T) (*sql.DB, func()) {
	t.Helper()
	ctx := context.Background()

	pgContainer, err := postgres.Run(ctx,
		"postgres:16-alpine",
		postgres.WithDatabase("testdb"),
		postgres.WithUsername("testuser"),
		postgres.WithPassword("testpass"),
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").
				WithOccurrence(2).
				WithStartupTimeout(30*time.Second),
		),
	)
	require.NoError(t, err)

	connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
	require.NoError(t, err)

	db, err := sql.Open("postgres", connStr)
	require.NoError(t, err)

	// Wait for connection
	require.NoError(t, db.PingContext(ctx))

	// Run migrations
	_, err = db.ExecContext(ctx, `
		CREATE TABLE IF NOT EXISTS users (
			id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
			email VARCHAR(255) NOT NULL UNIQUE,
			name VARCHAR(255) NOT NULL,
			created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
		)
	`)
	require.NoError(t, err)

	cleanup := func() {
		db.Close()
		pgContainer.Terminate(ctx)
	}

	return db, cleanup
}

func TestUserStore_Create(t *testing.T) {
	db, cleanup := setupPostgresContainer(t)
	defer cleanup()

	userStore := store.NewUserStore(db)
	ctx := context.Background()

	user, err := userStore.Create(ctx, "john@example.com", "John Doe")
	require.NoError(t, err)
	assert.NotEmpty(t, user.ID)
	assert.Equal(t, "john@example.com", user.Email)
	assert.Equal(t, "John Doe", user.Name)
}

func TestUserStore_GetByEmail(t *testing.T) {
	db, cleanup := setupPostgresContainer(t)
	defer cleanup()

	userStore := store.NewUserStore(db)
	ctx := context.Background()

	// Create a user
	created, err := userStore.Create(ctx, "jane@example.com", "Jane Doe")
	require.NoError(t, err)

	// Retrieve by email
	found, err := userStore.GetByEmail(ctx, "jane@example.com")
	require.NoError(t, err)
	assert.Equal(t, created.ID, found.ID)
	assert.Equal(t, "Jane Doe", found.Name)
}

func TestUserStore_DuplicateEmail(t *testing.T) {
	db, cleanup := setupPostgresContainer(t)
	defer cleanup()

	userStore := store.NewUserStore(db)
	ctx := context.Background()

	_, err := userStore.Create(ctx, "dup@example.com", "User One")
	require.NoError(t, err)

	_, err = userStore.Create(ctx, "dup@example.com", "User Two")
	assert.Error(t, err)
	assert.ErrorIs(t, err, store.ErrDuplicateEmail)
}
```

### 10.2 Testing with Redis Container

```go
// internal/cache/redis_integration_test.go
//go:build integration

package cache_test

import (
	"context"
	"testing"
	"time"

	"github.com/redis/go-redis/v9"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"github.com/testcontainers/testcontainers-go"
	tcRedis "github.com/testcontainers/testcontainers-go/modules/redis"

	"github.com/myorg/myapp/internal/cache"
)

func setupRedisContainer(t *testing.T) (*redis.Client, func()) {
	t.Helper()
	ctx := context.Background()

	redisContainer, err := tcRedis.Run(ctx,
		"redis:7-alpine",
	)
	require.NoError(t, err)

	endpoint, err := redisContainer.Endpoint(ctx, "")
	require.NoError(t, err)

	client := redis.NewClient(&redis.Options{
		Addr: endpoint,
	})

	require.NoError(t, client.Ping(ctx).Err())

	cleanup := func() {
		client.Close()
		redisContainer.Terminate(ctx)
	}

	return client, cleanup
}

func TestCache_SetAndGet(t *testing.T) {
	client, cleanup := setupRedisContainer(t)
	defer cleanup()

	c := cache.New(client)
	ctx := context.Background()

	err := c.Set(ctx, "user:123", `{"name":"John"}`, 5*time.Minute)
	require.NoError(t, err)

	val, err := c.Get(ctx, "user:123")
	require.NoError(t, err)
	assert.Equal(t, `{"name":"John"}`, val)
}

func TestCache_Expiration(t *testing.T) {
	client, cleanup := setupRedisContainer(t)
	defer cleanup()

	c := cache.New(client)
	ctx := context.Background()

	err := c.Set(ctx, "temp", "value", 1*time.Second)
	require.NoError(t, err)

	time.Sleep(2 * time.Second)

	_, err = c.Get(ctx, "temp")
	assert.ErrorIs(t, err, cache.ErrNotFound)
}
```

### 10.3 Docker Compose for Integration Tests

For complex scenarios with multiple services, docker-compose is practical.

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qg'
    ports:
      - "9092:9092"

  localstack:
    image: localstack/localstack:latest
    environment:
      SERVICES: s3,sqs,sns
      DEFAULT_REGION: us-east-1
    ports:
      - "4566:4566"
```

```bash
#!/usr/bin/env bash
# scripts/integration-test.sh
set -euo pipefail

echo "Starting test infrastructure..."
docker compose -f docker-compose.test.yml up -d --wait

echo "Waiting for services to be healthy..."
sleep 5

echo "Running integration tests..."
DATABASE_URL="postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable" \
REDIS_URL="redis://localhost:6379" \
KAFKA_BROKERS="localhost:9092" \
AWS_ENDPOINT="http://localhost:4566" \
go test -tags=integration -v -count=1 -timeout=10m ./...
TEST_EXIT_CODE=$?

echo "Stopping test infrastructure..."
docker compose -f docker-compose.test.yml down -v

exit $TEST_EXIT_CODE
```

#### CI Integration

```yaml
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Start services
        run: docker compose -f docker-compose.test.yml up -d --wait

      - name: Run integration tests
        env:
          DATABASE_URL: "postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable"
          REDIS_URL: "redis://localhost:6379"
          KAFKA_BROKERS: "localhost:9092"
        run: go test -tags=integration -v -count=1 -timeout=10m ./...

      - name: Tear down services
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

### 10.4 Shared Test Helpers

```go
// internal/testutil/testutil.go
package testutil

import (
	"context"
	"database/sql"
	"os"
	"testing"
	"time"
)

// SkipIfShort skips the test if -short flag is provided.
func SkipIfShort(t *testing.T) {
	t.Helper()
	if testing.Short() {
		t.Skip("skipping integration test in short mode")
	}
}

// RequireEnv skips the test if required environment variables are missing.
func RequireEnv(t *testing.T, keys ...string) {
	t.Helper()
	for _, key := range keys {
		if os.Getenv(key) == "" {
			t.Skipf("required environment variable %s not set", key)
		}
	}
}

// ConnectDB establishes a database connection with retry logic for CI.
func ConnectDB(t *testing.T, dsn string) *sql.DB {
	t.Helper()

	var db *sql.DB
	var err error

	for i := 0; i < 30; i++ {
		db, err = sql.Open("postgres", dsn)
		if err == nil {
			ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
			err = db.PingContext(ctx)
			cancel()
			if err == nil {
				return db
			}
		}
		time.Sleep(1 * time.Second)
	}

	t.Fatalf("could not connect to database after 30 attempts: %v", err)
	return nil
}

// TruncateTables clears all data from the given tables.
func TruncateTables(t *testing.T, db *sql.DB, tables ...string) {
	t.Helper()
	for _, table := range tables {
		_, err := db.Exec("TRUNCATE TABLE " + table + " CASCADE")
		if err != nil {
			t.Fatalf("truncating table %s: %v", table, err)
		}
	}
}
```

---

## 11. Secrets Management in CI

### 11.1 GitHub Actions Secrets

**Types of secrets in GitHub:**

| Type | Scope | Access |
|------|-------|--------|
| **Repository secrets** | Single repository | All workflows |
| **Environment secrets** | Specific environment | Only jobs targeting that environment |
| **Organization secrets** | All repos in org | Configurable per repo |

#### Accessing Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Required to access environment secrets
    steps:
      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          ./deploy.sh
```

> **Warning**: Never echo secrets in logs. GitHub masks known secrets, but derived values
> (e.g., base64-encoded secrets) will not be masked automatically. Use `::add-mask::` to
> mask dynamic values:
> ```yaml
> - run: |
>     DERIVED_SECRET=$(echo "${{ secrets.MY_SECRET }}" | base64)
>     echo "::add-mask::${DERIVED_SECRET}"
> ```

### 11.2 OIDC Authentication (Keyless)

Modern CI avoids long-lived secrets by using OIDC (OpenID Connect) tokens.

```yaml
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC
      contents: read
    steps:
      # AWS with OIDC
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1

      # GCP with OIDC
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/github/providers/github'
          service_account: 'deploy@my-project.iam.gserviceaccount.com'
```

### 11.3 Runtime Secrets in Go Applications

```go
// internal/config/secrets.go
package config

import (
	"context"
	"encoding/json"
	"fmt"
	"os"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	awsconfig "github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/secretsmanager"
)

// SecretProvider abstracts secret retrieval.
type SecretProvider interface {
	GetSecret(ctx context.Context, name string) (string, error)
}

// EnvSecretProvider reads secrets from environment variables.
type EnvSecretProvider struct{}

func (p *EnvSecretProvider) GetSecret(_ context.Context, name string) (string, error) {
	val := os.Getenv(name)
	if val == "" {
		return "", fmt.Errorf("secret %q not found in environment", name)
	}
	return val, nil
}

// AWSSecretProvider reads secrets from AWS Secrets Manager.
type AWSSecretProvider struct {
	client *secretsmanager.Client
}

func NewAWSSecretProvider(ctx context.Context) (*AWSSecretProvider, error) {
	cfg, err := awsconfig.LoadDefaultConfig(ctx)
	if err != nil {
		return nil, fmt.Errorf("loading AWS config: %w", err)
	}
	return &AWSSecretProvider{
		client: secretsmanager.NewFromConfig(cfg),
	}, nil
}

func (p *AWSSecretProvider) GetSecret(ctx context.Context, name string) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	result, err := p.client.GetSecretValue(ctx, &secretsmanager.GetSecretValueInput{
		SecretId: aws.String(name),
	})
	if err != nil {
		return "", fmt.Errorf("getting secret %q: %w", name, err)
	}

	return *result.SecretString, nil
}

// DatabaseConfig holds database credentials.
type DatabaseConfig struct {
	Host     string `json:"host"`
	Port     int    `json:"port"`
	Username string `json:"username"`
	Password string `json:"password"`
	DBName   string `json:"dbname"`
}

// LoadDatabaseConfig loads database configuration from secrets.
func LoadDatabaseConfig(ctx context.Context, provider SecretProvider) (*DatabaseConfig, error) {
	secretJSON, err := provider.GetSecret(ctx, "myapp/database")
	if err != nil {
		return nil, err
	}

	var config DatabaseConfig
	if err := json.Unmarshal([]byte(secretJSON), &config); err != nil {
		return nil, fmt.Errorf("parsing database config: %w", err)
	}

	return &config, nil
}

// DSN returns a PostgreSQL connection string.
func (c *DatabaseConfig) DSN() string {
	return fmt.Sprintf(
		"host=%s port=%d user=%s password=%s dbname=%s sslmode=require",
		c.Host, c.Port, c.Username, c.Password, c.DBName,
	)
}
```

### 11.4 Secrets Best Practices

| Practice | Description |
|----------|-------------|
| **Never hardcode** | Use environment variables or secret managers |
| **Rotate regularly** | Automate secret rotation (AWS Secrets Manager, Vault) |
| **Least privilege** | Grant only the permissions the pipeline needs |
| **Audit access** | Log when and who accessed secrets |
| **Use OIDC** | Prefer short-lived tokens over long-lived credentials |
| **Separate environments** | Different secrets for dev, staging, production |
| **Encrypt at rest** | Use encrypted secret stores, not plain text files |
| **Scan for leaks** | Use tools like `gitleaks` or `trufflehog` in CI |

```yaml
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 12. Monitoring Deployments

### 12.1 Health Check Endpoint

```go
// internal/health/health.go
package health

import (
	"context"
	"database/sql"
	"encoding/json"
	"net/http"
	"sync"
	"time"
)

// Status represents the health status of a component.
type Status string

const (
	StatusUp       Status = "up"
	StatusDown     Status = "down"
	StatusDegraded Status = "degraded"
)

// Check is a function that checks the health of a dependency.
type Check func(ctx context.Context) error

// Result represents the result of a health check.
type Result struct {
	Status   Status        `json:"status"`
	Duration time.Duration `json:"duration"`
	Error    string        `json:"error,omitempty"`
}

// Response is the full health check response.
type Response struct {
	Status  Status            `json:"status"`
	Version string            `json:"version"`
	Checks  map[string]Result `json:"checks"`
}

// Handler provides HTTP health check endpoints.
type Handler struct {
	version string
	checks  map[string]Check
}

// NewHandler creates a new health check handler.
func NewHandler(version string) *Handler {
	return &Handler{
		version: version,
		checks:  make(map[string]Check),
	}
}

// AddCheck registers a health check.
func (h *Handler) AddCheck(name string, check Check) {
	h.checks[name] = check
}

// AddDatabaseCheck adds a database health check.
func (h *Handler) AddDatabaseCheck(name string, db *sql.DB) {
	h.AddCheck(name, func(ctx context.Context) error {
		ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
		defer cancel()
		return db.PingContext(ctx)
	})
}

// ServeHTTP handles health check requests.
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
	defer cancel()

	response := Response{
		Status:  StatusUp,
		Version: h.version,
		Checks:  make(map[string]Result),
	}

	var mu sync.Mutex
	var wg sync.WaitGroup

	for name, check := range h.checks {
		wg.Add(1)
		go func(name string, check Check) {
			defer wg.Done()

			start := time.Now()
			err := check(ctx)
			duration := time.Since(start)

			result := Result{
				Status:   StatusUp,
				Duration: duration,
			}
			if err != nil {
				result.Status = StatusDown
				result.Error = err.Error()
			}

			mu.Lock()
			response.Checks[name] = result
			if result.Status == StatusDown {
				response.Status = StatusDegraded
			}
			mu.Unlock()
		}(name, check)
	}

	wg.Wait()

	// If any critical check is down, overall status is down
	for _, result := range response.Checks {
		if result.Status == StatusDown {
			response.Status = StatusDown
			break
		}
	}

	w.Header().Set("Content-Type", "application/json")
	if response.Status != StatusUp {
		w.WriteHeader(http.StatusServiceUnavailable)
	}
	json.NewEncoder(w).Encode(response)
}
```

### 12.2 Deployment Metrics

```go
// internal/metrics/deployment.go
package metrics

import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var (
	// DeploymentInfo tracks the currently running deployment.
	DeploymentInfo = promauto.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "deployment_info",
			Help: "Information about the current deployment",
		},
		[]string{"version", "commit", "environment"},
	)

	// HTTPRequestsTotal tracks HTTP request counts.
	HTTPRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "Total number of HTTP requests",
		},
		[]string{"method", "path", "status"},
	)

	// HTTPRequestDuration tracks HTTP request latencies.
	HTTPRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "HTTP request latencies in seconds",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"method", "path"},
	)

	// HTTPRequestsInFlight tracks concurrent HTTP requests.
	HTTPRequestsInFlight = promauto.NewGauge(
		prometheus.GaugeOpts{
			Name: "http_requests_in_flight",
			Help: "Number of HTTP requests currently being processed",
		},
	)
)

// RecordDeployment records the current deployment version.
func RecordDeployment(version, commit, environment string) {
	DeploymentInfo.WithLabelValues(version, commit, environment).Set(1)
}
```

### 12.3 Canary Deployments

A canary deployment sends a small percentage of traffic to the new version before full
rollout.

```yaml
# deploy/helm/myapp/templates/canary.yaml
{{- if .Values.canary.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-canary
  labels:
    app: {{ .Release.Name }}
    track: canary
spec:
  replicas: {{ .Values.canary.replicas | default 1 }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
      track: canary
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
        track: canary
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
{{- end }}
```

#### Canary Promotion Script

```bash
#!/usr/bin/env bash
# scripts/canary-promote.sh
set -euo pipefail

NAMESPACE=$1
RELEASE=$2
IMAGE_TAG=$3
METRICS_URL=$4

echo "=== Starting canary deployment ==="

# Phase 1: Deploy canary (10% traffic)
echo "Phase 1: Deploying canary with 10% traffic..."
helm upgrade --install "$RELEASE" ./deploy/helm/myapp \
  --namespace "$NAMESPACE" \
  --set image.tag="$IMAGE_TAG" \
  --set canary.enabled=true \
  --set canary.weight=10 \
  --wait --timeout 5m

sleep 120  # Observe for 2 minutes

# Check error rate
ERROR_RATE=$(curl -s "$METRICS_URL/api/v1/query?query=rate(http_requests_total{status=~'5..',track='canary'}[2m])" | jq -r '.data.result[0].value[1] // "0"')
echo "Canary error rate: $ERROR_RATE"
if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
  echo "FAIL: Error rate too high, rolling back canary"
  helm upgrade --install "$RELEASE" ./deploy/helm/myapp \
    --namespace "$NAMESPACE" \
    --set canary.enabled=false \
    --wait --timeout 5m
  exit 1
fi

# Phase 2: Increase to 50%
echo "Phase 2: Increasing canary to 50%..."
helm upgrade --install "$RELEASE" ./deploy/helm/myapp \
  --namespace "$NAMESPACE" \
  --set image.tag="$IMAGE_TAG" \
  --set canary.enabled=true \
  --set canary.weight=50 \
  --wait --timeout 5m

sleep 300  # Observe for 5 minutes

# Check P99 latency
P99_LATENCY=$(curl -s "$METRICS_URL/api/v1/query?query=histogram_quantile(0.99,rate(http_request_duration_seconds_bucket{track='canary'}[5m]))" | jq -r '.data.result[0].value[1] // "0"')
echo "Canary P99 latency: ${P99_LATENCY}s"
if (( $(echo "$P99_LATENCY > 2.0" | bc -l) )); then
  echo "FAIL: P99 latency too high, rolling back canary"
  helm upgrade --install "$RELEASE" ./deploy/helm/myapp \
    --namespace "$NAMESPACE" \
    --set canary.enabled=false \
    --wait --timeout 5m
  exit 1
fi

# Phase 3: Full promotion
echo "Phase 3: Promoting canary to 100%..."
helm upgrade --install "$RELEASE" ./deploy/helm/myapp \
  --namespace "$NAMESPACE" \
  --set image.tag="$IMAGE_TAG" \
  --set canary.enabled=false \
  --set replicas=5 \
  --wait --timeout 10m

echo "=== Canary deployment complete ==="
```

### 12.4 Blue-Green Deployments

```go
// internal/deploy/bluegreen.go
package deploy

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"
)

// BlueGreenDeployer manages blue-green deployments.
type BlueGreenDeployer struct {
	loadBalancerURL string
	healthCheckURL  string
	httpClient      *http.Client
}

// NewBlueGreenDeployer creates a new deployer.
func NewBlueGreenDeployer(lbURL, healthURL string) *BlueGreenDeployer {
	return &BlueGreenDeployer{
		loadBalancerURL: lbURL,
		healthCheckURL:  healthURL,
		httpClient: &http.Client{
			Timeout: 5 * time.Second,
		},
	}
}

// HealthCheck verifies the target is healthy.
func (d *BlueGreenDeployer) HealthCheck(ctx context.Context, targetURL string) error {
	for i := 0; i < 30; i++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, targetURL+"/healthz", nil)
		if err != nil {
			return fmt.Errorf("creating request: %w", err)
		}

		resp, err := d.httpClient.Do(req)
		if err == nil && resp.StatusCode == http.StatusOK {
			resp.Body.Close()
			log.Printf("Health check passed on attempt %d", i+1)
			return nil
		}
		if resp != nil {
			resp.Body.Close()
		}

		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(2 * time.Second):
		}
	}

	return fmt.Errorf("health check failed after 30 attempts")
}

// SwitchTraffic switches load balancer from one color to another.
// In practice this would interact with your LB API (ALB, nginx, etc.).
func (d *BlueGreenDeployer) SwitchTraffic(ctx context.Context, fromColor, toColor string) error {
	log.Printf("Switching traffic from %s to %s", fromColor, toColor)

	// Example: Update nginx upstream or ALB target group
	// This is simplified; real implementation depends on your infrastructure.
	req, err := http.NewRequestWithContext(ctx, http.MethodPut,
		fmt.Sprintf("%s/api/switch?from=%s&to=%s", d.loadBalancerURL, fromColor, toColor), nil)
	if err != nil {
		return fmt.Errorf("creating switch request: %w", err)
	}

	resp, err := d.httpClient.Do(req)
	if err != nil {
		return fmt.Errorf("switching traffic: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("switch returned status %d", resp.StatusCode)
	}

	log.Printf("Traffic switched from %s to %s", fromColor, toColor)
	return nil
}
```

#### Blue-Green Deployment Strategy Diagram

```
                    Load Balancer
                         │
                    ┌────┴────┐
                    │         │
              ┌─────▼───┐ ┌──▼──────┐
              │  Blue    │ │  Green  │
              │  (v1.0)  │ │  (v1.1) │
              │  ACTIVE  │ │  IDLE   │
              └─────────┘ └─────────┘

Step 1: Deploy v1.1 to Green (idle)
Step 2: Run health checks on Green
Step 3: Switch load balancer to Green
Step 4: Green becomes ACTIVE, Blue becomes IDLE
Step 5: If problems, switch back to Blue (instant rollback)
```

---

## 13. Complete Pipeline: Commit to Production

This section ties everything together into a production-grade pipeline.

### 13.1 Repository Structure

```
myapp/
├── .github/
│   └── workflows/
│       ├── ci.yml              # CI for every push and PR
│       ├── release.yml         # Release on tag push
│       └── nightly.yml         # Nightly security scans
├── .golangci.yml               # Linter configuration
├── .goreleaser.yml             # Release configuration
├── Dockerfile                  # Production Docker image
├── Makefile                    # Build automation
├── docker-compose.test.yml     # Integration test infrastructure
├── cmd/
│   └── server/
│       └── main.go             # Application entry point
├── internal/
│   ├── config/                 # Configuration loading
│   ├── database/
│   │   └── migrations/         # SQL migration files
│   ├── features/               # Feature flags
│   ├── health/                 # Health checks
│   ├── metrics/                # Prometheus metrics
│   ├── store/                  # Data access layer
│   └── testutil/               # Test helpers
├── deploy/
│   └── helm/
│       └── myapp/              # Helm chart
├── scripts/
│   ├── bump-version.sh
│   ├── canary-promote.sh
│   ├── check-coverage.sh
│   └── integration-test.sh
├── go.mod
└── go.sum
```

### 13.2 The Complete CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write
  security-events: write

env:
  GO_VERSION: '1.23'
  GOLANGCI_LINT_VERSION: 'v1.62'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 1: Quick Validation (fail fast)
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  validate:
    name: Validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Check formatting
        run: test -z "$(gofmt -l .)" || (gofmt -l . && exit 1)

      - name: Check go mod tidy
        run: |
          go mod tidy
          git diff --exit-code go.mod go.sum

      - name: Run go vet
        run: go vet ./...

      - name: Verify build
        run: go build ./...

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 2: Testing (parallel with linting)
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: [validate]
    strategy:
      fail-fast: false
      matrix:
        go-version: ['1.22', '1.23']
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true

      - name: Run tests with race detector and coverage
        run: |
          go test \
            -race \
            -count=1 \
            -coverprofile=coverage.out \
            -covermode=atomic \
            -timeout=10m \
            ./...

      - name: Check coverage threshold
        if: matrix.go-version == '1.23'
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          echo "Total coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80.0" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below 80% threshold"
            exit 1
          fi

      - name: Upload coverage
        if: matrix.go-version == '1.23'
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out
          token: ${{ secrets.CODECOV_TOKEN }}

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 2: Linting (parallel with testing)
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  lint:
    name: Lint
    runs-on: ubuntu-latest
    needs: [validate]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: ${{ env.GOLANGCI_LINT_VERSION }}
          args: --timeout=5m

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 2: Security (parallel with testing and linting)
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  security:
    name: Security
    runs-on: ubuntu-latest
    needs: [validate]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: Run gosec
        uses: securego/gosec@master
        with:
          args: '-no-fail -fmt sarif -out gosec.sarif ./...'

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: gosec.sarif

      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 3: Integration Tests
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [test, lint]
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run database migrations
        env:
          DATABASE_URL: "postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable"
        run: |
          go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
          migrate -database "$DATABASE_URL" -path internal/database/migrations up

      - name: Run integration tests
        env:
          DATABASE_URL: "postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable"
          REDIS_URL: "redis://localhost:6379"
        run: |
          go test \
            -tags=integration \
            -race \
            -count=1 \
            -v \
            -timeout=15m \
            ./...

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 4: Build and Push Docker Image
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  build:
    name: Build & Push
    runs-on: ubuntu-latest
    needs: [test, lint, security, integration-test]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
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
            type=sha,prefix=
            type=ref,event=branch
            type=raw,value=latest

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.sha }}
            COMMIT=${{ github.sha }}
            BUILD_TIME=${{ github.event.head_commit.timestamp }}
          platforms: linux/amd64,linux/arm64

      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL'
          ignore-unfixed: true

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 5: Deploy to Staging
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build]
    environment:
      name: staging
      url: https://staging.myapp.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v4
        with:
          version: 'v3.16.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Run database migrations
        run: |
          kubectl run migrate-${{ github.sha }} \
            --namespace myapp-staging \
            --image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --restart=Never \
            --command -- /server migrate up
          kubectl wait --for=condition=complete \
            --namespace myapp-staging \
            job/migrate-${{ github.sha }} \
            --timeout=120s

      - name: Deploy to staging
        run: |
          helm upgrade --install myapp ./deploy/helm/myapp \
            --namespace myapp-staging \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --set replicas=2 \
            --wait \
            --timeout 5m

      - name: Smoke test
        run: |
          sleep 10
          ENDPOINT="https://staging.myapp.example.com"

          # Health check
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$ENDPOINT/healthz")
          if [ "$STATUS" != "200" ]; then
            echo "::error::Health check failed with status $STATUS"
            exit 1
          fi

          # Version check
          DEPLOYED_COMMIT=$(curl -s "$ENDPOINT/healthz" | jq -r '.commit')
          EXPECTED_COMMIT="${{ github.sha }}"
          if [ "$DEPLOYED_COMMIT" != "${EXPECTED_COMMIT:0:7}" ]; then
            echo "::error::Version mismatch: deployed=$DEPLOYED_COMMIT expected=${EXPECTED_COMMIT:0:7}"
            exit 1
          fi

          echo "Staging deployment verified: commit=$DEPLOYED_COMMIT"

  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Stage 6: Deploy to Production (Canary)
  # ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    environment:
      name: production
      url: https://myapp.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v4
        with:
          version: 'v3.16.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config

      - name: Run database migrations
        run: |
          kubectl run migrate-${{ github.sha }} \
            --namespace myapp-production \
            --image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --restart=Never \
            --command -- /server migrate up
          kubectl wait --for=condition=complete \
            --namespace myapp-production \
            job/migrate-${{ github.sha }} \
            --timeout=120s

      - name: Deploy canary (10% traffic)
        run: |
          helm upgrade --install myapp ./deploy/helm/myapp \
            --namespace myapp-production \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --set canary.enabled=true \
            --set canary.weight=10 \
            --wait \
            --timeout 5m

      - name: Monitor canary (3 minutes)
        run: |
          echo "Monitoring canary for 3 minutes..."
          for i in $(seq 1 18); do
            # Check error rate via Prometheus
            ERROR_RATE=$(curl -s "${{ secrets.PROMETHEUS_URL }}/api/v1/query?query=rate(http_requests_total{status=~'5..',track='canary',namespace='myapp-production'}[1m])" | jq -r '.data.result[0].value[1] // "0"')

            echo "Check $i/18: error_rate=${ERROR_RATE}"

            if (( $(echo "${ERROR_RATE} > 0.05" | bc -l) )); then
              echo "::error::Canary error rate too high (${ERROR_RATE}), rolling back"
              helm upgrade --install myapp ./deploy/helm/myapp \
                --namespace myapp-production \
                --set canary.enabled=false \
                --reuse-values \
                --wait --timeout 5m
              exit 1
            fi

            sleep 10
          done

      - name: Promote to full deployment
        run: |
          helm upgrade --install myapp ./deploy/helm/myapp \
            --namespace myapp-production \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --set canary.enabled=false \
            --set replicas=5 \
            --wait \
            --timeout 10m

      - name: Verify production
        run: |
          ENDPOINT="https://myapp.example.com"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$ENDPOINT/healthz")
          if [ "$STATUS" != "200" ]; then
            echo "::error::Production health check failed"
            exit 1
          fi
          echo "Production deployment successful!"

      - name: Notify on success
        if: success()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-Type: application/json' \
            -d "{\"text\":\"Deployed myapp ${{ github.sha }} to production\"}"

      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-Type: application/json' \
            -d "{\"text\":\"FAILED: myapp deployment ${{ github.sha }} to production. Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}"
```

### 13.3 The Complete Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  packages: write
  id-token: write

jobs:
  # Run the full CI first
  ci:
    uses: ./.github/workflows/ci.yml
    secrets: inherit

  release:
    name: Release
    runs-on: ubuntu-latest
    needs: [ci]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: '~> v2'
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}

      - name: Notify release
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-Type: application/json' \
            -d "{\"text\":\"Released myapp ${{ github.ref_name }}! https://github.com/${{ github.repository }}/releases/tag/${{ github.ref_name }}\"}"
```

### 13.4 Nightly Security Scan

```yaml
# .github/workflows/nightly.yml
name: Nightly Security Scan

on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC daily
  workflow_dispatch:

permissions:
  contents: read
  security-events: write

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Update dependencies
        run: |
          go get -u ./...
          go mod tidy

      - name: Check for new vulnerabilities
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

      - name: Run comprehensive gosec scan
        uses: securego/gosec@master
        with:
          args: '-fmt sarif -out gosec.sarif ./...'

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: gosec.sarif

      - name: Scan all container images
        run: |
          IMAGES=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep myapp)
          for img in $IMAGES; do
            echo "Scanning $img..."
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
              aquasec/trivy:latest image --severity CRITICAL,HIGH "$img"
          done

      - name: Notify on vulnerabilities
        if: failure()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK_URL }}" \
            -H 'Content-Type: application/json' \
            -d '{"text":"Security vulnerabilities found in myapp! Check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}'
```

### 13.5 Pipeline Flow Visualization

```
              ┌──────────────────────────────────────────────────────────────────┐
              │                         COMMIT / PR                              │
              └──────────────────────────┬───────────────────────────────────────┘
                                         │
                                         ▼
              ┌──────────────────────────────────────────────────────────────────┐
              │                      Stage 1: VALIDATE                           │
              │  gofmt check  │  go mod tidy  │  go vet  │  go build            │
              │  (~30 seconds)                                                   │
              └──────────────────────────┬───────────────────────────────────────┘
                                         │
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
              ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
              │ Stage 2: TEST │ │ Stage 2: LINT │ │Stage 2: SECUR │
              │               │ │               │ │               │
              │ go test -race │ │ golangci-lint │ │ govulncheck   │
              │ coverage 80%  │ │               │ │ gosec         │
              │ matrix build  │ │               │ │ trivy         │
              │ (~3 minutes)  │ │ (~2 minutes)  │ │ (~2 minutes)  │
              └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
                      │                 │                 │
                      └────────┬────────┘                 │
                               ▼                          │
              ┌──────────────────────────────┐            │
              │  Stage 3: INTEGRATION TEST   │            │
              │                              │            │
              │  postgres + redis services   │            │
              │  database migrations         │            │
              │  testcontainers              │            │
              │  (~5 minutes)                │            │
              └──────────────┬───────────────┘            │
                             │                            │
                             └──────────┬─────────────────┘
                                        ▼
              ┌──────────────────────────────────────────────────────────────────┐
              │                   Stage 4: BUILD & PUSH                          │
              │  Docker multi-platform build  │  Image scan  │  Push to GHCR    │
              │  (~5 minutes)                                                    │
              └──────────────────────────┬───────────────────────────────────────┘
                                         │
                                         ▼
              ┌──────────────────────────────────────────────────────────────────┐
              │                   Stage 5: DEPLOY STAGING                        │
              │  Database migration  │  Helm deploy  │  Smoke test              │
              │  (~3 minutes)                                                    │
              └──────────────────────────┬───────────────────────────────────────┘
                                         │
                                         ▼  (manual approval gate)
              ┌──────────────────────────────────────────────────────────────────┐
              │                  Stage 6: DEPLOY PRODUCTION                      │
              │  Database migration  │  Canary 10%  │  Monitor  │  Promote 100% │
              │  (~15 minutes with monitoring)                                   │
              └──────────────────────────────────────────────────────────────────┘
```

### 13.6 Summary of Best Practices

| Area | Best Practice |
|------|---------------|
| **Speed** | Fail fast with cheap checks first (fmt, vet) |
| **Caching** | Cache Go modules and build cache |
| **Testing** | Always use `-race`, enforce coverage thresholds |
| **Linting** | Use golangci-lint with a committed config file |
| **Security** | Run govulncheck, gosec, and trivy on every PR |
| **Docker** | Multi-stage builds, scratch/distroless base images |
| **Versioning** | Inject version via `-ldflags`, use SemVer |
| **Releases** | goreleaser for cross-compilation, changelogs, Homebrew |
| **Secrets** | OIDC where possible, environment-scoped secrets |
| **Migrations** | Test up+down in CI, use advisory locks in production |
| **Integration** | Testcontainers or docker-compose services |
| **Deployments** | Canary or blue-green with automated rollback |
| **Monitoring** | Health endpoints, Prometheus metrics, error rate alerts |
| **Reproducibility** | Pin all tool versions, use `-trimpath` |

> **Final tip**: Run `make ci` locally before pushing. The closer your local checks match
> CI, the fewer broken builds you'll see. Invest in a good Makefile -- it pays for itself
> on the first day.
