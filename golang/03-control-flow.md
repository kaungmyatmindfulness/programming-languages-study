# Chapter 3: Control Flow

Control flow is the backbone of any program -- it determines which code runs, when, how many times, and in what order. Go takes a distinctly minimalist approach to control flow compared to most languages. Where JavaScript gives you six or seven ways to loop, Go gives you one. Where JavaScript's `switch` falls through by default (a notorious source of bugs), Go's `switch` breaks by default. Every design choice in Go's control flow was made deliberately to reduce bugs, improve readability, and keep the language simple.

This chapter covers every control flow mechanism Go provides: conditionals, switches, loops, range iteration, labeled control flow, and the unique `defer` statement.

---

## Table of Contents

1. [if/else Statements](#1-ifelse-statements)
2. [switch Statements](#2-switch-statements)
3. [for Loops](#3-for-loops)
4. [range](#4-range)
5. [Labels, break, continue, goto](#5-labels-break-continue-goto)
6. [defer Statement](#6-defer-statement)
7. [Go vs Node.js Control Flow Comparison](#7-go-vs-nodejs-control-flow-comparison)
8. [Common Pitfalls and Gotchas](#8-common-pitfalls-and-gotchas)
9. [Key Takeaways](#9-key-takeaways)
10. [Practice Exercises](#10-practice-exercises)

---

## 1. if/else Statements

### Basic Syntax

Go's `if` statement looks similar to most C-family languages, with one key difference: **no parentheses around the condition**.

```go
package main

import "fmt"

func main() {
    age := 25

    // No parentheses around the condition -- this is mandatory in Go
    if age >= 18 {
        fmt.Println("You are an adult")
    } else if age >= 13 {
        fmt.Println("You are a teenager")
    } else {
        fmt.Println("You are a child")
    }
}
```

Compare with the same logic in Node.js:

```javascript
// Node.js -- parentheses are required
const age = 25;

if (age >= 18) {
    console.log("You are an adult");
} else if (age >= 13) {
    console.log("You are a teenager");
} else {
    console.log("You are a child");
}
```

### WHY Go Removed Parentheses

This was not an arbitrary aesthetic choice. The Go designers observed that in C-family languages, the parentheses around `if` conditions serve no grammatical purpose -- they exist only because the language grammar requires them. Since Go requires braces `{}` around the body of every `if` statement (even single-line bodies), the parser can unambiguously determine where the condition ends and the body begins without parentheses. Removing them eliminates visual noise and makes the code cleaner.

If you try to add parentheses, the code will still compile, but `gofmt` (Go's official formatter) will strip them out:

```go
// This compiles but gofmt will remove the parentheses
if (age >= 18) {
    fmt.Println("adult")
}

// gofmt rewrites it to:
if age >= 18 {
    fmt.Println("adult")
}
```

### Braces Are Always Required

Unlike JavaScript, Go does **not** allow brace-less if statements:

```go
// COMPILE ERROR -- Go requires braces
if age >= 18
    fmt.Println("adult")

// COMPILE ERROR -- even single-line bodies need braces
if age >= 18 fmt.Println("adult")
```

**WHY?** This eliminates an entire class of bugs. The infamous Apple "goto fail" SSL bug in 2014 was caused by a brace-less `if` statement in C. Go sidesteps this entirely by making braces mandatory.

### The Init Statement in if (Unique to Go)

This is one of Go's most elegant features. You can declare and initialize a variable **inside** the `if` statement, scoped only to the `if/else` block:

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // The variable 'err' is declared in the if-init statement.
    // It only exists within this if/else block.
    if err := doSomething(); err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
    // 'err' does not exist here -- it is out of scope
}

func doSomething() error {
    return nil
}
```

The syntax is: `if <init statement>; <condition> { ... }`. The semicolon separates the init statement from the condition.

Here is a more concrete example:

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    input := "42"

    // Parse the string to int. Both 'n' and 'err' are scoped to this if/else.
    if n, err := strconv.Atoi(input); err != nil {
        fmt.Println("Failed to parse:", err)
    } else {
        fmt.Println("Parsed number:", n) // 'n' is accessible in the else block too
    }

    // fmt.Println(n) -- COMPILE ERROR: 'n' is not defined here
}
```

**WHY does this exist?** It solves a real problem: in error handling (which is extremely common in Go), you often need a variable only for the duration of checking an error. Without the init statement, those variables would leak into the outer scope, polluting the namespace:

```go
// Without init statement -- 'err' leaks into the outer scope
err := doSomething()
if err != nil {
    fmt.Println("Error:", err)
}
// 'err' is still in scope here, even though we no longer need it.
// If we call another function, we cannot use := for 'err' again easily.

// With init statement -- clean and contained
if err := doSomething(); err != nil {
    fmt.Println("Error:", err)
}
// 'err' is gone. The outer scope stays clean.
```

**Node.js has no equivalent.** The closest you can get is:

```javascript
// Node.js -- no init statement in if
// You must declare the variable before the if
const result = doSomething();
if (result.error) {
    console.log("Error:", result.error);
}
// 'result' is still in scope even if you no longer need it
```

### Truthiness: Go Requires Explicit Booleans

In JavaScript, values are "truthy" or "falsy." The number `0`, empty string `""`, `null`, `undefined`, and `NaN` are all falsy. Everything else is truthy. This leads to code like:

```javascript
// Node.js -- truthy/falsy
const name = "";
if (name) {
    console.log("Has name");  // Does not run because "" is falsy
}

const count = 0;
if (count) {
    console.log("Has count"); // Does not run because 0 is falsy
}
```

Go has **none of this**. The condition in an `if` statement **must** be a `bool`. Period. No implicit conversions, no truthiness:

```go
name := ""
// if name {          // COMPILE ERROR: name (type string) cannot be used as bool
if name != "" {       // You must be explicit
    fmt.Println("Has name")
}

count := 0
// if count {         // COMPILE ERROR: count (type int) cannot be used as bool
if count != 0 {       // You must be explicit
    fmt.Println("Has count")
}
```

**WHY?** Implicit truthiness is a major source of subtle bugs. Consider the JavaScript expression `if (x)` -- does the developer mean "if x is not null," "if x is not zero," "if x is not an empty string," or "if x is not undefined"? The intent is ambiguous. Go forces you to state your intent explicitly: `if x != nil`, `if x != 0`, `if x != ""`. The code documents exactly what you mean.

---

## 2. switch Statements

### Expression Switch (Basic)

Go's `switch` is more powerful and safer than JavaScript's. Here is the basic form:

```go
package main

import "fmt"

func main() {
    day := "Tuesday"

    switch day {
    case "Monday":
        fmt.Println("Start of work week")
    case "Tuesday":
        fmt.Println("Second day")
    case "Wednesday":
        fmt.Println("Midweek")
    case "Thursday", "Friday":           // Multiple values in one case
        fmt.Println("Almost weekend")
    default:
        fmt.Println("Weekend!")
    }
}
// Output: Second day
```

Compare with Node.js:

```javascript
// Node.js
const day = "Tuesday";

switch (day) {
    case "Monday":
        console.log("Start of work week");
        break;                            // Must manually break!
    case "Tuesday":
        console.log("Second day");
        break;
    case "Wednesday":
        console.log("Midweek");
        break;
    case "Thursday":                      // Fall-through to Friday
    case "Friday":
        console.log("Almost weekend");
        break;
    default:
        console.log("Weekend!");
}
```

### WHY Go Breaks by Default (and JavaScript Falls Through)

JavaScript inherited `switch` from C, which falls through by default. This means if you forget a `break`, execution continues into the next case. This design decision from the 1970s has caused countless bugs. Studies show that the vast majority of `switch` cases are intended to break -- fall-through is the rare exception, not the rule.

Go flipped the default: **each case breaks automatically**. If you actually want fall-through (which is rare), you must explicitly say so with the `fallthrough` keyword. This aligns with the principle of making the common case easy and the rare case possible.

```go
// Go -- each case breaks automatically. No break keyword needed.
switch n {
case 1:
    fmt.Println("one")
    // Execution stops here. Does NOT fall through to case 2.
case 2:
    fmt.Println("two")
}
```

```javascript
// JavaScript -- falls through by default. Forgetting break is a bug.
switch (n) {
    case 1:
        console.log("one");
        // BUG: if you forget break, "two" prints too!
    case 2:
        console.log("two");
        break;
}
```

### The fallthrough Keyword

When you genuinely want fall-through behavior, Go makes you be explicit:

```go
package main

import "fmt"

func main() {
    n := 3

    switch {
    case n > 0:
        fmt.Println("positive")
        fallthrough                // Explicitly fall through to next case
    case n > -5:
        fmt.Println("greater than -5")
        fallthrough
    case n > -10:
        fmt.Println("greater than -10")
    }
}
// Output:
// positive
// greater than -5
// greater than -10
```

**Important caveat:** `fallthrough` transfers control to the **next case's body unconditionally**. It does not re-evaluate the next case's condition. In the example above, even though `n > -5` and `n > -10` are both true, `fallthrough` would execute them even if they were false. This is a common source of confusion, so use `fallthrough` sparingly.

### Multiple Values per Case

Instead of relying on fall-through for grouping (like JavaScript), Go lets you list multiple values in a single case:

```go
switch day {
case "Saturday", "Sunday":
    fmt.Println("Weekend")
case "Monday", "Tuesday", "Wednesday", "Thursday", "Friday":
    fmt.Println("Weekday")
}
```

This is cleaner and less error-prone than the JavaScript pattern of stacking empty cases:

```javascript
// JavaScript -- stacking cases for grouping
switch (day) {
    case "Saturday":
    case "Sunday":
        console.log("Weekend");
        break;
    case "Monday":
    case "Tuesday":
    case "Wednesday":
    case "Thursday":
    case "Friday":
        console.log("Weekday");
        break;
}
```

### Tagless Switch (switch without an expression)

Go allows a `switch` with no expression. Each case becomes a standalone boolean condition. This is essentially a cleaner way to write `if/else if/else` chains:

```go
package main

import "fmt"

func main() {
    score := 85

    // No expression after 'switch' -- each case is an independent condition
    switch {
    case score >= 90:
        fmt.Println("Grade: A")
    case score >= 80:
        fmt.Println("Grade: B")     // This runs
    case score >= 70:
        fmt.Println("Grade: C")
    case score >= 60:
        fmt.Println("Grade: D")
    default:
        fmt.Println("Grade: F")
    }
}
// Output: Grade: B
```

This is equivalent to:

```go
if score >= 90 {
    fmt.Println("Grade: A")
} else if score >= 80 {
    fmt.Println("Grade: B")
} else if score >= 70 {
    fmt.Println("Grade: C")
} else if score >= 60 {
    fmt.Println("Grade: D")
} else {
    fmt.Println("Grade: F")
}
```

The tagless switch is often preferred when you have three or more conditions because it is visually cleaner and the cases align vertically.

### Init Statement in switch

Just like `if`, a `switch` can have an init statement:

```go
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Printf("Unknown: %s\n", os)
}
// 'os' is not accessible here
```

### Type Switch

Go has a special switch form for determining the concrete type of an interface value. This is essential when working with interfaces (covered in depth in a later chapter):

```go
package main

import "fmt"

func classify(i interface{}) {
    switch v := i.(type) {      // 'v' gets the concrete typed value
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %q\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    case []int:
        fmt.Printf("Slice of ints: %v\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

func main() {
    classify(42)            // Integer: 42
    classify("hello")       // String: "hello"
    classify(true)          // Boolean: true
    classify([]int{1, 2})   // Slice of ints: [1 2]
    classify(3.14)          // Unknown type: float64
}
```

The syntax `i.(type)` can only be used inside a `switch` statement. The variable `v` is bound to the concrete typed value in each case, so you can use it as that specific type without further casting.

**Node.js comparison:**

```javascript
// Node.js -- typeof switch
function classify(value) {
    switch (typeof value) {
        case "number":
            console.log(`Number: ${value}`);
            break;
        case "string":
            console.log(`String: ${value}`);
            break;
        case "boolean":
            console.log(`Boolean: ${value}`);
            break;
        default:
            console.log(`Unknown: ${typeof value}`);
    }
}
```

JavaScript's `typeof` is much more limited. It returns `"object"` for arrays, null, and objects alike. Go's type switch gives you precise type information.

---

## 3. for Loops

### The Only Loop in Go

Go has **one** loop keyword: `for`. That is it. No `while`, no `do...while`, no `forEach`, no `for...of`, no `for...in`. Just `for`.

**WHY?** The Go designers observed that all loop constructs are variations of the same concept: repeat something until a condition is met. Having multiple keywords for the same concept adds complexity without adding capability. Go's single `for` keyword can express every looping pattern.

Compare the loop keywords available in each language:

| Pattern | Go | JavaScript |
|---|---|---|
| Traditional index loop | `for i := 0; i < n; i++` | `for (let i = 0; i < n; i++)` |
| Condition-only loop | `for condition` | `while (condition)` |
| Infinite loop | `for { }` | `while (true) { }` |
| Post-check loop | (use `for` with `break`) | `do { } while (condition)` |
| Iterate collection | `for i, v := range coll` | `for...of`, `forEach`, `.map()` |
| Iterate object keys | `for k := range m` | `for...in` |

### Traditional for Loop (Three-Component)

The most familiar form. Identical in structure to C/JavaScript, minus the parentheses:

```go
package main

import "fmt"

func main() {
    // init; condition; post
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
    // Output: 0, 1, 2, 3, 4
}
```

Node.js equivalent:

```javascript
for (let i = 0; i < 5; i++) {
    console.log(i);
}
```

The init statement (`i := 0`) runs once. The condition (`i < 5`) is checked before every iteration. The post statement (`i++`) runs after every iteration. The variable `i` is scoped to the loop.

### While-Style for Loop (Condition Only)

Drop the init and post statements. This is Go's version of `while`:

```go
package main

import "fmt"

func main() {
    n := 1

    // This is Go's "while" loop
    for n < 100 {
        n *= 2
    }

    fmt.Println(n) // 128
}
```

Node.js equivalent:

```javascript
let n = 1;
while (n < 100) {
    n *= 2;
}
console.log(n); // 128
```

### Infinite Loop

Drop everything. The bare `for` keyword creates an infinite loop:

```go
package main

import "fmt"

func main() {
    count := 0

    for {
        count++
        if count > 3 {
            break        // Exit the loop
        }
        fmt.Println(count)
    }
    // Output: 1, 2, 3
}
```

Node.js equivalent:

```javascript
let count = 0;
while (true) {
    count++;
    if (count > 3) {
        break;
    }
    console.log(count);
}
```

Infinite loops are common in Go for server main loops, event loops, and goroutine workers. They always exit via `break`, `return`, or `os.Exit`.

### Emulating do...while

Go has no `do...while`, but you can easily replicate it:

```go
// Go -- emulating do...while
for {
    // Body runs at least once
    result := doWork()
    if result == "done" {
        break
    }
}
```

```javascript
// JavaScript -- native do...while
do {
    result = doWork();
} while (result !== "done");
```

### break and continue

`break` exits the loop immediately. `continue` skips to the next iteration:

```go
package main

import "fmt"

func main() {
    for i := 0; i < 10; i++ {
        if i == 3 {
            continue   // Skip 3, jump to i++ and next iteration
        }
        if i == 7 {
            break      // Exit the loop entirely at 7
        }
        fmt.Println(i)
    }
    // Output: 0, 1, 2, 4, 5, 6
}
```

---

## 4. range

### What range Does

The `range` keyword is used with `for` to iterate over data structures. It returns **two values** per iteration: the index (or key) and a **copy** of the value.

### Iterating Over Slices and Arrays

```go
package main

import "fmt"

func main() {
    fruits := []string{"apple", "banana", "cherry"}

    // Both index and value
    for i, fruit := range fruits {
        fmt.Printf("Index %d: %s\n", i, fruit)
    }
    // Output:
    // Index 0: apple
    // Index 1: banana
    // Index 2: cherry

    // Index only (ignore value)
    for i := range fruits {
        fmt.Printf("Index: %d\n", i)
    }

    // Value only (ignore index with blank identifier)
    for _, fruit := range fruits {
        fmt.Printf("Fruit: %s\n", fruit)
    }
}
```

Node.js comparison:

```javascript
const fruits = ["apple", "banana", "cherry"];

// for...of gives values only (no index)
for (const fruit of fruits) {
    console.log(fruit);
}

// forEach gives value, index, and array
fruits.forEach((fruit, i) => {
    console.log(`Index ${i}: ${fruit}`);
});

// for...in gives string keys (indices) -- generally avoid for arrays
for (const i in fruits) {
    console.log(`Index ${i}: ${fruits[i]}`);
}
```

### Iterating Over Maps

```go
package main

import "fmt"

func main() {
    scores := map[string]int{
        "Alice": 92,
        "Bob":   87,
        "Carol": 95,
    }

    // Key and value
    for name, score := range scores {
        fmt.Printf("%s: %d\n", name, score)
    }

    // Key only
    for name := range scores {
        fmt.Println(name)
    }
}
```

**Important:** Map iteration order is **not guaranteed** in Go. Each time you range over a map, the order may differ. This is intentional -- Go randomizes map iteration to prevent developers from relying on a specific order.

Node.js comparison:

```javascript
const scores = { Alice: 92, Bob: 87, Carol: 95 };

// Object.entries
for (const [name, score] of Object.entries(scores)) {
    console.log(`${name}: ${score}`);
}

// for...in gives keys
for (const name in scores) {
    console.log(name);
}

// Or use Map for guaranteed insertion order
const scoreMap = new Map([["Alice", 92], ["Bob", 87], ["Carol", 95]]);
for (const [name, score] of scoreMap) {
    console.log(`${name}: ${score}`);
}
```

### Iterating Over Strings

When you range over a string in Go, you get **Unicode code points (runes)**, not bytes. This is crucial for correctly handling multi-byte characters:

```go
package main

import "fmt"

func main() {
    greeting := "Hello, world!"

    // Iterating over a string gives index and rune (Unicode code point)
    for i, ch := range greeting {
        fmt.Printf("Index %d: %c (Unicode: %U)\n", i, ch, ch)
    }

    // Multi-byte characters
    emoji := "Go is fun!"
    for i, ch := range emoji {
        fmt.Printf("Byte index %d: %c\n", i, ch)
    }
    // Note: the byte index may skip numbers for multi-byte characters
}
```

**Key detail:** The index returned by `range` on a string is the **byte position**, not the character position. For ASCII strings these are the same, but for strings with multi-byte UTF-8 characters (like emojis or CJK characters), the byte index will skip values.

Node.js comparison:

```javascript
const greeting = "Hello, world!";

// for...of iterates over Unicode code points
for (const ch of greeting) {
    console.log(ch);
}

// Spread operator also works with Unicode
const chars = [...greeting];
console.log(chars);
```

### Iterating Over Channels

Channels are a Go concurrency primitive (covered in depth in a later chapter). You can range over a channel to receive values until it is closed:

```go
package main

import "fmt"

func main() {
    ch := make(chan int)

    // Send values in a goroutine
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i        // Send i to the channel
        }
        close(ch)          // Close the channel when done
    }()

    // Range over the channel -- receives until channel is closed
    for val := range ch {
        fmt.Println(val)
    }
    // Output: 0, 1, 2, 3, 4
}
```

The `for range` on a channel blocks waiting for each value and exits automatically when the channel is closed. Without `range`, you would need to manually check whether the channel is closed using the `val, ok := <-ch` pattern.

### Value Semantics: range Creates Copies

This is a critical gotcha. The value variable in a `for range` loop is a **copy**, not a reference to the original element:

```go
package main

import "fmt"

func main() {
    numbers := []int{10, 20, 30}

    // WRONG: modifying 'v' does NOT modify the slice
    for _, v := range numbers {
        v *= 2    // This modifies the copy, not the original
    }
    fmt.Println(numbers) // [10 20 30] -- unchanged!

    // RIGHT: use the index to modify the original
    for i := range numbers {
        numbers[i] *= 2
    }
    fmt.Println(numbers) // [20 40 60] -- modified!
}
```

This is fundamentally different from JavaScript, where array iteration methods give you references to objects:

```javascript
// Node.js -- objects are references
const people = [{ name: "Alice" }, { name: "Bob" }];

for (const person of people) {
    person.name = person.name.toUpperCase(); // Modifies the original
}
console.log(people); // [{ name: "ALICE" }, { name: "BOB" }]
```

---

## 5. Labels, break, continue, goto

### Labels for Nested Loop Control

When you have nested loops and want to `break` or `continue` an **outer** loop from within an inner loop, you need labels:

```go
package main

import "fmt"

func main() {
    // The label "outer" is attached to the outer for loop
outer:
    for i := 0; i < 5; i++ {
        for j := 0; j < 5; j++ {
            if i+j == 4 {
                fmt.Printf("Breaking at i=%d, j=%d\n", i, j)
                break outer    // Breaks the OUTER loop, not just the inner one
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
    fmt.Println("Done")
}
// Output:
// i=0, j=0
// i=0, j=1
// i=0, j=2
// i=0, j=3
// Breaking at i=0, j=4
// Done
```

Without the label, `break` would only exit the inner loop, and the outer loop would continue. Labels make your intent explicit.

**WHY labels exist:** In many languages, breaking out of nested loops requires a flag variable or refactoring into a function. Labels provide a direct, readable way to express "I want to exit this specific loop." They are commonly used when searching through nested data structures.

### continue with Labels

`continue` can also target a label to skip to the next iteration of an outer loop:

```go
package main

import "fmt"

func main() {
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if j == 1 {
                continue outer  // Skip to next iteration of the OUTER loop
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
}
// Output:
// i=0, j=0
// i=1, j=0
// i=2, j=0
```

Each time `j` reaches 1, the inner loop is abandoned and the outer loop moves to its next iteration. Without the label, `continue` would only skip the current iteration of the inner loop.

### Labels with switch

Labels also work with `switch` inside a `for` loop. A `break` inside a `switch` normally exits the `switch`, not the enclosing loop. Use a label to break the loop from within a switch:

```go
package main

import "fmt"

func main() {
loop:
    for i := 0; i < 10; i++ {
        switch i {
        case 5:
            fmt.Println("Reached 5, exiting loop")
            break loop     // Without the label, this would only break the switch
        default:
            fmt.Println(i)
        }
    }
}
// Output: 0, 1, 2, 3, 4, Reached 5, exiting loop
```

This is a common pattern. Without the label, `break` inside a `switch` only exits the `switch`, and the loop continues.

### Node.js Comparison: Labels

JavaScript actually has labels too, but they are rarely used:

```javascript
// Node.js -- labels work the same way
outer:
for (let i = 0; i < 5; i++) {
    for (let j = 0; j < 5; j++) {
        if (i + j === 4) {
            break outer;
        }
        console.log(`i=${i}, j=${j}`);
    }
}
```

### goto

Go includes a `goto` statement. Yes, really.

```go
package main

import "fmt"

func main() {
    i := 0

start:                      // Label
    if i >= 5 {
        goto end            // Jump to 'end' label
    }
    fmt.Println(i)
    i++
    goto start              // Jump back to 'start' label

end:
    fmt.Println("Done")
}
// Output: 0, 1, 2, 3, 4, Done
```

**WHY does goto exist?** It exists primarily for machine-generated code (code generators, parser generators) and for specific low-level patterns where it genuinely simplifies the control flow -- for example, centralizing error cleanup at the end of a function. The Go standard library uses it sparingly in a few places.

**When should you use goto?** Almost never. In everyday Go code, `for`, `break`, `continue`, and labels cover every case where you might think you need `goto`. There are hard restrictions to prevent the worst abuses:

- You **cannot** jump over variable declarations
- You **cannot** jump into a different scope (into a block you are not already in)

```go
// COMPILE ERROR: goto jumps over declaration of 'x'
goto skip
x := 10
skip:
fmt.Println("skipped")

// COMPILE ERROR: goto jumps into block
goto inside
{
inside:
    fmt.Println("inside")
}
```

**Rule of thumb:** If you are writing application code and reaching for `goto`, there is almost certainly a better way. Use `for` loops with `break`/`continue`, or refactor into a function.

---

## 6. defer Statement

### What defer Does

The `defer` statement schedules a function call to execute **when the surrounding function returns**. It runs regardless of how the function returns -- whether by reaching the end, hitting a `return` statement, or panicking.

```go
package main

import "fmt"

func main() {
    fmt.Println("first")
    defer fmt.Println("deferred")     // Scheduled to run when main() exits
    fmt.Println("second")
}
// Output:
// first
// second
// deferred
```

The deferred call happens **after** "second" but **before** the function actually returns to its caller.

### WHY defer Exists

`defer` was designed to solve the **resource cleanup problem**. In any language, when you open a file, acquire a lock, or establish a connection, you must ensure the resource is released -- even if an error occurs. Without `defer`, you have to remember to close/release at every return point:

```go
// WITHOUT defer -- error-prone
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }

    data, err := io.ReadAll(f)
    if err != nil {
        f.Close()           // Must remember to close here
        return err
    }

    err = process(data)
    if err != nil {
        f.Close()           // Must remember to close here too
        return err
    }

    f.Close()               // And here
    return nil
}
```

```go
// WITH defer -- clean and safe
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()         // Guaranteed to run when function returns

    data, err := io.ReadAll(f)
    if err != nil {
        return err          // f.Close() runs automatically
    }

    err = process(data)
    if err != nil {
        return err          // f.Close() runs automatically
    }

    return nil              // f.Close() runs automatically
}
```

The `defer` version has the cleanup right next to the acquisition. You write it once and it runs no matter how the function exits. This pattern is idiomatic Go.

### LIFO (Last-In, First-Out) Order

When you defer multiple calls, they execute in **reverse order** -- like a stack:

```go
package main

import "fmt"

func main() {
    defer fmt.Println("first deferred")   // Runs third
    defer fmt.Println("second deferred")  // Runs second
    defer fmt.Println("third deferred")   // Runs first

    fmt.Println("main body")
}
// Output:
// main body
// third deferred
// second deferred
// first deferred
```

**WHY LIFO?** Resources are often interdependent. If you open resource A, then open resource B that depends on A, you need to close B before closing A. LIFO ensures this happens naturally:

```go
func doWork() {
    db := openDatabase()
    defer db.Close()             // Closes second (after tx)

    tx := db.BeginTransaction()
    defer tx.Rollback()          // Closes first (before db)

    // ... work with transaction ...
}
```

### defer Evaluates Arguments Immediately

This is a crucial detail. The arguments to a deferred function are evaluated **at the time of the defer statement**, not when the deferred function executes:

```go
package main

import "fmt"

func main() {
    x := 10
    defer fmt.Println("Deferred x:", x)  // x is captured as 10 NOW

    x = 20
    fmt.Println("Current x:", x)
}
// Output:
// Current x: 20
// Deferred x: 10      <-- NOT 20!
```

The value `10` was captured when `defer` was called. Changing `x` afterward has no effect on the deferred call.

If you need the deferred function to see the latest value, use a closure:

```go
package main

import "fmt"

func main() {
    x := 10
    defer func() {
        fmt.Println("Deferred x:", x)  // Closure captures the variable, not the value
    }()

    x = 20
    fmt.Println("Current x:", x)
}
// Output:
// Current x: 20
// Deferred x: 20      <-- Sees the updated value via closure
```

### Common defer Use Cases

**1. Closing files:**

```go
f, err := os.Open("data.txt")
if err != nil {
    log.Fatal(err)
}
defer f.Close()
```

**2. Unlocking mutexes:**

```go
mu.Lock()
defer mu.Unlock()
// ... critical section ...
```

**3. Closing database connections:**

```go
db, err := sql.Open("postgres", connectionString)
if err != nil {
    log.Fatal(err)
}
defer db.Close()
```

**4. Recovering from panics:**

```go
func safeFunction() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from panic:", r)
        }
    }()

    panic("something went wrong")
}
```

**5. Timing a function:**

```go
func expensiveOperation() {
    start := time.Now()
    defer func() {
        fmt.Printf("Operation took %v\n", time.Since(start))
    }()

    // ... expensive work ...
}
```

### defer and Named Return Values

`defer` can modify named return values. This is an advanced pattern used for error handling:

```go
package main

import (
    "errors"
    "fmt"
)

func divide(a, b float64) (result float64, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)  // Modifies named return value
        }
    }()

    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 3)
    fmt.Printf("Result: %.2f, Error: %v\n", result, err)
    // Result: 3.33, Error: <nil>

    result, err = divide(10, 0)
    fmt.Printf("Result: %.2f, Error: %v\n", result, err)
    // Result: 0.00, Error: division by zero
}
```

### Node.js Comparison: try/finally

JavaScript has no `defer`. The closest equivalent is `try/finally`:

```javascript
// Node.js -- try/finally for cleanup
async function processFile(path) {
    const file = await fs.promises.open(path, "r");
    try {
        const data = await file.readFile("utf8");
        return process(data);
    } finally {
        await file.close();  // Runs whether try block succeeds or throws
    }
}
```

Go's `defer` is more flexible:

| Feature | Go `defer` | JS `try/finally` |
|---|---|---|
| Multiple cleanups | Multiple `defer` calls, each independent | One `finally` block handles everything |
| Execution order | LIFO (automatic) | Manual ordering in `finally` block |
| Placement | Right next to resource acquisition | `finally` is far from the opening code |
| Scope | Function-level | Block-level |

The biggest advantage of `defer` is **locality**: the cleanup code sits right next to the acquisition code, making it easy to verify that every resource is properly cleaned up.

### defer in Loops: A Warning

Deferred calls do not run until the **function** returns, not when a loop iteration ends. Deferring inside a loop can lead to resource leaks:

```go
// BAD: all files stay open until the function returns
func processAll(paths []string) {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            continue
        }
        defer f.Close()  // Does NOT close at end of loop iteration!
        // ... process f ...
    }
    // ALL files are closed here, at function return.
    // If 'paths' has 10,000 entries, 10,000 files are open simultaneously.
}
```

The fix is to extract the loop body into its own function:

```go
// GOOD: each file is closed after processing
func processAll(paths []string) {
    for _, path := range paths {
        processOne(path)
    }
}

func processOne(path string) {
    f, err := os.Open(path)
    if err != nil {
        return
    }
    defer f.Close()  // Closes when processOne returns (each iteration)
    // ... process f ...
}
```

Or use an anonymous function:

```go
// ALSO GOOD: anonymous function scopes the defer
func processAll(paths []string) {
    for _, path := range paths {
        func() {
            f, err := os.Open(path)
            if err != nil {
                return
            }
            defer f.Close()
            // ... process f ...
        }()
    }
}
```

---

## 7. Go vs Node.js Control Flow Comparison

### Side-by-Side Summary

```
+-----------------------+---------------------------+-----------------------------------+
| Concept               | Go                        | Node.js                           |
+-----------------------+---------------------------+-----------------------------------+
| Conditional           | if/else (no parens)       | if/else (parens required)         |
| Init in conditional   | if x := f(); x > 0       | (no equivalent)                   |
| Truthiness            | Must be explicit bool     | Truthy/falsy coercion             |
| Switch default        | Auto-break                | Fall-through (must add break)     |
| Multiple case values  | case "a", "b":            | case "a": case "b":              |
| Type checking         | Type switch               | typeof / instanceof               |
| Loop keyword          | for (only one)            | for, while, do...while            |
| Collection iteration  | for range                 | for...of, forEach, .map()         |
| Object key iteration  | for k := range m          | for...in, Object.keys()           |
| Cleanup               | defer                     | try/finally                       |
| Labels                | Supported                 | Supported (rarely used)           |
| goto                  | Supported (restricted)    | Not available                     |
+-----------------------+---------------------------+-----------------------------------+
```

### Full Example: Reading and Processing a File

**Go:**

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    // if-init: open file and check error in one statement
    f, err := os.Open("data.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    defer f.Close()  // Cleanup: guaranteed to run

    lineCount := 0
    scanner := bufio.NewScanner(f)

    // for loop: the only loop keyword needed
    for scanner.Scan() {
        line := scanner.Text()

        // tagless switch: clean multi-condition logic
        switch {
        case strings.HasPrefix(line, "#"):
            continue  // Skip comments
        case strings.TrimSpace(line) == "":
            continue  // Skip blank lines
        default:
            lineCount++
            fmt.Printf("Line %d: %s\n", lineCount, line)
        }
    }

    if err := scanner.Err(); err != nil {
        fmt.Fprintf(os.Stderr, "Read error: %v\n", err)
    }

    fmt.Printf("\nTotal lines: %d\n", lineCount)
}
```

**Node.js equivalent:**

```javascript
const fs = require("fs");
const readline = require("readline");

async function main() {
    let fileHandle;
    try {
        fileHandle = await fs.promises.open("data.txt", "r");
        const rl = readline.createInterface({
            input: fileHandle.createReadStream(),
            crlfDelay: Infinity,
        });

        let lineCount = 0;

        for await (const line of rl) {
            if (line.startsWith("#")) continue;
            if (line.trim() === "") continue;
            lineCount++;
            console.log(`Line ${lineCount}: ${line}`);
        }

        console.log(`\nTotal lines: ${lineCount}`);
    } catch (err) {
        console.error("Error:", err.message);
        process.exit(1);
    } finally {
        if (fileHandle) await fileHandle.close();
    }
}

main();
```

Notice how Go's `defer f.Close()` appears right after opening the file, while JavaScript's `finally` block is far removed from the `open` call.

---

## 8. Common Pitfalls and Gotchas

### Pitfall 1: Forgetting That range Copies Values

```go
type Player struct {
    Name  string
    Score int
}

players := []Player{
    {Name: "Alice", Score: 10},
    {Name: "Bob", Score: 20},
}

// BUG: 'p' is a copy -- modifying it does nothing to the slice
for _, p := range players {
    p.Score += 100
}
fmt.Println(players) // [{Alice 10} {Bob 20}] -- unchanged!

// FIX: use the index
for i := range players {
    players[i].Score += 100
}
fmt.Println(players) // [{Alice 110} {Bob 120}] -- correct!
```

### Pitfall 2: defer in Loops

As shown earlier, `defer` runs at function exit, not at loop iteration end. Always extract loop bodies into separate functions when deferring inside loops.

### Pitfall 3: Variable Shadowing in if-init

```go
x := 10
if x := someFunction(); x > 5 {
    // This 'x' is a NEW variable, shadowing the outer 'x'
    fmt.Println("inner x:", x)
}
fmt.Println("outer x:", x) // Still 10, not affected by the inner 'x'
```

This is valid Go but can be confusing. The `x` in the if-init is a **new** variable that shadows the outer `x`. Tools like `go vet` and linters can catch this.

### Pitfall 4: fallthrough is Unconditional

```go
x := 1
switch x {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two")   // This prints even though x != 2
    fallthrough
case 100:
    fmt.Println("hundred") // This prints even though x != 100
}
// Output:
// one
// two
// hundred
```

`fallthrough` does **not** check the next case's condition. It unconditionally transfers control to the next case body. This surprises many developers coming from other languages.

### Pitfall 5: Map Iteration Order is Random

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

// The order will be different each time you run this
for k, v := range m {
    fmt.Printf("%s: %d\n", k, v)
}
```

If you need sorted or deterministic output, collect the keys, sort them, then iterate:

```go
import "sort"

keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
    fmt.Printf("%s: %d\n", k, m[k])
}
```

### Pitfall 6: defer Argument Evaluation Timing

```go
func logStatus() {
    status := "started"
    defer fmt.Println("status:", status) // Captures "started" NOW

    status = "finished"
    fmt.Println("work done")
}
// Output:
// work done
// status: started       <-- Not "finished"!
```

Use a closure if you need the deferred function to see the final value.

---

## 9. Key Takeaways

1. **No parentheses in `if`/`for`/`switch` conditions.** Go requires braces instead, eliminating an entire class of bugs.

2. **The if-init statement** (`if x := f(); x > 0`) is idiomatic Go. It keeps variables tightly scoped and is used constantly in error handling.

3. **Go requires explicit booleans.** There is no truthiness. `if x` only works if `x` is already a `bool`.

4. **Switch breaks by default.** Use `fallthrough` for the rare case where you need fall-through. Use commas to list multiple values in one case.

5. **`for` is the only loop.** It covers traditional loops, while loops, infinite loops, and range-based iteration. One keyword, zero ambiguity.

6. **`range` returns copies.** If you need to modify elements in place, use the index to access the original.

7. **Map iteration order is random.** Never rely on a specific order when ranging over maps.

8. **Labels** let you break or continue outer loops from inner loops. They are cleaner than flag variables.

9. **`goto` exists but is almost never appropriate** in application code. Prefer `for` loops with `break`/`continue`.

10. **`defer` guarantees cleanup** when the function returns. Place `defer` immediately after acquiring a resource. Multiple defers execute in LIFO order.

11. **`defer` evaluates arguments immediately** but executes the call later. Use closures if you need to capture the final value of a variable.

12. **Never `defer` in a loop** without wrapping the body in a function. Deferred calls accumulate until the enclosing function returns.

---

## 10. Practice Exercises

### Exercise 1: FizzBuzz with switch
Write a program that prints numbers 1 to 100. For multiples of 3, print "Fizz"; for multiples of 5, print "Buzz"; for multiples of both, print "FizzBuzz". Use a tagless `switch` statement instead of `if/else`.

### Exercise 2: Matrix Search with Labels
Given a 2D slice (slice of slices) of integers, write a function that searches for a target value. Use a labeled `for` loop to break out of both loops when the target is found. Return the row and column indices, or (-1, -1) if not found.

### Exercise 3: Word Frequency Counter
Write a program that reads a string, splits it into words, and counts the frequency of each word using a map. Then print the words sorted alphabetically with their counts. Practice `for range` on both strings and maps.

### Exercise 4: Stack Using defer
Write a function that takes a slice of strings and prints them in reverse order using only `defer`. Each `defer` should print one string. Verify that the LIFO order of `defer` produces the reversed output.

### Exercise 5: Safe Division
Write a function `safeDivide(a, b float64) (float64, error)` that returns an error for division by zero. Use the if-init pattern to call this function and handle the error. Write a wrapper function that uses `defer` with `recover()` to catch any panics.

### Exercise 6: Number Classifier
Write a program that takes a number and classifies it using a tagless switch:
- Negative, zero, or positive
- If positive: small (1-10), medium (11-100), or large (101+)
- If even or odd
Print all applicable classifications.

### Exercise 7: Go vs JavaScript Translation
Take this JavaScript code and rewrite it in idiomatic Go, using Go-specific features (if-init, tagless switch, single for loop, defer):

```javascript
const fs = require("fs");

function processData(filename) {
    let file;
    try {
        file = fs.openSync(filename, "r");
        const data = fs.readFileSync(file, "utf8");
        const lines = data.split("\n");

        let result = [];
        for (let i = 0; i < lines.length; i++) {
            const line = lines[i].trim();
            if (!line) continue;
            if (line.startsWith("//")) continue;

            const num = parseInt(line);
            if (isNaN(num)) {
                console.log(`Skipping non-number: ${line}`);
                continue;
            }

            let category;
            if (num < 0) {
                category = "negative";
            } else if (num === 0) {
                category = "zero";
            } else if (num <= 100) {
                category = "small";
            } else {
                category = "large";
            }

            result.push({ value: num, category: category });
        }

        return result;
    } finally {
        if (file !== undefined) fs.closeSync(file);
    }
}
```
