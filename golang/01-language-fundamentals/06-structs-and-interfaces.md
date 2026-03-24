# Chapter 6: Structs and Interfaces

> **Level:** Beginner | **Prerequisites:** Chapters 1-5 (basics, control flow, functions, arrays/slices, maps)
> **Goal:** Understand Go's type system through structs and interfaces -- the twin pillars that replace classes and inheritance from OOP languages.

---

## Table of Contents

1. [Structs](#1-structs)
2. [Struct Methods](#2-struct-methods)
3. [Struct Embedding](#3-struct-embedding-composition)
4. [Struct Tags](#4-struct-tags)
5. [Interfaces](#5-interfaces)
6. [The Empty Interface (any / interface{})](#6-the-empty-interface)
7. [Interface Composition](#7-interface-composition)
8. [Common Standard Library Interfaces](#8-common-standard-library-interfaces)
9. [Composition vs Inheritance -- The Philosophy](#9-composition-vs-inheritance)
10. [Key Takeaways](#10-key-takeaways)
11. [Practice Exercises](#11-practice-exercises)

---

## 1. Structs

### What Is a Struct?

A struct is a composite data type that groups together zero or more named fields, each with a type. If you come from JavaScript or TypeScript, think of a struct as a **typed, fixed-shape object** -- like a TypeScript interface that also *defines the data storage*, not just the shape.

```
+---------------------------------------------+
|  WHY structs instead of classes?             |
|                                              |
|  Go's creators (Rob Pike, Ken Thompson,      |
|  Robert Griesemer) deliberately removed:     |
|                                              |
|   - Class hierarchies                        |
|   - Constructors / destructors               |
|   - Method overloading                       |
|   - Inheritance chains                       |
|                                              |
|  Their reasoning: deep class hierarchies     |
|  create fragile, tightly-coupled code.       |
|  Structs + interfaces give you the same      |
|  power with far less complexity.             |
+---------------------------------------------+
```

### Defining a Struct

```go
// Go
type User struct {
    ID        int
    Username  string
    Email     string
    IsActive  bool
}
```

```js
// JavaScript/TypeScript equivalent
// TypeScript interface (shape only, no storage)
interface User {
    id: number;
    username: string;
    email: string;
    isActive: boolean;
}

// JavaScript class (shape + behavior + storage)
class User {
    constructor(id, username, email, isActive) {
        this.id = id;
        this.username = username;
        this.email = email;
        this.isActive = isActive;
    }
}
```

**Key difference:** In Go, data (struct) and behavior (methods) are defined *separately*. In JS/TS, a class bundles both together. Go's approach means you can add methods to types you define without touching the struct definition itself.

### Creating Instances

Go gives you multiple ways to create struct instances:

```go
package main

import "fmt"

type User struct {
    ID       int
    Username string
    Email    string
    IsActive bool
}

func main() {
    // 1. Zero-value initialization (all fields get their zero values)
    var u1 User
    fmt.Println(u1) // {0  "" "" false}

    // 2. Struct literal with named fields (PREFERRED -- order doesn't matter)
    u2 := User{
        ID:       1,
        Username: "alice",
        Email:    "alice@example.com",
        IsActive: true,
    }
    fmt.Println(u2) // {1 alice alice@example.com true}

    // 3. Struct literal without field names (positional -- FRAGILE, avoid in production)
    u3 := User{2, "bob", "bob@example.com", false}
    fmt.Println(u3) // {2 bob bob@example.com false}

    // 4. Using new() -- returns a POINTER to a zero-value struct
    u4 := new(User)
    u4.ID = 3
    u4.Username = "charlie"
    fmt.Println(*u4) // {3 charlie  false}

    // 5. Address-of struct literal -- returns a pointer (VERY common pattern)
    u5 := &User{
        ID:       4,
        Username: "diana",
        Email:    "diana@example.com",
        IsActive: true,
    }
    fmt.Println(*u5) // {4 diana diana@example.com true}
}
```

**WHY is `&User{...}` so common?** Because many functions accept pointers to structs (to avoid copying large structs or to allow mutation). Creating a pointer directly saves you from the two-step `u := User{...}; ptr := &u`.

### Field Access

```go
user := User{ID: 1, Username: "alice"}

// Direct field access
fmt.Println(user.Username)  // "alice"

// Modifying fields
user.Email = "alice@example.com"

// Pointer field access -- Go automatically dereferences!
ptr := &user
fmt.Println(ptr.Username)   // "alice" -- same as (*ptr).Username
ptr.Email = "new@email.com" // Go dereferences for you
```

**Coming from JS:** In JavaScript, `obj.field` and `obj->field` are not a thing -- you always use dot notation. Go is the same. Even when you have a pointer, Go lets you use dot notation. The compiler handles dereferencing behind the scenes.

### Zero Values Matter

Every field in a Go struct gets its zero value if not explicitly set:

```go
type Config struct {
    Host     string  // ""
    Port     int     // 0
    Debug    bool    // false
    Timeout  float64 // 0.0
    MaxRetry *int    // nil (pointer types zero to nil)
}

cfg := Config{}
// cfg.Host is "", cfg.Port is 0, cfg.Debug is false, etc.
```

**WHY this matters:** In JavaScript, unset fields are `undefined`. In Go, there is no `undefined` or `null` for value types. A string is always at least `""`, an int is always at least `0`. This eliminates an entire class of "undefined is not a function" bugs. However, it means you cannot distinguish "user explicitly set port to 0" from "user didn't set port at all" -- see the pointer pattern below.

```go
// The "pointer to distinguish zero from unset" pattern
type Config struct {
    Port *int // nil means "not set", &0 means "explicitly set to 0"
}
```

### Exported vs Unexported Fields

```go
type User struct {
    ID       int    // Exported (uppercase) -- visible outside the package
    Username string // Exported
    password string // Unexported (lowercase) -- only visible within this package
}
```

**Comparison with JS/TS:**
```js
// JavaScript -- no true private fields until recently
class User {
    #password;  // Private field (ES2022)
    constructor(id, username, password) {
        this.id = id;
        this.username = username;
        this.#password = password;
    }
}
```

Go uses *naming convention enforced by the compiler* (uppercase = exported, lowercase = unexported). No keywords like `public`, `private`, or `protected`. The visibility boundary is the **package**, not the struct.

### Anonymous Structs

Sometimes you need a one-off struct without defining a named type:

```go
// Inline / anonymous struct
point := struct {
    X, Y int
}{10, 20}

fmt.Println(point.X) // 10

// Very useful in tests and JSON handling
config := struct {
    Host string `json:"host"`
    Port int    `json:"port"`
}{
    Host: "localhost",
    Port: 8080,
}
```

**JS equivalent:**
```js
const point = { x: 10, y: 20 };  // JS objects are always "anonymous structs"
```

---

## 2. Struct Methods

### Defining Methods

In Go, methods are functions with a special **receiver** argument. The receiver binds the function to a type.

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Method with a value receiver
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Method with a value receiver
func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Println(rect.Area())      // 50
    fmt.Println(rect.Perimeter()) // 30
}
```

**JS equivalent:**
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
```

**Key difference:** In Go, methods are defined *outside* the struct body. The receiver `(r Rectangle)` is like an explicit `this`. In JS, `this` is implicit and sometimes confusing (arrow functions, `.bind()`, etc.). Go's explicit receiver eliminates all `this`-binding confusion.

### Value Receivers vs Pointer Receivers

This is one of the most important concepts in Go. Understand this and you understand half of Go's type system.

```go
type Counter struct {
    Count int
}

// Value receiver -- gets a COPY of the struct
func (c Counter) GetCount() int {
    return c.Count
}

// Value receiver -- CANNOT modify the original
func (c Counter) IncrementBroken() {
    c.Count++ // This modifies the copy, not the original!
}

// Pointer receiver -- gets a POINTER to the original struct
func (c *Counter) Increment() {
    c.Count++ // This modifies the original
}

func main() {
    counter := Counter{Count: 0}

    counter.IncrementBroken()
    fmt.Println(counter.Count) // 0 -- the original was NOT modified!

    counter.Increment()
    fmt.Println(counter.Count) // 1 -- the original WAS modified
}
```

**ASCII diagram of what happens:**

```
VALUE RECEIVER: func (c Counter) IncrementBroken()

    counter (original)          c (copy)
    +-----------+               +-----------+
    | Count: 0  |  -- copy -->  | Count: 0  |
    +-----------+               +-----------+
                                | Count: 1  |  (modified copy, then discarded)
                                +-----------+

POINTER RECEIVER: func (c *Counter) Increment()

    counter (original)
    +-----------+
    | Count: 0  |  <-- c points here
    +-----------+
    | Count: 1  |  (original modified via pointer)
    +-----------+
```

### When to Use Which: The Rule of Thumb

```
+---------------------------------------------------------------+
|  USE A POINTER RECEIVER (*T) WHEN:                            |
|                                                               |
|  1. The method needs to MODIFY the receiver                   |
|  2. The struct is LARGE (avoid copying overhead)              |
|  3. You need CONSISTENCY -- if any method uses a pointer      |
|     receiver, ALL methods on that type should too             |
|                                                               |
+---------------------------------------------------------------+
|  USE A VALUE RECEIVER (T) WHEN:                               |
|                                                               |
|  1. The method only READS, never modifies                     |
|  2. The struct is SMALL (a few fields, basic types)           |
|  3. You want the type to be safe for concurrent use           |
|     (copies can't cause data races)                           |
|                                                               |
+---------------------------------------------------------------+
|  THE GOLDEN RULE:                                             |
|                                                               |
|  When in doubt, use a pointer receiver.                       |
|  If ANY method on a type uses a pointer receiver,             |
|  ALL methods on that type should use pointer receivers.       |
|  Consistency matters more than micro-optimization.            |
+---------------------------------------------------------------+
```

### Real-World Example: A User Service

```go
type User struct {
    ID        int
    Username  string
    Email     string
    IsActive  bool
    loginCount int
}

// Constructor function (Go convention -- no actual constructors)
func NewUser(id int, username, email string) *User {
    return &User{
        ID:        id,
        Username:  username,
        Email:     email,
        IsActive:  true,
        loginCount: 0,
    }
}

// Pointer receiver -- modifies the user
func (u *User) Deactivate() {
    u.IsActive = false
}

// Pointer receiver -- modifies the user
func (u *User) RecordLogin() {
    u.loginCount++
}

// Pointer receiver -- consistency (even though it only reads)
func (u *User) String() string {
    status := "active"
    if !u.IsActive {
        status = "inactive"
    }
    return fmt.Sprintf("User{%s (%s), logins: %d}", u.Username, status, u.loginCount)
}
```

**JS equivalent:**
```js
class User {
    #loginCount = 0;

    constructor(id, username, email) {
        this.id = id;
        this.username = username;
        this.email = email;
        this.isActive = true;
    }

    deactivate() {
        this.isActive = false;
    }

    recordLogin() {
        this.#loginCount++;
    }

    toString() {
        const status = this.isActive ? "active" : "inactive";
        return `User{${this.username} (${status}), logins: ${this.#loginCount}}`;
    }
}
```

**Notice:** Go uses `NewUser()` as a convention for constructors. There is no `constructor` keyword. The function returns a pointer (`*User`) so callers can call pointer-receiver methods on it. This is one of the most common Go patterns.

### Methods Can Be Defined on Any Named Type

```go
// You can define methods on ANY named type, not just structs
type Celsius float64
type Fahrenheit float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (f Fahrenheit) ToCelsius() Celsius {
    return Celsius((f - 32) * 5 / 9)
}

func main() {
    boiling := Celsius(100)
    fmt.Println(boiling.ToFahrenheit()) // 212
}
```

This has no JS equivalent. In JavaScript, you cannot add methods to primitive types (without prototype hacking). Go's approach gives you type safety (`Celsius` and `Fahrenheit` are distinct types) with method convenience.

---

## 3. Struct Embedding (Composition)

### What Is Embedding?

Embedding is Go's replacement for inheritance. Instead of "User IS-A Person" (inheritance), Go says "User HAS-A Person" (composition), but makes it feel almost like inheritance by **promoting** the embedded type's fields and methods.

```go
// Base type
type Address struct {
    Street string
    City   string
    State  string
    Zip    string
}

func (a Address) FullAddress() string {
    return fmt.Sprintf("%s, %s, %s %s", a.Street, a.City, a.State, a.Zip)
}

// Embedding Address into User
type User struct {
    ID      int
    Name    string
    Address // <-- Embedded! No field name, just the type.
}

func main() {
    u := User{
        ID:   1,
        Name: "Alice",
        Address: Address{
            Street: "123 Main St",
            City:   "Springfield",
            State:  "IL",
            Zip:    "62701",
        },
    }

    // Promoted fields -- access Address fields directly on User
    fmt.Println(u.Street)       // "123 Main St"  (same as u.Address.Street)
    fmt.Println(u.City)         // "Springfield"

    // Promoted methods -- call Address methods directly on User
    fmt.Println(u.FullAddress()) // "123 Main St, Springfield, IL 62701"

    // You can still access the embedded struct explicitly
    fmt.Println(u.Address.City)  // "Springfield"
}
```

### Embedding vs Inheritance: Visual Comparison

```
INHERITANCE (JS / Java / C++):

    Person
      ^
      |  extends (IS-A)
      |
    Employee
      ^
      |  extends (IS-A)
      |
    Manager
      ^
      |  extends (IS-A)
      |
    SeniorManager

    Problem: Deep hierarchy. Change Person? Everything breaks.
             "The fragile base class problem."

-----------------------------------------------------------------

COMPOSITION / EMBEDDING (Go):

    +------------------+
    |     Manager      |
    |                  |
    |  +------------+  |     Manager HAS-A Employee
    |  |  Employee  |  |     Employee HAS-A Person
    |  |            |  |     Person HAS-A Address
    |  | +--------+ |  |
    |  | | Person | |  |     But thanks to promotion, you can call:
    |  | |        | |  |       manager.Name        (Person's field)
    |  | | +----+ | |  |       manager.Street       (Address's field)
    |  | | |Addr| | |  |       manager.FullAddress() (Address's method)
    |  | | +----+ | |  |
    |  | +--------+ |  |
    |  +------------+  |
    +------------------+
```

### JS Inheritance vs Go Embedding: Side by Side

```js
// JavaScript: Inheritance
class Animal {
    constructor(name) {
        this.name = name;
    }
    speak() {
        return `${this.name} makes a sound`;
    }
}

class Dog extends Animal {
    constructor(name, breed) {
        super(name);  // Must call super()!
        this.breed = breed;
    }
    speak() {  // Override
        return `${this.name} barks`;
    }
}

const d = new Dog("Rex", "Labrador");
console.log(d.speak());  // "Rex barks"
console.log(d.name);     // "Rex" (inherited)
```

```go
// Go: Embedding (composition)
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return a.Name + " makes a sound"
}

type Dog struct {
    Animal // Embed Animal
    Breed  string
}

// "Override" -- define Speak on Dog
func (d Dog) Speak() string {
    return d.Name + " barks"
}

func main() {
    d := Dog{
        Animal: Animal{Name: "Rex"},
        Breed:  "Labrador",
    }
    fmt.Println(d.Speak()) // "Rex barks" (Dog's method)
    fmt.Println(d.Name)    // "Rex" (promoted from Animal)

    // You can still call the "parent" method explicitly
    fmt.Println(d.Animal.Speak()) // "Rex makes a sound"
}
```

### WHY Go Chose Embedding Over Inheritance

1. **No fragile base class problem.** Changing the embedded struct does not silently break the outer struct's behavior through virtual dispatch.
2. **Explicit over implicit.** You can always see exactly which type provides which method. No hunting through a class hierarchy.
3. **Multiple embedding.** Go allows embedding multiple types (like multiple inheritance but without the "diamond problem").
4. **Flat is better than nested.** Rob Pike's design philosophy: shallow structures are easier to reason about.

### Multiple Embedding

```go
type Reader struct{}
func (r Reader) Read() string { return "reading data" }

type Writer struct{}
func (w Writer) Write(data string) { fmt.Println("writing:", data) }

// Embed BOTH Reader and Writer
type ReadWriter struct {
    Reader
    Writer
}

func main() {
    rw := ReadWriter{}
    fmt.Println(rw.Read()) // "reading data" -- promoted from Reader
    rw.Write("hello")      // "writing: hello" -- promoted from Writer
}
```

### Name Conflicts in Embedding

When two embedded types have the same field or method name, Go does NOT promote either -- you must access them explicitly:

```go
type A struct{}
func (A) Hello() string { return "Hello from A" }

type B struct{}
func (B) Hello() string { return "Hello from B" }

type C struct {
    A
    B
}

func main() {
    c := C{}
    // c.Hello() -- COMPILE ERROR: ambiguous selector c.Hello
    fmt.Println(c.A.Hello()) // "Hello from A" -- must be explicit
    fmt.Println(c.B.Hello()) // "Hello from B"
}
```

---

## 4. Struct Tags

### What Are Struct Tags?

Struct tags are string metadata attached to struct fields. They are invisible at runtime unless you use **reflection** to read them. The most common use case is controlling JSON serialization.

```go
type User struct {
    ID        int    `json:"id"`
    Username  string `json:"username"`
    Email     string `json:"email,omitempty"`
    Password  string `json:"-"`                    // Never include in JSON
    CreatedAt string `json:"created_at,omitempty"`
}
```

### JSON Tags in Action

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Price       float64 `json:"price"`
    Description string  `json:"description,omitempty"` // Omit if empty
    InternalSKU string  `json:"-"`                     // Never serialize
}

func main() {
    p := Product{
        ID:          1,
        Name:        "Go Book",
        Price:       29.99,
        Description: "", // empty
        InternalSKU: "SKU-12345",
    }

    data, _ := json.MarshalIndent(p, "", "  ")
    fmt.Println(string(data))
    // Output:
    // {
    //   "id": 1,
    //   "name": "Go Book",
    //   "price": 29.99
    // }
    // Note: "description" is omitted (omitempty), "InternalSKU" is hidden (-)
}
```

### Common Tag Formats

```go
type Example struct {
    // JSON tags
    Field1 string `json:"field_1"`                  // Rename
    Field2 string `json:"field_2,omitempty"`         // Rename + omit if empty
    Field3 string `json:"-"`                         // Skip entirely

    // Database tags (used by ORMs like GORM)
    Field4 int    `gorm:"primaryKey;autoIncrement"`
    Field5 string `gorm:"column:user_name;type:varchar(100);not null"`

    // Validation tags (used by libraries like go-validator)
    Field6 string `validate:"required,email"`
    Field7 int    `validate:"min=0,max=100"`

    // Multiple tags on one field
    Field8 string `json:"email" validate:"required,email" gorm:"unique"`
}
```

### Comparison with JS/TS Decorators

Struct tags serve a similar purpose to TypeScript/JavaScript decorators, but they are simpler and purely data-driven (no function execution at definition time).

```ts
// TypeScript with decorators (class-transformer + class-validator)
import { Expose, Exclude } from 'class-transformer';
import { IsEmail, MinLength } from 'class-validator';

class User {
    @Expose({ name: 'user_id' })   // Like `json:"user_id"`
    id: number;

    @IsEmail()                      // Like `validate:"email"`
    @Expose()
    email: string;

    @Exclude()                      // Like `json:"-"`
    password: string;

    @MinLength(3)                   // Like `validate:"min=3"`
    @Expose()
    username: string;
}
```

```go
// Go equivalent with struct tags
type User struct {
    ID       int    `json:"user_id"`
    Email    string `json:"email" validate:"email"`
    Password string `json:"-"`
    Username string `json:"username" validate:"min=3"`
}
```

**Key differences:**
| Aspect | Go Struct Tags | JS/TS Decorators |
|--------|---------------|------------------|
| Syntax | String literals on fields | `@Decorator()` functions |
| When processed | At runtime via reflection | At class definition time |
| Complexity | Simple key-value strings | Full functions, can have side effects |
| Custom tags | Just use any string key | Need decorator factories |
| Type safety | None (just strings) | Fully typed |

### Reading Tags with Reflection (Advanced Preview)

```go
package main

import (
    "fmt"
    "reflect"
)

type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
}

func main() {
    t := reflect.TypeOf(User{})

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("Field: %s\n", field.Name)
        fmt.Printf("  json tag:     %s\n", field.Tag.Get("json"))
        fmt.Printf("  validate tag: %s\n", field.Tag.Get("validate"))
    }
    // Output:
    // Field: Name
    //   json tag:     name
    //   validate tag: required
    // Field: Email
    //   json tag:     email
    //   validate tag: required,email
}
```

---

## 5. Interfaces

### What Is an Interface?

An interface in Go is a set of method signatures. Any type that implements all the methods of an interface **automatically** satisfies that interface. There is no `implements` keyword.

```go
// Define an interface
type Shape interface {
    Area() float64
    Perimeter() float64
}

// Circle implements Shape (implicitly!)
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

// Rectangle implements Shape (implicitly!)
type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// This function accepts ANY Shape
func PrintShapeInfo(s Shape) {
    fmt.Printf("Area: %.2f, Perimeter: %.2f\n", s.Area(), s.Perimeter())
}

func main() {
    c := Circle{Radius: 5}
    r := Rectangle{Width: 10, Height: 3}

    PrintShapeInfo(c) // Area: 78.54, Perimeter: 31.42
    PrintShapeInfo(r) // Area: 30.00, Perimeter: 26.00
}
```

### WHY Implicit Implementation? (This Is Profound)

This is one of Go's most distinctive design decisions. Here is why it matters:

```
EXPLICIT (TypeScript, Java, C#):

    // You must DECLARE that you implement the interface
    class Circle implements Shape {  // <-- declaration required
        area(): number { ... }
        perimeter(): number { ... }
    }

    Problem: The implementor must KNOW about the interface at
    compile time. This creates a dependency from implementation
    to interface.

    Circle ----depends on----> Shape

IMPLICIT (Go):

    // Just implement the methods. That's it.
    type Circle struct{ Radius float64 }
    func (c Circle) Area() float64 { ... }
    func (c Circle) Perimeter() float64 { ... }

    // Circle knows NOTHING about Shape. Shape is defined
    // wherever it's NEEDED, not where it's implemented.

    Circle ----no dependency----> Shape
    Shape is defined by the CONSUMER, not the provider.
```

**WHY this matters in practice:**

1. **Decoupling.** You can define an interface in package A that is satisfied by a type in package B, *without package B importing package A*. The dependency arrow goes the right way.

2. **Retroactive interface satisfaction.** You can define a new interface today that is satisfied by types written years ago. The `fmt.Stringer` interface is satisfied by any type with a `String() string` method -- including types that existed before `fmt` was written.

3. **Testing.** You can write a mock that satisfies an interface without importing the package that defines the real implementation.

### Go vs TypeScript Interfaces: Deep Comparison

```ts
// TypeScript: Structural typing (similar to Go!)
interface Writer {
    write(data: string): void;
}

// This satisfies Writer without "implements"
class FileWriter {
    write(data: string): void {
        // write to file
    }
}

// TypeScript checks structurally, like Go!
function save(w: Writer) {
    w.write("data");
}

save(new FileWriter()); // Works! Structural typing.
```

```go
// Go: Also structural (implicit) typing
type Writer interface {
    Write(data string)
}

type FileWriter struct{}

func (fw FileWriter) Write(data string) {
    // write to file
}

func Save(w Writer) {
    w.Write("data")
}

func main() {
    Save(FileWriter{}) // Works! Implicit satisfaction.
}
```

**They are surprisingly similar!** TypeScript also uses structural typing. The key difference:

| Aspect | Go Interfaces | TypeScript Interfaces |
|--------|--------------|----------------------|
| Check time | Compile time | Compile time (erased at runtime) |
| Runtime cost | Interface values carry type info | Zero (erased) |
| Method only | Yes (only methods) | No (can include properties) |
| Exists at runtime | Yes (for type assertions) | No (just a TS compile-time concept) |

### Compile-Time Verification Trick

Sometimes you want to verify at compile time that a type satisfies an interface. Use a blank identifier assignment:

```go
// This line produces a compile error if *Circle doesn't satisfy Shape
var _ Shape = (*Circle)(nil)
```

This is a zero-cost assertion. The compiler checks it and discards it. You will see this pattern in production Go code everywhere.

---

## 6. The Empty Interface

### What Is `interface{}` / `any`?

The empty interface has zero methods. Since every type implements zero methods, **every type satisfies the empty interface**. It can hold a value of any type.

```go
// These are identical (since Go 1.18):
var x interface{}
var y any  // "any" is just an alias for interface{}

x = 42
x = "hello"
x = true
x = []int{1, 2, 3}
x = struct{ Name string }{"Alice"}
// x can be literally anything
```

### Comparison with JavaScript / TypeScript

```ts
// TypeScript
let x: any = 42;     // Like Go's interface{}/any
x = "hello";         // No type checking
x = true;            // Anything goes

let y: unknown = 42; // Safer -- must assert before use
// y.toString()      // ERROR: Object is of type 'unknown'
(y as number).toString() // OK after assertion
```

```go
// Go
var x any = 42

// You CANNOT use x directly as a number:
// fmt.Println(x + 1) // COMPILE ERROR: mismatched types

// You must use type assertion:
num := x.(int)        // Like TypeScript's "as int"
fmt.Println(num + 1)  // 43
```

**Key insight:** Go's `any` is actually more like TypeScript's `unknown` than TypeScript's `any`. You *cannot* do anything with an `any` value without first asserting its type. This is much safer than JS's truly dynamic typing.

### Type Assertions

```go
var val any = "hello"

// Type assertion: val.(Type)
str, ok := val.(string)
if ok {
    fmt.Println("It's a string:", str)  // "It's a string: hello"
}

// Without the ok check (panics if wrong type!)
str2 := val.(string) // Works
fmt.Println(str2)    // "hello"

// num := val.(int) // PANIC: interface conversion: interface {} is string, not int

// ALWAYS use the two-value form in production code
num, ok := val.(int)
if !ok {
    fmt.Println("Not an int") // "Not an int"
}
```

### Type Switches

When you need to handle multiple possible types, use a type switch:

```go
func describe(val any) string {
    switch v := val.(type) {
    case int:
        return fmt.Sprintf("integer: %d", v)
    case string:
        return fmt.Sprintf("string: %q (len=%d)", v, len(v))
    case bool:
        if v {
            return "boolean: true"
        }
        return "boolean: false"
    case []int:
        return fmt.Sprintf("int slice with %d elements", len(v))
    case nil:
        return "nil value"
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

func main() {
    fmt.Println(describe(42))           // "integer: 42"
    fmt.Println(describe("hello"))      // "string: \"hello\" (len=5)"
    fmt.Println(describe(true))         // "boolean: true"
    fmt.Println(describe([]int{1,2}))   // "int slice with 2 elements"
    fmt.Println(describe(nil))          // "nil value"
    fmt.Println(describe(3.14))         // "unknown type: float64"
}
```

**JS equivalent:**
```js
function describe(val) {
    if (typeof val === 'number')  return `number: ${val}`;
    if (typeof val === 'string')  return `string: "${val}" (len=${val.length})`;
    if (typeof val === 'boolean') return `boolean: ${val}`;
    if (Array.isArray(val))       return `array with ${val.length} elements`;
    if (val === null)             return 'null value';
    return `unknown type: ${typeof val}`;
}
```

### When to Use `any` (and When NOT To)

```
+---------------------------------------------------------------+
|  GOOD uses of any / interface{}:                              |
|                                                               |
|  - JSON parsing (json.Unmarshal into map[string]any)          |
|  - Generic data stores / caches                               |
|  - Logging / debugging functions                              |
|  - Interfacing with reflection-heavy libraries                |
|                                                               |
+---------------------------------------------------------------+
|  BAD uses of any / interface{} (use generics instead):        |
|                                                               |
|  - Generic data structures (use Go 1.18+ generics)            |
|  - Function parameters that should accept "several types"     |
|  - Avoiding writing proper interfaces                         |
|  - "I don't know the type" -- figure it out!                  |
|                                                               |
+---------------------------------------------------------------+
```

Since Go 1.18, **generics** handle many cases where `interface{}` was previously the only option. Prefer generics when you want type safety with flexibility.

---

## 7. Interface Composition

### Combining Interfaces

Just as structs can embed other structs, interfaces can embed other interfaces:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Composed interfaces -- embed smaller interfaces
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

This is exactly how the standard library's `io` package works:

```
                io.Reader
                +-------+
                | Read  |
                +---+---+
                    |
    +---------------+---------------+
    |                               |
io.ReadCloser              io.ReadWriter
+----------+               +----------+
| Reader   |               | Reader   |
| Closer   |               | Writer   |
+----------+               +----------+
                                |
                    +-----------+-----------+
                    |                       |
           io.ReadWriteCloser      io.ReadWriteSeeker
           +--------------+        +---------------+
           | Reader       |        | Reader        |
           | Writer       |        | Writer        |
           | Closer       |        | Seeker        |
           +--------------+        +---------------+
```

### WHY Small Interfaces Are Idiomatic Go

The Go proverb says:

> **"The bigger the interface, the weaker the abstraction."** -- Rob Pike

```
SMALL INTERFACES (Go idiom):

    type Reader interface {
        Read(p []byte) (n int, err error)
    }

    - Easy to implement (just one method)
    - Easy to test (mock one method)
    - Easy to compose (combine into bigger interfaces)
    - Maximum flexibility

BIG INTERFACES (anti-pattern):

    type Repository interface {
        Create(item Item) error
        Read(id int) (Item, error)
        Update(item Item) error
        Delete(id int) error
        List() ([]Item, error)
        Search(query string) ([]Item, error)
        Count() (int, error)
        Migrate() error
        Backup() error
    }

    - Hard to implement (9 methods)
    - Hard to mock in tests
    - Hard to extend without breaking implementors
    - Most callers only need 1-2 methods
```

**The Go way:** Define interfaces where they are *consumed*, not where they are *produced*. Keep them small.

```go
// BAD: One big interface in the "repository" package
package repository

type UserRepository interface {
    Create(u User) error
    GetByID(id int) (User, error)
    Update(u User) error
    Delete(id int) error
    List() ([]User, error)
}

// GOOD: Small interfaces defined where they are USED
package handler

// Only needs the methods it actually calls
type UserGetter interface {
    GetByID(id int) (User, error)
}

type UserCreator interface {
    Create(u User) error
}

func HandleGetUser(getter UserGetter, id int) (User, error) {
    return getter.GetByID(id)  // Only depends on GetByID
}
```

### Comparison with JS/TS

```ts
// TypeScript: You'd typically define one big interface
interface UserRepository {
    create(user: User): Promise<User>;
    getById(id: number): Promise<User>;
    update(user: User): Promise<User>;
    delete(id: number): Promise<void>;
    list(): Promise<User[]>;
}

// To test, you must mock ALL 5 methods (or use Partial<UserRepository>)

// Go's approach: Define only what you need
// type UserGetter interface { GetByID(id int) (User, error) }
// Mock: just implement GetByID. Done.
```

---

## 8. Common Standard Library Interfaces

Go's standard library is built on a small set of powerful interfaces. Understanding these is essential.

### 8.1 fmt.Stringer

```go
// Defined in the fmt package
type Stringer interface {
    String() string
}
```

Any type with a `String() string` method is automatically used by `fmt.Println`, `fmt.Sprintf`, etc.

```go
type Color struct {
    R, G, B uint8
}

func (c Color) String() string {
    return fmt.Sprintf("#%02x%02x%02x", c.R, c.G, c.B)
}

func main() {
    red := Color{255, 0, 0}
    fmt.Println(red) // "#ff0000" -- String() is called automatically!
}
```

**JS equivalent:** `toString()` method on objects.
```js
class Color {
    constructor(r, g, b) { this.r = r; this.g = g; this.b = b; }
    toString() {
        return `#${this.r.toString(16).padStart(2,'0')}${this.g.toString(16).padStart(2,'0')}${this.b.toString(16).padStart(2,'0')}`;
    }
}
console.log(`${new Color(255, 0, 0)}`); // "#ff0000"
```

### 8.2 error

```go
// Defined in the builtin package
type error interface {
    Error() string
}
```

The `error` interface is the foundation of Go's error handling. Any type with an `Error() string` method is an error.

```go
// Custom error type
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
            Message: "unrealistically large",
        }
    }
    return nil // nil means "no error"
}

func main() {
    err := validateAge(-5)
    if err != nil {
        fmt.Println(err) // "validation failed on age: must be non-negative"

        // Type assertion to get the specific error type
        if ve, ok := err.(*ValidationError); ok {
            fmt.Println("Field:", ve.Field) // "Field: age"
        }
    }
}
```

### 8.3 io.Reader and io.Writer

These are the most important interfaces in Go. An enormous number of standard library functions accept or return these.

```go
// io.Reader -- anything you can read bytes from
type Reader interface {
    Read(p []byte) (n int, err error)
}

// io.Writer -- anything you can write bytes to
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Types that implement `io.Reader`: files, network connections, HTTP request bodies, gzip decoders, base64 decoders, strings (`strings.NewReader`), byte buffers, and many more.

Types that implement `io.Writer`: files, network connections, HTTP response writers, gzip encoders, hash functions, byte buffers, `os.Stdout`, and many more.

```go
package main

import (
    "fmt"
    "io"
    "os"
    "strings"
)

// This function works with ANY io.Reader -- file, network, string, etc.
func countBytes(r io.Reader) (int, error) {
    buf := make([]byte, 1024)
    total := 0
    for {
        n, err := r.Read(buf)
        total += n
        if err == io.EOF {
            break
        }
        if err != nil {
            return total, err
        }
    }
    return total, nil
}

func main() {
    // Read from a string
    r1 := strings.NewReader("Hello, World!")
    n1, _ := countBytes(r1)
    fmt.Println("String bytes:", n1) // 13

    // Read from stdin (or a file, network connection, etc.)
    // Same function, different source!
    // n2, _ := countBytes(os.Stdin)

    // Write to stdout (an io.Writer)
    io.WriteString(os.Stdout, "Hello via io.Writer!\n")
}
```

**WHY this pattern is powerful:**
```
                     +----> File
                     |
    countBytes() --->+----> Network Connection
    (accepts         |
     io.Reader)      +----> HTTP Body
                     |
                     +----> Gzip Stream
                     |
                     +----> String
                     |
                     +----> Your Custom Type

    ONE function handles ALL of these. That is the power
    of programming to interfaces.
```

### 8.4 sort.Interface

```go
// Defined in the sort package
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

```go
type ByAge []User

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
    users := []User{
        {Name: "Charlie", Age: 30},
        {Name: "Alice", Age: 25},
        {Name: "Bob", Age: 28},
    }

    sort.Sort(ByAge(users))
    // users is now sorted by age: Alice(25), Bob(28), Charlie(30)

    // Since Go 1.8, you can also use sort.Slice (simpler):
    sort.Slice(users, func(i, j int) bool {
        return users[i].Age < users[j].Age
    })
}
```

### Summary of Key Interfaces

```
+-------------------+--------------------+-----------------------------+
| Interface         | Methods            | Purpose                     |
+-------------------+--------------------+-----------------------------+
| fmt.Stringer      | String() string    | Custom string representation|
| error             | Error() string     | Error handling              |
| io.Reader         | Read([]byte)       | Read bytes from a source    |
| io.Writer         | Write([]byte)      | Write bytes to a destination|
| io.Closer         | Close() error      | Release resources           |
| sort.Interface    | Len, Less, Swap    | Custom sorting              |
| json.Marshaler    | MarshalJSON()      | Custom JSON encoding        |
| json.Unmarshaler  | UnmarshalJSON()    | Custom JSON decoding        |
| http.Handler      | ServeHTTP(w,r)     | Handle HTTP requests        |
+-------------------+--------------------+-----------------------------+
```

---

## 9. Composition vs Inheritance

### The Deep Philosophical Comparison

This section ties everything together. It is the "why" behind Go's entire type system.

### The Inheritance Mental Model

```
                    Vehicle
                   /       \
                Car         Truck
               /   \           \
          Sedan   SUV      PickupTruck
            |
       LuxurySedan

    "IS-A" relationships:
    - LuxurySedan IS-A Sedan IS-A Car IS-A Vehicle
    - PickupTruck IS-A Truck IS-A Vehicle

    Problems:
    1. Where does ElectricCar go? Under Car? New branch?
    2. What about an ElectricTruck? Multiple inheritance?
    3. Adding a feature to Vehicle affects EVERYTHING below it.
    4. "God object" at the top that does too much.
```

### The Composition Mental Model

```
    type Engine interface { Start(); Stop() }
    type Electric interface { ChargeBattery() }
    type GasPowered interface { Refuel() }
    type FourWheelDrive interface { EngageAWD() }

    +------------------+
    |   ElectricCar    |
    |                  |
    |  +----------+    |    Has Engine behavior
    |  | Battery  |    |    Has Electric behavior
    |  +----------+    |    Has Chassis
    |  +----------+    |
    |  | Chassis  |    |
    |  +----------+    |
    +------------------+

    +------------------+
    |  ElectricTruck   |
    |                  |
    |  +----------+    |    Has Engine behavior
    |  | Battery  |    |    Has Electric behavior
    |  +----------+    |    Has Chassis
    |  +----------+    |    Has FourWheelDrive behavior
    |  | Chassis  |    |
    |  +----------+    |
    |  +----------+    |
    |  | AWDUnit  |    |
    |  +----------+    |
    +------------------+

    No hierarchy. Mix and match capabilities.
    Adding ElectricTruck doesn't require restructuring anything.
```

### Real-World Example: Building a Notification System

**The inheritance approach (JS/TS):**

```ts
// JavaScript: Inheritance-based notification system
abstract class Notification {
    abstract send(message: string): Promise<void>;

    async sendWithRetry(message: string, retries: number): Promise<void> {
        for (let i = 0; i < retries; i++) {
            try {
                await this.send(message);
                return;
            } catch (e) {
                if (i === retries - 1) throw e;
            }
        }
    }
}

class EmailNotification extends Notification {
    constructor(private email: string) { super(); }
    async send(message: string): Promise<void> {
        // send email
    }
}

class SMSNotification extends Notification {
    constructor(private phone: string) { super(); }
    async send(message: string): Promise<void> {
        // send SMS
    }
}

// Problem: What if we want to add logging? Rate limiting?
// We'd need to modify the base class or create more subclasses.
// SlackNotificationWithLogging extends SlackNotification?
// SMSNotificationWithRateLimiting extends SMSNotification?
// The hierarchy explodes.
```

**The composition approach (Go):**

```go
// Go: Composition-based notification system

// Small, focused interface
type Notifier interface {
    Send(message string) error
}

// Concrete implementations
type EmailNotifier struct {
    Email string
}

func (e *EmailNotifier) Send(message string) error {
    fmt.Printf("Sending email to %s: %s\n", e.Email, message)
    return nil
}

type SMSNotifier struct {
    Phone string
}

func (s *SMSNotifier) Send(message string) error {
    fmt.Printf("Sending SMS to %s: %s\n", s.Phone, message)
    return nil
}

type SlackNotifier struct {
    WebhookURL string
}

func (s *SlackNotifier) Send(message string) error {
    fmt.Printf("Sending Slack message to %s: %s\n", s.WebhookURL, message)
    return nil
}

// Decorators via composition (not inheritance!)

// RetryNotifier wraps any Notifier with retry logic
type RetryNotifier struct {
    Wrapped    Notifier
    MaxRetries int
}

func (r *RetryNotifier) Send(message string) error {
    var err error
    for i := 0; i <= r.MaxRetries; i++ {
        err = r.Wrapped.Send(message)
        if err == nil {
            return nil
        }
        fmt.Printf("Retry %d/%d...\n", i+1, r.MaxRetries)
    }
    return fmt.Errorf("failed after %d retries: %w", r.MaxRetries, err)
}

// LoggingNotifier wraps any Notifier with logging
type LoggingNotifier struct {
    Wrapped Notifier
    Logger  *log.Logger
}

func (l *LoggingNotifier) Send(message string) error {
    l.Logger.Printf("Sending notification: %s", message)
    err := l.Wrapped.Send(message)
    if err != nil {
        l.Logger.Printf("Notification failed: %v", err)
    } else {
        l.Logger.Printf("Notification sent successfully")
    }
    return err
}

// RateLimitedNotifier wraps any Notifier with rate limiting
type RateLimitedNotifier struct {
    Wrapped     Notifier
    MinInterval time.Duration
    lastSent    time.Time
}

func (rl *RateLimitedNotifier) Send(message string) error {
    if time.Since(rl.lastSent) < rl.MinInterval {
        return fmt.Errorf("rate limited: please wait %v", rl.MinInterval-time.Since(rl.lastSent))
    }
    rl.lastSent = time.Now()
    return rl.Wrapped.Send(message)
}

// Now compose them freely!
func main() {
    // Email with retry and logging
    notifier := &LoggingNotifier{
        Wrapped: &RetryNotifier{
            Wrapped:    &EmailNotifier{Email: "alice@example.com"},
            MaxRetries: 3,
        },
        Logger: log.Default(),
    }

    // SMS with rate limiting
    smsNotifier := &RateLimitedNotifier{
        Wrapped:     &SMSNotifier{Phone: "+1234567890"},
        MinInterval: 10 * time.Second,
    }

    // Send to multiple notifiers
    notifiers := []Notifier{notifier, smsNotifier}
    for _, n := range notifiers {
        n.Send("Server is on fire!")
    }
}
```

```
Composition chain (like wrapping layers of an onion):

    +------------------------------------------+
    |  LoggingNotifier                         |
    |                                          |
    |  +------------------------------------+  |
    |  |  RetryNotifier                     |  |
    |  |                                    |  |
    |  |  +------------------------------+  |  |
    |  |  |  EmailNotifier               |  |  |
    |  |  |  (actually sends the email)  |  |  |
    |  |  +------------------------------+  |  |
    |  |  retries if inner fails            |  |
    |  +------------------------------------+  |
    |  logs before and after                   |
    +------------------------------------------+

    Each layer implements Notifier.
    Each layer wraps another Notifier.
    Mix and match any combination!
```

### Side-by-Side Summary

```
+--------------------------+------------------------------+
|    JS/TS Inheritance     |     Go Composition           |
+--------------------------+------------------------------+
| class Dog extends Animal | type Dog struct { Animal }   |
| super()                  | d.Animal.Method()            |
| IS-A relationship        | HAS-A relationship           |
| Single inheritance only  | Multiple embedding allowed   |
| abstract class           | Interface (implicit)         |
| interface (explicit)     | Interface (implicit)         |
| mixins (workaround)      | Embedding (first-class)      |
| Decorator pattern needs  | Composition IS the pattern   |
|   complex setup          |                              |
| Protected visibility     | No protected -- exported or  |
|                          |   unexported only            |
| constructor/super chain  | Factory functions (NewXxx)   |
+--------------------------+------------------------------+
```

### WHY Composition Wins (The Go Team's Argument)

1. **Flexibility.** You can compose behaviors in ways inheritance cannot express without contortion (the "diamond problem", the "multiple inheritance debate").

2. **Testability.** Small interfaces are trivial to mock. Test `HandleGetUser(getter UserGetter)` by providing a struct with one method, not mocking an entire ORM.

3. **Readability.** When you see `type RetryNotifier struct { Wrapped Notifier }`, you immediately know what it does. With inheritance, you must trace up the chain.

4. **Stability.** Changing a deeply-embedded struct only affects the types that directly embed it. Changing a base class in an inheritance hierarchy can cascade through every descendant.

5. **Simplicity.** Go has fewer concepts to learn. No `virtual`, `override`, `abstract`, `protected`, `super`, `final`. Just structs, methods, and interfaces.

---

## 10. Key Takeaways

```
+================================================================+
|                                                                |
|  1. STRUCTS are Go's data containers.                          |
|     - No classes, no constructors -- just fields + methods     |
|     - Use NewXxx() factory functions as constructors           |
|                                                                |
|  2. METHODS use receivers.                                     |
|     - Pointer receivers (*T): mutation, large structs          |
|     - Value receivers (T): read-only, small structs            |
|     - When in doubt, use pointer receivers                     |
|     - Be consistent within a type                              |
|                                                                |
|  3. EMBEDDING is composition, not inheritance.                 |
|     - Promoted fields and methods feel like inheritance        |
|     - But it is HAS-A, not IS-A                                |
|     - You can embed multiple types                             |
|                                                                |
|  4. STRUCT TAGS are metadata strings on fields.                |
|     - Most common: json:"name,omitempty"                       |
|     - Read via reflection at runtime                           |
|     - Like a simpler version of decorators                     |
|                                                                |
|  5. INTERFACES are implicit.                                   |
|     - No "implements" keyword                                  |
|     - Define interfaces where they are CONSUMED                |
|     - Keep them SMALL (1-3 methods ideally)                    |
|                                                                |
|  6. THE EMPTY INTERFACE (any) holds any value.                 |
|     - Requires type assertion or type switch to use            |
|     - Prefer generics (Go 1.18+) when possible                |
|                                                                |
|  7. INTERFACE COMPOSITION builds big from small.               |
|     - io.ReadWriteCloser = Reader + Writer + Closer            |
|     - Small interfaces compose better                          |
|                                                                |
|  8. STANDARD LIBRARY INTERFACES are your vocabulary.           |
|     - Learn: Stringer, error, io.Reader, io.Writer             |
|     - Implement them to plug into the ecosystem                |
|                                                                |
|  9. COMPOSITION > INHERITANCE.                                 |
|     - More flexible, testable, maintainable                    |
|     - Go made this a language-level decision, not a guideline  |
|                                                                |
+================================================================+
```

---

## 11. Practice Exercises

### Exercise 1: Shape Calculator (Structs + Interfaces)

Define a `Shape` interface with `Area() float64` and `Perimeter() float64`. Implement it for `Circle`, `Rectangle`, and `Triangle`. Write a function `PrintShapeReport(shapes []Shape)` that prints the area and perimeter of each shape.

**Bonus:** Implement `fmt.Stringer` for each shape.

### Exercise 2: Custom Error Types

Create a `ValidationError` struct with fields `Field`, `Value`, and `Message`. Implement the `error` interface. Write a function `ValidateUser(name, email string, age int)` that returns a slice of `ValidationError` values when validation fails (name required, email must contain "@", age must be 0-150).

### Exercise 3: Composable Middleware

Define a `Handler` interface with `Handle(request string) string`. Implement:
- `EchoHandler` -- returns the request as-is
- `UppercaseHandler` -- wraps any Handler and uppercases the result
- `PrefixHandler` -- wraps any Handler and adds a prefix to the result
- `LoggingHandler` -- wraps any Handler and prints before/after

Compose them: `LoggingHandler(PrefixHandler("Response: ", UppercaseHandler(EchoHandler{})))`.

### Exercise 4: Struct Embedding Challenge

Create an embedded struct hierarchy:
- `Address` with `Street`, `City`, `State`, `Zip` and a `Format() string` method
- `Person` with `Name`, `Age`, embedded `Address`
- `Employee` with `EmployeeID`, `Department`, embedded `Person`

Create an `Employee` and access `Street` directly (via promotion).

### Exercise 5: JSON Serialization with Tags

Define a `BlogPost` struct with appropriate JSON tags:
- `ID` maps to `"id"`
- `Title` maps to `"title"`
- `Content` maps to `"content"`
- `AuthorEmail` maps to `"author_email"` and omits if empty
- `DraftNotes` is never serialized (internal use only)

Write a program that marshals a `BlogPost` to JSON and unmarshals it back.

### Exercise 6: Interface Segregation

You have a `Database` struct with methods: `Get(id int)`, `Put(id int, val string)`, `Delete(id int)`, `List()`, and `Close()`. Define the *smallest possible interfaces* for:
- A function that only reads by ID
- A function that reads and writes
- A function that manages the lifecycle (close)

This exercise teaches the Go idiom of defining small interfaces at the consumer.

### Exercise 7: Implement io.Writer

Create a `WordCounter` struct that implements `io.Writer`. Instead of writing bytes somewhere, it counts the number of words written. Then use `fmt.Fprintf(yourCounter, "hello world how are you")` and verify the count is 5.

**Hint:** `io.Writer` requires `Write(p []byte) (n int, err error)`.

---

## Quick Reference Card

```
STRUCT DEFINITION:
    type Name struct {
        Field Type
        Field Type `tag:"value"`
    }

STRUCT CREATION:
    v := Name{Field: value}      // value
    p := &Name{Field: value}     // pointer
    v := Name{}                  // zero value
    p := new(Name)               // pointer to zero value

METHODS:
    func (r Type) Method() {}    // value receiver
    func (r *Type) Method() {}   // pointer receiver

EMBEDDING:
    type Outer struct {
        Inner                    // embedded (promoted fields/methods)
    }

INTERFACES:
    type Name interface {
        Method(args) returns
    }

TYPE ASSERTION:
    val, ok := iface.(ConcreteType)

TYPE SWITCH:
    switch v := iface.(type) {
    case Type1: ...
    case Type2: ...
    }

INTERFACE COMPOSITION:
    type ReadWriter interface {
        Reader
        Writer
    }

COMPILE-TIME CHECK:
    var _ Interface = (*Type)(nil)
```

---

**Next Chapter:** [Chapter 7 -- Error Handling] -- Go's explicit error handling philosophy, the `error` interface in depth, `errors.Is`, `errors.As`, wrapping errors, and why Go rejected exceptions.
