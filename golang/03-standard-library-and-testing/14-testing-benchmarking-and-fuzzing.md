# Chapter 14: Testing, Benchmarking & Fuzzing

## Table of Contents

1. [Testing Philosophy in Go](#1-testing-philosophy-in-go)
2. [Basic Tests](#2-basic-tests)
3. [Table-Driven Tests](#3-table-driven-tests)
4. [Test Helpers](#4-test-helpers)
5. [Subtests](#5-subtests)
6. [Test Coverage](#6-test-coverage)
7. [Benchmarks](#7-benchmarks)
8. [Fuzzing](#8-fuzzing)
9. [Test Doubles](#9-test-doubles---interfaces-for-mocking)
10. [Integration Tests](#10-integration-tests)
11. [Testify Package](#11-testify-package)
12. [go test Flags](#12-go-test-flags)
13. [Key Takeaways](#13-key-takeaways)
14. [Practice Exercises](#14-practice-exercises)

---

## 1. Testing Philosophy in Go

### Testing Is a First-Class Citizen

Go made a deliberate design decision: **testing is built into the language toolchain, not bolted on afterward**. There is no need to install a test runner, assertion library, coverage tool, or benchmarking framework. Everything ships with `go test`.

This was an intentional philosophy from Go's creators at Google, where large-scale software demands predictable, consistent testing across thousands of engineers. The reasoning:

- **Zero friction** -- If testing requires zero setup, developers write more tests.
- **Consistency** -- Every Go project has the same test structure, the same commands, the same conventions. A new contributor can run tests on any project immediately.
- **No dependency rot** -- External test frameworks evolve, break, and create version conflicts. Go sidesteps this entirely.
- **Toolchain integration** -- Because testing is part of `go`, it integrates seamlessly with `go vet`, `go build`, the race detector, profiling, and coverage analysis.

### Go vs. Node.js Testing Ecosystem

| Aspect | Go | Node.js |
|--------|-----|---------|
| Test runner | `go test` (built-in) | Jest, Mocha, Vitest, Ava, Tap, node:test... |
| Assertions | `if` + `t.Error`/`t.Fatal` | `expect()`, `assert()`, Chai, Should.js... |
| Coverage | `go test -cover` (built-in) | Istanbul, c8, nyc (separate install) |
| Benchmarking | `testing.B` (built-in) | benchmark.js, Tinybench (separate install) |
| Fuzz testing | `testing.F` (built-in, Go 1.18+) | Rare; no standard tool |
| Mocking | Interfaces (language feature) | `jest.mock()`, `jest.spyOn()`, Sinon |
| Config needed | None (zero-config) | `jest.config.js`, `.mocharc.yml`, etc. |
| Dependencies for testing | 0 | 3-10+ packages typically |

**The Node.js pain:** In a typical Node.js project, your `devDependencies` might include `jest`, `@types/jest`, `ts-jest`, `babel-jest`, `jest-environment-jsdom`, and several transform plugins -- just to run tests. Configuration files grow. Upgrades break. Teams argue over which framework to use.

**The Go answer:** `go test ./...` -- done. Every Go developer on Earth uses the same command.

---

## 2. Basic Tests

### File Naming Convention

Go test files must end with `_test.go`. The Go toolchain automatically excludes these from production builds -- they only compile when you run `go test`.

```
myproject/
  math.go           <- production code
  math_test.go      <- test code (same package)
  stringutil.go
  stringutil_test.go
```

### Your First Test

**Production code (`math.go`):**

```go
package mathutil

// Add returns the sum of two integers.
func Add(a, b int) int {
    return a + b
}

// Divide returns a/b. Returns an error if b is zero.
func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}
```

**Test code (`math_test.go`):**

```go
package mathutil

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

func TestAddNegative(t *testing.T) {
    result := Add(-1, -1)
    if result != -2 {
        t.Errorf("Add(-1, -1) = %d; want -2", result)
    }
}
```

Run it:

```bash
go test
# Output:
# PASS
# ok      mathutil    0.002s

go test -v
# Output:
# === RUN   TestAdd
# --- PASS: TestAdd (0.00s)
# === RUN   TestAddNegative
# --- PASS: TestAddNegative (0.00s)
# PASS
# ok      mathutil    0.002s
```

### The Rules

1. **File must end with `_test.go`.**
2. **Function must start with `Test` followed by an uppercase letter** (e.g., `TestAdd`, not `Testadd`).
3. **Function takes exactly one parameter: `*testing.T`.**
4. **No return value.** Signal failure through `t.Error`, `t.Fatal`, etc.

### The `testing.T` Methods

| Method | Behavior |
|--------|----------|
| `t.Error(args...)` | Log the error and **continue** running the test |
| `t.Errorf(format, args...)` | Formatted error, **continue** running |
| `t.Fatal(args...)` | Log the error and **stop** this test immediately |
| `t.Fatalf(format, args...)` | Formatted fatal, **stop** immediately |
| `t.Log(args...)` | Log a message (only visible with `-v`) |
| `t.Logf(format, args...)` | Formatted log |
| `t.Skip(args...)` | Skip this test (e.g., OS-specific tests) |
| `t.Skipf(format, args...)` | Formatted skip |
| `t.Fail()` | Mark test as failed, **continue** |
| `t.FailNow()` | Mark test as failed, **stop** immediately |
| `t.Cleanup(func())` | Register a cleanup function (runs after test completes) |
| `t.Parallel()` | Mark this test to run in parallel with other parallel tests |

**`Error` vs `Fatal` -- when to use which:**

- Use `t.Error`/`t.Errorf` when you want to report a failure but keep checking other things in the same test. This gives you more diagnostic information when something goes wrong.
- Use `t.Fatal`/`t.Fatalf` when continuing the test makes no sense -- for example, if setup fails and every subsequent check would panic.

```go
func TestDivide(t *testing.T) {
    // If this fails, the rest of the test is meaningless
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatalf("Divide(10, 2) returned unexpected error: %v", err)
    }

    // This is a secondary check -- report but don't stop
    if result != 5.0 {
        t.Errorf("Divide(10, 2) = %f; want 5.0", result)
    }
}
```

### Same-Package vs External Tests

**Same-package test (white-box):**

```go
// In math_test.go
package mathutil  // same package -- can access unexported identifiers

func TestInternalHelper(t *testing.T) {
    // Can test unexported function
    result := internalHelper(42)
    if result != 84 {
        t.Errorf("got %d; want 84", result)
    }
}
```

**External test (black-box):**

```go
// In math_test.go
package mathutil_test  // external package -- only exported API visible

import "myproject/mathutil"

func TestAdd(t *testing.T) {
    result := mathutil.Add(2, 3)
    if result != 5 {
        t.Errorf("got %d; want 5", result)
    }
}
```

Use the external test package (`_test` suffix) when you want to test only the public API, simulating what a real consumer of your package experiences. Use the same package when you need to test internal logic.

### Comparison: Go vs Jest

**Go:**

```go
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}
```

**Jest (JavaScript):**

```javascript
test('adds 2 + 3 to equal 5', () => {
  expect(add(2, 3)).toBe(5);
});
```

Key differences:
- Go uses plain `if` statements for assertions -- no special `expect` API to learn.
- Go test names are function names (must be valid identifiers).
- Jest uses string descriptions (more flexible but less greppable).
- Go reports **what happened vs what was expected** manually -- you control the error message.

---

## 3. Table-Driven Tests

### The Idiomatic Go Pattern

Table-driven tests are the **single most important Go testing pattern**. You will see them in the Go standard library, in every serious Go project, and in every Go job interview.

The idea: define a slice of test cases (the "table"), then loop over them.

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"zeros", 0, 0, 0},
        {"negative numbers", -1, -2, -3},
        {"mixed signs", -1, 5, 4},
        {"large numbers", 1000000, 2000000, 3000000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d", tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

Output with `go test -v`:

```
=== RUN   TestAdd
=== RUN   TestAdd/positive_numbers
=== RUN   TestAdd/zeros
=== RUN   TestAdd/negative_numbers
=== RUN   TestAdd/mixed_signs
=== RUN   TestAdd/large_numbers
--- PASS: TestAdd (0.00s)
    --- PASS: TestAdd/positive_numbers (0.00s)
    --- PASS: TestAdd/zeros (0.00s)
    --- PASS: TestAdd/negative_numbers (0.00s)
    --- PASS: TestAdd/mixed_signs (0.00s)
    --- PASS: TestAdd/large_numbers (0.00s)
```

### WHY Table-Driven Tests Are the Standard

1. **Adding a new test case is one line.** No new function, no boilerplate. Just append to the slice.
2. **All cases are visible at a glance.** You can scan the table and immediately see what is covered and what is missing.
3. **Consistent error messages.** Every case uses the same assertion logic with the same formatting.
4. **Each case gets a name.** With `t.Run`, you can run individual cases: `go test -run TestAdd/zeros`.
5. **Easy to refactor.** If the function signature changes, you update the struct and the loop -- not 20 separate test functions.

### A More Realistic Example

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        want      float64
        wantErr   bool
        errSubstr string // substring we expect in the error message
    }{
        {
            name: "simple division",
            a:    10, b: 2,
            want: 5.0,
        },
        {
            name: "division resulting in decimal",
            a:    7, b: 2,
            want: 3.5,
        },
        {
            name:      "divide by zero",
            a:         10, b: 0,
            wantErr:   true,
            errSubstr: "cannot divide by zero",
        },
        {
            name: "negative dividend",
            a:    -10, b: 2,
            want: -5.0,
        },
        {
            name: "both negative",
            a:    -10, b: -2,
            want: 5.0,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)

            // Check error expectation
            if tt.wantErr {
                if err == nil {
                    t.Fatalf("Divide(%v, %v) expected error, got nil", tt.a, tt.b)
                }
                if !strings.Contains(err.Error(), tt.errSubstr) {
                    t.Errorf("error = %q; want substring %q", err.Error(), tt.errSubstr)
                }
                return
            }
            if err != nil {
                t.Fatalf("Divide(%v, %v) unexpected error: %v", tt.a, tt.b, err)
            }

            // Check result
            if got != tt.want {
                t.Errorf("Divide(%v, %v) = %v; want %v", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### Comparison: Go Table-Driven vs Jest `each`

**Go:**

```go
tests := []struct {
    name     string
    input    string
    expected bool
}{
    {"empty string", "", false},
    {"valid email", "a@b.com", true},
    {"missing @", "ab.com", false},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        if got := IsValidEmail(tt.input); got != tt.expected {
            t.Errorf("IsValidEmail(%q) = %v; want %v", tt.input, got, tt.expected)
        }
    })
}
```

**Jest:**

```javascript
test.each([
  ['empty string', '', false],
  ['valid email', 'a@b.com', true],
  ['missing @', 'ab.com', false],
])('%s: IsValidEmail(%s) -> %s', (name, input, expected) => {
  expect(isValidEmail(input)).toBe(expected);
});
```

Go's approach is more verbose but more explicit. You define a clear struct with named fields rather than relying on positional array indexing. The Go version is easier to read when test cases have 5+ fields.

---

## 4. Test Helpers

### `t.Helper()`

When you extract common assertion logic into a helper function, Go will report the wrong line number on failure -- it reports the line inside the helper, not the line that called the helper. `t.Helper()` fixes this.

```go
// WITHOUT t.Helper() -- error points to this function, not the caller
func assertEqual(t *testing.T, got, want int) {
    if got != want {
        t.Errorf("got %d; want %d", got, want) // line 5 of helper -- unhelpful!
    }
}

// WITH t.Helper() -- error points to the caller
func assertEqual(t *testing.T, got, want int) {
    t.Helper() // tells the test framework: "I'm a helper, skip me in stack traces"
    if got != want {
        t.Errorf("got %d; want %d", got, want) // now reports the CALLER's line
    }
}
```

Usage:

```go
func TestMath(t *testing.T) {
    assertEqual(t, Add(2, 3), 5)    // if this fails, THIS line is reported
    assertEqual(t, Add(0, 0), 0)
    assertEqual(t, Add(-1, 1), 0)
}
```

### Building Richer Helpers

```go
// assertError checks that an error was returned and it contains the expected substring.
func assertError(t *testing.T, err error, wantSubstr string) {
    t.Helper()
    if err == nil {
        t.Fatal("expected an error but got nil")
    }
    if !strings.Contains(err.Error(), wantSubstr) {
        t.Errorf("error = %q; want substring %q", err.Error(), wantSubstr)
    }
}

// assertNoError fails the test immediately if err is not nil.
func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}
```

### `t.Cleanup()`

Register functions that run after the test completes, regardless of pass/fail. Cleanup functions run in LIFO (last registered, first executed) order.

```go
func TestWithTempFile(t *testing.T) {
    // Create a temp file for the test
    f, err := os.CreateTemp("", "test-*")
    if err != nil {
        t.Fatal(err)
    }

    // Register cleanup -- will run when this test function returns
    t.Cleanup(func() {
        os.Remove(f.Name())
    })

    // ... use f in your test ...
}
```

### The `testdata` Directory

Go has a special convention: any directory named `testdata` is ignored by the Go toolchain during builds. Use it to store test fixtures -- JSON files, SQL dumps, golden files, etc.

```
mypackage/
    parser.go
    parser_test.go
    testdata/
        valid_input.json
        invalid_input.json
        expected_output.golden
```

Access it in tests:

```go
func TestParseConfig(t *testing.T) {
    // testdata path is relative to the package directory
    data, err := os.ReadFile("testdata/valid_input.json")
    if err != nil {
        t.Fatal(err)
    }

    config, err := ParseConfig(data)
    if err != nil {
        t.Fatalf("ParseConfig() error: %v", err)
    }

    if config.Name != "myapp" {
        t.Errorf("Name = %q; want %q", config.Name, "myapp")
    }
}
```

### Golden File Testing

A common pattern: compare output against a saved "golden" file. Use `-update` flag to regenerate golden files.

```go
var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
    output := Render(inputData)

    goldenPath := filepath.Join("testdata", t.Name()+".golden")

    if *update {
        // Regenerate the golden file
        err := os.WriteFile(goldenPath, []byte(output), 0644)
        if err != nil {
            t.Fatal(err)
        }
    }

    expected, err := os.ReadFile(goldenPath)
    if err != nil {
        t.Fatal(err)
    }

    if output != string(expected) {
        t.Errorf("output does not match golden file.\nGot:\n%s\nWant:\n%s", output, expected)
    }
}
```

Update golden files: `go test -run TestRender -update`

### Comparison: Go Helpers vs Jest `beforeEach` / `afterEach`

**Jest:**

```javascript
describe('Database tests', () => {
  let db;

  beforeEach(async () => {
    db = await createTestDB();
  });

  afterEach(async () => {
    await db.close();
    await deleteTestDB();
  });

  test('inserts a record', async () => {
    await db.insert({ name: 'Alice' });
    expect(await db.count()).toBe(1);
  });
});
```

**Go equivalent:**

```go
func setupTestDB(t *testing.T) *DB {
    t.Helper()
    db, err := createTestDB()
    if err != nil {
        t.Fatal(err)
    }

    t.Cleanup(func() {
        db.Close()
        deleteTestDB()
    })

    return db
}

func TestInsertRecord(t *testing.T) {
    db := setupTestDB(t) // setup + automatic cleanup

    err := db.Insert(Record{Name: "Alice"})
    if err != nil {
        t.Fatal(err)
    }

    count, err := db.Count()
    if err != nil {
        t.Fatal(err)
    }
    if count != 1 {
        t.Errorf("count = %d; want 1", count)
    }
}
```

Go does not have `beforeEach`/`afterEach` lifecycle hooks. Instead, you write a helper function that returns the resource and registers cleanup. This is simpler, more explicit, and avoids the hidden state that lifecycle hooks introduce.

---

## 5. Subtests

### `t.Run()` for Organization

`t.Run()` creates a named subtest within a parent test. You already saw this in table-driven tests, but it is useful far beyond that.

```go
func TestUserValidation(t *testing.T) {
    t.Run("empty name is rejected", func(t *testing.T) {
        user := User{Name: "", Email: "a@b.com"}
        err := user.Validate()
        if err == nil {
            t.Error("expected error for empty name")
        }
    })

    t.Run("empty email is rejected", func(t *testing.T) {
        user := User{Name: "Alice", Email: ""}
        err := user.Validate()
        if err == nil {
            t.Error("expected error for empty email")
        }
    })

    t.Run("valid user passes", func(t *testing.T) {
        user := User{Name: "Alice", Email: "alice@example.com"}
        err := user.Validate()
        if err != nil {
            t.Errorf("unexpected error: %v", err)
        }
    })
}
```

### Running Specific Subtests

```bash
# Run all subtests of TestUserValidation
go test -run TestUserValidation -v

# Run only the "valid user passes" subtest
go test -run TestUserValidation/valid_user_passes -v

# Run all subtests containing "rejected"
go test -run TestUserValidation/rejected -v
```

Note: spaces in subtest names become underscores in the run pattern.

### Nested Subtests

You can nest `t.Run()` calls for deeper organization:

```go
func TestHTTPHandler(t *testing.T) {
    t.Run("GET", func(t *testing.T) {
        t.Run("returns 200 for existing resource", func(t *testing.T) {
            // ...
        })
        t.Run("returns 404 for missing resource", func(t *testing.T) {
            // ...
        })
    })

    t.Run("POST", func(t *testing.T) {
        t.Run("creates resource with valid body", func(t *testing.T) {
            // ...
        })
        t.Run("returns 400 for invalid body", func(t *testing.T) {
            // ...
        })
    })
}
```

Run just the GET tests: `go test -run TestHTTPHandler/GET`

### Parallel Subtests

Subtests can run in parallel for faster execution:

```go
func TestParallel(t *testing.T) {
    tests := []struct {
        name  string
        input int
        want  int
    }{
        {"case1", 1, 2},
        {"case2", 2, 4},
        {"case3", 3, 6},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // this subtest runs in parallel with other parallel subtests

            // IMPORTANT: tt is safe here because t.Run receives a NEW
            // variable in each iteration (Go 1.22+). In older Go versions,
            // you needed: tt := tt (re-declare to capture loop variable).
            got := Double(tt.input)
            if got != tt.want {
                t.Errorf("Double(%d) = %d; want %d", tt.input, got, tt.want)
            }
        })
    }
}
```

### Comparison: Go Subtests vs Jest `describe` / `it`

**Jest:**

```javascript
describe('User Validation', () => {
  it('rejects empty name', () => {
    expect(() => validate({ name: '', email: 'a@b.com' })).toThrow();
  });

  it('rejects empty email', () => {
    expect(() => validate({ name: 'Alice', email: '' })).toThrow();
  });

  it('accepts valid user', () => {
    expect(() => validate({ name: 'Alice', email: 'alice@b.com' })).not.toThrow();
  });
});
```

**Go equivalent:**

```go
func TestUserValidation(t *testing.T) {
    t.Run("rejects empty name", func(t *testing.T) {
        err := Validate(User{Name: "", Email: "a@b.com"})
        if err == nil {
            t.Error("expected error")
        }
    })

    t.Run("rejects empty email", func(t *testing.T) {
        err := Validate(User{Name: "Alice", Email: ""})
        if err == nil {
            t.Error("expected error")
        }
    })

    t.Run("accepts valid user", func(t *testing.T) {
        err := Validate(User{Name: "Alice", Email: "alice@b.com"})
        if err != nil {
            t.Errorf("unexpected error: %v", err)
        }
    })
}
```

The structure maps cleanly. `TestUserValidation` is the `describe` block. Each `t.Run` is an `it` block. The key difference is that Go subtests are just function calls -- there is no special DSL or callback registration system.

---

## 6. Test Coverage

### Running Coverage

```bash
# Show coverage percentage
go test -cover ./...
# Output: ok  mypackage  0.003s  coverage: 85.7% of statements

# Generate a coverage profile
go test -coverprofile=coverage.out ./...

# View coverage in the terminal, function by function
go tool cover -func=coverage.out
# Output:
# mypackage/math.go:5:    Add          100.0%
# mypackage/math.go:10:   Divide       75.0%
# total:                   (statements) 85.7%

# Generate an HTML report -- opens in browser
go tool cover -html=coverage.out

# Save HTML to file instead of opening browser
go tool cover -html=coverage.out -o coverage.html
```

### The HTML Report

The HTML report is incredibly useful. It color-codes your source:

- **Green** -- covered (executed during tests)
- **Red** -- not covered (never executed)
- **Gray** -- not trackable (declarations, comments)

This makes it immediately obvious which branches, error paths, and edge cases your tests miss.

### Coverage Modes

```bash
# Default: "set" mode -- was this statement executed? (yes/no)
go test -coverprofile=coverage.out ./...

# "count" mode -- how many times was each statement executed?
go test -covermode=count -coverprofile=coverage.out ./...

# "atomic" mode -- like count, but safe for parallel tests
go test -covermode=atomic -coverprofile=coverage.out ./...
```

### Cover Specific Packages

```bash
# Cover only this package
go test -cover ./mypackage

# Cover multiple packages and merge results
go test -coverprofile=coverage.out -coverpkg=./... ./...
```

The `-coverpkg` flag is important for integration tests. Without it, Go only measures coverage for the package being tested. With `-coverpkg=./...`, it measures coverage across all packages, so an integration test that exercises multiple packages gets proper coverage attribution.

### WHY Coverage Matters but Is Not Everything

Coverage tells you **what code your tests execute**, not **whether your tests are correct**. Consider:

```go
func TestAdd(t *testing.T) {
    Add(2, 3) // 100% coverage, ZERO assertions
}
```

This test achieves 100% coverage of `Add` but verifies nothing. A test that covers 80% with meaningful assertions is far more valuable than 100% coverage with empty tests.

**Guidelines:**
- **70-80% coverage** is a reasonable target for most projects.
- **100% coverage** is usually not worth the effort. Diminishing returns kick in hard past 90%.
- **Focus on critical paths.** Cover error handling, edge cases, and business logic thoroughly.
- **Use coverage to find gaps**, not as a score to maximize.

### Comparison: Go vs Node.js Coverage

| | Go | Node.js |
|---|---|---|
| Tool | `go test -cover` (built-in) | Istanbul / c8 / nyc (separate install) |
| Config | None | `.nycrc`, `jest --coverage` config |
| HTML report | `go tool cover -html` | `nyc report --reporter=html` |
| Merge profiles | `go tool covdata merge` (Go 1.20+) | `nyc merge` |
| CI integration | Just run the command | Need config + reporter packages |

---

## 7. Benchmarks

### WHY Benchmarking Is Built-In

Performance matters in Go's target domain (servers, CLI tools, infrastructure). By making benchmarks trivially easy to write, Go ensures that performance testing is not an afterthought.

In Node.js, benchmarking requires separate libraries (`benchmark.js`, `tinybench`) with different APIs and no integration with the test runner. In Go, benchmarks live alongside tests and run with the same `go test` command.

### Writing Benchmarks

Benchmark functions start with `Benchmark`, take `*testing.B`, and contain a loop over `b.N`:

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```

Run it:

```bash
go test -bench=.
# Output:
# BenchmarkAdd-8   1000000000   0.2500 ns/op
#    ^name   ^CPUs  ^iterations  ^time per operation
```

**How `b.N` works:** Go automatically determines the right value of `b.N`. It starts small (1, then 100, then 10000, etc.) and keeps increasing until the benchmark runs for a stable, statistically meaningful duration (at least 1 second by default). You do NOT set `b.N` yourself.

### Benchmarking with Setup

If your benchmark requires setup that should not be measured:

```go
func BenchmarkSort(b *testing.B) {
    // Setup: create a large slice (not measured)
    data := make([]int, 10000)
    for i := range data {
        data[i] = rand.Intn(10000)
    }

    b.ResetTimer() // reset the timer -- everything above is excluded

    for i := 0; i < b.N; i++ {
        // Make a copy so each iteration sorts unsorted data
        tmp := make([]int, len(data))
        copy(tmp, data)
        sort.Ints(tmp)
    }
}
```

### Memory Allocation Benchmarks

```go
func BenchmarkStringConcat(b *testing.B) {
    b.ReportAllocs() // report memory allocations

    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "x"
        }
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("x")
        }
        _ = sb.String()
    }
}
```

Output:

```
BenchmarkStringConcat-8      50000   28000 ns/op   10240 B/op   99 allocs/op
BenchmarkStringBuilder-8   1000000    1200 ns/op     512 B/op    4 allocs/op
```

This immediately shows that `strings.Builder` is **~23x faster** and uses **~20x less memory**. This kind of insight is trivial to get in Go and would require significant setup in Node.js.

### Sub-Benchmarks

Just like `t.Run`, you can use `b.Run` to create sub-benchmarks with different parameters:

```go
func BenchmarkSort(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := make([]int, size)
            for i := range data {
                data[i] = rand.Intn(size)
            }

            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                tmp := make([]int, len(data))
                copy(tmp, data)
                sort.Ints(tmp)
            }
        })
    }
}
```

Output:

```
BenchmarkSort/size=100-8       200000     5200 ns/op
BenchmarkSort/size=1000-8       20000    82000 ns/op
BenchmarkSort/size=10000-8       1000  1100000 ns/op
BenchmarkSort/size=100000-8       100 14000000 ns/op
```

### Comparing Benchmarks with `benchstat`

```bash
# Install benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# Run benchmarks and save results
go test -bench=. -count=10 > old.txt

# Make your optimization, then run again
go test -bench=. -count=10 > new.txt

# Compare
benchstat old.txt new.txt
```

`benchstat` provides statistically rigorous comparisons with confidence intervals, so you know whether a change is a real improvement or just noise.

### Benchmark Flags

```bash
# Run all benchmarks
go test -bench=.

# Run benchmarks matching a pattern
go test -bench=BenchmarkSort

# Set minimum benchmark time (default 1s)
go test -bench=. -benchtime=5s

# Set exact iteration count
go test -bench=. -benchtime=1000x

# Report memory allocations
go test -bench=. -benchmem

# Run benchmarks N times (for benchstat)
go test -bench=. -count=10
```

### Comparison: Go vs Node.js Benchmarking

**Go (built-in):**

```go
func BenchmarkFibonacci(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Fibonacci(20)
    }
}
// Run: go test -bench=BenchmarkFibonacci
```

**Node.js (requires benchmark.js or tinybench):**

```javascript
import Benchmark from 'benchmark';

const suite = new Benchmark.Suite();
suite
  .add('fibonacci', () => fibonacci(20))
  .on('complete', function() {
    console.log(this[0].toString());
  })
  .run();
// Run: node benchmark.js (separate file, separate run)
```

| Aspect | Go | Node.js |
|--------|-----|---------|
| Setup | Zero | Install benchmark.js / tinybench |
| Integration | Part of `go test` | Separate script, separate run |
| Memory profiling | `-benchmem` flag | Manual with `process.memoryUsage()` |
| Statistical comparison | `benchstat` | DIY or custom scripts |
| Sub-benchmarks | `b.Run()` | Manual loops |

---

## 8. Fuzzing

### What Is Fuzz Testing?

Fuzz testing feeds **random, automatically generated inputs** to your functions to find bugs you never thought to test for. Instead of you writing specific test cases, the fuzzer explores the input space and reports any input that causes a crash, panic, or test failure.

Go 1.18 introduced fuzz testing as a first-class feature of the `testing` package. This was a landmark addition -- very few languages have built-in fuzz testing.

### Why Fuzz Testing Matters

- **Humans are bad at edge cases.** We test `""`, `"hello"`, and `"a@b.com"` but forget about `"\x00\xff\xfe"` or strings with 10 million characters.
- **Security vulnerabilities** often live in unexpected input combinations.
- **Parsing code** is especially susceptible -- malformed JSON, invalid UTF-8, deeply nested structures.
- **Fuzzers have found thousands of real bugs** in production software (including Go's standard library itself).

### Writing Fuzz Tests

Fuzz functions start with `Fuzz`, take `*testing.F`, and use `f.Add()` to provide seed inputs (the "corpus"):

```go
func FuzzReverse(f *testing.F) {
    // Seed corpus: provide known interesting inputs
    f.Add("hello")
    f.Add("")
    f.Add("a")
    f.Add("abc")
    f.Add("!@#$%")

    // The fuzz target: called with random variations of the seed inputs
    f.Fuzz(func(t *testing.T, s string) {
        reversed := Reverse(s)
        doubleReversed := Reverse(reversed)

        // Property: reversing twice should give back the original
        if s != doubleReversed {
            t.Errorf("Reverse(Reverse(%q)) = %q; want %q", s, doubleReversed, s)
        }

        // Property: length should be preserved
        if len(reversed) != len(s) {
            t.Errorf("len(Reverse(%q)) = %d; want %d", s, len(reversed), len(s))
        }
    })
}
```

### Running Fuzz Tests

```bash
# Run as a normal test (only uses the seed corpus)
go test -run FuzzReverse

# Actually fuzz -- generates random inputs for 30 seconds
go test -fuzz FuzzReverse -fuzztime=30s

# Fuzz indefinitely (Ctrl+C to stop)
go test -fuzz FuzzReverse

# Fuzz with a minimum duration
go test -fuzz FuzzReverse -fuzztime=5m
```

When fuzzing discovers a failing input:

```
--- FAIL: FuzzReverse (0.50s)
    --- FAIL: FuzzReverse/abc123 (0.00s)
        reverse_test.go:20: Reverse(Reverse("ñ")) = "�"; want "ñ"

    Failing input written to testdata/fuzz/FuzzReverse/abc123
    To re-run this failing input, run:
        go test -run FuzzReverse/abc123
```

Go automatically saves the failing input to `testdata/fuzz/` so it becomes a permanent regression test.

### A Real-World Fuzz Example: URL Parser

```go
func FuzzParseURL(f *testing.F) {
    // Seed with known URLs
    f.Add("https://example.com")
    f.Add("http://localhost:8080/path?q=1")
    f.Add("ftp://user:pass@host.com/file")
    f.Add("")
    f.Add("not-a-url")

    f.Fuzz(func(t *testing.T, input string) {
        parsed, err := ParseURL(input)
        if err != nil {
            // Errors are fine -- we just want no panics
            return
        }

        // Property: if parsing succeeded, reconstructing should give a valid URL
        reconstructed := parsed.String()
        reparsed, err := ParseURL(reconstructed)
        if err != nil {
            t.Errorf("ParseURL(%q) succeeded, but ParseURL(%q) failed: %v",
                input, reconstructed, err)
        }

        // Property: scheme should not be empty if parsing succeeded
        if parsed.Scheme == "" {
            t.Errorf("ParseURL(%q) returned empty scheme", input)
        }

        _ = reparsed // suppress unused variable warning if needed
    })
}
```

### Supported Fuzz Types

The `f.Fuzz` target function can accept these types as parameters (after `*testing.T`):

- `string`
- `[]byte`
- `int`, `int8`, `int16`, `int32`, `int64`
- `uint`, `uint8`, `uint16`, `uint32`, `uint64`
- `float32`, `float64`
- `bool`
- `rune`

You can combine multiple parameters:

```go
f.Add("hello", 42, true)
f.Fuzz(func(t *testing.T, s string, n int, flag bool) {
    // fuzzer varies all three parameters independently
})
```

### The Corpus

The fuzzer maintains a corpus (collection of interesting inputs) in two places:

1. **Seed corpus** -- inputs you provide with `f.Add()`.
2. **Generated corpus** -- inputs the fuzzer discovers that increase code coverage, stored in `testdata/fuzz/<FuzzFuncName>/`.

The generated corpus is cached and reused across fuzzing sessions, so the fuzzer builds on previous work.

### Comparison: Go vs Node.js Fuzzing

| Aspect | Go | Node.js |
|--------|-----|---------|
| Built-in | Yes (Go 1.18+) | No |
| Tools available | `go test -fuzz` | jsfuzz, fast-check (property testing) |
| Integration | Part of test runner | Separate tool/library |
| Corpus management | Automatic | Manual |
| Regression tests | Auto-saved to testdata | Manual |
| Community adoption | Widespread | Rare |

Node.js does not have a standard fuzzing story. Libraries like `fast-check` offer property-based testing (related but distinct), and `jsfuzz` exists but is not widely adopted. Go's built-in support makes fuzz testing accessible to every Go developer.

---

## 9. Test Doubles -- Interfaces for Mocking

### Go's Approach: Interfaces, Not Magic

Go does not have built-in mocking primitives like `jest.mock()` or `jest.spyOn()`. Instead, Go leans on **interfaces** -- define what you depend on as an interface, then swap in a test implementation.

This is a core design pattern in Go, not a workaround. It produces cleaner, more decoupled code.

### The Pattern

**Step 1: Define the dependency as an interface.**

```go
// In your production code
type UserRepository interface {
    GetByID(id string) (*User, error)
    Save(user *User) error
}

type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) GetUser(id string) (*User, error) {
    user, err := s.repo.GetByID(id)
    if err != nil {
        return nil, fmt.Errorf("getting user %s: %w", id, err)
    }
    return user, nil
}
```

**Step 2: Create a mock implementation in your test.**

```go
// In your test file
type mockUserRepo struct {
    getUserFunc func(id string) (*User, error)
    saveFunc    func(user *User) error
}

func (m *mockUserRepo) GetByID(id string) (*User, error) {
    return m.getUserFunc(id)
}

func (m *mockUserRepo) Save(user *User) error {
    return m.saveFunc(user)
}
```

**Step 3: Use the mock in your test.**

```go
func TestGetUser(t *testing.T) {
    t.Run("returns user when found", func(t *testing.T) {
        mock := &mockUserRepo{
            getUserFunc: func(id string) (*User, error) {
                return &User{ID: id, Name: "Alice"}, nil
            },
        }

        svc := NewUserService(mock)
        user, err := svc.GetUser("123")

        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
        if user.Name != "Alice" {
            t.Errorf("Name = %q; want %q", user.Name, "Alice")
        }
    })

    t.Run("returns error when not found", func(t *testing.T) {
        mock := &mockUserRepo{
            getUserFunc: func(id string) (*User, error) {
                return nil, fmt.Errorf("not found")
            },
        }

        svc := NewUserService(mock)
        _, err := svc.GetUser("456")

        if err == nil {
            t.Fatal("expected error, got nil")
        }
    })
}
```

### Verifying Call Arguments

You can capture and verify arguments:

```go
func TestSaveUser(t *testing.T) {
    var savedUser *User

    mock := &mockUserRepo{
        saveFunc: func(user *User) error {
            savedUser = user // capture the argument
            return nil
        },
    }

    svc := NewUserService(mock)
    err := svc.CreateUser("Alice", "alice@example.com")

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    // Verify the argument
    if savedUser == nil {
        t.Fatal("Save was never called")
    }
    if savedUser.Name != "Alice" {
        t.Errorf("saved Name = %q; want %q", savedUser.Name, "Alice")
    }
    if savedUser.Email != "alice@example.com" {
        t.Errorf("saved Email = %q; want %q", savedUser.Email, "alice@example.com")
    }
}
```

### Popular Mocking Libraries

While manual mocks work well for small interfaces, larger projects often use libraries.

**`gomock` (official Google mock library):**

```bash
go install go.uber.org/mock/mockgen@latest
```

```go
//go:generate mockgen -source=repository.go -destination=mock_repository_test.go -package=mypackage

func TestWithGoMock(t *testing.T) {
    ctrl := gomock.NewController(t)

    mockRepo := NewMockUserRepository(ctrl)
    mockRepo.EXPECT().
        GetByID("123").
        Return(&User{ID: "123", Name: "Alice"}, nil).
        Times(1)

    svc := NewUserService(mockRepo)
    user, err := svc.GetUser("123")

    if err != nil {
        t.Fatal(err)
    }
    if user.Name != "Alice" {
        t.Errorf("Name = %q; want Alice", user.Name)
    }
}
```

**`testify/mock`:**

```go
type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) GetByID(id string) (*User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func TestWithTestifyMock(t *testing.T) {
    mockRepo := new(MockUserRepo)
    mockRepo.On("GetByID", "123").Return(&User{ID: "123", Name: "Alice"}, nil)

    svc := NewUserService(mockRepo)
    user, err := svc.GetUser("123")

    if err != nil {
        t.Fatal(err)
    }
    if user.Name != "Alice" {
        t.Errorf("Name = %q; want Alice", user.Name)
    }

    mockRepo.AssertExpectations(t) // verify all expected calls were made
}
```

### Comparison: Go vs Jest Mocking

**Jest:**

```javascript
// Jest can mock ANY module -- even ones you don't own
jest.mock('./database');
const db = require('./database');
db.getUser.mockResolvedValue({ id: '123', name: 'Alice' });

// Jest can spy on methods
jest.spyOn(console, 'log');
```

**Go:**

```go
// Go requires you to depend on interfaces, not concrete types
// You can only mock what's behind an interface

// This is GOOD -- testable:
type UserService struct {
    repo UserRepository // interface
}

// This is BAD -- not testable without the real DB:
type UserService struct {
    repo *PostgresUserRepo // concrete type
}
```

Key differences:

| Aspect | Go | Jest |
|--------|-----|------|
| Mechanism | Interfaces (compile-time) | Runtime monkey-patching |
| Scope | Only interface-typed dependencies | Any module, function, or method |
| Design impact | Forces good architecture (dependency injection) | Can test poorly designed code |
| Type safety | Full type safety -- mock must implement interface | No compile-time checks |
| Magic level | Zero -- just a struct implementing an interface | High -- module system interception |

Go's approach is more restrictive but produces better code. If you cannot mock something, it means your code has a tight coupling problem that should be fixed.

---

## 10. Integration Tests

### Build Tags for Test Separation

Use build tags to separate unit tests from integration tests that require external dependencies (databases, APIs, etc.).

```go
//go:build integration

package mypackage

import "testing"

func TestDatabaseIntegration(t *testing.T) {
    db, err := sql.Open("postgres", os.Getenv("TEST_DATABASE_URL"))
    if err != nil {
        t.Fatal(err)
    }
    defer db.Close()

    // ... run real database tests ...
}
```

Run them selectively:

```bash
# Run only unit tests (default -- skips integration tag)
go test ./...

# Run only integration tests
go test -tags=integration ./...

# Run both
go test -tags=integration ./...
# (Unit tests always run; integration tests run when the tag is present)
```

### Using `-short` Flag

An alternative approach using the `-short` flag:

```go
func TestDatabaseQuery(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }

    // ... slow integration test ...
}
```

```bash
# Run all tests including slow ones
go test ./...

# Skip slow tests
go test -short ./...
```

### `TestMain` for Global Setup/Teardown

`TestMain` gives you control over the entire test execution for a package. It runs before any tests and lets you do global setup and teardown.

```go
package mypackage

import (
    "os"
    "testing"
)

var testDB *sql.DB

func TestMain(m *testing.M) {
    // ---- SETUP ----
    var err error
    testDB, err = sql.Open("postgres", os.Getenv("TEST_DATABASE_URL"))
    if err != nil {
        fmt.Fprintf(os.Stderr, "cannot connect to test DB: %v\n", err)
        os.Exit(1)
    }

    // Run migrations
    if err := runMigrations(testDB); err != nil {
        fmt.Fprintf(os.Stderr, "migration failed: %v\n", err)
        os.Exit(1)
    }

    // ---- RUN TESTS ----
    exitCode := m.Run()

    // ---- TEARDOWN ----
    testDB.Close()
    // Clean up test database, temp files, etc.

    os.Exit(exitCode)
}

func TestInsertUser(t *testing.T) {
    // testDB is available here -- set up in TestMain
    _, err := testDB.Exec("INSERT INTO users (name) VALUES ($1)", "Alice")
    if err != nil {
        t.Fatal(err)
    }
    // ...
}
```

**Important:** `TestMain` replaces the default test runner for the package. You MUST call `m.Run()` to execute the tests, and you MUST call `os.Exit()` with the result. If you forget `m.Run()`, no tests will execute. If you forget `os.Exit()`, the process will always exit 0 even on failure.

### HTTP Integration Testing with `httptest`

Go's `net/http/httptest` package makes it easy to test HTTP handlers without starting a real server:

```go
package api

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
)

func TestHealthEndpoint(t *testing.T) {
    // Create a request
    req := httptest.NewRequest(http.MethodGet, "/health", nil)

    // Create a response recorder
    w := httptest.NewRecorder()

    // Call the handler directly
    HealthHandler(w, req)

    // Check the response
    resp := w.Result()
    if resp.StatusCode != http.StatusOK {
        t.Errorf("status = %d; want %d", resp.StatusCode, http.StatusOK)
    }
}

func TestCreateUser(t *testing.T) {
    body := `{"name": "Alice", "email": "alice@example.com"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    w := httptest.NewRecorder()

    CreateUserHandler(w, req)

    resp := w.Result()
    if resp.StatusCode != http.StatusCreated {
        t.Errorf("status = %d; want %d", resp.StatusCode, http.StatusCreated)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        t.Fatalf("cannot decode response: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("Name = %q; want %q", user.Name, "Alice")
    }
}
```

### Full Test Server with `httptest.NewServer`

When you need to test HTTP client code:

```go
func TestAPIClient(t *testing.T) {
    // Create a test server
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/users/123" {
            json.NewEncoder(w).Encode(User{ID: "123", Name: "Alice"})
            return
        }
        http.NotFound(w, r)
    }))
    defer server.Close()

    // Point your client at the test server
    client := NewAPIClient(server.URL)
    user, err := client.GetUser("123")

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("Name = %q; want %q", user.Name, "Alice")
    }
}
```

### Comparison: Go vs Node.js Integration Tests

| Aspect | Go | Node.js |
|--------|-----|---------|
| Build tags | `//go:build integration` | Custom env vars + scripts |
| Global setup | `TestMain` | Jest `globalSetup`/`globalTeardown` |
| HTTP testing | `httptest` (built-in) | `supertest` (separate package) |
| Skip tests | `t.Skip()`, `-short` | `test.skip()`, `--testPathPattern` |
| Test organization | Packages + build tags | Directories + config patterns |

---

## 11. Testify Package

### Why Testify Exists

Go's standard testing philosophy is minimalist: use `if` statements and `t.Error`. Some developers find this verbose, especially when:

- You need deep equality checks on complex structs.
- Error messages for failed comparisons become repetitive.
- You want `assert.Equal` instead of writing `if got != want { t.Errorf(...) }` everywhere.

The `testify` package is the most popular third-party testing library in Go. It provides three sub-packages.

```bash
go get github.com/stretchr/testify
```

### `assert` -- Non-Fatal Assertions

```go
package mypackage

import (
    "testing"

    "github.com/stretchr/testify/assert"
)

func TestWithAssert(t *testing.T) {
    // assert functions log failures but do NOT stop the test
    result := Add(2, 3)
    assert.Equal(t, 5, result)                    // checks equality
    assert.Equal(t, 5, result, "Add should work")  // with custom message
    assert.NotEqual(t, 0, result)
    assert.True(t, result > 0)
    assert.Nil(t, err)
    assert.NotNil(t, user)
    assert.Contains(t, "hello world", "world")
    assert.Len(t, items, 3)
    assert.Empty(t, slice)

    // Error assertions
    _, err := Divide(10, 0)
    assert.Error(t, err)
    assert.ErrorContains(t, err, "divide by zero")
    assert.ErrorIs(t, err, ErrDivideByZero) // checks error wrapping chain

    // Deep equality for structs, slices, maps
    expected := User{Name: "Alice", Age: 30}
    actual := GetUser("123")
    assert.Equal(t, expected, actual)  // uses reflect.DeepEqual internally
}
```

On failure, testify gives you rich, readable output:

```
--- FAIL: TestWithAssert (0.00s)
    user_test.go:15:
        Error Trace:    user_test.go:15
        Error:          Not equal:
                        expected: 5
                        actual  : 4
        Test:           TestWithAssert
```

### `require` -- Fatal Assertions

Same API as `assert`, but **stops the test immediately** on failure (like `t.Fatal`):

```go
import "github.com/stretchr/testify/require"

func TestWithRequire(t *testing.T) {
    user, err := GetUser("123")
    require.NoError(t, err)         // stops here if err != nil
    require.NotNil(t, user)          // stops here if user is nil

    // Safe to access user fields because require ensured it's not nil
    assert.Equal(t, "Alice", user.Name)
    assert.Equal(t, 30, user.Age)
}
```

**Rule of thumb:** Use `require` for preconditions (setup, getting a handle). Use `assert` for the actual test assertions. This way you see all failures in the assertions without the test panicking from a nil pointer.

### `suite` -- Test Suites

For complex test setups with shared state:

```go
import (
    "testing"

    "github.com/stretchr/testify/suite"
)

type UserServiceTestSuite struct {
    suite.Suite
    db      *sql.DB
    service *UserService
}

// SetupSuite runs once before all tests in the suite
func (s *UserServiceTestSuite) SetupSuite() {
    db, err := sql.Open("postgres", os.Getenv("TEST_DB_URL"))
    s.Require().NoError(err)
    s.db = db
    s.service = NewUserService(db)
}

// TearDownSuite runs once after all tests
func (s *UserServiceTestSuite) TearDownSuite() {
    s.db.Close()
}

// SetupTest runs before each test
func (s *UserServiceTestSuite) SetupTest() {
    _, err := s.db.Exec("DELETE FROM users")
    s.Require().NoError(err)
}

// Test methods must start with "Test"
func (s *UserServiceTestSuite) TestCreateUser() {
    user, err := s.service.Create("Alice", "alice@example.com")
    s.Require().NoError(err)
    s.Assert().Equal("Alice", user.Name)
    s.Assert().NotEmpty(user.ID)
}

func (s *UserServiceTestSuite) TestGetUser() {
    // Create a user first
    created, err := s.service.Create("Bob", "bob@example.com")
    s.Require().NoError(err)

    // Retrieve it
    fetched, err := s.service.GetByID(created.ID)
    s.Require().NoError(err)
    s.Assert().Equal(created.ID, fetched.ID)
    s.Assert().Equal("Bob", fetched.Name)
}

// Entry point: this is what go test actually runs
func TestUserServiceSuite(t *testing.T) {
    suite.Run(t, new(UserServiceTestSuite))
}
```

### The Debate: Testify vs Standard Library

**Arguments FOR testify:**

- Less boilerplate -- `assert.Equal(t, want, got)` vs a 3-line `if` block.
- Better error messages by default -- shows expected vs actual clearly.
- `require` prevents nil pointer panics after failed setup.
- Deep equality for complex types is built in.

**Arguments AGAINST testify:**

- Go's philosophy is to minimize dependencies, even in tests.
- Standard `if` + `t.Errorf` is more explicit -- you see exactly what is happening.
- Custom error messages in standard tests can be more descriptive.
- Testify's `assert.Equal(t, expected, actual)` argument order is easy to mix up.
- The `suite` package introduces lifecycle hooks that Go deliberately avoided.

**Practical advice:** Both approaches work. For new Go developers, testify reduces friction. For experienced Go developers working on libraries or the standard library, raw `testing` is preferred. Many production codebases use `assert`/`require` but skip `suite`.

---

## 12. `go test` Flags

### Essential Flags Reference

```bash
# -v: verbose output (show all test names and log output)
go test -v ./...

# -run: run only tests matching a regex
go test -run TestAdd            # runs TestAdd, TestAddNegative, etc.
go test -run "^TestAdd$"        # runs exactly TestAdd
go test -run TestAdd/zeros      # runs the "zeros" subtest of TestAdd
go test -run "TestUser|TestOrder"  # runs tests matching either pattern

# -count: run tests N times (useful for detecting flaky tests)
go test -count=5 ./...
# NOTE: -count=1 also disables test result caching

# -race: enable the race detector (CRITICAL for concurrent code)
go test -race ./...

# -short: signal tests to skip slow operations
go test -short ./...

# -parallel: max number of tests to run in parallel (default: GOMAXPROCS)
go test -parallel=4 ./...

# -timeout: max time for all tests in a package (default: 10 minutes)
go test -timeout=30s ./...

# -failfast: stop at first test failure
go test -failfast ./...

# -shuffle: randomize test execution order (Go 1.17+)
go test -shuffle=on ./...

# -list: list tests matching a regex WITHOUT running them
go test -list ".*" ./...
```

### The Race Detector (`-race`)

The race detector is one of Go's most powerful testing tools. It detects data races at runtime -- when two goroutines access the same variable concurrently and at least one access is a write.

```go
// THIS CODE HAS A RACE CONDITION
var counter int

func increment() {
    counter++ // unsynchronized write
}

func TestRace(t *testing.T) {
    for i := 0; i < 100; i++ {
        go increment()
    }
    time.Sleep(time.Second)
    t.Log(counter)
}
```

```bash
go test -race
# Output:
# ==================
# WARNING: DATA RACE
# Write at 0x00c0000a4010 by goroutine 8:
#   mypackage.increment()
#       /path/to/code.go:6 +0x38
# Previous write at 0x00c0000a4010 by goroutine 7:
#   mypackage.increment()
#       /path/to/code.go:6 +0x38
# ==================
# FAIL
```

**Always run `go test -race` in CI.** It catches real bugs that are otherwise nearly impossible to reproduce.

### Combining Flags

```bash
# Run a specific test verbosely with race detection
go test -v -race -run TestConcurrentMap ./...

# Run all tests with coverage, race detection, 30s timeout
go test -race -cover -timeout=30s ./...

# Run benchmarks (not tests) with memory allocation reporting
go test -bench=. -benchmem -run=^$ ./...
# -run=^$ matches no tests, so only benchmarks run

# Run a specific benchmark 10 times for statistical comparison
go test -bench=BenchmarkSort -count=10 -benchtime=2s -run=^$

# Fuzz a specific function for 2 minutes
go test -fuzz=FuzzParseURL -fuzztime=2m

# Full CI pipeline command
go test -v -race -cover -coverprofile=coverage.out -shuffle=on -count=1 ./...
```

### Test Caching

Go caches test results. If you run `go test ./...` twice without changing any code, the second run is instant:

```
ok      mypackage   (cached)
```

To force re-execution:

```bash
# Method 1: use -count=1
go test -count=1 ./...

# Method 2: clean the cache
go clean -testcache
```

### Comparison: `go test` vs `npx jest`

| Flag / Feature | `go test` | `jest` |
|----------------|-----------|--------|
| Verbose | `-v` | `--verbose` |
| Run specific test | `-run TestName` | `--testNamePattern` or `.only` |
| Coverage | `-cover` | `--coverage` |
| Watch mode | Not built-in (use `gowatch`) | `--watch` (built-in) |
| Parallel | `-parallel=N` | `--maxWorkers=N` |
| Fail fast | `-failfast` | `--bail` |
| Timeout | `-timeout=30s` | `--testTimeout=30000` |
| Race detection | `-race` | N/A (single-threaded) |
| Fuzz testing | `-fuzz=FuzzName` | N/A |
| Benchmarks | `-bench=.` | N/A |
| Shuffle | `-shuffle=on` | `--randomize` |
| Caching | Automatic | `--cache` |
| Config file | None needed | `jest.config.js` |

Note how many Go features (race detection, fuzzing, benchmarks) simply have no Jest equivalent. These are areas where Go's built-in tooling provides capabilities that the Node.js ecosystem has to assemble from multiple tools -- or cannot provide at all.

---

## 13. Key Takeaways

1. **Testing is built into Go's DNA.** `go test` is all you need -- no config files, no framework choices, no `devDependencies`. This is by design and it works remarkably well.

2. **Table-driven tests are the idiom.** If you write Go tests any other way, experienced Go developers will immediately refactor them into tables. Learn this pattern cold.

3. **`t.Error` continues, `t.Fatal` stops.** Use `Fatal` for preconditions, `Error` for assertions. This gives maximum diagnostic information per test run.

4. **`t.Helper()` makes helpers behave.** Always call it as the first line of any test helper function so error messages point to the right place.

5. **Coverage is a tool, not a goal.** Use `go test -cover` and `go tool cover -html` to find untested code paths, not to hit an arbitrary percentage.

6. **Benchmarks are trivially easy.** `b.N` loop, `b.ResetTimer()`, `b.ReportAllocs()` -- that is the entire API. No excuse not to benchmark performance-critical code.

7. **Fuzz testing finds bugs you cannot imagine.** Spend 30 seconds writing a fuzz test for your parser, validator, or encoder. The fuzzer has found bugs in the Go standard library itself.

8. **Mock through interfaces, not magic.** If something is hard to test, the problem is your architecture, not your mocking tool. Depend on interfaces, inject dependencies.

9. **`-race` is non-negotiable.** Run `go test -race ./...` in every CI pipeline. Data races cause the worst kind of bugs -- intermittent, unreproducible, and catastrophic.

10. **Go trades verbosity for clarity.** `if got != want { t.Errorf(...) }` is more characters than `expect(got).toBe(want)`, but it is explicit, requires no framework, and gives you full control over error messages.

---

## 14. Practice Exercises

### Exercise 1: Table-Driven Test Fundamentals

Write a function `Factorial(n int) (int, error)` that returns the factorial of n, or an error for negative inputs. Write comprehensive table-driven tests covering:

- Positive numbers (1! = 1, 5! = 120, 10! = 3628800)
- Zero (0! = 1)
- Negative numbers (should return an error)
- Large numbers (consider overflow)

### Exercise 2: Test Coverage Gap Analysis

Write a function `Grade(score int) string` that returns:
- "A" for 90-100
- "B" for 80-89
- "C" for 70-79
- "D" for 60-69
- "F" for 0-59
- "Invalid" for anything outside 0-100

Write tests, then run `go test -cover -coverprofile=coverage.out` followed by `go tool cover -html=coverage.out`. Identify any uncovered branches and add tests until you reach 100%.

### Exercise 3: Benchmark Comparison

Write two implementations of a function that checks whether a string is a palindrome:
1. One that reverses the string and compares.
2. One that uses two pointers from each end.

Write benchmarks for both with various input sizes (10, 100, 1000, 10000 characters). Determine which is faster and explain why.

### Exercise 4: Fuzz Testing a Parser

Write a simple `ParseKeyValue(s string) (key, value string, err error)` function that parses strings in the format `"key=value"`. Then write a fuzz test that verifies:
- Parsing does not panic on any input.
- If parsing succeeds, `key + "=" + value` reconstructs a valid input.
- Both key and value are non-empty on success.

Run the fuzzer for 60 seconds and fix any bugs it discovers.

### Exercise 5: Interface Mocking

Design a `NotificationService` that depends on a `Sender` interface:

```go
type Sender interface {
    Send(to, subject, body string) error
}
```

Write tests for `NotificationService.NotifyUser()` using a hand-written mock that:
- Verifies the correct recipient receives the notification.
- Verifies the subject contains the user's name.
- Tests the error path when `Send` fails.

### Exercise 6: Integration Test with `httptest`

Build a simple REST API with `GET /users/:id` and `POST /users` endpoints. Write integration tests using `httptest` that:
- Create a user via POST and verify the 201 response.
- Retrieve the created user via GET and verify the data matches.
- Request a non-existent user and verify 404 is returned.
- Send an invalid POST body and verify 400 is returned.

### Exercise 7: The Full Pipeline

Create a small package with at least 3 exported functions. Write:
1. Table-driven unit tests for each function.
2. Benchmarks for the most performance-critical function.
3. A fuzz test for any function that accepts string input.
4. Run the complete pipeline:

```bash
# Unit tests with race detection and coverage
go test -v -race -cover -coverprofile=coverage.out ./...

# View coverage
go tool cover -html=coverage.out

# Benchmarks with memory reporting
go test -bench=. -benchmem -run=^$ ./...

# Fuzz for 30 seconds
go test -fuzz=FuzzYourFunction -fuzztime=30s ./...
```

Document what you find: coverage gaps, performance characteristics, and any bugs the fuzzer discovers.
