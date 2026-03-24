# Chapter 5: Arrays, Slices, and Maps

> **Level: Beginner** | Go's collection types are deceptively simple on the surface, but their internals -- especially slices -- are one of the most important things to deeply understand. This chapter takes you from zero to confident.

---

## Table of Contents

1. [Arrays: Fixed-Size Value Types](#1-arrays-fixed-size-value-types)
2. [Slices: Dynamic Arrays Built on Arrays](#2-slices-dynamic-arrays-built-on-arrays)
3. [Slice Internals: The Deep Dive](#3-slice-internals-the-deep-dive)
4. [Slice Operations](#4-slice-operations)
5. [Maps: Key-Value Collections](#5-maps-key-value-collections)
6. [Map Internals](#6-map-internals)
7. [Common Patterns: Filter, Map, Reduce](#7-common-patterns-filter-map-reduce)
8. [Strings, Bytes, and Runes](#8-strings-bytes-and-runes)
9. [Go vs Node.js Comparison Summary](#9-go-vs-nodejs-comparison-summary)
10. [Key Takeaways](#10-key-takeaways)
11. [Practice Exercises](#11-practice-exercises)

---

## 1. Arrays: Fixed-Size Value Types

### What Is an Array in Go?

An array in Go is a **fixed-length**, **ordered** sequence of elements of a single type. The length is part of the type itself -- `[3]int` and `[5]int` are completely different types.

```go
// Declaring arrays
var a [5]int                        // [0, 0, 0, 0, 0] -- zero-valued
b := [3]string{"go", "is", "fun"}  // literal initialization
c := [...]int{10, 20, 30}          // compiler counts: [3]int
```

### Arrays Are Value Types (This Is Huge)

In Go, arrays are **value types**. When you assign an array to a new variable or pass it to a function, the **entire array is copied**.

```go
package main

import "fmt"

func main() {
    original := [3]int{1, 2, 3}
    copied := original       // Full copy -- not a reference!

    copied[0] = 999

    fmt.Println(original)    // [1 2 3]   -- unchanged!
    fmt.Println(copied)      // [999 2 3] -- only the copy changed
}
```

#### Memory Layout of an Array

```
original (on stack):
+-----+-----+-----+
|  1  |  2  |  3  |
+-----+-----+-----+
  [0]   [1]   [2]

copied := original   -->  Full byte-for-byte copy

copied (on stack, separate memory):
+-----+-----+-----+
|  1  |  2  |  3  |
+-----+-----+-----+
  [0]   [1]   [2]
```

Every element lives in **contiguous memory**. No pointers, no indirection. This is great for cache performance but expensive for large arrays passed by value.

### WHY Are Arrays Value Types in Go?

This is a deliberate design decision:

1. **Simplicity and predictability.** Value semantics mean no surprise mutations. If you pass an array to a function, the function cannot change your original. You reason locally about code.

2. **Composability.** Because arrays are values, you can embed them in structs and the whole struct copies cleanly -- no hidden shared state.

3. **Performance control.** Go gives you the choice: pass by value for small arrays (safe, no allocation), pass a pointer for large arrays (efficient, explicit sharing). In JS, you have no choice -- arrays are always references.

4. **Equality.** You can compare arrays with `==` because they are values. You cannot do this with slices or maps.

```go
a := [3]int{1, 2, 3}
b := [3]int{1, 2, 3}
fmt.Println(a == b)  // true -- element-wise comparison
```

### Comparison: Go Arrays vs JS Arrays

```
 Aspect              | Go Array              | JS Array
─────────────────────┼───────────────────────┼──────────────────────────
 Size                | Fixed at compile time  | Dynamic (grows/shrinks)
 Type                | Value type             | Reference type (object)
 Assignment          | Full copy              | Copies reference (shared)
 Equality            | == works               | == checks reference, not content
 Length is part of   | The type ([3]int)      | Not the type
 type?               |                        |
 Heterogeneous?      | No (single type only)  | Yes (any mix of types)
 Zero value          | All elements zero-val  | undefined / empty
```

**JS equivalent of Go's copy behavior:**

```javascript
// Node.js -- arrays are reference types
const original = [1, 2, 3];
const copied = original;       // NOT a copy -- same reference!
copied[0] = 999;
console.log(original);         // [999, 2, 3]  <-- mutated!

// To actually copy in JS, you must be explicit:
const realCopy = [...original]; // spread operator
const realCopy2 = original.slice(); // or .slice()
```

In Go, copying is the **default**. In JS, sharing is the default. This is a fundamental philosophy difference.

### Passing Arrays to Functions

```go
func double(arr [3]int) [3]int {
    for i := range arr {
        arr[i] *= 2
    }
    return arr
}

func main() {
    nums := [3]int{1, 2, 3}
    result := double(nums)

    fmt.Println(nums)    // [1 2 3] -- original untouched (passed by value)
    fmt.Println(result)  // [2 4 6]
}
```

If you want to modify the original, pass a pointer:

```go
func doubleInPlace(arr *[3]int) {
    for i := range arr {
        arr[i] *= 2
    }
}

func main() {
    nums := [3]int{1, 2, 3}
    doubleInPlace(&nums)
    fmt.Println(nums) // [2 4 6] -- modified via pointer
}
```

> **In practice, you almost never use arrays directly in Go.** You use slices. Arrays exist primarily as the backing store for slices. Think of arrays as the implementation detail and slices as the interface.

---

## 2. Slices: Dynamic Arrays Built on Arrays

### What Is a Slice?

A slice is a **dynamically-sized**, flexible view into an underlying array. It is the workhorse collection type in Go -- when someone says "list" or "array" in Go, they almost always mean a slice.

```go
// Creating slices
s1 := []int{1, 2, 3}              // slice literal (no size in brackets)
s2 := make([]int, 5)              // make: length 5, capacity 5, all zeros
s3 := make([]int, 3, 10)          // make: length 3, capacity 10
var s4 []int                       // nil slice (length 0, capacity 0)
```

Notice the syntactic difference:
```go
[3]int{1, 2, 3}  // ARRAY  -- size in brackets
[]int{1, 2, 3}   // SLICE  -- no size in brackets
```

### WHY Does Go Separate Arrays and Slices?

This is a question every newcomer asks. The answer reveals Go's design philosophy:

1. **Arrays give you a concrete, fixed-size, value-type block of memory.** They are the raw building block. The compiler knows exactly how big they are.

2. **Slices give you a flexible, reference-like view over arrays.** They carry a pointer, length, and capacity -- a tiny header that is cheap to copy.

3. **Separation of concerns.** The array handles storage. The slice handles access patterns. This lets multiple slices share one array (powerful but requires understanding).

4. **No hidden allocations.** In JS, pushing to an array might allocate behind the scenes and you never know. In Go, `append()` returns a new slice explicitly when capacity is exceeded. You always know when allocations happen.

### The Three Core Slice Operations

#### `make()` -- Create a Slice with Specific Length and Capacity

```go
// make([]T, length, capacity)
s := make([]int, 3, 10)
fmt.Println(len(s))  // 3
fmt.Println(cap(s))  // 10
fmt.Println(s)       // [0 0 0]
```

- `length`: how many elements are currently accessible
- `capacity`: how many elements the underlying array can hold before needing to grow

If you omit capacity, it defaults to length:
```go
s := make([]int, 5) // length=5, capacity=5
```

#### `append()` -- Add Elements to a Slice

```go
s := []int{1, 2, 3}
s = append(s, 4)          // append one element
s = append(s, 5, 6, 7)    // append multiple elements
s = append(s, []int{8, 9}...) // append another slice (spread with ...)

fmt.Println(s) // [1 2 3 4 5 6 7 8 9]
```

**Critical rule: always reassign the result of `append`.** `append` may return a new slice header pointing to a new underlying array if capacity was exceeded.

```go
s := make([]int, 0, 3)
s = append(s, 1) // fits in capacity -- same underlying array
s = append(s, 2) // fits
s = append(s, 3) // fits
s = append(s, 4) // EXCEEDS capacity -- new array allocated, data copied
```

#### `copy()` -- Copy Elements Between Slices

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)

n := copy(dst, src)      // copies min(len(dst), len(src)) elements
fmt.Println(dst)         // [1 2 3]
fmt.Println(n)           // 3 (number of elements copied)
```

`copy` is the safe way to duplicate a slice's data without sharing the underlying array.

```go
// Deep copy pattern
original := []int{1, 2, 3}
clone := make([]int, len(original))
copy(clone, original)

clone[0] = 999
fmt.Println(original) // [1 2 3] -- safe, independent copy
```

### Comparison: Go Slices vs JS Arrays

```javascript
// Node.js
const arr = [1, 2, 3];
arr.push(4);           // Go: s = append(s, 4)
arr.length;            // Go: len(s)
arr.slice(1, 3);       // Go: s[1:3]
[...arr];              // Go: copy into new slice
```

Key difference: JS `push()` mutates in place and never returns a new array. Go `append()` might return a completely new slice.

---

## 3. Slice Internals: The Deep Dive

This section is the most important in the entire chapter. Understanding slice internals prevents an entire class of bugs.

### The Slice Header

A slice is a **3-word struct** (24 bytes on 64-bit systems):

```go
// This is the runtime representation (you don't write this yourself)
type slice struct {
    ptr *array   // pointer to the underlying array
    len int      // number of elements currently in the slice
    cap int      // total capacity of the underlying array from ptr
}
```

#### Visual: Slice Header and Underlying Array

```
Slice Header (24 bytes on 64-bit):
+----------+----------+----------+
|   ptr    |   len    |   cap    |
| 0xC000.. |    3     |    5     |
+----------+----------+----------+
     |
     v
Underlying Array (contiguous memory):
+-----+-----+-----+-----+-----+
|  10 |  20 |  30 |  0  |  0  |
+-----+-----+-----+-----+-----+
  [0]   [1]   [2]   [3]   [4]
  ^                 ^           ^
  |                 |           |
  ptr            len=3        cap=5
```

```go
s := make([]int, 3, 5)
s[0], s[1], s[2] = 10, 20, 30

fmt.Println(len(s))  // 3 -- accessible elements
fmt.Println(cap(s))  // 5 -- room for 2 more before reallocation
```

### How `append` Works Internally

When you call `append(s, elem)`, the runtime checks:

1. **Is `len(s) < cap(s)`?** If yes, write the element at index `len`, increment `len`, return the updated header. Same underlying array.

2. **Is `len(s) == cap(s)`?** If yes, allocate a **new, larger array**, copy all existing elements, write the new element, return a new header pointing to the new array.

#### Growth Strategy

The growth factor is approximately:
- **Double** the capacity when cap < 256
- **Grow by ~25%** (cap + cap/4 + 192) when cap >= 256

(The exact formula changed in Go 1.18 to smooth the transition.)

```go
s := make([]int, 0)
for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}
// Output:
// len=1  cap=1
// len=2  cap=2
// len=3  cap=4    <-- doubled from 2
// len=4  cap=4
// len=5  cap=8    <-- doubled from 4
// len=6  cap=8
// len=7  cap=8
// len=8  cap=8
// len=9  cap=16   <-- doubled from 8
// len=10 cap=16
```

#### Visual: append with Reallocation

```
BEFORE append (len=3, cap=3):
Header: [ptr=0xA000 | len=3 | cap=3]
            |
            v
        +---+---+---+
        | 1 | 2 | 3 |  <-- array at 0xA000 (full!)
        +---+---+---+

s = append(s, 4)  // No room! Must reallocate.

AFTER append (len=4, cap=6):
Header: [ptr=0xB000 | len=4 | cap=6]    <-- NEW pointer!
            |
            v
        +---+---+---+---+---+---+
        | 1 | 2 | 3 | 4 | 0 | 0 |  <-- NEW array at 0xB000
        +---+---+---+---+---+---+

Old array at 0xA000 is now eligible for garbage collection
(unless another slice still references it!)
```

### Shared Underlying Arrays (The Big Gotcha)

When you create a slice from another slice (or from an array), they **share the same underlying array**. Mutations through one slice are visible through the other.

```go
package main

import "fmt"

func main() {
    original := []int{1, 2, 3, 4, 5}
    sub := original[1:3]   // sub = [2, 3]

    fmt.Println(sub)       // [2 3]

    sub[0] = 999

    fmt.Println(original)  // [1 999 3 4 5]  <-- MUTATED!
    fmt.Println(sub)       // [999 3]
}
```

#### Visual: Shared Underlying Array

```
original: [ptr=0xA000 | len=5 | cap=5]
               |
               v
           +---+-----+---+---+---+
           | 1 | 999 | 3 | 4 | 5 |
           +---+-----+---+---+---+
            [0]  [1]  [2] [3] [4]
                  ^
                  |
sub:       [ptr=0xA008 | len=2 | cap=4]
           (ptr points to original[1])

sub[0] modifies the byte at 0xA008, which is original[1].
Both slices see the change because they share the same memory.
```

### The Append + Shared Array Gotcha

This is one of Go's most notorious gotchas:

```go
func main() {
    original := []int{1, 2, 3, 4, 5}
    sub := original[1:3]   // sub = [2, 3], len=2, cap=4

    // sub has capacity 4 (from index 1 to end of original)
    // Appending fits within capacity -- NO new array allocated!
    sub = append(sub, 99)

    fmt.Println(original)  // [1 2 3 99 5]  <-- original[3] was overwritten!
    fmt.Println(sub)       // [2 3 99]
}
```

#### Visual: The Gotcha

```
BEFORE append:
original: [ptr=0xA000 | len=5 | cap=5]
               |
               v
           +---+---+---+---+---+
           | 1 | 2 | 3 | 4 | 5 |
           +---+---+---+---+---+

sub:       [ptr=0xA008 | len=2 | cap=4]
                               ^       ^
                            sub can write here (cap=4)

AFTER sub = append(sub, 99):
sub:       [ptr=0xA008 | len=3 | cap=4]
               |
               v (still same underlying array!)
           +---+---+---+----+---+
           | 1 | 2 | 3 | 99 | 5 |   <-- original[3] = 99 !!!
           +---+---+---+----+---+
```

**The fix: use a full slice expression** (three-index slice) to limit capacity:

```go
sub := original[1:3:3]  // len=2, cap=2 (cap is limited to 3-1=2)
sub = append(sub, 99)   // Now this MUST allocate a new array
fmt.Println(original)   // [1 2 3 4 5] -- safe!
```

### Gotcha Summary Table

```
 Gotcha                             | Symptom                        | Fix
────────────────────────────────────┼────────────────────────────────┼──────────────────────────
 Forgetting to reassign append      | Lost appended data             | Always: s = append(s, x)
 Shared underlying array mutation   | Modifying one slice mutates    | Use copy() or 3-index
                                    | another                        | slice [low:high:max]
 Append through sub-slice           | Overwrites original's data     | Limit cap with 3-index
 Nil slice vs empty slice           | Unexpected JSON serialization  | Use make([]T, 0) or []T{}
 Passing slice to goroutine in loop | Race condition / wrong values  | Capture loop variable
```

---

## 4. Slice Operations

### Slicing Syntax: `[low:high:max]`

Go supports two forms of slicing:

```go
s := []int{0, 10, 20, 30, 40, 50}

// Two-index slice: s[low:high]
a := s[1:4]       // [10, 20, 30]    len=3, cap=5

// Three-index slice: s[low:high:max]
b := s[1:4:4]     // [10, 20, 30]    len=3, cap=3 (max-low)
```

#### How Indices Work

```
s = [0, 10, 20, 30, 40, 50]     len=6, cap=6
     [0] [1] [2]  [3]  [4] [5]

s[1:4] -->
     Elements from index 1 up to (not including) index 4
     Result: [10, 20, 30]
     len = high - low = 4 - 1 = 3
     cap = cap(s) - low = 6 - 1 = 5

s[1:4:4] -->
     Same elements, but max caps the capacity
     len = high - low = 4 - 1 = 3
     cap = max - low  = 4 - 1 = 3
```

#### Default Values

```go
s := []int{0, 10, 20, 30, 40}

s[:]     // same as s[0:len(s)]    -- full slice
s[2:]    // same as s[2:len(s)]    -- from index 2 to end
s[:3]    // same as s[0:3]         -- from start to index 3
```

### Comparison: Go Slice vs JS Array.slice()

```go
// Go
s := []int{0, 10, 20, 30, 40}
sub := s[1:4]    // [10, 20, 30]

// SHARES underlying array -- mutations are visible!
sub[0] = 999
fmt.Println(s)    // [0, 999, 20, 30, 40]
```

```javascript
// Node.js
const s = [0, 10, 20, 30, 40];
const sub = s.slice(1, 4);  // [10, 20, 30]

// INDEPENDENT copy -- mutations are NOT visible
sub[0] = 999;
console.log(s);   // [0, 10, 20, 30, 40]  -- unchanged
```

**Key difference: Go slicing creates a view (shared memory). JS `slice()` creates a copy.**

### Nil Slices vs Empty Slices

```go
var nilSlice []int          // nil slice
emptySlice := []int{}       // empty slice
madeSlice := make([]int, 0) // also empty slice

fmt.Println(nilSlice == nil)    // true
fmt.Println(emptySlice == nil)  // false
fmt.Println(madeSlice == nil)   // false

// BUT -- they behave identically for len, cap, append, range:
fmt.Println(len(nilSlice))      // 0
fmt.Println(cap(nilSlice))      // 0
fmt.Println(len(emptySlice))    // 0

nilSlice = append(nilSlice, 1)  // works fine!

for _, v := range nilSlice {    // no panic, just doesn't iterate
    fmt.Println(v)
}
```

#### When It Matters: JSON Serialization

```go
import "encoding/json"

var nilSlice []int
emptySlice := []int{}

nilJSON, _ := json.Marshal(nilSlice)
emptyJSON, _ := json.Marshal(emptySlice)

fmt.Println(string(nilJSON))    // null     <-- might break API clients!
fmt.Println(string(emptyJSON))  // []       <-- what clients usually expect
```

**Rule of thumb:** Use `make([]T, 0)` or `[]T{}` when the result will be serialized. Use `var s []T` (nil) when it is an internal-only variable.

### Multi-Dimensional Slices

Go does not have true multi-dimensional arrays like NumPy. You create slices of slices:

```go
// Create a 3x4 matrix
rows := 3
cols := 4
matrix := make([][]int, rows)
for i := range matrix {
    matrix[i] = make([]int, cols)
}

matrix[1][2] = 42
fmt.Println(matrix)
// [[0 0 0 0] [0 0 42 0] [0 0 0 0]]
```

#### Memory Layout (Slice of Slices)

```
matrix ([][]int):
+--------+--------+--------+
| header | header | header |
|  [0]   |  [1]   |  [2]   |
+--------+--------+--------+
    |         |         |
    v         v         v
 +--+--+--+--+  +--+--+--+--+  +--+--+--+--+
 |0 |0 |0 |0 |  |0 |0 |42|0 |  |0 |0 |0 |0 |
 +--+--+--+--+  +--+--+--+--+  +--+--+--+--+

Each row is a separate allocation. Rows are NOT necessarily
contiguous in memory (unlike C 2D arrays).
```

### Deleting Elements from a Slice

Go has no built-in `delete` for slices (unlike maps). You do it manually:

```go
// Delete element at index i (preserving order)
s := []int{10, 20, 30, 40, 50}
i := 2
s = append(s[:i], s[i+1:]...)
fmt.Println(s)  // [10 20 40 50]

// Delete element at index i (NOT preserving order -- faster)
s2 := []int{10, 20, 30, 40, 50}
i = 2
s2[i] = s2[len(s2)-1]   // overwrite with last element
s2 = s2[:len(s2)-1]      // shrink
fmt.Println(s2)           // [10 20 50 40]
```

**Go 1.21+ introduced `slices.Delete` in the standard library:**

```go
import "slices"

s := []int{10, 20, 30, 40, 50}
s = slices.Delete(s, 2, 3)  // delete elements from index 2 to 3
fmt.Println(s)               // [10 20 40 50]
```

### Iterating with `range`

```go
fruits := []string{"apple", "banana", "cherry"}

// Both index and value
for i, fruit := range fruits {
    fmt.Printf("%d: %s\n", i, fruit)
}

// Value only
for _, fruit := range fruits {
    fmt.Println(fruit)
}

// Index only
for i := range fruits {
    fmt.Println(i)
}
```

#### Comparison: Go `range` vs JS iteration

```javascript
// Node.js equivalents
const fruits = ["apple", "banana", "cherry"];

// for...of (like range with _ index)
for (const fruit of fruits) {
    console.log(fruit);
}

// forEach (like range with both index and value)
fruits.forEach((fruit, i) => {
    console.log(`${i}: ${fruit}`);
});

// for...in (iterates keys as strings -- usually avoid for arrays)
for (const i in fruits) {
    console.log(i); // "0", "1", "2" (strings, not numbers!)
}
```

---

## 5. Maps: Key-Value Collections

### Declaration and Initialization

```go
// Method 1: var declaration (nil map -- READ is ok, WRITE panics)
var m1 map[string]int
fmt.Println(m1 == nil)     // true
fmt.Println(m1["missing"]) // 0 (zero value, no panic)
// m1["key"] = 1           // PANIC: assignment to entry in nil map

// Method 2: make (initialized, ready to use)
m2 := make(map[string]int)
m2["key"] = 1              // works fine

// Method 3: literal
m3 := map[string]int{
    "alice": 95,
    "bob":   87,
    "carol": 92,  // trailing comma required
}
```

### CRUD Operations

```go
m := make(map[string]int)

// CREATE / UPDATE
m["alice"] = 95     // create
m["alice"] = 100    // update (same syntax)

// READ
score := m["alice"]
fmt.Println(score)  // 100

// READ (missing key returns zero value!)
missing := m["nobody"]
fmt.Println(missing) // 0 -- but was it 0 or missing?

// DELETE
delete(m, "alice")
fmt.Println(m["alice"]) // 0 (gone)

// SIZE
fmt.Println(len(m))
```

### The Comma-Ok Idiom (Essential)

Since missing keys return the zero value, how do you distinguish "key exists with value 0" from "key doesn't exist"? The comma-ok idiom:

```go
m := map[string]int{
    "alice": 95,
    "bob":   0,    // Bob has a score of zero
}

// The comma-ok pattern
score, ok := m["bob"]
fmt.Println(score, ok)  // 0 true  -- exists with value 0

score, ok = m["nobody"]
fmt.Println(score, ok)  // 0 false -- does not exist

// Common pattern: check and act
if score, ok := m["alice"]; ok {
    fmt.Printf("Alice scored %d\n", score)
} else {
    fmt.Println("Alice not found")
}
```

### Comparison: Go Comma-Ok vs JS

```javascript
// Node.js -- checking existence
const m = { alice: 95, bob: 0 };

// Problem: falsy values
if (m["bob"]) {
    // This does NOT run because 0 is falsy!
}

// Solution 1: "in" operator
if ("bob" in m) {
    console.log(m["bob"]); // 0
}

// Solution 2: hasOwnProperty
if (m.hasOwnProperty("bob")) { ... }

// Solution 3: optional chaining (different use case)
console.log(m?.bob ?? "default");

// JS Map (closer to Go maps)
const jsMap = new Map();
jsMap.set("bob", 0);
console.log(jsMap.has("bob")); // true
console.log(jsMap.get("bob")); // 0
```

Go's comma-ok idiom is more elegant than JS's various workarounds for falsy values.

### Iterating Over Maps

```go
m := map[string]int{"alice": 95, "bob": 87, "carol": 92}

// Key and value
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// Keys only
for key := range m {
    fmt.Println(key)
}
```

**Important: Map iteration order is NOT guaranteed.** In fact, Go deliberately randomizes it (more on this in section 6).

### Maps with Slice Values

```go
// Grouping: map of string to slice of strings
groups := make(map[string][]string)

groups["fruit"] = append(groups["fruit"], "apple")
groups["fruit"] = append(groups["fruit"], "banana")
groups["veggie"] = append(groups["veggie"], "carrot")

fmt.Println(groups)
// map[fruit:[apple banana] veggie:[carrot]]
```

This works because `groups["fruit"]` returns the zero value (`nil` slice) if the key doesn't exist, and `append` handles nil slices gracefully.

### Map Key Constraints

Map keys must be **comparable** (support `==`). This means:

```
 Type            | Can be a map key?
─────────────────┼──────────────────
 int, float, etc | Yes
 string          | Yes
 bool            | Yes
 pointer         | Yes
 struct          | Yes (if all fields are comparable)
 array           | Yes (fixed size, comparable)
 slice           | NO
 map             | NO
 function        | NO
```

```go
// Struct as map key (useful for composite keys)
type Point struct {
    X, Y int
}

visited := make(map[Point]bool)
visited[Point{1, 2}] = true
visited[Point{3, 4}] = true

fmt.Println(visited[Point{1, 2}]) // true
```

---

## 6. Map Internals

### Hash Map Implementation

Go maps are **hash tables**. Internally:

```
map[string]int

+-------------------+
|  Hash Function    |
+-------------------+
         |
         v
+--------+--------+--------+--------+
| Bucket | Bucket | Bucket | Bucket |  ...
|   0    |   1    |   2    |   3    |
+--------+--------+--------+--------+
    |
    v
+------+-------+------+-------+
| key1 | val1  | key2 | val2  |  (up to 8 key-value pairs per bucket)
+------+-------+------+-------+
    |
    v
  overflow bucket (linked list if needed)
```

Each bucket holds up to 8 key-value pairs. When the load factor exceeds a threshold, the map grows by allocating a new, larger bucket array and rehashing entries incrementally.

### WHY Go Randomizes Iteration Order

If you run this code multiple times:

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k, v := range m {
    fmt.Printf("%s:%d ", k, v)
}
```

You will get a **different order each time**. This is intentional. Here is why:

1. **Prevent accidental dependencies.** If iteration were deterministic, programmers would write code that accidentally depends on a specific order. Then, a Go version upgrade changes the hash function, and everything breaks. By randomizing from the start, no one can depend on order.

2. **Security.** Deterministic hash ordering has been exploited for denial-of-service attacks (hash collision attacks). Randomized iteration makes this harder.

3. **Honest API contract.** The spec says "unordered." Randomization enforces this contract rather than providing a false sense of order.

#### Comparison: JS Object vs JS Map Ordering

```javascript
// JS Objects: insertion order for string keys (since ES2015)
const obj = {};
obj["c"] = 3;
obj["a"] = 1;
obj["b"] = 2;
console.log(Object.keys(obj)); // ["c", "a", "b"] -- insertion order

// BUT: integer keys are sorted numerically first!
const obj2 = {};
obj2["3"] = "c";
obj2["1"] = "a";
obj2["2"] = "b";
console.log(Object.keys(obj2)); // ["1", "2", "3"] -- sorted!

// JS Map: guaranteed insertion order
const jsMap = new Map();
jsMap.set("c", 3);
jsMap.set("a", 1);
jsMap.set("b", 2);
for (const [k, v] of jsMap) {
    console.log(k, v);  // Always: c 3, a 1, b 2
}
```

Go maps: no ordering guarantees.
JS objects: insertion order (mostly).
JS Map: strict insertion order.

If you need ordered keys in Go, sort them yourself:

```go
import "sort"

m := map[string]int{"banana": 2, "apple": 1, "cherry": 3}

keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
    fmt.Printf("%s: %d\n", k, m[k])
}
// apple: 1
// banana: 2
// cherry: 3
```

### Concurrent Access Issues

**Maps are NOT safe for concurrent use.** If you read and write a map from multiple goroutines without synchronization, your program will crash with a fatal error (not a panic -- unrecoverable).

```go
// THIS WILL CRASH
m := make(map[int]int)

go func() {
    for i := 0; i < 1000; i++ {
        m[i] = i  // concurrent write
    }
}()

go func() {
    for i := 0; i < 1000; i++ {
        _ = m[i]   // concurrent read
    }
}()

// fatal error: concurrent map read and map write
```

Go added a **race detector** at runtime specifically for this. Build with `-race` to detect it:

```bash
go run -race main.go
```

#### Solutions for Concurrent Map Access

**Option 1: `sync.Mutex`**

```go
import "sync"

type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}
```

**Option 2: `sync.Map` (built-in)**

```go
import "sync"

var m sync.Map

m.Store("alice", 95)
value, ok := m.Load("alice")
m.Delete("alice")
m.Range(func(key, value any) bool {
    fmt.Println(key, value)
    return true // continue iteration
})
```

`sync.Map` is optimized for two patterns: (1) keys written once, read many times, and (2) many goroutines each reading/writing disjoint sets of keys. For other patterns, a `sync.RWMutex` with a regular map is typically faster.

---

## 7. Common Patterns: Filter, Map, Reduce

### The Elephant in the Room

Go (before generics) had **no built-in `filter`, `map`, or `reduce`** functions. Coming from JS, this feels like a shock. The Go philosophy is: a simple `for` loop is clear, efficient, and not that much more code.

### Pre-Generics: Manual Loops (Go < 1.18)

```go
// FILTER: keep even numbers
func filterEvens(nums []int) []int {
    result := make([]int, 0, len(nums))
    for _, n := range nums {
        if n%2 == 0 {
            result = append(result, n)
        }
    }
    return result
}

// MAP: double every number
func doubleAll(nums []int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = n * 2
    }
    return result
}

// REDUCE: sum all numbers
func sum(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
```

### With Generics (Go 1.18+)

```go
// Generic Filter
func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0)
    for _, item := range slice {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result
}

// Generic Map
func Map[T any, U any](slice []T, transform func(T) U) []U {
    result := make([]U, len(slice))
    for i, item := range slice {
        result[i] = transform(item)
    }
    return result
}

// Generic Reduce
func Reduce[T any, U any](slice []T, initial U, reducer func(U, T) U) U {
    acc := initial
    for _, item := range slice {
        acc = reducer(acc, item)
    }
    return acc
}

// Usage
func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    evens := Filter(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println(evens)   // [2 4 6 8 10]

    doubled := Map(nums, func(n int) int { return n * 2 })
    fmt.Println(doubled) // [2 4 6 8 10 12 14 16 18 20]

    total := Reduce(nums, 0, func(acc, n int) int { return acc + n })
    fmt.Println(total)   // 55

    // Chaining (not as clean as JS, but works)
    result := Reduce(
        Filter(nums, func(n int) bool { return n%2 == 0 }),
        0,
        func(acc, n int) int { return acc + n },
    )
    fmt.Println(result) // 30 (sum of evens)
}
```

### Go 1.21+ Standard Library: `slices` and `maps` Packages

```go
import (
    "fmt"
    "slices"
    "maps"
)

func main() {
    // slices package
    s := []int{3, 1, 4, 1, 5, 9}

    slices.Sort(s)
    fmt.Println(s)                          // [1 1 3 4 5 9]

    fmt.Println(slices.Contains(s, 4))      // true
    fmt.Println(slices.Index(s, 4))         // 3

    slices.Reverse(s)
    fmt.Println(s)                          // [9 5 4 3 1 1]

    compact := slices.Compact([]int{1, 1, 2, 2, 3})
    fmt.Println(compact)                    // [1 2 3]

    // maps package
    m := map[string]int{"a": 1, "b": 2, "c": 3}
    keys := maps.Keys(m)     // returns an iterator (Go 1.23+)
    vals := maps.Values(m)   // returns an iterator (Go 1.23+)

    fmt.Println(slices.Sorted(keys))   // [a b c]
    fmt.Println(slices.Sorted(vals))   // this won't compile for int;
                                        // use slices.Collect + slices.Sort

    maps.DeleteFunc(m, func(k string, v int) bool {
        return v < 2
    })
    fmt.Println(m)                     // map[b:2 c:3]
}
```

### Side-by-Side: Go vs Node.js

```go
// Go
nums := []int{1, 2, 3, 4, 5}

// Filter
evens := Filter(nums, func(n int) bool { return n%2 == 0 })

// Map
doubled := Map(nums, func(n int) int { return n * 2 })

// Reduce
sum := Reduce(nums, 0, func(acc, n int) int { return acc + n })

// Chain
result := Reduce(
    Map(
        Filter(nums, func(n int) bool { return n%2 == 0 }),
        func(n int) int { return n * 2 },
    ),
    0,
    func(acc, n int) int { return acc + n },
)
```

```javascript
// Node.js
const nums = [1, 2, 3, 4, 5];

// Filter
const evens = nums.filter(n => n % 2 === 0);

// Map
const doubled = nums.map(n => n * 2);

// Reduce
const sum = nums.reduce((acc, n) => acc + n, 0);

// Chain (elegant!)
const result = nums
    .filter(n => n % 2 === 0)
    .map(n => n * 2)
    .reduce((acc, n) => acc + n, 0);
```

JS wins on ergonomics here. Go's approach is more verbose but compiles to more predictable, efficient machine code with no runtime overhead from method dispatch or closure allocation on the heap.

---

## 8. Strings, Bytes, and Runes

### Strings Are Read-Only Byte Slices

In Go, a string is internally a **read-only slice of bytes**:

```go
// Internal representation (simplified)
type string struct {
    ptr *byte
    len int
}
```

```go
s := "Hello"
fmt.Println(len(s))    // 5 (bytes, not characters!)
fmt.Println(s[0])      // 72 (the byte value of 'H')
fmt.Printf("%c\n", s[0]) // H

// Strings are immutable -- this won't compile:
// s[0] = 'h'  // compile error: cannot assign to s[0]
```

### The Unicode Problem

```go
s := "Hello, World!"   // ASCII: 1 byte per character, len = 13
fmt.Println(len(s))     // 13

s2 := "Hello, world!"  // still ASCII
fmt.Println(len(s2))    // 13

s3 := "Hello, Cafes?"   // with accent... just kidding, let's use real Unicode
emoji := "Go is fun! 🎉"
fmt.Println(len(emoji))  // 14 (not 11!) -- emoji is 4 bytes in UTF-8

japanese := "こんにちは"
fmt.Println(len(japanese)) // 15 (not 5!) -- each character is 3 bytes in UTF-8
```

#### Why `len()` Returns Bytes, Not Characters

Go strings are sequences of **bytes**, and Go uses **UTF-8** encoding (designed by Ken Thompson and Rob Pike, who also created Go). In UTF-8:
- ASCII characters (English letters, digits) = 1 byte each
- Accented characters (e, n) = 2 bytes each
- CJK characters (Chinese, Japanese, Korean) = 3 bytes each
- Emojis = 4 bytes each

### Three Ways to View a String

```go
s := "cafe\u0301"  // "cafe" + combining accent = "cafe" (5 code points, 6 bytes)

// 1. As bytes ([]byte)
fmt.Println([]byte(s))  // [99 97 102 101 204 129]  -- 6 bytes
fmt.Println(len(s))     // 6

// 2. As runes ([]rune) -- Unicode code points
fmt.Println([]rune(s))           // [99 97 102 101 769]  -- 5 runes
fmt.Println(len([]rune(s)))      // 5

// 3. As grapheme clusters (what humans perceive as characters)
// Go's standard library doesn't handle this directly.
// Need golang.org/x/text for proper grapheme segmentation.
```

### `[]byte` vs `[]rune` vs `string`

```
 Type      | What it represents         | When to use
───────────┼────────────────────────────┼─────────────────────────────────
 string    | Immutable byte sequence    | Default for text. Pass around,
           |                            | compare, use as map keys.
 []byte    | Mutable byte sequence      | I/O operations, building strings
           |                            | efficiently, binary data.
 []rune    | Mutable Unicode code point | When you need to manipulate
           | sequence                   | individual characters.
```

### Converting Between Them

```go
s := "Hello, world!"

// string -> []byte (copies the data)
b := []byte(s)
b[0] = 'h'
fmt.Println(string(b))  // "hello, world!"
fmt.Println(s)           // "Hello, world!" -- original unchanged (copy was made)

// string -> []rune (copies and decodes UTF-8)
r := []rune(s)
fmt.Println(len(r))     // 13 (rune count)

// []byte -> string
s2 := string([]byte{72, 101, 108, 108, 111})
fmt.Println(s2)  // "Hello"

// []rune -> string
s3 := string([]rune{72, 101, 108, 108, 111})
fmt.Println(s3)  // "Hello"
```

### Iterating Over Strings

```go
s := "Go is 🔥"

// Byte iteration (usually wrong for text)
for i := 0; i < len(s); i++ {
    fmt.Printf("byte[%d] = %x\n", i, s[i])
}
// byte[0] = 47  (G)
// byte[1] = 6f  (o)
// ...
// byte[6] = f0  (first byte of 🔥)
// byte[7] = 9f  (second byte of 🔥)
// byte[8] = 94  (third byte of 🔥)
// byte[9] = a5  (fourth byte of 🔥)

// Rune iteration (usually correct for text)
for i, r := range s {
    fmt.Printf("rune at byte %d = %c (U+%04X)\n", i, r, r)
}
// rune at byte 0 = G (U+0047)
// rune at byte 1 = o (U+006F)
// rune at byte 2 =   (U+0020)
// rune at byte 3 = i (U+0069)
// rune at byte 4 = s (U+0073)
// rune at byte 5 =   (U+0020)
// rune at byte 6 = 🔥 (U+1F525)   <-- note: next index is 10, not 7
```

**`range` over a string decodes UTF-8 runes automatically.** The index `i` is the byte position, and `r` is the decoded rune.

### Efficient String Building

String concatenation with `+` creates a new string each time (strings are immutable). For building strings in a loop, use `strings.Builder`:

```go
import "strings"

// BAD: O(n^2) -- each += allocates a new string
func buildBad(words []string) string {
    result := ""
    for _, w := range words {
        result += w + " "
    }
    return result
}

// GOOD: O(n) -- writes to an internal buffer
func buildGood(words []string) string {
    var b strings.Builder
    for _, w := range words {
        b.WriteString(w)
        b.WriteByte(' ')
    }
    return b.String()
}

// ALSO GOOD for simple joining:
result := strings.Join(words, " ")
```

### Comparison: Go Strings vs JS Strings

```
 Aspect                | Go                          | JS
────────────────────────┼─────────────────────────────┼──────────────────────────
 Encoding              | UTF-8 (byte-based)          | UTF-16 internally
 Immutable?            | Yes                         | Yes
 len()                 | Byte count                  | UTF-16 code unit count
 Indexing s[i]         | Returns a byte              | Returns a string (1 char)
 Character type        | rune (int32)                | No separate char type
 Equality              | == compares bytes           | === compares code units
 Iteration             | range = runes; index = bytes| for...of = code points
```

```javascript
// Node.js
const s = "Hello 🔥";
console.log(s.length);    // 8 (not 7!) -- 🔥 is 2 UTF-16 code units
console.log(s[6]);        // '\uD83D' (half of the emoji -- broken!)
console.log([...s].length); // 7 -- spread decodes to code points

// Proper iteration
for (const char of s) {
    console.log(char);    // G, o, ... 🔥 (correct)
}
```

Both Go and JS trip up beginners with string length and indexing, but for opposite reasons: Go counts bytes (UTF-8), JS counts code units (UTF-16).

### The `strings` and `bytes` Packages

Go provides rich string/byte manipulation without methods on the string type:

```go
import "strings"

s := "Hello, World!"

strings.Contains(s, "World")       // true
strings.HasPrefix(s, "Hello")      // true
strings.HasSuffix(s, "!")          // true
strings.Index(s, "World")          // 7
strings.ToUpper(s)                 // "HELLO, WORLD!"
strings.ToLower(s)                 // "hello, world!"
strings.TrimSpace("  hi  ")       // "hi"
strings.Split("a,b,c", ",")       // ["a", "b", "c"]
strings.Join([]string{"a","b"}, "-") // "a-b"
strings.ReplaceAll(s, "l", "L")   // "HeLLo, WorLd!"
strings.Count(s, "l")             // 3
strings.Repeat("ha", 3)           // "hahaha"
```

The `bytes` package provides identical functions for `[]byte` arguments, which avoids `string` <-> `[]byte` conversions in hot paths.

---

## 9. Go vs Node.js Comparison Summary

```
 Concept                  | Go                              | Node.js
──────────────────────────┼─────────────────────────────────┼────────────────────────────
 Fixed-size collection    | [N]T (array, value type)        | No equivalent (TypedArrays
                          |                                 | are closest)
 Dynamic collection       | []T (slice)                     | Array
 Copy semantics           | Arrays: copy by default         | Arrays: reference by default
                          | Slices: header copied, data     |
                          | shared                          |
 Key-value collection     | map[K]V                         | Object, Map
 Check key existence      | val, ok := m[key]               | key in obj / map.has(key)
 Delete from collection   | delete(m, key) for maps         | delete obj.key / map.delete()
                          | append(s[:i], s[i+1:]...) for   | arr.splice(i, 1)
                          | slices                          |
 Iteration                | for i, v := range collection    | for...of, forEach, for...in
 Map ordering             | Randomized (no guarantees)      | Object: insertion order*
                          |                                 | Map: insertion order
 Nil/empty distinction    | var s []T (nil) vs []T{} (empty)| null vs []
 Functional methods       | Write your own (or use generics)| Built-in: filter, map, reduce
 String type              | Immutable []byte (UTF-8)        | Immutable (UTF-16)
 Character type           | rune (int32)                    | No separate type
 Concurrent map access    | Fatal error without sync        | Single-threaded (no issue)
```

---

## 10. Key Takeaways

1. **Arrays are value types.** They copy on assignment. You almost never use them directly -- use slices instead.

2. **Slices are headers (pointer + length + capacity) that reference an underlying array.** Copying a slice copies the header, not the data. Multiple slices can share one array.

3. **Always reassign the result of `append`.** `s = append(s, x)` -- never just `append(s, x)`.

4. **Beware of shared underlying arrays.** Sub-slicing creates a view, not a copy. Use the three-index slice `s[low:high:max]` or `copy()` to avoid surprises.

5. **Pre-allocate when you know the size.** `make([]T, 0, expectedSize)` avoids repeated allocations during `append`.

6. **Maps must be initialized before writing.** `var m map[K]V` is nil -- reads return zero values, but writes panic. Use `make()` or a literal.

7. **Use the comma-ok idiom** to distinguish "key exists with zero value" from "key doesn't exist."

8. **Maps are not concurrency-safe.** Use `sync.Mutex` or `sync.Map` when sharing across goroutines.

9. **`len(string)` returns bytes, not characters.** Use `[]rune` or `range` for correct Unicode handling.

10. **Go favors explicit loops over functional abstractions.** A `for` loop with `range` is idiomatic. With Go 1.18+ generics, you can write reusable filter/map/reduce, but the community still often prefers simple loops.

---

## 11. Practice Exercises

### Exercise 1: Slice Detective
What does this program print? Work it out by hand before running it.

```go
package main

import "fmt"

func main() {
    s := make([]int, 3, 5)
    s[0], s[1], s[2] = 1, 2, 3

    a := s[1:3]
    a = append(a, 4)

    s[1] = 99

    fmt.Println(s)
    fmt.Println(a)
    fmt.Println(len(s), cap(s))
    fmt.Println(len(a), cap(a))
}
```

<details>
<summary>Answer</summary>

```
[1 99 3]
[99 3 4]
3 5
3 4
```

`a` shares the underlying array with `s`. When `s[1] = 99`, both `s` and `a` see the change at that position. The `append` wrote `4` into `s`'s underlying array at index 3 (within capacity), but `s` still has `len=3` so it does not see index 3.

</details>

---

### Exercise 2: Word Frequency Counter
Write a function that takes a string and returns a `map[string]int` of word frequencies.

```go
package main

import (
    "fmt"
    "strings"
)

func wordFrequency(text string) map[string]int {
    // Your implementation here
}

func main() {
    text := "the quick brown fox jumps over the lazy dog the fox"
    freq := wordFrequency(text)
    for word, count := range freq {
        fmt.Printf("%s: %d\n", word, count)
    }
}
```

<details>
<summary>Solution</summary>

```go
func wordFrequency(text string) map[string]int {
    freq := make(map[string]int)
    words := strings.Fields(strings.ToLower(text))
    for _, word := range words {
        freq[word]++
    }
    return freq
}
```

`strings.Fields` splits on any whitespace. `freq[word]++` works because the zero value of `int` is 0 -- a missing key returns 0, then increments to 1.

</details>

---

### Exercise 3: Safe Sub-Slice
Write a function that returns a sub-slice that is completely independent of the original (no shared underlying array).

```go
func safeSlice(s []int, low, high int) []int {
    // Your implementation here
}
```

<details>
<summary>Solution</summary>

```go
func safeSlice(s []int, low, high int) []int {
    sub := make([]int, high-low)
    copy(sub, s[low:high])
    return sub
}
```

By using `make` + `copy`, the returned slice has its own backing array. Mutating it never affects the original.

</details>

---

### Exercise 4: Generic Group-By
Write a generic function that groups slice elements by a key function.

```go
func GroupBy[T any, K comparable](items []T, keyFn func(T) K) map[K][]T {
    // Your implementation here
}

// Usage:
// people := []string{"Alice", "Bob", "Anna", "Bill", "Charlie", "Cathy"}
// grouped := GroupBy(people, func(s string) byte { return s[0] })
// Result: map[65:[Alice Anna] 66:[Bob Bill] 67:[Charlie Cathy]]
```

<details>
<summary>Solution</summary>

```go
func GroupBy[T any, K comparable](items []T, keyFn func(T) K) map[K][]T {
    result := make(map[K][]T)
    for _, item := range items {
        key := keyFn(item)
        result[key] = append(result[key], item)
    }
    return result
}
```

This leverages the fact that accessing a missing map key returns the zero value (`nil` for a slice), and `append` works correctly with `nil` slices.

</details>

---

### Exercise 5: Reverse a String (Unicode-Safe)
Write a function that reverses a string properly, handling multi-byte Unicode characters.

```go
func reverseString(s string) string {
    // Your implementation here
}

// reverseString("Hello, 世界!") should return "!界世 ,olleH"
```

<details>
<summary>Solution</summary>

```go
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

Converting to `[]rune` handles multi-byte characters correctly. A byte-level reverse would corrupt UTF-8 sequences.

</details>

---

### Exercise 6: Implement a Stack Using a Slice

```go
type Stack[T any] struct {
    items []T
}

// Implement: Push(T), Pop() (T, bool), Peek() (T, bool), Len() int
```

<details>
<summary>Solution</summary>

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Len() int {
    return len(s.items)
}
```

Note how `Pop` uses the comma-ok pattern, matching Go's idiom for maps. Also note the `var zero T` pattern for returning the zero value of a generic type.

</details>

---

**Next Chapter: [Chapter 6 -- Structs and Methods](/golang/06-structs-and-methods.md)**
