# Chapter 18: Reflection & Code Generation

## Why Does a Statically Typed Language Need Reflection?

Go is designed around simplicity and compile-time safety. The type system catches errors before code ever runs. So why would Go include a mechanism -- reflection -- that deliberately sidesteps compile-time guarantees to inspect and manipulate types at runtime?

The answer is pragmatic: **some problems cannot be solved at compile time**. Consider `encoding/json`. The standard library must marshal *any* struct into JSON and unmarshal JSON into *any* struct, but the library authors cannot know your struct types in advance. They need a way to ask, at runtime, "what fields does this struct have? what are their types? what are their JSON tags?" That mechanism is reflection.

Reflection powers a surprising amount of Go's ecosystem:

- **`encoding/json`, `encoding/xml`, `encoding/gob`** -- serialization and deserialization of arbitrary types
- **`database/sql`** -- scanning query results into struct fields
- **`text/template` and `html/template`** -- accessing struct fields and calling methods by name
- **`fmt.Printf`** -- the `%v` verb must print *any* value
- **ORM libraries** (GORM, Ent) -- mapping structs to database tables
- **Dependency injection** (Wire, dig) -- wiring up types at runtime
- **Validation libraries** -- reading struct tags to enforce constraints

The core tension: Go's philosophy prefers compile-time safety and explicit code, but some essential patterns require runtime type information. Reflection is Go's carefully controlled escape hatch.

> **The Rule of Thumb:** If you can solve it without reflection, do so. But when you truly need runtime type inspection, the `reflect` package gives you the tools.

### How This Differs from Node.js

In JavaScript, reflection is not a special feature -- it is the default mode of operation. Every object can be inspected at runtime. You can enumerate properties with `Object.keys()`, check types with `typeof`, and add/remove properties freely. JavaScript *never* has compile-time type information (even TypeScript erases types before runtime).

```javascript
// Node.js -- runtime inspection is natural and free
const user = { name: "Alice", age: 30 };
Object.keys(user);          // ["name", "age"]
typeof user.name;            // "string"
user.email = "a@b.com";     // add property at runtime -- no problem
```

Go's reflection is the *opposite* philosophy: types are fixed at compile time, and reflection is an explicit, costly opt-in to inspect them at runtime.

| Aspect | Go | Node.js |
|--------|-----|---------|
| Default mode | Static types, no runtime inspection | Dynamic types, everything inspectable |
| Reflection cost | Significant overhead (allocations, indirection) | Already paying the cost of dynamic dispatch |
| When needed | Serialization, generic algorithms, metaprogramming | Rarely needed explicitly -- language is already dynamic |
| API | `reflect` package | `Reflect` API, `Proxy`, `Object.*` methods |

---

## 1. The `reflect` Package: Two Core Types

Everything in Go's reflection revolves around two types:

- **`reflect.Type`** -- describes *what* a value is (its type, fields, methods, tags)
- **`reflect.Value`** -- holds the *actual data* and lets you read or modify it

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    x := 42
    name := "Alice"
    scores := []int{90, 85, 92}

    // reflect.TypeOf() returns reflect.Type
    fmt.Println(reflect.TypeOf(x))       // int
    fmt.Println(reflect.TypeOf(name))    // string
    fmt.Println(reflect.TypeOf(scores))  // []int

    // reflect.ValueOf() returns reflect.Value
    fmt.Println(reflect.ValueOf(x))      // 42
    fmt.Println(reflect.ValueOf(name))   // Alice
    fmt.Println(reflect.ValueOf(scores)) // [90 85 92]
}
```

### Kind vs Type

This distinction trips up many beginners. A `Type` is the *specific, named* type. A `Kind` is the *underlying category*.

```go
type UserID int64
type Config struct {
    Debug bool
}

var id UserID = 42
var cfg Config

fmt.Println(reflect.TypeOf(id))         // main.UserID     (the specific type)
fmt.Println(reflect.TypeOf(id).Kind())  // int64            (the underlying kind)

fmt.Println(reflect.TypeOf(cfg))        // main.Config
fmt.Println(reflect.TypeOf(cfg).Kind()) // struct
```

The `reflect.Kind` constants include: `Bool`, `Int`, `Int8`, `Int16`, `Int32`, `Int64`, `Uint`, `Float32`, `Float64`, `Complex64`, `Complex128`, `Array`, `Chan`, `Func`, `Interface`, `Map`, `Pointer`, `Slice`, `String`, `Struct`, `UnsafePointer`.

**Why this matters:** When you write generic reflection code, you usually switch on `Kind`, not `Type`. You care whether something is "a struct" or "a slice," not whether it is specifically `Config` or `UserID`.

```go
func describe(i interface{}) {
    t := reflect.TypeOf(i)
    v := reflect.ValueOf(i)

    fmt.Printf("Type: %s\n", t)
    fmt.Printf("Kind: %s\n", t.Kind())
    fmt.Printf("Value: %v\n", v)

    switch t.Kind() {
    case reflect.Struct:
        fmt.Printf("  Fields: %d\n", t.NumField())
    case reflect.Slice:
        fmt.Printf("  Length: %d, Cap: %d\n", v.Len(), v.Cap())
    case reflect.Map:
        fmt.Printf("  Keys: %v\n", v.MapKeys())
    case reflect.Ptr:
        fmt.Printf("  Points to: %s\n", t.Elem())
    }
}
```

### Node.js Comparison: Kind vs Type

In JavaScript, `typeof` is roughly analogous to `Kind` -- it tells you the broad category:

```javascript
// Node.js
typeof 42;            // "number" (kind-like)
typeof "hello";       // "string"
typeof {};            // "object"  -- but this is too coarse
typeof [];            // "object"  -- arrays and objects are the same "kind"

// For more precision, you need instanceof or constructor checks
[] instanceof Array;  // true
({}).constructor.name; // "Object"
```

Go's `Kind` is far more granular than JavaScript's `typeof` (Go distinguishes `int8` from `int64`, `array` from `slice`), and Go's `Type` gives you the exact named type, which JavaScript simply does not have at runtime.

---

## 2. Inspecting Types in Depth

### Struct Fields

Structs are the most commonly reflected type. You can iterate over fields, read their names, types, and tags.

```go
type User struct {
    ID        int       `json:"id" db:"user_id"`
    Name      string    `json:"name" validate:"required"`
    Email     string    `json:"email" validate:"required,email"`
    IsAdmin   bool      `json:"is_admin"`
    createdAt time.Time // unexported -- reflection can see it, but not set it
}

func inspectStruct(i interface{}) {
    t := reflect.TypeOf(i)

    // If a pointer was passed, get the element type
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    if t.Kind() != reflect.Struct {
        fmt.Println("Not a struct")
        return
    }

    fmt.Printf("Struct: %s\n", t.Name())
    fmt.Printf("Package: %s\n", t.PkgPath())
    fmt.Printf("Fields: %d\n\n", t.NumField())

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("  Field %d:\n", i)
        fmt.Printf("    Name:      %s\n", field.Name)
        fmt.Printf("    Type:      %s\n", field.Type)
        fmt.Printf("    Kind:      %s\n", field.Type.Kind())
        fmt.Printf("    Exported:  %t\n", field.IsExported())
        fmt.Printf("    Tag:       %s\n", field.Tag)
        fmt.Printf("    json tag:  %s\n", field.Tag.Get("json"))
        fmt.Printf("    db tag:    %s\n", field.Tag.Get("db"))
        fmt.Println()
    }
}

func main() {
    inspectStruct(User{})
}
```

Output:

```
Struct: User
Package: main
Fields: 5

  Field 0:
    Name:      ID
    Type:      int
    Kind:      int
    Exported:  true
    Tag:       json:"id" db:"user_id"
    json tag:  id
    db tag:    user_id

  Field 1:
    Name:      Name
    Type:      string
    Kind:      string
    Exported:  true
    Tag:       json:"name" validate:"required"
    json tag:  name
    db tag:

  ...
```

### Methods

You can also enumerate methods on a type:

```go
type Greeter struct {
    Name string
}

func (g Greeter) Hello() string {
    return "Hello, " + g.Name
}

func (g Greeter) Goodbye(farewell string) string {
    return farewell + ", " + g.Name
}

func (g *Greeter) SetName(name string) {
    g.Name = name
}

func inspectMethods(i interface{}) {
    t := reflect.TypeOf(i)

    fmt.Printf("Methods on %s:\n", t)
    for i := 0; i < t.NumMethod(); i++ {
        method := t.Method(i)
        fmt.Printf("  %s: %s\n", method.Name, method.Type)
    }

    // Pointer methods include value receiver methods too
    if t.Kind() != reflect.Ptr {
        pt := reflect.PointerTo(t)
        fmt.Printf("\nMethods on *%s:\n", t)
        for i := 0; i < pt.NumMethod(); i++ {
            method := pt.Method(i)
            fmt.Printf("  %s: %s\n", method.Name, method.Type)
        }
    }
}

func main() {
    g := Greeter{Name: "World"}
    inspectMethods(g)
}
```

Output:

```
Methods on main.Greeter:
  Goodbye: func(main.Greeter, string) string
  Hello: func(main.Greeter) string

Methods on *main.Greeter:
  Goodbye: func(*main.Greeter, string) string
  Hello: func(*main.Greeter) string
  SetName: func(*main.Greeter, string)
```

Notice that pointer receiver methods (`SetName`) only appear in the pointer method set.

### Embedded Types and Interface Checking

```go
type Animal interface {
    Speak() string
}

type Base struct {
    ID int
}

type Dog struct {
    Base         // embedded
    Name  string
    Breed string
}

func (d Dog) Speak() string {
    return "Woof!"
}

func main() {
    t := reflect.TypeOf(Dog{})

    // Check if type implements an interface
    animalType := reflect.TypeOf((*Animal)(nil)).Elem()
    fmt.Println("Dog implements Animal:", t.Implements(animalType)) // true

    // Check for embedded fields
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        if field.Anonymous {
            fmt.Printf("Embedded field: %s (type: %s)\n", field.Name, field.Type)
        }
    }

    // Access nested fields via FieldByName -- works through embeddings
    idField, ok := t.FieldByName("ID")
    if ok {
        fmt.Printf("ID field found at index path: %v\n", idField.Index) // [0 0]
    }
}
```

### Node.js Comparison: Inspecting Types

JavaScript has no struct concept, but you can inspect object shapes:

```javascript
// Node.js -- inspecting an object
const user = {
    id: 1,
    name: "Alice",
    email: "alice@example.com",
};

// Get all property names
Object.keys(user);                    // ["id", "name", "email"]

// Get property descriptors (closest to Go field metadata)
Object.getOwnPropertyDescriptors(user);
// {
//   id:    { value: 1, writable: true, enumerable: true, configurable: true },
//   name:  { value: "Alice", ... },
//   email: { value: "alice@example.com", ... }
// }

// Check prototype chain (closest to Go interface checking)
user instanceof Object;  // true

// Get methods from prototype
Object.getOwnPropertyNames(Object.getPrototypeOf([]));
// ["constructor", "concat", "copyWithin", "fill", "find", ...]
```

The key difference: JavaScript inspection tells you what an object *currently looks like* (it can change at any moment). Go reflection tells you what a type *is defined as* (it is fixed forever).

---

## 3. Modifying Values with Reflection

This is where reflection gets tricky. You cannot modify just any reflected value -- the value must be **addressable**.

### The Problem: Non-Addressable Values

```go
x := 42
v := reflect.ValueOf(x) // v holds a COPY of x

fmt.Println(v.CanSet()) // false -- modifying v would not affect x

// This panics:
// v.SetInt(100) // panic: reflect.Value.SetInt using unaddressable value
```

When you call `reflect.ValueOf(x)`, Go passes `x` by value. The `reflect.Value` holds a copy. Setting it would modify the copy, which is useless, so Go prevents it entirely.

### The Solution: Pass a Pointer

```go
x := 42
v := reflect.ValueOf(&x) // pass a pointer

fmt.Println(v.Kind())           // ptr
fmt.Println(v.CanSet())         // false -- the pointer itself is not what we want to set

// Dereference the pointer with Elem()
elem := v.Elem()
fmt.Println(elem.Kind())        // int
fmt.Println(elem.CanSet())      // true -- now we can modify x through the pointer

elem.SetInt(100)
fmt.Println(x) // 100 -- x has been modified!
```

The pattern is always: **pass a pointer, call `.Elem()`, then set**.

### Modifying Struct Fields

```go
type Config struct {
    Host    string
    Port    int
    Debug   bool
    secret  string // unexported
}

func modifyConfig(cfg *Config) {
    v := reflect.ValueOf(cfg).Elem() // dereference the pointer

    // Modify exported fields
    hostField := v.FieldByName("Host")
    if hostField.IsValid() && hostField.CanSet() {
        hostField.SetString("localhost")
    }

    portField := v.FieldByName("Port")
    if portField.IsValid() && portField.CanSet() {
        portField.SetInt(8080)
    }

    debugField := v.FieldByName("Debug")
    if debugField.IsValid() && debugField.CanSet() {
        debugField.SetBool(true)
    }

    // Unexported fields -- CanSet() returns false
    secretField := v.FieldByName("secret")
    fmt.Println("Can set secret:", secretField.CanSet()) // false
    // secretField.SetString("password") // would panic
}

func main() {
    cfg := Config{}
    modifyConfig(&cfg)
    fmt.Printf("%+v\n", cfg)
    // {Host:localhost Port:8080 Debug:true secret:}
}
```

### Why Addressability Matters

The "addressable" requirement exists because Go's `reflect.Value` must know it can safely write back to the original memory location. A value is addressable when it was obtained by:

1. Dereferencing a pointer (`reflect.ValueOf(&x).Elem()`)
2. Indexing a slice (`reflect.ValueOf(slice).Index(i)`)
3. Accessing a field of an addressable struct

A value is **not** addressable when it was obtained by:

1. Directly calling `reflect.ValueOf(x)` where `x` is not a pointer
2. Indexing a map (`reflect.ValueOf(m).MapIndex(key)`) -- map values are not addressable
3. Method calls that return values

```go
// Addressable examples
slice := []int{1, 2, 3}
sv := reflect.ValueOf(slice)
sv.Index(0).SetInt(99)          // OK -- slice elements are addressable
fmt.Println(slice)               // [99 2 3]

// Non-addressable: map values
m := map[string]int{"a": 1}
mv := reflect.ValueOf(m)
// mv.MapIndex(reflect.ValueOf("a")).SetInt(99) // PANIC

// For maps, use MapIndex to read and SetMapIndex to write
mv.SetMapIndex(reflect.ValueOf("a"), reflect.ValueOf(99))
fmt.Println(m) // map[a:99]
```

---

## 4. Struct Tags with Reflection

Struct tags are one of Go's most distinctive features. They are string annotations on struct fields, invisible to normal code but accessible through reflection. This is the mechanism behind JSON marshaling, database mapping, validation, and more.

### Reading Tags

```go
type Article struct {
    ID        int       `json:"id" db:"article_id" validate:"required"`
    Title     string    `json:"title" db:"title" validate:"required,min=1,max=200"`
    Body      string    `json:"body" db:"body"`
    Published bool      `json:"published" db:"is_published"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
    DeletedAt time.Time `json:"deleted_at,omitempty" db:"deleted_at"`
}

func readTags(i interface{}) {
    t := reflect.TypeOf(i)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        tag := field.Tag

        // Get individual tag values
        jsonTag := tag.Get("json")
        dbTag := tag.Get("db")
        validateTag := tag.Get("validate")

        fmt.Printf("%-10s json=%-25s db=%-15s validate=%s\n",
            field.Name, jsonTag, dbTag, validateTag)
    }
}
```

### Parsing Tag Values

Tag values can contain options separated by commas. Here is how to parse them properly:

```go
func parseJSONTag(tag string) (name string, omitempty bool, skip bool) {
    if tag == "-" {
        return "", false, true // field should be skipped
    }

    parts := strings.Split(tag, ",")
    name = parts[0]

    for _, opt := range parts[1:] {
        if opt == "omitempty" {
            omitempty = true
        }
    }

    return name, omitempty, false
}

// Usage
field, _ := reflect.TypeOf(Article{}).FieldByName("DeletedAt")
name, omitempty, skip := parseJSONTag(field.Tag.Get("json"))
fmt.Println(name, omitempty, skip) // "deleted_at" true false
```

### Building a Simple Struct Validator

Here is a practical example: a validator that reads `validate` tags and enforces constraints.

```go
package main

import (
    "fmt"
    "reflect"
    "strconv"
    "strings"
)

type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

func Validate(s interface{}) []ValidationError {
    var errors []ValidationError

    v := reflect.ValueOf(s)
    t := reflect.TypeOf(s)

    if t.Kind() == reflect.Ptr {
        v = v.Elem()
        t = t.Elem()
    }

    if t.Kind() != reflect.Struct {
        return []ValidationError{{Field: "", Message: "expected a struct"}}
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        value := v.Field(i)
        tag := field.Tag.Get("validate")

        if tag == "" || tag == "-" {
            continue
        }

        rules := strings.Split(tag, ",")
        for _, rule := range rules {
            if err := applyRule(field.Name, value, rule); err != nil {
                errors = append(errors, *err)
            }
        }
    }

    return errors
}

func applyRule(fieldName string, value reflect.Value, rule string) *ValidationError {
    parts := strings.SplitN(rule, "=", 2)
    ruleName := parts[0]
    ruleParam := ""
    if len(parts) > 1 {
        ruleParam = parts[1]
    }

    switch ruleName {
    case "required":
        if value.IsZero() {
            return &ValidationError{
                Field:   fieldName,
                Message: "is required",
            }
        }

    case "min":
        minVal, _ := strconv.Atoi(ruleParam)
        switch value.Kind() {
        case reflect.String:
            if value.Len() < minVal {
                return &ValidationError{
                    Field:   fieldName,
                    Message: fmt.Sprintf("must be at least %d characters", minVal),
                }
            }
        case reflect.Int, reflect.Int64:
            if value.Int() < int64(minVal) {
                return &ValidationError{
                    Field:   fieldName,
                    Message: fmt.Sprintf("must be at least %d", minVal),
                }
            }
        }

    case "max":
        maxVal, _ := strconv.Atoi(ruleParam)
        switch value.Kind() {
        case reflect.String:
            if value.Len() > maxVal {
                return &ValidationError{
                    Field:   fieldName,
                    Message: fmt.Sprintf("must be at most %d characters", maxVal),
                }
            }
        case reflect.Int, reflect.Int64:
            if value.Int() > int64(maxVal) {
                return &ValidationError{
                    Field:   fieldName,
                    Message: fmt.Sprintf("must be at most %d", maxVal),
                }
            }
        }
    }

    return nil
}

// Usage
type CreateUserRequest struct {
    Name  string `validate:"required,min=2,max=50"`
    Email string `validate:"required"`
    Age   int    `validate:"min=0,max=150"`
}

func main() {
    req := CreateUserRequest{
        Name:  "A",      // too short
        Email: "",       // required
        Age:   200,      // too high
    }

    errs := Validate(req)
    for _, err := range errs {
        fmt.Println(err)
    }
    // Name: must be at least 2 characters
    // Email: is required
    // Age: must be at most 150
}
```

### How `encoding/json` Uses Reflection Internally

When you call `json.Marshal(v)`, here is a simplified version of what happens:

```go
// Simplified illustration of what encoding/json does internally
func simpleMarshal(v interface{}) ([]byte, error) {
    val := reflect.ValueOf(v)
    typ := reflect.TypeOf(v)

    if typ.Kind() == reflect.Ptr {
        val = val.Elem()
        typ = typ.Elem()
    }

    switch typ.Kind() {
    case reflect.Struct:
        return marshalStruct(val, typ)
    case reflect.String:
        return []byte(`"` + val.String() + `"`), nil
    case reflect.Int, reflect.Int64:
        return []byte(strconv.FormatInt(val.Int(), 10)), nil
    case reflect.Bool:
        if val.Bool() {
            return []byte("true"), nil
        }
        return []byte("false"), nil
    case reflect.Slice:
        return marshalSlice(val, typ)
    default:
        return nil, fmt.Errorf("unsupported type: %s", typ.Kind())
    }
}

func marshalStruct(val reflect.Value, typ reflect.Type) ([]byte, error) {
    var buf bytes.Buffer
    buf.WriteByte('{')

    first := true
    for i := 0; i < typ.NumField(); i++ {
        field := typ.Field(i)
        fieldVal := val.Field(i)

        // Skip unexported fields
        if !field.IsExported() {
            continue
        }

        // Read JSON tag
        jsonTag := field.Tag.Get("json")
        name, omitempty, skip := parseJSONTag(jsonTag)
        if skip {
            continue
        }
        if name == "" {
            name = field.Name
        }

        // Handle omitempty
        if omitempty && fieldVal.IsZero() {
            continue
        }

        if !first {
            buf.WriteByte(',')
        }
        first = false

        // Write field name
        fmt.Fprintf(&buf, `"%s":`, name)

        // Recursively marshal the field value
        fieldBytes, err := simpleMarshal(fieldVal.Interface())
        if err != nil {
            return nil, err
        }
        buf.Write(fieldBytes)
    }

    buf.WriteByte('}')
    return buf.Bytes(), nil
}
```

This illustrates why `encoding/json` is slower than hand-written JSON serialization. Every call walks the struct via reflection, reads tags, checks types at runtime. Libraries like `easyjson` and `jsoniter` use code generation to avoid this overhead.

### Node.js Comparison: Struct Tags vs Decorators

Go struct tags serve a similar purpose to JavaScript/TypeScript decorators and metadata:

```go
// Go -- struct tags
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
}
```

```typescript
// TypeScript -- decorators (similar concept)
class User {
    @JsonProperty("name")
    @IsNotEmpty()
    name: string;

    @JsonProperty("email")
    @IsNotEmpty()
    @IsEmail()
    email: string;
}
```

The key difference: Go struct tags are plain strings parsed at runtime with reflection. TypeScript decorators are functions that execute at class definition time and can modify the class itself. Go's approach is more limited but also simpler -- tags cannot inject behavior, only provide metadata.

---

## 5. Dynamic Function Calls

Reflection lets you call functions dynamically, by name or by reference, without knowing their signatures at compile time.

### Basic Function Calls

```go
func Add(a, b int) int {
    return a + b
}

func Greet(name string) string {
    return "Hello, " + name
}

func main() {
    // Call Add via reflection
    addFn := reflect.ValueOf(Add)
    args := []reflect.Value{
        reflect.ValueOf(3),
        reflect.ValueOf(4),
    }
    results := addFn.Call(args)
    fmt.Println(results[0].Int()) // 7

    // Call Greet via reflection
    greetFn := reflect.ValueOf(Greet)
    results = greetFn.Call([]reflect.Value{reflect.ValueOf("World")})
    fmt.Println(results[0].String()) // Hello, World
}
```

### Building a Function Dispatcher

A practical pattern: routing string command names to handler functions.

```go
type CommandRegistry struct {
    commands map[string]reflect.Value
}

func NewCommandRegistry() *CommandRegistry {
    return &CommandRegistry{
        commands: make(map[string]reflect.Value),
    }
}

func (r *CommandRegistry) Register(name string, fn interface{}) error {
    v := reflect.ValueOf(fn)
    if v.Kind() != reflect.Func {
        return fmt.Errorf("%s is not a function", name)
    }
    r.commands[name] = v
    return nil
}

func (r *CommandRegistry) Execute(name string, args ...interface{}) ([]interface{}, error) {
    fn, ok := r.commands[name]
    if !ok {
        return nil, fmt.Errorf("command %q not found", name)
    }

    // Convert args to reflect.Value
    fnType := fn.Type()
    if len(args) != fnType.NumIn() {
        return nil, fmt.Errorf("command %q expects %d args, got %d",
            name, fnType.NumIn(), len(args))
    }

    in := make([]reflect.Value, len(args))
    for i, arg := range args {
        in[i] = reflect.ValueOf(arg)
        // Type check
        if in[i].Type() != fnType.In(i) {
            return nil, fmt.Errorf("arg %d: expected %s, got %s",
                i, fnType.In(i), in[i].Type())
        }
    }

    // Call the function
    results := fn.Call(in)

    // Convert results back to interface{}
    out := make([]interface{}, len(results))
    for i, r := range results {
        out[i] = r.Interface()
    }

    return out, nil
}

// Usage
func main() {
    reg := NewCommandRegistry()
    reg.Register("add", func(a, b int) int { return a + b })
    reg.Register("greet", func(name string) string { return "Hello, " + name })
    reg.Register("upper", strings.ToUpper)

    result, _ := reg.Execute("add", 10, 20)
    fmt.Println(result[0]) // 30

    result, _ = reg.Execute("greet", "Go")
    fmt.Println(result[0]) // Hello, Go

    result, _ = reg.Execute("upper", "hello world")
    fmt.Println(result[0]) // HELLO WORLD
}
```

### Calling Methods by Name

```go
type MathService struct{}

func (m MathService) Add(a, b float64) float64      { return a + b }
func (m MathService) Multiply(a, b float64) float64  { return a * b }
func (m MathService) Square(x float64) float64       { return x * x }

func callMethod(obj interface{}, methodName string, args ...interface{}) (interface{}, error) {
    v := reflect.ValueOf(obj)
    method := v.MethodByName(methodName)

    if !method.IsValid() {
        return nil, fmt.Errorf("method %q not found", methodName)
    }

    in := make([]reflect.Value, len(args))
    for i, arg := range args {
        in[i] = reflect.ValueOf(arg)
    }

    results := method.Call(in)
    if len(results) == 0 {
        return nil, nil
    }
    return results[0].Interface(), nil
}

func main() {
    svc := MathService{}

    result, _ := callMethod(svc, "Add", 3.0, 4.0)
    fmt.Println(result) // 7

    result, _ = callMethod(svc, "Square", 5.0)
    fmt.Println(result) // 25

    _, err := callMethod(svc, "Divide", 10.0, 2.0)
    fmt.Println(err) // method "Divide" not found
}
```

### Creating Functions Dynamically with `reflect.MakeFunc`

```go
// Create a generic logging wrapper for any function
func makeLogged(fn interface{}) interface{} {
    fnVal := reflect.ValueOf(fn)
    fnType := fnVal.Type()

    wrapper := reflect.MakeFunc(fnType, func(args []reflect.Value) []reflect.Value {
        // Log the call
        argStrs := make([]string, len(args))
        for i, a := range args {
            argStrs[i] = fmt.Sprintf("%v", a.Interface())
        }
        fmt.Printf("Calling %s(%s)\n", runtime.FuncForPC(fnVal.Pointer()).Name(),
            strings.Join(argStrs, ", "))

        // Call the original function
        start := time.Now()
        results := fnVal.Call(args)
        elapsed := time.Since(start)

        // Log the result
        resultStrs := make([]string, len(results))
        for i, r := range results {
            resultStrs[i] = fmt.Sprintf("%v", r.Interface())
        }
        fmt.Printf("  -> returned %s (took %s)\n",
            strings.Join(resultStrs, ", "), elapsed)

        return results
    })

    return wrapper.Interface()
}

// Usage
func main() {
    add := func(a, b int) int { return a + b }
    loggedAdd := makeLogged(add).(func(int, int) int)

    result := loggedAdd(3, 4)
    // Output:
    // Calling main.main.func1(3, 4)
    //   -> returned 7 (took 125ns)
    fmt.Println(result) // 7
}
```

---

## 6. The Three Laws of Reflection

Rob Pike defined three laws that govern Go's reflection system. Understanding these clarifies *why* reflection works the way it does.

### Law 1: Reflection goes from interface value to reflection object.

Every value passed to `reflect.TypeOf()` or `reflect.ValueOf()` is first converted to an `interface{}`. The reflection functions then extract the type and value from that interface.

```go
var x float64 = 3.14

// x is implicitly converted to interface{} when passed to ValueOf
v := reflect.ValueOf(x)

// v is a reflect.Value that holds a float64
fmt.Println(v.Type())    // float64
fmt.Println(v.Kind())    // float64
fmt.Println(v.Float())   // 3.14
```

This means reflection always starts from an interface. The interface carries the concrete type information that reflection inspects.

### Law 2: Reflection goes from reflection object to interface value.

The `Interface()` method on `reflect.Value` recovers the original value as an `interface{}`. This is the inverse of Law 1.

```go
v := reflect.ValueOf(3.14)

// Go from reflect.Value back to interface{}
i := v.Interface()

// Type-assert to get the concrete value
f := i.(float64)
fmt.Println(f) // 3.14

// Or use it directly in fmt (which accepts interface{})
fmt.Println(v.Interface()) // 3.14
```

Laws 1 and 2 form a round trip:

```
concrete value --> interface{} --> reflect.Value --> interface{} --> concrete value
```

### Law 3: To modify a reflection object, the value must be settable.

This is the law that explains the addressability requirement we discussed earlier.

```go
// This does NOT work
v := reflect.ValueOf(3.14)
fmt.Println(v.CanSet()) // false
// v.SetFloat(2.71)     // PANIC

// This DOES work -- pass a pointer, then Elem()
x := 3.14
v = reflect.ValueOf(&x).Elem()
fmt.Println(v.CanSet()) // true
v.SetFloat(2.71)
fmt.Println(x) // 2.71
```

**Why this law exists:** Think of it like passing arguments to a function. If you pass `x` by value, the function gets a copy -- modifying it has no effect. If you pass `&x`, the function can modify the original. Reflection follows the exact same principle. `reflect.ValueOf(x)` gets a copy. `reflect.ValueOf(&x).Elem()` gets access to the original.

### Summary of the Three Laws

| Law | Direction | Key Point |
|-----|-----------|-----------|
| 1 | Value -> Reflection | Reflection starts from an interface value |
| 2 | Reflection -> Value | `Interface()` converts back to an interface value |
| 3 | Modification | Only settable (addressable) values can be modified |

---

## 7. Performance Cost of Reflection

Reflection is not free. It adds overhead in every dimension: CPU time, memory allocations, and lost compiler optimizations.

### Why Reflection Is Slow

1. **Runtime type checking** -- The compiler cannot verify types at compile time. Every operation must check types at runtime.
2. **Memory allocations** -- `reflect.ValueOf()` often allocates on the heap. Interface boxing requires allocation.
3. **No inlining** -- The compiler cannot inline reflected function calls.
4. **No devirtualization** -- Each reflected operation goes through multiple layers of indirection.
5. **Cache misses** -- Reflected access patterns are less cache-friendly than direct field access.

### Benchmarks: Reflection vs Direct Code

```go
package bench

import (
    "reflect"
    "testing"
)

type Point struct {
    X float64
    Y float64
}

// Benchmark 1: Direct field access vs reflected field access
func BenchmarkDirectFieldAccess(b *testing.B) {
    p := Point{X: 1.0, Y: 2.0}
    var sum float64
    for i := 0; i < b.N; i++ {
        sum = p.X + p.Y
    }
    _ = sum
}

func BenchmarkReflectFieldAccess(b *testing.B) {
    p := Point{X: 1.0, Y: 2.0}
    v := reflect.ValueOf(p)
    var sum float64
    for i := 0; i < b.N; i++ {
        sum = v.Field(0).Float() + v.Field(1).Float()
    }
    _ = sum
}

// Benchmark 2: Direct function call vs reflected function call
func add(a, b int) int { return a + b }

func BenchmarkDirectCall(b *testing.B) {
    var result int
    for i := 0; i < b.N; i++ {
        result = add(3, 4)
    }
    _ = result
}

func BenchmarkReflectCall(b *testing.B) {
    fn := reflect.ValueOf(add)
    args := []reflect.Value{reflect.ValueOf(3), reflect.ValueOf(4)}
    var result int
    for i := 0; i < b.N; i++ {
        results := fn.Call(args)
        result = int(results[0].Int())
    }
    _ = result
}

// Benchmark 3: Direct struct creation vs reflected struct creation
func BenchmarkDirectCreate(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = Point{X: 1.0, Y: 2.0}
    }
}

func BenchmarkReflectCreate(b *testing.B) {
    t := reflect.TypeOf(Point{})
    for i := 0; i < b.N; i++ {
        v := reflect.New(t).Elem()
        v.Field(0).SetFloat(1.0)
        v.Field(1).SetFloat(2.0)
        _ = v.Interface()
    }
}
```

Typical results (your numbers will vary, but the ratios are consistent):

```
BenchmarkDirectFieldAccess-8     1000000000    0.28 ns/op    0 B/op    0 allocs/op
BenchmarkReflectFieldAccess-8     200000000    6.50 ns/op    0 B/op    0 allocs/op

BenchmarkDirectCall-8            1000000000    0.55 ns/op    0 B/op    0 allocs/op
BenchmarkReflectCall-8             10000000  145.00 ns/op   80 B/op    4 allocs/op

BenchmarkDirectCreate-8          1000000000    0.30 ns/op    0 B/op    0 allocs/op
BenchmarkReflectCreate-8           10000000  180.00 ns/op   48 B/op    2 allocs/op
```

Key observations:

- **Field access**: ~23x slower with reflection
- **Function calls**: ~260x slower with reflection (plus allocations)
- **Struct creation**: ~600x slower with reflection

These are microbenchmarks, and in real applications the overhead is amortized across other work. But in hot paths (tight loops, high-throughput servers), reflection cost adds up.

### Caching Reflected Type Information

A common optimization: reflect on a type *once* and cache the results.

```go
// Bad: reflects every time
func getFieldValue(obj interface{}, fieldName string) interface{} {
    v := reflect.ValueOf(obj)
    return v.FieldByName(fieldName).Interface()
}

// Better: cache field indices
type FieldAccessor struct {
    fieldIndex map[string][]int
}

func NewFieldAccessor(t reflect.Type) *FieldAccessor {
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }
    fa := &FieldAccessor{fieldIndex: make(map[string][]int)}
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fa.fieldIndex[field.Name] = field.Index
    }
    return fa
}

func (fa *FieldAccessor) Get(obj interface{}, fieldName string) interface{} {
    v := reflect.ValueOf(obj)
    if v.Kind() == reflect.Ptr {
        v = v.Elem()
    }
    idx, ok := fa.fieldIndex[fieldName]
    if !ok {
        return nil
    }
    return v.FieldByIndex(idx).Interface()
}

// Usage -- create accessor once, use many times
var pointAccessor = NewFieldAccessor(reflect.TypeOf(Point{}))

func main() {
    p := Point{X: 1.0, Y: 2.0}
    fmt.Println(pointAccessor.Get(p, "X")) // 1
    fmt.Println(pointAccessor.Get(p, "Y")) // 2
}
```

This pattern is used internally by `encoding/json` (which caches struct field metadata) and many ORM libraries.

### Node.js Comparison: Performance

In JavaScript, dynamic property access is the *default* mechanism, so there is no "reflection overhead" to compare against -- you are always paying the dynamic dispatch cost. JavaScript engines like V8 use hidden classes and inline caches to optimize dynamic access, making it surprisingly fast in practice:

```javascript
// Node.js -- dynamic access is how everything works
const obj = { x: 1.0, y: 2.0 };

// These are all "reflected" access by Go standards, but they're
// the normal (and fast) way to access properties in JS
obj.x;           // direct property access -- V8 optimizes this heavily
obj["x"];        // dynamic key access -- still fast thanks to inline caches
Reflect.get(obj, "x"); // explicit Reflect API -- rarely needed
```

The philosophical difference: Go pays no cost for direct access and a heavy cost for reflection. JavaScript always pays a small cost for dynamic dispatch but optimizes it well. Go's ceiling for direct code is higher, but its floor with reflection is lower.

---

## 8. `go generate`: Code Generation as an Alternative

If reflection is slow, can we get the same flexibility at compile time? Yes -- that is what code generation does. Instead of inspecting types at runtime, we inspect them *before* compiling and generate specific, optimized code for each type.

### The `//go:generate` Directive

`go generate` is a tool that scans your source files for special comments and runs the commands they specify.

```go
//go:generate stringer -type=Color

type Color int

const (
    Red Color = iota
    Green
    Blue
)
```

Running `go generate ./...` finds the `//go:generate` comment and runs `stringer -type=Color`, which produces a file `color_string.go` containing:

```go
// Code generated by "stringer -type=Color"; DO NOT EDIT.

package main

import "strconv"

func _() {
    // An "invalid array index" compiler error signifies that the constant values have changed.
    var x [1]struct{}
    _ = x[Red-0]
    _ = x[Green-1]
    _ = x[Blue-2]
}

const _Color_name = "RedGreenBlue"

var _Color_index = [...]uint8{0, 3, 8, 12}

func (i Color) String() string {
    if i < 0 || i >= Color(len(_Color_index)-1) {
        return "Color(" + strconv.FormatInt(int64(i), 10) + ")"
    }
    return _Color_name[_Color_index[i]:_Color_index[i+1]]
}
```

This generated code is:
- **Zero reflection** -- pure compile-time code
- **Fast** -- just an array lookup and a string slice
- **Type-safe** -- the compiler verifies everything
- **Automatically regenerated** when you run `go generate`

### How `go generate` Works

1. You add `//go:generate <command>` comments to your `.go` files
2. You run `go generate ./...` (or for specific packages)
3. `go generate` scans for those comments and executes the commands
4. The commands produce new `.go` files
5. You commit the generated files to version control
6. Normal `go build` compiles everything together -- no generation needed at build time

```go
// Multiple generate directives in one file
//go:generate stringer -type=Status
//go:generate mockgen -source=repository.go -destination=mock_repository.go
//go:generate protoc --go_out=. --go-grpc_out=. api.proto
```

**Important:** `go generate` is a *development-time* tool, not a build-time tool. Generated files are committed to the repository. CI/CD runs `go build`, not `go generate`.

### Node.js Comparison: Code Generation

JavaScript's build toolchain has analogous concepts:

| Go | Node.js |
|----|---------|
| `//go:generate stringer` | Babel plugin that transforms code |
| `go generate ./...` | `npm run build` with webpack/rollup |
| `protoc --go_out` | `protoc --js_out` or `grpc-tools` |
| `mockgen` | `jest --automock` or manual mock factories |
| Generated `.go` files committed to repo | Typically generated in `dist/` during build, not committed |

Key difference: In Go, generated code is *source code* that you commit and review. In JavaScript, generated code is usually build output that you `.gitignore`. Go treats code generation as part of the development process; JavaScript treats it as part of the build process.

```javascript
// Node.js -- Babel transform (compile-time code generation)
// .babelrc plugin that adds logging to every function
module.exports = function(babel) {
    const t = babel.types;
    return {
        visitor: {
            FunctionDeclaration(path) {
                const name = path.node.id.name;
                const logStatement = t.expressionStatement(
                    t.callExpression(
                        t.memberExpression(t.identifier("console"), t.identifier("log")),
                        [t.stringLiteral(`Calling ${name}`)]
                    )
                );
                path.get("body").unshiftContainer("body", logStatement);
            }
        }
    };
};
```

---

## 9. Code Generation Tools

### `stringer` -- String Representation for Constants

As shown above, generates `String()` methods for `iota` constants.

```bash
go install golang.org/x/tools/cmd/stringer@latest
```

### `mockgen` -- Mock Generation for Interfaces

Generates mock implementations for testing:

```go
// repository.go
//go:generate mockgen -source=repository.go -destination=mock_repository.go -package=main

type UserRepository interface {
    FindByID(id int) (*User, error)
    Save(user *User) error
    Delete(id int) error
}
```

After running `go generate`, you get a `mock_repository.go` with a fully functional mock that you can use in tests with `gomock`:

```go
func TestCreateUser(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockRepo := NewMockUserRepository(ctrl)
    mockRepo.EXPECT().Save(gomock.Any()).Return(nil)

    svc := NewUserService(mockRepo)
    err := svc.CreateUser("Alice", "alice@example.com")
    assert.NoError(t, err)
}
```

### `protoc` -- Protocol Buffer Code Generation

Protocol Buffers generate Go structs and gRPC service stubs from `.proto` files:

```protobuf
// api.proto
syntax = "proto3";
package api;
option go_package = "./api";

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
}

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc CreateUser(CreateUserRequest) returns (User);
}
```

```go
//go:generate protoc --go_out=. --go-grpc_out=. api.proto
```

### `ent` -- Entity Framework for Go

Ent generates a full type-safe ORM from schema definitions:

```go
// ent/schema/user.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/edge"
)

type User struct {
    ent.Schema
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").NotEmpty(),
        field.String("email").Unique(),
        field.Int("age").Positive(),
    }
}

func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("posts", Post.Type),
    }
}
```

Running `go generate ./ent` produces thousands of lines of type-safe query builder code:

```go
// Generated code -- fully type-safe, zero reflection
users, err := client.User.
    Query().
    Where(user.AgeGT(18)).
    WithPosts().
    All(ctx)
```

### Building Your Own Code Generator

You can write custom code generators using `go/ast` and `go/parser` to parse Go source files and `go/format` to produce properly formatted output.

Here is a complete example: a generator that creates constructor functions for structs.

```go
// gen_constructors.go -- run with: go run gen_constructors.go -type=User,Config
package main

import (
    "bytes"
    "flag"
    "fmt"
    "go/ast"
    "go/format"
    "go/parser"
    "go/token"
    "log"
    "os"
    "strings"
    "text/template"
)

var typeNames = flag.String("type", "", "comma-separated list of type names")

const tmpl = `// Code generated by gen_constructors; DO NOT EDIT.

package {{.Package}}

{{range .Types}}
// New{{.Name}} creates a new {{.Name}} with the given parameters.
func New{{.Name}}({{.Params}}) *{{.Name}} {
    return &{{.Name}}{
        {{.Fields}}
    }
}
{{end}}
`

type TypeInfo struct {
    Name   string
    Params string
    Fields string
}

type TemplateData struct {
    Package string
    Types   []TypeInfo
}

func main() {
    flag.Parse()
    if *typeNames == "" {
        log.Fatal("must specify -type flag")
    }

    targetTypes := strings.Split(*typeNames, ",")
    targetSet := make(map[string]bool)
    for _, t := range targetTypes {
        targetSet[strings.TrimSpace(t)] = true
    }

    // Parse the Go source files in the current directory
    fset := token.NewFileSet()
    pkgs, err := parser.ParseDir(fset, ".", nil, parser.ParseComments)
    if err != nil {
        log.Fatal(err)
    }

    var data TemplateData
    for pkgName, pkg := range pkgs {
        data.Package = pkgName

        for _, file := range pkg.Files {
            ast.Inspect(file, func(n ast.Node) bool {
                typeSpec, ok := n.(*ast.TypeSpec)
                if !ok || !targetSet[typeSpec.Name.Name] {
                    return true
                }

                structType, ok := typeSpec.Type.(*ast.StructType)
                if !ok {
                    return true
                }

                var params []string
                var fields []string

                for _, field := range structType.Fields.List {
                    for _, name := range field.Names {
                        if !name.IsExported() {
                            continue
                        }
                        typeName := fmt.Sprintf("%s", field.Type)
                        paramName := strings.ToLower(name.Name[:1]) + name.Name[1:]
                        params = append(params, fmt.Sprintf("%s %s", paramName, typeName))
                        fields = append(fields, fmt.Sprintf("%s: %s,", name.Name, paramName))
                    }
                }

                data.Types = append(data.Types, TypeInfo{
                    Name:   typeSpec.Name.Name,
                    Params: strings.Join(params, ", "),
                    Fields: strings.Join(fields, "\n\t\t"),
                })

                return true
            })
        }
    }

    // Execute the template
    t := template.Must(template.New("constructors").Parse(tmpl))
    var buf bytes.Buffer
    if err := t.Execute(&buf, data); err != nil {
        log.Fatal(err)
    }

    // Format the output
    formatted, err := format.Source(buf.Bytes())
    if err != nil {
        log.Fatal(err)
    }

    // Write to file
    outputFile := strings.ToLower(targetTypes[0]) + "_constructors.go"
    if err := os.WriteFile(outputFile, formatted, 0644); err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Generated %s\n", outputFile)
}
```

Usage in your source file:

```go
//go:generate go run gen_constructors.go -type=User,Config

type User struct {
    Name  string
    Email string
    Age   int
}

type Config struct {
    Host  string
    Port  int
    Debug bool
}
```

After `go generate`, it produces:

```go
// Code generated by gen_constructors; DO NOT EDIT.

package main

func NewUser(name string, email string, age int) *User {
    return &User{
        Name:  name,
        Email: email,
        Age:   age,
    }
}

func NewConfig(host string, port int, debug bool) *Config {
    return &Config{
        Host:  host,
        Port:  port,
        Debug: debug,
    }
}
```

---

## 10. Build Tags and Managing Generated Code

### The `// Code generated` Convention

Go has a standard convention for generated files. Tools and editors recognize this comment at the top of a file:

```go
// Code generated by <tool>; DO NOT EDIT.
```

`go generate` sets this comment automatically when tools follow the convention. The Go toolchain (including `go vet`) recognizes it to suppress certain warnings on generated code.

### Regeneration Workflow

A typical workflow for projects with code generation:

```makefile
# Makefile
.PHONY: generate build test

generate:
	go generate ./...

build: generate
	go build ./...

test: generate
	go test ./...

# Check that generated code is up-to-date (useful in CI)
check-generate:
	go generate ./...
	git diff --exit-code  # fails if generation produced changes
```

In CI, you want to ensure generated code is committed and up-to-date:

```yaml
# .github/workflows/ci.yml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-go@v5
    with:
      go-version: '1.22'
  - name: Check generated code
    run: |
      go generate ./...
      if [ -n "$(git status --porcelain)" ]; then
        echo "Generated code is out of date. Run 'go generate ./...' and commit."
        git diff
        exit 1
      fi
```

### Build Tags for Conditional Compilation

Build tags (also called build constraints) let you include or exclude files based on conditions. This is useful for generated code that should only compile on certain platforms or configurations.

```go
//go:build ignore
// +build ignore

// This file is a code generator, not part of the build.
// Run it with: go run gen.go
package main

func main() {
    // generator logic
}
```

The `//go:build ignore` tag ensures `go build` never compiles this file. You run it explicitly with `go run`.

For platform-specific generated code:

```go
//go:build linux
// +build linux

// Code generated by platform-gen; DO NOT EDIT.

package sys

const DefaultConfigPath = "/etc/myapp/config.yaml"
```

```go
//go:build darwin
// +build darwin

// Code generated by platform-gen; DO NOT EDIT.

package sys

const DefaultConfigPath = "/Library/Application Support/myapp/config.yaml"
```

### File Naming Conventions

Generated files commonly follow these patterns:

```
*_string.go       -- stringer output
*_gen.go          -- generic generated code
*_mock.go         -- mock implementations
mock_*.go         -- another mock convention
*.pb.go           -- protobuf generated code
*_enumer.go       -- enumerator generated code
zz_generated_*.go -- Kubernetes-style generated code
```

Many projects add generated files to `.gitattributes` to collapse them in diffs:

```
# .gitattributes
*_gen.go linguist-generated=true
*_string.go linguist-generated=true
*.pb.go linguist-generated=true
```

---

## 11. Reflection vs Generics (Go 1.18+)

Go 1.18 introduced type parameters (generics), which eliminated many previous use cases for reflection. Understanding when to use each is important.

### Before Generics: Reflection Was the Only Way

Before Go 1.18, writing a generic "contains" function required either:

```go
// Option 1: Separate function for each type (tedious)
func ContainsInt(slice []int, target int) bool { ... }
func ContainsString(slice []string, target string) bool { ... }

// Option 2: Reflection (slow, no type safety)
func Contains(slice interface{}, target interface{}) bool {
    sv := reflect.ValueOf(slice)
    if sv.Kind() != reflect.Slice {
        return false
    }
    for i := 0; i < sv.Len(); i++ {
        if reflect.DeepEqual(sv.Index(i).Interface(), target) {
            return true
        }
    }
    return false
}
```

### After Generics: Type-Safe and Fast

```go
// Option 3: Generics (type-safe AND fast)
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// Usage -- fully type-safe, no reflection, no overhead
Contains([]int{1, 2, 3}, 2)           // true
Contains([]string{"a", "b"}, "c")     // false
// Contains([]int{1, 2, 3}, "hello")  // compile error!
```

### When to Use Generics vs Reflection

| Scenario | Use Generics | Use Reflection |
|----------|-------------|----------------|
| Generic data structures (trees, queues, sets) | Yes | No |
| Generic algorithms (sort, filter, map, reduce) | Yes | No |
| Type-safe wrapper functions | Yes | No |
| Working with `comparable` or other constraints | Yes | No |
| Serialization of arbitrary types (JSON, XML) | No (types unknown at compile time) | Yes |
| ORM/database scanning | Limited | Yes (types unknown at compile time) |
| Reading struct tags | No (generics cannot access tags) | Yes |
| Template engines accessing fields by name | No | Yes |
| Building plugins/dynamic dispatch by name | No | Yes |
| Dependency injection | No | Yes |

### The Key Distinction

**Generics:** "I want to write code that works for *any type that satisfies a constraint*, but the type is known at compile time."

**Reflection:** "I want to write code that inspects types *that I cannot know at compile time*, examining their fields, methods, and tags at runtime."

```go
// Generics: the type is known at each call site
func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

Max(3, 4)       // T is int -- known at compile time
Max(3.14, 2.71) // T is float64 -- known at compile time

// Reflection: the type is truly unknown
func printFields(v interface{}) {
    // We have NO IDEA what type v is until we inspect it
    t := reflect.TypeOf(v)
    for i := 0; i < t.NumField(); i++ {
        fmt.Println(t.Field(i).Name)
    }
}
```

### Combining Generics and Reflection

Sometimes you use both:

```go
// Generic function that uses reflection for the parts generics cannot handle
func StructToMap[T any](s T) map[string]interface{} {
    result := make(map[string]interface{})

    v := reflect.ValueOf(s)
    t := reflect.TypeOf(s)

    if t.Kind() == reflect.Ptr {
        v = v.Elem()
        t = t.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        if !field.IsExported() {
            continue
        }

        // Use json tag for key name if available
        key := field.Tag.Get("json")
        if key == "" || key == "-" {
            key = field.Name
        }
        // Strip options like omitempty
        if idx := strings.Index(key, ","); idx != -1 {
            key = key[:idx]
        }

        result[key] = v.Field(i).Interface()
    }

    return result
}

// The generic parameter gives us type safety at the call site
// but reflection handles the field inspection at runtime
user := User{Name: "Alice", Email: "alice@example.com"}
m := StructToMap(user) // type-safe: must be called with a concrete type
```

### Node.js Comparison: Generics vs Dynamic Typing

JavaScript does not have this tension because everything is already dynamic:

```javascript
// Node.js -- no generics or reflection needed, it just works
function contains(array, target) {
    return array.includes(target);
}

contains([1, 2, 3], 2);          // true
contains(["a", "b"], "c");       // false
contains([1, 2, 3], "hello");    // false (no compile error, just false)
```

TypeScript adds generics for type safety, mirroring Go's approach:

```typescript
// TypeScript -- generics for compile-time safety
function contains<T>(array: T[], target: T): boolean {
    return array.includes(target);
}

contains([1, 2, 3], 2);          // OK
contains(["a", "b"], "c");       // OK
contains([1, 2, 3], "hello");    // Compile error!
```

But TypeScript generics are erased at runtime -- they provide no runtime type information. Go's generics actually generate specialized code, giving real performance benefits.

---

## 12. Real-World Example: Building a Simple JSON Marshaler

Let us put everything together by building a simplified JSON marshaler that demonstrates reflection in a realistic context.

```go
package main

import (
    "bytes"
    "fmt"
    "reflect"
    "strconv"
    "strings"
    "time"
)

// Marshal converts any Go value to JSON bytes.
func Marshal(v interface{}) ([]byte, error) {
    var buf bytes.Buffer
    if err := marshalValue(&buf, reflect.ValueOf(v)); err != nil {
        return nil, err
    }
    return buf.Bytes(), nil
}

func marshalValue(buf *bytes.Buffer, v reflect.Value) error {
    // Handle interface and pointer indirection
    for v.Kind() == reflect.Ptr || v.Kind() == reflect.Interface {
        if v.IsNil() {
            buf.WriteString("null")
            return nil
        }
        v = v.Elem()
    }

    switch v.Kind() {
    case reflect.String:
        buf.WriteString(strconv.Quote(v.String()))

    case reflect.Bool:
        if v.Bool() {
            buf.WriteString("true")
        } else {
            buf.WriteString("false")
        }

    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        buf.WriteString(strconv.FormatInt(v.Int(), 10))

    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        buf.WriteString(strconv.FormatUint(v.Uint(), 10))

    case reflect.Float32, reflect.Float64:
        buf.WriteString(strconv.FormatFloat(v.Float(), 'f', -1, 64))

    case reflect.Slice, reflect.Array:
        if v.Kind() == reflect.Slice && v.IsNil() {
            buf.WriteString("null")
            return nil
        }
        buf.WriteByte('[')
        for i := 0; i < v.Len(); i++ {
            if i > 0 {
                buf.WriteByte(',')
            }
            if err := marshalValue(buf, v.Index(i)); err != nil {
                return err
            }
        }
        buf.WriteByte(']')

    case reflect.Map:
        if v.IsNil() {
            buf.WriteString("null")
            return nil
        }
        buf.WriteByte('{')
        iter := v.MapRange()
        first := true
        for iter.Next() {
            if !first {
                buf.WriteByte(',')
            }
            first = false
            // Map keys must be strings for JSON
            if iter.Key().Kind() != reflect.String {
                return fmt.Errorf("map key must be string, got %s", iter.Key().Kind())
            }
            buf.WriteString(strconv.Quote(iter.Key().String()))
            buf.WriteByte(':')
            if err := marshalValue(buf, iter.Value()); err != nil {
                return err
            }
        }
        buf.WriteByte('}')

    case reflect.Struct:
        // Special case: time.Time
        if v.Type() == reflect.TypeOf(time.Time{}) {
            t := v.Interface().(time.Time)
            buf.WriteString(strconv.Quote(t.Format(time.RFC3339)))
            return nil
        }

        buf.WriteByte('{')
        t := v.Type()
        first := true
        for i := 0; i < t.NumField(); i++ {
            field := t.Field(i)
            fieldVal := v.Field(i)

            // Skip unexported fields
            if !field.IsExported() {
                continue
            }

            // Parse json tag
            tag := field.Tag.Get("json")
            if tag == "-" {
                continue
            }

            name := field.Name
            omitempty := false

            if tag != "" {
                parts := strings.Split(tag, ",")
                if parts[0] != "" {
                    name = parts[0]
                }
                for _, opt := range parts[1:] {
                    if opt == "omitempty" {
                        omitempty = true
                    }
                }
            }

            // Handle omitempty
            if omitempty && fieldVal.IsZero() {
                continue
            }

            if !first {
                buf.WriteByte(',')
            }
            first = false

            buf.WriteString(strconv.Quote(name))
            buf.WriteByte(':')
            if err := marshalValue(buf, fieldVal); err != nil {
                return err
            }
        }
        buf.WriteByte('}')

    default:
        return fmt.Errorf("unsupported type: %s", v.Kind())
    }

    return nil
}

// Test it out
type Address struct {
    Street string `json:"street"`
    City   string `json:"city"`
    State  string `json:"state"`
}

type Person struct {
    Name      string    `json:"name"`
    Age       int       `json:"age"`
    Email     string    `json:"email,omitempty"`
    IsActive  bool      `json:"is_active"`
    Address   Address   `json:"address"`
    Hobbies   []string  `json:"hobbies"`
    Scores    []int     `json:"scores,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    password  string    // unexported -- will be skipped
}

func main() {
    person := Person{
        Name:     "Alice",
        Age:      30,
        IsActive: true,
        Address: Address{
            Street: "123 Main St",
            City:   "Portland",
            State:  "OR",
        },
        Hobbies:   []string{"reading", "hiking", "coding"},
        CreatedAt: time.Date(2024, 1, 15, 10, 30, 0, 0, time.UTC),
        password:  "secret123",
    }

    data, err := Marshal(person)
    if err != nil {
        panic(err)
    }

    fmt.Println(string(data))
    // {"name":"Alice","age":30,"is_active":true,"address":{"street":"123 Main St",
    //  "city":"Portland","state":"OR"},"hobbies":["reading","hiking","coding"],
    //  "created_at":"2024-01-15T10:30:00Z"}
    //
    // Notice: "email" omitted (omitempty + zero value), "scores" omitted (omitempty),
    //         "password" omitted (unexported)
}
```

---

## 13. Advanced Pattern: reflect.DeepEqual and Its Pitfalls

`reflect.DeepEqual` is heavily used in tests to compare complex structures. Understanding its behavior is important.

```go
import "reflect"

// Basic usage
a := []int{1, 2, 3}
b := []int{1, 2, 3}
fmt.Println(reflect.DeepEqual(a, b)) // true

// Nil vs empty slice -- a common gotcha
var nilSlice []int
emptySlice := []int{}
fmt.Println(reflect.DeepEqual(nilSlice, emptySlice)) // FALSE!

// Both have length 0, but DeepEqual treats them differently
// nil slice is nil, empty slice is not nil

// Maps: same behavior
var nilMap map[string]int
emptyMap := map[string]int{}
fmt.Println(reflect.DeepEqual(nilMap, emptyMap)) // FALSE!

// Unexported fields -- DeepEqual CAN compare them
type internal struct {
    secret string
}
x := internal{secret: "a"}
y := internal{secret: "a"}
fmt.Println(reflect.DeepEqual(x, y)) // true

// Function values -- never equal (except both nil)
f1 := func() {}
f2 := func() {}
fmt.Println(reflect.DeepEqual(f1, f2)) // false (always)
```

### When to Avoid `reflect.DeepEqual`

1. **Performance-sensitive code** -- it is slow due to reflection.
2. **Nil vs empty** semantics -- it distinguishes them, which may not be what you want.
3. **Floating-point comparison** -- it uses exact equality, not approximate.
4. **Tests** -- consider `github.com/google/go-cmp/cmp` for better error messages and customizable comparison.

```go
// Using go-cmp instead of DeepEqual
import "github.com/google/go-cmp/cmp"

if diff := cmp.Diff(expected, actual); diff != "" {
    t.Errorf("mismatch (-want +got):\n%s", diff)
}
```

---

## 14. Summary: Go vs Node.js Reflection and Code Generation

| Aspect | Go | Node.js |
|--------|-----|---------|
| **Type system** | Static -- types fixed at compile time | Dynamic -- types checked at runtime |
| **Reflection need** | Explicit opt-in for runtime type inspection | Built into the language's core behavior |
| **Reflection API** | `reflect` package (`TypeOf`, `ValueOf`) | `Reflect` API, `Proxy`, `Object.*` |
| **Struct tags / metadata** | String annotations on struct fields | Decorators + `reflect-metadata` library |
| **Performance cost** | Significant (23-600x slower than direct code) | Negligible (already paying dynamic dispatch cost) |
| **Code generation** | `go generate` + tools (stringer, mockgen, protoc) | Babel transforms, webpack plugins, macros |
| **Generated code** | Committed to repo as source code | Usually build output, not committed |
| **Generics** | Type parameters (Go 1.18+), monomorphized | TypeScript generics (erased at runtime) |
| **Philosophy** | Prefer compile-time safety; reflection is escape hatch | Embrace runtime dynamism; reflection is natural |

---

## Key Takeaways

1. **Reflection is Go's runtime escape hatch** from static typing. Use it when types are genuinely unknown at compile time (serialization, ORMs, template engines), not as a substitute for proper type design.

2. **`reflect.Type` describes the shape; `reflect.Value` holds the data.** Kind is the broad category (struct, slice, int). Type is the specific named type.

3. **To modify values via reflection, pass a pointer and call `.Elem()`.** This is not arbitrary -- it mirrors Go's value/pointer semantics.

4. **Struct tags are Go's metadata system**, read via `field.Tag.Get("key")`. They power JSON marshaling, database mapping, validation, and more.

5. **Reflection is slow**: 23x for field access, 260x for function calls. Cache reflected type information, and avoid reflection in hot loops.

6. **`go generate` produces compile-time code as an alternative to runtime reflection.** Generated code is faster, type-safe, and easier to debug.

7. **Generics (Go 1.18+) replaced many reflection use cases**, particularly generic data structures and algorithms. Use generics when types satisfy a constraint; use reflection when types are truly unknown.

8. **The Three Laws of Reflection** (interface to reflection, reflection to interface, settability) govern how reflection works. Internalize them.

9. **Generated code is committed to the repository** in Go (unlike JavaScript's build output). Run `go generate` during development, not at build time.

10. **`reflect.DeepEqual`** is convenient but beware of nil vs empty semantics, function comparison, and performance.

---

## Practice Exercises

### Exercise 1: Struct to CSV Converter
Build a function `ToCSV(slice interface{}) (string, error)` that takes a slice of structs and outputs CSV. Use struct tags `csv:"column_name"` for header names. Handle `csv:"-"` to skip fields.

```go
type Employee struct {
    ID         int     `csv:"id"`
    Name       string  `csv:"name"`
    Department string  `csv:"department"`
    Salary     float64 `csv:"-"` // skip this field
}

employees := []Employee{
    {1, "Alice", "Engineering", 100000},
    {2, "Bob", "Marketing", 85000},
}

csv, _ := ToCSV(employees)
// id,name,department
// 1,Alice,Engineering
// 2,Bob,Marketing
```

### Exercise 2: Configuration Loader
Build a function that populates a struct from environment variables using struct tags:

```go
type AppConfig struct {
    Host     string `env:"APP_HOST" default:"localhost"`
    Port     int    `env:"APP_PORT" default:"8080"`
    Debug    bool   `env:"APP_DEBUG" default:"false"`
    LogLevel string `env:"LOG_LEVEL" default:"info"`
}

cfg := AppConfig{}
LoadFromEnv(&cfg)
```

Use reflection to read the `env` and `default` tags, look up environment variables, convert types, and set values.

### Exercise 3: Method Benchmarker
Write a function that takes any struct, finds all exported methods with no parameters and one return value, calls each one via reflection, and reports the execution time. Compare the reflected call time against a direct call.

### Exercise 4: Code Generator for Builders
Write a `go generate`-compatible tool that reads a struct definition and produces a Builder pattern implementation:

```go
//go:generate go run builder_gen.go -type=Server

type Server struct {
    Host    string
    Port    int
    TLS     bool
    Timeout time.Duration
}

// Generated:
// type ServerBuilder struct { ... }
// func NewServerBuilder() *ServerBuilder { ... }
// func (b *ServerBuilder) Host(v string) *ServerBuilder { ... }
// func (b *ServerBuilder) Port(v int) *ServerBuilder { ... }
// func (b *ServerBuilder) Build() Server { ... }
```

### Exercise 5: Reflection vs Generics Refactoring
Take the following reflection-based code and refactor it to use generics where possible. Identify which parts *still* need reflection:

```go
func Filter(slice interface{}, predicate interface{}) interface{} {
    sv := reflect.ValueOf(slice)
    pv := reflect.ValueOf(predicate)
    result := reflect.MakeSlice(sv.Type(), 0, sv.Len())
    for i := 0; i < sv.Len(); i++ {
        args := []reflect.Value{sv.Index(i)}
        if pv.Call(args)[0].Bool() {
            result = reflect.Append(result, sv.Index(i))
        }
    }
    return result.Interface()
}
```

Target:

```go
func Filter[T any](slice []T, predicate func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}
```

Notice: the generic version is shorter, faster, type-safe, and requires zero reflection.

---

## Further Reading

- [The Laws of Reflection](https://go.dev/blog/laws-of-reflection) -- Rob Pike's original blog post
- [Go `reflect` package documentation](https://pkg.go.dev/reflect)
- [Go generate documentation](https://go.dev/blog/generate)
- [Go `go/ast` package](https://pkg.go.dev/go/ast) -- for building code generators
- [easyjson](https://github.com/mailru/easyjson) -- code-generated JSON marshaling, zero reflection
- [go-cmp](https://github.com/google/go-cmp) -- better alternative to `reflect.DeepEqual`
- [Ent](https://entgo.io/) -- code-generated ORM framework
