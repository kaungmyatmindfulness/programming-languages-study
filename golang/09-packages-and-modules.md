# Chapter 9: Packages, Modules & Dependency Management

> **Level:** Intermediate
>
> **Prerequisites:** You should be comfortable with Go's basic syntax, types, functions, and structs (Chapters 1-8). Familiarity with Node.js and npm is helpful for the comparisons but not required.

---

## Table of Contents

1. [Packages: The Building Block of Go Code](#1-packages-the-building-block-of-go-code)
2. [Importing Packages](#2-importing-packages)
3. [Exported vs Unexported Identifiers](#3-exported-vs-unexported-identifiers)
4. [Go Modules](#4-go-modules)
5. [Dependency Management](#5-dependency-management)
6. [Internal Packages](#6-internal-packages)
7. [Standard Library Tour](#7-standard-library-tour)
8. [Creating and Publishing Modules](#8-creating-and-publishing-modules)
9. [Workspace Mode](#9-workspace-mode)
10. [Build Tags and Conditional Compilation](#10-build-tags-and-conditional-compilation)
11. [Go vs Node.js: Full Comparison Table](#11-go-vs-nodejs-full-comparison-table)
12. [Key Takeaways](#12-key-takeaways)
13. [Practice Exercises](#13-practice-exercises)

---

## 1. Packages: The Building Block of Go Code

### What Is a Package?

A **package** is Go's fundamental unit of code organization. Every `.go` file begins with a `package` declaration, and every Go program starts execution in a package named `main`.

```go
package main // This file belongs to the "main" package

import "fmt"

func main() {
    fmt.Println("Hello from package main")
}
```

A package is simply **a directory containing one or more `.go` files that all share the same `package` declaration at the top**. That is the entire mental model.

### The One-Package-Per-Directory Rule

Go enforces a strict rule: **all `.go` files in a single directory must declare the same package name** (with the sole exception of `_test.go` files, which may use a `_test` suffix for black-box testing).

```
myproject/
  main.go          // package main
  utils/
    strings.go     // package utils
    numbers.go     // package utils   <-- MUST match strings.go
    strings_test.go // package utils OR package utils_test (testing exception)
```

If you tried to put `package helpers` in `utils/numbers.go` while `utils/strings.go` says `package utils`, the Go compiler will refuse to build. No exceptions.

### WHY Does Go Have This Rule?

This is a deliberate design choice rooted in Go's philosophy of **simplicity and clarity**:

1. **No ambiguity.** When you import `"myproject/utils"`, you know exactly what package you get. There is no resolution algorithm to reason about, no index file to trace through, no barrel exports to unravel.

2. **The directory IS the package.** The file system is the source of truth. You never need to look inside a file to figure out what package it belongs to -- just look at which directory it sits in.

3. **Compilation speed.** The Go compiler can process one directory (package) as a single compilation unit. This enables Go's famously fast compilation because the dependency graph is determined by directory structure, not by parsing import chains across scattered files.

4. **Merge conflicts are rare.** Because you can split a package across multiple files in the same directory, two developers can add new functions to the same package without touching the same file. There is no single `index.js` bottleneck.

### Comparison with JavaScript's Module System

In JavaScript, the relationship between files and modules has evolved through multiple competing systems:

```
JavaScript Module History:
  CommonJS (CJS)  -->  require('...')     // Node.js original
  AMD              -->  define([...])      // RequireJS, browsers
  UMD              -->  (function(root){}) // Universal, both
  ESM              -->  import ... from    // Modern standard
```

```javascript
// JavaScript (ESM): each FILE is a module
// utils/strings.js -- this is its own module
export function capitalize(s) { return s[0].toUpperCase() + s.slice(1); }

// utils/numbers.js -- this is a separate module
export function double(n) { return n * 2; }

// utils/index.js -- barrel file to re-export everything
export { capitalize } from './strings.js';
export { double } from './numbers.js';
```

```go
// Go: the entire DIRECTORY is the package
// utils/strings.go
package utils

func Capitalize(s string) string { return strings.ToUpper(s[:1]) + s[1:] }

// utils/numbers.go
package utils

func Double(n int) int { return n * 2 }

// No index file needed. Both functions are available when you import "myproject/utils".
```

| Aspect | Go | JavaScript (ESM) |
|---|---|---|
| Unit of organization | Directory (package) | File (module) |
| Declaration | `package foo` at top of file | Implicit (every file is a module) |
| Multiple files, one unit? | Yes (all files in a dir) | No (need barrel/index file) |
| Circular dependencies | Compile-time error | Allowed (can cause bugs) |
| File = module? | No. File = part of a package | Yes |

**The key insight:** Go eliminated an entire class of "how do I organize my exports" problems by making the directory the atomic unit. You never need barrel files, re-exports, or `index.go`.

### Package Naming Conventions

Go has strong conventions (not just suggestions) for package names:

```go
// GOOD: short, lowercase, single-word
package http
package json
package sync
package user

// BAD: don't do these
package httpUtils    // no camelCase
package http_utils   // no underscores
package HTTPHandler  // no uppercase
package utilities    // too generic
package common       // too vague
package base         // meaningless
```

The package name becomes part of every qualified reference: `http.Get()`, `json.Marshal()`, `sync.Mutex`. This is why short names matter -- you will type them constantly.

**Avoid stutter.** If your package is named `user`, do not name a type `UserService`. Call it `Service`, so the caller writes `user.Service` -- which reads naturally. Compare `user.UserService` (stutters) vs `user.Service` (clean).

---

## 2. Importing Packages

### Basic Import Syntax

```go
// Single import
import "fmt"

// Grouped import (preferred style)
import (
    "fmt"
    "strings"
    "net/http"

    "github.com/gorilla/mux"    // third-party, separated by blank line
)
```

The convention is to group imports into blocks separated by blank lines:
1. Standard library packages
2. Third-party packages
3. Internal/project packages

Tools like `goimports` and `gofmt` will automatically sort and group these for you.

### Aliased Imports

You can give a package a local alias:

```go
import (
    "fmt"

    gohttp "net/http"                    // alias to avoid confusion
    myjson "encoding/json"              // alias for clarity
    v2 "github.com/example/lib/v2/api"  // alias for version clarity
)

func main() {
    gohttp.ListenAndServe(":8080", nil)
    data, _ := myjson.Marshal(map[string]int{"a": 1})
    fmt.Println(string(data))
}
```

Aliased imports are most useful when:
- Two packages have the same name (e.g., your `json` package and `encoding/json`)
- The auto-derived name from the import path is awkward
- You want to make the version explicit

### Dot Imports

A dot import merges the package's exported identifiers into the current file's namespace:

```go
import . "fmt"

func main() {
    Println("No 'fmt.' prefix needed") // Works, but generally frowned upon
}
```

**You should almost never use dot imports.** They obscure where identifiers come from and can cause name collisions. The only common use is in test files for domain-specific testing frameworks, and even that is debated.

### Blank Imports (The Underscore Import)

```go
import (
    "database/sql"
    _ "github.com/lib/pq" // blank import: imported for side effects only
)
```

A blank import executes the package's `init()` functions but does not make its names available. If you removed the underscore, the compiler would complain about an unused import.

### WHY Do Blank Imports Exist?

Go refuses to compile code with unused imports. This is a strict rule with no exceptions -- it keeps codebases clean. But some packages need to be imported purely for their **side effects**: code that runs in `init()` functions when the package loads.

The most common example is **database drivers**:

```go
// github.com/lib/pq/conn.go (inside the driver package)
func init() {
    sql.Register("postgres", &Driver{})
    // This registers the postgres driver with database/sql
    // so that sql.Open("postgres", connStr) works
}
```

```go
// Your code
import (
    "database/sql"
    _ "github.com/lib/pq" // Registers the "postgres" driver via init()
)

func main() {
    // This works because pq's init() registered the driver
    db, err := sql.Open("postgres", "host=localhost dbname=mydb")
    // ...
}
```

Other common blank imports:
- `_ "image/png"` -- registers PNG decoder with the `image` package
- `_ "image/jpeg"` -- registers JPEG decoder
- `_ "net/http/pprof"` -- registers profiling HTTP handlers
- `_ "expvar"` -- registers the `/debug/vars` HTTP handler

### The init() Function

Any package can define one or more `init()` functions. They run automatically when the package is imported, before `main()` executes:

```go
package mypackage

import "fmt"

func init() {
    fmt.Println("First init runs")
}

func init() {
    fmt.Println("Second init runs") // Yes, you can have multiple init() in one file
}

// You can also have init() functions in different files within the same package.
// Execution order within a package: alphabetical by filename, then top-to-bottom.
```

**Init order across packages:** determined by the import dependency graph. If package A imports B, then B's `init()` runs before A's `init()`. The `main` package's `init()` runs last, right before `main()`.

### Comparison with JavaScript Imports

```javascript
// JavaScript (ESM)
import express from 'express';           // default import
import { Router } from 'express';        // named import
import * as path from 'path';            // namespace import
import './side-effects.js';              // side-effect-only import (like Go's blank import)

// JavaScript (CJS)
const express = require('express');
const { Router } = require('express');
```

```go
// Go equivalents
import "github.com/gin-gonic/gin"       // import full package (like namespace import)
// No "named import" -- you always get the whole package
// Use gin.Router, gin.Default, etc.
import _ "some/package"                  // side-effect-only import
```

| Aspect | Go | JavaScript |
|---|---|---|
| Import granularity | Whole package only | Individual exports or whole module |
| Tree shaking | Done by compiler (dead code elimination) | Done by bundler (webpack, rollup) |
| Side-effect imports | `_ "pkg"` | `import './file.js'` |
| Unused import | Compile error | Linter warning (if configured) |
| Circular imports | Compile error | Allowed (can cause `undefined`) |
| Dynamic imports | Not possible | `import('./module.js')` returns Promise |

---

## 3. Exported vs Unexported Identifiers

### The Capitalization Rule

In Go, **visibility is determined by the first letter of an identifier's name**:

- **Uppercase first letter = Exported** (visible outside the package)
- **Lowercase first letter = Unexported** (visible only within the package)

This applies to everything: functions, types, struct fields, methods, constants, variables.

```go
package user

// Exported: visible to other packages
type User struct {
    Name  string   // Exported field
    Email string   // Exported field
    age   int      // Unexported field -- other packages cannot access this
}

// Exported function
func NewUser(name, email string, age int) User {
    return User{Name: name, Email: email, age: age}
}

// Unexported function -- only callable within the `user` package
func validate(email string) bool {
    return strings.Contains(email, "@")
}

// Exported method
func (u User) String() string {
    return fmt.Sprintf("%s <%s>", u.Name, u.Email)
}

// Unexported method
func (u User) isAdult() bool {
    return u.age >= 18
}
```

```go
package main

import "myproject/user"

func main() {
    u := user.NewUser("Alice", "alice@example.com", 30)  // OK: NewUser is exported
    fmt.Println(u.Name)                                    // OK: Name is exported
    fmt.Println(u.Email)                                   // OK: Email is exported
    // fmt.Println(u.age)                                  // COMPILE ERROR: age is unexported
    // user.validate("test")                               // COMPILE ERROR: validate is unexported
    // u.isAdult()                                         // COMPILE ERROR: isAdult is unexported
}
```

### WHY Capitalization Instead of a Keyword?

Most languages use explicit keywords for visibility:

```java
// Java
public class User {
    private int age;
    public String name;
    protected void validate() { }
}
```

```javascript
// JavaScript
export function createUser() { }     // exported
function validate() { }              // not exported (module-scoped)
export default class User { }        // default export
```

```go
// Go: no keyword needed
func CreateUser() { }  // Exported (uppercase C)
func validate() { }    // Unexported (lowercase v)
type User struct { }   // Exported (uppercase U)
```

Go chose capitalization for several deliberate reasons:

1. **Visibility at a glance.** You can scan Go code and instantly know what is part of the public API without searching for keywords. When you see `func ProcessOrder(...)`, the uppercase P tells you immediately -- this is public. No need to scroll up to check for an `export` statement or a `public` keyword.

2. **No boilerplate.** Java requires `public` or `private` on virtually every declaration. JavaScript requires `export` on everything you want to expose. Go encodes this information in a single character you were already typing.

3. **Consistent across all identifiers.** The same rule works for types, functions, methods, struct fields, constants, and variables. No special syntax for any of them.

4. **Enforced by the compiler.** It is not a convention you might forget to follow -- the compiler treats casing as semantically meaningful. There is no way to "accidentally" export something.

5. **Refactoring signal.** Changing visibility is a deliberate act: renaming `processOrder` to `ProcessOrder` makes it clear in a code review that the API surface changed. In JavaScript, you could add `export` and it might slip past review.

The tradeoff: you cannot have two identifiers that differ only in case (`User` and `user` in the same scope would be confusing, though technically allowed for different purposes -- type vs variable).

### Comparison with JavaScript Export Patterns

```javascript
// JavaScript: Multiple export patterns to learn
// 1. Named exports
export const API_KEY = "...";
export function fetchData() { }
export class UserService { }

// 2. Default export (one per module)
export default class User { }

// 3. Re-exports
export { fetchData as getData } from './api.js';
export * from './utils.js';

// 4. CommonJS
module.exports = { fetchData, User };
module.exports.API_KEY = "...";
```

```go
// Go: One rule to learn
const APIKey = "..."          // Exported (uppercase)
const maxRetries = 3          // Unexported (lowercase)

func FetchData() { }          // Exported
func fetchData() { }          // This would be a different, unexported function

type UserService struct { }   // Exported
type cache struct { }         // Unexported
```

| Aspect | Go | JavaScript |
|---|---|---|
| Export mechanism | Capitalization | `export` keyword |
| Default export | N/A (not a concept) | `export default` |
| Number of patterns | 1 rule | 4+ patterns (named, default, re-export, CJS) |
| Granularity | Per-identifier | Per-identifier |
| Struct field visibility | Per-field (capitalization) | No built-in private fields (# syntax is newer) |
| Refactoring cost | Rename the identifier | Add/remove `export` keyword |
| Visibility levels | 2 (exported, unexported) | 2 per module, but also `#private` for classes |

### Struct Field Visibility and JSON

The capitalization rule interacts importantly with serialization:

```go
type User struct {
    Name  string `json:"name"`   // Exported field, serialized as "name"
    Email string `json:"email"`  // Exported field, serialized as "email"
    age   int    `json:"age"`    // Unexported! json.Marshal CANNOT see this field
}

u := User{Name: "Alice", Email: "alice@example.com", age: 30}
data, _ := json.Marshal(u)
fmt.Println(string(data))
// Output: {"name":"Alice","email":"alice@example.com"}
// Note: "age" is MISSING because json.Marshal cannot access unexported fields
```

This is a common gotcha for beginners. The `encoding/json` package lives in a different package than your struct, so it can only see exported fields. Use struct tags to control the JSON key names while keeping Go's uppercase convention:

```go
type User struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Age      int    `json:"age"`       // Exported, so json can see it
    password string                     // Intentionally unexported -- never serialize this
}
```

---

## 4. Go Modules

### What Is a Module?

A **module** is a collection of Go packages that are versioned and distributed together. A module is defined by a `go.mod` file at the root of the module's directory tree.

Think of it this way:
- A **package** = a directory of `.go` files
- A **module** = a collection of packages with a shared version and dependency tree
- A **repository** = usually contains one module (sometimes more)

### WHY Modules Replaced GOPATH

Before Go 1.11 (2018), Go used **GOPATH** -- a single workspace directory where ALL Go code lived:

```
$GOPATH/
  src/
    github.com/
      alice/
        projectA/
          main.go
        projectB/
          main.go
      bob/
        libraryX/
          lib.go
  bin/
  pkg/
```

GOPATH had serious problems:

1. **No versioning.** You got whatever version of a dependency happened to be in your GOPATH. If projectA needed library v1.2 and projectB needed v1.5, tough luck -- there was one copy.

2. **No isolation.** All projects shared the same dependency tree. Updating a library for one project could break another.

3. **No reproducibility.** `go get` fetched the latest commit from `master`. Two developers running `go get` a week apart might get different code. Builds were not reproducible.

4. **Forced directory structure.** Your code HAD to live under `$GOPATH/src/`. You could not keep projects in `~/projects/` or wherever you liked.

5. **Vendor hacks.** The community invented vendoring tools (godep, glide, dep) to work around GOPATH's limitations, but they were all unofficial and incompatible.

Modules solved all of these problems. With modules:
- Your code can live anywhere on your filesystem
- Each project has its own `go.mod` listing exact dependency versions
- Builds are reproducible via `go.sum` checksums
- Multiple versions of the same module can coexist

### Initializing a Module

```bash
# Create a new project anywhere on your filesystem
mkdir ~/projects/myapp
cd ~/projects/myapp

# Initialize the module
go mod init github.com/yourusername/myapp
```

This creates a `go.mod` file:

```
module github.com/yourusername/myapp

go 1.22
```

The module path (`github.com/yourusername/myapp`) is the **import path prefix** for all packages in this module. If you have a package in `./handlers/`, it would be imported as `github.com/yourusername/myapp/handlers`.

For projects you do not plan to publish, you can use any path:

```bash
go mod init myapp          # Simple, local-only
go mod init example.com/myapp  # Also fine
```

### The go.mod File

A real-world `go.mod` looks like this:

```
module github.com/yourusername/myapp

go 1.22

require (
    github.com/gorilla/mux v1.8.1
    github.com/lib/pq v1.10.9
    golang.org/x/sync v0.6.0
)

require (
    github.com/gorilla/handlers v1.5.2 // indirect
    github.com/felixge/httpsnoop v1.0.4 // indirect
)
```

Breaking it down:

- **`module`**: The module's import path.
- **`go`**: The minimum Go version required. This also controls which language features are available.
- **`require`**: Direct dependencies your code imports. Go separates direct and `// indirect` dependencies.
- **`// indirect`**: Dependencies of your dependencies. They are listed for reproducibility. You did not import them directly, but they are needed.

Other directives you might see:

```
exclude github.com/broken/lib v1.0.0   // Never use this version
replace github.com/original/lib => github.com/fork/lib v1.2.3   // Use a fork
replace github.com/original/lib => ../local-lib   // Use a local directory (development)
retract v1.0.0   // Module author says: do not use this version
```

### Semantic Versioning

Go modules use **strict semantic versioning** (semver):

```
v1.2.3
│ │ │
│ │ └── Patch: bug fixes, no API changes
│ └──── Minor: new features, backward compatible
└────── Major: breaking changes
```

The critical rule: **major versions v2+ must be in the module path**:

```
github.com/example/lib       // v0.x.x or v1.x.x
github.com/example/lib/v2    // v2.x.x
github.com/example/lib/v3    // v3.x.x
```

This means you can import v1 and v2 of the same library simultaneously:

```go
import (
    v1 "github.com/example/lib"
    v2 "github.com/example/lib/v2"
)
```

**WHY embed the major version in the path?** This is Go's solution to "dependency hell." In npm, if package A needs `lib@1.x` and package B needs `lib@2.x`, npm must figure out how to install both (hoisting, nesting, etc.). In Go, `lib` and `lib/v2` are simply different packages -- no conflict resolution needed.

### The go.sum File

`go.sum` contains cryptographic hashes of every dependency:

```
github.com/gorilla/mux v1.8.1 h1:TuMoUvkRETdXqU+a3jEXMiYBhiUJ...
github.com/gorilla/mux v1.8.1/go.mod h1:DVbg23sWSpFRCP0SfiEN6jmj5...
```

Each entry has two lines:
1. Hash of the **source code** of the module
2. Hash of the **go.mod file** of the module

**Purpose:** If someone tampers with a published module (replaces the code at a given version), the hash will not match and your build will fail. This is a security feature.

Go also operates a **checksum database** at `sum.golang.org` -- a public, append-only log of module hashes. When you download a module, Go checks its hash against this global database. Even if the module host is compromised, the checksum database will catch the inconsistency.

### Comparison: go.mod vs package.json

```json
// package.json (Node.js)
{
  "name": "@yourusername/myapp",
  "version": "1.0.0",
  "description": "My application",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "jest",
    "build": "tsc"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "typescript": "^5.3.3"
  }
}
```

```
// go.mod (Go)
module github.com/yourusername/myapp

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/lib/pq v1.10.9
)
```

| Aspect | go.mod | package.json |
|---|---|---|
| Format | Custom, minimal | JSON |
| Version ranges | Exact versions only | Ranges (`^`, `~`, `>=`) |
| Dev dependencies | No concept (tests are built-in) | `devDependencies` |
| Scripts | No concept (use Makefile or taskfile) | `scripts` section |
| Entry point | `package main` convention | `"main"` field |
| Lock file | `go.sum` (hashes only) | `package-lock.json` (full dependency tree) |
| Size | Typically small | Can be enormous |

Key differences in philosophy:
- Go's `go.mod` records **exact versions**, not ranges. There is no `^` or `~`.
- Go has **no dev dependencies**. Test code lives alongside production code in `_test.go` files, and test dependencies are just regular dependencies.
- Go has **no scripts section**. Build, test, and run commands are built into the `go` tool itself (`go build`, `go test`, `go run`).

---

## 5. Dependency Management

### go get: Adding Dependencies

```bash
# Add or update a dependency
go get github.com/gorilla/mux@v1.8.1

# Get the latest version
go get github.com/gorilla/mux@latest

# Get a specific commit (for debugging, not recommended for production)
go get github.com/gorilla/mux@abc1234

# Update a dependency to the latest minor/patch
go get -u github.com/gorilla/mux

# Update ALL dependencies
go get -u ./...

# Remove a dependency (set version to "none")
go get github.com/gorilla/mux@none
```

Unlike `npm install`, `go get` in modern Go (1.18+) only modifies `go.mod` and `go.sum`. It does not create a `vendor/` directory or a `node_modules/` equivalent by default. Downloaded modules live in the **module cache** at `$GOPATH/pkg/mod/` and are shared across all your projects.

### go mod tidy: The Essential Command

```bash
go mod tidy
```

This single command does everything:
1. **Adds** missing dependencies that your code imports but `go.mod` does not list
2. **Removes** dependencies that `go.mod` lists but your code does not use
3. **Updates** `go.sum` to match

You should run `go mod tidy` after:
- Adding or removing imports in your code
- Pulling changes from other developers
- Before committing changes

This is roughly equivalent to running `npm install` to sync `node_modules` with `package.json`, but cleaner because it also removes unused entries.

### go mod vendor: Vendoring Dependencies

```bash
go mod vendor
```

This copies all dependencies into a `vendor/` directory within your project:

```
myapp/
  go.mod
  go.sum
  main.go
  vendor/
    github.com/
      gorilla/
        mux/
          mux.go
          ...
    modules.txt
```

**When to vendor:**
- Your CI/CD environment has no internet access
- You want to audit all dependency source code
- You want your repository to be fully self-contained
- Corporate policy requires vendoring

Build with vendor: `go build -mod=vendor ./...`

### Minimum Version Selection (MVS)

This is one of Go's most important and most misunderstood design decisions.

**The problem:** If your app depends on Library A (which requires Util >= v1.2) and Library B (which requires Util >= v1.4), what version of Util do you install?

**npm's approach (Maximal Version Selection):**
Install the **latest** version that satisfies all constraints. If Util v1.9 is the latest, install v1.9 even though nobody asked for it. The reasoning: newer is usually better, has more bug fixes.

**Go's approach (Minimum Version Selection):**
Install the **minimum** version that satisfies all constraints. In this case, install Util v1.4 (the highest minimum requirement). NOT v1.9 -- nobody asked for v1.9.

```
Your App
  ├── Library A (requires Util >= v1.2)
  └── Library B (requires Util >= v1.4)

npm would install:  Util v1.9 (latest)
Go MVS installs:    Util v1.4 (minimum that satisfies both)
```

### WHY Minimum Version Selection?

MVS is counterintuitive at first, but it has profound advantages:

1. **Reproducibility without a lock file.** With MVS, the `go.mod` files alone determine the exact versions used. Two developers running `go mod tidy` will always get the same result, even without consulting a lock file. (`go.sum` exists for integrity verification, not version resolution.)

2. **No surprises.** Your build never changes unless YOU change something. A new release of a dependency does not affect your build until you explicitly run `go get -u`. In npm, `npm install` on a fresh machine might pull newer versions (within semver ranges) and introduce regressions.

3. **Simplicity.** The MVS algorithm is simple and fast (essentially: take the maximum of all minimum requirements). npm's resolution algorithm handles version ranges, peer dependencies, optional dependencies, and conflict resolution -- it is significantly more complex.

4. **Predictability over freshness.** The Go team's philosophy: you should consciously choose to update, not have updates happen as a side effect of installing something else.

**The tradeoff:** You must actively update dependencies to get bug fixes and security patches. Go does not do this for you. This is why `go get -u` and `govulncheck` (vulnerability checker) exist.

### Comparison: go get vs npm install

```bash
# Node.js
npm install express           # Add dependency, installs to node_modules/
npm install                   # Install all deps from package.json
npm update                    # Update within semver ranges
npm audit                     # Check for vulnerabilities
npm audit fix                 # Auto-fix vulnerabilities
npm ci                        # Clean install from lock file (CI)
npm prune                     # Remove unused packages

# Go
go get github.com/gin-gonic/gin  # Add dependency, updates go.mod
go mod download                   # Download all deps to module cache
go get -u ./...                   # Update all deps
govulncheck ./...                 # Check for vulnerabilities (separate tool)
# No "go audit fix" equivalent -- you update manually
go mod tidy                       # Remove unused deps, add missing ones
go mod verify                     # Verify checksums match go.sum
```

| Aspect | Go | npm |
|---|---|---|
| Dependency storage | Global module cache (`$GOPATH/pkg/mod/`) | Local `node_modules/` per project |
| Disk usage | Shared across projects | Duplicated per project |
| Install speed | Fast (cached globally) | Slow (copies per project) |
| Version strategy | MVS (minimum) | Maximal (latest compatible) |
| Lock file needed? | No (go.sum is for integrity) | Yes (package-lock.json) |
| Version ranges | Not supported | `^`, `~`, `>=`, `*` |
| Peer dependencies | Not a concept | Complex resolution rules |
| Dev dependencies | Not a concept | `devDependencies` |
| Vulnerability scanning | `govulncheck` (official tool) | `npm audit` (built-in) |

---

## 6. Internal Packages

### The internal/ Directory Convention

Go has a special rule: **if a package's import path contains a segment named `internal`, it can only be imported by code rooted at the parent of that `internal` directory.**

This is enforced by the Go compiler, not merely a convention.

```
myapp/
  go.mod
  main.go                          // can import myapp/internal/auth
  internal/
    auth/
      auth.go                      // package auth
    database/
      db.go                        // package database
  handlers/
    user.go                        // can import myapp/internal/auth
  pkg/
    utils/
      utils.go                     // can import myapp/internal/auth
```

```go
// myapp/main.go -- ALLOWED
import "myapp/internal/auth"

// myapp/handlers/user.go -- ALLOWED (within the same module)
import "myapp/internal/auth"
```

But if someone else tries to import your internal package:

```go
// In some other module: github.com/someone/otherapp/main.go
import "myapp/internal/auth"  // COMPILE ERROR: use of internal package not allowed
```

### WHY Internal Packages?

The problem `internal/` solves: Go only has two visibility levels (exported and unexported). Sometimes you want a third level: **"shared within my module, but not part of my public API."**

Without `internal/`:
- If you export a function (`Uppercase`), anyone can import and use it, and you are implicitly promising to maintain backward compatibility.
- If you unexport it (`lowercase`), other packages within your own module cannot use it either.

With `internal/`:
- You can export identifiers freely within internal packages, use them across your own packages, but external consumers cannot access them.
- You can refactor internal packages without worrying about breaking other people's code.

### Common Project Layouts Using internal/

```
myapp/
  cmd/
    myapp/
      main.go           // package main, entry point
    mytool/
      main.go           // another binary
  internal/
    auth/               // shared within module, not public API
      auth.go
      auth_test.go
    config/
      config.go
    middleware/
      logging.go
  pkg/                  // explicitly public API (convention, not enforced)
    client/
      client.go
  api/
    routes.go
  go.mod
  go.sum
```

The `pkg/` directory is a community convention (not compiler-enforced) meaning "this code is intended for external consumption." The `internal/` directory is compiler-enforced.

### No Direct JavaScript Equivalent

JavaScript has no built-in mechanism to prevent imports from within a package. Some approximations:

```javascript
// JavaScript workarounds:
// 1. package.json "exports" field (Node.js 12+) -- restricts entry points
{
  "exports": {
    ".": "./src/index.js",
    "./client": "./src/client/index.js"
    // ./internal/* is simply not listed, so it cannot be imported
  }
}

// 2. ESLint rules (e.g., no-restricted-imports)
// 3. TypeScript path mapping restrictions
// 4. Naming convention (prefix with _ or underscore)
```

None of these are as clean or reliable as Go's `internal/` enforcement.

---

## 7. Standard Library Tour

Go ships with a comprehensive standard library -- often described as "batteries included." You can build production-ready web servers, work with JSON, handle concurrency, manage files, and more without installing a single third-party package.

This stands in contrast to Node.js, where even basic tasks often require npm packages (Express for HTTP routing, lodash for utilities, moment/dayjs for dates, etc.).

### fmt -- Formatted I/O

```go
import "fmt"

// Printing
fmt.Println("Hello, World")                    // Print with newline
fmt.Printf("Name: %s, Age: %d\n", "Alice", 30) // Formatted print
fmt.Fprintf(os.Stderr, "error: %v\n", err)     // Print to specific writer

// Formatting to string (no printing)
s := fmt.Sprintf("Hello, %s!", name)            // Like JavaScript template literals

// Scanning (reading input)
var name string
fmt.Scan(&name)                                  // Read from stdin
fmt.Scanf("%s %d", &name, &age)                  // Formatted scan
```

Common format verbs:
```
%v    value in default format           fmt.Printf("%v", myStruct)
%+v   struct with field names           fmt.Printf("%+v", myStruct)
%#v   Go syntax representation          fmt.Printf("%#v", myStruct)
%T    type of the value                 fmt.Printf("%T", myVar)
%d    integer (decimal)                 fmt.Printf("%d", 42)
%s    string                            fmt.Printf("%s", "hello")
%f    float                             fmt.Printf("%.2f", 3.14159)
%t    boolean                           fmt.Printf("%t", true)
%p    pointer                           fmt.Printf("%p", &myVar)
%x    hexadecimal                       fmt.Printf("%x", 255) // "ff"
```

### strings -- String Manipulation

```go
import "strings"

strings.Contains("Hello, World", "World")    // true
strings.HasPrefix("Hello", "He")             // true
strings.HasSuffix("Hello", "lo")             // true
strings.ToUpper("hello")                     // "HELLO"
strings.ToLower("HELLO")                     // "hello"
strings.TrimSpace("  hello  ")               // "hello"
strings.Trim("**hello**", "*")               // "hello"
strings.Split("a,b,c", ",")                  // ["a", "b", "c"]
strings.Join([]string{"a", "b", "c"}, ", ")  // "a, b, c"
strings.Replace("aaa", "a", "b", 2)          // "bba" (replace first 2)
strings.ReplaceAll("aaa", "a", "b")          // "bbb"
strings.Count("hello", "l")                  // 2
strings.Index("hello", "ll")                 // 2
strings.Repeat("ha", 3)                      // "hahaha"

// strings.Builder for efficient concatenation (like StringBuilder in Java)
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("hello ")
}
result := b.String()
```

### strconv -- String Conversions

```go
import "strconv"

// String to number
i, err := strconv.Atoi("42")           // string to int
f, err := strconv.ParseFloat("3.14", 64) // string to float64
b, err := strconv.ParseBool("true")    // string to bool

// Number to string
s := strconv.Itoa(42)                   // int to string: "42"
s = strconv.FormatFloat(3.14, 'f', 2, 64) // float to string: "3.14"
s = strconv.FormatBool(true)             // bool to string: "true"

// Quote and unquote
s = strconv.Quote(`Hello "World"`)       // `"Hello \"World\""`
s, err = strconv.Unquote(`"Hello \"World\""`)
```

### os -- Operating System Interface

```go
import "os"

// Environment variables
home := os.Getenv("HOME")              // Get env var (empty string if not set)
os.Setenv("MY_VAR", "value")           // Set env var
pairs := os.Environ()                   // All env vars as []string

// Command-line arguments
args := os.Args                          // []string, os.Args[0] is the program name

// File operations
data, err := os.ReadFile("config.json")  // Read entire file
err = os.WriteFile("out.txt", data, 0644) // Write entire file
err = os.Mkdir("newdir", 0755)           // Create directory
err = os.MkdirAll("path/to/dir", 0755)  // Create nested directories
err = os.Remove("file.txt")             // Delete file
err = os.RemoveAll("dir/")              // Delete directory recursively
err = os.Rename("old.txt", "new.txt")   // Rename/move

// File info
info, err := os.Stat("file.txt")
if os.IsNotExist(err) {
    fmt.Println("File does not exist")
}
fmt.Println(info.Size(), info.ModTime(), info.IsDir())

// Working directory
dir, err := os.Getwd()                  // Current working directory

// Exit
os.Exit(1)                               // Exit with status code (defers do NOT run)
```

### io -- I/O Primitives

The `io` package defines the most fundamental interfaces in Go:

```go
import "io"

// The two most important interfaces in all of Go:
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Everything in Go works with these interfaces: files, network connections, HTTP bodies, compression streams, encryption, buffers. This is Go's answer to Node.js streams.

```go
// Copy from reader to writer
io.Copy(dst, src)                         // Like piping in Node.js: src.pipe(dst)

// Read all bytes from a reader
data, err := io.ReadAll(resp.Body)        // Like collecting a stream into a buffer

// Multi-writer (write to multiple destinations)
w := io.MultiWriter(file, os.Stdout)      // Write to file AND stdout
fmt.Fprintln(w, "logged to both")

// Limit reader
limited := io.LimitReader(r, 1024*1024)   // Read at most 1MB

// NopCloser wraps a Reader as a ReadCloser with a no-op Close
rc := io.NopCloser(strings.NewReader("hello"))
```

### net/http -- HTTP Client and Server

Go's HTTP package is production-ready out of the box. No Express needed.

```go
import "net/http"

// === HTTP Server ===

// Simple handler function
func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Query().Get("name"))
}

// Register handler and start server
http.HandleFunc("/hello", helloHandler)
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"status":"ok"}`))
})
log.Fatal(http.ListenAndServe(":8080", nil))

// Using a custom ServeMux (router)
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUserHandler)   // Go 1.22+ path patterns
mux.HandleFunc("POST /users", createUserHandler)
server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
}
log.Fatal(server.ListenAndServe())

// === HTTP Client ===

// Simple GET
resp, err := http.Get("https://api.example.com/data")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()
body, err := io.ReadAll(resp.Body)

// Custom request with headers
req, err := http.NewRequest("POST", "https://api.example.com/data",
    strings.NewReader(`{"name":"Alice"}`))
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Authorization", "Bearer token123")

client := &http.Client{Timeout: 10 * time.Second}
resp, err = client.Do(req)
```

**Comparison with Node.js/Express:**

```javascript
// Node.js: Requires installing Express (or similar)
const express = require('express');
const app = express();

app.get('/hello', (req, res) => {
    res.send(`Hello, ${req.query.name}!`);
});

app.listen(8080);
```

```go
// Go: Zero dependencies, standard library only
http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Query().Get("name"))
})
http.ListenAndServe(":8080", nil)
```

### encoding/json -- JSON Encoding/Decoding

```go
import "encoding/json"

type User struct {
    Name  string   `json:"name"`
    Email string   `json:"email"`
    Age   int      `json:"age,omitempty"`       // Omit if zero value
    Admin bool     `json:"is_admin"`
    Notes string   `json:"-"`                   // Never include in JSON
}

// Struct to JSON (Marshal)
user := User{Name: "Alice", Email: "alice@example.com", Age: 30, Admin: true}
data, err := json.Marshal(user)
// {"name":"Alice","email":"alice@example.com","age":30,"is_admin":true}

// Pretty print
data, err = json.MarshalIndent(user, "", "  ")

// JSON to struct (Unmarshal)
var u User
err = json.Unmarshal([]byte(`{"name":"Bob","email":"bob@example.com"}`), &u)

// Streaming: encode to writer / decode from reader
// Useful for HTTP handlers
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    user := User{Name: "Alice", Email: "alice@example.com"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)   // Stream directly to response
}

func createUserHandler(w http.ResponseWriter, r *http.Request) {
    var user User
    err := json.NewDecoder(r.Body).Decode(&user)  // Stream from request body
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    // ... use user
}

// Working with unknown JSON structure
var result map[string]interface{}
json.Unmarshal(data, &result)

// Or use json.RawMessage to delay parsing
type Envelope struct {
    Type    string          `json:"type"`
    Payload json.RawMessage `json:"payload"`  // Parse later based on Type
}
```

### time -- Time and Duration

```go
import "time"

// Current time
now := time.Now()
fmt.Println(now)                           // 2024-01-15 10:30:00.123456 -0700 MST

// Creating specific times
t := time.Date(2024, time.January, 15, 10, 30, 0, 0, time.UTC)

// Duration
d := 5 * time.Second
d = 100 * time.Millisecond
d = 2*time.Hour + 30*time.Minute

// Parsing and formatting
// Go uses a REFERENCE TIME: Mon Jan 2 15:04:05 MST 2006 (1/2 3:4:5 6 7)
t, err := time.Parse("2006-01-02", "2024-01-15")
t, err = time.Parse(time.RFC3339, "2024-01-15T10:30:00Z")
s := now.Format("2006-01-02 15:04:05")    // "2024-01-15 10:30:00"
s = now.Format(time.RFC3339)               // "2024-01-15T10:30:00-07:00"

// Arithmetic
future := now.Add(24 * time.Hour)          // Add duration
elapsed := time.Since(start)               // Duration since a time
remaining := time.Until(deadline)          // Duration until a time
diff := t2.Sub(t1)                         // Duration between two times

// Comparison
t1.Before(t2)
t1.After(t2)
t1.Equal(t2)

// Timers and tickers
timer := time.NewTimer(5 * time.Second)    // Fire once after duration
<-timer.C                                   // Block until timer fires

ticker := time.NewTicker(1 * time.Second)  // Fire repeatedly
for tick := range ticker.C {
    fmt.Println("Tick:", tick)
}

// Sleep
time.Sleep(2 * time.Second)                // Block for duration
```

**Go's reference time** is one of its quirkiest features. Instead of `YYYY-MM-DD`, Go uses the literal values `2006-01-02` because the reference date is `01/02 03:04:05PM '06 -0700` -- each component is a different sequential digit.

### sync -- Synchronization Primitives

```go
import "sync"

// Mutex: mutual exclusion lock
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}

// RWMutex: multiple readers OR one writer
var rwmu sync.RWMutex
var cache map[string]string

func read(key string) string {
    rwmu.RLock()              // Multiple goroutines can read simultaneously
    defer rwmu.RUnlock()
    return cache[key]
}

func write(key, value string) {
    rwmu.Lock()               // Exclusive access for writing
    defer rwmu.Unlock()
    cache[key] = value
}

// WaitGroup: wait for a collection of goroutines to finish
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Printf("Worker %d done\n", id)
    }(i)
}
wg.Wait() // Block until all 10 goroutines call Done()

// Once: execute a function exactly once (great for initialization)
var once sync.Once
var db *Database

func getDB() *Database {
    once.Do(func() {
        db = connectToDatabase() // This runs only once, even if called from many goroutines
    })
    return db
}

// sync.Map: concurrent-safe map (use sparingly; regular map + mutex is usually better)
var m sync.Map
m.Store("key", "value")
val, ok := m.Load("key")
m.Delete("key")
m.Range(func(key, value interface{}) bool {
    fmt.Println(key, value)
    return true // return false to stop iteration
})
```

### context -- Cancellation and Deadlines

```go
import "context"

// context.Background(): the root context, never canceled
ctx := context.Background()

// context.WithCancel: manual cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // Always defer cancel to avoid context leak

go func() {
    // Do work...
    select {
    case <-ctx.Done():
        fmt.Println("Canceled:", ctx.Err())
        return
    case result := <-workCh:
        fmt.Println("Result:", result)
    }
}()

cancel() // Signal all goroutines listening on this context to stop

// context.WithTimeout: auto-cancel after duration
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// context.WithDeadline: auto-cancel at a specific time
deadline := time.Now().Add(30 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// context.WithValue: attach request-scoped data (use sparingly)
type contextKey string
const userKey contextKey = "user"

ctx = context.WithValue(ctx, userKey, "alice")
user := ctx.Value(userKey).(string)

// Real-world: HTTP handler with context
func apiHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // Every HTTP request already has a context

    result, err := fetchFromDB(ctx) // Pass context to downstream calls
    if err != nil {
        if ctx.Err() == context.Canceled {
            return // Client disconnected, no point responding
        }
        http.Error(w, err.Error(), 500)
        return
    }
    json.NewEncoder(w).Encode(result)
}
```

### Standard Library vs Node.js Core + npm

| Task | Go (standard library) | Node.js |
|---|---|---|
| HTTP server | `net/http` | `http` (bare) or Express/Fastify (npm) |
| JSON handling | `encoding/json` | `JSON.parse/stringify` (built-in) |
| File I/O | `os`, `io` | `fs`, `fs/promises` |
| Path manipulation | `path/filepath` | `path` |
| URL parsing | `net/url` | `url`, `URL` (built-in) |
| String utilities | `strings` | No equivalent (lodash on npm) |
| Regex | `regexp` | `RegExp` (built-in) |
| Crypto/hashing | `crypto/*` | `crypto` |
| Testing | `testing` (built-in) | Jest/Mocha/Vitest (npm) |
| Benchmarking | `testing` (built-in) | benchmark.js (npm) |
| Templating | `html/template`, `text/template` | EJS/Handlebars (npm) |
| Compression | `compress/gzip`, `compress/zlib` | `zlib` |
| SQL databases | `database/sql` | pg/mysql2/better-sqlite3 (npm) |
| TLS/HTTPS | `crypto/tls` | `tls` |
| WebSocket | Not in stdlib (use gorilla/websocket) | ws (npm) |

**Key insight:** Go's standard library covers about 80% of what you need for typical backend development. In Node.js, the ratio is closer to 30-40%, with npm packages filling the gap. This means Go projects tend to have far fewer dependencies, which reduces supply chain risk.

---

## 8. Creating and Publishing Modules

### Creating a Module from Scratch

Let us walk through creating a reusable Go module step by step.

**Step 1: Create the project structure**

```bash
mkdir -p ~/projects/mathkit
cd ~/projects/mathkit
go mod init github.com/yourusername/mathkit
```

**Step 2: Write the package code**

```
mathkit/
  go.mod
  mathkit.go       // package mathkit (root package)
  mathkit_test.go
  stats/
    stats.go       // package stats (sub-package)
    stats_test.go
  matrix/
    matrix.go      // package matrix (sub-package)
```

```go
// mathkit.go
package mathkit

import "math"

// Abs returns the absolute value of x.
func Abs(x float64) float64 {
    return math.Abs(x)
}

// Clamp restricts x to the range [min, max].
func Clamp(x, min, max float64) float64 {
    if x < min {
        return min
    }
    if x > max {
        return max
    }
    return x
}

// Lerp performs linear interpolation between a and b by t.
// t=0 returns a, t=1 returns b.
func Lerp(a, b, t float64) float64 {
    return a + (b-a)*t
}
```

```go
// stats/stats.go
package stats

import "math"

// Mean calculates the arithmetic mean of a slice of float64 values.
// It returns 0 if the slice is empty.
func Mean(data []float64) float64 {
    if len(data) == 0 {
        return 0
    }
    sum := 0.0
    for _, v := range data {
        sum += v
    }
    return sum / float64(len(data))
}

// StdDev calculates the population standard deviation.
func StdDev(data []float64) float64 {
    if len(data) == 0 {
        return 0
    }
    m := Mean(data)
    sum := 0.0
    for _, v := range data {
        d := v - m
        sum += d * d
    }
    return math.Sqrt(sum / float64(len(data)))
}
```

```go
// mathkit_test.go
package mathkit

import "testing"

func TestClamp(t *testing.T) {
    tests := []struct {
        x, min, max, want float64
    }{
        {5, 0, 10, 5},     // within range
        {-5, 0, 10, 0},    // below range
        {15, 0, 10, 10},   // above range
    }
    for _, tt := range tests {
        got := Clamp(tt.x, tt.min, tt.max)
        if got != tt.want {
            t.Errorf("Clamp(%v, %v, %v) = %v, want %v",
                tt.x, tt.min, tt.max, got, tt.want)
        }
    }
}

func TestLerp(t *testing.T) {
    if got := Lerp(0, 10, 0.5); got != 5 {
        t.Errorf("Lerp(0, 10, 0.5) = %v, want 5", got)
    }
}
```

**Step 3: Write documentation**

Go documentation is written as comments directly above the declarations:

```go
// Package mathkit provides mathematical utility functions
// including interpolation, clamping, and basic operations.
//
// Example usage:
//
//     result := mathkit.Lerp(0, 100, 0.5) // 50.0
//     clamped := mathkit.Clamp(150, 0, 100) // 100.0
package mathkit
```

View documentation locally:

```bash
# Install the doc server
go install golang.org/x/pkgsite/cmd/pkgsite@latest

# Run it
pkgsite
# Open http://localhost:8080/github.com/yourusername/mathkit
```

Or the simpler built-in tool:

```bash
go doc github.com/yourusername/mathkit         # Package docs
go doc github.com/yourusername/mathkit.Clamp   # Specific function
go doc github.com/yourusername/mathkit/stats   # Sub-package
```

**Step 4: Tag a version and publish**

```bash
# Commit your code
git init
git add .
git commit -m "Initial implementation of mathkit"

# Push to GitHub
git remote add origin https://github.com/yourusername/mathkit.git
git push -u origin main

# Tag a version
git tag v1.0.0
git push origin v1.0.0
```

That is it. There is no `go publish` command. Once the code is on a publicly accessible Git repository with a version tag, anyone can use it:

```bash
go get github.com/yourusername/mathkit@v1.0.0
```

Go uses a **module proxy** (`proxy.golang.org`) that automatically fetches and caches modules from Git repositories. The first time anyone requests your module, the proxy fetches it from your Git host, caches it, and serves it from then on.

### Publishing a v2 (Breaking Change)

If you need to make breaking changes:

```bash
# Option 1: Major branch (simpler)
mkdir v2/
# Copy code to v2/, update go.mod:
# module github.com/yourusername/mathkit/v2

# Option 2: Major subdirectory
# Update go.mod in root:
# module github.com/yourusername/mathkit/v2

git tag v2.0.0
git push origin v2.0.0
```

Users import the new version explicitly:

```go
import "github.com/yourusername/mathkit/v2"
```

### Comparison with npm publish

```bash
# npm workflow
npm init                    # Create package.json
npm login                   # Authenticate with npm registry
npm publish                 # Publish to npm registry
npm version patch           # Bump version in package.json
npm publish                 # Publish new version
npm deprecate pkg@"<1.0.0" "Use >= 1.0.0"

# Go workflow
go mod init github.com/user/pkg   # Create go.mod
git tag v1.0.0                      # Tag the version
git push origin v1.0.0              # Push tag (that is the publish step)
# Module proxy picks it up automatically
```

| Aspect | Go | npm |
|---|---|---|
| Registry | No registry; uses Git hosts + module proxy | npm registry (npmjs.com) |
| Authentication | None needed (public Git repos) | `npm login` |
| Publish command | `git tag` + `git push` | `npm publish` |
| Private modules | Private Git repos + `GOPRIVATE` env var | npm private packages (paid) |
| Namespace | Git URL path (natural, unique) | Scoped packages (`@scope/pkg`) |
| Name squatting | Not possible (tied to your Git URL) | Common problem |

---

## 9. Workspace Mode

### What Is Workspace Mode?

Go 1.18 introduced **workspaces** (`go.work`) for developing multiple modules simultaneously. This is essential when you are working on a library and an application that uses it at the same time.

### The Problem Workspaces Solve

Imagine you are developing two modules:

```
~/projects/
  mylib/        # A library (github.com/you/mylib)
    go.mod
    lib.go
  myapp/        # An app that depends on mylib (github.com/you/myapp)
    go.mod
    main.go     # imports "github.com/you/mylib"
```

Without workspaces, to test local changes to `mylib` in `myapp`, you would need to:

```
// myapp/go.mod -- hacky replace directive
replace github.com/you/mylib => ../mylib
```

This works but is error-prone. You might accidentally commit the `replace` directive, breaking your CI/CD. With workspaces, there is a cleaner solution.

### Creating a Workspace

```bash
# From the parent directory
cd ~/projects

# Initialize workspace
go work init ./mylib ./myapp
```

This creates a `go.work` file:

```
go 1.22

use (
    ./mylib
    ./myapp
)
```

Now, when you build `myapp`, Go automatically uses your local copy of `mylib` instead of the published version. No `replace` directives needed in `go.mod`.

### Workspace Commands

```bash
go work init           # Create go.work in current directory
go work use ./module   # Add a module to the workspace
go work edit           # Edit go.work programmatically
go work sync           # Sync workspace dependencies to module go.mod files
```

### Important Rules

- **Never commit `go.work` to version control** (add it to `.gitignore`). It is a local development tool, not a project configuration.
- The workspace only affects local development. Published modules still use their `go.mod` files.
- `go.work` can include a `replace` directive that applies to all modules in the workspace.

### Comparison with npm Workspaces / Monorepos

```json
// package.json (npm workspaces)
{
  "name": "my-monorepo",
  "workspaces": [
    "packages/mylib",
    "packages/myapp"
  ]
}
```

```
// go.work (Go workspace)
go 1.22

use (
    ./packages/mylib
    ./packages/myapp
)
```

| Aspect | Go Workspaces | npm Workspaces |
|---|---|---|
| Config file | `go.work` (not committed) | `package.json` (committed) |
| Purpose | Local multi-module development | Monorepo management |
| Hoisting | N/A (no node_modules) | Hoists shared dependencies |
| Symlinks | N/A | Creates symlinks in node_modules |
| Committed to VCS? | No (`.gitignore` it) | Yes |
| Tools | Built into `go` tool | npm, yarn, pnpm, lerna, nx, turborepo |
| Scope | Local development only | Full monorepo workflow |

Go workspaces are intentionally simpler in scope. They solve exactly one problem: "use local versions of modules during development." The broader monorepo problem (shared scripts, coordinated publishing, dependency graph analysis) is handled by external tools in Go or is simply not needed due to Go's simpler dependency model.

---

## 10. Build Tags and Conditional Compilation

### What Are Build Tags?

Build tags (also called build constraints) let you include or exclude files from compilation based on conditions like the target operating system, architecture, or custom tags.

### Syntax (Go 1.17+)

```go
//go:build linux

package mypackage

// This entire file is only compiled on Linux
```

```go
//go:build windows

package mypackage

// This file is only compiled on Windows
```

```go
//go:build !windows

package mypackage

// This file is compiled on everything EXCEPT Windows
```

### Boolean Expressions

```go
//go:build linux && amd64
// Linux AND 64-bit AMD/Intel

//go:build linux || darwin
// Linux OR macOS

//go:build (linux || darwin) && !arm64
// (Linux OR macOS) AND NOT ARM64

//go:build ignore
// Never compiled (useful for example files or scratch code)
```

### Practical Example: Platform-Specific Code

```
myapp/
  terminal.go           // Shared code
  terminal_unix.go      // Unix-specific implementation
  terminal_windows.go   // Windows-specific implementation
```

```go
// terminal.go
package myapp

// ClearScreen clears the terminal screen.
// Implementation is platform-specific.
func ClearScreen() {
    clearScreen() // Calls the platform-specific version
}
```

```go
//go:build !windows

// terminal_unix.go
package myapp

import (
    "fmt"
    "os"
    "os/exec"
)

func clearScreen() {
    cmd := exec.Command("clear")
    cmd.Stdout = os.Stdout
    cmd.Run()
}
```

```go
//go:build windows

// terminal_windows.go
package myapp

import (
    "os"
    "os/exec"
)

func clearScreen() {
    cmd := exec.Command("cmd", "/c", "cls")
    cmd.Stdout = os.Stdout
    cmd.Run()
}
```

### Custom Build Tags

You can define your own tags:

```go
//go:build debug

package myapp

import "fmt"

func DebugLog(msg string) {
    fmt.Println("[DEBUG]", msg)
}
```

```go
//go:build !debug

package myapp

func DebugLog(msg string) {
    // No-op in non-debug builds
}
```

Build with the tag:

```bash
go build -tags debug ./...              # Include files with //go:build debug
go build ./...                          # Default: uses the !debug version
go test -tags "debug integration" ./... # Multiple tags
```

### File Name Conventions

Go also supports implicit build constraints based on file names:

```
file_linux.go          // Only compiled on Linux
file_windows.go        // Only compiled on Windows
file_darwin.go         // Only compiled on macOS
file_amd64.go          // Only compiled on amd64 architecture
file_linux_amd64.go    // Only compiled on Linux amd64
file_test.go           // Only compiled during testing
```

These are equivalent to having the corresponding `//go:build` tag at the top, but the tag syntax is preferred for clarity since it is more explicit.

### The Old Syntax (Pre-Go 1.17)

You may encounter the old syntax in older codebases:

```go
// +build linux,amd64       // OLD syntax (still works but deprecated)

//go:build linux && amd64   // NEW syntax (preferred)
```

The old syntax uses commas for AND and spaces for OR -- confusing. The new syntax uses proper boolean operators. Use the new syntax for all new code.

### No Direct JavaScript Equivalent

JavaScript does not have built-in conditional compilation. Approximations include:

```javascript
// JavaScript: Runtime checks (not compile-time)
if (process.platform === 'win32') {
    // Windows-specific code -- still shipped to all platforms
}

// Webpack DefinePlugin (compile-time, sort of)
if (process.env.NODE_ENV === 'production') {
    // Dead-code elimination removes this in development builds
}

// Conditional require (CommonJS)
const impl = process.platform === 'win32'
    ? require('./impl-windows')
    : require('./impl-unix');
```

The difference: Go's build tags operate at **compile time**. The excluded code is never compiled into the binary, reducing binary size and eliminating unreachable code paths. JavaScript's approach is mostly runtime (with bundler optimizations as an exception).

---

## 11. Go vs Node.js: Full Comparison Table

| Concept | Go | Node.js |
|---|---|---|
| **Code organization** | Package (directory) | Module (file) |
| **Dependency file** | `go.mod` | `package.json` |
| **Lock/checksum file** | `go.sum` | `package-lock.json` |
| **Dependency storage** | Global module cache | Local `node_modules/` |
| **Export mechanism** | Capitalization | `export` keyword |
| **Import syntax** | `import "path"` | `import ... from "path"` |
| **Add dependency** | `go get pkg@version` | `npm install pkg@version` |
| **Update dependencies** | `go get -u ./...` | `npm update` |
| **Remove unused** | `go mod tidy` | `npm prune` |
| **Vendor/offline** | `go mod vendor` | `npm ci` with cache |
| **Version strategy** | MVS (minimum version) | Maximal version |
| **Version ranges** | Not supported | `^`, `~`, `>=`, `*` |
| **Registry** | Module proxy + Git | npmjs.com |
| **Publish** | `git tag` + `git push` | `npm publish` |
| **Private packages** | `GOPRIVATE` env var | npm private packages |
| **Workspaces** | `go.work` (not committed) | `package.json` workspaces |
| **Build tags** | `//go:build` (compile-time) | Bundler flags (build-time) |
| **Testing** | `go test` (built-in) | Jest/Mocha/Vitest (npm) |
| **Formatting** | `gofmt` (built-in, canonical) | Prettier (npm, configurable) |
| **Linting** | `go vet` + staticcheck | ESLint (npm) |
| **Circular deps** | Compile error | Allowed (can cause bugs) |
| **Dev dependencies** | No concept | `devDependencies` |
| **Tree shaking** | Compiler does it automatically | Bundler does it (webpack, etc.) |

---

## 12. Key Takeaways

1. **A package is a directory.** All `.go` files in a directory share the same package name. This is simpler and more predictable than JavaScript's one-file-one-module approach.

2. **Capitalization IS your API.** Uppercase = exported, lowercase = unexported. No keywords needed. This is visible at a glance and enforced by the compiler.

3. **Modules solved GOPATH's problems.** `go.mod` provides versioning, isolation, and reproducibility. Think of it as a stricter, simpler `package.json`.

4. **MVS gives you reproducibility for free.** By selecting minimum versions, Go eliminates the need for a lock file to achieve reproducible builds. You trade "always latest" for "always predictable."

5. **The standard library is your first dependency.** Before reaching for a third-party package, check the standard library. `net/http`, `encoding/json`, `testing`, `context`, and `sync` cover an enormous amount of backend development.

6. **`internal/` is your friend.** Use it to share code within your module without committing to a public API. This is a compiler-enforced boundary with no JavaScript equivalent.

7. **Build tags enable true conditional compilation.** Platform-specific code is handled cleanly at compile time, not through runtime checks.

8. **Publishing is just Git.** Tag a version, push it, and the module proxy handles distribution. No registry account, no publish command, no name-squatting risk.

9. **Always run `go mod tidy`.** It is the single most important module maintenance command. Run it before every commit.

10. **Fewer dependencies is a feature.** Go's rich standard library and culture of minimal dependencies means smaller dependency trees, faster builds, and reduced supply chain risk.

---

## 13. Practice Exercises

### Exercise 1: Create a Multi-Package Module
Create a module called `textkit` with three packages:
- Root package (`textkit`): `Reverse(s string) string` and `IsPalindrome(s string) bool`
- Sub-package `transform`: `CamelCase(s string) string`, `SnakeCase(s string) string`, `KebabCase(s string) string`
- Sub-package `validate`: `IsEmail(s string) bool`, `IsURL(s string) bool`

Make sure to:
- Use proper exported/unexported naming
- Write doc comments for every exported function
- Write tests for each package
- Use `internal/` for any helper functions shared between sub-packages

### Exercise 2: Explore the Standard Library
Write a program using ONLY the standard library (zero third-party dependencies) that:
1. Starts an HTTP server on port 8080
2. Has a `GET /time` endpoint that returns the current time as JSON
3. Has a `POST /echo` endpoint that returns the request body back as JSON
4. Logs every request with the method, path, and duration
5. Gracefully shuts down on SIGINT/SIGTERM using `context`

### Exercise 3: Dependency Management Practice
1. Create a new module
2. Add `github.com/fatih/color` as a dependency using `go get`
3. Write code that uses it to print colored output
4. Run `go mod tidy` and examine `go.mod` and `go.sum`
5. Run `go mod vendor` and explore the `vendor/` directory
6. Run `go mod graph` to see the dependency tree
7. Run `go mod why github.com/fatih/color` to see why it is needed

### Exercise 4: Build Tags
Create a module with a `Logger` that behaves differently based on build tags:
- Default build: logs to `os.Stdout` with timestamps
- `//go:build debug`: logs to `os.Stdout` with timestamps, file names, and line numbers
- `//go:build silent`: all log functions are no-ops

Test with: `go run .`, `go run -tags debug .`, `go run -tags silent .`

### Exercise 5: Workspace Mode
1. Create two modules in separate directories: `mathlib` and `calculator`
2. `calculator` depends on `mathlib`
3. Set up a `go.work` file so you can develop both simultaneously
4. Make a change to `mathlib` and verify `calculator` picks it up immediately without publishing
5. Verify that both modules build independently without `go.work` (using their published versions or replace directives)

### Exercise 6: Internal Packages
Refactor one of your previous exercises to use the `internal/` pattern:
1. Move shared helper functions to `internal/helpers/`
2. Verify that packages within your module can import from `internal/`
3. Create a separate module and verify that it CANNOT import from your `internal/` packages (you should get a compile error)

---

> **Next Chapter:** Chapter 10 will cover **Concurrency: Goroutines and Channels** -- one of Go's most powerful and distinctive features, and a paradigm shift from Node.js's single-threaded event loop.
