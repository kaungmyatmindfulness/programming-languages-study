# Chapter 23: Configuration & Environment Management

This chapter marks the beginning of a new phase in this study guide. Chapters 1 through 22 covered Go fundamentals -- from variables and control flow through generics, concurrency, microservices, and system programming. Starting now, we shift focus to **real-world patterns** drawn from production Go codebases. Every example in this chapter is inspired by patterns you would find in a production API project like "krafty-core" -- a Go REST API with JWT authentication, PostgreSQL, CORS, cookie-based sessions, and structured logging.

Configuration management is the first thing that bites you in production. Your application works perfectly on your laptop with hardcoded values. Then you deploy it and everything breaks because the database URL is wrong, the JWT secret is missing, CORS blocks your frontend, and cookies are not secure. This chapter teaches you how to handle configuration the way production Go applications do -- cleanly, safely, and with confidence that your app will tell you exactly what is wrong before it starts serving traffic.

## Prerequisites

You should be comfortable with:
- Go structs, methods, and interfaces (Chapters 4, 6)
- Error handling patterns (Chapter 8)
- Packages and modules (Chapter 9)
- Testing in Go (Chapter 14)
- Basic HTTP server concepts (Chapter 12)
- Environment variables (basic understanding)

If you have used `dotenv` in Node.js or `process.env` in Express, the comparisons throughout this chapter will be especially valuable.

---

## Table of Contents

1. [Why Configuration Matters](#1-why-configuration-matters)
2. [Environment Variables with os.Getenv](#2-environment-variables-with-osgetenv)
3. [Loading .env Files with godotenv](#3-loading-env-files-with-godotenv)
4. [Building a Config Struct](#4-building-a-config-struct)
5. [Config Validation](#5-config-validation)
6. [Sensible Defaults](#6-sensible-defaults)
7. [Duration Parsing](#7-duration-parsing)
8. [Comma-Separated Values](#8-comma-separated-values)
9. [Database Pool Configuration](#9-database-pool-configuration)
10. [Configuration with Functional Options](#10-configuration-with-functional-options)
11. [Secrets Management](#11-secrets-management)
12. [Configuration Testing](#12-configuration-testing)
13. [Real-World Example: Complete Config Package](#13-real-world-example-complete-config-package)
14. [Key Takeaways](#14-key-takeaways)
15. [Practice Exercises](#15-practice-exercises)

---

## 1. Why Configuration Matters

### The Problem

Every non-trivial application has values that change between environments: database connection strings, API keys, feature flags, timeouts, port numbers. The question is not *whether* your application needs configuration -- it is *how* you manage it.

Here is what happens when you get configuration wrong:

```
Development machine:   PORT=3000, DB=localhost:5432, CORS=*
Staging server:        PORT=8080, DB=staging-db:5432, CORS=https://staging.app.com
Production server:     PORT=8080, DB=prod-db:5432, CORS=https://app.com

What if you deploy to production with CORS=*?
    → Any website can make authenticated requests to your API
    → Security breach

What if you deploy with the staging database URL?
    → Production users see test data, or worse, staging users see production data
    → Data breach

What if you forget to set the JWT signing key?
    → Application starts with an empty key
    → Every JWT is signed with "" -- trivially forgeable
    → Complete authentication bypass
```

Configuration errors are some of the most common causes of production incidents. They are also some of the most preventable.

### The Twelve-Factor App

The [Twelve-Factor App](https://12factor.net) methodology, written by Heroku co-founder Adam Wiggins, defines best practices for building modern web applications. Factor III is about configuration:

> **III. Config: Store config in the environment**
>
> An app's config is everything that is likely to vary between deploys (staging, production, developer environments, etc). This includes:
> - Resource handles to the database, caching systems, and other backing services
> - Credentials to external services (Amazon S3, Twitter, etc)
> - Per-deploy values such as the canonical hostname for the deploy

The key insight: **configuration belongs in the environment, not in the code.** Your code should be identical across all environments. The only difference is the environment variables.

```
┌──────────────────────────────────────────────────────────┐
│                    Your Go Binary                         │
│                  (identical everywhere)                   │
│                                                          │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐              │
│   │ Config  │   │ Config  │   │ Config  │              │
│   │ (dev)   │   │ (stage) │   │ (prod)  │              │
│   │         │   │         │   │         │              │
│   │ PORT=   │   │ PORT=   │   │ PORT=   │              │
│   │ 3000    │   │ 8080    │   │ 8080    │              │
│   │         │   │         │   │         │              │
│   │ DB=     │   │ DB=     │   │ DB=     │              │
│   │ local   │   │ stage   │   │ prod    │              │
│   │         │   │         │   │         │              │
│   │ JWT=    │   │ JWT=    │   │ JWT=    │              │
│   │ dev-key │   │ stg-key │   │ prod-key│              │
│   └─────────┘   └─────────┘   └─────────┘              │
│                                                          │
│   All from environment variables, not from code          │
└──────────────────────────────────────────────────────────┘
```

### WHY Not Config Files?

You might wonder: "Why not just use a `config.json` or `config.yaml` per environment?" There are several reasons:

1. **Config files get committed to version control.** Secrets in Git are a security incident waiting to happen. Even if you `.gitignore` them, someone eventually commits a config file with production credentials.

2. **Config files create environment coupling.** You need a different file for dev, staging, production, CI, and every developer's machine. Managing these files becomes its own problem.

3. **Environment variables are universal.** Every operating system, every container runtime, every CI/CD system, every cloud provider supports environment variables. They are the lowest common denominator.

4. **Environment variables work with secrets management.** Tools like HashiCorp Vault, AWS Secrets Manager, and Kubernetes Secrets all inject secrets as environment variables. Config files require additional tooling.

### Node.js Comparison

In the Node.js/Express ecosystem, the standard approach is:

```javascript
// Node.js: process.env + dotenv
require('dotenv').config();

const port = process.env.PORT || 3000;
const dbUrl = process.env.DATABASE_URL;
const jwtSecret = process.env.JWT_SECRET;

// Problems:
// 1. No type safety -- everything is a string
// 2. No validation -- if JWT_SECRET is missing, you find out when signing fails
// 3. process.env is globally mutable -- any code can change it
// 4. No centralized schema of what config your app needs
```

Go does not have a built-in `process.env` equivalent that is globally accessible everywhere. Instead, Go encourages you to build a `Config` struct, load it once at startup, validate it, and pass it explicitly to the components that need it. This is more work upfront, but it eliminates entire categories of bugs.

---

## 2. Environment Variables with os.Getenv

### The Basics

Go's standard library provides environment variable access through the `os` package:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// os.Getenv returns the value of an environment variable.
	// If the variable is not set, it returns an empty string "".
	port := os.Getenv("PORT")
	fmt.Printf("PORT = %q\n", port)

	// os.LookupEnv returns the value AND a boolean indicating
	// whether the variable was set at all. This lets you distinguish
	// between "not set" and "set to empty string".
	dbURL, exists := os.LookupEnv("DATABASE_URL")
	if !exists {
		fmt.Println("DATABASE_URL is not set")
	} else if dbURL == "" {
		fmt.Println("DATABASE_URL is set but empty")
	} else {
		fmt.Printf("DATABASE_URL = %s\n", dbURL)
	}

	// os.Setenv sets an environment variable for the current process.
	// This does NOT affect the parent shell or other processes.
	os.Setenv("APP_MODE", "development")
	fmt.Printf("APP_MODE = %s\n", os.Getenv("APP_MODE"))

	// os.Unsetenv removes an environment variable.
	os.Unsetenv("APP_MODE")
	fmt.Printf("APP_MODE after unset = %q\n", os.Getenv("APP_MODE"))

	// os.Environ returns all environment variables as a slice of "KEY=VALUE" strings.
	// Useful for debugging, but be careful -- this may include secrets!
	for _, env := range os.Environ() {
		// Don't print these in production logs!
		fmt.Println(env)
	}
}
```

### The getEnv Helper Pattern

The most common pattern in Go applications is a helper function that returns a default value when an environment variable is not set:

```go
package main

import (
	"fmt"
	"os"
	"strconv"
	"time"
)

// getEnv returns the value of an environment variable, or a default value
// if the variable is not set. This is the single most useful config helper.
func getEnv(key, defaultValue string) string {
	if value, exists := os.LookupEnv(key); exists {
		return value
	}
	return defaultValue
}

// getEnvInt returns an environment variable as an integer.
// Returns the default if the variable is not set or cannot be parsed.
func getEnvInt(key string, defaultValue int) int {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := strconv.Atoi(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

// getEnvBool returns an environment variable as a boolean.
// Accepts "true", "1", "yes" (case-insensitive) as true.
// Returns the default for anything else or if not set.
func getEnvBool(key string, defaultValue bool) bool {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := strconv.ParseBool(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

// getEnvDuration returns an environment variable as a time.Duration.
// Expects Go duration strings like "15m", "1h30m", "500ms".
// Returns the default if not set or cannot be parsed.
func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := time.ParseDuration(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func main() {
	port := getEnv("PORT", "8080")
	debug := getEnvBool("DEBUG", false)
	maxConns := getEnvInt("DB_MAX_OPEN_CONNS", 25)
	readTimeout := getEnvDuration("READ_TIMEOUT", 15*time.Second)

	fmt.Printf("Port: %s\n", port)
	fmt.Printf("Debug: %v\n", debug)
	fmt.Printf("Max DB Connections: %d\n", maxConns)
	fmt.Printf("Read Timeout: %s\n", readTimeout)
}
```

### WHY LookupEnv vs Getenv

This is a subtle but important distinction:

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	// Scenario 1: Variable is not set at all
	os.Unsetenv("MY_VAR")

	val1 := os.Getenv("MY_VAR")
	fmt.Printf("Getenv: %q\n", val1) // ""

	val2, exists := os.LookupEnv("MY_VAR")
	fmt.Printf("LookupEnv: %q, exists: %v\n", val2, exists) // "", false

	// Scenario 2: Variable is set to empty string
	os.Setenv("MY_VAR", "")

	val3 := os.Getenv("MY_VAR")
	fmt.Printf("Getenv: %q\n", val3) // ""

	val4, exists := os.LookupEnv("MY_VAR")
	fmt.Printf("LookupEnv: %q, exists: %v\n", val4, exists) // "", true

	// With os.Getenv, both scenarios produce the same result: "".
	// With os.LookupEnv, you can tell them apart.
	//
	// WHY this matters:
	// Some systems set variables to empty to explicitly disable a feature.
	// For example, setting CORS_ORIGINS="" might mean "disable CORS"
	// while not setting it means "use the default".
}
```

### Node.js Comparison

```javascript
// Node.js equivalent of getEnv with default
const port = process.env.PORT || '3000';

// Problem: || treats "" and "0" as falsy!
const debug = process.env.DEBUG || 'false';
// If DEBUG="" is intentionally set to empty, this returns 'false'

// Better approach in Node.js (modern):
const maxConns = process.env.DB_MAX_OPEN_CONNS ?? '25';
// ?? only falls through on null/undefined, not on ""

// Go's os.LookupEnv is analogous to checking undefined vs empty:
// const [value, exists] = lookupEnv("KEY")
// There is no direct equivalent in Node.js --
// process.env.KEY is undefined if not set, "" if set to empty.
```

---

## 3. Loading .env Files with godotenv

### The Development Workflow Problem

In production, environment variables are set by the deployment system (Kubernetes, Docker, systemd, etc.). But in development, you need a way to set them without polluting your shell profile. The solution is a `.env` file.

### Using godotenv

The `github.com/joho/godotenv` package is the Go equivalent of Node.js's `dotenv`:

```go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/joho/godotenv"
)

func main() {
	// Load .env file. This reads the file and sets the environment variables.
	// It does NOT override variables that are already set in the environment.
	// This is important -- system-level env vars take precedence over .env files.
	err := godotenv.Load()
	if err != nil {
		// In production, there is no .env file -- that is expected.
		// Only log a warning, do not crash.
		log.Printf("Warning: .env file not found, using environment variables")
	}

	// Now os.Getenv works with both real env vars and .env file values.
	port := os.Getenv("PORT")
	dbURL := os.Getenv("DATABASE_URL")

	fmt.Printf("PORT=%s\n", port)
	fmt.Printf("DATABASE_URL=%s\n", dbURL)
}
```

### The .env File Format

```bash
# .env
# Lines starting with # are comments
# No spaces around the = sign
# No quoting required for simple values
# Blank lines are ignored

PORT=8080
DATABASE_URL=postgres://user:password@localhost:5432/mydb?sslmode=disable
JWT_SIGNING_KEY=a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=168h
CORS_ORIGINS=http://localhost:4200,http://localhost:3000
LOG_LEVEL=debug
COOKIE_DOMAIN=localhost
COOKIE_SECURE=false
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=5m
```

### The .env.example Pattern

**Never commit `.env` to version control.** Instead, commit a `.env.example` file that documents every variable your application needs, with safe placeholder values:

```bash
# .env.example
# Copy this to .env and fill in real values:
#   cp .env.example .env

# Server
PORT=8080

# Database
DATABASE_URL=postgres://user:password@localhost:5432/dbname?sslmode=disable

# JWT Authentication
# Generate a 32-byte hex key: openssl rand -hex 32
JWT_SIGNING_KEY=<generate-with-openssl-rand-hex-32>
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=168h

# CORS (comma-separated origins, no trailing slashes)
CORS_ORIGINS=http://localhost:4200

# Logging
LOG_LEVEL=debug

# Cookies
COOKIE_DOMAIN=localhost
COOKIE_SECURE=false

# Database Pool
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=5m
```

Your `.gitignore` should include:

```
# .gitignore
.env
.env.local
.env.*.local
# But NOT .env.example -- that is documentation
```

### Loading Multiple .env Files

Sometimes you want environment-specific defaults:

```go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/joho/godotenv"
)

func main() {
	env := os.Getenv("APP_ENV")
	if env == "" {
		env = "development"
	}

	// Load environment-specific file first, then base .env.
	// godotenv.Load does NOT overwrite existing variables,
	// so load files from most specific to least specific.
	//
	// Priority (highest to lowest):
	// 1. Actual environment variables (set by OS/container/CI)
	// 2. .env.{environment}.local -- local overrides for this env
	// 3. .env.{environment}       -- env-specific defaults
	// 4. .env                     -- base defaults

	files := []string{
		fmt.Sprintf(".env.%s.local", env),
		fmt.Sprintf(".env.%s", env),
		".env",
	}

	for _, file := range files {
		if err := godotenv.Load(file); err != nil {
			// File not found is fine -- not every file needs to exist
			log.Printf("Info: %s not found, skipping", file)
		}
	}

	fmt.Printf("Running in %s mode\n", env)
	fmt.Printf("PORT=%s\n", os.Getenv("PORT"))
}
```

### godotenv.Overload vs godotenv.Load

```go
package main

import (
	"fmt"
	"os"

	"github.com/joho/godotenv"
)

func main() {
	// Set a real environment variable
	os.Setenv("PORT", "9090")

	// godotenv.Load does NOT overwrite.
	// If PORT is already set to "9090", the .env file's PORT value is ignored.
	_ = godotenv.Load() // .env has PORT=8080
	fmt.Println(os.Getenv("PORT")) // "9090" -- real env var wins

	// godotenv.Overload DOES overwrite.
	// Use this only in testing or very specific scenarios.
	_ = godotenv.Overload() // .env has PORT=8080
	fmt.Println(os.Getenv("PORT")) // "8080" -- .env file wins

	// In production, you almost always want Load, not Overload.
	// The system environment should be the source of truth.
}
```

### Node.js Comparison

```javascript
// Node.js with dotenv
require('dotenv').config();
// Equivalent to godotenv.Load() -- does not overwrite existing env vars

// Node.js with dotenv + override
require('dotenv').config({ override: true });
// Equivalent to godotenv.Overload()

// Node.js with multiple env files
const dotenv = require('dotenv');
dotenv.config({ path: `.env.${process.env.NODE_ENV}.local` });
dotenv.config({ path: `.env.${process.env.NODE_ENV}` });
dotenv.config({ path: '.env' });
```

The pattern is essentially the same. The difference is that in Go, we go further: we load values into a typed struct and validate them before the application starts.

---

## 4. Building a Config Struct

### WHY a Struct

In Node.js, configuration is typically scattered across the codebase:

```javascript
// Scattered across many files:
const port = process.env.PORT || 3000;             // in server.js
const dbUrl = process.env.DATABASE_URL;             // in db.js
const jwtSecret = process.env.JWT_SECRET;           // in auth.js
const corsOrigins = process.env.CORS_ORIGINS;       // in middleware.js
```

This has problems:
1. No single place to see all configuration your app needs.
2. No way to validate all config at startup.
3. Easy to typo environment variable names (`DATABSE_URL` instead of `DATABASE_URL`).
4. No type safety -- everything is a string.

In Go, we define a single `Config` struct that represents the complete configuration:

```go
package config

import (
	"encoding/hex"
	"fmt"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/joho/godotenv"
)

// Config holds all configuration for the application.
// Every field corresponds to an environment variable.
// The struct is the single source of truth for "what does this app need?"
type Config struct {
	// Server
	Port     string
	LogLevel string

	// Database
	DatabaseURL       string
	DBMaxOpenConns    int
	DBMaxIdleConns    int
	DBConnMaxLifetime time.Duration
	DBConnMaxIdleTime time.Duration

	// JWT Authentication
	JWTSigningKey []byte        // Decoded from hex string
	JWTAccessTTL  time.Duration // e.g., 15m
	JWTRefreshTTL time.Duration // e.g., 168h (7 days)

	// CORS
	CORSOrigins []string // Parsed from comma-separated string

	// Cookies
	CookieDomain string
	CookieSecure bool
}

// Load reads configuration from environment variables.
// It loads .env files in development and applies sensible defaults.
func Load() (*Config, error) {
	// Load .env file -- ignore error (file may not exist in production)
	_ = godotenv.Load()

	cfg := &Config{
		// Server
		Port:     getEnv("PORT", "8080"),
		LogLevel: getEnv("LOG_LEVEL", "info"),

		// Database
		DatabaseURL:       os.Getenv("DATABASE_URL"),
		DBMaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
		DBMaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 5),
		DBConnMaxLifetime: getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
		DBConnMaxIdleTime: getEnvDuration("DB_CONN_MAX_IDLE_TIME", 1*time.Minute),

		// JWT
		JWTAccessTTL:  getEnvDuration("JWT_ACCESS_TTL", 15*time.Minute),
		JWTRefreshTTL: getEnvDuration("JWT_REFRESH_TTL", 168*time.Hour),

		// CORS
		CORSOrigins: parseCSV(os.Getenv("CORS_ORIGINS")),

		// Cookies
		CookieDomain: getEnv("COOKIE_DOMAIN", "localhost"),
		CookieSecure: getEnvBool("COOKIE_SECURE", false),
	}

	// Parse the JWT signing key from hex
	jwtKeyHex := os.Getenv("JWT_SIGNING_KEY")
	if jwtKeyHex != "" {
		key, err := hex.DecodeString(jwtKeyHex)
		if err != nil {
			return nil, fmt.Errorf("JWT_SIGNING_KEY is not valid hex: %w", err)
		}
		cfg.JWTSigningKey = key
	}

	// Validate the configuration
	if err := cfg.validate(); err != nil {
		return nil, err
	}

	return cfg, nil
}

// Helper functions

func getEnv(key, defaultValue string) string {
	if value, exists := os.LookupEnv(key); exists {
		return value
	}
	return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := strconv.Atoi(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func getEnvBool(key string, defaultValue bool) bool {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := strconv.ParseBool(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := time.ParseDuration(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func parseCSV(value string) []string {
	if value == "" {
		return nil
	}
	parts := strings.Split(value, ",")
	result := make([]string, 0, len(parts))
	for _, part := range parts {
		trimmed := strings.TrimSpace(part)
		if trimmed != "" {
			result = append(result, trimmed)
		}
	}
	return result
}
```

### Using the Config in main.go

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"yourproject/internal/config"
)

func main() {
	// Load configuration once at startup
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("Failed to load configuration: %v", err)
	}

	// Pass config to components that need it
	fmt.Printf("Starting server on port %s\n", cfg.Port)
	fmt.Printf("Database: %s\n", cfg.DatabaseURL)
	fmt.Printf("JWT Access TTL: %s\n", cfg.JWTAccessTTL)
	fmt.Printf("CORS Origins: %v\n", cfg.CORSOrigins)

	// In a real application, you would pass cfg to your router, database,
	// middleware, etc:
	//
	//   db, err := database.Connect(cfg.DatabaseURL, cfg.DBMaxOpenConns, ...)
	//   router := api.NewRouter(cfg, db)
	//   server := &http.Server{Addr: ":" + cfg.Port, Handler: router}

	server := &http.Server{
		Addr:    ":" + cfg.Port,
		Handler: http.DefaultServeMux,
	}

	log.Fatal(server.ListenAndServe())
}
```

### Nested Config Structs

For larger applications, group related configuration into nested structs:

```go
package config

import (
	"time"
)

// Config is the root configuration struct.
type Config struct {
	Server   ServerConfig
	Database DatabaseConfig
	JWT      JWTConfig
	CORS     CORSConfig
	Cookie   CookieConfig
}

// ServerConfig holds HTTP server configuration.
type ServerConfig struct {
	Port         string
	ReadTimeout  time.Duration
	WriteTimeout time.Duration
	IdleTimeout  time.Duration
	LogLevel     string
}

// DatabaseConfig holds database connection configuration.
type DatabaseConfig struct {
	URL            string
	MaxOpenConns   int
	MaxIdleConns   int
	ConnMaxLife    time.Duration
	ConnMaxIdle    time.Duration
	MigrationsPath string
}

// JWTConfig holds JWT authentication configuration.
type JWTConfig struct {
	SigningKey  []byte
	AccessTTL  time.Duration
	RefreshTTL time.Duration
	Issuer     string
}

// CORSConfig holds CORS configuration.
type CORSConfig struct {
	AllowedOrigins []string
	AllowedMethods []string
	AllowedHeaders []string
	MaxAge         time.Duration
}

// CookieConfig holds cookie configuration.
type CookieConfig struct {
	Domain   string
	Secure   bool
	HTTPOnly bool
	SameSite string
	Path     string
}
```

This approach lets each component receive only the configuration it needs:

```go
package main

import (
	"log"
	"yourproject/internal/config"
	"yourproject/internal/database"
	"yourproject/internal/auth"
	"yourproject/internal/middleware"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("config: %v", err)
	}

	// Each component gets only what it needs
	db, err := database.Connect(cfg.Database)   // Only DatabaseConfig
	authService := auth.New(cfg.JWT)             // Only JWTConfig
	corsMiddleware := middleware.CORS(cfg.CORS)   // Only CORSConfig

	_ = db
	_ = authService
	_ = corsMiddleware
}
```

### WHY Structs Over Maps

You might ask: "Why not just use `map[string]string`?" Consider:

```go
// BAD: map-based config
config := map[string]string{
	"port":         os.Getenv("PORT"),
	"database_url": os.Getenv("DATABASE_URL"),
}

// Problems:
// 1. No compile-time checks: config["databse_url"] compiles but is wrong
// 2. No type safety: config["port"] is always a string
// 3. No documentation: what keys exist? what types? what defaults?
// 4. No IDE autocomplete

// GOOD: struct-based config
type Config struct {
	Port        string        // IDE autocomplete, compile-time checks
	DatabaseURL string        // Typos caught at compile time
	MaxConns    int           // Proper type, not string
	Timeout     time.Duration // Proper type, not string
}

// cfg.Databse_URL → compile error (caught immediately)
// cfg.DatabaseURL → works, IDE autocomplete helps
```

---

## 5. Config Validation

### WHY Validate at Startup

The worst time to discover a configuration error is when a user hits the broken code path. If `JWT_SIGNING_KEY` is missing, you want to know when the application starts, not when the first user tries to log in.

**Fail fast, fail loudly.** An application that refuses to start with a clear error message is infinitely better than one that starts successfully but silently fails on every authentication attempt.

### Collecting ALL Errors at Once

A common mistake is returning on the first validation error. This forces operators to fix one error, restart, find the next error, fix it, restart, and repeat. Instead, collect all errors and report them together:

```go
package config

import (
	"encoding/hex"
	"fmt"
	"net"
	"os"
	"strconv"
	"strings"
	"time"
)

type Config struct {
	Port          string
	DatabaseURL   string
	JWTSigningKey []byte
	JWTAccessTTL  time.Duration
	JWTRefreshTTL time.Duration
	CORSOrigins   []string
	LogLevel      string
	CookieDomain  string
	CookieSecure  bool
}

func (c *Config) validate() error {
	var errs []string

	// --- Required fields ---

	if c.Port == "" {
		errs = append(errs, "PORT is required")
	} else {
		// Validate port is a number in the valid range
		port, err := strconv.Atoi(c.Port)
		if err != nil {
			errs = append(errs, fmt.Sprintf("PORT must be a number, got %q", c.Port))
		} else if port < 1 || port > 65535 {
			errs = append(errs, fmt.Sprintf("PORT must be between 1 and 65535, got %d", port))
		}
	}

	if c.DatabaseURL == "" {
		errs = append(errs, "DATABASE_URL is required")
	}

	// --- JWT validation ---

	if len(c.JWTSigningKey) == 0 {
		errs = append(errs, "JWT_SIGNING_KEY is required")
	} else if len(c.JWTSigningKey) < 32 {
		errs = append(errs,
			fmt.Sprintf("JWT_SIGNING_KEY must be at least 32 bytes (64 hex chars), got %d bytes",
				len(c.JWTSigningKey)))
	}

	if c.JWTAccessTTL <= 0 {
		errs = append(errs, "JWT_ACCESS_TTL must be positive")
	} else if c.JWTAccessTTL > 1*time.Hour {
		errs = append(errs,
			fmt.Sprintf("JWT_ACCESS_TTL should not exceed 1 hour for security, got %s",
				c.JWTAccessTTL))
	}

	if c.JWTRefreshTTL <= 0 {
		errs = append(errs, "JWT_REFRESH_TTL must be positive")
	} else if c.JWTRefreshTTL > 30*24*time.Hour { // 30 days
		errs = append(errs,
			fmt.Sprintf("JWT_REFRESH_TTL should not exceed 30 days, got %s",
				c.JWTRefreshTTL))
	}

	// --- CORS validation ---

	if len(c.CORSOrigins) == 0 {
		errs = append(errs, "CORS_ORIGINS is required (comma-separated list of allowed origins)")
	}

	for _, origin := range c.CORSOrigins {
		if origin == "*" && c.CookieSecure {
			errs = append(errs,
				"CORS_ORIGINS cannot be '*' when COOKIE_SECURE is true (production). "+
					"Specify exact origins instead")
			break
		}
	}

	// --- Log level validation ---

	validLogLevels := map[string]bool{
		"debug": true, "info": true, "warn": true, "error": true,
	}
	if !validLogLevels[c.LogLevel] {
		errs = append(errs,
			fmt.Sprintf("LOG_LEVEL must be one of: debug, info, warn, error -- got %q",
				c.LogLevel))
	}

	// --- Cookie validation ---

	if c.CookieSecure && c.CookieDomain == "localhost" {
		errs = append(errs,
			"COOKIE_DOMAIN cannot be 'localhost' when COOKIE_SECURE is true. "+
				"Set to your production domain")
	}

	// --- Return all errors ---

	if len(errs) > 0 {
		return fmt.Errorf("configuration errors:\n  - %s", strings.Join(errs, "\n  - "))
	}
	return nil
}

// Load creates a Config from environment variables.
func Load() (*Config, error) {
	cfg := &Config{
		Port:         getEnv("PORT", "8080"),
		DatabaseURL:  os.Getenv("DATABASE_URL"),
		JWTAccessTTL: getEnvDuration("JWT_ACCESS_TTL", 15*time.Minute),
		JWTRefreshTTL: getEnvDuration("JWT_REFRESH_TTL", 168*time.Hour),
		CORSOrigins:  parseCSV(os.Getenv("CORS_ORIGINS")),
		LogLevel:     getEnv("LOG_LEVEL", "info"),
		CookieDomain: getEnv("COOKIE_DOMAIN", "localhost"),
		CookieSecure: getEnvBool("COOKIE_SECURE", false),
	}

	// Parse hex-encoded JWT key
	jwtKeyHex := os.Getenv("JWT_SIGNING_KEY")
	if jwtKeyHex != "" {
		key, err := hex.DecodeString(jwtKeyHex)
		if err != nil {
			return nil, fmt.Errorf(
				"JWT_SIGNING_KEY must be a hex-encoded string: %w", err)
		}
		cfg.JWTSigningKey = key
	}

	if err := cfg.validate(); err != nil {
		return nil, err
	}

	return cfg, nil
}

// --- helpers (same as before) ---

func getEnv(key, defaultValue string) string {
	if v, ok := os.LookupEnv(key); ok {
		return v
	}
	return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
	v, ok := os.LookupEnv(key)
	if !ok {
		return defaultValue
	}
	b, err := strconv.ParseBool(v)
	if err != nil {
		return defaultValue
	}
	return b
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
	v, ok := os.LookupEnv(key)
	if !ok {
		return defaultValue
	}
	d, err := time.ParseDuration(v)
	if err != nil {
		return defaultValue
	}
	return d
}

func getEnvInt(key string, defaultValue int) int {
	v, ok := os.LookupEnv(key)
	if !ok {
		return defaultValue
	}
	i, err := strconv.Atoi(v)
	if err != nil {
		return defaultValue
	}
	return i
}

func parseCSV(s string) []string {
	if s == "" {
		return nil
	}
	parts := strings.Split(s, ",")
	out := make([]string, 0, len(parts))
	for _, p := range parts {
		if t := strings.TrimSpace(p); t != "" {
			out = append(out, t)
		}
	}
	return out
}

// Ignore the unused import warnings -- net is used in extended validation.
var _ = net.SplitHostPort
```

### What the Error Output Looks Like

When multiple config values are wrong, the operator sees everything at once:

```
2024/01/15 10:30:00 Failed to load configuration: configuration errors:
  - DATABASE_URL is required
  - JWT_SIGNING_KEY is required
  - CORS_ORIGINS is required (comma-separated list of allowed origins)
  - LOG_LEVEL must be one of: debug, info, warn, error -- got "verbose"
```

This is vastly better than:

```
2024/01/15 10:30:00 Error: DATABASE_URL is required
# Fix it, restart...
2024/01/15 10:31:00 Error: JWT_SIGNING_KEY is required
# Fix it, restart...
2024/01/15 10:32:00 Error: CORS_ORIGINS is required
# Fix it, restart... (3 restarts for 3 errors)
```

### Validation Patterns for Common Types

#### Validating URLs

```go
package main

import (
	"fmt"
	"net/url"
)

func validateDatabaseURL(raw string) error {
	u, err := url.Parse(raw)
	if err != nil {
		return fmt.Errorf("invalid URL: %w", err)
	}

	// Check scheme
	switch u.Scheme {
	case "postgres", "postgresql":
		// OK
	default:
		return fmt.Errorf("unsupported database scheme %q (expected postgres or postgresql)", u.Scheme)
	}

	// Check host is present
	if u.Host == "" {
		return fmt.Errorf("database URL must include a host")
	}

	// Check database name (path after host)
	if u.Path == "" || u.Path == "/" {
		return fmt.Errorf("database URL must include a database name")
	}

	return nil
}

func main() {
	urls := []string{
		"postgres://user:pass@localhost:5432/mydb?sslmode=disable",
		"mysql://user:pass@localhost:3306/mydb",
		"not-a-url",
		"postgres://localhost/",
	}

	for _, u := range urls {
		if err := validateDatabaseURL(u); err != nil {
			fmt.Printf("INVALID: %s\n  Reason: %v\n", u, err)
		} else {
			fmt.Printf("VALID: %s\n", u)
		}
	}
}
```

#### Validating Port Numbers

```go
package main

import (
	"fmt"
	"strconv"
)

func validatePort(portStr string) (int, error) {
	port, err := strconv.Atoi(portStr)
	if err != nil {
		return 0, fmt.Errorf("port must be a number, got %q", portStr)
	}
	if port < 1 || port > 65535 {
		return 0, fmt.Errorf("port must be between 1 and 65535, got %d", port)
	}
	// Ports below 1024 require root privileges on Unix
	if port < 1024 {
		fmt.Printf("Warning: port %d requires root/admin privileges\n", port)
	}
	return port, nil
}

func main() {
	ports := []string{"8080", "0", "99999", "abc", "443", "3000"}
	for _, p := range ports {
		port, err := validatePort(p)
		if err != nil {
			fmt.Printf("INVALID: %s -> %v\n", p, err)
		} else {
			fmt.Printf("VALID: %s -> %d\n", p, port)
		}
	}
}
```

### Node.js Comparison

In Node.js, libraries like `joi`, `zod`, or `envalid` provide config validation:

```javascript
// Node.js with envalid
const { cleanEnv, str, port, num, bool, url } = require('envalid');

const env = cleanEnv(process.env, {
  PORT: port({ default: 8080 }),
  DATABASE_URL: url(),
  JWT_SECRET: str({ desc: 'JWT signing key, at least 32 chars' }),
  CORS_ORIGINS: str({ desc: 'Comma-separated origins' }),
  LOG_LEVEL: str({ choices: ['debug', 'info', 'warn', 'error'], default: 'info' }),
  COOKIE_SECURE: bool({ default: false }),
});

// envalid validates all at once and reports all errors -- same idea as our Go code.
// However, envalid is a runtime library with no compile-time type safety.
```

Go does not need a validation library because:
1. The struct provides compile-time type safety.
2. The `validate()` method provides runtime validation with custom business rules.
3. The code is explicit, readable, and has zero dependencies.

---

## 6. Sensible Defaults

### The Principle of Least Surprise

A developer who clones your project and runs `go run .` should get a working application (or at least a clear error about what is missing). This means every non-secret configuration value should have a sensible default for local development.

### Default Values Strategy

```go
package config

import (
	"os"
	"time"
)

// DefaultConfig returns a Config with sensible defaults for local development.
// Required secrets (like JWT_SIGNING_KEY and DATABASE_URL) are not defaulted
// because defaulting secrets creates a false sense of security.
func DefaultConfig() *Config {
	return &Config{
		Server: ServerConfig{
			Port:         "8080",
			ReadTimeout:  15 * time.Second,
			WriteTimeout: 15 * time.Second,
			IdleTimeout:  60 * time.Second,
			LogLevel:     "debug", // Verbose in development
		},
		Database: DatabaseConfig{
			// No default -- must be explicitly set
			URL:            "",
			MaxOpenConns:   25,
			MaxIdleConns:   5,
			ConnMaxLife:    5 * time.Minute,
			ConnMaxIdle:    1 * time.Minute,
			MigrationsPath: "migrations",
		},
		JWT: JWTConfig{
			// No default signing key -- must be explicitly set
			SigningKey:  nil,
			AccessTTL:  15 * time.Minute,
			RefreshTTL: 168 * time.Hour, // 7 days
			Issuer:     "krafty-core",
		},
		CORS: CORSConfig{
			AllowedOrigins: []string{},
			AllowedMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
			AllowedHeaders: []string{"Content-Type", "Authorization"},
			MaxAge:         12 * time.Hour,
		},
		Cookie: CookieConfig{
			Domain:   "localhost",
			Secure:   false, // HTTP in development
			HTTPOnly: true,
			SameSite: "lax",
			Path:     "/",
		},
	}
}

// Load starts with defaults and overrides with environment variables.
func Load() (*Config, error) {
	cfg := DefaultConfig()

	// Override with environment variables
	if v := os.Getenv("PORT"); v != "" {
		cfg.Server.Port = v
	}
	if v := os.Getenv("DATABASE_URL"); v != "" {
		cfg.Database.URL = v
	}
	if v := os.Getenv("LOG_LEVEL"); v != "" {
		cfg.Server.LogLevel = v
	}
	// ... more overrides ...

	if err := cfg.validate(); err != nil {
		return nil, err
	}

	return cfg, nil
}
```

### What SHOULD and SHOULD NOT Have Defaults

```
┌─────────────────────────┬─────────────┬──────────────────────────────────┐
│ Variable                │ Default?    │ Why                              │
├─────────────────────────┼─────────────┼──────────────────────────────────┤
│ PORT                    │ Yes: "8080" │ Convention, safe                 │
│ LOG_LEVEL               │ Yes: "info" │ Safe default                     │
│ READ_TIMEOUT            │ Yes: 15s    │ Reasonable for most APIs         │
│ DB_MAX_OPEN_CONNS       │ Yes: 25     │ Reasonable for most apps         │
│ JWT_ACCESS_TTL          │ Yes: 15m    │ Security best practice           │
│ COOKIE_HTTPONLY          │ Yes: true   │ Always true for session cookies  │
│ COOKIE_SAMESITE         │ Yes: "lax"  │ Good security default            │
├─────────────────────────┼─────────────┼──────────────────────────────────┤
│ DATABASE_URL            │ No          │ Connection string is a secret    │
│ JWT_SIGNING_KEY         │ No          │ Secret, must be generated        │
│ CORS_ORIGINS            │ No          │ Must be explicit per environment │
│ COOKIE_DOMAIN           │ Maybe       │ "localhost" for dev only         │
│ COOKIE_SECURE           │ Maybe       │ false for dev, MUST be true prod │
└─────────────────────────┴─────────────┴──────────────────────────────────┘
```

### Development vs Production Defaults

Some applications detect the environment and adjust defaults:

```go
package config

import (
	"os"
	"time"
)

// Environment represents the running environment.
type Environment string

const (
	EnvDevelopment Environment = "development"
	EnvStaging     Environment = "staging"
	EnvProduction  Environment = "production"
	EnvTest        Environment = "test"
)

func getEnvironment() Environment {
	switch os.Getenv("APP_ENV") {
	case "production", "prod":
		return EnvProduction
	case "staging", "stage":
		return EnvStaging
	case "test":
		return EnvTest
	default:
		return EnvDevelopment
	}
}

// IsDevelopment returns true if running in development mode.
func (c *Config) IsDevelopment() bool {
	return c.Environment == EnvDevelopment
}

// IsProduction returns true if running in production mode.
func (c *Config) IsProduction() bool {
	return c.Environment == EnvProduction
}

type Config struct {
	Environment Environment
	Server      ServerConfig
	Cookie      CookieConfig
	// ... other fields
}

type ServerConfig struct {
	Port     string
	LogLevel string
}

type CookieConfig struct {
	Secure bool
	Domain string
}

func DefaultConfig() *Config {
	env := getEnvironment()

	cfg := &Config{
		Environment: env,
	}

	switch env {
	case EnvProduction, EnvStaging:
		cfg.Server = ServerConfig{
			Port:     "8080",
			LogLevel: "info", // Less verbose in production
		}
		cfg.Cookie = CookieConfig{
			Secure: true,  // HTTPS only in production
			Domain: "",    // Must be explicitly set
		}
	default: // development, test
		cfg.Server = ServerConfig{
			Port:     "8080",
			LogLevel: "debug", // Verbose in development
		}
		cfg.Cookie = CookieConfig{
			Secure: false,       // HTTP allowed in development
			Domain: "localhost",
		}
	}

	return cfg
}
```

---

## 7. Duration Parsing

### Go's time.Duration

Go has first-class support for durations. The `time.ParseDuration` function parses strings like `"15m"`, `"1h30m"`, `"500ms"`, `"168h"` into `time.Duration` values. This is one of Go's most practical features for configuration.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	// Supported units:
	// "ns"  - nanoseconds
	// "us"  - microseconds (also "µs")
	// "ms"  - milliseconds
	// "s"   - seconds
	// "m"   - minutes
	// "h"   - hours

	examples := []string{
		"15m",      // 15 minutes (JWT access token TTL)
		"168h",     // 168 hours = 7 days (JWT refresh token TTL)
		"5s",       // 5 seconds (database query timeout)
		"500ms",    // 500 milliseconds (HTTP client timeout)
		"1h30m",    // 1 hour 30 minutes (combined)
		"2h45m30s", // 2 hours, 45 minutes, 30 seconds
	}

	for _, s := range examples {
		d, err := time.ParseDuration(s)
		if err != nil {
			fmt.Printf("ERROR: %q -> %v\n", s, err)
			continue
		}
		fmt.Printf("%q -> %s (%.0f seconds)\n", s, d, d.Seconds())
	}

	// Output:
	// "15m"      -> 15m0s (900 seconds)
	// "168h"     -> 168h0m0s (604800 seconds)
	// "5s"       -> 5s (5 seconds)
	// "500ms"    -> 500ms (0 seconds)
	// "1h30m"    -> 1h30m0s (5400 seconds)
	// "2h45m30s" -> 2h45m30s (9930 seconds)

	// NOTE: There is no "d" (days) unit!
	// "7d" would fail. You must use "168h" for 7 days.
	_, err := time.ParseDuration("7d")
	fmt.Printf("Error: %v\n", err)
	// Error: time: unknown unit "d" in duration "7d"
}
```

### Adding Day Support

Since Go does not support "d" for days, many production applications add this:

```go
package main

import (
	"fmt"
	"strconv"
	"strings"
	"time"
)

// parseDuration extends time.ParseDuration with support for "d" (days).
// Examples: "7d", "30d", "7d12h", "1d6h30m"
func parseDuration(s string) (time.Duration, error) {
	// Check if the string contains "d"
	if idx := strings.Index(s, "d"); idx >= 0 {
		// Extract the number before "d"
		dayStr := s[:idx]
		days, err := strconv.Atoi(dayStr)
		if err != nil {
			return 0, fmt.Errorf("invalid day count %q in duration %q", dayStr, s)
		}

		// Convert days to hours and parse the remainder
		remainder := s[idx+1:]
		totalDuration := time.Duration(days) * 24 * time.Hour

		if remainder != "" {
			extra, err := time.ParseDuration(remainder)
			if err != nil {
				return 0, fmt.Errorf("invalid duration %q: %w", s, err)
			}
			totalDuration += extra
		}

		return totalDuration, nil
	}

	// No "d" found, use standard parser
	return time.ParseDuration(s)
}

func main() {
	examples := []string{
		"7d",      // 7 days
		"30d",     // 30 days
		"1d6h30m", // 1 day, 6 hours, 30 minutes
		"15m",     // Standard: 15 minutes
		"168h",    // Standard: 168 hours
	}

	for _, s := range examples {
		d, err := parseDuration(s)
		if err != nil {
			fmt.Printf("ERROR: %q -> %v\n", s, err)
			continue
		}
		fmt.Printf("%q -> %s (%.1f hours)\n", s, d, d.Hours())
	}

	// Output:
	// "7d"      -> 168h0m0s (168.0 hours)
	// "30d"     -> 720h0m0s (720.0 hours)
	// "1d6h30m" -> 30h30m0s (30.5 hours)
	// "15m"     -> 15m0s (0.2 hours)
	// "168h"    -> 168h0m0s (168.0 hours)
}
```

### Duration in Config

```go
package main

import (
	"fmt"
	"os"
	"time"
)

type JWTConfig struct {
	AccessTTL  time.Duration
	RefreshTTL time.Duration
}

func loadJWTConfig() JWTConfig {
	return JWTConfig{
		AccessTTL:  getEnvDuration("JWT_ACCESS_TTL", 15*time.Minute),
		RefreshTTL: getEnvDuration("JWT_REFRESH_TTL", 168*time.Hour), // 7 days
	}
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
	v, ok := os.LookupEnv(key)
	if !ok {
		return defaultValue
	}
	d, err := time.ParseDuration(v)
	if err != nil {
		// In a real application, you might want to return the error.
		// Silently falling back to default is a tradeoff:
		//   Pro: Application does not crash on bad input
		//   Con: Operator does not know their value was ignored
		// A better approach: log a warning and use the default.
		fmt.Printf("Warning: invalid duration for %s=%q, using default %s\n",
			key, v, defaultValue)
		return defaultValue
	}
	return d
}

func main() {
	cfg := loadJWTConfig()
	fmt.Printf("Access token TTL:  %s\n", cfg.AccessTTL)
	fmt.Printf("Refresh token TTL: %s\n", cfg.RefreshTTL)

	// Using the duration:
	accessExpiry := time.Now().Add(cfg.AccessTTL)
	refreshExpiry := time.Now().Add(cfg.RefreshTTL)

	fmt.Printf("Access token expires at:  %s\n", accessExpiry.Format(time.RFC3339))
	fmt.Printf("Refresh token expires at: %s\n", refreshExpiry.Format(time.RFC3339))
}
```

### Node.js Comparison

Node.js has no built-in duration parsing. Libraries like `ms` fill the gap:

```javascript
// Node.js
const ms = require('ms');

const accessTTL = ms(process.env.JWT_ACCESS_TTL || '15m');
// ms('15m') returns 900000 (milliseconds)

// Go's time.ParseDuration is built into the standard library.
// No third-party dependency needed.
// Go returns a proper type (time.Duration), not just a number.
// You can do d.Hours(), d.Minutes(), d.Seconds() on it.
```

---

## 8. Comma-Separated Values

### The Pattern

Many configuration values are naturally lists: CORS origins, allowed headers, feature flags, admin email addresses. The environment variable standard does not support arrays, so the convention is comma-separated strings.

```bash
# .env
CORS_ORIGINS=http://localhost:4200,http://localhost:3000,https://app.example.com
ALLOWED_METHODS=GET,POST,PUT,DELETE
ADMIN_EMAILS=alice@example.com,bob@example.com
```

### Parsing CSV Environment Variables

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

// parseCSV splits a comma-separated string into a slice.
// It trims whitespace from each element and ignores empty elements.
func parseCSV(s string) []string {
	if s == "" {
		return nil
	}

	parts := strings.Split(s, ",")
	result := make([]string, 0, len(parts))

	for _, part := range parts {
		trimmed := strings.TrimSpace(part)
		if trimmed != "" {
			result = append(result, trimmed)
		}
	}

	return result
}

func main() {
	// Test cases demonstrating edge handling
	examples := map[string]string{
		"Normal":            "http://localhost:4200,https://app.example.com",
		"With spaces":       "http://localhost:4200 , https://app.example.com",
		"Trailing comma":    "http://localhost:4200,https://app.example.com,",
		"Empty":             "",
		"Single value":      "http://localhost:4200",
		"Multiple commas":   "a,,b,,c",
		"Only commas":       ",,,",
	}

	for name, value := range examples {
		result := parseCSV(value)
		fmt.Printf("%-20s %q -> %v (len=%d)\n", name, value, result, len(result))
	}

	// In practice:
	os.Setenv("CORS_ORIGINS", "http://localhost:4200, https://app.example.com")
	origins := parseCSV(os.Getenv("CORS_ORIGINS"))
	for _, origin := range origins {
		fmt.Printf("Allowed origin: %s\n", origin)
	}
}
```

### Validating CORS Origins

CORS origins require specific validation:

```go
package main

import (
	"fmt"
	"net/url"
	"strings"
)

// validateCORSOrigins checks that each origin is a valid URL with scheme and host.
func validateCORSOrigins(origins []string, isProduction bool) []string {
	var errs []string

	if len(origins) == 0 {
		errs = append(errs, "at least one CORS origin is required")
		return errs
	}

	for _, origin := range origins {
		if origin == "*" {
			if isProduction {
				errs = append(errs,
					"CORS wildcard '*' is not allowed in production -- "+
						"specify exact origins")
			}
			continue
		}

		u, err := url.Parse(origin)
		if err != nil {
			errs = append(errs, fmt.Sprintf("invalid CORS origin %q: %v", origin, err))
			continue
		}

		if u.Scheme == "" {
			errs = append(errs,
				fmt.Sprintf("CORS origin %q must include scheme (http:// or https://)", origin))
		}

		if u.Host == "" {
			errs = append(errs,
				fmt.Sprintf("CORS origin %q must include a host", origin))
		}

		// Origins should not have paths (except empty or "/")
		if u.Path != "" && u.Path != "/" {
			errs = append(errs,
				fmt.Sprintf("CORS origin %q should not include a path (got %q)", origin, u.Path))
		}

		// Origins should not have trailing slashes
		if strings.HasSuffix(origin, "/") {
			errs = append(errs,
				fmt.Sprintf("CORS origin %q should not have a trailing slash", origin))
		}

		// In production, origins should use HTTPS
		if isProduction && u.Scheme != "https" {
			errs = append(errs,
				fmt.Sprintf("CORS origin %q should use https:// in production", origin))
		}
	}

	return errs
}

func main() {
	// Development -- permissive
	origins := []string{"http://localhost:4200", "http://localhost:3000"}
	errs := validateCORSOrigins(origins, false)
	fmt.Printf("Dev origins: %v\n", errs) // no errors

	// Production -- strict
	prodOrigins := []string{"https://app.example.com", "http://admin.example.com"}
	errs = validateCORSOrigins(prodOrigins, true)
	for _, e := range errs {
		fmt.Printf("Prod error: %s\n", e)
	}
	// Prod error: CORS origin "http://admin.example.com" should use https:// in production

	// Wildcard in production
	errs = validateCORSOrigins([]string{"*"}, true)
	for _, e := range errs {
		fmt.Printf("Error: %s\n", e)
	}
	// Error: CORS wildcard '*' is not allowed in production -- specify exact origins
}
```

---

## 9. Database Pool Configuration

### WHY Pool Configuration Matters

A database connection pool is one of the most critical pieces of your application's infrastructure. Misconfigure it and you will either exhaust database connections (too many open), waste resources (too many idle), or experience connection errors from stale connections.

```
┌────────────────────────────────────────────────────────────┐
│                  Go Application                            │
│                                                            │
│   Incoming HTTP requests                                   │
│   ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐            │
│   │ R │ │ R │ │ R │ │ R │ │ R │ │ R │ │ R │  ...        │
│   └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘            │
│     │     │     │     │     │     │     │              │
│   ┌─┴─────┴─────┴─────┴─────┴─────┴─────┴──┐            │
│   │         Connection Pool                  │            │
│   │                                          │            │
│   │  MaxOpenConns = 25                       │            │
│   │  (Total connections, active + idle)      │            │
│   │                                          │            │
│   │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐    │            │
│   │  │Conn│ │Conn│ │Conn│ │Conn│ │Conn│    │            │
│   │  │ 1  │ │ 2  │ │ 3  │ │ 4  │ │ 5  │    │            │
│   │  └──┬─┘ └──┬─┘ └──┬─┘ └────┘ └────┘    │            │
│   │     │      │      │    idle    idle      │            │
│   │     │active│active│active                │            │
│   │  MaxIdleConns = 5                        │            │
│   │  (Connections kept alive when idle)      │            │
│   └─────┼──────┼──────┼─────────────────────┘            │
│         │      │      │                                    │
└─────────┼──────┼──────┼────────────────────────────────────┘
          │      │      │
     ┌────┴──────┴──────┴────┐
     │     PostgreSQL         │
     │  max_connections=100   │
     └────────────────────────┘
```

### The Four Pool Settings

Go's `database/sql` package provides four configuration knobs:

```go
package main

import (
	"database/sql"
	"fmt"
	"time"

	_ "github.com/lib/pq" // PostgreSQL driver
)

func configurePool(db *sql.DB) {
	// 1. MaxOpenConns: Maximum number of open connections to the database.
	//    This includes both in-use and idle connections.
	//    Default: 0 (unlimited) -- NEVER use the default in production!
	//
	//    Set this to limit how many connections your app opens.
	//    Consider: PostgreSQL default max_connections is 100.
	//    If you have 4 app instances, each should use at most 25.
	db.SetMaxOpenConns(25)

	// 2. MaxIdleConns: Maximum number of idle connections to keep alive.
	//    Idle connections are ready to be reused without the overhead
	//    of establishing a new TCP connection and PostgreSQL handshake.
	//    Default: 2 -- too low for most production apps.
	//
	//    Rule of thumb: Set to 25-50% of MaxOpenConns.
	//    Too low: Connections are constantly opened/closed (slow).
	//    Too high: Idle connections waste database resources.
	db.SetMaxIdleConns(5)

	// 3. ConnMaxLifetime: Maximum amount of time a connection can be reused.
	//    After this duration, the connection is closed and removed from the pool,
	//    even if it is idle. A new connection will be created on demand.
	//    Default: 0 (no limit) -- connections live forever.
	//
	//    WHY set this:
	//    - Database servers, load balancers, and firewalls may close idle
	//      connections after a timeout. If Go tries to use a closed connection,
	//      the query fails.
	//    - Helps distribute connections across multiple database replicas
	//      behind a load balancer.
	//    - Prevents memory leaks from long-lived connections.
	//
	//    Rule of thumb: 5 minutes is a good starting point.
	db.SetConnMaxLifetime(5 * time.Minute)

	// 4. ConnMaxIdleTime: Maximum amount of time a connection can be idle.
	//    If a connection has been idle for this long, it is closed.
	//    Default: 0 (no limit).
	//
	//    This is different from ConnMaxLifetime:
	//    - ConnMaxLifetime: total age of connection
	//    - ConnMaxIdleTime: time since last use
	//
	//    Set shorter than ConnMaxLifetime. Helps return connections
	//    during low-traffic periods.
	db.SetConnMaxIdleTime(1 * time.Minute)
}

func main() {
	db, err := sql.Open("postgres",
		"postgres://user:password@localhost:5432/mydb?sslmode=disable")
	if err != nil {
		panic(err)
	}
	defer db.Close()

	configurePool(db)

	// Verify the connection works
	if err := db.Ping(); err != nil {
		panic(fmt.Sprintf("Cannot reach database: %v", err))
	}

	// Check pool stats
	stats := db.Stats()
	fmt.Printf("Max Open Connections: %d\n", stats.MaxOpenConnections)
	fmt.Printf("Open Connections:     %d\n", stats.OpenConnections)
	fmt.Printf("In Use:               %d\n", stats.InUse)
	fmt.Printf("Idle:                 %d\n", stats.Idle)
}
```

### Database Config in the Config Struct

```go
package config

import (
	"os"
	"time"
)

type DatabaseConfig struct {
	URL            string
	MaxOpenConns   int
	MaxIdleConns   int
	ConnMaxLife    time.Duration
	ConnMaxIdle    time.Duration
}

func loadDatabaseConfig() DatabaseConfig {
	return DatabaseConfig{
		URL:          os.Getenv("DATABASE_URL"),
		MaxOpenConns: getEnvInt("DB_MAX_OPEN_CONNS", 25),
		MaxIdleConns: getEnvInt("DB_MAX_IDLE_CONNS", 5),
		ConnMaxLife:  getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
		ConnMaxIdle:  getEnvDuration("DB_CONN_MAX_IDLE_TIME", 1*time.Minute),
	}
}

// ApplyToPool configures a database connection pool.
// This is a convenient method that applies all settings at once.
func (c DatabaseConfig) ApplyToPool(db interface {
	SetMaxOpenConns(n int)
	SetMaxIdleConns(n int)
	SetConnMaxLifetime(d time.Duration)
	SetConnMaxIdleTime(d time.Duration)
}) {
	db.SetMaxOpenConns(c.MaxOpenConns)
	db.SetMaxIdleConns(c.MaxIdleConns)
	db.SetConnMaxLifetime(c.ConnMaxLife)
	db.SetConnMaxIdleTime(c.ConnMaxIdle)
}
```

Using it:

```go
package main

import (
	"database/sql"
	"log"

	_ "github.com/lib/pq"
)

func main() {
	cfg := loadDatabaseConfig()

	db, err := sql.Open("postgres", cfg.URL)
	if err != nil {
		log.Fatalf("Failed to open database: %v", err)
	}
	defer db.Close()

	// Apply all pool settings in one call
	cfg.ApplyToPool(db)

	if err := db.Ping(); err != nil {
		log.Fatalf("Failed to ping database: %v", err)
	}

	log.Printf("Connected to database with pool: maxOpen=%d, maxIdle=%d, maxLife=%s, maxIdleTime=%s",
		cfg.MaxOpenConns, cfg.MaxIdleConns, cfg.ConnMaxLife, cfg.ConnMaxIdle)
}
```

### Validating Pool Settings

```go
func (c DatabaseConfig) validate() []string {
	var errs []string

	if c.URL == "" {
		errs = append(errs, "DATABASE_URL is required")
	}

	if c.MaxOpenConns < 1 {
		errs = append(errs, "DB_MAX_OPEN_CONNS must be at least 1")
	}

	if c.MaxIdleConns < 0 {
		errs = append(errs, "DB_MAX_IDLE_CONNS cannot be negative")
	}

	if c.MaxIdleConns > c.MaxOpenConns {
		errs = append(errs,
			fmt.Sprintf("DB_MAX_IDLE_CONNS (%d) should not exceed DB_MAX_OPEN_CONNS (%d)",
				c.MaxIdleConns, c.MaxOpenConns))
	}

	if c.ConnMaxLife < 0 {
		errs = append(errs, "DB_CONN_MAX_LIFETIME cannot be negative")
	}

	if c.ConnMaxIdle < 0 {
		errs = append(errs, "DB_CONN_MAX_IDLE_TIME cannot be negative")
	}

	if c.ConnMaxIdle > 0 && c.ConnMaxLife > 0 && c.ConnMaxIdle > c.ConnMaxLife {
		errs = append(errs,
			fmt.Sprintf("DB_CONN_MAX_IDLE_TIME (%s) should not exceed DB_CONN_MAX_LIFETIME (%s)",
				c.ConnMaxIdle, c.ConnMaxLife))
	}

	return errs
}
```

### Production Sizing Guidelines

```
┌────────────────────────────┬─────────────────────────────────────────┐
│ Setting                    │ Guideline                               │
├────────────────────────────┼─────────────────────────────────────────┤
│ MaxOpenConns               │ PostgreSQL max_connections / app_count  │
│                            │ e.g., 100 / 4 instances = 25 each      │
│                            │                                         │
│ MaxIdleConns               │ 25-50% of MaxOpenConns                  │
│                            │ Higher if traffic is bursty             │
│                            │                                         │
│ ConnMaxLifetime            │ 5 minutes typical                       │
│                            │ Shorter if behind a load balancer       │
│                            │ Must be less than DB/firewall timeout   │
│                            │                                         │
│ ConnMaxIdleTime            │ 1-2 minutes typical                     │
│                            │ Helps clean up during low traffic       │
└────────────────────────────┴─────────────────────────────────────────┘

Example calculations:
  PostgreSQL max_connections = 100
  Reserved for admin/monitoring = 10
  Available for app = 90
  Number of app instances = 3
  MaxOpenConns per instance = 90 / 3 = 30

  MaxIdleConns = 30 * 0.3 = ~10
  ConnMaxLifetime = 5m (AWS RDS proxy timeout is 15m, stay well under)
  ConnMaxIdleTime = 1m
```

### Node.js Comparison

```javascript
// Node.js with pg (node-postgres)
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 25,                        // MaxOpenConns equivalent
  idleTimeoutMillis: 60000,       // ConnMaxIdleTime equivalent (1 minute)
  connectionTimeoutMillis: 5000,  // No Go equivalent (Go blocks waiting)
});

// Node.js pg does not have:
// - MaxIdleConns (idle connections are just kept or timed out)
// - ConnMaxLifetime (connections live until idle timeout or error)
//
// Go's pool is more configurable, which matters in production
// with multiple app instances sharing a PostgreSQL server.
```

---

## 10. Configuration with Functional Options

### The Options Pattern

When building reusable libraries or configurable components, the **functional options pattern** is Go's idiomatic approach. Instead of requiring callers to create a full config struct, you provide option functions that modify an internal configuration.

### WHY Functional Options

Consider building an HTTP server wrapper:

```go
// Approach 1: Large config struct (works but cumbersome)
type ServerConfig struct {
    Port         string
    ReadTimeout  time.Duration
    WriteTimeout time.Duration
    IdleTimeout  time.Duration
    MaxBodySize  int64
    TLSCertFile  string
    TLSKeyFile   string
    // ... 20 more fields
}

// Caller must know about ALL fields, even if they only want to set one:
server := NewServer(ServerConfig{
    Port:        "8080",
    ReadTimeout: 15 * time.Second,
    // What are the defaults for everything else?
})

// Approach 2: Functional options (idiomatic Go)
server := NewServer(
    WithPort("8080"),
    WithReadTimeout(15 * time.Second),
)
// Clean, self-documenting, backwards-compatible
```

### Complete Implementation

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// --- Server with functional options ---

// Server wraps http.Server with sensible defaults and options.
type Server struct {
	port         string
	readTimeout  time.Duration
	writeTimeout time.Duration
	idleTimeout  time.Duration
	maxBodySize  int64
	logger       *slog.Logger
	handler      http.Handler
}

// Option is a function that configures the Server.
type Option func(*Server)

// WithPort sets the server port.
func WithPort(port string) Option {
	return func(s *Server) {
		s.port = port
	}
}

// WithReadTimeout sets the maximum duration for reading the entire request.
func WithReadTimeout(d time.Duration) Option {
	return func(s *Server) {
		s.readTimeout = d
	}
}

// WithWriteTimeout sets the maximum duration before timing out writes of the response.
func WithWriteTimeout(d time.Duration) Option {
	return func(s *Server) {
		s.writeTimeout = d
	}
}

// WithIdleTimeout sets the maximum amount of time to wait for the next request
// when keep-alives are enabled.
func WithIdleTimeout(d time.Duration) Option {
	return func(s *Server) {
		s.idleTimeout = d
	}
}

// WithMaxBodySize sets the maximum allowed size of the request body.
func WithMaxBodySize(size int64) Option {
	return func(s *Server) {
		s.maxBodySize = size
	}
}

// WithLogger sets the server's logger.
func WithLogger(logger *slog.Logger) Option {
	return func(s *Server) {
		s.logger = logger
	}
}

// NewServer creates a Server with sensible defaults, then applies options.
func NewServer(handler http.Handler, opts ...Option) *Server {
	// Start with sensible defaults
	s := &Server{
		port:         "8080",
		readTimeout:  15 * time.Second,
		writeTimeout: 15 * time.Second,
		idleTimeout:  60 * time.Second,
		maxBodySize:  1 << 20, // 1 MB
		logger:       slog.Default(),
		handler:      handler,
	}

	// Apply each option
	for _, opt := range opts {
		opt(s)
	}

	return s
}

// Start begins listening and serving. It blocks until the context is cancelled
// or a termination signal is received, then shuts down gracefully.
func (s *Server) Start(ctx context.Context) error {
	httpServer := &http.Server{
		Addr:         ":" + s.port,
		Handler:      http.MaxBytesHandler(s.handler, s.maxBodySize),
		ReadTimeout:  s.readTimeout,
		WriteTimeout: s.writeTimeout,
		IdleTimeout:  s.idleTimeout,
	}

	// Channel to receive server errors
	errCh := make(chan error, 1)

	go func() {
		s.logger.Info("server starting",
			"port", s.port,
			"readTimeout", s.readTimeout,
			"writeTimeout", s.writeTimeout,
		)
		errCh <- httpServer.ListenAndServe()
	}()

	// Wait for context cancellation or signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-errCh:
		return fmt.Errorf("server error: %w", err)
	case sig := <-sigCh:
		s.logger.Info("received signal, shutting down", "signal", sig)
	case <-ctx.Done():
		s.logger.Info("context cancelled, shutting down")
	}

	// Graceful shutdown with timeout
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := httpServer.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown error: %w", err)
	}

	s.logger.Info("server stopped gracefully")
	return nil
}

// --- Usage ---

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello, World!")
	})

	// Create server with custom options -- only override what you need
	server := NewServer(mux,
		WithPort("3000"),
		WithReadTimeout(30*time.Second),
		WithLogger(slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
			Level: slog.LevelDebug,
		}))),
	)

	// Start blocks until shutdown
	if err := server.Start(context.Background()); err != nil {
		fmt.Fprintf(os.Stderr, "server error: %v\n", err)
		os.Exit(1)
	}
}
```

### Connecting Options to Config

In a real application, you bridge the config struct and functional options:

```go
package main

import (
	"context"
	"log"
	"net/http"

	"yourproject/internal/config"
	"yourproject/internal/server"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		log.Fatalf("config: %v", err)
	}

	mux := http.NewServeMux()
	// ... register routes ...

	// Convert config into server options
	srv := server.NewServer(mux,
		server.WithPort(cfg.Server.Port),
		server.WithReadTimeout(cfg.Server.ReadTimeout),
		server.WithWriteTimeout(cfg.Server.WriteTimeout),
		server.WithIdleTimeout(cfg.Server.IdleTimeout),
	)

	if err := srv.Start(context.Background()); err != nil {
		log.Fatalf("server: %v", err)
	}
}
```

### WHY Not Just Pass the Config Struct

You might ask: "Why not just pass `cfg.Server` directly to `NewServer`?" Both approaches work, but functional options have advantages when building **libraries**:

1. **Backwards compatibility.** Adding a new option does not break existing callers.
2. **Self-documenting.** `WithReadTimeout(30*time.Second)` reads like English.
3. **Conditional options.** You can conditionally apply options:

```go
opts := []server.Option{
	server.WithPort(cfg.Server.Port),
}

if cfg.IsProduction() {
	opts = append(opts,
		server.WithReadTimeout(5*time.Second),  // Stricter in production
		server.WithMaxBodySize(512<<10),          // 512 KB max
	)
}

srv := server.NewServer(mux, opts...)
```

For **application-internal** configuration, a config struct is simpler and perfectly fine. Use functional options when building libraries or configurable components that will be used across multiple projects.

---

## 11. Secrets Management

### What is a Secret

A secret is any configuration value that, if leaked, would compromise your application's security. Common secrets include:

- Database connection strings (contains password)
- JWT signing keys
- API keys for third-party services
- OAuth client secrets
- Encryption keys
- SMTP passwords

### Secrets in Environment Variables

The simplest approach is environment variables, but you must follow strict rules:

```go
package main

import (
	"encoding/hex"
	"fmt"
	"os"
)

func main() {
	// Rule 1: NEVER log secrets
	// BAD:
	fmt.Printf("JWT Key: %s\n", os.Getenv("JWT_SIGNING_KEY"))
	// If this ends up in a log aggregator, you have a breach.

	// GOOD: Log that the secret exists, not its value
	if os.Getenv("JWT_SIGNING_KEY") != "" {
		fmt.Println("JWT_SIGNING_KEY: [set]")
	} else {
		fmt.Println("JWT_SIGNING_KEY: [NOT SET]")
	}

	// Rule 2: NEVER include secrets in error messages
	// BAD:
	// return fmt.Errorf("failed to connect to %s", dbURL)
	// GOOD:
	// return fmt.Errorf("failed to connect to database: %w", err)

	// Rule 3: NEVER commit secrets to version control
	// .env should be in .gitignore
	// .env.example should have placeholders, not real values

	// Rule 4: Use hex encoding for binary secrets (like signing keys)
	// Hex is safe for environment variables (no special characters)
	keyHex := os.Getenv("JWT_SIGNING_KEY")
	if keyHex != "" {
		key, err := hex.DecodeString(keyHex)
		if err != nil {
			fmt.Printf("JWT_SIGNING_KEY is not valid hex: %v\n", err)
			return
		}
		fmt.Printf("JWT key loaded: %d bytes\n", len(key))
	}
}
```

### Hex-Encoded Secrets

Binary keys (for HMAC-SHA256, AES, etc.) cannot be stored directly in environment variables because they may contain null bytes, newlines, or other characters that shells mishandle. The standard approach is hex encoding:

```go
package main

import (
	"crypto/hmac"
	"crypto/rand"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"os"
)

// generateKey creates a new random key and prints it as hex.
// Run this once to generate a key for your .env file:
//   go run keygen.go
// Or use: openssl rand -hex 32
func generateKey(byteLength int) string {
	key := make([]byte, byteLength)
	if _, err := rand.Read(key); err != nil {
		panic(fmt.Sprintf("failed to generate random key: %v", err))
	}
	return hex.EncodeToString(key)
}

func main() {
	// Generate a new 32-byte (256-bit) key
	fmt.Printf("New JWT key: %s\n", generateKey(32))
	// Output: New JWT key: a1b2c3d4e5f6...  (64 hex characters = 32 bytes)

	// In production, this key is set as an environment variable:
	//   JWT_SIGNING_KEY=a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2

	// Loading the key:
	keyHex := os.Getenv("JWT_SIGNING_KEY")
	if keyHex == "" {
		fmt.Println("JWT_SIGNING_KEY is not set")
		// For demonstration, use a generated key
		keyHex = generateKey(32)
		os.Setenv("JWT_SIGNING_KEY", keyHex)
	}

	key, err := hex.DecodeString(keyHex)
	if err != nil {
		fmt.Printf("Invalid hex key: %v\n", err)
		return
	}

	// Validate key length
	if len(key) < 32 {
		fmt.Printf("Key too short: %d bytes (need at least 32)\n", len(key))
		return
	}

	fmt.Printf("Key loaded successfully: %d bytes\n", len(key))

	// Use the key for HMAC-SHA256 signing
	mac := hmac.New(sha256.New, key)
	mac.Write([]byte("token-payload-here"))
	signature := mac.Sum(nil)
	fmt.Printf("Signature: %s\n", hex.EncodeToString(signature))
}
```

### WHY Hex Instead of Base64

Both hex and base64 encode binary data as ASCII strings. The choice matters:

```go
package main

import (
	"encoding/base64"
	"encoding/hex"
	"fmt"
)

func main() {
	// A 32-byte key
	key := []byte{0xA1, 0xB2, 0xC3, 0xD4, 0xE5, 0xF6, 0x07, 0x18,
		0x29, 0x3A, 0x4B, 0x5C, 0x6D, 0x7E, 0x8F, 0x90,
		0xA1, 0xB2, 0xC3, 0xD4, 0xE5, 0xF6, 0x07, 0x18,
		0x29, 0x3A, 0x4B, 0x5C, 0x6D, 0x7E, 0x8F, 0x90}

	hexStr := hex.EncodeToString(key)
	b64Str := base64.StdEncoding.EncodeToString(key)

	fmt.Printf("Hex:    %s (%d chars)\n", hexStr, len(hexStr))
	fmt.Printf("Base64: %s (%d chars)\n", b64Str, len(b64Str))

	// Hex:    a1b2c3d4e5f6071829...  (64 chars)
	// Base64: obLD1OX2BxgpOktcbX6P... (44 chars)

	// Hex advantages:
	// 1. Only uses [0-9a-f] -- no +, /, = that shells might interpret
	// 2. Easy to visually inspect (each byte is exactly 2 chars)
	// 3. Easy to validate length (64 hex chars = 32 bytes)
	// 4. Compatible with openssl rand -hex 32

	// Base64 advantages:
	// 1. Shorter (33% smaller than hex)
	// 2. Standard in JWTs and web APIs

	// For environment variables, hex is safer and more common.
	// For wire protocols (JWT payloads, HTTP headers), base64 is standard.
}
```

### The Config Struct with Masked Secrets

Add a `String()` method that masks sensitive values for safe logging:

```go
package main

import (
	"fmt"
	"strings"
	"time"
)

type Config struct {
	Port          string
	DatabaseURL   string
	JWTSigningKey []byte
	JWTAccessTTL  time.Duration
	JWTRefreshTTL time.Duration
	CORSOrigins   []string
	LogLevel      string
	CookieDomain  string
	CookieSecure  bool
}

// String returns a safe-to-log representation of the config.
// Secrets are masked, not printed.
func (c *Config) String() string {
	var b strings.Builder
	b.WriteString("Configuration:\n")
	b.WriteString(fmt.Sprintf("  Port:           %s\n", c.Port))
	b.WriteString(fmt.Sprintf("  DatabaseURL:    %s\n", maskDatabaseURL(c.DatabaseURL)))
	b.WriteString(fmt.Sprintf("  JWTSigningKey:  %s\n", maskBytes(c.JWTSigningKey)))
	b.WriteString(fmt.Sprintf("  JWTAccessTTL:   %s\n", c.JWTAccessTTL))
	b.WriteString(fmt.Sprintf("  JWTRefreshTTL:  %s\n", c.JWTRefreshTTL))
	b.WriteString(fmt.Sprintf("  CORSOrigins:    %v\n", c.CORSOrigins))
	b.WriteString(fmt.Sprintf("  LogLevel:       %s\n", c.LogLevel))
	b.WriteString(fmt.Sprintf("  CookieDomain:   %s\n", c.CookieDomain))
	b.WriteString(fmt.Sprintf("  CookieSecure:   %v\n", c.CookieSecure))
	return b.String()
}

// maskBytes returns "[N bytes]" for non-empty keys, "[not set]" for empty.
func maskBytes(data []byte) string {
	if len(data) == 0 {
		return "[not set]"
	}
	return fmt.Sprintf("[%d bytes]", len(data))
}

// maskDatabaseURL replaces the password in a database URL.
func maskDatabaseURL(rawURL string) string {
	if rawURL == "" {
		return "[not set]"
	}
	// Simple approach: find the password between : and @ after //
	// postgres://user:password@host:5432/db
	idx := strings.Index(rawURL, "://")
	if idx < 0 {
		return "[invalid URL]"
	}
	rest := rawURL[idx+3:]
	atIdx := strings.Index(rest, "@")
	if atIdx < 0 {
		return rawURL // No credentials in URL
	}
	colonIdx := strings.Index(rest[:atIdx], ":")
	if colonIdx < 0 {
		return rawURL // No password in URL
	}
	// Replace password with ****
	return rawURL[:idx+3+colonIdx+1] + "****" + rawURL[idx+3+atIdx:]
}

func main() {
	cfg := &Config{
		Port:          "8080",
		DatabaseURL:   "postgres://appuser:s3cret_p@ss@db.example.com:5432/mydb",
		JWTSigningKey: make([]byte, 32),
		JWTAccessTTL:  15 * time.Minute,
		JWTRefreshTTL: 168 * time.Hour,
		CORSOrigins:   []string{"https://app.example.com"},
		LogLevel:      "info",
		CookieDomain:  "example.com",
		CookieSecure:  true,
	}

	// Safe to log -- secrets are masked
	fmt.Println(cfg)
	// Configuration:
	//   Port:           8080
	//   DatabaseURL:    postgres://appuser:****@db.example.com:5432/mydb
	//   JWTSigningKey:  [32 bytes]
	//   JWTAccessTTL:   15m0s
	//   JWTRefreshTTL:  168h0m0s
	//   CORSOrigins:    [https://app.example.com]
	//   LogLevel:       info
	//   CookieDomain:   example.com
	//   CookieSecure:   true
}
```

### Production Secrets Management

For production environments, secrets typically come from dedicated secrets management systems rather than plain environment variables:

```
┌──────────────────────────────────────────────────────────┐
│           Secrets Management Options                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Kubernetes Secrets                                      │
│  └─ Injected as env vars or mounted as files             │
│  └─ Encrypted at rest in etcd                            │
│  └─ RBAC-controlled access                               │
│                                                          │
│  HashiCorp Vault                                         │
│  └─ Dynamic secrets (rotated automatically)              │
│  └─ Audit logging                                        │
│  └─ Multiple auth methods                                │
│                                                          │
│  AWS Secrets Manager / SSM Parameter Store               │
│  └─ Integrated with IAM                                  │
│  └─ Automatic rotation                                   │
│  └─ Cross-region replication                             │
│                                                          │
│  Google Secret Manager                                   │
│  └─ Integrated with Cloud IAM                            │
│  └─ Versioned secrets                                    │
│                                                          │
│  All of these can inject secrets as environment          │
│  variables, so your Config.Load() works unchanged.       │
│  The secret lifecycle management happens outside the app.│
└──────────────────────────────────────────────────────────┘
```

Example Kubernetes Secret that injects as environment variables:

```yaml
# kubernetes/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: krafty-core-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:password@prod-db:5432/krafty?sslmode=require"
  JWT_SIGNING_KEY: "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2"

---
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: krafty-core
spec:
  template:
    spec:
      containers:
      - name: krafty-core
        image: krafty-core:latest
        envFrom:
        - secretRef:
            name: krafty-core-secrets
        env:
        - name: PORT
          value: "8080"
        - name: CORS_ORIGINS
          value: "https://app.krafty.com"
        - name: COOKIE_SECURE
          value: "true"
        - name: COOKIE_DOMAIN
          value: "krafty.com"
        - name: LOG_LEVEL
          value: "info"
```

Your Go application's `config.Load()` function does not know or care where the environment variables come from. It just reads them with `os.Getenv`. This is the power of the 12-factor approach.

---

## 12. Configuration Testing

### WHY Test Configuration

Configuration loading and validation logic is code. It can have bugs. Common bugs include:
- Default value typos
- Missing validation rules
- Validation logic that accepts invalid values
- CSV parsing that loses items
- Duration parsing that silently falls back to defaults

### Testing the Load Function

```go
package config

import (
	"os"
	"strings"
	"testing"
	"time"
)

// setEnvVars is a test helper that sets environment variables
// and returns a cleanup function that restores the original values.
func setEnvVars(t *testing.T, vars map[string]string) {
	t.Helper()

	originals := make(map[string]string)
	unset := make([]string, 0)

	for key, value := range vars {
		if orig, exists := os.LookupEnv(key); exists {
			originals[key] = orig
		} else {
			unset = append(unset, key)
		}
		os.Setenv(key, value)
	}

	// Register cleanup to restore environment after test
	t.Cleanup(func() {
		for key, orig := range originals {
			os.Setenv(key, orig)
		}
		for _, key := range unset {
			os.Unsetenv(key)
		}
	})
}

// clearEnvVars unsets all config-related environment variables.
func clearEnvVars(t *testing.T) {
	t.Helper()

	keys := []string{
		"PORT", "DATABASE_URL", "JWT_SIGNING_KEY",
		"JWT_ACCESS_TTL", "JWT_REFRESH_TTL", "CORS_ORIGINS",
		"LOG_LEVEL", "COOKIE_DOMAIN", "COOKIE_SECURE",
		"DB_MAX_OPEN_CONNS", "DB_MAX_IDLE_CONNS",
		"DB_CONN_MAX_LIFETIME", "DB_CONN_MAX_IDLE_TIME",
	}

	originals := make(map[string]string)
	unset := make([]string, 0)

	for _, key := range keys {
		if orig, exists := os.LookupEnv(key); exists {
			originals[key] = orig
		} else {
			unset = append(unset, key)
		}
		os.Unsetenv(key)
	}

	t.Cleanup(func() {
		for key, orig := range originals {
			os.Setenv(key, orig)
		}
		for _, key := range unset {
			os.Unsetenv(key)
		}
	})
}

func TestLoad_ValidConfig(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"PORT":              "3000",
		"DATABASE_URL":      "postgres://user:pass@localhost:5432/testdb?sslmode=disable",
		"JWT_SIGNING_KEY":   "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2",
		"JWT_ACCESS_TTL":    "30m",
		"JWT_REFRESH_TTL":   "72h",
		"CORS_ORIGINS":      "http://localhost:4200,http://localhost:3000",
		"LOG_LEVEL":         "debug",
		"COOKIE_DOMAIN":     "localhost",
		"COOKIE_SECURE":     "false",
		"DB_MAX_OPEN_CONNS": "10",
	})

	cfg, err := Load()
	if err != nil {
		t.Fatalf("expected no error, got: %v", err)
	}

	if cfg.Port != "3000" {
		t.Errorf("expected Port=3000, got %s", cfg.Port)
	}

	if cfg.JWTAccessTTL != 30*time.Minute {
		t.Errorf("expected JWTAccessTTL=30m, got %s", cfg.JWTAccessTTL)
	}

	if len(cfg.CORSOrigins) != 2 {
		t.Errorf("expected 2 CORS origins, got %d", len(cfg.CORSOrigins))
	}

	if len(cfg.JWTSigningKey) != 32 {
		t.Errorf("expected 32-byte JWT key, got %d bytes", len(cfg.JWTSigningKey))
	}
}

func TestLoad_MissingRequired(t *testing.T) {
	clearEnvVars(t)

	// Do not set DATABASE_URL or JWT_SIGNING_KEY
	setEnvVars(t, map[string]string{
		"CORS_ORIGINS": "http://localhost:4200",
	})

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for missing required fields")
	}

	// Should report BOTH errors, not just the first one
	errStr := err.Error()
	if !strings.Contains(errStr, "DATABASE_URL") {
		t.Errorf("expected error about DATABASE_URL, got: %s", errStr)
	}
	if !strings.Contains(errStr, "JWT_SIGNING_KEY") {
		t.Errorf("expected error about JWT_SIGNING_KEY, got: %s", errStr)
	}
}

func TestLoad_InvalidPort(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"PORT":            "99999", // Invalid: > 65535
		"DATABASE_URL":    "postgres://user:pass@localhost:5432/testdb",
		"JWT_SIGNING_KEY": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2",
		"CORS_ORIGINS":    "http://localhost:4200",
	})

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for invalid port")
	}

	if !strings.Contains(err.Error(), "PORT") {
		t.Errorf("expected error about PORT, got: %s", err.Error())
	}
}

func TestLoad_InvalidHexKey(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"DATABASE_URL":    "postgres://user:pass@localhost:5432/testdb",
		"JWT_SIGNING_KEY": "not-valid-hex",
		"CORS_ORIGINS":    "http://localhost:4200",
	})

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for invalid hex key")
	}

	if !strings.Contains(err.Error(), "hex") {
		t.Errorf("expected hex-related error, got: %s", err.Error())
	}
}

func TestLoad_ShortKey(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"DATABASE_URL":    "postgres://user:pass@localhost:5432/testdb",
		"JWT_SIGNING_KEY": "aabbccdd", // Only 4 bytes (8 hex chars)
		"CORS_ORIGINS":    "http://localhost:4200",
	})

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for short JWT key")
	}

	if !strings.Contains(err.Error(), "32 bytes") {
		t.Errorf("expected error about key length, got: %s", err.Error())
	}
}

func TestLoad_Defaults(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"DATABASE_URL":    "postgres://user:pass@localhost:5432/testdb",
		"JWT_SIGNING_KEY": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2",
		"CORS_ORIGINS":    "http://localhost:4200",
		// Do not set PORT, LOG_LEVEL, etc -- they should use defaults
	})

	cfg, err := Load()
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	if cfg.Port != "8080" {
		t.Errorf("expected default Port=8080, got %s", cfg.Port)
	}

	if cfg.LogLevel != "info" {
		t.Errorf("expected default LogLevel=info, got %s", cfg.LogLevel)
	}

	if cfg.JWTAccessTTL != 15*time.Minute {
		t.Errorf("expected default JWTAccessTTL=15m, got %s", cfg.JWTAccessTTL)
	}

	if cfg.JWTRefreshTTL != 168*time.Hour {
		t.Errorf("expected default JWTRefreshTTL=168h, got %s", cfg.JWTRefreshTTL)
	}
}

func TestLoad_CORSWildcardInProduction(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"DATABASE_URL":    "postgres://user:pass@localhost:5432/testdb",
		"JWT_SIGNING_KEY": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2",
		"CORS_ORIGINS":    "*",
		"COOKIE_SECURE":   "true", // Production-like setting
		"COOKIE_DOMAIN":   "example.com",
	})

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for CORS wildcard with COOKIE_SECURE=true")
	}

	if !strings.Contains(err.Error(), "CORS") {
		t.Errorf("expected CORS error, got: %s", err.Error())
	}
}

func TestLoad_InvalidLogLevel(t *testing.T) {
	clearEnvVars(t)
	setEnvVars(t, map[string]string{
		"DATABASE_URL":    "postgres://user:pass@localhost:5432/testdb",
		"JWT_SIGNING_KEY": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2",
		"CORS_ORIGINS":    "http://localhost:4200",
		"LOG_LEVEL":       "verbose", // Not a valid level
	})

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for invalid log level")
	}

	if !strings.Contains(err.Error(), "LOG_LEVEL") {
		t.Errorf("expected LOG_LEVEL error, got: %s", err.Error())
	}
}
```

### Testing Helper Functions

```go
package config

import (
	"testing"
	"time"
)

func TestParseCSV(t *testing.T) {
	tests := []struct {
		name     string
		input    string
		expected []string
	}{
		{
			name:     "empty string",
			input:    "",
			expected: nil,
		},
		{
			name:     "single value",
			input:    "http://localhost:4200",
			expected: []string{"http://localhost:4200"},
		},
		{
			name:     "multiple values",
			input:    "http://localhost:4200,https://app.example.com",
			expected: []string{"http://localhost:4200", "https://app.example.com"},
		},
		{
			name:     "with spaces",
			input:    "http://localhost:4200 , https://app.example.com , https://admin.example.com",
			expected: []string{"http://localhost:4200", "https://app.example.com", "https://admin.example.com"},
		},
		{
			name:     "trailing comma",
			input:    "http://localhost:4200,",
			expected: []string{"http://localhost:4200"},
		},
		{
			name:     "empty elements",
			input:    "a,,b,,c",
			expected: []string{"a", "b", "c"},
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := parseCSV(tt.input)

			if tt.expected == nil && result != nil {
				t.Errorf("expected nil, got %v", result)
				return
			}

			if len(result) != len(tt.expected) {
				t.Errorf("expected %d elements, got %d: %v",
					len(tt.expected), len(result), result)
				return
			}

			for i, v := range result {
				if v != tt.expected[i] {
					t.Errorf("element %d: expected %q, got %q", i, tt.expected[i], v)
				}
			}
		})
	}
}

func TestGetEnvDuration(t *testing.T) {
	tests := []struct {
		name     string
		envValue string
		envSet   bool
		defVal   time.Duration
		expected time.Duration
	}{
		{
			name:     "not set uses default",
			envSet:   false,
			defVal:   15 * time.Minute,
			expected: 15 * time.Minute,
		},
		{
			name:     "valid duration",
			envValue: "30m",
			envSet:   true,
			defVal:   15 * time.Minute,
			expected: 30 * time.Minute,
		},
		{
			name:     "invalid duration uses default",
			envValue: "not-a-duration",
			envSet:   true,
			defVal:   15 * time.Minute,
			expected: 15 * time.Minute,
		},
		{
			name:     "hours",
			envValue: "168h",
			envSet:   true,
			defVal:   24 * time.Hour,
			expected: 168 * time.Hour,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			key := "TEST_DURATION_" + tt.name
			if tt.envSet {
				t.Setenv(key, tt.envValue)
			}

			result := getEnvDuration(key, tt.defVal)
			if result != tt.expected {
				t.Errorf("expected %s, got %s", tt.expected, result)
			}
		})
	}
}

func TestGetEnvBool(t *testing.T) {
	tests := []struct {
		name     string
		envValue string
		envSet   bool
		defVal   bool
		expected bool
	}{
		{name: "not set uses default true", envSet: false, defVal: true, expected: true},
		{name: "not set uses default false", envSet: false, defVal: false, expected: false},
		{name: "true", envValue: "true", envSet: true, defVal: false, expected: true},
		{name: "false", envValue: "false", envSet: true, defVal: true, expected: false},
		{name: "1", envValue: "1", envSet: true, defVal: false, expected: true},
		{name: "0", envValue: "0", envSet: true, defVal: true, expected: false},
		{name: "invalid uses default", envValue: "yes", envSet: true, defVal: true, expected: true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			key := "TEST_BOOL_" + tt.name
			if tt.envSet {
				t.Setenv(key, tt.envValue)
			}

			result := getEnvBool(key, tt.defVal)
			if result != tt.expected {
				t.Errorf("expected %v, got %v", tt.expected, result)
			}
		})
	}
}
```

### Using t.Setenv (Go 1.17+)

Go 1.17 introduced `t.Setenv`, which automatically restores the environment variable after the test:

```go
func TestWithSetenv(t *testing.T) {
	// t.Setenv sets the variable and automatically restores it after the test.
	// It also calls t.Parallel() safety check -- tests using t.Setenv
	// cannot be run in parallel (because env vars are process-global).
	t.Setenv("PORT", "9090")
	t.Setenv("DATABASE_URL", "postgres://user:pass@localhost:5432/testdb")

	port := getEnv("PORT", "8080")
	if port != "9090" {
		t.Errorf("expected 9090, got %s", port)
	}
}
```

### Testing Tip: Parallel Tests and Environment Variables

```go
// WARNING: These tests CANNOT run in parallel because they modify
// process-global environment variables.
//
// Do NOT add t.Parallel() to config tests.
// Go's testing framework will detect this and panic:
//   "panic: testing: t.Setenv called after t.Parallel"

func TestConfigLoad(t *testing.T) {
	// DO NOT call t.Parallel() here
	t.Setenv("PORT", "3000")
	// ...
}
```

---

## 13. Real-World Example: Complete Config Package

This section brings everything together into a complete, production-ready configuration package inspired by the krafty-core project. This is the kind of code you would find in `internal/config/config.go` of a real Go API.

### File: internal/config/config.go

```go
package config

import (
	"encoding/hex"
	"fmt"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/joho/godotenv"
)

// Config holds the complete application configuration.
// All values are loaded from environment variables at startup.
// Required values have no defaults and must be explicitly set.
// Optional values have sensible defaults documented in each field's comment.
type Config struct {
	// Environment is the running environment (development, staging, production, test).
	// Default: "development"
	// Env: APP_ENV
	Environment Environment

	// Server configuration
	Server ServerConfig

	// Database configuration
	Database DatabaseConfig

	// JWT authentication configuration
	JWT JWTConfig

	// CORS configuration
	CORS CORSConfig

	// Cookie configuration
	Cookie CookieConfig
}

// Environment represents the application environment.
type Environment string

const (
	EnvDevelopment Environment = "development"
	EnvStaging     Environment = "staging"
	EnvProduction  Environment = "production"
	EnvTest        Environment = "test"
)

// ServerConfig holds HTTP server settings.
type ServerConfig struct {
	// Port for the HTTP server. Default: "8080". Env: PORT
	Port string

	// ReadTimeout is the maximum duration for reading the entire request.
	// Default: 15s. Env: SERVER_READ_TIMEOUT
	ReadTimeout time.Duration

	// WriteTimeout is the maximum duration before timing out writes.
	// Default: 15s. Env: SERVER_WRITE_TIMEOUT
	WriteTimeout time.Duration

	// IdleTimeout is the max time to wait for the next request on keep-alive.
	// Default: 60s. Env: SERVER_IDLE_TIMEOUT
	IdleTimeout time.Duration

	// LogLevel controls the logging verbosity.
	// Valid values: "debug", "info", "warn", "error".
	// Default: "debug" (development), "info" (production). Env: LOG_LEVEL
	LogLevel string
}

// DatabaseConfig holds database connection and pool settings.
type DatabaseConfig struct {
	// URL is the database connection string.
	// Required. No default. Env: DATABASE_URL
	// Format: postgres://user:password@host:port/dbname?sslmode=disable
	URL string

	// MaxOpenConns is the maximum number of open connections to the database.
	// Default: 25. Env: DB_MAX_OPEN_CONNS
	MaxOpenConns int

	// MaxIdleConns is the maximum number of idle connections in the pool.
	// Default: 5. Env: DB_MAX_IDLE_CONNS
	MaxIdleConns int

	// ConnMaxLifetime is the maximum time a connection can be reused.
	// Default: 5m. Env: DB_CONN_MAX_LIFETIME
	ConnMaxLifetime time.Duration

	// ConnMaxIdleTime is the maximum time a connection can be idle.
	// Default: 1m. Env: DB_CONN_MAX_IDLE_TIME
	ConnMaxIdleTime time.Duration
}

// JWTConfig holds JWT authentication settings.
type JWTConfig struct {
	// SigningKey is the HMAC-SHA256 signing key (raw bytes, decoded from hex).
	// Required. No default. Env: JWT_SIGNING_KEY (hex-encoded, 64 chars = 32 bytes)
	// Generate with: openssl rand -hex 32
	SigningKey []byte

	// AccessTTL is the lifetime of access tokens.
	// Default: 15m. Env: JWT_ACCESS_TTL
	AccessTTL time.Duration

	// RefreshTTL is the lifetime of refresh tokens.
	// Default: 168h (7 days). Env: JWT_REFRESH_TTL
	RefreshTTL time.Duration

	// Issuer is the "iss" claim in JWTs.
	// Default: "krafty-core". Env: JWT_ISSUER
	Issuer string
}

// CORSConfig holds Cross-Origin Resource Sharing settings.
type CORSConfig struct {
	// AllowedOrigins is the list of origins allowed to make cross-origin requests.
	// Required. No default. Env: CORS_ORIGINS (comma-separated)
	AllowedOrigins []string

	// AllowedMethods is the list of HTTP methods allowed in CORS requests.
	// Default: GET, POST, PUT, PATCH, DELETE, OPTIONS.
	AllowedMethods []string

	// AllowedHeaders is the list of headers allowed in CORS requests.
	// Default: Content-Type, Authorization.
	AllowedHeaders []string

	// MaxAge is how long the browser should cache preflight results.
	// Default: 12h. Env: CORS_MAX_AGE
	MaxAge time.Duration
}

// CookieConfig holds cookie settings for session management.
type CookieConfig struct {
	// Domain is the domain for cookies.
	// Default: "localhost" (development). Env: COOKIE_DOMAIN
	Domain string

	// Secure controls the Secure flag on cookies.
	// Default: false (development), should be true in production.
	// Env: COOKIE_SECURE
	Secure bool

	// HTTPOnly controls the HttpOnly flag on cookies.
	// Default: true. Should always be true for session cookies.
	HTTPOnly bool

	// SameSite controls the SameSite attribute.
	// Valid values: "strict", "lax", "none".
	// Default: "lax". Env: COOKIE_SAMESITE
	SameSite string

	// Path is the cookie path.
	// Default: "/". Env: COOKIE_PATH
	Path string
}

// IsDevelopment returns true if running in development mode.
func (c *Config) IsDevelopment() bool {
	return c.Environment == EnvDevelopment
}

// IsProduction returns true if running in production mode.
func (c *Config) IsProduction() bool {
	return c.Environment == EnvProduction
}

// IsTest returns true if running in test mode.
func (c *Config) IsTest() bool {
	return c.Environment == EnvTest
}

// Load reads the configuration from environment variables.
// It loads .env files for development convenience, applies defaults,
// parses typed values, and validates the complete configuration.
//
// It returns a fully validated Config or an error describing ALL
// validation failures (not just the first one).
func Load() (*Config, error) {
	// Load .env file for development. Ignored if file does not exist.
	// godotenv.Load does NOT overwrite existing environment variables.
	_ = godotenv.Load()

	env := parseEnvironment(getEnv("APP_ENV", "development"))

	cfg := &Config{
		Environment: env,

		Server: ServerConfig{
			Port:         getEnv("PORT", "8080"),
			ReadTimeout:  getEnvDuration("SERVER_READ_TIMEOUT", 15*time.Second),
			WriteTimeout: getEnvDuration("SERVER_WRITE_TIMEOUT", 15*time.Second),
			IdleTimeout:  getEnvDuration("SERVER_IDLE_TIMEOUT", 60*time.Second),
			LogLevel:     getEnv("LOG_LEVEL", envDefault(env, "debug", "info")),
		},

		Database: DatabaseConfig{
			URL:             os.Getenv("DATABASE_URL"),
			MaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
			MaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 5),
			ConnMaxLifetime: getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
			ConnMaxIdleTime: getEnvDuration("DB_CONN_MAX_IDLE_TIME", 1*time.Minute),
		},

		JWT: JWTConfig{
			AccessTTL:  getEnvDuration("JWT_ACCESS_TTL", 15*time.Minute),
			RefreshTTL: getEnvDuration("JWT_REFRESH_TTL", 168*time.Hour),
			Issuer:     getEnv("JWT_ISSUER", "krafty-core"),
		},

		CORS: CORSConfig{
			AllowedOrigins: parseCSV(os.Getenv("CORS_ORIGINS")),
			AllowedMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
			AllowedHeaders: []string{"Content-Type", "Authorization"},
			MaxAge:         getEnvDuration("CORS_MAX_AGE", 12*time.Hour),
		},

		Cookie: CookieConfig{
			Domain:   getEnv("COOKIE_DOMAIN", "localhost"),
			Secure:   getEnvBool("COOKIE_SECURE", env == EnvProduction || env == EnvStaging),
			HTTPOnly: getEnvBool("COOKIE_HTTPONLY", true),
			SameSite: getEnv("COOKIE_SAMESITE", "lax"),
			Path:     getEnv("COOKIE_PATH", "/"),
		},
	}

	// Parse hex-encoded JWT signing key
	jwtKeyHex := os.Getenv("JWT_SIGNING_KEY")
	if jwtKeyHex != "" {
		key, err := hex.DecodeString(jwtKeyHex)
		if err != nil {
			return nil, fmt.Errorf("JWT_SIGNING_KEY is not valid hex: %w", err)
		}
		cfg.JWT.SigningKey = key
	}

	// Validate everything
	if err := cfg.validate(); err != nil {
		return nil, err
	}

	return cfg, nil
}

// validate checks all configuration values and returns ALL errors at once.
func (c *Config) validate() error {
	var errs []string

	// --- Server ---
	errs = append(errs, validatePort(c.Server.Port)...)
	errs = append(errs, validateLogLevel(c.Server.LogLevel)...)

	// --- Database ---
	if c.Database.URL == "" {
		errs = append(errs, "DATABASE_URL is required")
	}
	if c.Database.MaxOpenConns < 1 {
		errs = append(errs, "DB_MAX_OPEN_CONNS must be at least 1")
	}
	if c.Database.MaxIdleConns < 0 {
		errs = append(errs, "DB_MAX_IDLE_CONNS cannot be negative")
	}
	if c.Database.MaxIdleConns > c.Database.MaxOpenConns {
		errs = append(errs, fmt.Sprintf(
			"DB_MAX_IDLE_CONNS (%d) should not exceed DB_MAX_OPEN_CONNS (%d)",
			c.Database.MaxIdleConns, c.Database.MaxOpenConns))
	}

	// --- JWT ---
	if len(c.JWT.SigningKey) == 0 {
		errs = append(errs, "JWT_SIGNING_KEY is required (generate with: openssl rand -hex 32)")
	} else if len(c.JWT.SigningKey) < 32 {
		errs = append(errs, fmt.Sprintf(
			"JWT_SIGNING_KEY must be at least 32 bytes (64 hex chars), got %d bytes (%d hex chars)",
			len(c.JWT.SigningKey), len(c.JWT.SigningKey)*2))
	}

	if c.JWT.AccessTTL <= 0 {
		errs = append(errs, "JWT_ACCESS_TTL must be a positive duration")
	} else if c.JWT.AccessTTL > 1*time.Hour {
		errs = append(errs, fmt.Sprintf(
			"JWT_ACCESS_TTL should not exceed 1 hour for security (got %s)", c.JWT.AccessTTL))
	}

	if c.JWT.RefreshTTL <= 0 {
		errs = append(errs, "JWT_REFRESH_TTL must be a positive duration")
	} else if c.JWT.RefreshTTL > 30*24*time.Hour {
		errs = append(errs, fmt.Sprintf(
			"JWT_REFRESH_TTL should not exceed 30 days (got %s)", c.JWT.RefreshTTL))
	}

	if c.JWT.RefreshTTL > 0 && c.JWT.AccessTTL > 0 && c.JWT.AccessTTL >= c.JWT.RefreshTTL {
		errs = append(errs, fmt.Sprintf(
			"JWT_ACCESS_TTL (%s) must be shorter than JWT_REFRESH_TTL (%s)",
			c.JWT.AccessTTL, c.JWT.RefreshTTL))
	}

	// --- CORS ---
	if len(c.CORS.AllowedOrigins) == 0 {
		errs = append(errs, "CORS_ORIGINS is required (comma-separated list of allowed origins)")
	}
	for _, origin := range c.CORS.AllowedOrigins {
		if origin == "*" && c.IsProduction() {
			errs = append(errs,
				"CORS_ORIGINS cannot be '*' in production -- specify exact origins")
			break
		}
	}

	// --- Cookies ---
	if c.Cookie.Secure && c.Cookie.Domain == "localhost" {
		errs = append(errs,
			"COOKIE_DOMAIN should not be 'localhost' when COOKIE_SECURE is true")
	}

	validSameSite := map[string]bool{"strict": true, "lax": true, "none": true}
	if !validSameSite[strings.ToLower(c.Cookie.SameSite)] {
		errs = append(errs, fmt.Sprintf(
			"COOKIE_SAMESITE must be 'strict', 'lax', or 'none' (got %q)", c.Cookie.SameSite))
	}

	if strings.ToLower(c.Cookie.SameSite) == "none" && !c.Cookie.Secure {
		errs = append(errs,
			"COOKIE_SECURE must be true when COOKIE_SAMESITE is 'none' (browser requirement)")
	}

	// --- Return all errors ---
	if len(errs) > 0 {
		return fmt.Errorf("configuration errors:\n  - %s", strings.Join(errs, "\n  - "))
	}
	return nil
}

// validatePort checks that the port is a valid number in range 1-65535.
func validatePort(port string) []string {
	if port == "" {
		return []string{"PORT is required"}
	}
	p, err := strconv.Atoi(port)
	if err != nil {
		return []string{fmt.Sprintf("PORT must be a number, got %q", port)}
	}
	if p < 1 || p > 65535 {
		return []string{fmt.Sprintf("PORT must be between 1 and 65535, got %d", p)}
	}
	return nil
}

// validateLogLevel checks that the log level is valid.
func validateLogLevel(level string) []string {
	valid := map[string]bool{
		"debug": true, "info": true, "warn": true, "error": true,
	}
	if !valid[level] {
		return []string{fmt.Sprintf(
			"LOG_LEVEL must be one of: debug, info, warn, error (got %q)", level)}
	}
	return nil
}

// String returns a safe-to-log representation of the config.
// Secrets are masked.
func (c *Config) String() string {
	var b strings.Builder
	fmt.Fprintf(&b, "Configuration (%s):\n", c.Environment)
	fmt.Fprintf(&b, "  Server:\n")
	fmt.Fprintf(&b, "    Port:          %s\n", c.Server.Port)
	fmt.Fprintf(&b, "    ReadTimeout:   %s\n", c.Server.ReadTimeout)
	fmt.Fprintf(&b, "    WriteTimeout:  %s\n", c.Server.WriteTimeout)
	fmt.Fprintf(&b, "    IdleTimeout:   %s\n", c.Server.IdleTimeout)
	fmt.Fprintf(&b, "    LogLevel:      %s\n", c.Server.LogLevel)
	fmt.Fprintf(&b, "  Database:\n")
	fmt.Fprintf(&b, "    URL:           %s\n", maskDatabaseURL(c.Database.URL))
	fmt.Fprintf(&b, "    MaxOpenConns:  %d\n", c.Database.MaxOpenConns)
	fmt.Fprintf(&b, "    MaxIdleConns:  %d\n", c.Database.MaxIdleConns)
	fmt.Fprintf(&b, "    ConnMaxLife:   %s\n", c.Database.ConnMaxLifetime)
	fmt.Fprintf(&b, "    ConnMaxIdle:   %s\n", c.Database.ConnMaxIdleTime)
	fmt.Fprintf(&b, "  JWT:\n")
	fmt.Fprintf(&b, "    SigningKey:    %s\n", maskBytes(c.JWT.SigningKey))
	fmt.Fprintf(&b, "    AccessTTL:    %s\n", c.JWT.AccessTTL)
	fmt.Fprintf(&b, "    RefreshTTL:   %s\n", c.JWT.RefreshTTL)
	fmt.Fprintf(&b, "    Issuer:       %s\n", c.JWT.Issuer)
	fmt.Fprintf(&b, "  CORS:\n")
	fmt.Fprintf(&b, "    Origins:      %v\n", c.CORS.AllowedOrigins)
	fmt.Fprintf(&b, "    Methods:      %v\n", c.CORS.AllowedMethods)
	fmt.Fprintf(&b, "    MaxAge:       %s\n", c.CORS.MaxAge)
	fmt.Fprintf(&b, "  Cookie:\n")
	fmt.Fprintf(&b, "    Domain:       %s\n", c.Cookie.Domain)
	fmt.Fprintf(&b, "    Secure:       %v\n", c.Cookie.Secure)
	fmt.Fprintf(&b, "    HTTPOnly:     %v\n", c.Cookie.HTTPOnly)
	fmt.Fprintf(&b, "    SameSite:     %s\n", c.Cookie.SameSite)
	fmt.Fprintf(&b, "    Path:         %s\n", c.Cookie.Path)
	return b.String()
}

// --- Internal helpers ---

func parseEnvironment(s string) Environment {
	switch strings.ToLower(s) {
	case "production", "prod":
		return EnvProduction
	case "staging", "stage":
		return EnvStaging
	case "test":
		return EnvTest
	default:
		return EnvDevelopment
	}
}

// envDefault returns devValue for development/test, prodValue for staging/production.
func envDefault(env Environment, devValue, prodValue string) string {
	switch env {
	case EnvProduction, EnvStaging:
		return prodValue
	default:
		return devValue
	}
}

func getEnv(key, defaultValue string) string {
	if value, exists := os.LookupEnv(key); exists {
		return value
	}
	return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := strconv.Atoi(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func getEnvBool(key string, defaultValue bool) bool {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := strconv.ParseBool(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func getEnvDuration(key string, defaultValue time.Duration) time.Duration {
	valueStr, exists := os.LookupEnv(key)
	if !exists {
		return defaultValue
	}
	value, err := time.ParseDuration(valueStr)
	if err != nil {
		return defaultValue
	}
	return value
}

func parseCSV(value string) []string {
	if value == "" {
		return nil
	}
	parts := strings.Split(value, ",")
	result := make([]string, 0, len(parts))
	for _, part := range parts {
		trimmed := strings.TrimSpace(part)
		if trimmed != "" {
			result = append(result, trimmed)
		}
	}
	return result
}

func maskBytes(data []byte) string {
	if len(data) == 0 {
		return "[not set]"
	}
	return fmt.Sprintf("[%d bytes]", len(data))
}

func maskDatabaseURL(rawURL string) string {
	if rawURL == "" {
		return "[not set]"
	}
	idx := strings.Index(rawURL, "://")
	if idx < 0 {
		return "[set]"
	}
	rest := rawURL[idx+3:]
	atIdx := strings.Index(rest, "@")
	if atIdx < 0 {
		return rawURL
	}
	colonIdx := strings.Index(rest[:atIdx], ":")
	if colonIdx < 0 {
		return rawURL
	}
	return rawURL[:idx+3+colonIdx+1] + "****" + rawURL[idx+3+atIdx:]
}
```

### File: internal/config/config_test.go

```go
package config

import (
	"os"
	"strings"
	"testing"
	"time"
)

// validEnvVars returns a complete set of valid environment variables.
// Tests that need to check specific failures can override individual values.
func validEnvVars() map[string]string {
	return map[string]string{
		"APP_ENV":              "development",
		"PORT":                 "8080",
		"DATABASE_URL":         "postgres://user:pass@localhost:5432/testdb?sslmode=disable",
		"JWT_SIGNING_KEY":      "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2a3b4c5d6a7b8c9d0e1f2",
		"JWT_ACCESS_TTL":       "15m",
		"JWT_REFRESH_TTL":      "168h",
		"CORS_ORIGINS":         "http://localhost:4200",
		"LOG_LEVEL":            "debug",
		"COOKIE_DOMAIN":        "localhost",
		"COOKIE_SECURE":        "false",
		"COOKIE_SAMESITE":      "lax",
		"DB_MAX_OPEN_CONNS":    "25",
		"DB_MAX_IDLE_CONNS":    "5",
		"DB_CONN_MAX_LIFETIME": "5m",
		"DB_CONN_MAX_IDLE_TIME": "1m",
	}
}

// setAll sets all environment variables from the map.
func setAll(t *testing.T, vars map[string]string) {
	t.Helper()
	// First, clear all known config keys
	for key := range validEnvVars() {
		t.Setenv(key, "")
		os.Unsetenv(key)
	}
	// Then set the provided values
	for key, val := range vars {
		t.Setenv(key, val)
	}
}

func TestLoad_CompleteValidConfig(t *testing.T) {
	setAll(t, validEnvVars())

	cfg, err := Load()
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	// Verify all parsed values
	if cfg.Environment != EnvDevelopment {
		t.Errorf("Environment: got %s, want development", cfg.Environment)
	}
	if cfg.Server.Port != "8080" {
		t.Errorf("Port: got %s, want 8080", cfg.Server.Port)
	}
	if cfg.Database.URL != "postgres://user:pass@localhost:5432/testdb?sslmode=disable" {
		t.Errorf("Database.URL: got %s, want postgres://...", cfg.Database.URL)
	}
	if len(cfg.JWT.SigningKey) != 32 {
		t.Errorf("JWT.SigningKey: got %d bytes, want 32", len(cfg.JWT.SigningKey))
	}
	if cfg.JWT.AccessTTL != 15*time.Minute {
		t.Errorf("JWT.AccessTTL: got %s, want 15m", cfg.JWT.AccessTTL)
	}
	if cfg.JWT.RefreshTTL != 168*time.Hour {
		t.Errorf("JWT.RefreshTTL: got %s, want 168h", cfg.JWT.RefreshTTL)
	}
	if len(cfg.CORS.AllowedOrigins) != 1 || cfg.CORS.AllowedOrigins[0] != "http://localhost:4200" {
		t.Errorf("CORS.AllowedOrigins: got %v, want [http://localhost:4200]", cfg.CORS.AllowedOrigins)
	}
	if cfg.Cookie.Secure != false {
		t.Errorf("Cookie.Secure: got %v, want false", cfg.Cookie.Secure)
	}
}

func TestLoad_MissingDatabaseURL(t *testing.T) {
	vars := validEnvVars()
	delete(vars, "DATABASE_URL")
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for missing DATABASE_URL")
	}
	if !strings.Contains(err.Error(), "DATABASE_URL") {
		t.Errorf("error should mention DATABASE_URL: %s", err.Error())
	}
}

func TestLoad_MissingJWTKey(t *testing.T) {
	vars := validEnvVars()
	delete(vars, "JWT_SIGNING_KEY")
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for missing JWT_SIGNING_KEY")
	}
	if !strings.Contains(err.Error(), "JWT_SIGNING_KEY") {
		t.Errorf("error should mention JWT_SIGNING_KEY: %s", err.Error())
	}
}

func TestLoad_MultipleErrors(t *testing.T) {
	vars := validEnvVars()
	delete(vars, "DATABASE_URL")
	delete(vars, "JWT_SIGNING_KEY")
	delete(vars, "CORS_ORIGINS")
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for multiple missing fields")
	}

	errStr := err.Error()
	// All three errors should be reported
	if !strings.Contains(errStr, "DATABASE_URL") {
		t.Errorf("error should mention DATABASE_URL: %s", errStr)
	}
	if !strings.Contains(errStr, "JWT_SIGNING_KEY") {
		t.Errorf("error should mention JWT_SIGNING_KEY: %s", errStr)
	}
	if !strings.Contains(errStr, "CORS_ORIGINS") {
		t.Errorf("error should mention CORS_ORIGINS: %s", errStr)
	}
}

func TestLoad_InvalidPortRange(t *testing.T) {
	vars := validEnvVars()
	vars["PORT"] = "70000"
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for out-of-range port")
	}
	if !strings.Contains(err.Error(), "PORT") {
		t.Errorf("error should mention PORT: %s", err.Error())
	}
}

func TestLoad_CORSWildcardProduction(t *testing.T) {
	vars := validEnvVars()
	vars["APP_ENV"] = "production"
	vars["CORS_ORIGINS"] = "*"
	vars["COOKIE_SECURE"] = "true"
	vars["COOKIE_DOMAIN"] = "example.com"
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for CORS wildcard in production")
	}
	if !strings.Contains(err.Error(), "CORS") {
		t.Errorf("error should mention CORS: %s", err.Error())
	}
}

func TestLoad_SameSiteNoneRequiresSecure(t *testing.T) {
	vars := validEnvVars()
	vars["COOKIE_SAMESITE"] = "none"
	vars["COOKIE_SECURE"] = "false"
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error for SameSite=none without Secure")
	}
	if !strings.Contains(err.Error(), "COOKIE_SECURE") {
		t.Errorf("error should mention COOKIE_SECURE: %s", err.Error())
	}
}

func TestLoad_AccessTTLExceedsRefreshTTL(t *testing.T) {
	vars := validEnvVars()
	vars["JWT_ACCESS_TTL"] = "30m"
	vars["JWT_REFRESH_TTL"] = "15m" // Shorter than access -- invalid
	setAll(t, vars)

	_, err := Load()
	if err == nil {
		t.Fatal("expected error when access TTL >= refresh TTL")
	}
	if !strings.Contains(err.Error(), "shorter") {
		t.Errorf("error should mention 'shorter': %s", err.Error())
	}
}

func TestLoad_ProductionDefaults(t *testing.T) {
	vars := validEnvVars()
	vars["APP_ENV"] = "production"
	vars["COOKIE_DOMAIN"] = "example.com"
	// Do not set LOG_LEVEL -- should default to "info" in production
	delete(vars, "LOG_LEVEL")
	// Cookie.Secure should default to true in production
	delete(vars, "COOKIE_SECURE")
	setAll(t, vars)

	cfg, err := Load()
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	if cfg.Server.LogLevel != "info" {
		t.Errorf("expected production default LogLevel=info, got %s", cfg.Server.LogLevel)
	}
	if cfg.Cookie.Secure != true {
		t.Errorf("expected production default Cookie.Secure=true, got %v", cfg.Cookie.Secure)
	}
}

func TestConfig_String_MasksSecrets(t *testing.T) {
	cfg := &Config{
		Environment: EnvDevelopment,
		Server: ServerConfig{Port: "8080", LogLevel: "debug",
			ReadTimeout: 15 * time.Second, WriteTimeout: 15 * time.Second,
			IdleTimeout: 60 * time.Second},
		Database: DatabaseConfig{
			URL:          "postgres://admin:super_secret_password@db.example.com:5432/app",
			MaxOpenConns: 25, MaxIdleConns: 5,
			ConnMaxLifetime: 5 * time.Minute, ConnMaxIdleTime: 1 * time.Minute,
		},
		JWT: JWTConfig{
			SigningKey: make([]byte, 32), AccessTTL: 15 * time.Minute,
			RefreshTTL: 168 * time.Hour, Issuer: "krafty-core",
		},
		CORS: CORSConfig{
			AllowedOrigins: []string{"http://localhost:4200"},
			AllowedMethods: []string{"GET", "POST"},
			MaxAge:         12 * time.Hour,
		},
		Cookie: CookieConfig{
			Domain: "localhost", Secure: false, HTTPOnly: true,
			SameSite: "lax", Path: "/",
		},
	}

	output := cfg.String()

	// Password should be masked
	if strings.Contains(output, "super_secret_password") {
		t.Error("String() should mask database password")
	}
	if !strings.Contains(output, "****") {
		t.Error("String() should contain masked password indicator")
	}

	// JWT key should be masked
	if !strings.Contains(output, "[32 bytes]") {
		t.Error("String() should show JWT key as '[32 bytes]'")
	}
}

func TestConfig_IsDevelopment(t *testing.T) {
	cfg := &Config{Environment: EnvDevelopment}
	if !cfg.IsDevelopment() {
		t.Error("expected IsDevelopment() to return true")
	}
	if cfg.IsProduction() {
		t.Error("expected IsProduction() to return false")
	}
}

func TestConfig_IsProduction(t *testing.T) {
	cfg := &Config{Environment: EnvProduction}
	if !cfg.IsProduction() {
		t.Error("expected IsProduction() to return true")
	}
	if cfg.IsDevelopment() {
		t.Error("expected IsDevelopment() to return false")
	}
}
```

### File: main.go (Using the Config Package)

```go
package main

import (
	"context"
	"fmt"
	"log"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"yourproject/internal/config"
)

func main() {
	// ===== Step 1: Load Configuration =====
	cfg, err := config.Load()
	if err != nil {
		// Print the error and exit immediately.
		// Do not start the server with invalid configuration.
		fmt.Fprintf(os.Stderr, "FATAL: %v\n", err)
		os.Exit(1)
	}

	// Log the configuration (secrets are masked by the String() method)
	fmt.Println(cfg)

	// ===== Step 2: Set Up Structured Logger =====
	var logLevel slog.Level
	switch cfg.Server.LogLevel {
	case "debug":
		logLevel = slog.LevelDebug
	case "warn":
		logLevel = slog.LevelWarn
	case "error":
		logLevel = slog.LevelError
	default:
		logLevel = slog.LevelInfo
	}

	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: logLevel,
	}))
	slog.SetDefault(logger)

	logger.Info("configuration loaded",
		"environment", cfg.Environment,
		"port", cfg.Server.Port,
	)

	// ===== Step 3: Set Up Database =====
	// db, err := sql.Open("postgres", cfg.Database.URL)
	// if err != nil {
	//     logger.Error("failed to open database", "error", err)
	//     os.Exit(1)
	// }
	// defer db.Close()
	//
	// db.SetMaxOpenConns(cfg.Database.MaxOpenConns)
	// db.SetMaxIdleConns(cfg.Database.MaxIdleConns)
	// db.SetConnMaxLifetime(cfg.Database.ConnMaxLifetime)
	// db.SetConnMaxIdleTime(cfg.Database.ConnMaxIdleTime)
	//
	// if err := db.PingContext(context.Background()); err != nil {
	//     logger.Error("failed to ping database", "error", err)
	//     os.Exit(1)
	// }
	// logger.Info("database connected")

	// ===== Step 4: Set Up Router =====
	mux := http.NewServeMux()

	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, `{"status":"ok","environment":"%s"}`, cfg.Environment)
	})

	mux.HandleFunc("GET /", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintf(w, `{"message":"Welcome to krafty-core","version":"1.0.0"}`)
	})

	// ===== Step 5: Create and Start Server =====
	server := &http.Server{
		Addr:         ":" + cfg.Server.Port,
		Handler:      mux,
		ReadTimeout:  cfg.Server.ReadTimeout,
		WriteTimeout: cfg.Server.WriteTimeout,
		IdleTimeout:  cfg.Server.IdleTimeout,
	}

	// Start server in a goroutine
	errCh := make(chan error, 1)
	go func() {
		logger.Info("server starting", "addr", server.Addr)
		errCh <- server.ListenAndServe()
	}()

	// ===== Step 6: Graceful Shutdown =====
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-errCh:
		if err != nil && err != http.ErrServerClosed {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	case sig := <-sigCh:
		logger.Info("received shutdown signal", "signal", sig)
	}

	// Give in-flight requests 10 seconds to complete
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		logger.Error("shutdown error", "error", err)
		os.Exit(1)
	}

	logger.Info("server stopped gracefully")
}
```

### File: .env.example

```bash
# Application Environment
# Valid: development, staging, production, test
APP_ENV=development

# Server
PORT=8080
SERVER_READ_TIMEOUT=15s
SERVER_WRITE_TIMEOUT=15s
SERVER_IDLE_TIMEOUT=60s
LOG_LEVEL=debug

# Database (PostgreSQL)
# Format: postgres://user:password@host:port/dbname?sslmode=disable
DATABASE_URL=postgres://user:password@localhost:5432/krafty_dev?sslmode=disable

# JWT Authentication
# Generate a 32-byte (256-bit) hex-encoded key:
#   openssl rand -hex 32
JWT_SIGNING_KEY=<run: openssl rand -hex 32>
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=168h
JWT_ISSUER=krafty-core

# CORS (comma-separated origins, no trailing slashes)
CORS_ORIGINS=http://localhost:4200

# CORS preflight cache duration
CORS_MAX_AGE=12h

# Cookies
COOKIE_DOMAIN=localhost
COOKIE_SECURE=false
COOKIE_HTTPONLY=true
COOKIE_SAMESITE=lax
COOKIE_PATH=/

# Database Connection Pool
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=5m
DB_CONN_MAX_IDLE_TIME=1m
```

### Project Structure

```
krafty-core/
├── cmd/
│   └── server/
│       └── main.go              # Application entry point
├── internal/
│   ├── config/
│   │   ├── config.go            # Config struct + Load() + validate()
│   │   └── config_test.go       # Comprehensive config tests
│   ├── database/
│   │   └── database.go          # DB connection using config.DatabaseConfig
│   ├── auth/
│   │   └── jwt.go               # JWT service using config.JWTConfig
│   └── middleware/
│       ├── cors.go              # CORS middleware using config.CORSConfig
│       └── cookie.go            # Cookie helpers using config.CookieConfig
├── .env                          # Local dev values (git-ignored)
├── .env.example                  # Template (committed to git)
├── .gitignore
├── go.mod
└── go.sum
```

### The Complete Flow

```
Application Startup:

1. main.go calls config.Load()
2. config.Load():
   a. Loads .env file (godotenv.Load, ignores if missing)
   b. Reads all environment variables with sensible defaults
   c. Parses typed values (durations, booleans, integers, hex keys)
   d. Runs validate() which checks ALL rules and collects ALL errors
   e. Returns *Config or error

3. If config.Load() returns error:
   → Print error to stderr, exit(1)
   → Operator sees ALL configuration problems at once

4. If config.Load() succeeds:
   → Log masked config (secrets hidden)
   → Initialize database with cfg.Database
   → Initialize JWT service with cfg.JWT
   → Initialize CORS middleware with cfg.CORS
   → Initialize cookie helpers with cfg.Cookie
   → Start HTTP server with cfg.Server
   → Wait for shutdown signal

This is the pattern used by virtually every production Go API.
```

---

## 14. Key Takeaways

1. **Store configuration in environment variables, not in code.** This is the third factor of the Twelve-Factor App methodology. Your binary should be identical across all environments. Only the environment variables change.

2. **Use a `.env` file for local development, never for production.** Production configuration comes from the deployment system (Kubernetes Secrets, Vault, cloud provider secrets managers). The `.env` file is a convenience for developers.

3. **Always commit a `.env.example` file.** This documents every environment variable your application needs. New team members should be able to `cp .env.example .env`, fill in a few values, and run the application.

4. **Build a Config struct.** A centralized, typed configuration struct provides compile-time safety, IDE autocomplete, and a single place to see everything your application needs. This is fundamentally superior to scattered `process.env` calls.

5. **Validate all configuration at startup.** Fail fast, fail loudly. Report ALL errors at once so operators can fix everything in a single deploy cycle, not one error per restart.

6. **Never default secrets.** `DATABASE_URL`, `JWT_SIGNING_KEY`, and other sensitive values must be explicitly set. A default secret creates a false sense of security and a real vulnerability if someone forgets to override it.

7. **Use `time.Duration` for timeouts and TTLs.** Go's duration parsing (`"15m"`, `"168h"`, `"500ms"`) is built into the standard library and produces properly typed values. This is one of Go's most practical advantages over dynamically typed languages.

8. **Mask secrets in logs.** Implement a `String()` method on your Config that replaces passwords and keys with `****` or `[N bytes]`. Never print raw configuration that contains secrets.

9. **Test your configuration loading.** Config code has bugs like any other code. Test the happy path, missing required fields, invalid values, edge cases in parsing, and default values. Use `t.Setenv` for clean environment management in tests.

10. **Use functional options for library-level configuration.** When building reusable components, the options pattern (`WithPort("8080")`, `WithTimeout(5*time.Second)`) is backwards-compatible, self-documenting, and idiomatic Go.

11. **Configure your database pool explicitly.** Never use the defaults for `MaxOpenConns` (unlimited) or `MaxIdleConns` (2). Set `MaxOpenConns` based on your database's `max_connections` divided by the number of application instances. Set `ConnMaxLifetime` shorter than any firewall or load balancer timeout.

12. **Separate config by concern with nested structs.** Group related settings (`ServerConfig`, `DatabaseConfig`, `JWTConfig`, `CORSConfig`, `CookieConfig`) so each component receives only the configuration it needs. This follows the principle of least privilege for data access.

---

## 15. Practice Exercises

### Exercise 1: Basic Config Package

Build a configuration package for a simple blog API with:
- `PORT` (default: 3000)
- `DATABASE_URL` (required, no default)
- `API_KEY` (required, at least 16 characters)
- `LOG_LEVEL` (default: info, valid: debug/info/warn/error)
- `MAX_UPLOAD_SIZE` (default: 5MB, parsed from string like "5MB" or "10MB")

Implement `Load()` with validation that reports all errors. Write tests for every validation rule.

### Exercise 2: Multi-Environment Config

Extend Exercise 1 to support three environments:
- **development**: permissive defaults, verbose logging
- **staging**: production-like but with debug endpoints enabled
- **production**: strict validation, no debug endpoints

Add a `DEBUG_ENDPOINTS` boolean field that defaults to `true` in development, `true` in staging, and `false` in production. Add validation that ensures `DEBUG_ENDPOINTS` is `false` when `APP_ENV=production`.

### Exercise 3: Config with Feature Flags

Build a config package that includes feature flags:

```bash
FEATURES=new_dashboard,beta_api,dark_mode
```

Parse this into a `map[string]bool` and add methods:
- `IsFeatureEnabled(name string) bool`
- `EnabledFeatures() []string`

Write tests that verify feature flag parsing, including edge cases like empty strings, duplicate flags, and whitespace.

### Exercise 4: Database Config Validator

Write a `DatabaseConfig` struct with pool settings and a `validate()` method that enforces:
- `MaxOpenConns` is between 1 and 500
- `MaxIdleConns` is between 0 and `MaxOpenConns`
- `ConnMaxLifetime` is between 1 minute and 1 hour
- `ConnMaxIdleTime` is between 30 seconds and `ConnMaxLifetime`
- If the `DATABASE_URL` contains `sslmode=disable`, warn (do not error) unless `APP_ENV=development`

Write table-driven tests with at least 10 test cases covering valid and invalid combinations.

### Exercise 5: Secrets Rotation Simulation

Build a config package that supports hot-reloading secrets without restarting the application:

1. On startup, load `JWT_SIGNING_KEY` from the environment.
2. Every 60 seconds, check if the environment variable has changed (simulating a secrets manager rotation).
3. If changed, decode the new hex key and atomically swap it using `sync.RWMutex` or `atomic.Value`.
4. Provide a `GetJWTKey() []byte` method that is safe for concurrent use.

Write a test that starts the config, changes the environment variable, triggers a reload, and verifies the new key is returned.

### Exercise 6: Configuration from Multiple Sources

Build a config loader that merges configuration from three sources in priority order:
1. **Environment variables** (highest priority)
2. **Config file** (`config.yaml` or `config.json`)
3. **Default values** (lowest priority)

Use `encoding/json` or a YAML library. Implement the merge logic manually (no third-party config libraries). Write tests that verify the priority ordering works correctly.

### Exercise 7: Full Production Config Package

Build a complete config package for a "Task Management API" with:
- Server settings (port, timeouts, log level)
- PostgreSQL database (URL, pool settings)
- Redis cache (URL, pool size, TTL defaults)
- JWT authentication (signing key, access/refresh TTLs)
- CORS (allowed origins, methods, headers)
- Rate limiting (requests per minute, burst size)
- SMTP email (host, port, username, password, from address)
- File storage (provider: "local" or "s3", local path or S3 bucket/region/access key)

Requirements:
- All secrets masked in `String()` output
- All fields validated with descriptive errors
- All errors collected and reported at once
- Comprehensive test suite with 90%+ coverage
- `.env.example` file documenting every variable
- Works with both `.env` files and real environment variables

### Exercise 8: Config Diff Tool

Build a CLI tool that compares two `.env` files and reports:
- Variables in file A but not in file B
- Variables in file B but not in file A
- Variables with different values (mask any that contain "KEY", "SECRET", or "PASSWORD" in the name)

Use `godotenv.Read()` (which returns `map[string]string` without setting env vars) to read the files. Write tests using temporary files.

### Exercise 9: Go vs Node.js Config Comparison

Build the same config loading system in both Go and Node.js:

Go:
- Config struct with typed fields
- `Load()` function with validation
- Tests

Node.js:
- Use `zod` or `envalid` for schema validation
- TypeScript for type safety
- Tests with Jest

Compare:
- Lines of code
- Type safety (compile-time vs runtime)
- Error messages (quality and completeness)
- Test coverage and test ergonomics
- Dependency count (Go: godotenv only; Node.js: dotenv + zod/envalid + typescript)

Document your findings about the tradeoffs between the two approaches.
