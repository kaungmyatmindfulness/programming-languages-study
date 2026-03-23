# Chapter 2: Variables, Types, and Constants

> **Difficulty: Beginner** | **Estimated Study Time: 2-3 hours**
>
> This chapter is the bedrock of everything you will write in Go. If Chapter 1 got you running your first program, this chapter teaches you how Go thinks about data. Coming from Node.js, many of these concepts will feel restrictive at first -- that restriction is the point. Every constraint Go imposes exists to eliminate a class of bugs you have already encountered in production JavaScript.

---

## Table of Contents

1. [Variable Declaration](#1-variable-declaration)
2. [Basic Types](#2-basic-types)
3. [Type System Philosophy](#3-type-system-philosophy)
4. [String Internals](#4-string-internals)
5. [Type Conversion](#5-type-conversion)
6. [Constants](#6-constants)
7. [Type Aliases and Custom Types](#7-type-aliases-and-custom-types)
8. [Variable Scope and Shadowing](#8-variable-scope-and-shadowing)
9. [Go vs Node.js Comparison Summary](#9-go-vs-nodejs-comparison-summary)
10. [Key Takeaways](#10-key-takeaways)
11. [Practice Exercises](#11-practice-exercises)

---

## 1. Variable Declaration

Go gives you multiple ways to declare variables, but every one of them is explicit about what is happening. There is no hoisting, no temporal dead zone, no `undefined` silently lurking. You declare a variable, and it exists with a known value from that exact line forward.

### 1.1 The `var` Keyword

The most explicit way to declare a variable:

```go
package main

import "fmt"

func main() {
    var name string = "Alice"  // Fully explicit: keyword + name + type + value
    var age int = 30           // Same pattern for an integer
    fmt.Println(name, age)     // Output: Alice 30
}
```

**Line-by-line breakdown:**
- `var name string = "Alice"` -- The `var` keyword tells the compiler "I am creating a new variable." Then comes the name (`name`), the type (`string`), and the initial value (`"Alice"`).
- This is deliberately verbose. You can read it aloud: "variable name of type string equals Alice."

You can also let Go infer the type from the value:

```go
var name = "Alice"  // Go infers type 'string' from the right-hand side
var age = 30        // Go infers type 'int'
```

Or declare without an initial value (the variable gets its **zero value**, which we will discuss shortly):

```go
var name string  // name is "" (empty string)
var age int      // age is 0
var active bool  // active is false
```

**Node.js equivalent:**

```javascript
let name = "Alice";   // JS: type is determined at runtime, can change later
let age = 30;
let active;           // JS: this is 'undefined', not a zero value
```

### 1.2 Short Variable Declaration (`:=`)

Inside functions, Go offers a shorthand that is by far the most common way to declare variables:

```go
func main() {
    name := "Alice"   // Short declaration: Go infers the type, creates the variable
    age := 30
    active := true
    fmt.Println(name, age, active)
}
```

**Why does `:=` exist?** Because Go is pragmatic. The designers knew that inside function bodies, you almost always have a value to assign immediately. Writing `var name string = "Alice"` every time is tedious when `name := "Alice"` conveys the same information. The compiler can infer the type, so why force the programmer to write it?

**Critical rule:** `:=` only works inside functions. At the package level, you must use `var`:

```go
package main

// This works:
var PackageLevelVar = "I am accessible throughout this package"

// This does NOT compile:
// shortVar := "nope"  // syntax error: non-declaration statement outside function body

func main() {
    localVar := "I only exist inside main()"
    fmt.Println(PackageLevelVar, localVar)
}
```

**Why this restriction?** Package-level declarations are processed at compile time in a specific order. The `:=` syntax implies an imperative "do this now" action, which does not fit the declarative nature of package-level code. The `var` keyword makes the intent unambiguous.

### 1.3 Multiple Variable Declarations

Go lets you declare multiple variables in a single statement:

```go
// Multiple variables of the same type
var x, y, z int  // all three are int, all zero-valued to 0

// Multiple variables with values
var a, b, c = 1, "hello", true  // a is int, b is string, c is bool

// Short declaration with multiple variables
name, age := "Alice", 30

// Grouped var block (common at package level)
var (
    host     string = "localhost"
    port     int    = 8080
    debug    bool   = false
    maxConns int    = 100
)
```

The grouped `var` block is idiomatic Go for declaring related package-level variables. It communicates "these variables belong together conceptually."

**Node.js equivalent:**

```javascript
// JS destructuring is the closest analogy
const [a, b, c] = [1, "hello", true];

// JS has no grouped declaration block -- you just write separate lines
const host = "localhost";
const port = 8080;
const debug = false;
const maxConns = 100;
```

### 1.4 Zero Values -- Go's Answer to `undefined` and `null`

This is one of Go's most important design decisions. When you declare a variable without assigning a value, Go does **not** leave it undefined. It initializes it to a **zero value** determined by the type:

| Type | Zero Value |
|------|-----------|
| `int`, `float64`, etc. | `0` |
| `bool` | `false` |
| `string` | `""` (empty string) |
| Pointers, slices, maps, channels, interfaces, functions | `nil` |

```go
func main() {
    var i int        // 0
    var f float64    // 0.0
    var b bool       // false
    var s string     // ""

    fmt.Printf("int: %d, float: %f, bool: %t, string: %q\n", i, f, b, s)
    // Output: int: 0, float: 0.000000, bool: false, string: ""
}
```

**Why zero values instead of `undefined`/`null`?**

In Node.js, you fight two separate "nothing" values:

```javascript
let x;               // undefined -- not yet assigned
let y = null;        // null -- explicitly "no value"
console.log(x == y); // true (loose equality says they are the same)
console.log(x === y); // false (strict equality says they are different)

// This leads to defensive code everywhere:
function greet(name) {
    if (name === undefined || name === null) {
        name = "World";
    }
    // or the modern: name = name ?? "World";
    console.log(`Hello, ${name}`);
}
```

Go eliminates this entire class of bugs. A `string` is always a `string`. It may be empty, but it is never `undefined`, never `null`, never "not a string." You can call `len(s)` on a zero-valued string and get `0`. You can concatenate it with another string. It just works.

```go
func greet(name string) {
    // name is ALWAYS a valid string. It might be "", but it is never undefined.
    if name == "" {
        name = "World"
    }
    fmt.Printf("Hello, %s\n", name)
}
```

**The philosophy:** Every variable should be usable from the moment it is declared. No "half-initialized" states. No "check if it exists before using it." This eliminates one of the most common categories of runtime errors in JavaScript: `TypeError: Cannot read property 'x' of undefined`.

---

## 2. Basic Types

Go gives you fine-grained control over your data types. Where JavaScript has `number` for everything numeric, Go has a dozen numeric types. This is not accidental -- it is a direct consequence of Go being a systems language that cares about memory layout and performance.

### 2.1 Integer Types

```go
// Signed integers (can be negative)
var a int8   = 127        // -128 to 127                  (1 byte)
var b int16  = 32767      // -32,768 to 32,767            (2 bytes)
var c int32  = 2147483647 // -2,147,483,648 to 2.1 billion (4 bytes)
var d int64  = 9223372036854775807 // huge range            (8 bytes)

// Unsigned integers (zero and positive only)
var e uint8  = 255        // 0 to 255                     (1 byte)
var f uint16 = 65535      // 0 to 65,535                  (2 bytes)
var g uint32 = 4294967295 // 0 to ~4.2 billion            (4 bytes)
var h uint64 = 18446744073709551615 // massive             (8 bytes)

// Platform-dependent types
var i int    = 42         // 32-bit on 32-bit systems, 64-bit on 64-bit systems
var j uint   = 42         // same, but unsigned

// Special types
var k byte   = 65         // alias for uint8 (used for raw byte data)
var l rune   = 'A'        // alias for int32 (used for Unicode code points)
```

**Why so many integer types?**

In Node.js, `number` is always a 64-bit IEEE 754 floating-point value. Every number -- whether it is `1`, `3.14`, or `9007199254740991` -- occupies 8 bytes in memory and uses floating-point arithmetic:

```javascript
// JS: everything is a 64-bit float
const age = 30;           // 8 bytes for a value that fits in 1 byte
const statusCode = 200;   // 8 bytes for a value that fits in 2 bytes
const pixel = 255;        // 8 bytes for a value that fits in 1 byte
```

Go lets you choose the right tool for the job:

```go
age := uint8(30)          // 1 byte -- age will never be negative or > 255
statusCode := uint16(200) // 2 bytes -- HTTP codes fit in 0-65535
pixel := uint8(255)       // 1 byte -- perfect for RGBA values
```

**When does this matter?**
- **Arrays of millions of elements:** An array of 10 million `uint8` values uses 10 MB. An array of 10 million `int64` values uses 80 MB. If your values only need 1 byte, you are wasting 7x the memory with `int64`.
- **Network protocols:** When you are reading binary data from a TCP connection, you need exact control over byte sizes. A protocol might say "the next 2 bytes are the message length as a uint16." In Go, you read a `uint16`. In Node.js, you read a `Buffer` and manually extract bytes.
- **Hardware interaction:** Embedded systems, drivers, and low-level code need exact type sizes.

**For most application code,** just use `int`. It is the default, it is fast on your platform, and the compiler optimizes it well. Use specific sizes when you have a reason.

### 2.2 Floating-Point Types

```go
var pi32 float32 = 3.14159265    // 32-bit float (~7 decimal digits precision)
var pi64 float64 = 3.14159265358979323846 // 64-bit float (~15 decimal digits)
```

**Why two float types?**

`float64` gives you approximately the same precision as JavaScript's `number`. Use `float64` as your default for floating-point math. Use `float32` only when interfacing with systems that require it (some graphics APIs, certain file formats) or when memory is critical (millions of float values).

```go
// Default to float64
price := 19.99          // Go infers float64
var temperature = -40.0 // Also float64

// float32 only when you have a specific reason
var gpuVertex float32 = 1.5 // Graphics APIs often use float32
```

**Floating-point pitfalls (same in Go and JS):**

```go
fmt.Println(0.1 + 0.2)  // 0.30000000000000004 -- same as JavaScript!
```

```javascript
console.log(0.1 + 0.2);  // 0.30000000000000004 -- same problem
```

This is not a Go bug or a JS bug. It is how IEEE 754 floating-point arithmetic works on every computer. For financial calculations, use integers (cents instead of dollars) or a decimal library.

### 2.3 Complex Number Types

Go has built-in complex number support, which is unusual among mainstream languages:

```go
var c1 complex64  = 3 + 4i   // float32 real and imaginary parts
var c2 complex128 = 3 + 4i   // float64 real and imaginary parts

fmt.Println(c2)              // (3+4i)
fmt.Println(real(c2))        // 3  -- extract real part
fmt.Println(imag(c2))        // 4  -- extract imaginary part
```

You will rarely use these in web-facing code, but they are valuable in scientific computing, signal processing, and graphics -- domains where Go is used increasingly.

### 2.4 Boolean Type

```go
var active bool = true
var deleted bool       // zero value: false

// Boolean operations
result := active && !deleted  // logical AND, NOT
fmt.Println(result)           // true
```

**Critical difference from JavaScript:** Go booleans are strictly `true` or `false`. There is no truthy/falsy system:

```go
// This does NOT compile in Go:
// if 1 { ... }           // cannot use 1 (type int) as type bool
// if "hello" { ... }     // cannot use "hello" (type string) as type bool
// if somePointer { ... } // cannot use somePointer (type *int) as type bool
```

```javascript
// All of these "work" in JavaScript (for better or worse):
if (1) { /* runs */ }
if ("hello") { /* runs */ }
if ([]) { /* runs -- empty array is truthy! */ }
if (0) { /* skipped */ }
if ("") { /* skipped */ }
if (null) { /* skipped */ }
```

**Why Go rejects truthy/falsy:** Because truthy/falsy rules are a source of bugs. Quick: is `[]` truthy or falsy in JavaScript? (It is truthy.) Is `"0"` truthy or falsy? (Truthy.) Is `0` truthy or falsy? (Falsy.) So `"0" == false` is `true` but `Boolean("0")` is `true`. Go says: "If you mean to check a boolean condition, write a boolean expression."

### 2.5 String Type

```go
var greeting string = "Hello, World!"
name := "Go"

// String concatenation
message := greeting + " My name is " + name
fmt.Println(message)

// String length (in bytes, not characters!)
fmt.Println(len("Hello"))    // 5
fmt.Println(len("Hello!"))  // 7 (the emoji is multi-byte)
```

Strings in Go are immutable sequences of bytes. We will dive deep into this in [Section 4](#4-string-internals).

### 2.6 Byte and Rune

These are not separate types -- they are **aliases**:

```go
// byte is an alias for uint8
var b byte = 'A'      // stores the ASCII value 65
fmt.Println(b)        // 65
fmt.Printf("%c\n", b) // A

// rune is an alias for int32
var r rune = 'A'             // also 65, but can hold any Unicode code point
var emoji rune = '\U0001F600' // Grinning face emoji, code point 128512
fmt.Println(emoji)           // 128512
fmt.Printf("%c\n", emoji)   // prints the grinning face emoji
```

**Why two aliases?**
- `byte` communicates "I am working with raw bytes" -- file I/O, network data, binary protocols.
- `rune` communicates "I am working with a Unicode character" -- text processing, parsing, user-facing strings.

They convey **intent** to anyone reading your code, even though `byte` is just `uint8` and `rune` is just `int32` under the hood.

---

## 3. Type System Philosophy

This section explains one of the most fundamental differences between Go and JavaScript. Understanding this deeply will change how you think about writing code.

### 3.1 Static Typing vs Dynamic Typing

**Go is statically typed:** Every variable has a fixed type determined at compile time. The type can never change.

**JavaScript is dynamically typed:** A variable can hold any type at any time. Types are checked at runtime, if at all.

```go
// Go: the type is locked at declaration
age := 30          // age is int, forever
// age = "thirty"  // COMPILE ERROR: cannot use "thirty" (type string) as type int
```

```javascript
// JavaScript: the type is whatever you put in it
let age = 30;        // age is a number... for now
age = "thirty";      // now it's a string. JS doesn't care.
age = [30];          // now it's an array. Still no complaints.
age = { years: 30 }; // now it's an object. JavaScript is fine with this.
```

### 3.2 Why Go Chose Static Typing

**Reason 1: Bugs are caught before your code runs.**

Consider a function that calculates a discount:

```javascript
// JavaScript version
function applyDiscount(price, discount) {
    return price - (price * discount);
}

// This works fine:
applyDiscount(100, 0.2);  // 80

// This is a bug, but JS won't tell you until runtime:
applyDiscount("100", 0.2);  // "100" - "1000.2" = NaN... wait, actually:
                              // "100" * 0.2 = 20, "100" - 20 = 80
                              // JS coerced the string to a number. Sometimes.

applyDiscount("one hundred", 0.2);  // NaN -- silent failure at runtime
```

```go
// Go version
func applyDiscount(price float64, discount float64) float64 {
    return price - (price * discount)
}

// This works:
applyDiscount(100.0, 0.2)  // 80.0

// This does NOT compile -- the bug is caught immediately:
// applyDiscount("100", 0.2)
// Error: cannot use "100" (untyped string constant) as float64 value in argument
```

**Reason 2: Your editor becomes incredibly powerful.**

Because the compiler knows every type, your editor can provide:
- Autocomplete that shows exactly which methods are available on a value
- Inline error detection as you type (not after you run the code)
- Safe refactoring -- rename a field and the compiler tells you every place that breaks

In JavaScript, your editor is guessing. TypeScript was invented specifically to give JavaScript these benefits, which tells you something about how valuable static typing is.

**Reason 3: Code is self-documenting.**

```go
func processOrder(orderID int64, items []OrderItem, coupon string) (*Receipt, error) {
    // The function signature tells you EXACTLY what this function needs
    // and what it returns. No JSDoc needed. No runtime surprises.
}
```

```javascript
/**
 * @param {number} orderId
 * @param {Array<Object>} items
 * @param {string} coupon
 * @returns {Object}
 */
function processOrder(orderId, items, coupon) {
    // Without JSDoc (or TypeScript), you have no idea what this function
    // expects or returns. And even with JSDoc, nothing enforces it.
}
```

### 3.3 The Compile-Time Safety Net

Here is a real-world scenario. You have a Node.js API handler:

```javascript
// Node.js -- this bug ships to production
app.get("/users/:id", async (req, res) => {
    const user = await db.findUser(req.params.id);
    res.json({
        name: user.name,
        email: user.emal,  // TYPO: "emal" instead of "email"
    });
    // JavaScript says nothing. The response includes { email: undefined }.
    // Your frontend silently shows a blank email field.
    // Nobody notices for weeks.
});
```

```go
// Go -- this bug is caught before compilation finishes
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    user, err := db.FindUser(id)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    response := map[string]string{
        "name":  user.Name,
        "email": user.Emal,  // COMPILE ERROR: user.Emal undefined
        //                       (did you mean user.Email?)
    }
    json.NewEncoder(w).Encode(response)
}
```

The Go compiler catches the typo instantly. You fix it before the code ever runs. In JavaScript, that typo becomes a production bug that might go unnoticed for days.

---

## 4. String Internals

Strings in Go look simple on the surface, but their internals are fundamentally different from JavaScript strings. Understanding this will save you from subtle bugs when working with non-ASCII text.

### 4.1 Strings as Byte Slices

In Go, a string is a **read-only slice of bytes**. Under the hood, a string value consists of two things:
1. A pointer to an array of bytes
2. The length (in bytes)

```go
s := "Hello"
fmt.Println(len(s))    // 5 -- five bytes
fmt.Println(s[0])      // 72 -- the byte value of 'H'
fmt.Printf("%c\n", s[0]) // H -- print as character
```

This is straightforward for ASCII text. But watch what happens with Unicode:

```go
s := "Hello, world" // "world" in Chinese characters
fmt.Println(len(s))        // 19, NOT 12!

// Why? Let's look at the bytes:
// "Hello, " = 7 bytes (ASCII, 1 byte each)
// "world" (5 Chinese chars) = 12 bytes (3 bytes each in UTF-8)
// Total: 19 bytes
```

### 4.2 UTF-8 Encoding

Go source files are UTF-8 encoded, and strings contain raw UTF-8 bytes. Different characters use different numbers of bytes:

```go
func main() {
    ascii := "A"
    emoji := "\U0001F600" // grinning face
    chinese := "\u4e16"   // Chinese character for "world"

    fmt.Println(len(ascii))   // 1 byte -- ASCII fits in one byte
    fmt.Println(len(emoji))   // 4 bytes -- emoji needs 4 bytes in UTF-8
    fmt.Println(len(chinese)) // 3 bytes -- Chinese character needs 3 bytes
}
```

**Comparison with JavaScript:**

```javascript
// JavaScript strings are UTF-16 encoded sequences
const ascii = "A";
const emoji = "\u{1F600}";
const chinese = "\u4e16";

console.log(ascii.length);   // 1
console.log(emoji.length);   // 2 (!) -- emoji is a "surrogate pair" in UTF-16
console.log(chinese.length); // 1
```

Both languages have string length quirks, but the quirks are different:
- Go's `len()` returns **bytes**, not characters
- JavaScript's `.length` returns **UTF-16 code units**, not characters

Neither gives you the intuitive "number of visible characters" for all inputs.

### 4.3 Runes: Go's Answer to "Give Me the Characters"

To iterate over the actual characters (Unicode code points) in a Go string, you use **runes**:

```go
s := "Hello, Go!" // The last character is an emoji

// WRONG: iterating over bytes
for i := 0; i < len(s); i++ {
    fmt.Printf("%c ", s[i])
    // This will print garbled output for multi-byte characters
}
fmt.Println()

// CORRECT: iterating over runes using range
for i, r := range s {
    fmt.Printf("index=%d rune=%c (U+%04X)\n", i, r, r)
    // 'i' is the byte index, 'r' is the rune (character)
}

// Convert string to a slice of runes for "character count"
runes := []rune(s)
fmt.Println(len(runes))  // Actual character count
```

**Key insight:** When you use `range` on a string, Go automatically decodes the UTF-8 bytes into runes for you. The index `i` jumps by the byte width of each rune (1 for ASCII, 2-4 for others).

### 4.4 String Immutability

Go strings are immutable. You cannot change a byte within a string:

```go
s := "Hello"
// s[0] = 'h'  // COMPILE ERROR: cannot assign to s[0]

// To "modify" a string, you create a new one:
s = "h" + s[1:]  // "hello" -- creates a brand new string
```

This is the same as JavaScript:

```javascript
let s = "Hello";
s[0] = "h";        // Silently does nothing (no error, no change!)
console.log(s);     // "Hello" -- unchanged

// You create a new string instead:
s = "h" + s.slice(1);  // "hello"
```

**Why immutability?** Immutable strings are safe to share between goroutines (threads) without locks. This is crucial for Go's concurrency model, which you will learn in a later chapter.

### 4.5 String Operations and `fmt.Sprintf`

Common string operations and their JavaScript equivalents:

```go
import (
    "fmt"
    "strings"
)

func main() {
    // Concatenation
    full := "Hello" + " " + "World"

    // String formatting (Go's answer to template literals)
    name := "Alice"
    age := 30
    msg := fmt.Sprintf("Name: %s, Age: %d", name, age)
    // JavaScript: `Name: ${name}, Age: ${age}`

    // Common string functions
    fmt.Println(strings.Contains("Hello", "ell"))    // true
    fmt.Println(strings.HasPrefix("Hello", "He"))    // true
    fmt.Println(strings.HasSuffix("Hello", "lo"))    // true
    fmt.Println(strings.ToUpper("hello"))            // HELLO
    fmt.Println(strings.ToLower("HELLO"))            // hello
    fmt.Println(strings.TrimSpace("  hello  "))      // "hello"
    fmt.Println(strings.Split("a,b,c", ","))         // [a b c]
    fmt.Println(strings.Join([]string{"a","b","c"}, "-")) // "a-b-c"
    fmt.Println(strings.ReplaceAll("aabba", "a", "x"))    // "xxbbx"
    fmt.Println(strings.Index("hello", "ll"))              // 2
}
```

**Side-by-side comparison:**

| Operation | Go | JavaScript |
|-----------|------|-----------|
| Format string | `fmt.Sprintf("Hi %s", name)` | `` `Hi ${name}` `` |
| Contains | `strings.Contains(s, sub)` | `s.includes(sub)` |
| Starts with | `strings.HasPrefix(s, pre)` | `s.startsWith(pre)` |
| Ends with | `strings.HasSuffix(s, suf)` | `s.endsWith(suf)` |
| Uppercase | `strings.ToUpper(s)` | `s.toUpperCase()` |
| Split | `strings.Split(s, sep)` | `s.split(sep)` |
| Join | `strings.Join(arr, sep)` | `arr.join(sep)` |
| Trim spaces | `strings.TrimSpace(s)` | `s.trim()` |
| Replace all | `strings.ReplaceAll(s, old, new)` | `s.replaceAll(old, new)` |

Notice the difference in style: Go uses top-level functions in the `strings` package, while JavaScript uses methods on string objects. Go's approach is more functional; JavaScript's is more object-oriented.

### 4.6 Building Strings Efficiently

When you need to build a string from many parts, concatenation with `+` creates a new string each time (because strings are immutable). For loops, use `strings.Builder`:

```go
import "strings"

func main() {
    // SLOW for large strings: O(n^2) because each + creates a new string
    result := ""
    for i := 0; i < 1000; i++ {
        result += fmt.Sprintf("item %d, ", i)
    }

    // FAST: strings.Builder minimizes allocations
    var builder strings.Builder
    for i := 0; i < 1000; i++ {
        fmt.Fprintf(&builder, "item %d, ", i)
    }
    result = builder.String()
}
```

**JavaScript equivalent:**

```javascript
// Same problem -- concatenation in a loop is slow:
let result = "";
for (let i = 0; i < 1000; i++) {
    result += `item ${i}, `;
}

// Better approach:
const parts = [];
for (let i = 0; i < 1000; i++) {
    parts.push(`item ${i}`);
}
const result = parts.join(", ");
```

---

## 5. Type Conversion

Go requires **explicit** type conversion for every type change. JavaScript performs **implicit** type coercion. This is one of the biggest sources of bugs in JavaScript and one of Go's strongest safety features.

### 5.1 Explicit Conversion in Go

```go
func main() {
    // int to float64
    i := 42
    f := float64(i)    // Explicit: you are telling Go "convert this"
    fmt.Println(f)     // 42.0

    // float64 to int (truncates, does not round!)
    pi := 3.99
    n := int(pi)
    fmt.Println(n)     // 3 -- the .99 is chopped off, NOT rounded

    // int to string (does NOT do what you might think!)
    num := 65
    s1 := string(num)        // "A" -- converts to the Unicode character with code point 65!
    s2 := fmt.Sprintf("%d", num) // "65" -- this is what you actually want
    s3 := strconv.Itoa(num)     // "65" -- more efficient than Sprintf

    // string to int
    str := "42"
    val, err := strconv.Atoi(str)  // Returns value AND error
    if err != nil {
        fmt.Println("Not a valid number:", err)
        return
    }
    fmt.Println(val)  // 42
}
```

**Notice how `strconv.Atoi` returns an error?** This is Go saying: "Converting a string to an int can fail. You must handle that possibility." JavaScript silently returns `NaN` and lets the bug propagate.

### 5.2 JavaScript's Implicit Type Coercion

JavaScript converts types automatically in many contexts. This "helpfulness" is the source of countless bugs:

```javascript
// The Famous JavaScript WAT Moments

console.log("5" + 3);       // "53"  -- + with a string does concatenation
console.log("5" - 3);       // 2     -- - with a string does subtraction (?!)
console.log("5" * "3");     // 15    -- * coerces both to numbers
console.log(true + true);   // 2     -- true is 1
console.log([] + []);        // ""    -- empty arrays become empty strings
console.log([] + {});        // "[object Object]"  -- array + object = string
console.log({} + []);        // 0 (in some engines) or "[object Object]" (in others)

// Real-world bugs:
const userId = "123";
const nextId = userId + 1;     // "1231" -- string, not 124!
const amount = "10.50";
const tax = amount * 0.08;     // 0.84 -- this accidentally works
const total = amount + tax;     // "10.500.84" -- this doesn't!
```

**In Go, none of this compiles:**

```go
// Every single one of these is a compile error in Go:
// "5" + 3       -- mismatched types string and int
// "5" - 3       -- mismatched types
// true + true   -- cannot use + with bool

// You must be explicit about what you want:
result := "5" + strconv.Itoa(3)         // "53" -- string concatenation
result2 := 5 + 3                        // 8    -- integer addition
result3 := strconv.Atoi("5") + 3        // won't even compile without handling the error
```

### 5.3 Numeric Type Conversions

Go does not even convert between numeric types implicitly:

```go
var i int32 = 10
var j int64 = 20

// sum := i + j  // COMPILE ERROR: mismatched types int32 and int64

// You must explicitly convert:
sum := int64(i) + j  // Works: both are now int64
```

**Why is Go so strict?** Because implicit numeric conversion can lose data silently:

```go
var big int64 = 1000000
var small int8 = int8(big)  // You must explicitly opt into this
fmt.Println(small)           // 64 -- data loss! 1000000 doesn't fit in int8
// But at least you CHOSE to do this. The compiler made you write int8(big).
```

In JavaScript, all numbers are float64, so you never think about this. But you also cannot control memory usage, and you silently lose precision for integers above 2^53.

### 5.4 The `strconv` Package

This is your go-to package for string-to-number and number-to-string conversions:

```go
import "strconv"

func main() {
    // String to int
    i, err := strconv.Atoi("42")        // Atoi = "ASCII to integer"

    // String to int64 with base and bit size
    i64, err := strconv.ParseInt("FF", 16, 64)  // Parse hex "FF" as int64 = 255

    // String to float64
    f, err := strconv.ParseFloat("3.14", 64)

    // String to bool
    b, err := strconv.ParseBool("true")  // Also accepts "1", "t", "TRUE", etc.

    // Int to string
    s := strconv.Itoa(42)               // "42"

    // Float to string
    s2 := strconv.FormatFloat(3.14, 'f', 2, 64)  // "3.14"

    // Bool to string
    s3 := strconv.FormatBool(true)       // "true"
}
```

Every `Parse*` function returns `(value, error)`. You must handle the error. This is Go forcing you to think about: "What if this string is not a valid number?"

---

## 6. Constants

Constants in Go are truly constant -- their values are determined at compile time and can never change. This is fundamentally different from JavaScript's `const`.

### 6.1 Basic Constants

```go
const Pi = 3.14159265358979323846
const MaxRetries = 3
const AppName = "MyService"

func main() {
    fmt.Println(Pi)        // 3.141592653589793
    // Pi = 3.14           // COMPILE ERROR: cannot assign to Pi
}
```

**Grouped constant declarations:**

```go
const (
    StatusOK       = 200
    StatusNotFound = 404
    StatusError    = 500
)
```

### 6.2 JavaScript `const` vs Go `const`

This is a critical distinction. JavaScript's `const` does not make the value constant -- it makes the **binding** constant:

```javascript
// JavaScript const: the VARIABLE cannot be reassigned
const user = { name: "Alice", age: 30 };
// user = { name: "Bob" };  // TypeError: Assignment to constant variable

// But the VALUE is fully mutable!
user.name = "Bob";      // This works! The object is modified.
user.age = 31;          // This works too!
user.email = "b@b.com"; // You can even add new properties.

const arr = [1, 2, 3];
arr.push(4);            // [1, 2, 3, 4] -- the array is modified
arr[0] = 99;            // [99, 2, 3, 4] -- elements are mutable

// JavaScript const is really "const reference" not "const value"
```

Go's `const` means the value itself is immutable and must be known at compile time:

```go
const x = 42
// x = 43  // COMPILE ERROR

// You cannot use const with runtime values:
// const now = time.Now()  // COMPILE ERROR: time.Now() is not a constant expression

// You cannot use const with compound types:
// const user = User{Name: "Alice"}  // COMPILE ERROR: not a constant type

// Constants can only be: numbers, strings, booleans
const (
    name    = "Alice"
    age     = 30
    pi      = 3.14
    isAdmin = true
)
```

**Go constants must be determinable at compile time.** This means no function calls, no struct literals, no slices. Only primitive values and expressions composed of other constants.

### 6.3 Typed vs Untyped Constants

This is a uniquely Go concept with no JavaScript equivalent. It is subtle but powerful.

**Untyped constants** have a "kind" but no specific Go type:

```go
const x = 42       // untyped integer constant
const y = 3.14     // untyped floating-point constant
const z = "hello"  // untyped string constant
```

**Typed constants** have a specific Go type:

```go
const x int = 42         // typed: this is specifically an int
const y float64 = 3.14   // typed: this is specifically a float64
const z string = "hello" // typed: this is specifically a string
```

**Why does this distinction matter?**

Untyped constants are more flexible -- they can be used in expressions with different types without explicit conversion:

```go
const x = 42  // untyped -- think of it as "just the number 42"

var i int = x       // Works: 42 fits in int
var f float64 = x   // Works: 42 fits in float64
var b byte = x      // Works: 42 fits in byte (0-255)

// But typed constants are strict:
const y int = 42
var f2 float64 = y  // COMPILE ERROR: cannot use y (type int) as type float64
var f3 float64 = float64(y)  // OK: explicit conversion
```

**The design insight:** Untyped constants behave like mathematical values. The number 42 is not inherently an `int` or a `float64` -- it is just 42. Go lets you keep that flexibility until the constant is actually used in a context that requires a specific type.

### 6.4 `iota`: The Constant Enumerator

`iota` is Go's tool for creating incrementing constants. It starts at 0 and increments by 1 for each constant in a group:

```go
const (
    Sunday    = iota  // 0
    Monday            // 1 (iota increments automatically)
    Tuesday           // 2
    Wednesday         // 3
    Thursday          // 4
    Friday            // 5
    Saturday          // 6
)
```

**How `iota` works:**
- `iota` starts at 0 within each `const` block
- It increments by 1 for each successive constant specification
- Subsequent constants without an explicit value repeat the previous expression with the incremented `iota`

**Common patterns with `iota`:**

```go
// Skipping the zero value (useful for "unset" detection)
const (
    _         = iota  // 0, discarded (underscore means "ignore")
    RoleUser          // 1
    RoleAdmin         // 2
    RoleSuper         // 3
)

// Bit flags (powers of 2)
const (
    FlagRead    = 1 << iota  // 1 << 0 = 1
    FlagWrite                // 1 << 1 = 2
    FlagExecute              // 1 << 2 = 4
)
// Usage: permissions := FlagRead | FlagWrite  // 3 = read + write

// File sizes
const (
    _  = iota              // ignore 0
    KB = 1 << (10 * iota)  // 1 << 10 = 1024
    MB                     // 1 << 20 = 1,048,576
    GB                     // 1 << 30 = 1,073,741,824
    TB                     // 1 << 40 = 1,099,511,627,776
)
```

**JavaScript equivalent (there is no real equivalent):**

```javascript
// JavaScript has no iota. You manually define each value:
const ROLE_USER = 1;
const ROLE_ADMIN = 2;
const ROLE_SUPER = 3;

// Or use an object:
const Roles = Object.freeze({
    USER: 1,
    ADMIN: 2,
    SUPER: 3,
});

// TypeScript added enums to fill this gap:
// enum Role { User = 1, Admin, Super }
```

`iota` prevents off-by-one errors and keeps enumerated constants in sync as you add or remove values. If you insert a new role between `RoleUser` and `RoleAdmin`, everything below it shifts automatically.

---

## 7. Type Aliases and Custom Types

Go's `type` keyword lets you create new types based on existing ones. This is one of Go's most powerful features for writing clear, self-documenting code.

### 7.1 Creating Custom Types

```go
// Define custom types based on underlying types
type Celsius float64
type Fahrenheit float64
type UserID int64
type Email string

func main() {
    var temp Celsius = 100.0
    var bodyTemp Fahrenheit = 98.6

    // Even though both are float64 underneath, you cannot mix them:
    // total := temp + bodyTemp  // COMPILE ERROR: mismatched types Celsius and Fahrenheit

    // You must explicitly convert:
    total := temp + Celsius(bodyTemp)  // This compiles but is semantically wrong!
    // The type system warned you -- adding Celsius and Fahrenheit doesn't make sense.

    var id UserID = 12345
    var email Email = "alice@example.com"
    fmt.Println(id, email)
}
```

**Why custom types matter:**

Consider this function signature without custom types:

```go
func CreateUser(name string, email string, age int, departmentID int) error
```

Now consider this one with custom types:

```go
type Email string
type Age uint8
type DepartmentID int64

func CreateUser(name string, email Email, age Age, dept DepartmentID) error
```

The second version is self-documenting. And more importantly, the compiler prevents you from accidentally swapping arguments:

```go
// Without custom types -- this compiles but is a bug:
CreateUser("Alice", "Engineering", 30, 42)
// "Engineering" went into the email parameter. Oops.

// With custom types -- the compiler catches it:
CreateUser("Alice", Email("Engineering"), Age(30), DepartmentID(42))
// Wait, "Engineering" as an Email? The programmer is forced to think about it.
```

### 7.2 Methods on Custom Types

Custom types can have methods, which is how Go achieves object-oriented behavior without classes:

```go
type Celsius float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (c Celsius) String() string {
    return fmt.Sprintf("%.1f°C", c)
}

func main() {
    boiling := Celsius(100)
    fmt.Println(boiling)                // 100.0°C (String() method is called automatically)
    fmt.Println(boiling.ToFahrenheit()) // 212.0°F
}
```

You cannot add methods to built-in types directly. That is what custom types are for.

### 7.3 Type Aliases (Go 1.9+)

A type **alias** creates a new name for an existing type but they are the **same type**:

```go
// Type DEFINITION -- creates a new, distinct type
type UserID int64

// Type ALIAS -- creates just another name for the same type
type NodeID = int64
```

```go
var uid UserID = 42
var nid NodeID = 42
var raw int64 = 42

// nid = raw  // Works! NodeID is just another name for int64
// uid = raw  // COMPILE ERROR! UserID is a distinct type from int64
```

**When to use aliases:** Mainly during large refactorings to gradually migrate code. In everyday code, prefer type definitions for the safety they provide.

### 7.4 No JavaScript Equivalent

Vanilla JavaScript has no concept of custom types:

```javascript
// JavaScript: these are all just numbers. Nothing prevents mixing them.
const temperatureCelsius = 100;
const temperatureFahrenheit = 212;
const userId = 100;

// This is "valid" JavaScript -- adding a temperature and a user ID:
const nonsense = temperatureCelsius + userId; // 200 -- no error, no warning

// TypeScript's "branded types" pattern is the closest equivalent:
// type Celsius = number & { readonly __brand: unique symbol };
// But it's a workaround, not a language feature.
```

---

## 8. Variable Scope and Shadowing

Understanding scope in Go is straightforward if you come from JavaScript's `let`/`const` world. But there are important differences, especially around package-level scope and Go's capitalization convention.

### 8.1 Block Scope

Like JavaScript's `let` and `const`, Go variables are block-scoped:

```go
func main() {
    x := 10

    if true {
        y := 20          // y only exists inside this if block
        fmt.Println(x, y) // 10 20 -- x is visible from the outer scope
    }

    // fmt.Println(y)    // COMPILE ERROR: undefined: y
    fmt.Println(x)       // 10 -- x is still in scope
}
```

```javascript
// Same behavior with let/const in JavaScript:
function main() {
    let x = 10;

    if (true) {
        let y = 20;
        console.log(x, y);  // 10 20
    }

    // console.log(y);  // ReferenceError: y is not defined
    console.log(x);      // 10
}
```

### 8.2 Variable Shadowing

Shadowing occurs when an inner scope declares a variable with the same name as an outer scope:

```go
func main() {
    x := 10
    fmt.Println(x)  // 10

    if true {
        x := 20         // This creates a NEW variable x that shadows the outer one
        fmt.Println(x)  // 20 -- this is the inner x
    }

    fmt.Println(x)  // 10 -- the outer x was never modified!
}
```

**This is a common pitfall**, especially with Go's `:=` operator. Consider this bug:

```go
func getUser() (*User, error) {
    user, err := lookupInCache()
    if err != nil {
        // Trying to fall back to database
        user, err := lookupInDB()  // BUG: := creates NEW variables that shadow the outer ones!
        if err != nil {
            return nil, err
        }
        // The outer 'user' is still the one from lookupInCache()!
        _ = user // this is the inner user, which is about to go out of scope
    }
    return user, nil  // Returns the cache result, not the DB result!
}

// Fixed version:
func getUser() (*User, error) {
    user, err := lookupInCache()
    if err != nil {
        user, err = lookupInDB()  // = not := -- assigns to existing variables
        if err != nil {
            return nil, err
        }
    }
    return user, nil
}
```

**Rule of thumb:** Use `:=` when creating a new variable. Use `=` when assigning to an existing one. The compiler will tell you if you use `=` on an undeclared variable, but it will not warn you about accidental shadowing with `:=`. Some linters (like `go vet` with shadow checking or `golangci-lint`) can catch this.

### 8.3 Package Scope

Variables declared outside of any function are accessible throughout the entire package:

```go
package main

var appName = "MyService"  // Package-scoped: accessible in all functions in this file
                           // and all files in the "main" package

func main() {
    fmt.Println(appName)     // "MyService"
    printAppName()
}

func printAppName() {
    fmt.Println(appName)     // "MyService" -- same variable
}
```

### 8.4 Exported vs Unexported (The Capitalization Convention)

This is one of Go's most distinctive features. Instead of using keywords like `public`/`private`, Go uses capitalization:

- **Capitalized = Exported (public):** Accessible from other packages
- **Lowercase = Unexported (private):** Only accessible within the same package

```go
package user

// Exported: other packages can use these
var MaxLoginAttempts = 5              // Accessible as user.MaxLoginAttempts
type User struct {                    // Accessible as user.User
    Name  string                      // Exported field
    Email string                      // Exported field
    age   int                         // Unexported field -- only this package can see it
}

func NewUser(name, email string) User { // Exported function
    return User{Name: name, Email: email, age: 0}
}

// Unexported: only this package can use these
var defaultTimeout = 30                  // Other packages cannot see this
func validateEmail(email string) bool {  // Other packages cannot call this
    return strings.Contains(email, "@")
}
```

Using from another package:

```go
package main

import "myapp/user"

func main() {
    u := user.NewUser("Alice", "alice@example.com") // OK: NewUser is exported
    fmt.Println(u.Name)                              // OK: Name is exported
    // fmt.Println(u.age)                            // COMPILE ERROR: age is unexported
    // user.validateEmail("test")                    // COMPILE ERROR: unexported function
}
```

**Why capitalization instead of keywords?**

1. **No syntax overhead:** You don't need `public`/`private`/`protected` keywords cluttering your code.
2. **Visible at a glance:** When reading code, you instantly know what is exported by looking at the first letter.
3. **Consistent convention:** Every Go programmer follows the same rule. There is no debate about access modifier style.

**JavaScript comparison:**

```javascript
// JavaScript has no true access control in plain code.
// Everything on an object is public by default.

class User {
    constructor(name, email) {
        this.name = name;     // "public" (convention, not enforced)
        this._age = 0;        // "private" by convention (underscore prefix), but fully accessible
    }
}

const u = new User("Alice", "alice@example.com");
console.log(u._age);  // 0 -- the underscore convention is not enforced

// ES2022 added real private fields:
class ModernUser {
    #age = 0;                // Actually private
    constructor(name) {
        this.name = name;
    }
}
const mu = new ModernUser("Alice");
// console.log(mu.#age);   // SyntaxError: Private field '#age' must be declared
```

### 8.5 JavaScript `var` vs `let` vs `const` -- Scoping Comparison

For completeness, here is how JavaScript's three variable keywords compare to Go's approach:

```javascript
// JavaScript var: function-scoped, hoisted, can be redeclared
function example() {
    console.log(x);  // undefined (hoisted, but not initialized)
    var x = 10;
    var x = 20;       // No error! var allows redeclaration.

    if (true) {
        var y = 30;   // NOT block-scoped! y leaks out of the if block.
    }
    console.log(y);   // 30 -- y is accessible here
}

// JavaScript let: block-scoped, not hoisted (TDZ), cannot be redeclared
function example2() {
    // console.log(x);  // ReferenceError: Cannot access 'x' before initialization
    let x = 10;
    // let x = 20;      // SyntaxError: Identifier 'x' has already been declared

    if (true) {
        let y = 30;
    }
    // console.log(y);  // ReferenceError: y is not defined
}

// JavaScript const: like let, but the binding is immutable
function example3() {
    const x = 10;
    // x = 20;          // TypeError: Assignment to constant variable
}
```

**Go's approach is closest to JavaScript's `let`:** block-scoped, no hoisting, no redeclaration in the same scope. But Go has no equivalent of JavaScript's `var` (function-scoped, hoisted) because that behavior is widely considered a design mistake.

---

## 9. Go vs Node.js Comparison Summary

| Concept | Go | Node.js (JavaScript) |
|---------|------|---------------------|
| Variable declaration | `var x int = 10` or `x := 10` | `let x = 10` or `const x = 10` |
| Type system | Static (compile-time) | Dynamic (runtime) |
| Type annotation | Built into the language | None (use TypeScript) |
| Uninitialized value | Zero value (`0`, `""`, `false`, `nil`) | `undefined` |
| Null-like value | `nil` (only for pointers, slices, maps, etc.) | `null` and `undefined` (two!) |
| Type conversion | Explicit only: `float64(x)` | Implicit coercion: `"5" - 3 === 2` |
| String encoding | UTF-8 bytes | UTF-16 code units |
| Character type | `rune` (alias for `int32`) | No dedicated type |
| Constants | Truly immutable, compile-time only | `const` is immutable binding, value can mutate |
| Enums | `iota` in const blocks | No native support (Objects or TypeScript enums) |
| Custom types | `type UserID int64` | No native support |
| Access control | Capitalization: `Exported` vs `unexported` | `#private` fields (ES2022) |
| Scope | Block scope only | `var`: function scope; `let`/`const`: block scope |
| String formatting | `fmt.Sprintf("Hi %s", name)` | `` `Hi ${name}` `` (template literals) |
| String to number | `strconv.Atoi("42")` returns `(int, error)` | `parseInt("42")` returns `NaN` on failure |
| Number to string | `strconv.Itoa(42)` | `String(42)` or `(42).toString()` |

---

## 10. Key Takeaways

1. **Zero values eliminate `undefined`.** Every variable in Go has a known, usable value from the moment it is declared. This alone prevents a huge category of JavaScript bugs.

2. **Static typing is not a burden -- it is a safety net.** The compiler catches type mismatches, typos in field names, and wrong argument types before your code ever runs. TypeScript exists because this safety is so valuable that the JavaScript community reinvented it.

3. **Explicit conversion prevents coercion bugs.** Go will never silently convert `"5" + 3` to `"53"` or `2`. Every type conversion is your deliberate choice.

4. **Go's `const` is a true constant.** Unlike JavaScript's `const`, which only prevents reassignment of the variable, Go's `const` means the value itself is immutable and known at compile time.

5. **Custom types create semantic meaning.** `type Celsius float64` tells both the compiler and other developers that this number represents a temperature in Celsius, not just any float.

6. **Capitalization is access control.** Exported (`Public`) and unexported (`private`) identifiers are determined by the first letter. No keywords needed.

7. **Strings are byte slices.** Use `len()` for byte count, `[]rune(s)` for character count, and `range` for iterating over characters. This matters for any non-ASCII text.

8. **`:=` creates, `=` assigns.** Be vigilant about accidental shadowing when using `:=` inside inner blocks.

---

## 11. Practice Exercises

### Exercise 1: Zero Value Explorer
Write a program that declares variables of every basic type (int, float64, bool, string, byte, rune) without assigning values, then prints their zero values using `fmt.Printf` with appropriate format verbs.

```go
// Expected output:
// int: 0
// float64: 0.000000
// bool: false
// string: ""
// byte: 0 (character: )
// rune: 0 (character: )
```

### Exercise 2: Temperature Converter
Create custom types `Celsius` and `Fahrenheit`. Write methods on each type to convert to the other. The program should:
- Define `Celsius` and `Fahrenheit` as custom types based on `float64`
- Add a `ToFahrenheit()` method on `Celsius`
- Add a `ToCelsius()` method on `Fahrenheit`
- Add a `String()` method on each that formats with the degree symbol and unit
- Test with boiling point (100C / 212F) and freezing point (0C / 32F)

### Exercise 3: String Analyzer
Write a function that takes a string and returns:
- The number of bytes
- The number of runes (characters)
- The number of words (split by spaces)
- Whether the byte count and rune count differ (indicating multi-byte characters)

Test it with: `"Hello"`, `"Hello, World"` (with non-ASCII characters), and `"Go is fun!"` (with emoji).

### Exercise 4: Type-Safe ID System
Create custom types for `UserID`, `OrderID`, and `ProductID` (all based on `int64`). Write a function `GetOrderDetails(orderID OrderID)` and try to call it with a `UserID`. Observe the compile error. Then write proper conversion functions.

### Exercise 5: Iota Exploration
Create a `const` block using `iota` that defines file permissions (Read, Write, Execute) as bit flags. Write a function that takes a permission value and prints which permissions are set. For example, `Read | Write` should print "Read, Write".

```go
// Expected usage:
// permissions := Read | Write
// printPermissions(permissions)
// Output: Read Write
```

### Exercise 6: Coercion Bug Hunt
Take the following JavaScript code and rewrite it in Go. Notice how many bugs the Go compiler catches that JavaScript silently accepts:

```javascript
function processPayment(amount, currency, userId) {
    const tax = amount * 0.08;
    const total = amount + tax;
    const receipt = "Payment: $" + total + " " + currency + " for user " + userId;

    if (total) {  // truthy check
        return receipt;
    }
    return "Payment failed";
}

// These all "work" in JavaScript:
processPayment(100, "USD", 42);
processPayment("100", "USD", 42);       // amount is a string
processPayment(100, 42, "USD");          // arguments swapped
processPayment(undefined, "USD", 42);    // amount is undefined
```

---

**Next Chapter:** [Chapter 3 - Control Flow: if, for, switch, and defer](./03-control-flow.md) -- Go's approach to flow control, including why there is only one loop keyword and how `defer` replaces try/finally.
