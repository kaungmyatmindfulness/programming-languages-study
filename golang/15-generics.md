# Chapter 15: Generics & Type Parameters

## Table of Contents
1. [Why Generics?](#1-why-generics)
2. [Type Parameters](#2-type-parameters)
3. [Type Constraints](#3-type-constraints)
4. [Type Sets](#4-type-sets)
5. [Generic Types](#5-generic-types)
6. [Type Inference](#6-type-inference)
7. [The constraints Package](#7-the-constraints-package)
8. [The slices and maps Packages](#8-the-slices-and-maps-packages)
9. [When to Use Generics](#9-when-to-use-generics)
10. [Limitations](#10-limitations)
11. [Generic Design Patterns](#11-generic-design-patterns)
12. [Go vs TypeScript Generics](#12-go-vs-typescript-generics)
13. [Key Takeaways](#13-key-takeaways)
14. [Practice Exercises](#14-practice-exercises)

---

## 1. Why Generics?

### The Problem Before Generics

Before Go 1.18, you had exactly three strategies for writing code that worked across multiple types. All of them had significant trade-offs.

**Strategy 1: Copy-Paste (Most Common)**

You literally wrote the same function for every type:

```go
// Pre-generics: want a "contains" function? Write it for each type.
func ContainsInt(slice []int, target int) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

func ContainsString(slice []string, target string) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

func ContainsFloat64(slice []float64, target float64) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}
```

This is the same logic three times. If you found a bug, you had to fix it in three places. If you needed `float32`, you wrote it a fourth time. The Go standard library itself had `sort.Ints()`, `sort.Strings()`, and `sort.Float64s()` -- three identical algorithms differing only in type.

**Strategy 2: `interface{}` (The "Any" Escape Hatch)**

Go's empty interface `interface{}` accepted any value, so you could write type-agnostic code:

```go
// Pre-generics: use interface{} to accept anything
func Contains(slice []interface{}, target interface{}) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// Usage -- ugly and error-prone
nums := []interface{}{1, 2, 3, 4, 5}  // can't use []int directly
found := Contains(nums, 3)

// This compiles but is nonsense -- comparing int with string
Contains(nums, "hello") // no compile error, just returns false
```

Problems with `interface{}`:
- **No type safety.** The compiler could not catch mismatched types.
- **Requires type assertions.** Getting values back meant `value.(int)` everywhere, with runtime panic risk.
- **No direct slice conversion.** You could not pass `[]int` where `[]interface{}` was expected -- you had to convert element by element.
- **Performance cost.** Boxing values into interfaces caused heap allocations.

**Strategy 3: Code Generation**

Tools like `go generate` and third-party tools (e.g., `genny`, `gen`) would read template files and produce typed code:

```go
//go:generate genny -in=contains.go -out=gen-contains.go gen "T=int,string,float64"

func ContainsT(slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}
```

This generated type-safe code, but it was clunky. Generated files cluttered the repo, the template syntax was non-standard, and the tooling was fragile.

### Why Go Waited Until 1.18

Go was released in 2009. Generics arrived in 2022 with Go 1.18 -- a 13-year gap. This was not an oversight. It was deliberate.

The Go team's reasoning:

1. **"Get it right, not fast."** Rob Pike, Robert Griesemer, and Ian Lance Taylor studied generics proposals for over a decade. There were at least four major proposals before the final design was accepted. They watched how generics played out in other languages (Java's type erasure headaches, C++ template error messages, Rust's steep learning curve with traits and lifetimes).

2. **Simplicity is a feature.** Go's core philosophy is that simple code beats clever code. The team worried that generics would encourage over-abstraction -- the kind of `AbstractGenericFactoryBuilderProvider<T, U, V>` code that plagues enterprise Java. They wanted a design that made simple things simple and did not reward complexity.

3. **Most Go code did not need generics.** Interfaces and concrete types handled 90%+ of real-world use cases. The Go team studied the actual pain points (container types, sorting, Map/Filter/Reduce utilities) and designed generics to address those specific gaps.

4. **The constraint system needed time.** The final design uses interfaces as constraints -- a genuinely novel approach. Interfaces already existed in Go, so reusing them as the constraint mechanism kept the language coherent. This insight did not come quickly.

The final proposal (by Ian Lance Taylor and Robert Griesemer) was accepted in 2021, and Go 1.18 shipped with generics on March 15, 2022.

---

## 2. Type Parameters

### Basic Syntax

A generic function declares **type parameters** in square brackets before the regular parameters:

```go
func FunctionName[T constraint](params) returnType {
    // T is available as a type inside this function
}
```

Here is the `Contains` function, rewritten with generics:

```go
// One function. Works for any comparable type.
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// Usage -- fully type-safe, no interface{} in sight
nums := []int{1, 2, 3, 4, 5}
found := Contains(nums, 3)          // T inferred as int
found = Contains(nums, "hello")     // COMPILE ERROR: string != int

names := []string{"alice", "bob"}
found = Contains(names, "bob")      // T inferred as string
```

The compiler knows `T` is `int` when you pass `[]int`, so passing `"hello"` is caught at compile time. The function is written once and works for every comparable type.

### Multiple Type Parameters

You can declare multiple type parameters:

```go
// Map transforms a slice of one type to a slice of another type
func Map[T any, U any](slice []T, transform func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = transform(v)
    }
    return result
}

// Usage
names := []string{"alice", "bob", "charlie"}
lengths := Map(names, func(s string) int { return len(s) })
// lengths = [5, 3, 7]

nums := []int{1, 2, 3, 4}
doubled := Map(nums, func(n int) int { return n * 2 })
// doubled = [2, 4, 6, 8]

strs := Map(nums, func(n int) string { return fmt.Sprintf("%d", n) })
// strs = ["1", "2", "3", "4"]
```

### Naming Conventions

Go's convention for type parameter names:

| Convention | When to Use | Example |
|---|---|---|
| `T` | Single type parameter, generic context | `func Foo[T any](x T)` |
| `T, U, V` | Multiple unrelated type parameters | `func Map[T, U any](...)` |
| `K, V` | Map key and value types | `func MapKeys[K comparable, V any](m map[K]V)` |
| `E` | Element type (for slices/collections) | `type Stack[E any] struct{...}` |
| `S` | Slice type | `func Sort[S ~[]E, E Ordered](s S)` |

Unlike Java (`<T extends Comparable<T>>`) or C++ (`template<typename T>`), Go uses square brackets `[T constraint]`. This avoids ambiguity with comparison operators -- `f<T>(x)` could mean "call generic function f" or "compare f with T, then compare result with x".

---

## 3. Type Constraints

### What Constraints Are and Why They Exist

A **constraint** tells the compiler what operations are valid on a type parameter. Without constraints, the compiler cannot know whether `T` supports `+`, `<`, `.String()`, or anything else.

```go
// This does NOT compile -- what does "+" mean for an arbitrary type?
func Sum[T any](nums []T) T {
    var total T
    for _, n := range nums {
        total += n  // ERROR: operator += not defined for T
    }
    return total
}
```

The compiler rejects this because `any` (alias for `interface{}`) allows *every* type, including structs, slices, and channels, where `+` makes no sense. Constraints narrow the set of allowed types.

### The `any` Constraint

`any` is the loosest constraint. It allows every type but permits almost no operations:

```go
func Print[T any](value T) {
    fmt.Println(value) // Works: fmt.Println accepts interface{}
}

func Identity[T any](value T) T {
    return value // Works: just passing T through
}
```

With `any`, you can only:
- Pass values around
- Assign to variables of the same type
- Pass to functions accepting `interface{}`

You **cannot** compare, add, subtract, index, or call methods on `T any`.

### The `comparable` Constraint

`comparable` allows `==` and `!=`. It includes all basic types (int, string, float64, bool, etc.), arrays, structs (if all fields are comparable), and pointers -- but not slices, maps, or functions.

```go
// comparable lets us use == and !=
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target { // This is why we need "comparable"
            return true
        }
    }
    return false
}

func IndexOf[T comparable](slice []T, target T) int {
    for i, v := range slice {
        if v == target {
            return i
        }
    }
    return -1
}
```

### Custom Constraints with Interfaces

In Go 1.18+, interfaces serve double duty: they define method sets (classic role) AND they define type constraints (new role).

```go
// A constraint that requires a String() method
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

// A constraint that requires both a method AND comparable
type ComparableStringer interface {
    comparable
    String() string
}
```

You can combine method requirements with type restrictions in a single interface, which we explore next.

---

## 4. Type Sets

### The Paradigm Shift in Go 1.18

Before Go 1.18, an interface defined a **method set** -- "any type that has these methods." Go 1.18 redefined interfaces to define a **type set** -- "any type that belongs to this set." Method sets are one way to define the set, but now you can also enumerate specific types.

### Union Types in Constraints

You can list specific types a constraint allows using the `|` operator:

```go
// Only allows int, float64, and string
type Number interface {
    int | float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n // Now the compiler knows += is valid for int and float64
    }
    return total
}

// Usage
Sum([]int{1, 2, 3})           // returns 6
Sum([]float64{1.1, 2.2, 3.3}) // returns 6.6
Sum([]string{"a", "b"})       // COMPILE ERROR: string not in Number
```

### The `~` Operator (Underlying Types)

This is where Go generics get powerful. The `~` prefix means "any type whose **underlying type** is this":

```go
// Without ~ : only exact int type
type StrictInt interface {
    int
}

// With ~ : int AND any type defined as int
type FlexibleInt interface {
    ~int
}

// This matters for named types:
type UserID int
type Age int

func Double[T StrictInt](n T) T { return n + n }
Double(int(5))      // OK
Double(UserID(5))   // COMPILE ERROR: UserID is not int

func DoubleFlex[T FlexibleInt](n T) T { return n + n }
DoubleFlex(int(5))    // OK
DoubleFlex(UserID(5)) // OK -- UserID's underlying type is int
DoubleFlex(Age(25))   // OK -- Age's underlying type is int
```

This is critical for real-world Go code, where named types are extremely common. Without `~`, generics would be nearly useless for custom types.

### Building Rich Constraints

You can combine unions, `~`, methods, and embedding:

```go
// Numbers: any type whose underlying type is a numeric type
type Number interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

// Ordered: any type that supports < > <= >=
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64 |
    ~string
}

func Min[T Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

func Max[T Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// Mix type lists with methods
type StringableNumber interface {
    ~int | ~float64
    String() string
}
```

**Important rule:** Constraints with type lists (unions) can only be used as constraints, not as regular interface types. You cannot declare a variable of type `Number` -- it only works inside `[T Number]`.

```go
var x Number    // COMPILE ERROR: cannot use Number as type
var y any = 5   // OK: any is still usable as a regular interface
```

---

## 5. Generic Types

### Generic Structs

You can parameterize structs with type parameters:

```go
// A generic Pair
type Pair[T, U any] struct {
    First  T
    Second U
}

func NewPair[T, U any](first T, second U) Pair[T, U] {
    return Pair[T, U]{First: first, Second: second}
}

// Usage
p1 := Pair[string, int]{First: "age", Second: 30}
p2 := NewPair("name", "Alice") // inferred: Pair[string, string]
p3 := NewPair(1, true)         // inferred: Pair[int, bool]
```

### Generic Stack

A classic data structure, now type-safe without `interface{}`:

```go
type Stack[T any] struct {
    items []T
}

func NewStack[T any]() *Stack[T] {
    return &Stack[T]{items: make([]T, 0)}
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Len() int {
    return len(s.items)
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

// Usage
intStack := NewStack[int]()
intStack.Push(1)
intStack.Push(2)
intStack.Push(3)

val, ok := intStack.Pop() // val=3, ok=true
// intStack.Push("hello") // COMPILE ERROR: string != int

strStack := NewStack[string]()
strStack.Push("hello")
strStack.Push("world")
```

Compare this with the pre-generics approach:

```go
// Pre-generics stack -- no type safety
type OldStack struct {
    items []interface{}
}

func (s *OldStack) Push(item interface{}) {
    s.items = append(s.items, item)
}

func (s *OldStack) Pop() interface{} {
    if len(s.items) == 0 {
        return nil
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}

// Usage -- dangerous
s := &OldStack{}
s.Push(1)
s.Push("oops") // No compile error -- mixed types
val := s.Pop()
num := val.(int) // RUNTIME PANIC: "oops" is a string, not int
```

### Generic Queue

```go
type Queue[T any] struct {
    items []T
}

func NewQueue[T any]() *Queue[T] {
    return &Queue[T]{items: make([]T, 0)}
}

func (q *Queue[T]) Enqueue(item T) {
    q.items = append(q.items, item)
}

func (q *Queue[T]) Dequeue() (T, bool) {
    if len(q.items) == 0 {
        var zero T
        return zero, false
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item, true
}

func (q *Queue[T]) Peek() (T, bool) {
    if len(q.items) == 0 {
        var zero T
        return zero, false
    }
    return q.items[0], true
}

func (q *Queue[T]) Len() int {
    return len(q.items)
}
```

### Generic Linked List

```go
type Node[T any] struct {
    Value T
    Next  *Node[T]
}

type LinkedList[T any] struct {
    Head *Node[T]
    size int
}

func NewLinkedList[T any]() *LinkedList[T] {
    return &LinkedList[T]{}
}

func (l *LinkedList[T]) Prepend(value T) {
    l.Head = &Node[T]{Value: value, Next: l.Head}
    l.size++
}

func (l *LinkedList[T]) Append(value T) {
    newNode := &Node[T]{Value: value}
    if l.Head == nil {
        l.Head = newNode
        l.size++
        return
    }
    current := l.Head
    for current.Next != nil {
        current = current.Next
    }
    current.Next = newNode
    l.size++
}

func (l *LinkedList[T]) ToSlice() []T {
    result := make([]T, 0, l.size)
    current := l.Head
    for current != nil {
        result = append(result, current.Value)
        current = current.Next
    }
    return result
}

func (l *LinkedList[T]) Len() int {
    return l.size
}
```

### Generic Map Wrapper (Thread-Safe)

```go
import "sync"

type SafeMap[K comparable, V any] struct {
    mu   sync.RWMutex
    data map[K]V
}

func NewSafeMap[K comparable, V any]() *SafeMap[K, V] {
    return &SafeMap[K, V]{data: make(map[K]V)}
}

func (m *SafeMap[K, V]) Set(key K, value V) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key] = value
}

func (m *SafeMap[K, V]) Get(key K) (V, bool) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    val, ok := m.data[key]
    return val, ok
}

func (m *SafeMap[K, V]) Delete(key K) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.data, key)
}

func (m *SafeMap[K, V]) Len() int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return len(m.data)
}

// Usage
cache := NewSafeMap[string, int]()
cache.Set("users", 42)
count, ok := cache.Get("users") // count=42, ok=true
```

---

## 6. Type Inference

Go's compiler can often **infer** type arguments from the regular arguments, so you do not need to write them explicitly.

### When Inference Works

```go
func Contains[T comparable](slice []T, target T) bool { ... }

// Inference works -- T is deduced from the arguments
Contains([]int{1, 2, 3}, 4)         // T inferred as int
Contains([]string{"a", "b"}, "c")   // T inferred as string

// Explicit type arguments -- legal but unnecessary here
Contains[int]([]int{1, 2, 3}, 4)
Contains[string]([]string{"a", "b"}, "c")
```

### When Inference Fails (Must Specify Explicitly)

**Case 1: The type parameter is only in the return type.**

```go
func Zero[T any]() T {
    var zero T
    return zero
}

// Compiler cannot infer T -- no arguments to work with
val := Zero()       // COMPILE ERROR: cannot infer T
val := Zero[int]()  // OK: explicitly int
val := Zero[string]() // OK: explicitly string
```

**Case 2: Constructing generic types.**

```go
// Generic types always require explicit type arguments
s := NewStack[int]()    // Must specify int
m := NewSafeMap[string, int]() // Must specify both K and V

// There is no inference for type instantiation:
s := NewStack()         // COMPILE ERROR
```

**Case 3: Ambiguous inference.**

```go
func Convert[From, To any](val From, converter func(From) To) To {
    return converter(val)
}

// Both types inferred from arguments -- works
result := Convert(42, func(n int) string { return fmt.Sprintf("%d", n) })

// But if you use a generic converter:
func ToString[T any](val T) string { return fmt.Sprintf("%v", val) }

// Must specify because the function literal's type params add ambiguity
result := Convert[int, string](42, ToString[int])
```

### Partial Inference (Not Supported)

Go does not support partial type argument specification. You either provide all type arguments or none:

```go
func Transform[T, U any](val T, fn func(T) U) U { return fn(val) }

// All or nothing:
Transform(42, strconv.Itoa)                    // All inferred -- OK
Transform[int, string](42, strconv.Itoa)       // All explicit -- OK
// Transform[int](42, strconv.Itoa)            // Partial -- COMPILE ERROR
```

---

## 7. The constraints Package

Go provides a `golang.org/x/exp/constraints` package with commonly needed constraints. These were experimental initially, and key ones have been adopted into the standard library or superseded by standard library packages.

### Key Constraints

```go
import "golang.org/x/exp/constraints"

// constraints.Ordered -- any type that supports < > <= >=
// Equivalent to:
// ~int | ~int8 | ~int16 | ~int32 | ~int64 |
// ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
// ~float32 | ~float64 | ~string

func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// constraints.Integer -- all integer types
// constraints.Float -- all float types
// constraints.Complex -- all complex types
// constraints.Signed -- all signed integer types
// constraints.Unsigned -- all unsigned integer types

func Abs[T constraints.Signed | constraints.Float](n T) T {
    if n < 0 {
        return -n
    }
    return n
}
```

### The `cmp` Package (Go 1.21+)

Go 1.21 moved ordering into the standard library with the `cmp` package:

```go
import "cmp"

// cmp.Ordered is the standard-library equivalent of constraints.Ordered
func Min[T cmp.Ordered](a, b T) T {
    return cmp.Or(a, b) // not quite -- use cmp.Compare
}

// cmp.Compare returns -1, 0, or +1
func SortBy[T any, K cmp.Ordered](slice []T, key func(T) K) {
    slices.SortFunc(slice, func(a, b T) int {
        return cmp.Compare(key(a), key(b))
    })
}

// cmp.Or returns the first non-zero value (useful for default values)
name := cmp.Or(userInput, envVar, "default")
```

---

## 8. The slices and maps Packages

Go 1.21 introduced `slices` and `maps` packages in the standard library -- the first generic packages available without any external dependency. These replaced a large number of hand-rolled utility functions.

### The `slices` Package

```go
import "slices"

// --- Searching ---

nums := []int{3, 1, 4, 1, 5, 9, 2, 6}

slices.Contains(nums, 5)    // true
slices.Contains(nums, 42)   // false
slices.Index(nums, 4)       // 2 (index of first 4)
slices.Index(nums, 42)      // -1 (not found)

// --- Sorting ---

slices.Sort(nums) // sorts in-place: [1, 1, 2, 3, 4, 5, 6, 9]
slices.IsSorted(nums) // true

// Sort with custom comparison
type Person struct {
    Name string
    Age  int
}

people := []Person{
    {"Charlie", 30},
    {"Alice", 25},
    {"Bob", 35},
}

slices.SortFunc(people, func(a, b Person) int {
    return cmp.Compare(a.Age, b.Age)
})
// [{Alice 25} {Charlie 30} {Bob 35}]

// --- Comparison ---

a := []int{1, 2, 3}
b := []int{1, 2, 3}
c := []int{1, 2, 4}

slices.Equal(a, b)   // true
slices.Equal(a, c)   // false
slices.Compare(a, c) // -1 (a < c because 3 < 4)

// --- Manipulation ---

nums = []int{1, 2, 3, 4, 5}
reversed := slices.Clone(nums)
slices.Reverse(reversed) // [5, 4, 3, 2, 1]

compact := []int{1, 1, 2, 2, 3, 3}
compact = slices.Compact(compact) // [1, 2, 3] (removes consecutive duplicates)

// --- Min/Max ---

slices.Min([]int{3, 1, 4, 1, 5}) // 1
slices.Max([]int{3, 1, 4, 1, 5}) // 5

// --- Binary Search (on sorted slices) ---

sorted := []int{1, 3, 5, 7, 9}
idx, found := slices.BinarySearch(sorted, 5)  // idx=2, found=true
idx, found = slices.BinarySearch(sorted, 6)   // idx=3, found=false

// --- Delete ---

nums = []int{1, 2, 3, 4, 5}
nums = slices.Delete(nums, 1, 3) // removes indices 1..2: [1, 4, 5]

// --- Insert ---

nums = []int{1, 2, 5}
nums = slices.Insert(nums, 2, 3, 4) // [1, 2, 3, 4, 5]

// --- Grow/Clip ---

nums = slices.Grow(nums, 100)  // ensure capacity for 100 more elements
nums = slices.Clip(nums)       // remove unused capacity
```

### The `maps` Package

```go
import "maps"

m := map[string]int{"a": 1, "b": 2, "c": 3}

// --- Keys and Values ---

keys := slices.Collect(maps.Keys(m))     // ["a", "b", "c"] (order not guaranteed)
values := slices.Collect(maps.Values(m)) // [1, 2, 3] (order not guaranteed)

// --- Clone ---

m2 := maps.Clone(m) // shallow copy

// --- Equal ---

m3 := map[string]int{"a": 1, "b": 2, "c": 3}
maps.Equal(m, m3) // true

// --- Copy (merge src into dst) ---

dst := map[string]int{"a": 10, "d": 4}
maps.Copy(dst, m) // dst is now {"a": 1, "b": 2, "c": 3, "d": 4}

// --- DeleteFunc ---

m4 := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4}
maps.DeleteFunc(m4, func(k string, v int) bool {
    return v%2 == 0 // delete even values
})
// m4 is now {"a": 1, "c": 3}
```

### Before vs After: The Transformation

```go
// BEFORE (pre-generics, pre-slices package):
func containsString(slice []string, target string) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

func containsInt(slice []int, target int) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

sort.Ints(nums)
sort.Strings(names)
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})

// AFTER (Go 1.21+):
slices.Contains(names, "alice")
slices.Contains(nums, 42)
slices.Sort(nums)
slices.Sort(names)
slices.SortFunc(people, func(a, b Person) int {
    return cmp.Compare(a.Age, b.Age)
})
```

---

## 9. When to Use Generics

The Go team published clear guidance. Rob Pike's one-liner: **"Don't use generics unless you find yourself writing the exact same code multiple times with only the type changed."**

### Use Generics When...

**1. Writing general-purpose data structures.**

```go
// YES -- a tree, stack, queue, or cache that works with any type
type LRUCache[K comparable, V any] struct { ... }
type PriorityQueue[T any] struct { ... }
type Set[T comparable] struct { ... }
```

**2. Writing functions that operate on slices, maps, or channels of any element type.**

```go
// YES -- the type of the elements doesn't matter to the algorithm
func Filter[T any](slice []T, predicate func(T) bool) []T { ... }
func Reduce[T, U any](slice []T, init U, fn func(U, T) U) U { ... }
func Chunk[T any](slice []T, size int) [][]T { ... }
func Unique[T comparable](slice []T) []T { ... }
```

**3. Writing type-safe wrappers where the type relationship matters.**

```go
// YES -- the result carries the type through
type Result[T any] struct {
    Value T
    Err   error
}
```

### Do NOT Use Generics When...

**1. The function body does different things based on type.**

```go
// NO -- just write separate functions
func Process[T any](val T) string {
    // If you need to switch on T, generics are the wrong tool
    switch v := any(val).(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %s", v)
    }
    return "unknown"
}

// YES -- separate functions are clearer
func ProcessInt(val int) string    { return fmt.Sprintf("int: %d", val) }
func ProcessString(val string) string { return fmt.Sprintf("string: %s", val) }
```

**2. An interface works just as well.**

```go
// NO -- unnecessary generics
func PrintStringer[T fmt.Stringer](val T) {
    fmt.Println(val.String())
}

// YES -- a plain interface parameter does the same job
func PrintStringer(val fmt.Stringer) {
    fmt.Println(val.String())
}
```

The difference: generic version creates a specialized function per type (monomorphization). Interface version uses dynamic dispatch. If your function only calls methods defined in the constraint, an interface parameter is simpler and equivalent.

**3. One concrete type is all you need right now.**

```go
// NO -- premature generics
type UserCache[K comparable, V any] struct { ... }

// YES -- start concrete, generalize later IF needed
type UserCache struct {
    data map[int]*User  // key is always user ID, value is always *User
}
```

The Go team's mantra: **"Start with concrete types. Generalize when you see the pattern repeat."**

---

## 10. Limitations

Go's generics are deliberately more constrained than generics in TypeScript, Rust, or C++. Each limitation has a reason.

### No Generic Methods

You can define methods on generic types, but you cannot add new type parameters to a method:

```go
type Container[T any] struct {
    items []T
}

// OK -- method on a generic type (uses T from the type definition)
func (c *Container[T]) Add(item T) {
    c.items = append(c.items, item)
}

// NOT OK -- method cannot introduce new type parameters
// func (c *Container[T]) Transform[U any](fn func(T) U) *Container[U] {
//     // COMPILE ERROR: method must have no type parameters
// }

// Workaround: use a top-level generic function instead
func Transform[T, U any](c *Container[T], fn func(T) U) *Container[U] {
    result := &Container[U]{items: make([]U, len(c.items))}
    for i, item := range c.items {
        result.items[i] = fn(item)
    }
    return result
}
```

**Why?** Generic methods would require the Go interface system to handle method sets with unbounded type parameters. Consider: if `Container[int]` implements an interface, does `Transform[string]` count as part of its method set? This opens a can of worms around interface satisfaction and virtual dispatch. The Go team decided to keep the interface system simple.

### No Specialization

You cannot provide a different implementation for specific type arguments:

```go
// NOT possible in Go:
// func Sum[T Number](nums []T) T { /* general */ }
// func Sum[int](nums []int) int { /* optimized for int */ }

// You must use runtime checks or separate functions:
func SumInts(nums []int) int { /* optimized */ }
func Sum[T Number](nums []T) T { /* general */ }
```

**Why?** Specialization creates a complex override resolution system (see C++ template specialization). Go chose simplicity.

### No Type Assertions on Type Parameters

You cannot use type assertions or type switches directly on type parameter values:

```go
func Process[T any](val T) {
    // This does NOT work as you might expect:
    // switch val.(type) {    // COMPILE ERROR
    // case int:
    //     ...
    // }

    // Workaround: convert to any first
    switch v := any(val).(type) {
    case int:
        fmt.Println("int:", v)
    case string:
        fmt.Println("string:", v)
    default:
        fmt.Println("other:", v)
    }
}
```

**Why?** If you need to branch on the concrete type, generics are likely the wrong abstraction. The `any(val)` workaround exists for the rare case you genuinely need it.

### No Generic Type Aliases (Until Go 1.24)

Prior to Go 1.24, you could not create a type alias for a generic type:

```go
// Before Go 1.24:
// type IntStack = Stack[int]  // COMPILE ERROR in Go 1.18-1.23

// This works as a type definition (not alias):
type IntStack Stack[int] // This is a new type, not an alias -- no methods inherited
```

Go 1.24 (released February 2025) added support for generic type aliases, resolving this limitation.

### No Variadic Type Parameters

You cannot have a variable number of type parameters:

```go
// NOT possible:
// func Zip[T1, T2, ...Tn any](s1 []T1, s2 []T2, ...sn []Tn) []Tuple[T1, T2, ...Tn]

// You must write specific arities:
func Zip2[T1, T2 any](s1 []T1, s2 []T2) []Pair[T1, T2] { ... }
func Zip3[T1, T2, T3 any](s1 []T1, s2 []T2, s3 []T3) []Triple[T1, T2, T3] { ... }
```

### No Operator Overloading Through Constraints

Constraints can require operators for built-in types via type unions, but you cannot make custom types support operators:

```go
type Addable interface {
    ~int | ~float64 | ~string // built-in types that support +
}

// You cannot make your own Matrix type satisfy Addable
// because Go does not have operator overloading
type Matrix struct { ... }
// Matrix does NOT support + even if you define an Add method
```

---

## 11. Generic Design Patterns

### Pattern 1: Result Type

Encapsulate a value-or-error pair with type safety:

```go
type Result[T any] struct {
    value T
    err   error
    ok    bool
}

func Ok[T any](value T) Result[T] {
    return Result[T]{value: value, ok: true}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err, ok: false}
}

func (r Result[T]) Unwrap() (T, error) {
    if !r.ok {
        var zero T
        return zero, r.err
    }
    return r.value, nil
}

func (r Result[T]) UnwrapOr(defaultVal T) T {
    if !r.ok {
        return defaultVal
    }
    return r.value
}

func (r Result[T]) IsOk() bool {
    return r.ok
}

// Map transforms the value inside a Result if it's Ok
func MapResult[T, U any](r Result[T], fn func(T) U) Result[U] {
    if !r.ok {
        return Err[U](r.err)
    }
    return Ok(fn(r.value))
}

// Usage
func FetchUser(id int) Result[User] {
    user, err := db.GetUser(id)
    if err != nil {
        return Err[User](err)
    }
    return Ok(user)
}

result := FetchUser(42)
name := MapResult(result, func(u User) string { return u.Name })
fmt.Println(name.UnwrapOr("anonymous"))
```

### Pattern 2: Option Type

Represent the presence or absence of a value without nil pointers:

```go
type Option[T any] struct {
    value T
    some  bool
}

func Some[T any](value T) Option[T] {
    return Option[T]{value: value, some: true}
}

func None[T any]() Option[T] {
    return Option[T]{some: false}
}

func (o Option[T]) IsSome() bool {
    return o.some
}

func (o Option[T]) IsNone() bool {
    return !o.some
}

func (o Option[T]) Unwrap() T {
    if !o.some {
        panic("called Unwrap on None")
    }
    return o.value
}

func (o Option[T]) UnwrapOr(defaultVal T) T {
    if !o.some {
        return defaultVal
    }
    return o.value
}

func (o Option[T]) UnwrapOrElse(fn func() T) T {
    if !o.some {
        return fn()
    }
    return o.value
}

// Usage
func FindUser(name string) Option[User] {
    user, err := db.FindByName(name)
    if err != nil {
        return None[User]()
    }
    return Some(user)
}

user := FindUser("alice")
if user.IsSome() {
    fmt.Println("Found:", user.Unwrap().Name)
}
```

### Pattern 3: Functional Utilities (Map, Filter, Reduce)

```go
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0)
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

func Reduce[T, U any](slice []T, initial U, fn func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = fn(result, v)
    }
    return result
}

func ForEach[T any](slice []T, fn func(T)) {
    for _, v := range slice {
        fn(v)
    }
}

func Any[T any](slice []T, predicate func(T) bool) bool {
    for _, v := range slice {
        if predicate(v) {
            return true
        }
    }
    return false
}

func All[T any](slice []T, predicate func(T) bool) bool {
    for _, v := range slice {
        if !predicate(v) {
            return false
        }
    }
    return true
}

func FlatMap[T, U any](slice []T, fn func(T) []U) []U {
    result := make([]U, 0)
    for _, v := range slice {
        result = append(result, fn(v)...)
    }
    return result
}

// Usage
nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

evens := Filter(nums, func(n int) bool { return n%2 == 0 })
// [2, 4, 6, 8, 10]

doubled := Map(evens, func(n int) int { return n * 2 })
// [4, 8, 12, 16, 20]

sum := Reduce(doubled, 0, func(acc, n int) int { return acc + n })
// 60

strs := Map(nums, func(n int) string { return fmt.Sprintf("#%d", n) })
// ["#1", "#2", "#3", ...]

hasNegative := Any(nums, func(n int) bool { return n < 0 })
// false

allPositive := All(nums, func(n int) bool { return n > 0 })
// true
```

### Pattern 4: Type-Safe Set

```go
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable](items ...T) *Set[T] {
    s := &Set[T]{items: make(map[T]struct{})}
    for _, item := range items {
        s.items[item] = struct{}{}
    }
    return s
}

func (s *Set[T]) Add(item T) {
    s.items[item] = struct{}{}
}

func (s *Set[T]) Remove(item T) {
    delete(s.items, item)
}

func (s *Set[T]) Contains(item T) bool {
    _, ok := s.items[item]
    return ok
}

func (s *Set[T]) Len() int {
    return len(s.items)
}

func (s *Set[T]) ToSlice() []T {
    result := make([]T, 0, len(s.items))
    for item := range s.items {
        result = append(result, item)
    }
    return result
}

func (s *Set[T]) Union(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for item := range s.items {
        result.Add(item)
    }
    for item := range other.items {
        result.Add(item)
    }
    return result
}

func (s *Set[T]) Intersection(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for item := range s.items {
        if other.Contains(item) {
            result.Add(item)
        }
    }
    return result
}

func (s *Set[T]) Difference(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for item := range s.items {
        if !other.Contains(item) {
            result.Add(item)
        }
    }
    return result
}

// Usage
s1 := NewSet(1, 2, 3, 4, 5)
s2 := NewSet(3, 4, 5, 6, 7)

union := s1.Union(s2)              // {1, 2, 3, 4, 5, 6, 7}
intersection := s1.Intersection(s2) // {3, 4, 5}
diff := s1.Difference(s2)          // {1, 2}

s1.Contains(3) // true
s1.Contains(9) // false
```

### Pattern 5: Generic Repository / Data Access

```go
type Entity interface {
    comparable
    GetID() string
}

type Repository[T Entity] struct {
    store map[string]T
}

func NewRepository[T Entity]() *Repository[T] {
    return &Repository[T]{store: make(map[string]T)}
}

func (r *Repository[T]) Save(entity T) {
    r.store[entity.GetID()] = entity
}

func (r *Repository[T]) FindByID(id string) (T, bool) {
    entity, ok := r.store[id]
    return entity, ok
}

func (r *Repository[T]) FindAll() []T {
    result := make([]T, 0, len(r.store))
    for _, entity := range r.store {
        result = append(result, entity)
    }
    return result
}

func (r *Repository[T]) FindWhere(predicate func(T) bool) []T {
    result := make([]T, 0)
    for _, entity := range r.store {
        if predicate(entity) {
            result = append(result, entity)
        }
    }
    return result
}

func (r *Repository[T]) Delete(id string) {
    delete(r.store, id)
}

func (r *Repository[T]) Count() int {
    return len(r.store)
}

// Usage
type User struct {
    ID   string
    Name string
    Age  int
}

func (u User) GetID() string { return u.ID }

userRepo := NewRepository[User]()
userRepo.Save(User{ID: "1", Name: "Alice", Age: 30})
userRepo.Save(User{ID: "2", Name: "Bob", Age: 25})

user, found := userRepo.FindByID("1")
adults := userRepo.FindWhere(func(u User) bool { return u.Age >= 18 })
```

### Pattern 6: Grouping and Aggregation

```go
func GroupBy[T any, K comparable](slice []T, keyFn func(T) K) map[K][]T {
    result := make(map[K][]T)
    for _, item := range slice {
        key := keyFn(item)
        result[key] = append(result[key], item)
    }
    return result
}

func CountBy[T any, K comparable](slice []T, keyFn func(T) K) map[K]int {
    result := make(map[K]int)
    for _, item := range slice {
        result[keyFn(item)]++
    }
    return result
}

func Unique[T comparable](slice []T) []T {
    seen := make(map[T]struct{})
    result := make([]T, 0)
    for _, item := range slice {
        if _, ok := seen[item]; !ok {
            seen[item] = struct{}{}
            result = append(result, item)
        }
    }
    return result
}

func UniqueBy[T any, K comparable](slice []T, keyFn func(T) K) []T {
    seen := make(map[K]struct{})
    result := make([]T, 0)
    for _, item := range slice {
        key := keyFn(item)
        if _, ok := seen[key]; !ok {
            seen[key] = struct{}{}
            result = append(result, item)
        }
    }
    return result
}

// Usage
type Order struct {
    CustomerID string
    Amount     float64
    Category   string
}

orders := []Order{
    {"c1", 100, "electronics"},
    {"c2", 50, "books"},
    {"c1", 200, "electronics"},
    {"c3", 75, "books"},
    {"c2", 150, "clothing"},
}

byCustomer := GroupBy(orders, func(o Order) string { return o.CustomerID })
// {"c1": [...], "c2": [...], "c3": [...]}

byCategory := CountBy(orders, func(o Order) string { return o.Category })
// {"electronics": 2, "books": 2, "clothing": 1}

customers := Unique(Map(orders, func(o Order) string { return o.CustomerID }))
// ["c1", "c2", "c3"]
```

---

## 12. Go vs TypeScript Generics

### Side-by-Side Syntax Comparison

**Basic generic function:**

```go
// Go
func Identity[T any](val T) T {
    return val
}
result := Identity(42)
result := Identity[string]("hello")
```

```typescript
// TypeScript
function identity<T>(val: T): T {
    return val;
}
const result = identity(42);
const result = identity<string>("hello");
```

**Constrained generic:**

```go
// Go -- constraints use interfaces with type sets
type Number interface {
    ~int | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

```typescript
// TypeScript -- constraints use extends
function sum<T extends number>(nums: T[]): T {
    return nums.reduce((acc, n) => (acc + n) as T, 0 as T);
}
```

**Generic struct/interface:**

```go
// Go
type Box[T any] struct {
    Value T
}

func (b Box[T]) Get() T {
    return b.Value
}
```

```typescript
// TypeScript
interface Box<T> {
    value: T;
}

class ConcreteBox<T> implements Box<T> {
    constructor(public value: T) {}
    get(): T {
        return this.value;
    }
}
```

### Key Philosophical Differences

| Aspect | Go | TypeScript |
|---|---|---|
| **Syntax** | `[T constraint]` | `<T extends Type>` |
| **Constraint system** | Interfaces with type sets | `extends` keyword with types |
| **Underlying type matching** | `~int` matches `type MyInt int` | No equivalent -- structural typing handles this |
| **Type erasure** | No -- monomorphized (specialized code per type) at compile time via dictionary or stenciling | Yes -- generics erased at compile time (JS has no generics) |
| **Generic methods** | Not allowed (methods cannot introduce new type params) | Fully allowed |
| **Conditional types** | Not available | `T extends U ? X : Y` -- extremely powerful |
| **Mapped types** | Not available | `{ [K in keyof T]: ... }` |
| **Variance annotations** | Implicit | Explicit `in`/`out` keywords (TS 4.7+) |
| **Default type params** | Not available | `<T = string>` |
| **Runtime type info** | Available via `any(val).(type)` workaround | Erased -- no runtime generic info |

### The `~` Operator -- Go's Unique Feature

Go's `~` operator has no TypeScript equivalent because TypeScript uses structural typing:

```go
// Go: ~ matches underlying types
type Celsius float64
type Fahrenheit float64

type Temperature interface {
    ~float64
}

// Both Celsius and Fahrenheit satisfy Temperature
func Average[T Temperature](temps []T) T {
    var sum T
    for _, t := range temps {
        sum += t
    }
    return sum / T(len(temps))
}
```

```typescript
// TypeScript: structural typing -- Celsius and Fahrenheit are just numbers
type Celsius = number;      // type alias, not a distinct type
type Fahrenheit = number;   // same -- both are just number

// No need for ~ because there's no nominal typing to work around
function average(temps: number[]): number {
    return temps.reduce((a, b) => a + b, 0) / temps.length;
}
```

Go is a **nominally typed** language (types are distinct by name). TypeScript is **structurally typed** (types are compatible if shapes match). The `~` operator bridges Go's nominal system for generics. TypeScript never needs this bridge.

### TypeScript Has Far More Generic Power

TypeScript's generic system is essentially a type-level programming language:

```typescript
// Conditional types -- Go cannot do this
type IsString<T> = T extends string ? true : false;
type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Mapped types -- Go cannot do this
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Partial<T> = { [K in keyof T]?: T[K] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };

// Template literal types -- Go cannot do this
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<"click">; // "onClick"

// Infer keyword -- Go cannot do this
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
```

Go's generics are deliberately simpler. The Go team's position: complex type-level computation makes code harder to understand and debug. Go optimizes for readability at the cost of expressiveness.

### Pre-Generics Workarounds: Go vs JavaScript

JavaScript never "needed" generics because it has no static types:

```javascript
// JavaScript: dynamic typing means everything is generic by default
function contains(slice, target) {
    return slice.includes(target);
}

contains([1, 2, 3], 4);          // works
contains(["a", "b"], "c");       // works
contains([1, "mixed", true], 1); // works (but questionable)
```

Go needed generics because its static type system demanded explicit types. JavaScript's dynamic typing is the ultimate "generic" -- everything works on everything, at the cost of no compile-time guarantees.

TypeScript added generics precisely to bring Go-like type safety to JavaScript's dynamic world. The irony: TypeScript had generics from its very first release (2012), while Go -- the statically typed language that arguably needed them more -- waited until 2022.

### "Earn Your Complexity" vs "Full Power From Day 1"

Go's philosophy is incremental complexity. Start with concrete types. Copy-paste once or twice. Only reach for generics when the repetition becomes clear and painful. The language makes generics possible but does not make them the default tool.

TypeScript's philosophy is expressive power. The generic system is available from the start, deeply integrated into the type system, and used pervasively in the ecosystem. Libraries like `lodash`, `zod`, and `trpc` would be impossible to type without advanced generics.

Neither approach is wrong. They optimize for different things:

- **Go:** Large teams, long-lived codebases, readable-by-anyone code. Generics are a scalpel.
- **TypeScript:** Rich type inference, library authoring, DX-focused type safety. Generics are the foundation.

---

## 13. Key Takeaways

1. **Generics solve the "same logic, different types" problem.** Before Go 1.18, you had to copy-paste, use `interface{}`, or generate code. Now you write it once with type parameters.

2. **Constraints make generics safe.** The compiler knows exactly what operations are valid on `T` because the constraint declares them. `any` allows nothing; `comparable` allows `==`/`!=`; custom constraints allow operators or methods.

3. **The `~` operator is essential for real Go code.** Named types (`type UserID int`) are everywhere in Go. Without `~`, generic functions would reject them. `~int` means "int and any type defined as int."

4. **Type inference keeps generic code clean.** Most of the time, you do not write type arguments -- the compiler figures them out from the values you pass.

5. **The `slices`, `maps`, and `cmp` packages are your first stop.** Before writing generic utility functions, check if the standard library already has them. As of Go 1.21+, most common slice and map operations are covered.

6. **Start concrete, generalize later.** The Go team's #1 guideline. Do not write generic code until you have written the same concrete code at least twice and see the pattern.

7. **Generic methods do not exist.** You cannot add new type parameters to methods -- only to functions and type definitions. Use top-level generic functions as the workaround.

8. **Go's generics are deliberately simpler than TypeScript's.** No conditional types, no mapped types, no type-level programming. This is by design -- Go optimizes for code readability in large teams.

9. **Use interfaces when methods are the constraint.** If your generic function only calls methods on `T`, a regular interface parameter is simpler and achieves the same thing.

10. **Generics are not the answer to everything.** They shine for data structures, collection operations, and type-safe wrappers. They are overkill for application-level business logic that works with specific types.

---

## 14. Practice Exercises

### Exercise 1: Generic `Chunk` Function
Write a function that splits a slice into chunks of a given size.

```go
func Chunk[T any](slice []T, size int) [][]T {
    // Your implementation here
}

// Expected behavior:
// Chunk([]int{1,2,3,4,5}, 2) => [[1,2], [3,4], [5]]
// Chunk([]string{"a","b","c","d"}, 3) => [["a","b","c"], ["d"]]
```

### Exercise 2: Generic `Zip` Function
Write a function that combines two slices into a slice of pairs.

```go
func Zip[T, U any](a []T, b []U) []Pair[T, U] {
    // Your implementation here
    // Use the shorter slice's length
}

// Expected behavior:
// Zip([]int{1,2,3}, []string{"a","b","c"}) => [{1,"a"}, {2,"b"}, {3,"c"}]
// Zip([]int{1,2}, []string{"a","b","c"}) => [{1,"a"}, {2,"b"}]
```

### Exercise 3: Generic `Cache` with TTL
Build a generic cache that stores key-value pairs with an expiration time.

```go
type Cache[K comparable, V any] struct {
    // Your fields here
}

func NewCache[K comparable, V any](ttl time.Duration) *Cache[K, V]
func (c *Cache[K, V]) Set(key K, value V)
func (c *Cache[K, V]) Get(key K) (V, bool) // returns false if expired or missing
func (c *Cache[K, V]) Delete(key K)
func (c *Cache[K, V]) Cleanup()             // remove all expired entries
```

### Exercise 4: Before and After Refactor
Take this pre-generics code and refactor it to use generics:

```go
// Pre-generics: three nearly identical functions
func MaxInt(a, b int) int {
    if a > b { return a }
    return b
}

func MaxFloat64(a, b float64) float64 {
    if a > b { return a }
    return b
}

func MaxString(a, b string) string {
    if a > b { return a }
    return b
}

// Also refactor these:
func ReverseInts(s []int) []int { ... }
func ReverseStrings(s []string) []string { ... }
func ReverseFloat64s(s []float64) []float64 { ... }
```

### Exercise 5: Type-Safe Event Emitter
Build a generic event system where events carry typed payloads:

```go
type EventHandler[T any] func(payload T)

type EventEmitter[T any] struct {
    // Your fields here
}

func NewEventEmitter[T any]() *EventEmitter[T]
func (e *EventEmitter[T]) On(handler EventHandler[T])
func (e *EventEmitter[T]) Emit(payload T)
func (e *EventEmitter[T]) Off(handler EventHandler[T]) // bonus: remove handler

// Usage:
type UserCreated struct { Name string; Email string }
emitter := NewEventEmitter[UserCreated]()
emitter.On(func(event UserCreated) {
    fmt.Printf("Welcome, %s!\n", event.Name)
})
emitter.Emit(UserCreated{Name: "Alice", Email: "alice@example.com"})
```

### Exercise 6: Pipeline Pattern
Build a generic pipeline that chains transformation steps:

```go
type Pipeline[T any] struct {
    // Your fields here
}

func NewPipeline[T any]() *Pipeline[T]
func (p *Pipeline[T]) Then(step func(T) T) *Pipeline[T]
func (p *Pipeline[T]) Execute(input T) T

// Usage:
result := NewPipeline[string]().
    Then(strings.TrimSpace).
    Then(strings.ToLower).
    Then(func(s string) string { return strings.ReplaceAll(s, " ", "-") }).
    Execute("  Hello World  ")
// result: "hello-world"
```

### Exercise 7: Build Your Own `slices.SortFunc`
Implement a generic sort function that takes a comparison function:

```go
func SortFunc[T any](slice []T, cmp func(a, b T) int) {
    // Implement using any sorting algorithm (quicksort, mergesort, etc.)
    // cmp returns: negative if a < b, zero if a == b, positive if a > b
}

// Test with:
nums := []int{5, 3, 8, 1, 9, 2}
SortFunc(nums, func(a, b int) int { return a - b })
// nums: [1, 2, 3, 5, 8, 9]

people := []Person{{Name: "Charlie", Age: 30}, {Name: "Alice", Age: 25}}
SortFunc(people, func(a, b Person) int { return cmp.Compare(a.Name, b.Name) })
// [{Alice 25} {Charlie 30}]
```
