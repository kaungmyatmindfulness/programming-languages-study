# Chapter 4: Functions, Closures, and Methods

## Table of Contents

1. [Function Declaration](#1-function-declaration)
2. [Named Return Values](#2-named-return-values)
3. [Variadic Functions](#3-variadic-functions)
4. [First-Class Functions](#4-first-class-functions)
5. [Anonymous Functions and Closures](#5-anonymous-functions-and-closures)
6. [Methods (Receiver Functions)](#6-methods-receiver-functions)
7. [The init() Function](#7-the-init-function)
8. [The Blank Identifier (_)](#8-the-blank-identifier-_)
9. [Error Handling with Functions](#9-error-handling-with-functions)
10. [Go vs Node.js Comparison Summary](#10-go-vs-nodejs-comparison-summary)
11. [Key Takeaways](#11-key-takeaways)
12. [Practice Exercises](#12-practice-exercises)

---

## 1. Function Declaration

### The Basics

Every Go program starts with a function: `main()`. Functions in Go are declared with the `func` keyword, and unlike JavaScript, **parameter types are mandatory** because Go is statically typed.

```go
// Basic function declaration
func greet(name string) {
    fmt.Println("Hello,", name)
}

// Function with a return type
func add(a int, b int) int {
    return a + b
}

// When consecutive parameters share the same type, you can shorten it
func add(a, b int) int {
    return a + b
}
```

**Node.js equivalent:**

```js
function greet(name) {
    console.log("Hello,", name);
}

function add(a, b) {
    return a + b;
}
```

Notice in JS, there are no type annotations (unless you use TypeScript). In Go, the compiler enforces types at compile time -- there is no runtime type guessing.

### Multiple Return Values

This is one of Go's most distinctive features. A function can return **more than one value**. JavaScript cannot do this natively.

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result) // Result: 3.3333333333333335
}
```

**Node.js -- the closest pattern:**

```js
function divide(a, b) {
    if (b === 0) {
        return { result: null, error: new Error("cannot divide by zero") };
    }
    return { result: a / b, error: null };
}

const { result, error } = divide(10, 3);
if (error) {
    console.log("Error:", error.message);
} else {
    console.log("Result:", result);
}
```

### WHY Does Go Support Multiple Returns?

This is a critical design decision. Go's creators (Rob Pike, Ken Thompson, Robert Griesemer) deliberately chose multiple return values for one dominant reason: **explicit error handling**.

In most languages, errors are handled through exceptions (`try/catch`). The Go team believed exceptions create invisible control flow -- when you read code, you cannot tell which lines might throw and where execution will jump to. Multiple returns make the error path **visible** at every call site.

The `(result, error)` pattern became Go's idiom:

```go
file, err := os.Open("data.txt")
if err != nil {
    log.Fatal(err)
}
// use file...
```

Every time you call a function that can fail, **you see it**. There is no hidden `throw` that silently propagates up the call stack. This is not accidental -- it is the entire philosophy.

Other uses of multiple returns beyond error handling:

```go
// Returning a value and a boolean "ok" flag (comma-ok idiom)
value, ok := myMap["key"]
if !ok {
    fmt.Println("key not found")
}

// Returning multiple computed results
func minMax(numbers []int) (int, int) {
    min, max := numbers[0], numbers[0]
    for _, n := range numbers {
        if n < min {
            min = n
        }
        if n > max {
            max = n
        }
    }
    return min, max
}
```

---

## 2. Named Return Values

Go lets you **name** your return values in the function signature. When you do, they are treated as variables declared at the top of the function, initialized to their zero values.

```go
func rectangleArea(width, height float64) (area float64) {
    area = width * height
    return // "naked return" -- returns the named value
}
```

This is equivalent to:

```go
func rectangleArea(width, height float64) float64 {
    area := width * height
    return area
}
```

### A More Realistic Example

Named returns shine when a function has multiple return values and complex logic:

```go
func parseUserInput(input string) (name string, age int, err error) {
    parts := strings.SplitN(input, ",", 2)
    if len(parts) != 2 {
        err = fmt.Errorf("invalid format: expected 'name,age'")
        return // returns "", 0, err (zero values for name and age)
    }

    name = strings.TrimSpace(parts[0])
    if name == "" {
        err = fmt.Errorf("name cannot be empty")
        return
    }

    age, err = strconv.Atoi(strings.TrimSpace(parts[1]))
    if err != nil {
        err = fmt.Errorf("invalid age: %w", err)
        return
    }

    return // returns name, age, nil
}
```

### When to Use Named Returns

**Use them when:**

- The function has multiple return values and their purpose is not obvious from the types alone. Named returns serve as documentation:

```go
// Without named returns -- what do these two ints mean?
func split(sum int) (int, int)

// With named returns -- self-documenting
func split(sum int) (quotient int, remainder int)
```

- You need to modify a return value in a `defer` statement (this is an advanced but important pattern):

```go
func readFile(path string) (content string, err error) {
    f, err := os.Open(path)
    if err != nil {
        return
    }
    defer func() {
        if closeErr := f.Close(); closeErr != nil && err == nil {
            err = closeErr // modifying the named return value
        }
    }()

    data, err := io.ReadAll(f)
    if err != nil {
        return
    }
    content = string(data)
    return
}
```

**Avoid them when:**

- The function is short and the return types are self-explanatory.
- Using naked returns makes the code harder to follow. If the function body is longer than ~10 lines, naked returns obscure what is actually being returned.

### Pros and Cons

| Pros | Cons |
|------|------|
| Self-documenting return values | Naked returns hide what is being returned |
| Required for modifying returns in `defer` | Can lead to accidental shadowing |
| Zero-value initialization is convenient | Longer function signatures |
| Shows up in generated documentation | Temptation to overuse naked returns |

### No JavaScript Equivalent

JavaScript has no concept of named return values. The closest pattern is returning an object with named fields:

```js
function parseUserInput(input) {
    // ...
    return { name, age, error: null };
}
```

But this is fundamentally different -- it is a single return value (an object), not multiple return values baked into the language.

---

## 3. Variadic Functions

A variadic function accepts **a variable number of arguments** of the same type. The syntax uses `...` before the type.

```go
func sum(numbers ...int) int {
    total := 0
    for _, n := range numbers {
        total += n
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))       // 6
    fmt.Println(sum(10, 20, 30, 40)) // 100
    fmt.Println(sum())               // 0

    // You can also pass a slice by "spreading" it with ...
    nums := []int{5, 10, 15}
    fmt.Println(sum(nums...)) // 30
}
```

### Comparison with JavaScript Rest Parameters

```js
function sum(...numbers) {
    return numbers.reduce((total, n) => total + n, 0);
}

console.log(sum(1, 2, 3));        // 6
console.log(sum(10, 20, 30, 40)); // 100
console.log(sum());               // 0

// Spreading an array
const nums = [5, 10, 15];
console.log(sum(...nums)); // 30
```

The syntax is almost identical. The key difference is **type safety**: in Go, `numbers ...int` means every argument must be an `int`. In JS, `...numbers` accepts anything.

### Important Rules for Variadic Parameters

1. **Only the last parameter** can be variadic:

```go
// Valid
func log(level string, messages ...string) { }

// INVALID -- variadic param must be last
// func log(messages ...string, level string) { }
```

2. Inside the function, the variadic parameter is a **slice**:

```go
func printAll(values ...string) {
    // values is of type []string
    fmt.Printf("Type: %T, Length: %d\n", values, len(values))
    for i, v := range values {
        fmt.Printf("  [%d] %s\n", i, v)
    }
}
```

3. Passing a slice requires the `...` spread operator:

```go
words := []string{"hello", "world"}
printAll(words...)    // correct
// printAll(words)    // COMPILE ERROR: cannot use []string as string
```

### Real-World Use: fmt.Println

The most famous variadic function in Go is one you have already been using:

```go
// Simplified signature of fmt.Println
func Println(a ...interface{}) (n int, err error)
```

It accepts any number of arguments of any type (`interface{}` is Go's "any" type, now also spelled `any` since Go 1.18).

---

## 4. First-Class Functions

In Go, functions are **first-class citizens**. They can be:
- Assigned to variables
- Passed as arguments to other functions
- Returned from functions

This is the same as JavaScript, where functions are objects. But Go adds **static typing** to function types.

### Function Types

Every function in Go has a type, defined by its parameter types and return types:

```go
// This function has type: func(int, int) int
func add(a, b int) int {
    return a + b
}

// You can declare a variable of that function type
var operation func(int, int) int

func main() {
    operation = add
    result := operation(3, 4)
    fmt.Println(result) // 7
}
```

### Named Function Types

You can give function types a name for clarity:

```go
type MathOperation func(int, int) int

func apply(a, b int, op MathOperation) int {
    return op(a, b)
}

func add(a, b int) int { return a + b }
func multiply(a, b int) int { return a * b }

func main() {
    fmt.Println(apply(3, 4, add))      // 7
    fmt.Println(apply(3, 4, multiply)) // 12
}
```

### Comparison with JavaScript

```js
// JavaScript -- functions are already first-class
function add(a, b) { return a + b; }
function multiply(a, b) { return a * b; }

function apply(a, b, op) {
    return op(a, b);
}

console.log(apply(3, 4, add));      // 7
console.log(apply(3, 4, multiply)); // 12
```

The behavior is identical. The difference is that Go **enforces the function signature at compile time**. If you try to pass a function with the wrong signature, the compiler rejects it:

```go
func greet(name string) string {
    return "Hello, " + name
}

// COMPILE ERROR: cannot use greet (type func(string) string)
// as type MathOperation (func(int, int) int)
fmt.Println(apply(3, 4, greet))
```

In JavaScript, you would only discover this mismatch at runtime (or with TypeScript).

### Returning Functions from Functions

```go
func makeMultiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}

func main() {
    double := makeMultiplier(2)
    triple := makeMultiplier(3)

    fmt.Println(double(5))  // 10
    fmt.Println(triple(5))  // 15
}
```

**Node.js equivalent:**

```js
function makeMultiplier(factor) {
    return function(n) {
        return n * factor;
    };
}

const double = makeMultiplier(2);
const triple = makeMultiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
```

This pattern -- a function that returns a function -- is called a **higher-order function**. It works identically in both languages.

---

## 5. Anonymous Functions and Closures

### Anonymous Functions

An anonymous function is a function without a name. In Go, you define one inline and either call it immediately or assign it to a variable.

```go
func main() {
    // Assign to a variable
    greet := func(name string) string {
        return "Hello, " + name
    }
    fmt.Println(greet("Kaung")) // Hello, Kaung

    // Immediately Invoked Function Expression (IIFE)
    result := func(a, b int) int {
        return a + b
    }(3, 4)
    fmt.Println(result) // 7
}
```

**Node.js equivalent:**

```js
// Arrow function (most common)
const greet = (name) => `Hello, ${name}`;
console.log(greet("Kaung")); // Hello, Kaung

// IIFE
const result = ((a, b) => a + b)(3, 4);
console.log(result); // 7
```

Go does not have arrow function syntax. Every anonymous function uses the full `func` keyword. This is more verbose but consistent -- there is only one way to write a function.

### Closures

A closure is a function that **captures variables from its enclosing scope**. Both Go and JavaScript support closures, but there are subtle differences.

```go
func counter() func() int {
    count := 0
    return func() int {
        count++ // captures 'count' from the enclosing scope
        return count
    }
}

func main() {
    next := counter()
    fmt.Println(next()) // 1
    fmt.Println(next()) // 2
    fmt.Println(next()) // 3

    // Each call to counter() creates a NEW closure with its own 'count'
    anotherCounter := counter()
    fmt.Println(anotherCounter()) // 1 (independent)
}
```

**Node.js equivalent:**

```js
function counter() {
    let count = 0;
    return function() {
        count++;
        return count;
    };
}

const next = counter();
console.log(next()); // 1
console.log(next()); // 2
console.log(next()); // 3

const anotherCounter = counter();
console.log(anotherCounter()); // 1
```

This works identically. Both languages close over the **variable itself** (not a copy of its value), meaning mutations to the captured variable are visible across all closures sharing it.

### The Classic Closure Gotcha: Loop Variable Capture

This is one of the most important things to understand about closures. Both Go and JavaScript historically had the same trap, but both have evolved.

**The problem (Go before 1.22):**

```go
func main() {
    funcs := make([]func(), 5)
    for i := 0; i < 5; i++ {
        funcs[i] = func() {
            fmt.Println(i) // captures the variable i, not its value
        }
    }
    for _, f := range funcs {
        f()
    }
    // Before Go 1.22: prints 5, 5, 5, 5, 5
    // Go 1.22+: prints 0, 1, 2, 3, 4
}
```

Before Go 1.22, the loop variable `i` was a single variable reused across iterations. Every closure captured the **same** variable, which had value `5` after the loop finished.

**Go 1.22 fixed this.** Starting with Go 1.22, each loop iteration gets its own copy of the loop variable. This is a breaking change (in a good way) that eliminates an entire class of bugs.

**The old workaround (still good to know):**

```go
for i := 0; i < 5; i++ {
    i := i // shadow i with a new variable scoped to this iteration
    funcs[i] = func() {
        fmt.Println(i)
    }
}
```

**JavaScript had the exact same problem:**

```js
// With var (the old way) -- BROKEN
var funcs = [];
for (var i = 0; i < 5; i++) {
    funcs.push(function() {
        console.log(i);
    });
}
funcs.forEach(f => f()); // 5, 5, 5, 5, 5

// With let (modern JS) -- FIXED
let funcs2 = [];
for (let i = 0; i < 5; i++) {
    funcs2.push(function() {
        console.log(i);
    });
}
funcs2.forEach(f => f()); // 0, 1, 2, 3, 4
```

JavaScript's `let` creates a new binding per iteration, just like Go 1.22's fix. The `var` keyword behaves like Go pre-1.22 -- a single variable shared across iterations.

### Closure Variable Capture: Deeper Differences

| Aspect | Go | JavaScript |
|--------|-----|-----------|
| Capture mechanism | Captures the variable (by reference) | Captures the variable (by reference) |
| Loop variables (modern) | New copy per iteration (Go 1.22+) | New binding per iteration (`let`) |
| Mutability | Captured variables are fully mutable | Captured variables are fully mutable |
| Goroutine gotcha | Closures in goroutines need care | Closures in async callbacks need care |
| Garbage collection | Captured variables kept alive | Captured variables kept alive |

**Goroutine closure gotcha:**

```go
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // Before Go 1.22: race condition + wrong values
    }()
}
// With Go 1.22+, each goroutine gets its own i
// But you still need synchronization to ensure all goroutines finish
```

---

## 6. Methods (Receiver Functions)

This is where Go diverges dramatically from JavaScript and most other object-oriented languages.

### What is a Method?

A method in Go is a function with a **receiver**. The receiver binds the function to a type, so you can call it using dot notation.

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Area is a method on Rectangle
// (r Rectangle) is the receiver
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Println(rect.Area())      // 50
    fmt.Println(rect.Perimeter()) // 30
}
```

### Comparison with JavaScript Classes

```js
class Rectangle {
    constructor(width, height) {
        this.width = width;
        this.height = height;
    }

    area() {
        return this.width * this.height;
    }

    perimeter() {
        return 2 * (this.width + this.height);
    }
}

const rect = new Rectangle(10, 5);
console.log(rect.area());      // 50
console.log(rect.perimeter()); // 30
```

### WHY Go Uses Receivers Instead of Classes

This is a fundamental design philosophy. Go deliberately does not have classes, inheritance, or the `this`/`self` keyword. Here is why:

1. **Decoupling data from behavior.** In Go, structs hold data. Methods are functions that happen to be associated with a type. You can define methods on a type in **any file in the same package** -- they do not need to live inside a class body. This is more flexible than classes where all methods must be defined inside the class.

2. **No `this` confusion.** JavaScript's `this` is notoriously confusing -- its value depends on how a function is called, not where it is defined. Go's receiver is explicit: you name it, you see it, there is no ambiguity.

3. **Composition over inheritance.** Go's designers believed class inheritance creates tight coupling and fragile hierarchies. Instead, Go uses **struct embedding** (composition) and **interfaces** (which we will cover in a later chapter).

4. **Simplicity.** There is no `new` keyword, no constructor functions, no `super`, no prototype chain. A method is just a function with a receiver.

### Value Receivers vs Pointer Receivers

This is a critical concept. The receiver can be either a **value** or a **pointer**.

**Value receiver -- gets a COPY of the struct:**

```go
type Point struct {
    X, Y float64
}

func (p Point) Translate(dx, dy float64) Point {
    // p is a COPY -- modifying it does NOT affect the original
    return Point{X: p.X + dx, Y: p.Y + dy}
}

func main() {
    p := Point{X: 1, Y: 2}
    p2 := p.Translate(10, 20)
    fmt.Println(p)  // {1 2}   -- unchanged
    fmt.Println(p2) // {11 22} -- new point
}
```

**Pointer receiver -- gets a POINTER to the struct (can modify it):**

```go
func (p *Point) Move(dx, dy float64) {
    // p is a pointer -- modifying it DOES affect the original
    p.X += dx
    p.Y += dy
}

func main() {
    p := Point{X: 1, Y: 2}
    p.Move(10, 20)       // Go automatically takes the address (&p)
    fmt.Println(p)        // {11 22} -- modified in place
}
```

### When to Use Which Receiver

| Use Value Receiver | Use Pointer Receiver |
|---|---|
| The method does not modify the receiver | The method needs to modify the receiver |
| The struct is small (a few fields) | The struct is large (avoids copying) |
| You want immutable semantics | You need mutation |
| Types like `time.Time` use this pattern | Types like `bytes.Buffer` use this pattern |

**Rule of thumb:** If any method on a type needs a pointer receiver, make ALL methods on that type use pointer receivers for consistency. Mixing receivers on the same type is confusing and can cause subtle bugs with interfaces.

### Methods on Non-Struct Types

You can define methods on **any named type**, not just structs:

```go
type Celsius float64
type Fahrenheit float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

func main() {
    bodyTemp := Celsius(37)
    fmt.Printf("%.1f C = %.1f F\n", bodyTemp, bodyTemp.ToFahrenheit())
    // 37.0 C = 98.6 F
}
```

This has no JavaScript equivalent. In JS, you cannot add methods to primitive types (you can extend prototypes, but it is considered bad practice). In Go, you create a named type and attach methods to it cleanly.

### JavaScript `this` vs Go Receiver -- A Deep Comparison

```js
// JavaScript -- 'this' is determined by HOW the function is called
class Timer {
    constructor() {
        this.seconds = 0;
    }

    start() {
        // BUG: 'this' is lost when passed as callback
        setInterval(this.tick, 1000);
    }

    // Fix requires: arrow function or .bind(this)
    startFixed() {
        setInterval(() => this.tick(), 1000);
    }

    tick() {
        this.seconds++;
        console.log(this.seconds);
    }
}
```

```go
// Go -- receiver is explicit, no ambiguity
type Timer struct {
    Seconds int
}

func (t *Timer) Tick() {
    t.Seconds++
    fmt.Println(t.Seconds)
}

// When you pass a method as a value, you can bind the receiver:
func main() {
    t := &Timer{}
    tick := t.Tick // method value -- receiver is bound to t
    tick()         // 1
    tick()         // 2
}
```

In Go, when you write `t.Tick`, you get a **method value** with the receiver already bound. There is no `this` ambiguity, no `.bind()` needed.

---

## 7. The init() Function

Every Go package can have one or more `init()` functions. They run **automatically** before `main()` and cannot be called explicitly.

```go
package main

import "fmt"

var message string

func init() {
    // Runs before main()
    message = "initialized"
    fmt.Println("init() called")
}

func main() {
    fmt.Println("main() called")
    fmt.Println("message:", message)
}

// Output:
// init() called
// main() called
// message: initialized
```

### Execution Order

The order is deterministic and important:

1. **Imported packages** are initialized first (recursively -- their `init()` runs before yours).
2. **Package-level variables** are initialized in declaration order.
3. **`init()` functions** run in the order they appear in the file (and files are processed in alphabetical order within a package).
4. **`main()`** runs last.

```
Package A imports Package B imports Package C

Execution order:
  1. Package C: package-level vars -> init()
  2. Package B: package-level vars -> init()
  3. Package A: package-level vars -> init()
  4. main()
```

### Multiple init() Functions

A single file can have multiple `init()` functions (unusual but legal), and a package can have `init()` in multiple files:

```go
package main

import "fmt"

func init() {
    fmt.Println("first init")
}

func init() {
    fmt.Println("second init")
}

func main() {
    fmt.Println("main")
}

// Output:
// first init
// second init
// main
```

### Use Cases

1. **Initializing package-level state** that requires computation:

```go
var config map[string]string

func init() {
    config = map[string]string{
        "env":  os.Getenv("APP_ENV"),
        "port": os.Getenv("PORT"),
    }
    if config["port"] == "" {
        config["port"] = "8080"
    }
}
```

2. **Registering drivers/plugins** (very common pattern in Go):

```go
import (
    "database/sql"
    _ "github.com/lib/pq" // blank import -- only runs init()
)
// pq's init() registers itself with database/sql
```

3. **Validating configuration or preconditions:**

```go
func init() {
    if os.Getenv("API_KEY") == "" {
        log.Fatal("API_KEY environment variable is required")
    }
}
```

### No Direct JavaScript Equivalent

The closest thing in JavaScript is **top-level module code** that runs when the module is first imported:

```js
// config.js (ESM)
console.log("module loaded"); // runs on first import

export const config = {
    env: process.env.APP_ENV,
    port: process.env.PORT || "8080",
};
```

But there are differences:

| Feature | Go `init()` | JS Module Top-Level |
|---------|------------|-------------------|
| Multiple per file | Yes | N/A (code runs once) |
| Explicit function | Yes (`func init()`) | No (just code at top level) |
| Guaranteed order | Yes (import graph order) | Yes (import graph order) |
| Can be called explicitly | No | N/A |
| Side-effect imports | Yes (`_ "pkg"`) | Yes (`import "pkg"`) |

### Caution with init()

While `init()` is powerful, overusing it can make your code hard to test and reason about:

- **Hidden dependencies:** `init()` runs implicitly. If it does something important, readers might not realize it.
- **Testing difficulty:** You cannot skip `init()` in tests. If it connects to a database, your tests will too.
- **Ordering fragility:** Relying on init order across files can lead to bugs when files are renamed or added.

**Best practice:** Use `init()` sparingly. Prefer explicit initialization functions that you call from `main()`.

---

## 8. The Blank Identifier (_)

The blank identifier `_` is Go's way of saying "I know there is a value here, but I do not need it." It is used to **explicitly ignore** values.

### Ignoring Return Values

Go's compiler enforces that every declared variable must be used. If a function returns multiple values and you only need one, you must use `_`:

```go
// os.Open returns (*File, error)
// If you only want the file and are sure there is no error (not recommended):
file, _ := os.Open("data.txt")

// If you only want to check the error:
_, err := os.Open("nonexistent.txt")
if err != nil {
    fmt.Println("Error:", err)
}
```

### Ignoring Loop Variables

```go
names := []string{"Alice", "Bob", "Charlie"}

// If you do not need the index:
for _, name := range names {
    fmt.Println(name)
}

// If you do not need the value (just the index):
for i := range names {
    fmt.Println(i)
}
// Note: for just the index, you do not even need _
```

### Import Side Effects (Blank Import)

This is one of the most important uses. Importing a package with `_` runs its `init()` function without making its exported names available:

```go
import (
    "database/sql"
    _ "github.com/lib/pq"           // PostgreSQL driver -- just runs init()
    _ "net/http/pprof"              // Registers profiling HTTP handlers
    _ "image/png"                   // Registers PNG decoder with image package
)
```

Without the `_`, the Go compiler would reject the import as unused. The blank identifier says: "I know I am not using any exported names from this package, but I need its side effects."

### Type Assertion Check

```go
// Check if a value implements an interface at compile time
var _ io.Reader = (*MyType)(nil)
// If MyType does not implement io.Reader, this line causes a compile error
// The _ discards the value -- it is purely a compile-time assertion
```

### Comparison with JavaScript

JavaScript has no equivalent enforcement. Unused variables are warnings (via ESLint), not compile errors:

```js
// JavaScript -- you can just ignore return values
const [result] = someFunctionReturningArray();
// or use destructuring to skip:
const [, second, , fourth] = [1, 2, 3, 4];
```

The philosophical difference: Go forces you to be **explicit** about what you ignore. You cannot accidentally forget a return value. In JavaScript, you can silently ignore anything.

```go
// Go WILL NOT compile if you assign to a variable and never use it
result, err := doSomething()
// If you never use 'result' or 'err', the compiler rejects this

// You must either use both:
result, err := doSomething()
if err != nil { ... }
fmt.Println(result)

// Or explicitly ignore what you do not need:
_, err := doSomething()
result, _ := doSomething()
```

---

## 9. Error Handling with Functions

This is the most philosophically different aspect of Go compared to JavaScript (and most other languages). Understanding it deeply is essential.

### The (result, error) Pattern

In Go, functions that can fail return an `error` as their last return value:

```go
func readConfig(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return Config{}, fmt.Errorf("reading config: %w", err)
    }

    var config Config
    err = json.Unmarshal(data, &config)
    if err != nil {
        return Config{}, fmt.Errorf("parsing config: %w", err)
    }

    return config, nil
}
```

### The error Interface

`error` in Go is just an interface:

```go
type error interface {
    Error() string
}
```

Any type that has an `Error() string` method satisfies this interface. You can create custom errors:

```go
// Simple custom error
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{
            Field:   "age",
            Message: "must be non-negative",
        }
    }
    if age > 150 {
        return &ValidationError{
            Field:   "age",
            Message: "unrealistic value",
        }
    }
    return nil
}
```

### WHY Go Chose Explicit Error Returns Over Exceptions

This is one of the most debated decisions in Go's design. Here is the reasoning:

**1. Errors are values, not control flow.**

In exception-based languages, an error fundamentally changes how your program executes -- it unwinds the call stack, skipping code, until someone catches it. In Go, an error is just a value you check. Control flow remains linear and visible.

```go
// Go -- you can see exactly what happens at each step
file, err := os.Open(path)
if err != nil {
    return err
}
data, err := io.ReadAll(file)
if err != nil {
    return err
}
// process data...
```

**2. You cannot accidentally ignore an error (in practice).**

While you *can* assign an error to `_`, the convention is so strong that code reviewers and linters (`errcheck`, `golangci-lint`) will flag it. In JavaScript, any function can throw, and if you forget `try/catch`, the error propagates silently.

**3. Errors are expected, not exceptional.**

Go treats errors as a normal part of program flow. A file not existing is not "exceptional" -- it is expected. Go's design says: if something can go wrong, the function signature tells you, and you handle it right there.

**4. No hidden control flow.**

In JavaScript:

```js
function processData() {
    const data = readFile();      // might throw
    const parsed = parseJSON(data); // might throw
    const result = transform(parsed); // might throw
    return result;
}
```

Any of these lines might throw an exception. You cannot tell by reading the code. In Go, every fallible operation is visible:

```go
func processData() (Result, error) {
    data, err := readFile()
    if err != nil {
        return Result{}, err
    }
    parsed, err := parseJSON(data)
    if err != nil {
        return Result{}, err
    }
    result, err := transform(parsed)
    if err != nil {
        return Result{}, err
    }
    return result, nil
}
```

Yes, it is more verbose. That is the trade-off. Go values **clarity over brevity**.

### Deep Comparison: Go Error Handling vs JavaScript try/catch

**Go pattern -- sequential error checking:**

```go
func fetchUserData(userID string) (*UserData, error) {
    // Step 1: Validate input
    if userID == "" {
        return nil, fmt.Errorf("userID cannot be empty")
    }

    // Step 2: Fetch from database
    user, err := db.FindUser(userID)
    if err != nil {
        return nil, fmt.Errorf("finding user %s: %w", userID, err)
    }

    // Step 3: Fetch user's orders
    orders, err := db.FindOrders(user.ID)
    if err != nil {
        return nil, fmt.Errorf("finding orders for user %s: %w", userID, err)
    }

    // Step 4: Build response
    return &UserData{
        User:   user,
        Orders: orders,
    }, nil
}

// Calling code
data, err := fetchUserData("123")
if err != nil {
    log.Printf("failed to fetch user data: %v", err)
    http.Error(w, "internal error", 500)
    return
}
```

**JavaScript equivalent -- try/catch with async/await:**

```js
async function fetchUserData(userID) {
    // Step 1: Validate input
    if (!userID) {
        throw new Error("userID cannot be empty");
    }

    // Step 2: Fetch from database
    const user = await db.findUser(userID); // might throw

    // Step 3: Fetch user's orders
    const orders = await db.findOrders(user.id); // might throw

    // Step 4: Build response
    return { user, orders };
}

// Calling code
try {
    const data = await fetchUserData("123");
} catch (err) {
    console.error("failed to fetch user data:", err.message);
    res.status(500).json({ error: "internal error" });
    return;
}
```

### Error Wrapping (%w)

Go 1.13 introduced error wrapping with `%w` in `fmt.Errorf`. This creates an error chain:

```go
func readConfig(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("readConfig(%s): %w", path, err)
    }
    return data, nil
}

// Later, you can unwrap and check the original error:
_, err := readConfig("/etc/app/config.json")
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("Config file does not exist -- using defaults")
    } else {
        fmt.Println("Unexpected error:", err)
    }
}
```

Key functions for working with error chains:

```go
// errors.Is -- checks if any error in the chain matches a target
errors.Is(err, os.ErrNotExist) // like === for errors

// errors.As -- checks if any error in the chain is a specific type
var validErr *ValidationError
if errors.As(err, &validErr) {
    fmt.Printf("Validation failed: field=%s msg=%s\n",
        validErr.Field, validErr.Message)
}
```

**JavaScript equivalent:**

```js
// JavaScript has no built-in error wrapping.
// Common pattern: create a new error with the original as a cause (ES2022+)
try {
    // ...
} catch (originalError) {
    throw new Error(`readConfig(${path})`, { cause: originalError });
}
```

### Sentinel Errors

Go uses predefined error values (called sentinel errors) for well-known error conditions:

```go
import (
    "errors"
    "io"
)

// io.EOF is a sentinel error
data := make([]byte, 100)
n, err := reader.Read(data)
if err != nil {
    if errors.Is(err, io.EOF) {
        fmt.Println("Reached end of file")
    } else {
        fmt.Println("Read error:", err)
    }
}
```

You can define your own sentinel errors:

```go
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
)

func getResource(id string) (*Resource, error) {
    // ...
    if !found {
        return nil, ErrNotFound
    }
    if !authorized {
        return nil, ErrUnauthorized
    }
    return resource, nil
}
```

### panic and recover -- Go's "Exceptions" (Use Sparingly)

Go does have a mechanism for truly exceptional situations: `panic` and `recover`. But it is explicitly not for normal error handling.

```go
// panic stops normal execution
func mustParseURL(rawURL string) *url.URL {
    u, err := url.Parse(rawURL)
    if err != nil {
        panic(fmt.Sprintf("invalid URL %q: %v", rawURL, err))
    }
    return u
}

// recover catches a panic (only works inside a deferred function)
func safeOperation() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from panic:", r)
        }
    }()

    // This would normally crash the program
    panic("something went terribly wrong")
}
```

**When to use panic:**

- Programmer errors (not runtime conditions): nil pointer that should never be nil, index out of bounds on data you control, etc.
- Initialization failures where the program cannot possibly continue.
- Functions prefixed with `Must` (like `template.Must()`, `regexp.MustCompile()`).

**When NOT to use panic:**

- File not found, network timeout, invalid user input -- these are expected errors. Return `error`.

**JavaScript comparison:**

| Go | JavaScript |
|----|-----------|
| `return err` | `throw new Error()` |
| `if err != nil { ... }` | `try { ... } catch (e) { ... }` |
| `panic()` | `throw` (for truly fatal situations) |
| `recover()` | `catch` (sort of) |
| Custom error types | Custom Error subclasses |
| `errors.Is()` | `err instanceof ErrorType` |
| `errors.As()` | `err instanceof ErrorType` + cast |
| `fmt.Errorf("...: %w", err)` | `new Error("...", { cause: err })` |

---

## 10. Go vs Node.js Comparison Summary

### Multiple Returns vs Object Destructuring

```go
// Go -- true multiple returns
func getUser() (User, error) {
    return User{Name: "Alice"}, nil
}
user, err := getUser()
```

```js
// JavaScript -- single return with destructuring
function getUser() {
    return { user: { name: "Alice" }, error: null };
}
const { user, error } = getUser();
```

Go's multiple returns are a language-level feature. JavaScript simulates it with objects or arrays.

### Error Handling Philosophy

```go
// Go -- explicit at every step
result, err := step1()
if err != nil {
    return err
}
result2, err := step2(result)
if err != nil {
    return err
}
```

```js
// JavaScript -- exception-based
try {
    const result = await step1();
    const result2 = await step2(result);
} catch (err) {
    // handle any error from any step
}
```

Go: verbose but clear. JavaScript: concise but hides which steps can fail.

### Methods: Receivers vs `this`

```go
// Go -- explicit receiver, no ambiguity
type Server struct {
    Port int
}

func (s *Server) Start() {
    fmt.Printf("Starting on port %d\n", s.Port)
}
```

```js
// JavaScript -- 'this' depends on call context
class Server {
    constructor(port) {
        this.port = port;
    }

    start() {
        console.log(`Starting on port ${this.port}`);
    }
}

// Danger zone:
const server = new Server(8080);
const startFn = server.start;
startFn(); // 'this' is undefined in strict mode -- bug!

// Go equivalent works fine:
// startFn := server.Start
// startFn() -- receiver is already bound
```

### Closures

Both languages have closures that work nearly identically. The main differences:

1. **Loop variable capture:** Both have fixed this in modern versions (Go 1.22, JS `let`).
2. **Syntax:** Go uses `func(...) {...}`, JS has arrow functions `(...) => {...}`.
3. **Type safety:** Go closures must match their declared function type.

### Function References

```go
// Go -- function types are explicit
type Handler func(http.ResponseWriter, *http.Request)

var myHandler Handler = func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello"))
}
```

```js
// JavaScript -- any function reference, no type constraint
const myHandler = (req, res) => {
    res.send("Hello");
};
```

---

## 11. Key Takeaways

1. **Functions are the building blocks of Go programs.** Unlike JavaScript where you have functions, arrow functions, generators, and async functions, Go has one kind of function. Simple.

2. **Multiple return values exist primarily for error handling.** The `(result, error)` pattern is Go's most important idiom. Learn it, love it, use it everywhere.

3. **Named return values are documentation tools.** Use them to clarify what a function returns. Avoid naked returns in long functions.

4. **Methods use receivers, not classes.** The receiver is just a regular parameter with special syntax. Value receivers get copies; pointer receivers can mutate.

5. **Use pointer receivers when in doubt.** If any method on your type needs a pointer receiver, make all of them use pointer receivers.

6. **Go closures work like JavaScript closures.** Both capture variables by reference. Go 1.22 fixed the loop variable capture problem, just like JS `let` fixed it for JavaScript.

7. **Error handling is explicit and verbose -- by design.** Every error is visible. No hidden throws, no invisible control flow. The trade-off is more lines of code for more clarity.

8. **`init()` runs before `main()`.** Use it sparingly. Prefer explicit initialization.

9. **The blank identifier `_` is your way of saying "I know about this value but I do not need it."** It is not laziness -- it is an explicit statement of intent.

10. **`panic` is not `throw`.** Use `panic` only for programmer errors and truly unrecoverable situations, never for expected runtime errors.

---

## 12. Practice Exercises

### Exercise 1: Multiple Returns and Error Handling

Write a function `safeDivide(a, b float64) (float64, error)` that:
- Returns the division result and `nil` if `b` is not zero.
- Returns `0` and an appropriate error if `b` is zero.
- Write a `main()` function that calls it with several inputs and handles errors properly.

### Exercise 2: Variadic Function

Write a function `joinStrings(sep string, parts ...string) string` that joins all the parts with the given separator. Do not use `strings.Join` -- implement it manually.

```go
// Expected:
joinStrings(", ", "a", "b", "c") // "a, b, c"
joinStrings("-", "2024", "01", "15") // "2024-01-15"
joinStrings("", "Go", "Lang") // "GoLang"
```

### Exercise 3: Higher-Order Functions

Write a function `filter(numbers []int, predicate func(int) bool) []int` that returns a new slice containing only the numbers for which the predicate returns `true`.

```go
// Expected:
evens := filter([]int{1, 2, 3, 4, 5, 6}, func(n int) bool {
    return n%2 == 0
})
// evens = [2, 4, 6]
```

### Exercise 4: Closures

Write a function `makeAccumulator(initial int) func(int) int` that returns a closure. Each time the closure is called with a value, it adds that value to a running total and returns the new total.

```go
// Expected:
acc := makeAccumulator(100)
fmt.Println(acc(10))  // 110
fmt.Println(acc(20))  // 130
fmt.Println(acc(5))   // 135
```

### Exercise 5: Methods and Receivers

Create a `BankAccount` struct with `Owner string` and `Balance float64`. Implement these methods:

- `Deposit(amount float64) error` -- adds money (returns error if amount is negative).
- `Withdraw(amount float64) error` -- removes money (returns error if insufficient funds or negative amount).
- `String() string` -- returns a formatted string like `"Alice: $1234.56"`.

Decide whether each method should use a value or pointer receiver, and explain your choice.

### Exercise 6: Error Wrapping

Write a function `loadUserProfile(userID string) (Profile, error)` that:
1. Calls `findUser(userID)` (simulated -- returns an error if ID is empty).
2. Calls `loadPreferences(user)` (simulated -- returns an error if user is inactive).
3. Wraps each error with context using `fmt.Errorf("...: %w", err)`.
4. In `main()`, use `errors.Is()` and `errors.As()` to inspect the error chain.

### Exercise 7: Function Pipeline

Build a simple text processing pipeline using first-class functions:

```go
type Transform func(string) string

func pipeline(input string, transforms ...Transform) string {
    // Apply each transform in order and return the final result
}

// Usage:
result := pipeline("  Hello, World!  ",
    strings.TrimSpace,
    strings.ToLower,
    func(s string) string {
        return strings.ReplaceAll(s, " ", "-")
    },
)
// result: "hello,-world!"
```

---

**Next Chapter:** [Chapter 5 - Structs, Interfaces, and Composition] -- where we dive into Go's type system, how interfaces work implicitly, and why Go chose composition over inheritance.
