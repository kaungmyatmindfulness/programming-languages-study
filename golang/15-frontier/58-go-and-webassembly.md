# Chapter 58: Go and WebAssembly

## Table of Contents

1. [What Is WebAssembly and Why It Matters](#1-what-is-webassembly-and-why-it-matters)
2. [Go's WASM Support](#2-gos-wasm-support)
3. [Building Your First Go WASM Module](#3-building-your-first-go-wasm-module)
4. [wasm_exec.js and Loading Go WASM in the Browser](#4-wasm_execjs-and-loading-go-wasm-in-the-browser)
5. [The syscall/js Package](#5-the-syscalljs-package)
6. [TinyGo for Smaller WASM Binaries](#6-tinygo-for-smaller-wasm-binaries)
7. [WASI: WebAssembly System Interface](#7-wasi-webassembly-system-interface)
8. [Wazero: Pure Go WebAssembly Runtime](#8-wazero-pure-go-webassembly-runtime)
9. [Performance Considerations](#9-performance-considerations)
10. [Real-World Use Cases](#10-real-world-use-cases)
11. [Current Limitations and Future of Go + WASM](#11-current-limitations-and-future-of-go--wasm)
12. [Complete Example Projects](#12-complete-example-projects)
13. [Summary](#13-summary)

---

## 1. What Is WebAssembly and Why It Matters

### 1.1 Definition

WebAssembly (WASM) is a binary instruction format for a stack-based virtual machine. It is
designed as a portable compilation target for programming languages, enabling deployment on
the web for client and server applications.

Think of it as a compact bytecode that runs at near-native speed in every modern browser and
increasingly in server-side runtimes.

### 1.2 Key Properties

| Property            | Description                                                        |
|---------------------|--------------------------------------------------------------------|
| **Portable**        | Runs on any platform with a WASM runtime (browsers, servers, edge) |
| **Fast**            | Near-native execution speed via AOT/JIT compilation                |
| **Safe**            | Sandboxed execution with no direct access to host memory           |
| **Language-agnostic** | Compile from C, C++, Rust, Go, AssemblyScript, and more         |
| **Compact**         | Binary format is smaller than equivalent text-based code           |
| **Deterministic**   | Same inputs produce same outputs across platforms                  |

### 1.3 Why It Matters for Go Developers

1. **Run Go in the browser**: Build interactive web applications with Go logic.
2. **Portable plugins**: Create sandboxed plugin systems that work everywhere.
3. **Edge computing**: Deploy Go code to Cloudflare Workers, Fastly Compute, etc.
4. **Polyglot systems**: Let Go applications execute modules written in any language.
5. **Security**: WASM's sandbox model is ideal for running untrusted code.

### 1.4 The WASM Ecosystem at a Glance

```
                    ┌─────────────────────────────────────────┐
                    │           Source Languages               │
                    │   Go  |  Rust  |  C/C++  |  Zig  | ...  │
                    └────────────────┬────────────────────────┘
                                     │ compile
                                     ▼
                    ┌─────────────────────────────────────────┐
                    │         WebAssembly (.wasm)              │
                    │      Binary instruction format           │
                    └────────────────┬────────────────────────┘
                                     │ execute
                    ┌────────────────┼────────────────────────┐
                    ▼                ▼                         ▼
              ┌──────────┐   ┌────────────┐          ┌──────────────┐
              │ Browsers │   │  Runtimes  │          │  Embedded    │
              │ Chrome   │   │  Wasmtime  │          │  wazero      │
              │ Firefox  │   │  Wasmer    │          │  wasmtime-go │
              │ Safari   │   │  WasmEdge  │          │              │
              └──────────┘   └────────────┘          └──────────────┘
```

---

## 2. Go's WASM Support

### 2.1 The js/wasm Target

Since Go 1.11, the compiler natively supports building WebAssembly modules via the
`GOOS=js GOARCH=wasm` cross-compilation target. This produces a `.wasm` binary designed
to run inside a JavaScript environment (browser or Node.js).

```bash
# Build a Go program to WebAssembly
GOOS=js GOARCH=wasm go build -o main.wasm main.go
```

### 2.2 The wasip1/wasm Target (Go 1.21+)

Go 1.21 introduced a second WASM target: `GOOS=wasip1 GOARCH=wasm`. This targets the
WASI (WebAssembly System Interface) preview 1, enabling Go WASM modules to run outside
the browser in standalone runtimes.

```bash
# Build a Go program targeting WASI
GOOS=wasip1 GOARCH=wasm go build -o main.wasm main.go
```

### 2.3 Comparison of Targets

| Feature                  | `GOOS=js GOARCH=wasm`     | `GOOS=wasip1 GOARCH=wasm` |
|--------------------------|---------------------------|----------------------------|
| **Runtime environment**  | Browser / Node.js         | WASI-compatible runtimes   |
| **JS interop**           | Full (syscall/js)         | None                       |
| **File I/O**             | Via JS shims              | Native WASI file I/O       |
| **Networking**           | Via JS fetch              | Limited (runtime-dependent)|
| **Binary size**          | ~2-10 MB                  | ~2-10 MB                   |
| **Available since**      | Go 1.11                   | Go 1.21                    |
| **Use case**             | Browser apps, DOM access  | CLI tools, server-side     |

### 2.4 Checking Your Go Version

```bash
$ go version
go version go1.23.0 darwin/arm64

# List all supported platforms
$ go tool dist list | grep wasm
js/wasm
wasip1/wasm
```

> **Tip**: Always use the latest stable Go version for the best WASM support. Each
> release improves binary size, performance, and runtime compatibility.

---

## 3. Building Your First Go WASM Module

### 3.1 Hello, WebAssembly!

Create a simple Go program that prints to the console.

**main.go**:
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello from Go WebAssembly!")
}
```

### 3.2 Compile to WASM

```bash
GOOS=js GOARCH=wasm go build -o main.wasm main.go
```

Check the output:

```bash
$ ls -lh main.wasm
-rwxr-xr-x  1 user  staff  2.4M  main.wasm
```

> **Warning**: Standard Go WASM binaries are relatively large (typically 2-10 MB)
> because they include the entire Go runtime and garbage collector. We will discuss
> strategies to reduce this later.

### 3.3 Copy the JavaScript Glue Code

Go ships a JavaScript support file that bridges WASM and the browser's JavaScript engine.

```bash
# Copy wasm_exec.js from your Go installation
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" .
```

### 3.4 Create the HTML Host Page

**index.html**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Go WebAssembly Demo</title>
</head>
<body>
    <h1>Go WebAssembly Demo</h1>
    <p>Open the browser console to see output.</p>

    <script src="wasm_exec.js"></script>
    <script>
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject)
            .then((result) => {
                go.run(result.instance);
            })
            .catch((err) => {
                console.error("Failed to load WASM:", err);
            });
    </script>
</body>
</html>
```

### 3.5 Serve and Test

You need an HTTP server because browsers block `file://` fetches for WASM.

```bash
# Simple Go file server
go run serve.go
```

**serve.go**:
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    // Ensure .wasm files are served with the correct MIME type
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/main.wasm" {
            w.Header().Set("Content-Type", "application/wasm")
        }
        http.FileServer(http.Dir(".")).ServeHTTP(w, r)
    })

    fmt.Println("Serving on http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Open `http://localhost:8080` and check the browser console -- you should see
`Hello from Go WebAssembly!`.

### 3.6 Project Structure

```
my-wasm-project/
├── main.go          # Go source code
├── main.wasm        # Compiled WASM binary (generated)
├── wasm_exec.js     # Go WASM glue (copied from $GOROOT)
├── index.html       # HTML host page
└── serve.go         # Development server
```

---

## 4. wasm_exec.js and Loading Go WASM in the Browser

### 4.1 What Is wasm_exec.js?

`wasm_exec.js` is Go's official JavaScript support file. It provides:

- The `Go` class that manages the WASM instance lifecycle.
- Implementations of Go's system calls (write to console, memory allocation, etc.).
- The bridge between Go's `syscall/js` package and browser APIs.
- Goroutine scheduling support via JavaScript event loop integration.

> **Important**: The `wasm_exec.js` version **must match** your Go compiler version.
> Always copy it from `$(go env GOROOT)/misc/wasm/wasm_exec.js` after upgrading Go.

### 4.2 The Go Class API

```javascript
// Create a new Go instance
const go = new Go();

// go.importObject - Pass this to WebAssembly.instantiate()
// go.run(instance) - Start the Go program, returns a Promise
// go.exit          - Called when the Go program exits
// go.argv          - Command-line arguments (array of strings)
// go.env           - Environment variables (object)
```

### 4.3 Loading Strategies

#### Strategy 1: instantiateStreaming (Recommended)

```javascript
const go = new Go();
const result = await WebAssembly.instantiateStreaming(
    fetch("main.wasm"),
    go.importObject
);
await go.run(result.instance);
```

This is the most efficient method. It compiles the WASM module while downloading it.

> **Warning**: `instantiateStreaming` requires the server to serve `.wasm` files with
> `Content-Type: application/wasm`. If your server does not, use the fallback below.

#### Strategy 2: ArrayBuffer Fallback

```javascript
const go = new Go();
const response = await fetch("main.wasm");
const bytes = await response.arrayBuffer();
const result = await WebAssembly.instantiate(bytes, go.importObject);
await go.run(result.instance);
```

#### Strategy 3: With a Loading Indicator

```javascript
async function loadWasm() {
    const statusEl = document.getElementById("status");
    statusEl.textContent = "Loading WebAssembly...";

    try {
        const go = new Go();
        const result = await WebAssembly.instantiateStreaming(
            fetch("main.wasm"),
            go.importObject
        );

        statusEl.textContent = "Running...";
        await go.run(result.instance);
        statusEl.textContent = "Done.";
    } catch (err) {
        statusEl.textContent = "Error: " + err.message;
        console.error(err);
    }
}

loadWasm();
```

### 4.4 Passing Arguments and Environment Variables

```javascript
const go = new Go();

// Simulate command-line arguments
go.argv = ["myapp", "--verbose", "--port=8080"];

// Set environment variables
go.env = {
    "HOME": "/home/user",
    "APP_MODE": "production",
};

const result = await WebAssembly.instantiateStreaming(
    fetch("main.wasm"),
    go.importObject
);
await go.run(result.instance);
```

Read them in Go:

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    fmt.Println("Args:", os.Args)
    fmt.Println("APP_MODE:", os.Getenv("APP_MODE"))
}
```

### 4.5 Running in Node.js

Go WASM modules can also run in Node.js. Go provides a helper script.

```bash
# Copy the Node.js runner
cp "$(go env GOROOT)/misc/wasm/wasm_exec_node.js" .

# Run with Node.js
node wasm_exec_node.js main.wasm
```

Or manually:

```javascript
// run_node.mjs
import { readFile } from "fs/promises";
import "./wasm_exec.js";

const go = new globalThis.Go();
const wasmBuffer = await readFile("main.wasm");
const result = await WebAssembly.instantiate(wasmBuffer, go.importObject);
await go.run(result.instance);
```

---

## 5. The syscall/js Package

The `syscall/js` package is Go's bridge to JavaScript. It allows Go code to call
JavaScript functions, access DOM elements, handle events, and expose Go functions to
JavaScript.

### 5.1 Core Types

```go
import "syscall/js"
```

| Type          | Description                                              |
|---------------|----------------------------------------------------------|
| `js.Value`    | Represents any JavaScript value (object, function, etc.) |
| `js.Func`     | A wrapped Go function callable from JavaScript           |
| `js.Global()` | Returns the JavaScript global object (`window` / `globalThis`) |
| `js.Null()`   | JavaScript `null`                                        |
| `js.Undefined()` | JavaScript `undefined`                               |

### 5.2 Calling JavaScript from Go

#### Accessing Global Objects

```go
package main

import (
    "fmt"
    "syscall/js"
)

func main() {
    // Access the global window object
    window := js.Global()

    // Access navigator.userAgent
    userAgent := window.Get("navigator").Get("userAgent").String()
    fmt.Println("User Agent:", userAgent)

    // Access window.location.href
    href := window.Get("location").Get("href").String()
    fmt.Println("Current URL:", href)

    // Access window.innerWidth
    width := window.Get("innerWidth").Int()
    fmt.Println("Window width:", width)
}
```

#### Calling JavaScript Functions

```go
package main

import (
    "syscall/js"
)

func main() {
    // console.log("Hello from Go!")
    js.Global().Get("console").Call("log", "Hello from Go!")

    // alert("Go says hi!")
    js.Global().Call("alert", "Go says hi!")

    // Math.random()
    random := js.Global().Get("Math").Call("random").Float()
    js.Global().Get("console").Call("log", "Random number:", random)

    // JSON.stringify({name: "Go", version: 1.23})
    obj := js.Global().Get("Object").New()
    obj.Set("name", "Go")
    obj.Set("version", 1.23)
    jsonStr := js.Global().Get("JSON").Call("stringify", obj).String()
    js.Global().Get("console").Call("log", "JSON:", jsonStr)
}
```

#### Creating JavaScript Objects

```go
package main

import "syscall/js"

func main() {
    // Create a plain object: {}
    obj := js.Global().Get("Object").New()
    obj.Set("name", "Alice")
    obj.Set("age", 30)
    obj.Set("active", true)

    // Create an array: [1, 2, 3]
    arr := js.Global().Get("Array").New(1, 2, 3)
    arr.Call("push", 4)

    length := arr.Get("length").Int()
    js.Global().Get("console").Call("log", "Array length:", length)

    // Create a Date
    date := js.Global().Get("Date").New()
    js.Global().Get("console").Call("log", "Current time:", date.Call("toISOString"))
}
```

### 5.3 Exposing Go Functions to JavaScript

To keep the Go program alive and expose functions that JavaScript can call, you need to
register functions and block the main goroutine.

```go
package main

import (
    "fmt"
    "syscall/js"
)

// add is a Go function exposed to JavaScript
func add(this js.Value, args []js.Value) interface{} {
    if len(args) != 2 {
        return "Error: expected 2 arguments"
    }
    a := args[0].Float()
    b := args[1].Float()
    return a + b
}

// greet returns a greeting string
func greet(this js.Value, args []js.Value) interface{} {
    if len(args) != 1 {
        return "Error: expected 1 argument"
    }
    name := args[0].String()
    return fmt.Sprintf("Hello, %s! Greetings from Go.", name)
}

// fibonacci calculates the nth Fibonacci number
func fibonacci(this js.Value, args []js.Value) interface{} {
    if len(args) != 1 {
        return "Error: expected 1 argument"
    }
    n := args[0].Int()
    if n <= 1 {
        return n
    }
    a, b := 0, 1
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}

func main() {
    fmt.Println("Go WASM module loaded")

    // Register functions on the global object
    js.Global().Set("goAdd", js.FuncOf(add))
    js.Global().Set("goGreet", js.FuncOf(greet))
    js.Global().Set("goFibonacci", js.FuncOf(fibonacci))

    fmt.Println("Functions registered: goAdd, goGreet, goFibonacci")

    // Block the main goroutine to keep the program alive
    // Without this, the Go program exits immediately
    <-make(chan struct{})
}
```

Call from JavaScript:

```javascript
// After WASM is loaded and running:
console.log(goAdd(3, 4));          // 7
console.log(goGreet("World"));     // "Hello, World! Greetings from Go."
console.log(goFibonacci(10));      // 55
```

> **Important**: The `<-make(chan struct{})` pattern blocks forever, keeping the Go
> WASM runtime alive so JavaScript can call the registered functions. Without it,
> the Go program exits and the functions become unavailable.

### 5.4 Manipulating the DOM

```go
package main

import (
    "fmt"
    "strconv"
    "syscall/js"
)

var document js.Value

func init() {
    document = js.Global().Get("document")
}

// createElement creates a DOM element with optional text content
func createElement(tag, text string) js.Value {
    el := document.Call("createElement", tag)
    if text != "" {
        el.Set("textContent", text)
    }
    return el
}

// getElementById retrieves a DOM element by ID
func getElementById(id string) js.Value {
    return document.Call("getElementById", id)
}

// buildUI creates a dynamic UI
func buildUI() {
    app := getElementById("app")

    // Create a heading
    h1 := createElement("h1", "Go WASM DOM Manipulation")
    h1.Get("style").Set("color", "#2563eb")
    app.Call("appendChild", h1)

    // Create an input field
    input := createElement("input", "")
    input.Set("type", "text")
    input.Set("placeholder", "Enter a number...")
    input.Set("id", "numberInput")
    input.Get("style").Set("padding", "8px")
    input.Get("style").Set("fontSize", "16px")
    input.Get("style").Set("marginRight", "8px")
    app.Call("appendChild", input)

    // Create a button
    btn := createElement("button", "Calculate Fibonacci")
    btn.Get("style").Set("padding", "8px 16px")
    btn.Get("style").Set("fontSize", "16px")
    btn.Get("style").Set("cursor", "pointer")
    app.Call("appendChild", btn)

    // Create result display
    result := createElement("p", "")
    result.Set("id", "result")
    result.Get("style").Set("fontSize", "20px")
    result.Get("style").Set("fontWeight", "bold")
    app.Call("appendChild", result)

    // Add click handler
    btn.Call("addEventListener", "click", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        inputEl := getElementById("numberInput")
        valueStr := inputEl.Get("value").String()
        n, err := strconv.Atoi(valueStr)
        if err != nil {
            getElementById("result").Set("textContent", "Please enter a valid number")
            return nil
        }

        // Calculate fibonacci
        fib := calcFib(n)
        getElementById("result").Set("textContent",
            fmt.Sprintf("Fibonacci(%d) = %d", n, fib))
        return nil
    }))

    // Create a dynamic list
    ul := createElement("ul", "")
    for i := 0; i < 10; i++ {
        li := createElement("li", fmt.Sprintf("Fibonacci(%d) = %d", i, calcFib(i)))
        ul.Call("appendChild", li)
    }
    app.Call("appendChild", ul)
}

func calcFib(n int) int {
    if n <= 1 {
        return n
    }
    a, b := 0, 1
    for i := 2; i <= n; i++ {
        a, b = b, a+b
    }
    return b
}

func main() {
    fmt.Println("Starting Go WASM DOM app...")
    buildUI()

    // Keep alive
    <-make(chan struct{})
}
```

**index.html**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Go WASM DOM Demo</title>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        #app { max-width: 600px; margin: 0 auto; }
    </style>
</head>
<body>
    <div id="app"></div>
    <script src="wasm_exec.js"></script>
    <script>
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject)
            .then(result => go.run(result.instance));
    </script>
</body>
</html>
```

### 5.5 Event Handling

```go
package main

import (
    "fmt"
    "syscall/js"
)

func main() {
    document := js.Global().Get("document")

    // Mouse move handler
    document.Call("addEventListener", "mousemove",
        js.FuncOf(func(this js.Value, args []js.Value) interface{} {
            event := args[0]
            x := event.Get("clientX").Int()
            y := event.Get("clientY").Int()

            coordsEl := document.Call("getElementById", "coords")
            if !coordsEl.IsNull() {
                coordsEl.Set("textContent", fmt.Sprintf("Mouse: (%d, %d)", x, y))
            }
            return nil
        }))

    // Keyboard handler
    document.Call("addEventListener", "keydown",
        js.FuncOf(func(this js.Value, args []js.Value) interface{} {
            event := args[0]
            key := event.Get("key").String()
            code := event.Get("code").String()

            fmt.Printf("Key pressed: %s (code: %s)\n", key, code)

            // Prevent default for certain keys
            if key == "s" && event.Get("ctrlKey").Bool() {
                event.Call("preventDefault")
                fmt.Println("Ctrl+S intercepted!")
            }
            return nil
        }))

    // Resize handler
    js.Global().Call("addEventListener", "resize",
        js.FuncOf(func(this js.Value, args []js.Value) interface{} {
            w := js.Global().Get("innerWidth").Int()
            h := js.Global().Get("innerHeight").Int()
            fmt.Printf("Window resized to: %dx%d\n", w, h)
            return nil
        }))

    // Form submission
    form := document.Call("getElementById", "myForm")
    if !form.IsNull() {
        form.Call("addEventListener", "submit",
            js.FuncOf(func(this js.Value, args []js.Value) interface{} {
                event := args[0]
                event.Call("preventDefault")

                formData := js.Global().Get("FormData").New(form)
                name := formData.Call("get", "name").String()
                email := formData.Call("get", "email").String()

                fmt.Printf("Form submitted: name=%s, email=%s\n", name, email)
                return nil
            }))
    }

    fmt.Println("Event handlers registered")
    <-make(chan struct{})
}
```

### 5.6 Promises and Async Operations

Go's goroutines integrate with JavaScript's async model through Promises.

```go
package main

import (
    "fmt"
    "syscall/js"
    "time"
)

// fetchData performs an HTTP GET using the browser's fetch API
func fetchData(this js.Value, args []js.Value) interface{} {
    if len(args) < 1 {
        return js.Global().Get("Promise").Call("reject", "URL required")
    }
    url := args[0].String()

    // Create a Promise that wraps the async operation
    handler := js.FuncOf(func(this js.Value, promiseArgs []js.Value) interface{} {
        resolve := promiseArgs[0]
        reject := promiseArgs[1]

        // Run the fetch in a goroutine to avoid blocking
        go func() {
            // Use JavaScript's fetch API
            fetchPromise := js.Global().Call("fetch", url)

            // Set up the then/catch chain
            fetchPromise.Call("then",
                js.FuncOf(func(this js.Value, args []js.Value) interface{} {
                    response := args[0]
                    if !response.Get("ok").Bool() {
                        reject.Invoke(fmt.Sprintf("HTTP error: %d",
                            response.Get("status").Int()))
                        return nil
                    }
                    // Parse response as JSON
                    response.Call("json").Call("then",
                        js.FuncOf(func(this js.Value, args []js.Value) interface{} {
                            resolve.Invoke(args[0])
                            return nil
                        }),
                    )
                    return nil
                }),
            ).Call("catch",
                js.FuncOf(func(this js.Value, args []js.Value) interface{} {
                    reject.Invoke(args[0])
                    return nil
                }),
            )
        }()
        return nil
    })

    return js.Global().Get("Promise").New(handler)
}

// goSleep demonstrates a Go goroutine returning a Promise
func goSleep(this js.Value, args []js.Value) interface{} {
    ms := 1000
    if len(args) > 0 {
        ms = args[0].Int()
    }

    handler := js.FuncOf(func(this js.Value, promiseArgs []js.Value) interface{} {
        resolve := promiseArgs[0]

        go func() {
            time.Sleep(time.Duration(ms) * time.Millisecond)
            resolve.Invoke(fmt.Sprintf("Slept for %d ms", ms))
        }()

        return nil
    })

    return js.Global().Get("Promise").New(handler)
}

func main() {
    js.Global().Set("goFetchData", js.FuncOf(fetchData))
    js.Global().Set("goSleep", js.FuncOf(goSleep))

    fmt.Println("Async functions registered")
    <-make(chan struct{})
}
```

Call from JavaScript:

```javascript
// Using the Go-exposed async functions
async function demo() {
    // Fetch data through Go
    const data = await goFetchData("https://jsonplaceholder.typicode.com/todos/1");
    console.log("Fetched:", data);

    // Sleep using Go's goroutine
    const msg = await goSleep(2000);
    console.log(msg); // "Slept for 2000 ms"
}

demo();
```

### 5.7 Working with Typed Arrays and Binary Data

```go
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "fmt"
    "syscall/js"
)

// hashData takes a Uint8Array and returns its SHA-256 hash
func hashData(this js.Value, args []js.Value) interface{} {
    if len(args) < 1 {
        return "Error: expected Uint8Array argument"
    }

    // Read the JavaScript Uint8Array into a Go byte slice
    jsArray := args[0]
    length := jsArray.Get("length").Int()
    data := make([]byte, length)
    js.CopyBytesToGo(data, jsArray)

    // Compute SHA-256
    hash := sha256.Sum256(data)
    return hex.EncodeToString(hash[:])
}

// processImage takes image data and inverts the colors
func processImage(this js.Value, args []js.Value) interface{} {
    if len(args) < 1 {
        return js.Undefined()
    }

    jsArray := args[0]
    length := jsArray.Get("length").Int()
    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, jsArray)

    // Invert colors (RGBA format)
    for i := 0; i < length; i += 4 {
        pixels[i] = 255 - pixels[i]     // R
        pixels[i+1] = 255 - pixels[i+1] // G
        pixels[i+2] = 255 - pixels[i+2] // B
        // pixels[i+3] is alpha - leave unchanged
    }

    // Copy back to JavaScript
    result := js.Global().Get("Uint8Array").New(length)
    js.CopyBytesToJS(result, pixels)

    return result
}

func main() {
    js.Global().Set("goHashData", js.FuncOf(hashData))
    js.Global().Set("goProcessImage", js.FuncOf(processImage))

    fmt.Println("Binary processing functions registered")
    <-make(chan struct{})
}
```

Usage in JavaScript:

```javascript
// Hash a string
const encoder = new TextEncoder();
const data = encoder.encode("Hello, WebAssembly!");
console.log("SHA-256:", goHashData(data));

// Process image data from a canvas
const canvas = document.getElementById("myCanvas");
const ctx = canvas.getContext("2d");
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
const inverted = goProcessImage(imageData.data);
const newImageData = new ImageData(
    new Uint8ClampedArray(inverted.buffer),
    canvas.width,
    canvas.height
);
ctx.putImageData(newImageData, 0, 0);
```

### 5.8 Cleaning Up Resources

Always release `js.Func` values when they are no longer needed to prevent memory leaks.

```go
package main

import (
    "fmt"
    "syscall/js"
)

func main() {
    // Create a function
    callback := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        fmt.Println("Callback invoked")
        return nil
    })

    // Use it
    js.Global().Get("setTimeout").Invoke(callback, 1000)

    // IMPORTANT: Release when no longer needed
    // For one-shot callbacks, you can release inside the callback:
    oneShot := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        fmt.Println("One-shot callback")
        // Note: In practice, store the js.Func in a variable
        // and release it after invocation
        return nil
    })
    _ = oneShot // Use it, then call oneShot.Release() later

    // For a cleanup pattern with registered functions:
    done := make(chan struct{})

    cleanupFn := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        fmt.Println("Cleanup requested")
        close(done)
        return nil
    })
    js.Global().Set("cleanup", cleanupFn)

    // Wait for cleanup
    <-done

    // Release all functions
    callback.Release()
    oneShot.Release()
    cleanupFn.Release()

    fmt.Println("All resources released")
}
```

---

## 6. TinyGo for Smaller WASM Binaries

### 6.1 The Size Problem

Standard Go WASM binaries include the full Go runtime, garbage collector, and reflection
support. Even a "Hello, World!" program compiles to roughly 2-3 MB. For web applications
where download size matters, this is significant.

### 6.2 What Is TinyGo?

TinyGo is an alternative Go compiler designed for small targets: microcontrollers,
WebAssembly, and other constrained environments. It uses LLVM as its backend and produces
dramatically smaller WASM binaries.

### 6.3 Installation

```bash
# macOS
brew install tinygo

# Linux (Ubuntu/Debian)
wget https://github.com/tinygo-org/tinygo/releases/download/v0.32.0/tinygo_0.32.0_amd64.deb
sudo dpkg -i tinygo_0.32.0_amd64.deb

# Verify installation
tinygo version
```

### 6.4 Building with TinyGo

```bash
# Standard Go build
GOOS=js GOARCH=wasm go build -o main-go.wasm main.go

# TinyGo build
tinygo build -o main-tinygo.wasm -target wasm main.go
```

### 6.5 Size Comparison

| Program                | Standard Go  | TinyGo      | Reduction |
|------------------------|-------------|-------------|-----------|
| Hello World            | ~2.4 MB     | ~250 KB     | ~90%      |
| JSON processing        | ~4.1 MB     | ~400 KB     | ~90%      |
| HTTP handler           | ~6.2 MB     | ~800 KB     | ~87%      |
| Complex math           | ~3.5 MB     | ~300 KB     | ~91%      |

> **Tip**: You can further reduce TinyGo binaries with `wasm-opt` from the Binaryen
> toolkit:
> ```bash
> wasm-opt -Oz -o main-optimized.wasm main-tinygo.wasm
> ```

### 6.6 TinyGo WASM Example

**main.go** (TinyGo-compatible):
```go
package main

import (
    "fmt"
)

//export multiply
func multiply(a, b int) int {
    return a * b
}

//export greet
func greet(name string) string {
    return fmt.Sprintf("Hello, %s!", name)
}

func main() {
    fmt.Println("TinyGo WASM module loaded")
}
```

Build:
```bash
tinygo build -o main.wasm -target wasm -no-debug main.go
```

### 6.7 TinyGo vs Standard Go: Trade-offs

| Feature                | Standard Go          | TinyGo                      |
|------------------------|---------------------|------------------------------|
| **Binary size**        | 2-10 MB             | 100 KB - 1 MB               |
| **Full stdlib**        | Yes                 | Partial (growing)            |
| **Reflection**         | Full                | Limited                      |
| **Goroutines**         | Full (M:N scheduler)| Cooperative (single-threaded)|
| **GC**                 | Concurrent, generational | Simple mark-sweep       |
| **cgo**                | Via JS shims        | Via LLVM                     |
| **Compile speed**      | Fast                | Slower (LLVM backend)        |
| **`//export` funcs**   | No                  | Yes                          |
| **WASI support**       | Go 1.21+            | Built-in                     |
| **net/http**           | Partial             | No                           |
| **encoding/json**      | Yes                 | Yes (most features)          |

> **Warning**: TinyGo does not support the entire Go standard library. Packages relying
> heavily on reflection (like `encoding/gob`) may not compile. Always test your specific
> dependencies. Check https://tinygo.org/docs/reference/lang-support/ for details.

### 6.8 TinyGo WASI Target

```bash
# Build for WASI
tinygo build -o main.wasm -target wasi main.go

# Run with wasmtime
wasmtime main.wasm

# Run with wasmer
wasmer main.wasm
```

### 6.9 TinyGo with JavaScript Interop

TinyGo uses a different JavaScript glue file than standard Go.

```bash
# Copy TinyGo's wasm glue
cp $(tinygo env TINYGOROOT)/targets/wasm_exec.js .
```

```go
// main.go - TinyGo WASM with JS interop
package main

import "syscall/js"

//export add
func add(a, b int) int {
    return a + b
}

func main() {
    // TinyGo also supports syscall/js
    js.Global().Set("goAdd", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        a := args[0].Int()
        b := args[1].Int()
        return a + b
    }))

    // Keep alive
    select {}
}
```

---

## 7. WASI: WebAssembly System Interface

### 7.1 What Is WASI?

WASI (WebAssembly System Interface) is a set of standard APIs that allows WebAssembly
modules to interact with the host operating system in a portable, sandboxed way. Think of
it as a "POSIX for WebAssembly."

WASI provides:
- File system access (sandboxed to specific directories)
- Standard I/O (stdin, stdout, stderr)
- Environment variables
- Clock/time access
- Random number generation

### 7.2 Why WASI Matters

```
                    Without WASI                        With WASI
              ┌────────────────────┐            ┌────────────────────┐
              │  WASM Module       │            │  WASM Module       │
              │  (browser only)    │            │  (runs anywhere)   │
              └────────┬───────────┘            └────────┬───────────┘
                       │                                 │
                       ▼                                 ▼
              ┌────────────────────┐            ┌────────────────────┐
              │  JavaScript APIs   │            │  WASI Interface    │
              │  (browser-specific)│            │  (standardized)    │
              └────────────────────┘            └────────┬───────────┘
                                                         │
                                          ┌──────────────┼──────────────┐
                                          ▼              ▼              ▼
                                    ┌──────────┐  ┌──────────┐  ┌──────────┐
                                    │ Wasmtime │  │ Wasmer   │  │ Node.js  │
                                    │ Wasmer   │  │          │  │ Deno     │
                                    │ wazero   │  │          │  │          │
                                    └──────────┘  └──────────┘  └──────────┘
```

### 7.3 Building Go for WASI

```bash
# Go 1.21+ supports WASI natively
GOOS=wasip1 GOARCH=wasm go build -o main.wasm main.go
```

**main.go**:
```go
package main

import (
    "fmt"
    "os"
    "runtime"
)

func main() {
    fmt.Printf("Hello from Go on WASI!\n")
    fmt.Printf("OS: %s, Arch: %s\n", runtime.GOOS, runtime.GOARCH)
    fmt.Printf("Args: %v\n", os.Args)

    // Environment variables work
    if val, ok := os.LookupEnv("MY_VAR"); ok {
        fmt.Printf("MY_VAR = %s\n", val)
    }

    // File I/O works with WASI
    data := []byte("Hello from WASI!\n")
    err := os.WriteFile("/tmp/output.txt", data, 0644)
    if err != nil {
        fmt.Printf("Write error (expected in sandboxed mode): %v\n", err)
    }
}
```

### 7.4 Running Go WASM with Different Runtimes

#### Wasmtime

Wasmtime is a fast, secure WebAssembly runtime by the Bytecode Alliance.

```bash
# Install
curl https://wasmtime.dev/install.sh -sSf | bash

# Run
wasmtime main.wasm

# With arguments
wasmtime main.wasm -- --verbose --port=8080

# With directory access
wasmtime --dir=/tmp main.wasm

# With environment variables
wasmtime --env MY_VAR=hello main.wasm
```

#### Wasmer

Wasmer is a fast, universal WebAssembly runtime.

```bash
# Install
curl https://get.wasmer.io -sSfL | sh

# Run
wasmer main.wasm

# With directory mapping
wasmer --dir /tmp main.wasm

# With environment variables
wasmer --env MY_VAR=hello main.wasm
```

#### Wazero

Wazero is a zero-dependency WebAssembly runtime written in pure Go.

```bash
# Install CLI
go install github.com/tetratelabs/wazero/cmd/wazero@latest

# Run
wazero run main.wasm

# With directory access
wazero run --mount /tmp:/tmp main.wasm

# With environment variables
wazero run --env MY_VAR=hello main.wasm
```

### 7.5 WASI File I/O Example

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    // Check if a file argument was provided
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "Usage: wordcount <filename>")
        os.Exit(1)
    }

    filename := os.Args[1]

    file, err := os.Open(filename)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error opening file: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    var lines, words, chars int
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        line := scanner.Text()
        lines++
        words += len(strings.Fields(line))
        chars += len(line) + 1 // +1 for newline
    }

    if err := scanner.Err(); err != nil {
        fmt.Fprintf(os.Stderr, "Error reading file: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("  %d  %d  %d  %s\n", lines, words, chars, filename)
}
```

Build and run:

```bash
# Build for WASI
GOOS=wasip1 GOARCH=wasm go build -o wordcount.wasm wordcount.go

# Create a test file
echo "Hello World\nThis is a test\nGo and WebAssembly" > /tmp/test.txt

# Run with wasmtime (grant access to /tmp)
wasmtime --dir=/tmp wordcount.wasm -- /tmp/test.txt

# Run with wazero
wazero run --mount /tmp:/tmp wordcount.wasm /tmp/test.txt
```

### 7.6 Go's WASI Limitations

As of Go 1.23, the `wasip1` target has some notable limitations:

- **No networking**: WASI preview 1 does not define a sockets API.
- **No goroutine parallelism**: WASM is single-threaded (threads proposal is in progress).
- **Limited signal handling**: Signal-based features do not work.
- **No `plugin` package**: Dynamic loading is not supported.

> **Note**: WASI Preview 2 (also called "WASI 0.2") is a major revision with a
> component model, improved networking, and more. Go support is expected in future releases.

---

## 8. Wazero: Pure Go WebAssembly Runtime

### 8.1 What Is Wazero?

Wazero is a WebAssembly runtime written entirely in Go with **zero dependencies** --
not even CGo. This makes it ideal for embedding WebAssembly execution directly in Go
applications without external libraries.

Key features:
- Pure Go -- no C dependencies, no CGo
- Supports WebAssembly 1.0 (Core Specification)
- WASI Preview 1 support
- Host function support for extending WASM modules
- Optimizing compiler for near-native performance
- Well-suited for plugin systems

### 8.2 Installation

```bash
go get github.com/tetratelabs/wazero@latest
```

### 8.3 Running a WASM Module in Go

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

func main() {
    ctx := context.Background()

    // Create a new runtime
    r := wazero.NewRuntime(ctx)
    defer r.Close(ctx)

    // Instantiate WASI for the module to use
    wasi_snapshot_preview1.MustInstantiate(ctx, r)

    // Read the WASM binary
    wasmBytes, err := os.ReadFile("hello.wasm")
    if err != nil {
        log.Fatalf("Failed to read wasm: %v", err)
    }

    // Configure the module
    config := wazero.NewModuleConfig().
        WithStdout(os.Stdout).
        WithStderr(os.Stderr).
        WithArgs("hello", "--verbose").
        WithEnv("GREETING", "Hello from wazero!")

    // Instantiate and run the module
    _, err = r.InstantiateWithConfig(ctx, wasmBytes, config)
    if err != nil {
        log.Fatalf("Failed to instantiate module: %v", err)
    }

    fmt.Println("Module executed successfully")
}
```

### 8.4 Host Functions

Host functions let Go code expose functionality to WASM modules. This is the foundation
for building plugin systems.

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/api"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

func main() {
    ctx := context.Background()

    r := wazero.NewRuntime(ctx)
    defer r.Close(ctx)

    wasi_snapshot_preview1.MustInstantiate(ctx, r)

    // Define a host module with functions the WASM module can call
    _, err := r.NewHostModuleBuilder("env").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context, m api.Module, offset, length uint32) {
            // Read a string from WASM memory
            buf, ok := m.Memory().Read(offset, length)
            if !ok {
                log.Printf("Memory read failed at offset=%d, length=%d", offset, length)
                return
            }
            fmt.Printf("[HOST LOG] %s\n", string(buf))
        }).
        WithParameterNames("offset", "length").
        Export("host_log").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context) uint64 {
            // Return current timestamp as i64
            return uint64(1234567890)
        }).
        Export("host_time").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context, m api.Module, keyPtr, keyLen, valPtr, valLen uint32) {
            key, _ := m.Memory().Read(keyPtr, keyLen)
            val, _ := m.Memory().Read(valPtr, valLen)
            fmt.Printf("[HOST KV SET] %s = %s\n", string(key), string(val))
        }).
        Export("host_kv_set").
        Instantiate(ctx)
    if err != nil {
        log.Fatalf("Failed to instantiate host module: %v", err)
    }

    // Load and run the guest WASM module
    wasmBytes, err := os.ReadFile("plugin.wasm")
    if err != nil {
        log.Fatalf("Failed to read wasm: %v", err)
    }

    config := wazero.NewModuleConfig().
        WithStdout(os.Stdout).
        WithStderr(os.Stderr)

    mod, err := r.InstantiateWithConfig(ctx, wasmBytes, config)
    if err != nil {
        log.Fatalf("Failed to instantiate: %v", err)
    }

    // Call an exported function from the WASM module
    processFunc := mod.ExportedFunction("process")
    if processFunc != nil {
        results, err := processFunc.Call(ctx, 42)
        if err != nil {
            log.Fatalf("Failed to call process: %v", err)
        }
        fmt.Printf("process(42) returned: %d\n", results[0])
    }
}
```

### 8.5 Building a Plugin System with WASM

This is a complete example of a plugin architecture using wazero.

#### Plugin Interface (shared contract)

The host and plugin agree on a set of exported/imported functions:

```
Plugin must export:
  - init()                    -> called once when plugin loads
  - process(input_ptr i32, input_len i32) -> (output_ptr i32)
  - get_output_len()          -> (i32)
  - alloc(size i32)           -> (ptr i32)
  - dealloc(ptr i32, size i32)

Host provides (imported by plugin):
  - env.host_log(ptr i32, len i32)
  - env.host_get_config(key_ptr i32, key_len i32) -> (val_ptr i32)
```

#### Plugin Host (Go application)

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "path/filepath"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/api"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

// Plugin represents a loaded WASM plugin
type Plugin struct {
    name    string
    module  api.Module
    runtime wazero.Runtime
}

// PluginHost manages multiple WASM plugins
type PluginHost struct {
    plugins map[string]*Plugin
    config  map[string]string
}

// NewPluginHost creates a new plugin host
func NewPluginHost() *PluginHost {
    return &PluginHost{
        plugins: make(map[string]*Plugin),
        config: map[string]string{
            "version": "1.0.0",
            "mode":    "production",
        },
    }
}

// LoadPlugin loads a WASM plugin from a file
func (h *PluginHost) LoadPlugin(ctx context.Context, name, path string) error {
    wasmBytes, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("reading plugin %s: %w", name, err)
    }

    r := wazero.NewRuntime(ctx)

    // Instantiate WASI
    wasi_snapshot_preview1.MustInstantiate(ctx, r)

    // Register host functions
    _, err = r.NewHostModuleBuilder("env").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context, m api.Module, ptr, length uint32) {
            buf, ok := m.Memory().Read(ptr, length)
            if !ok {
                return
            }
            fmt.Printf("[%s] %s\n", name, string(buf))
        }).
        Export("host_log").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context, m api.Module, keyPtr, keyLen uint32) uint32 {
            keyBuf, ok := m.Memory().Read(keyPtr, keyLen)
            if !ok {
                return 0
            }
            key := string(keyBuf)
            val, exists := h.config[key]
            if !exists {
                return 0
            }
            // Allocate memory in WASM for the response
            allocFn := m.ExportedFunction("alloc")
            if allocFn == nil {
                return 0
            }
            results, err := allocFn.Call(ctx, uint64(len(val)))
            if err != nil {
                return 0
            }
            ptr := uint32(results[0])
            m.Memory().Write(ptr, []byte(val))
            return ptr
        }).
        Export("host_get_config").
        Instantiate(ctx)
    if err != nil {
        r.Close(ctx)
        return fmt.Errorf("host module for %s: %w", name, err)
    }

    // Instantiate the plugin module
    config := wazero.NewModuleConfig().
        WithStdout(os.Stdout).
        WithStderr(os.Stderr).
        WithName(name)

    mod, err := r.InstantiateWithConfig(ctx, wasmBytes, config)
    if err != nil {
        r.Close(ctx)
        return fmt.Errorf("instantiating %s: %w", name, err)
    }

    h.plugins[name] = &Plugin{
        name:    name,
        module:  mod,
        runtime: r,
    }

    // Call the plugin's init function if it exists
    initFn := mod.ExportedFunction("init")
    if initFn != nil {
        _, err = initFn.Call(ctx)
        if err != nil {
            return fmt.Errorf("init %s: %w", name, err)
        }
    }

    fmt.Printf("Plugin '%s' loaded successfully\n", name)
    return nil
}

// CallPlugin invokes a function on a specific plugin
func (h *PluginHost) CallPlugin(ctx context.Context, pluginName, funcName string, args ...uint64) ([]uint64, error) {
    plugin, ok := h.plugins[pluginName]
    if !ok {
        return nil, fmt.Errorf("plugin %s not found", pluginName)
    }

    fn := plugin.module.ExportedFunction(funcName)
    if fn == nil {
        return nil, fmt.Errorf("function %s not found in %s", funcName, pluginName)
    }

    return fn.Call(ctx, args...)
}

// SendData writes data to WASM memory and calls the process function
func (h *PluginHost) SendData(ctx context.Context, pluginName string, data interface{}) ([]byte, error) {
    plugin, ok := h.plugins[pluginName]
    if !ok {
        return nil, fmt.Errorf("plugin %s not found", pluginName)
    }

    // Serialize data to JSON
    inputBytes, err := json.Marshal(data)
    if err != nil {
        return nil, fmt.Errorf("marshaling input: %w", err)
    }

    // Allocate memory in WASM for the input
    allocFn := plugin.module.ExportedFunction("alloc")
    if allocFn == nil {
        return nil, fmt.Errorf("alloc function not found in %s", pluginName)
    }

    results, err := allocFn.Call(ctx, uint64(len(inputBytes)))
    if err != nil {
        return nil, fmt.Errorf("allocating memory: %w", err)
    }
    inputPtr := uint32(results[0])

    // Write input data to WASM memory
    ok = plugin.module.Memory().Write(inputPtr, inputBytes)
    if !ok {
        return nil, fmt.Errorf("writing to WASM memory failed")
    }

    // Call the process function
    processFn := plugin.module.ExportedFunction("process")
    if processFn == nil {
        return nil, fmt.Errorf("process function not found in %s", pluginName)
    }

    results, err = processFn.Call(ctx, uint64(inputPtr), uint64(len(inputBytes)))
    if err != nil {
        return nil, fmt.Errorf("calling process: %w", err)
    }
    outputPtr := uint32(results[0])

    // Get the output length
    getLenFn := plugin.module.ExportedFunction("get_output_len")
    if getLenFn == nil {
        return nil, fmt.Errorf("get_output_len not found in %s", pluginName)
    }

    results, err = getLenFn.Call(ctx)
    if err != nil {
        return nil, fmt.Errorf("getting output length: %w", err)
    }
    outputLen := uint32(results[0])

    // Read output from WASM memory
    output, ok := plugin.module.Memory().Read(outputPtr, outputLen)
    if !ok {
        return nil, fmt.Errorf("reading output from WASM memory failed")
    }

    // Make a copy (WASM memory may be invalidated)
    result := make([]byte, len(output))
    copy(result, output)

    return result, nil
}

// UnloadPlugin unloads a plugin and frees its resources
func (h *PluginHost) UnloadPlugin(ctx context.Context, name string) error {
    plugin, ok := h.plugins[name]
    if !ok {
        return fmt.Errorf("plugin %s not found", name)
    }

    err := plugin.runtime.Close(ctx)
    delete(h.plugins, name)

    if err != nil {
        return fmt.Errorf("closing plugin %s: %w", name, err)
    }

    fmt.Printf("Plugin '%s' unloaded\n", name)
    return nil
}

func main() {
    ctx := context.Background()
    host := NewPluginHost()

    // Load all .wasm plugins from a directory
    pluginDir := "./plugins"
    entries, err := os.ReadDir(pluginDir)
    if err != nil {
        log.Fatalf("Reading plugin directory: %v", err)
    }

    for _, entry := range entries {
        if filepath.Ext(entry.Name()) == ".wasm" {
            name := entry.Name()[:len(entry.Name())-5] // remove .wasm
            path := filepath.Join(pluginDir, entry.Name())
            if err := host.LoadPlugin(ctx, name, path); err != nil {
                log.Printf("Failed to load plugin %s: %v", name, err)
                continue
            }
        }
    }

    // Process data through a plugin
    input := map[string]interface{}{
        "message": "Hello, Plugin!",
        "count":   42,
    }

    for name := range host.plugins {
        output, err := host.SendData(ctx, name, input)
        if err != nil {
            log.Printf("Plugin %s error: %v", name, err)
            continue
        }
        fmt.Printf("Plugin %s output: %s\n", name, string(output))
    }

    // Clean up
    for name := range host.plugins {
        host.UnloadPlugin(ctx, name)
    }
}
```

### 8.6 Wazero Compilation Modes

Wazero offers two compilation modes:

```go
package main

import (
    "context"
    "github.com/tetratelabs/wazero"
)

func main() {
    ctx := context.Background()

    // Interpreter mode: slower execution, faster startup
    // Good for short-lived modules or development
    rInterp := wazero.NewRuntimeWithConfig(ctx,
        wazero.NewRuntimeConfigInterpreter())
    defer rInterp.Close(ctx)

    // Compiler mode (default): faster execution, slower startup
    // Good for long-running or frequently called modules
    rCompiler := wazero.NewRuntimeWithConfig(ctx,
        wazero.NewRuntimeConfigCompiler())
    defer rCompiler.Close(ctx)
}
```

### 8.7 Wazero Compilation Cache

Reuse compiled modules across restarts with a filesystem cache:

```go
package main

import (
    "context"
    "log"
    "os"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

func main() {
    ctx := context.Background()

    // Create a compilation cache on disk
    cache, err := wazero.NewCompilationCacheWithDir("/tmp/wazero-cache")
    if err != nil {
        log.Fatal(err)
    }

    // Use the cache in the runtime config
    config := wazero.NewRuntimeConfig().WithCompilationCache(cache)
    r := wazero.NewRuntimeWithConfig(ctx, config)
    defer r.Close(ctx)

    wasi_snapshot_preview1.MustInstantiate(ctx, r)

    wasmBytes, err := os.ReadFile("module.wasm")
    if err != nil {
        log.Fatal(err)
    }

    // First run: compiles and caches
    // Subsequent runs: loads from cache (much faster startup)
    compiled, err := r.CompileModule(ctx, wasmBytes)
    if err != nil {
        log.Fatal(err)
    }

    modConfig := wazero.NewModuleConfig().WithStdout(os.Stdout)
    _, err = r.InstantiateModule(ctx, compiled, modConfig)
    if err != nil {
        log.Fatal(err)
    }
}
```

### 8.8 Memory Management with Wazero

```go
package main

import (
    "context"
    "encoding/binary"
    "fmt"
    "log"
    "os"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/api"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

// writeString writes a string to WASM memory and returns the pointer and length
func writeString(ctx context.Context, mod api.Module, s string) (uint32, uint32, error) {
    allocFn := mod.ExportedFunction("alloc")
    if allocFn == nil {
        return 0, 0, fmt.Errorf("alloc not exported")
    }

    size := uint64(len(s))
    results, err := allocFn.Call(ctx, size)
    if err != nil {
        return 0, 0, err
    }

    ptr := uint32(results[0])
    if !mod.Memory().Write(ptr, []byte(s)) {
        return 0, 0, fmt.Errorf("memory write failed")
    }

    return ptr, uint32(size), nil
}

// readString reads a string from WASM memory at the given pointer and length
func readString(mod api.Module, ptr, length uint32) (string, error) {
    buf, ok := mod.Memory().Read(ptr, length)
    if !ok {
        return "", fmt.Errorf("memory read failed at %d (len %d)", ptr, length)
    }
    return string(buf), nil
}

// writeUint32 writes a uint32 to WASM memory
func writeUint32(mod api.Module, ptr, value uint32) error {
    buf := make([]byte, 4)
    binary.LittleEndian.PutUint32(buf, value)
    if !mod.Memory().Write(ptr, buf) {
        return fmt.Errorf("memory write failed at %d", ptr)
    }
    return nil
}

// readUint32 reads a uint32 from WASM memory
func readUint32(mod api.Module, ptr uint32) (uint32, error) {
    buf, ok := mod.Memory().Read(ptr, 4)
    if !ok {
        return 0, fmt.Errorf("memory read failed at %d", ptr)
    }
    return binary.LittleEndian.Uint32(buf), nil
}

func main() {
    ctx := context.Background()

    r := wazero.NewRuntime(ctx)
    defer r.Close(ctx)

    wasi_snapshot_preview1.MustInstantiate(ctx, r)

    wasmBytes, err := os.ReadFile("module.wasm")
    if err != nil {
        log.Fatal(err)
    }

    config := wazero.NewModuleConfig().WithStdout(os.Stdout)
    mod, err := r.InstantiateWithConfig(ctx, wasmBytes, config)
    if err != nil {
        log.Fatal(err)
    }

    // Write a string to WASM memory
    ptr, length, err := writeString(ctx, mod, "Hello, WASM!")
    if err != nil {
        log.Fatal(err)
    }

    // Read it back
    str, err := readString(mod, ptr, length)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Read from WASM memory: %q\n", str)

    // Check memory size
    memSize := mod.Memory().Size()
    fmt.Printf("WASM memory size: %d bytes (%d pages)\n", memSize, memSize/65536)
}
```

---

## 9. Performance Considerations

### 9.1 Binary Size

Binary size is the most common concern with Go WASM. Here are strategies to reduce it.

#### Size Reduction Techniques

| Technique                        | Size Reduction | Notes                            |
|----------------------------------|---------------|----------------------------------|
| Use TinyGo                       | ~90%          | Best reduction, some limitations |
| Strip debug info (`-ldflags "-s -w"`) | ~20%     | Works with standard Go           |
| Use `wasm-opt -Oz`              | ~10-15%       | Post-processing with Binaryen    |
| Gzip/Brotli compression         | ~70-80%       | Applied by web server            |
| Avoid large stdlib packages     | Varies        | `fmt` alone adds ~400 KB         |

```bash
# Standard build
GOOS=js GOARCH=wasm go build -o main.wasm main.go
# Result: ~2.4 MB

# Strip debug info
GOOS=js GOARCH=wasm go build -ldflags="-s -w" -o main-stripped.wasm main.go
# Result: ~1.8 MB

# TinyGo build
tinygo build -o main-tiny.wasm -target wasm -no-debug main.go
# Result: ~250 KB

# TinyGo + wasm-opt
tinygo build -o main-tiny.wasm -target wasm -no-debug main.go
wasm-opt -Oz -o main-optimized.wasm main-tiny.wasm
# Result: ~200 KB

# Check compressed sizes (what users actually download)
gzip -k main.wasm && ls -lh main.wasm.gz
brotli -k main.wasm && ls -lh main.wasm.br
```

### 9.2 Startup Time

WASM modules need to be downloaded, compiled, and instantiated before they can run.

```
┌──────────┐    ┌──────────┐    ┌───────────────┐    ┌──────────┐
│ Download │───▶│  Compile │───▶│ Instantiate   │───▶│   Run    │
│  (slow)  │    │  (slow)  │    │   (fast)      │    │  (fast)  │
└──────────┘    └──────────┘    └───────────────┘    └──────────┘
     ▲                ▲
     │                │
  Smaller binary    Streaming compilation
  helps here       helps here
```

#### Techniques to Improve Startup Time

1. **Streaming compilation**: Use `WebAssembly.instantiateStreaming()` to compile while
   downloading.

2. **Caching compiled modules**: Cache the compiled module in IndexedDB.

```javascript
async function loadWasmCached(url) {
    const dbName = "wasm-cache";
    const storeName = "modules";
    const cacheKey = url;

    // Try to load from cache first
    const db = await openDB(dbName, storeName);
    const cached = await getFromDB(db, storeName, cacheKey);

    if (cached) {
        const go = new Go();
        const instance = await WebAssembly.instantiate(cached, go.importObject);
        go.run(instance);
        return;
    }

    // Download and compile
    const go = new Go();
    const response = await fetch(url);
    const bytes = await response.arrayBuffer();
    const module = await WebAssembly.compile(bytes);

    // Cache the compiled module
    await putInDB(db, storeName, cacheKey, module);

    // Instantiate and run
    const instance = await WebAssembly.instantiate(module, go.importObject);
    go.run(instance);
}

// IndexedDB helpers
function openDB(name, storeName) {
    return new Promise((resolve, reject) => {
        const request = indexedDB.open(name, 1);
        request.onupgradeneeded = () => {
            request.result.createObjectStore(storeName);
        };
        request.onsuccess = () => resolve(request.result);
        request.onerror = () => reject(request.error);
    });
}

function getFromDB(db, storeName, key) {
    return new Promise((resolve, reject) => {
        const tx = db.transaction(storeName, "readonly");
        const store = tx.objectStore(storeName);
        const request = store.get(key);
        request.onsuccess = () => resolve(request.result);
        request.onerror = () => reject(request.error);
    });
}

function putInDB(db, storeName, key, value) {
    return new Promise((resolve, reject) => {
        const tx = db.transaction(storeName, "readwrite");
        const store = tx.objectStore(storeName);
        const request = store.put(value, key);
        request.onsuccess = () => resolve();
        request.onerror = () => reject(request.error);
    });
}
```

3. **Lazy loading**: Only load WASM when the feature is needed.

```javascript
let wasmReady = null;

function ensureWasmLoaded() {
    if (!wasmReady) {
        wasmReady = new Promise(async (resolve) => {
            const go = new Go();
            const result = await WebAssembly.instantiateStreaming(
                fetch("main.wasm"),
                go.importObject
            );
            go.run(result.instance);
            resolve();
        });
    }
    return wasmReady;
}

// Usage: only loads WASM when the button is clicked
document.getElementById("processBtn").addEventListener("click", async () => {
    await ensureWasmLoaded();
    const result = goProcess(inputData);
    displayResult(result);
});
```

### 9.3 Runtime Performance

| Operation              | Native Go | Go WASM     | WASM vs Native |
|------------------------|-----------|-------------|----------------|
| Integer arithmetic     | 1x        | ~1.2-1.5x   | Competitive    |
| Floating-point math    | 1x        | ~1.1-1.3x   | Competitive    |
| String processing      | 1x        | ~2-3x       | Moderate        |
| Memory allocation      | 1x        | ~2-4x       | Slower          |
| Goroutine creation     | 1x        | ~3-5x       | Slower          |
| JS interop calls       | N/A       | ~10-100 us  | Significant overhead |

> **Tip**: Minimize the number of calls between Go and JavaScript. Batch operations
> and pass data in bulk rather than making many small calls.

### 9.4 Memory Usage

Go's WASM runtime allocates a linear memory that grows as needed. Key points:

- Initial memory: ~16 MB (configurable)
- Maximum memory: ~4 GB (WASM 32-bit address space limit)
- GC runs in the WASM linear memory
- Each goroutine stack: ~8 KB initial (grows as needed)

```go
package main

import (
    "fmt"
    "runtime"
    "syscall/js"
)

func memStats(this js.Value, args []js.Value) interface{} {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    stats := js.Global().Get("Object").New()
    stats.Set("allocMB", fmt.Sprintf("%.2f", float64(m.Alloc)/1024/1024))
    stats.Set("totalAllocMB", fmt.Sprintf("%.2f", float64(m.TotalAlloc)/1024/1024))
    stats.Set("sysMB", fmt.Sprintf("%.2f", float64(m.Sys)/1024/1024))
    stats.Set("numGC", m.NumGC)
    stats.Set("goroutines", runtime.NumGoroutine())

    return stats
}

func main() {
    js.Global().Set("goMemStats", js.FuncOf(memStats))
    <-make(chan struct{})
}
```

### 9.5 Benchmarking Go WASM

```go
package main

import (
    "fmt"
    "syscall/js"
    "time"
)

func benchmark(this js.Value, args []js.Value) interface{} {
    iterations := 1_000_000
    if len(args) > 0 {
        iterations = args[0].Int()
    }

    // Benchmark: pure computation (no JS interop)
    start := time.Now()
    sum := 0
    for i := 0; i < iterations; i++ {
        sum += i * i
    }
    computeTime := time.Since(start)

    // Benchmark: JS interop overhead
    console := js.Global().Get("console")
    start = time.Now()
    for i := 0; i < 1000; i++ {
        console.Call("log", i) // This is intentionally slow
    }
    interopTime := time.Since(start)

    result := js.Global().Get("Object").New()
    result.Set("computeMs", computeTime.Milliseconds())
    result.Set("interopMs", interopTime.Milliseconds())
    result.Set("iterations", iterations)
    result.Set("sum", sum)

    return result
}

func main() {
    js.Global().Set("goBenchmark", js.FuncOf(benchmark))
    fmt.Println("Benchmark functions registered")
    <-make(chan struct{})
}
```

---

## 10. Real-World Use Cases

### 10.1 Browser-Based Tools

#### Markdown Editor with Live Preview

```go
package main

import (
    "fmt"
    "strings"
    "syscall/js"
)

// SimpleMarkdown is a basic markdown-to-HTML converter
// (In production, use a proper library like goldmark)
func simpleMarkdownToHTML(md string) string {
    var html strings.Builder
    lines := strings.Split(md, "\n")

    inCodeBlock := false
    inList := false

    for _, line := range lines {
        trimmed := strings.TrimSpace(line)

        // Code blocks
        if strings.HasPrefix(trimmed, "```") {
            if inCodeBlock {
                html.WriteString("</code></pre>\n")
                inCodeBlock = false
            } else {
                lang := strings.TrimPrefix(trimmed, "```")
                html.WriteString(fmt.Sprintf(
                    `<pre><code class="language-%s">`, lang))
                inCodeBlock = true
            }
            continue
        }

        if inCodeBlock {
            html.WriteString(escapeHTML(line))
            html.WriteString("\n")
            continue
        }

        // Close list if needed
        if inList && !strings.HasPrefix(trimmed, "- ") && !strings.HasPrefix(trimmed, "* ") {
            html.WriteString("</ul>\n")
            inList = false
        }

        // Headings
        if strings.HasPrefix(trimmed, "### ") {
            html.WriteString(fmt.Sprintf("<h3>%s</h3>\n",
                escapeHTML(strings.TrimPrefix(trimmed, "### "))))
        } else if strings.HasPrefix(trimmed, "## ") {
            html.WriteString(fmt.Sprintf("<h2>%s</h2>\n",
                escapeHTML(strings.TrimPrefix(trimmed, "## "))))
        } else if strings.HasPrefix(trimmed, "# ") {
            html.WriteString(fmt.Sprintf("<h1>%s</h1>\n",
                escapeHTML(strings.TrimPrefix(trimmed, "# "))))
        } else if strings.HasPrefix(trimmed, "- ") || strings.HasPrefix(trimmed, "* ") {
            // Unordered list
            if !inList {
                html.WriteString("<ul>\n")
                inList = true
            }
            content := strings.TrimPrefix(strings.TrimPrefix(trimmed, "- "), "* ")
            html.WriteString(fmt.Sprintf("  <li>%s</li>\n", escapeHTML(content)))
        } else if strings.HasPrefix(trimmed, "> ") {
            // Blockquote
            content := strings.TrimPrefix(trimmed, "> ")
            html.WriteString(fmt.Sprintf("<blockquote>%s</blockquote>\n",
                escapeHTML(content)))
        } else if trimmed == "" {
            html.WriteString("<br>\n")
        } else {
            // Apply inline formatting
            formatted := applyInlineFormatting(trimmed)
            html.WriteString(fmt.Sprintf("<p>%s</p>\n", formatted))
        }
    }

    if inList {
        html.WriteString("</ul>\n")
    }
    if inCodeBlock {
        html.WriteString("</code></pre>\n")
    }

    return html.String()
}

func escapeHTML(s string) string {
    s = strings.ReplaceAll(s, "&", "&amp;")
    s = strings.ReplaceAll(s, "<", "&lt;")
    s = strings.ReplaceAll(s, ">", "&gt;")
    return s
}

func applyInlineFormatting(s string) string {
    // Bold: **text**
    for strings.Contains(s, "**") {
        start := strings.Index(s, "**")
        end := strings.Index(s[start+2:], "**")
        if end == -1 {
            break
        }
        end += start + 2
        content := s[start+2 : end]
        s = s[:start] + "<strong>" + content + "</strong>" + s[end+2:]
    }

    // Italic: *text*
    for strings.Contains(s, "*") {
        start := strings.Index(s, "*")
        end := strings.Index(s[start+1:], "*")
        if end == -1 {
            break
        }
        end += start + 1
        content := s[start+1 : end]
        s = s[:start] + "<em>" + content + "</em>" + s[end+1:]
    }

    // Code: `text`
    for strings.Contains(s, "`") {
        start := strings.Index(s, "`")
        end := strings.Index(s[start+1:], "`")
        if end == -1 {
            break
        }
        end += start + 1
        content := s[start+1 : end]
        s = s[:start] + "<code>" + escapeHTML(content) + "</code>" + s[end+1:]
    }

    return s
}

func main() {
    document := js.Global().Get("document")

    // Register markdown conversion function
    js.Global().Set("goMarkdownToHTML", js.FuncOf(
        func(this js.Value, args []js.Value) interface{} {
            if len(args) < 1 {
                return ""
            }
            return simpleMarkdownToHTML(args[0].String())
        }))

    // Set up live preview if the elements exist
    editor := document.Call("getElementById", "editor")
    preview := document.Call("getElementById", "preview")

    if !editor.IsNull() && !preview.IsNull() {
        editor.Call("addEventListener", "input",
            js.FuncOf(func(this js.Value, args []js.Value) interface{} {
                md := editor.Get("value").String()
                html := simpleMarkdownToHTML(md)
                preview.Set("innerHTML", html)
                return nil
            }))
    }

    fmt.Println("Markdown editor ready")
    <-make(chan struct{})
}
```

#### JSON Formatter/Validator

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "syscall/js"
)

func formatJSON(this js.Value, args []js.Value) interface{} {
    if len(args) < 1 {
        return map[string]interface{}{
            "error": "No input provided",
        }
    }

    input := args[0].String()
    indent := "  "
    if len(args) > 1 {
        indent = args[1].String()
    }

    // Validate the JSON
    var raw json.RawMessage
    if err := json.Unmarshal([]byte(input), &raw); err != nil {
        result := js.Global().Get("Object").New()
        result.Set("valid", false)
        result.Set("error", fmt.Sprintf("Invalid JSON: %v", err))
        return result
    }

    // Pretty-print
    var buf bytes.Buffer
    if err := json.Indent(&buf, raw, "", indent); err != nil {
        result := js.Global().Get("Object").New()
        result.Set("valid", false)
        result.Set("error", fmt.Sprintf("Format error: %v", err))
        return result
    }

    result := js.Global().Get("Object").New()
    result.Set("valid", true)
    result.Set("formatted", buf.String())
    result.Set("minified", string(raw))
    result.Set("byteSize", len(input))
    return result
}

func minifyJSON(this js.Value, args []js.Value) interface{} {
    if len(args) < 1 {
        return ""
    }

    input := args[0].String()
    var buf bytes.Buffer
    if err := json.Compact(&buf, []byte(input)); err != nil {
        return fmt.Sprintf("Error: %v", err)
    }
    return buf.String()
}

func main() {
    js.Global().Set("goFormatJSON", js.FuncOf(formatJSON))
    js.Global().Set("goMinifyJSON", js.FuncOf(minifyJSON))

    fmt.Println("JSON tools registered")
    <-make(chan struct{})
}
```

### 10.2 Edge Computing

#### Cloudflare Workers with Go WASM

Go can be compiled to WASM and deployed to Cloudflare Workers. While Cloudflare Workers
natively use JavaScript/V8, you can run WASM modules within them.

```go
// worker/main.go
package main

import (
    "encoding/json"
    "fmt"
    "net/url"
    "strings"
    "syscall/js"
)

type Request struct {
    Method  string            `json:"method"`
    URL     string            `json:"url"`
    Headers map[string]string `json:"headers"`
    Body    string            `json:"body"`
}

type Response struct {
    Status  int               `json:"status"`
    Headers map[string]string `json:"headers"`
    Body    string            `json:"body"`
}

func handleRequest(this js.Value, args []js.Value) interface{} {
    reqJSON := args[0].String()

    var req Request
    if err := json.Unmarshal([]byte(reqJSON), &req); err != nil {
        return errorResponse(400, "Invalid request")
    }

    parsedURL, err := url.Parse(req.URL)
    if err != nil {
        return errorResponse(400, "Invalid URL")
    }

    // Simple router
    path := parsedURL.Path
    switch {
    case path == "/" || path == "":
        return jsonResponse(200, map[string]string{
            "message": "Hello from Go WASM on the edge!",
            "runtime": "WebAssembly",
        })

    case strings.HasPrefix(path, "/api/upper"):
        text := parsedURL.Query().Get("text")
        return jsonResponse(200, map[string]string{
            "result": strings.ToUpper(text),
        })

    case strings.HasPrefix(path, "/api/reverse"):
        text := parsedURL.Query().Get("text")
        runes := []rune(text)
        for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
            runes[i], runes[j] = runes[j], runes[i]
        }
        return jsonResponse(200, map[string]string{
            "result": string(runes),
        })

    default:
        return errorResponse(404, "Not found")
    }
}

func jsonResponse(status int, data interface{}) interface{} {
    body, _ := json.Marshal(data)
    resp := Response{
        Status: status,
        Headers: map[string]string{
            "Content-Type": "application/json",
        },
        Body: string(body),
    }
    respJSON, _ := json.Marshal(resp)
    return string(respJSON)
}

func errorResponse(status int, message string) interface{} {
    return jsonResponse(status, map[string]string{"error": message})
}

func main() {
    js.Global().Set("goHandleRequest", js.FuncOf(handleRequest))
    fmt.Println("Edge worker ready")
    <-make(chan struct{})
}
```

### 10.3 Plugin Architecture

A practical example of loading user-defined plugins as WASM modules.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/api"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

// PluginManifest describes a plugin
type PluginManifest struct {
    Name        string   `json:"name"`
    Version     string   `json:"version"`
    Description string   `json:"description"`
    WasmFile    string   `json:"wasm_file"`
    Exports     []string `json:"exports"`
}

// PluginRegistry manages loaded plugins
type PluginRegistry struct {
    plugins  map[string]*LoadedPlugin
    registry map[string]PluginManifest
}

// LoadedPlugin represents a loaded and running plugin
type LoadedPlugin struct {
    Manifest PluginManifest
    Module   api.Module
    Runtime  wazero.Runtime
}

// NewPluginRegistry creates a new registry
func NewPluginRegistry() *PluginRegistry {
    return &PluginRegistry{
        plugins:  make(map[string]*LoadedPlugin),
        registry: make(map[string]PluginManifest),
    }
}

// Register adds a plugin manifest to the registry
func (r *PluginRegistry) Register(manifest PluginManifest) {
    r.registry[manifest.Name] = manifest
    fmt.Printf("Registered plugin: %s v%s - %s\n",
        manifest.Name, manifest.Version, manifest.Description)
}

// Load loads and instantiates a registered plugin
func (r *PluginRegistry) Load(ctx context.Context, name string) error {
    manifest, ok := r.registry[name]
    if !ok {
        return fmt.Errorf("plugin %q not registered", name)
    }

    wasmBytes, err := os.ReadFile(manifest.WasmFile)
    if err != nil {
        return fmt.Errorf("reading %s: %w", manifest.WasmFile, err)
    }

    runtime := wazero.NewRuntime(ctx)
    wasi_snapshot_preview1.MustInstantiate(ctx, runtime)

    // Provide host APIs to the plugin
    _, err = runtime.NewHostModuleBuilder("host").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context, m api.Module, ptr, len uint32) {
            msg, _ := readMemory(m, ptr, len)
            log.Printf("[plugin:%s] %s", name, msg)
        }).
        Export("log").
        NewFunctionBuilder().
        WithFunc(func(ctx context.Context, m api.Module, eventPtr, eventLen, dataPtr, dataLen uint32) {
            event, _ := readMemory(m, eventPtr, eventLen)
            data, _ := readMemory(m, dataPtr, dataLen)
            fmt.Printf("[plugin:%s] Event: %s, Data: %s\n", name, event, data)
        }).
        Export("emit_event").
        Instantiate(ctx)
    if err != nil {
        runtime.Close(ctx)
        return fmt.Errorf("host module for %s: %w", name, err)
    }

    config := wazero.NewModuleConfig().
        WithStdout(os.Stdout).
        WithStderr(os.Stderr).
        WithName(name)

    mod, err := runtime.InstantiateWithConfig(ctx, wasmBytes, config)
    if err != nil {
        runtime.Close(ctx)
        return fmt.Errorf("instantiating %s: %w", name, err)
    }

    r.plugins[name] = &LoadedPlugin{
        Manifest: manifest,
        Module:   mod,
        Runtime:  runtime,
    }

    fmt.Printf("Loaded plugin: %s\n", name)
    return nil
}

// Call invokes a function on a loaded plugin
func (r *PluginRegistry) Call(ctx context.Context, pluginName, funcName string, args ...uint64) ([]uint64, error) {
    plugin, ok := r.plugins[pluginName]
    if !ok {
        return nil, fmt.Errorf("plugin %q not loaded", pluginName)
    }

    fn := plugin.Module.ExportedFunction(funcName)
    if fn == nil {
        return nil, fmt.Errorf("function %q not found in plugin %q", funcName, pluginName)
    }

    return fn.Call(ctx, args...)
}

// Unload removes a plugin and frees resources
func (r *PluginRegistry) Unload(ctx context.Context, name string) error {
    plugin, ok := r.plugins[name]
    if !ok {
        return fmt.Errorf("plugin %q not loaded", name)
    }

    err := plugin.Runtime.Close(ctx)
    delete(r.plugins, name)
    return err
}

// ListPlugins returns information about loaded plugins
func (r *PluginRegistry) ListPlugins() []PluginManifest {
    var manifests []PluginManifest
    for _, p := range r.plugins {
        manifests = append(manifests, p.Manifest)
    }
    return manifests
}

func readMemory(mod api.Module, ptr, length uint32) (string, bool) {
    buf, ok := mod.Memory().Read(ptr, length)
    if !ok {
        return "", false
    }
    return string(buf), true
}

func main() {
    ctx := context.Background()
    registry := NewPluginRegistry()

    // Load plugin manifests from a config file
    configData, err := os.ReadFile("plugins.json")
    if err != nil {
        log.Fatalf("Reading plugin config: %v", err)
    }

    var manifests []PluginManifest
    if err := json.Unmarshal(configData, &manifests); err != nil {
        log.Fatalf("Parsing plugin config: %v", err)
    }

    // Register and load plugins
    for _, m := range manifests {
        registry.Register(m)
        if err := registry.Load(ctx, m.Name); err != nil {
            log.Printf("Failed to load %s: %v", m.Name, err)
        }
    }

    // Use plugins
    loaded := registry.ListPlugins()
    fmt.Printf("\nLoaded %d plugins:\n", len(loaded))
    for _, p := range loaded {
        fmt.Printf("  - %s v%s: %s\n", p.Name, p.Version, p.Description)
    }

    // Clean up
    for _, p := range loaded {
        registry.Unload(ctx, p.Name)
    }
}
```

**plugins.json**:
```json
[
    {
        "name": "text-processor",
        "version": "1.0.0",
        "description": "Text processing utilities",
        "wasm_file": "plugins/text-processor.wasm",
        "exports": ["to_upper", "to_lower", "word_count"]
    },
    {
        "name": "image-filter",
        "version": "2.1.0",
        "description": "Image filtering and transformation",
        "wasm_file": "plugins/image-filter.wasm",
        "exports": ["grayscale", "blur", "sharpen"]
    }
]
```

### 10.4 Serverless Functions

Go WASM modules can be used as serverless functions with platforms that support WASM.

```go
// A serverless function that processes webhook payloads
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "os"
    "strings"
    "time"
)

// WebhookPayload represents an incoming webhook
type WebhookPayload struct {
    Event     string          `json:"event"`
    Timestamp string          `json:"timestamp"`
    Data      json.RawMessage `json:"data"`
    Signature string          `json:"signature"`
}

// ProcessResult is the output of processing
type ProcessResult struct {
    Accepted  bool   `json:"accepted"`
    EventType string `json:"event_type"`
    Message   string `json:"message"`
    ProcessedAt string `json:"processed_at"`
}

func verifySignature(payload []byte, signature, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}

func processWebhook(payload WebhookPayload) ProcessResult {
    result := ProcessResult{
        EventType:   payload.Event,
        ProcessedAt: time.Now().UTC().Format(time.RFC3339),
    }

    switch payload.Event {
    case "user.created":
        var data struct {
            UserID string `json:"user_id"`
            Email  string `json:"email"`
        }
        json.Unmarshal(payload.Data, &data)
        result.Accepted = true
        result.Message = fmt.Sprintf("Welcome user %s (%s)", data.UserID, data.Email)

    case "order.completed":
        var data struct {
            OrderID string  `json:"order_id"`
            Amount  float64 `json:"amount"`
        }
        json.Unmarshal(payload.Data, &data)
        result.Accepted = true
        result.Message = fmt.Sprintf("Order %s completed: $%.2f", data.OrderID, data.Amount)

    default:
        result.Accepted = false
        result.Message = fmt.Sprintf("Unknown event: %s", payload.Event)
    }

    return result
}

func main() {
    // Read input from stdin (common pattern for WASI serverless)
    var input strings.Builder
    buf := make([]byte, 4096)
    for {
        n, err := os.Stdin.Read(buf)
        if n > 0 {
            input.Write(buf[:n])
        }
        if err != nil {
            break
        }
    }

    var payload WebhookPayload
    if err := json.Unmarshal([]byte(input.String()), &payload); err != nil {
        fmt.Fprintf(os.Stderr, "Invalid input: %v\n", err)
        os.Exit(1)
    }

    result := processWebhook(payload)

    output, _ := json.MarshalIndent(result, "", "  ")
    fmt.Println(string(output))
}
```

Build and test:
```bash
GOOS=wasip1 GOARCH=wasm go build -o webhook.wasm webhook.go

echo '{"event":"user.created","data":{"user_id":"123","email":"alice@example.com"}}' \
  | wasmtime webhook.wasm
```

---

## 11. Current Limitations and Future of Go + WASM

### 11.1 Current Limitations

| Limitation                          | Details                                              | Workaround                    |
|-------------------------------------|------------------------------------------------------|-------------------------------|
| **Large binary size**               | Standard Go: 2-10 MB for simple programs             | Use TinyGo, compression       |
| **Single-threaded execution**       | WASM runs on one thread; no goroutine parallelism    | Use Web Workers for parallelism |
| **No direct DOM access**            | Must go through `syscall/js` (slow interop)          | Minimize JS boundary crossings|
| **No network in WASI preview 1**    | Sockets are not part of WASIp1                       | Use host functions or WASIp2  |
| **Garbage collector pauses**        | GC runs in the WASM module, can cause jank           | Minimize allocations          |
| **Limited debugging**               | Source maps not fully supported                      | Use `console.log` debugging   |
| **No `plugin` package**             | Cannot dynamically load Go plugins in WASM           | Use wazero for plugin systems |
| **No `unsafe` pointer operations**  | Limited `unsafe` package support in WASM             | Redesign without `unsafe`     |

### 11.2 The Threading Proposal

The WebAssembly threads proposal (now at Phase 4) adds:
- Shared linear memory between WASM instances
- Atomic operations
- `wait` and `notify` instructions

This will eventually allow Go's goroutine scheduler to use true parallelism in WASM.

### 11.3 The Component Model

The WebAssembly Component Model is the successor to the current "core" WASM specification.
It introduces:

- **Interface types**: Rich type system beyond i32/i64/f32/f64
- **Components**: Higher-level modules with well-defined imports/exports
- **WIT (WASM Interface Type)**: IDL for defining component interfaces
- **Composability**: Link components together without shared memory

```wit
// Example WIT interface definition
package example:text-processor@1.0.0;

interface processor {
    record options {
        uppercase: bool,
        trim: bool,
        max-length: option<u32>,
    }

    process: func(input: string, opts: options) -> result<string, string>;
    word-count: func(input: string) -> u32;
}

world text-plugin {
    export processor;
}
```

### 11.4 WASI Preview 2

WASI Preview 2 (WASI 0.2) introduces:
- **wasi:http**: HTTP client and server
- **wasi:sockets**: TCP/UDP networking
- **wasi:io**: Streams and polling
- **wasi:clocks**: Monotonic and wall clocks
- **wasi:filesystem**: Enhanced file system access

Go support for WASI Preview 2 is expected in future releases.

### 11.5 Go-Specific Roadmap

Key areas of improvement for Go's WASM support:

1. **Smaller binaries**: Ongoing work to reduce the runtime footprint.
2. **Better GC**: Integrating with the WASM GC proposal for lower overhead.
3. **WASI Preview 2**: Full support for the new WASI APIs.
4. **Threads**: Leveraging WASM threads for goroutine parallelism.
5. **64-bit memory**: Support for memory64 proposal (>4 GB).

### 11.6 When to Use Go WASM (Decision Guide)

```
Should you use Go + WASM?

Start here:
│
├─ Running in browser?
│  ├─ Yes, heavy computation? ──────────── YES (use Go or TinyGo)
│  ├─ Yes, lots of DOM manipulation? ──── MAYBE (JS is faster for DOM)
│  └─ Yes, porting existing Go code? ──── YES (minimal rewrite)
│
├─ Plugin system needed?
│  ├─ Yes, must be sandboxed? ─────────── YES (use wazero)
│  ├─ Yes, multi-language support? ────── YES (WASM is polyglot)
│  └─ Yes, but Go-only? ──────────────── MAYBE (consider native plugins)
│
├─ Edge computing?
│  ├─ Yes, Cloudflare/Fastly? ─────────── YES (WASM is first-class)
│  └─ Yes, custom infrastructure? ────── CONSIDER (depends on runtime)
│
├─ Server-side?
│  ├─ Yes, sandboxed execution? ───────── YES
│  ├─ Yes, needs networking? ──────────── NOT YET (wait for WASIp2)
│  └─ Yes, for performance? ──────────── NO (native Go is faster)
│
└─ Binary size matters?
   ├─ Yes, < 500 KB needed? ──────────── Use TinyGo
   ├─ Yes, < 5 MB acceptable? ────────── Standard Go + compression
   └─ No ──────────────────────────────── Standard Go is fine
```

---

## 12. Complete Example Projects

### 12.1 Project: Interactive Image Processing Tool (Browser)

A complete browser-based image processing tool written in Go WASM.

**main.go**:
```go
package main

import (
    "fmt"
    "math"
    "syscall/js"
)

var document js.Value

func init() {
    document = js.Global().Get("document")
}

// applyGrayscale converts image data to grayscale
func applyGrayscale(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    data := imageData.Get("data")
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    for i := 0; i < length; i += 4 {
        r := float64(pixels[i])
        g := float64(pixels[i+1])
        b := float64(pixels[i+2])
        // Luminosity method
        gray := byte(0.299*r + 0.587*g + 0.114*b)
        pixels[i] = gray
        pixels[i+1] = gray
        pixels[i+2] = gray
    }

    js.CopyBytesToJS(data, pixels)
    return imageData
}

// applySepia applies a sepia tone filter
func applySepia(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    data := imageData.Get("data")
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    for i := 0; i < length; i += 4 {
        r := float64(pixels[i])
        g := float64(pixels[i+1])
        b := float64(pixels[i+2])

        newR := math.Min(255, 0.393*r+0.769*g+0.189*b)
        newG := math.Min(255, 0.349*r+0.686*g+0.168*b)
        newB := math.Min(255, 0.272*r+0.534*g+0.131*b)

        pixels[i] = byte(newR)
        pixels[i+1] = byte(newG)
        pixels[i+2] = byte(newB)
    }

    js.CopyBytesToJS(data, pixels)
    return imageData
}

// applyBrightness adjusts brightness (-100 to +100)
func applyBrightness(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    adjustment := args[1].Float()
    data := imageData.Get("data")
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    for i := 0; i < length; i += 4 {
        pixels[i] = clampByte(float64(pixels[i]) + adjustment)
        pixels[i+1] = clampByte(float64(pixels[i+1]) + adjustment)
        pixels[i+2] = clampByte(float64(pixels[i+2]) + adjustment)
    }

    js.CopyBytesToJS(data, pixels)
    return imageData
}

// applyContrast adjusts contrast (-100 to +100)
func applyContrast(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    adjustment := args[1].Float()
    data := imageData.Get("data")
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    factor := (259 * (adjustment + 255)) / (255 * (259 - adjustment))

    for i := 0; i < length; i += 4 {
        pixels[i] = clampByte(factor*(float64(pixels[i])-128) + 128)
        pixels[i+1] = clampByte(factor*(float64(pixels[i+1])-128) + 128)
        pixels[i+2] = clampByte(factor*(float64(pixels[i+2])-128) + 128)
    }

    js.CopyBytesToJS(data, pixels)
    return imageData
}

// applyBlur applies a simple box blur
func applyBlur(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    radius := 1
    if len(args) > 1 {
        radius = args[1].Int()
    }

    data := imageData.Get("data")
    width := imageData.Get("width").Int()
    height := imageData.Get("height").Int()
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    output := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    for y := 0; y < height; y++ {
        for x := 0; x < width; x++ {
            var rSum, gSum, bSum float64
            var count float64

            for dy := -radius; dy <= radius; dy++ {
                for dx := -radius; dx <= radius; dx++ {
                    nx := x + dx
                    ny := y + dy
                    if nx >= 0 && nx < width && ny >= 0 && ny < height {
                        idx := (ny*width + nx) * 4
                        rSum += float64(pixels[idx])
                        gSum += float64(pixels[idx+1])
                        bSum += float64(pixels[idx+2])
                        count++
                    }
                }
            }

            idx := (y*width + x) * 4
            output[idx] = byte(rSum / count)
            output[idx+1] = byte(gSum / count)
            output[idx+2] = byte(bSum / count)
            output[idx+3] = pixels[idx+3] // preserve alpha
        }
    }

    js.CopyBytesToJS(data, output)
    return imageData
}

// applyInvert inverts all colors
func applyInvert(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    data := imageData.Get("data")
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    for i := 0; i < length; i += 4 {
        pixels[i] = 255 - pixels[i]
        pixels[i+1] = 255 - pixels[i+1]
        pixels[i+2] = 255 - pixels[i+2]
    }

    js.CopyBytesToJS(data, pixels)
    return imageData
}

// getHistogram computes RGB histogram data
func getHistogram(this js.Value, args []js.Value) interface{} {
    imageData := args[0]
    data := imageData.Get("data")
    length := data.Get("length").Int()

    pixels := make([]byte, length)
    js.CopyBytesToGo(pixels, data)

    rHist := make([]int, 256)
    gHist := make([]int, 256)
    bHist := make([]int, 256)

    for i := 0; i < length; i += 4 {
        rHist[pixels[i]]++
        gHist[pixels[i+1]]++
        bHist[pixels[i+2]]++
    }

    result := js.Global().Get("Object").New()

    rArray := js.Global().Get("Array").New(256)
    gArray := js.Global().Get("Array").New(256)
    bArray := js.Global().Get("Array").New(256)

    for i := 0; i < 256; i++ {
        rArray.SetIndex(i, rHist[i])
        gArray.SetIndex(i, gHist[i])
        bArray.SetIndex(i, bHist[i])
    }

    result.Set("red", rArray)
    result.Set("green", gArray)
    result.Set("blue", bArray)

    return result
}

func clampByte(v float64) byte {
    if v < 0 {
        return 0
    }
    if v > 255 {
        return 255
    }
    return byte(v)
}

func main() {
    fmt.Println("Image processing WASM module loaded")

    // Register all filter functions
    js.Global().Set("goGrayscale", js.FuncOf(applyGrayscale))
    js.Global().Set("goSepia", js.FuncOf(applySepia))
    js.Global().Set("goBrightness", js.FuncOf(applyBrightness))
    js.Global().Set("goContrast", js.FuncOf(applyContrast))
    js.Global().Set("goBlur", js.FuncOf(applyBlur))
    js.Global().Set("goInvert", js.FuncOf(applyInvert))
    js.Global().Set("goHistogram", js.FuncOf(getHistogram))

    fmt.Println("All image filters registered")

    // Keep alive
    <-make(chan struct{})
}
```

**index.html** (for the image processing tool):
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Go WASM Image Processor</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, sans-serif;
            max-width: 900px;
            margin: 0 auto;
            padding: 20px;
            background: #f5f5f5;
        }
        h1 { color: #333; }
        .controls {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
            margin: 20px 0;
        }
        button {
            padding: 8px 16px;
            border: none;
            border-radius: 4px;
            background: #2563eb;
            color: white;
            cursor: pointer;
            font-size: 14px;
        }
        button:hover { background: #1d4ed8; }
        canvas {
            border: 2px solid #ccc;
            border-radius: 4px;
            max-width: 100%;
        }
        .slider-group {
            display: flex;
            align-items: center;
            gap: 10px;
        }
        input[type="range"] { width: 200px; }
        #status { color: #666; font-style: italic; }
    </style>
</head>
<body>
    <h1>Go WASM Image Processor</h1>
    <p id="status">Loading WebAssembly...</p>

    <input type="file" id="fileInput" accept="image/*">

    <div class="controls">
        <button onclick="applyFilter('grayscale')">Grayscale</button>
        <button onclick="applyFilter('sepia')">Sepia</button>
        <button onclick="applyFilter('invert')">Invert</button>
        <button onclick="applyFilter('blur')">Blur</button>
        <button onclick="resetImage()">Reset</button>
    </div>

    <div class="controls">
        <div class="slider-group">
            <label>Brightness:</label>
            <input type="range" id="brightness" min="-100" max="100" value="0"
                   oninput="applyAdjustment()">
            <span id="brightnessValue">0</span>
        </div>
        <div class="slider-group">
            <label>Contrast:</label>
            <input type="range" id="contrast" min="-100" max="100" value="0"
                   oninput="applyAdjustment()">
            <span id="contrastValue">0</span>
        </div>
    </div>

    <canvas id="canvas"></canvas>

    <script src="wasm_exec.js"></script>
    <script>
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let originalImageData = null;

        // Load WASM
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("main.wasm"), go.importObject)
            .then(result => {
                go.run(result.instance);
                document.getElementById('status').textContent = 'Ready!';
            });

        // File input handler
        document.getElementById('fileInput').addEventListener('change', (e) => {
            const file = e.target.files[0];
            if (!file) return;

            const img = new Image();
            img.onload = () => {
                canvas.width = img.width;
                canvas.height = img.height;
                ctx.drawImage(img, 0, 0);
                originalImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
            };
            img.src = URL.createObjectURL(file);
        });

        function applyFilter(name) {
            if (!originalImageData) return;

            const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

            switch (name) {
                case 'grayscale': goGrayscale(imageData); break;
                case 'sepia':     goSepia(imageData); break;
                case 'invert':    goInvert(imageData); break;
                case 'blur':      goBlur(imageData, 2); break;
            }

            ctx.putImageData(imageData, 0, 0);
        }

        function applyAdjustment() {
            if (!originalImageData) return;

            const brightness = parseInt(document.getElementById('brightness').value);
            const contrast = parseInt(document.getElementById('contrast').value);

            document.getElementById('brightnessValue').textContent = brightness;
            document.getElementById('contrastValue').textContent = contrast;

            // Start from original
            const imageData = new ImageData(
                new Uint8ClampedArray(originalImageData.data),
                originalImageData.width,
                originalImageData.height
            );

            if (brightness !== 0) goBrightness(imageData, brightness);
            if (contrast !== 0)   goContrast(imageData, contrast);

            ctx.putImageData(imageData, 0, 0);
        }

        function resetImage() {
            if (!originalImageData) return;
            ctx.putImageData(originalImageData, 0, 0);
            document.getElementById('brightness').value = 0;
            document.getElementById('contrast').value = 0;
            document.getElementById('brightnessValue').textContent = '0';
            document.getElementById('contrastValue').textContent = '0';
        }
    </script>
</body>
</html>
```

### 12.2 Project: WASM-Based Rule Engine (Server-Side with Wazero)

A complete rule engine that evaluates user-defined rules compiled to WASM.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "sync"
    "time"

    "github.com/tetratelabs/wazero"
    "github.com/tetratelabs/wazero/api"
    "github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
)

// Rule represents a WASM-based rule
type Rule struct {
    Name        string `json:"name"`
    Description string `json:"description"`
    WasmPath    string `json:"wasm_path"`
    Priority    int    `json:"priority"`
}

// RuleResult is the outcome of a rule evaluation
type RuleResult struct {
    RuleName  string        `json:"rule_name"`
    Matched   bool          `json:"matched"`
    Score     float64       `json:"score"`
    Action    string        `json:"action"`
    Duration  time.Duration `json:"duration"`
}

// RuleEngine manages and executes WASM rules
type RuleEngine struct {
    mu       sync.RWMutex
    rules    []Rule
    compiled map[string]wazero.CompiledModule
    runtime  wazero.Runtime
    cache    wazero.CompilationCache
}

// NewRuleEngine creates a new rule engine
func NewRuleEngine(ctx context.Context) (*RuleEngine, error) {
    cache, err := wazero.NewCompilationCacheWithDir("/tmp/rule-engine-cache")
    if err != nil {
        return nil, fmt.Errorf("creating cache: %w", err)
    }

    config := wazero.NewRuntimeConfig().WithCompilationCache(cache)
    r := wazero.NewRuntimeWithConfig(ctx, config)

    wasi_snapshot_preview1.MustInstantiate(ctx, r)

    return &RuleEngine{
        compiled: make(map[string]wazero.CompiledModule),
        runtime:  r,
        cache:    cache,
    }, nil
}

// LoadRules loads and compiles all rule modules
func (e *RuleEngine) LoadRules(ctx context.Context, rules []Rule) error {
    e.mu.Lock()
    defer e.mu.Unlock()

    for _, rule := range rules {
        wasmBytes, err := os.ReadFile(rule.WasmPath)
        if err != nil {
            return fmt.Errorf("reading %s: %w", rule.WasmPath, err)
        }

        compiled, err := e.runtime.CompileModule(ctx, wasmBytes)
        if err != nil {
            return fmt.Errorf("compiling %s: %w", rule.Name, err)
        }

        e.compiled[rule.Name] = compiled
        log.Printf("Rule compiled: %s", rule.Name)
    }

    e.rules = rules
    return nil
}

// Evaluate runs all rules against the given input
func (e *RuleEngine) Evaluate(ctx context.Context, input map[string]interface{}) ([]RuleResult, error) {
    e.mu.RLock()
    defer e.mu.RUnlock()

    inputJSON, err := json.Marshal(input)
    if err != nil {
        return nil, fmt.Errorf("marshaling input: %w", err)
    }

    var results []RuleResult

    for _, rule := range e.rules {
        compiled, ok := e.compiled[rule.Name]
        if !ok {
            continue
        }

        start := time.Now()
        result, err := e.evaluateRule(ctx, compiled, rule, inputJSON)
        if err != nil {
            log.Printf("Rule %s failed: %v", rule.Name, err)
            continue
        }
        result.Duration = time.Since(start)

        results = append(results, result)
    }

    return results, nil
}

func (e *RuleEngine) evaluateRule(ctx context.Context, compiled wazero.CompiledModule, rule Rule, inputJSON []byte) (RuleResult, error) {
    // Each evaluation gets its own module instance (isolation)
    config := wazero.NewModuleConfig().
        WithName(rule.Name + "-" + fmt.Sprint(time.Now().UnixNano())).
        WithStdout(os.Stdout).
        WithStderr(os.Stderr)

    mod, err := e.runtime.InstantiateModule(ctx, compiled, config)
    if err != nil {
        return RuleResult{}, fmt.Errorf("instantiating: %w", err)
    }
    defer mod.Close(ctx)

    // Allocate memory for input
    allocFn := mod.ExportedFunction("alloc")
    if allocFn == nil {
        return RuleResult{}, fmt.Errorf("alloc function not found")
    }

    results, err := allocFn.Call(ctx, uint64(len(inputJSON)))
    if err != nil {
        return RuleResult{}, fmt.Errorf("alloc: %w", err)
    }
    inputPtr := uint32(results[0])

    // Write input to WASM memory
    if !mod.Memory().Write(inputPtr, inputJSON) {
        return RuleResult{}, fmt.Errorf("memory write failed")
    }

    // Call the evaluate function
    evalFn := mod.ExportedFunction("evaluate")
    if evalFn == nil {
        return RuleResult{}, fmt.Errorf("evaluate function not found")
    }

    results, err = evalFn.Call(ctx, uint64(inputPtr), uint64(len(inputJSON)))
    if err != nil {
        return RuleResult{}, fmt.Errorf("evaluate: %w", err)
    }

    // Read the result
    resultPtr := uint32(results[0])
    getLenFn := mod.ExportedFunction("get_result_len")
    if getLenFn == nil {
        return RuleResult{}, fmt.Errorf("get_result_len not found")
    }

    results, err = getLenFn.Call(ctx)
    if err != nil {
        return RuleResult{}, fmt.Errorf("get_result_len: %w", err)
    }
    resultLen := uint32(results[0])

    resultBytes, ok := mod.Memory().Read(resultPtr, resultLen)
    if !ok {
        return RuleResult{}, fmt.Errorf("reading result failed")
    }

    var ruleResult RuleResult
    if err := json.Unmarshal(resultBytes, &ruleResult); err != nil {
        return RuleResult{}, fmt.Errorf("unmarshaling result: %w", err)
    }
    ruleResult.RuleName = rule.Name

    return ruleResult, nil
}

// Close releases all resources
func (e *RuleEngine) Close(ctx context.Context) error {
    return e.runtime.Close(ctx)
}

func main() {
    ctx := context.Background()

    engine, err := NewRuleEngine(ctx)
    if err != nil {
        log.Fatalf("Creating engine: %v", err)
    }
    defer engine.Close(ctx)

    // Load rules from configuration
    rules := []Rule{
        {
            Name:        "fraud-detection",
            Description: "Detects potentially fraudulent transactions",
            WasmPath:    "rules/fraud.wasm",
            Priority:    1,
        },
        {
            Name:        "rate-limiting",
            Description: "Applies rate limiting rules",
            WasmPath:    "rules/ratelimit.wasm",
            Priority:    2,
        },
        {
            Name:        "geo-blocking",
            Description: "Blocks requests from certain regions",
            WasmPath:    "rules/geoblock.wasm",
            Priority:    3,
        },
    }

    if err := engine.LoadRules(ctx, rules); err != nil {
        log.Fatalf("Loading rules: %v", err)
    }

    // Evaluate rules against a sample input
    input := map[string]interface{}{
        "user_id":     "user-123",
        "amount":      999.99,
        "currency":    "USD",
        "country":     "US",
        "ip_address":  "192.168.1.1",
        "device_type": "mobile",
    }

    results, err := engine.Evaluate(ctx, input)
    if err != nil {
        log.Fatalf("Evaluation failed: %v", err)
    }

    fmt.Println("\n=== Rule Evaluation Results ===")
    for _, r := range results {
        fmt.Printf("Rule: %-20s | Matched: %-5v | Score: %.2f | Action: %-10s | Time: %v\n",
            r.RuleName, r.Matched, r.Score, r.Action, r.Duration)
    }
}
```

### 12.3 Project: Go WASM Canvas Game (Browser)

A simple bouncing ball animation rendered entirely from Go.

**game.go**:
```go
package main

import (
    "fmt"
    "math"
    "math/rand"
    "syscall/js"
)

const (
    canvasWidth  = 800
    canvasHeight = 600
)

// Ball represents a bouncing ball
type Ball struct {
    X, Y   float64
    VX, VY float64
    Radius float64
    Color  string
}

// Game holds the game state
type Game struct {
    canvas    js.Value
    ctx       js.Value
    balls     []*Ball
    animFrame js.Func
    running   bool
}

func NewGame() *Game {
    document := js.Global().Get("document")

    canvas := document.Call("getElementById", "gameCanvas")
    canvas.Set("width", canvasWidth)
    canvas.Set("height", canvasHeight)

    ctx := canvas.Call("getContext", "2d")

    g := &Game{
        canvas: canvas,
        ctx:    ctx,
    }

    // Create initial balls
    colors := []string{
        "#ef4444", "#f97316", "#eab308", "#22c55e",
        "#3b82f6", "#8b5cf6", "#ec4899", "#14b8a6",
    }

    for i := 0; i < 20; i++ {
        radius := 10 + rand.Float64()*30
        g.balls = append(g.balls, &Ball{
            X:      radius + rand.Float64()*(canvasWidth-2*radius),
            Y:      radius + rand.Float64()*(canvasHeight-2*radius),
            VX:     (rand.Float64() - 0.5) * 6,
            VY:     (rand.Float64() - 0.5) * 6,
            Radius: radius,
            Color:  colors[i%len(colors)],
        })
    }

    return g
}

func (g *Game) update() {
    for _, ball := range g.balls {
        ball.X += ball.VX
        ball.Y += ball.VY

        // Bounce off walls
        if ball.X-ball.Radius < 0 {
            ball.X = ball.Radius
            ball.VX = math.Abs(ball.VX)
        }
        if ball.X+ball.Radius > canvasWidth {
            ball.X = canvasWidth - ball.Radius
            ball.VX = -math.Abs(ball.VX)
        }
        if ball.Y-ball.Radius < 0 {
            ball.Y = ball.Radius
            ball.VY = math.Abs(ball.VY)
        }
        if ball.Y+ball.Radius > canvasHeight {
            ball.Y = canvasHeight - ball.Radius
            ball.VY = -math.Abs(ball.VY)
        }

        // Simple ball-to-ball collision
        for _, other := range g.balls {
            if ball == other {
                continue
            }
            dx := other.X - ball.X
            dy := other.Y - ball.Y
            dist := math.Sqrt(dx*dx + dy*dy)
            minDist := ball.Radius + other.Radius

            if dist < minDist && dist > 0 {
                // Normalize
                nx := dx / dist
                ny := dy / dist

                // Separate balls
                overlap := minDist - dist
                ball.X -= nx * overlap * 0.5
                ball.Y -= ny * overlap * 0.5
                other.X += nx * overlap * 0.5
                other.Y += ny * overlap * 0.5

                // Swap velocity components along collision normal
                dvx := ball.VX - other.VX
                dvy := ball.VY - other.VY
                dvn := dvx*nx + dvy*ny

                ball.VX -= dvn * nx
                ball.VY -= dvn * ny
                other.VX += dvn * nx
                other.VY += dvn * ny
            }
        }
    }
}

func (g *Game) draw() {
    ctx := g.ctx

    // Clear canvas
    ctx.Set("fillStyle", "#1a1a2e")
    ctx.Call("fillRect", 0, 0, canvasWidth, canvasHeight)

    // Draw balls with gradient
    for _, ball := range g.balls {
        gradient := ctx.Call("createRadialGradient",
            ball.X-ball.Radius*0.3, ball.Y-ball.Radius*0.3, ball.Radius*0.1,
            ball.X, ball.Y, ball.Radius)
        gradient.Call("addColorStop", 0, "white")
        gradient.Call("addColorStop", 0.3, ball.Color)
        gradient.Call("addColorStop", 1, "rgba(0,0,0,0.3)")

        ctx.Call("beginPath")
        ctx.Call("arc", ball.X, ball.Y, ball.Radius, 0, 2*math.Pi)
        ctx.Set("fillStyle", gradient)
        ctx.Call("fill")
    }

    // Draw info
    ctx.Set("fillStyle", "white")
    ctx.Set("font", "14px monospace")
    ctx.Call("fillText", fmt.Sprintf("Balls: %d", len(g.balls)), 10, 20)
}

func (g *Game) gameLoop(this js.Value, args []js.Value) interface{} {
    if !g.running {
        return nil
    }

    g.update()
    g.draw()

    js.Global().Call("requestAnimationFrame", g.animFrame)
    return nil
}

func (g *Game) Start() {
    g.running = true
    g.animFrame = js.FuncOf(g.gameLoop)
    js.Global().Call("requestAnimationFrame", g.animFrame)
}

func (g *Game) Stop() {
    g.running = false
}

func (g *Game) AddBall() {
    radius := 10 + rand.Float64()*30
    g.balls = append(g.balls, &Ball{
        X:      canvasWidth / 2,
        Y:      canvasHeight / 2,
        VX:     (rand.Float64() - 0.5) * 8,
        VY:     (rand.Float64() - 0.5) * 8,
        Radius: radius,
        Color:  fmt.Sprintf("hsl(%d, 70%%, 60%%)", rand.Intn(360)),
    })
}

func main() {
    game := NewGame()

    // Expose controls to JavaScript
    js.Global().Set("gameStart", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        game.Start()
        return nil
    }))

    js.Global().Set("gameStop", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        game.Stop()
        return nil
    }))

    js.Global().Set("gameAddBall", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        game.AddBall()
        return nil
    }))

    fmt.Println("Game loaded - call gameStart() to begin")

    // Auto-start
    game.Start()

    <-make(chan struct{})
}
```

**game.html**:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Go WASM Canvas Game</title>
    <style>
        body {
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            background: #0a0a1a;
            color: white;
            font-family: sans-serif;
        }
        canvas {
            border: 2px solid #333;
            margin-top: 20px;
        }
        .controls {
            margin: 15px;
            display: flex;
            gap: 10px;
        }
        button {
            padding: 10px 20px;
            border: none;
            border-radius: 6px;
            background: #3b82f6;
            color: white;
            cursor: pointer;
            font-size: 16px;
        }
        button:hover { background: #2563eb; }
    </style>
</head>
<body>
    <h1>Go WASM Bouncing Balls</h1>
    <div class="controls">
        <button onclick="gameStart()">Start</button>
        <button onclick="gameStop()">Stop</button>
        <button onclick="gameAddBall()">Add Ball</button>
    </div>
    <canvas id="gameCanvas"></canvas>
    <script src="wasm_exec.js"></script>
    <script>
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("game.wasm"), go.importObject)
            .then(result => go.run(result.instance));
    </script>
</body>
</html>
```

### 12.4 Project: WASI-Based Data Pipeline Tool

A command-line data pipeline tool compiled to WASI that can run anywhere.

```go
package main

import (
    "bufio"
    "encoding/csv"
    "encoding/json"
    "fmt"
    "io"
    "os"
    "sort"
    "strconv"
    "strings"
)

// Record represents a data row
type Record map[string]string

// Pipeline defines a series of transformations
type Pipeline struct {
    steps []Step
}

// Step is a single pipeline step
type Step struct {
    Type   string `json:"type"`
    Field  string `json:"field,omitempty"`
    Value  string `json:"value,omitempty"`
    Rename string `json:"rename,omitempty"`
}

func (p *Pipeline) AddStep(step Step) {
    p.steps = append(p.steps, step)
}

func (p *Pipeline) Execute(records []Record) []Record {
    result := records
    for _, step := range p.steps {
        switch step.Type {
        case "filter":
            result = filterRecords(result, step.Field, step.Value)
        case "sort":
            result = sortRecords(result, step.Field)
        case "select":
            fields := strings.Split(step.Field, ",")
            result = selectFields(result, fields)
        case "rename":
            result = renameField(result, step.Field, step.Rename)
        case "uppercase":
            result = transformField(result, step.Field, strings.ToUpper)
        case "lowercase":
            result = transformField(result, step.Field, strings.ToLower)
        case "trim":
            result = transformField(result, step.Field, strings.TrimSpace)
        }
    }
    return result
}

func filterRecords(records []Record, field, value string) []Record {
    var result []Record
    for _, r := range records {
        if v, ok := r[field]; ok && strings.Contains(
            strings.ToLower(v), strings.ToLower(value)) {
            result = append(result, r)
        }
    }
    return result
}

func sortRecords(records []Record, field string) []Record {
    sorted := make([]Record, len(records))
    copy(sorted, records)
    sort.Slice(sorted, func(i, j int) bool {
        vi := sorted[i][field]
        vj := sorted[j][field]
        // Try numeric comparison first
        ni, errI := strconv.ParseFloat(vi, 64)
        nj, errJ := strconv.ParseFloat(vj, 64)
        if errI == nil && errJ == nil {
            return ni < nj
        }
        return vi < vj
    })
    return sorted
}

func selectFields(records []Record, fields []string) []Record {
    var result []Record
    for _, r := range records {
        newR := make(Record)
        for _, f := range fields {
            f = strings.TrimSpace(f)
            if v, ok := r[f]; ok {
                newR[f] = v
            }
        }
        result = append(result, newR)
    }
    return result
}

func renameField(records []Record, oldName, newName string) []Record {
    for _, r := range records {
        if v, ok := r[oldName]; ok {
            r[newName] = v
            delete(r, oldName)
        }
    }
    return records
}

func transformField(records []Record, field string, fn func(string) string) []Record {
    for _, r := range records {
        if v, ok := r[field]; ok {
            r[field] = fn(v)
        }
    }
    return records
}

// readCSV reads CSV data from a reader
func readCSV(r io.Reader) ([]Record, error) {
    reader := csv.NewReader(r)
    headers, err := reader.Read()
    if err != nil {
        return nil, fmt.Errorf("reading headers: %w", err)
    }

    var records []Record
    for {
        row, err := reader.Read()
        if err == io.EOF {
            break
        }
        if err != nil {
            return nil, fmt.Errorf("reading row: %w", err)
        }

        record := make(Record)
        for i, header := range headers {
            if i < len(row) {
                record[header] = row[i]
            }
        }
        records = append(records, record)
    }

    return records, nil
}

// writeJSON outputs records as JSON
func writeJSON(w io.Writer, records []Record) error {
    encoder := json.NewEncoder(w)
    encoder.SetIndent("", "  ")
    return encoder.Encode(records)
}

// writeCSV outputs records as CSV
func writeCSV(w io.Writer, records []Record) error {
    if len(records) == 0 {
        return nil
    }

    // Collect all headers
    headerSet := make(map[string]bool)
    for _, r := range records {
        for k := range r {
            headerSet[k] = true
        }
    }

    var headers []string
    for h := range headerSet {
        headers = append(headers, h)
    }
    sort.Strings(headers)

    writer := csv.NewWriter(w)
    defer writer.Flush()

    writer.Write(headers)

    for _, r := range records {
        row := make([]string, len(headers))
        for i, h := range headers {
            row[i] = r[h]
        }
        writer.Write(row)
    }

    return nil
}

func printUsage() {
    fmt.Fprintf(os.Stderr, `Usage: pipeline [options]

Reads CSV from stdin, applies transformations, writes to stdout.

Options:
  --filter FIELD=VALUE     Filter records where FIELD contains VALUE
  --sort FIELD             Sort by FIELD
  --select FIELD1,FIELD2   Select only specified fields
  --rename OLD=NEW         Rename a field
  --upper FIELD            Convert field to uppercase
  --lower FIELD            Convert field to lowercase
  --trim FIELD             Trim whitespace from field
  --format json|csv        Output format (default: json)
  --help                   Show this help

Example:
  cat data.csv | pipeline --filter country=US --sort age --select name,age --format json
`)
}

func main() {
    args := os.Args[1:]

    if len(args) == 0 || contains(args, "--help") {
        printUsage()
        os.Exit(0)
    }

    pipeline := &Pipeline{}
    outputFormat := "json"

    for i := 0; i < len(args); i++ {
        switch args[i] {
        case "--filter":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --filter requires FIELD=VALUE")
                os.Exit(1)
            }
            parts := strings.SplitN(args[i], "=", 2)
            if len(parts) != 2 {
                fmt.Fprintln(os.Stderr, "Error: --filter format is FIELD=VALUE")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "filter", Field: parts[0], Value: parts[1]})

        case "--sort":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --sort requires FIELD")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "sort", Field: args[i]})

        case "--select":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --select requires FIELD1,FIELD2,...")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "select", Field: args[i]})

        case "--rename":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --rename requires OLD=NEW")
                os.Exit(1)
            }
            parts := strings.SplitN(args[i], "=", 2)
            if len(parts) != 2 {
                fmt.Fprintln(os.Stderr, "Error: --rename format is OLD=NEW")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "rename", Field: parts[0], Rename: parts[1]})

        case "--upper":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --upper requires FIELD")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "uppercase", Field: args[i]})

        case "--lower":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --lower requires FIELD")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "lowercase", Field: args[i]})

        case "--trim":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --trim requires FIELD")
                os.Exit(1)
            }
            pipeline.AddStep(Step{Type: "trim", Field: args[i]})

        case "--format":
            i++
            if i >= len(args) {
                fmt.Fprintln(os.Stderr, "Error: --format requires json|csv")
                os.Exit(1)
            }
            outputFormat = args[i]
        }
    }

    // Read CSV from stdin
    reader := bufio.NewReader(os.Stdin)
    records, err := readCSV(reader)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error reading input: %v\n", err)
        os.Exit(1)
    }

    fmt.Fprintf(os.Stderr, "Read %d records\n", len(records))

    // Execute pipeline
    result := pipeline.Execute(records)

    fmt.Fprintf(os.Stderr, "Output: %d records\n", len(result))

    // Write output
    switch outputFormat {
    case "json":
        writeJSON(os.Stdout, result)
    case "csv":
        writeCSV(os.Stdout, result)
    default:
        fmt.Fprintf(os.Stderr, "Unknown format: %s\n", outputFormat)
        os.Exit(1)
    }
}

func contains(slice []string, item string) bool {
    for _, s := range slice {
        if s == item {
            return true
        }
    }
    return false
}
```

Build and test:
```bash
# Build for WASI
GOOS=wasip1 GOARCH=wasm go build -o pipeline.wasm pipeline.go

# Create test data
cat > /tmp/data.csv << 'EOF'
name,age,country,department
Alice,30,US,Engineering
Bob,25,UK,Marketing
Charlie,35,US,Engineering
Diana,28,DE,Design
Eve,32,US,Marketing
Frank,40,UK,Engineering
Grace,27,JP,Design
EOF

# Run with wasmtime
cat /tmp/data.csv | wasmtime pipeline.wasm -- \
  --filter country=US \
  --sort age \
  --select name,age,department \
  --format json

# Run with wazero
cat /tmp/data.csv | wazero run pipeline.wasm \
  --filter country=US \
  --sort age \
  --select name,age,department \
  --format json
```

Expected output:
```json
[
  {
    "age": "30",
    "department": "Engineering",
    "name": "Alice"
  },
  {
    "age": "32",
    "department": "Marketing",
    "name": "Eve"
  },
  {
    "age": "35",
    "department": "Engineering",
    "name": "Charlie"
  }
]
```

---

## 13. Summary

### Key Takeaways

1. **Go compiles natively to WebAssembly** via two targets:
   - `GOOS=js GOARCH=wasm` for browser/Node.js with JavaScript interop
   - `GOOS=wasip1 GOARCH=wasm` for standalone WASI runtimes

2. **The `syscall/js` package** bridges Go and JavaScript, enabling DOM manipulation,
   event handling, and bidirectional function calls.

3. **TinyGo** produces dramatically smaller WASM binaries (~90% reduction) at the cost
   of some standard library support and a simpler garbage collector.

4. **WASI** enables Go WASM modules to run outside the browser with file I/O,
   environment variables, and standard I/O.

5. **Wazero** is a pure Go WebAssembly runtime ideal for embedding WASM execution in Go
   applications, building plugin systems, and running untrusted code safely.

6. **Performance trade-offs** center on binary size (mitigated by TinyGo and compression),
   startup time (mitigated by caching and streaming compilation), and JS interop overhead
   (mitigated by batching operations).

7. **Real-world applications** include browser tools, edge computing, plugin architectures,
   and serverless functions.

8. **The future** is bright, with WASI Preview 2, the Component Model, threading support,
   and the WASM GC proposal all advancing Go's WASM capabilities.

### Quick Reference

```bash
# Build for browser
GOOS=js GOARCH=wasm go build -o app.wasm .

# Build for WASI
GOOS=wasip1 GOARCH=wasm go build -o app.wasm .

# Copy JS glue file
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" .

# Build with TinyGo (browser)
tinygo build -o app.wasm -target wasm .

# Build with TinyGo (WASI)
tinygo build -o app.wasm -target wasi .

# Run with wasmtime
wasmtime --dir=/tmp app.wasm -- arg1 arg2

# Run with wazero
wazero run --mount /tmp:/tmp app.wasm arg1 arg2

# Optimize with wasm-opt (Binaryen)
wasm-opt -Oz -o app-opt.wasm app.wasm

# Strip debug info (standard Go)
GOOS=js GOARCH=wasm go build -ldflags="-s -w" -o app.wasm .

# Check binary size
ls -lh app.wasm
```

### Further Reading

- Go Wiki WebAssembly: https://go.dev/wiki/WebAssembly
- TinyGo WASM Guide: https://tinygo.org/docs/guides/webassembly/
- Wazero Documentation: https://wazero.io/
- WASI Specification: https://wasi.dev/
- WebAssembly Specification: https://webassembly.org/specs/
- Bytecode Alliance: https://bytecodealliance.org/

---

*Next chapter: [Chapter 59 - Go and eBPF]*
