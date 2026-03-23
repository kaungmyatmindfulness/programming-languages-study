# Chapter 13: File I/O, OS Operations & CLI Tools (INTERMEDIATE)

---

## Table of Contents

1. [Reading Files](#1-reading-files)
2. [Writing Files](#2-writing-files)
3. [File Operations](#3-file-operations)
4. [The io.Reader/io.Writer Interfaces](#4-the-ioreaderiowriter-interfaces)
5. [Working with Directories](#5-working-with-directories)
6. [Environment Variables](#6-environment-variables)
7. [Command Line Arguments](#7-command-line-arguments)
8. [Building CLI Tools](#8-building-cli-tools)
9. [os/exec - Running External Commands](#9-osexec---running-external-commands)
10. [Cross-Compilation](#10-cross-compilation)
11. [Go vs Node.js Comparison Summary](#11-go-vs-nodejs-comparison-summary)
12. [Key Takeaways](#12-key-takeaways)
13. [Practice Exercises](#13-practice-exercises)

---

## 1. Reading Files

Go provides multiple ways to read files, and each exists for a reason. Understanding **when** to use which approach is far more important than memorizing the APIs.

### The Big Picture: Why So Many Ways?

| Method | Best For | Memory Usage | Complexity |
|--------|----------|-------------|------------|
| `os.ReadFile` | Small files (< a few MB) | Loads entire file into memory | Simplest |
| `os.Open` + `io.ReadAll` | When you need the `*os.File` handle | Loads entire file into memory | Simple |
| `bufio.Scanner` | Line-by-line processing | One line at a time | Moderate |
| `bufio.Reader` | Chunk/token-based processing | Buffered chunks | Moderate |
| `os.Open` + manual `Read` | Maximum control, binary files | You control the buffer | Most complex |

### 1.1 os.ReadFile -- The Simple Case

When you have a small file and just want its contents, `os.ReadFile` is your go-to. It reads the entire file into memory in one shot.

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.ReadFile returns []byte and error
    // It opens, reads, and closes the file for you
    data, err := os.ReadFile("config.json")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error reading file: %v\n", err)
        os.Exit(1)
    }

    fmt.Println(string(data))
}
```

**Node.js equivalent:**

```javascript
// Synchronous -- analogous to Go's os.ReadFile
const data = fs.readFileSync('config.json', 'utf8');

// Async with promises -- no direct Go equivalent (Go is synchronous by default,
// you'd use goroutines for concurrency)
const data = await fs.promises.readFile('config.json', 'utf8');
```

**When NOT to use `os.ReadFile`:** If the file could be hundreds of megabytes or larger, you will consume that much memory. A 2GB log file read with `os.ReadFile` means 2GB of RAM used just for that variable.

### 1.2 os.Open + io.ReadAll -- When You Need the File Handle

Sometimes you need the `*os.File` object (to check stats, pass to another function, etc.) but still want all the content.

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func main() {
    // os.Open opens for reading only (O_RDONLY)
    file, err := os.Open("data.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close() // ALWAYS defer Close right after error check

    // io.ReadAll reads from any io.Reader until EOF
    data, err := io.ReadAll(file)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error reading: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Read %d bytes\n", len(data))
    fmt.Println(string(data))
}
```

**Critical pattern: `defer file.Close()`**

Always close files you open. The `defer` keyword ensures the file is closed when the function returns, even if a panic occurs. This is one of Go's most elegant patterns -- it keeps the cleanup code right next to the allocation code, unlike try/finally blocks that separate them.

```go
// WRONG -- if something panics between Open and Close, the file leaks
file, err := os.Open("data.txt")
// ... lots of code ...
file.Close()

// RIGHT -- Close is guaranteed to run when the function exits
file, err := os.Open("data.txt")
if err != nil {
    return err
}
defer file.Close()
```

### 1.3 bufio.Scanner -- Line-by-Line Reading

This is the workhorse for processing text files line by line. It uses a fixed-size buffer (default 64KB per line) so memory usage stays constant regardless of file size.

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    file, err := os.Open("server.log")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)

    lineNum := 0
    errorCount := 0

    for scanner.Scan() { // Scan() advances to next token (line by default)
        lineNum++
        line := scanner.Text() // Text() returns the current token as string

        if strings.Contains(line, "ERROR") {
            errorCount++
            fmt.Printf("Line %d: %s\n", lineNum, line)
        }
    }

    // IMPORTANT: always check for scanner errors after the loop
    if err := scanner.Err(); err != nil {
        fmt.Fprintf(os.Stderr, "error scanning: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("\nFound %d errors in %d lines\n", errorCount, lineNum)
}
```

**Node.js equivalent using readline:**

```javascript
const rl = readline.createInterface({
    input: fs.createReadStream('server.log'),
    crlfDelay: Infinity
});

let lineNum = 0;
let errorCount = 0;

for await (const line of rl) {
    lineNum++;
    if (line.includes('ERROR')) {
        errorCount++;
        console.log(`Line ${lineNum}: ${line}`);
    }
}
```

**Handling long lines:** The default scanner buffer is 64KB per line. If your file has longer lines (e.g., minified JSON), increase it:

```go
scanner := bufio.NewScanner(file)
// Increase buffer to 1MB max per line
scanner.Buffer(make([]byte, 0, 1024*1024), 1024*1024)
```

### 1.4 bufio.Scanner with Custom Split Functions

The scanner's default split function is `bufio.ScanLines`, but you can change it to scan words, bytes, or anything custom.

```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    file, err := os.Open("article.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    scanner.Split(bufio.ScanWords) // Split on whitespace instead of newlines

    wordCount := 0
    wordFreq := make(map[string]int)

    for scanner.Scan() {
        word := scanner.Text()
        wordCount++
        wordFreq[word]++
    }

    if err := scanner.Err(); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Total words: %d\n", wordCount)
    fmt.Printf("Unique words: %d\n", len(wordFreq))
}
```

Built-in split functions:
- `bufio.ScanLines` -- split on `\n` (default)
- `bufio.ScanWords` -- split on whitespace
- `bufio.ScanBytes` -- one byte at a time
- `bufio.ScanRunes` -- one UTF-8 rune at a time

### 1.5 Reading in Chunks -- Maximum Control

For binary files or when you need precise control over buffering:

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func main() {
    file, err := os.Open("large-binary.dat")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    buf := make([]byte, 4096) // Read 4KB at a time
    totalBytes := 0

    for {
        n, err := file.Read(buf)
        if n > 0 {
            totalBytes += n
            // Process buf[:n] -- note: use buf[:n], NOT buf
            // because n might be less than len(buf) on the last read
        }
        if err == io.EOF {
            break // End of file -- this is normal, not an error
        }
        if err != nil {
            fmt.Fprintf(os.Stderr, "read error: %v\n", err)
            os.Exit(1)
        }
    }

    fmt.Printf("Read %d bytes total\n", totalBytes)
}
```

**Important subtlety:** `Read` can return both data AND an error (including `io.EOF`) in the same call. Always process `n > 0` bytes before checking the error. This catches the case where the last read returns some bytes along with `io.EOF`.

---

## 2. Writing Files

### 2.1 os.WriteFile -- The Simple Case

Just like `os.ReadFile`, this is the all-in-one approach for small writes.

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type Config struct {
    Host    string `json:"host"`
    Port    int    `json:"port"`
    Debug   bool   `json:"debug"`
}

func main() {
    config := Config{
        Host:  "localhost",
        Port:  8080,
        Debug: true,
    }

    data, err := json.MarshalIndent(config, "", "  ")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error marshaling: %v\n", err)
        os.Exit(1)
    }

    // 0644 = owner read/write, group read, others read
    err = os.WriteFile("config.json", data, 0644)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error writing: %v\n", err)
        os.Exit(1)
    }

    fmt.Println("Config written successfully")
}
```

**Node.js equivalent:**

```javascript
const config = { host: 'localhost', port: 8080, debug: true };
fs.writeFileSync('config.json', JSON.stringify(config, null, 2));
```

**File permissions (the `0644` argument):** This is a Unix file mode. Go requires you to specify it explicitly, unlike Node.js which defaults to `0o666`. Common values:

| Mode | Meaning |
|------|---------|
| `0644` | Owner: read+write. Group/Others: read only. Standard for files. |
| `0755` | Owner: read+write+execute. Group/Others: read+execute. Standard for executables/directories. |
| `0600` | Owner: read+write. No access for anyone else. Use for secrets/keys. |

### 2.2 os.Create + Write -- More Control

`os.Create` creates or truncates a file for writing.

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.Create truncates existing file or creates new one
    // It opens with O_RDWR|O_CREATE|O_TRUNC and permission 0666
    file, err := os.Create("output.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    // Write a string
    _, err = file.WriteString("Hello, World!\n")
    if err != nil {
        fmt.Fprintf(os.Stderr, "write error: %v\n", err)
        os.Exit(1)
    }

    // Write bytes
    _, err = file.Write([]byte("Second line\n"))
    if err != nil {
        fmt.Fprintf(os.Stderr, "write error: %v\n", err)
        os.Exit(1)
    }

    // Formatted writing using fmt.Fprintf
    for i := 1; i <= 5; i++ {
        _, err = fmt.Fprintf(file, "Line %d: some data\n", i)
        if err != nil {
            fmt.Fprintf(os.Stderr, "write error: %v\n", err)
            os.Exit(1)
        }
    }
}
```

### 2.3 os.OpenFile -- Full Control Over Open Flags

When you need to append, or control exactly how the file is opened:

```go
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    // Open for appending; create if doesn't exist
    file, err := os.OpenFile("app.log",
        os.O_APPEND|os.O_CREATE|os.O_WRONLY, // flags
        0644, // permissions
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    timestamp := time.Now().Format("2006-01-02 15:04:05")
    _, err = fmt.Fprintf(file, "[%s] Application started\n", timestamp)
    if err != nil {
        fmt.Fprintf(os.Stderr, "write error: %v\n", err)
        os.Exit(1)
    }
}
```

**Open flags explained:**

| Flag | Purpose |
|------|---------|
| `os.O_RDONLY` | Read only (used by `os.Open`) |
| `os.O_WRONLY` | Write only |
| `os.O_RDWR` | Read and write |
| `os.O_APPEND` | Append to end of file |
| `os.O_CREATE` | Create file if it doesn't exist |
| `os.O_TRUNC` | Truncate (empty) file when opening |
| `os.O_EXCL` | Used with O_CREATE: fail if file already exists |

### 2.4 bufio.Writer -- Buffered Writing

When writing many small pieces of data, each `Write` call results in a system call. `bufio.Writer` batches them.

```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    file, err := os.Create("large-output.csv")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    // bufio.Writer batches writes into a buffer (default 4096 bytes)
    // then flushes to the underlying writer
    writer := bufio.NewWriter(file)

    // Write CSV header
    writer.WriteString("id,name,email\n")

    // Write 100,000 rows -- each WriteString doesn't hit the disk individually
    for i := 0; i < 100_000; i++ {
        fmt.Fprintf(writer, "%d,user_%d,user_%d@example.com\n", i, i, i)
    }

    // CRITICAL: Flush remaining data in the buffer to the file
    // Without this, the last partial buffer is lost!
    if err := writer.Flush(); err != nil {
        fmt.Fprintf(os.Stderr, "flush error: %v\n", err)
        os.Exit(1)
    }
}
```

**Why buffered writing matters:**

Without `bufio.Writer`, 100,000 `Fprintf` calls = 100,000 system calls. With it, the data accumulates in a 4KB memory buffer and flushes to disk only when the buffer is full. That is roughly 100,000 vs ~1,500 system calls. The performance difference is enormous for many small writes.

**Node.js comparison:**

```javascript
// Node.js Writable streams are buffered by default
const stream = fs.createWriteStream('large-output.csv');
stream.write('id,name,email\n');
for (let i = 0; i < 100000; i++) {
    stream.write(`${i},user_${i},user_${i}@example.com\n`);
}
stream.end(); // equivalent to Flush + Close
```

### 2.5 Writing Files Safely with Atomic Writes

A common bug: your program crashes mid-write and you end up with a corrupted file. The solution is to write to a temporary file, then rename it.

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

// SafeWriteFile writes data to a file atomically
func SafeWriteFile(filename string, data []byte, perm os.FileMode) error {
    // Write to a temporary file in the same directory
    // (same directory ensures same filesystem for rename)
    tmpFile := filename + ".tmp"

    if err := os.WriteFile(tmpFile, data, perm); err != nil {
        return fmt.Errorf("writing temp file: %w", err)
    }

    // os.Rename is atomic on most filesystems
    if err := os.Rename(tmpFile, filename); err != nil {
        os.Remove(tmpFile) // clean up on failure
        return fmt.Errorf("renaming temp file: %w", err)
    }

    return nil
}

func main() {
    data, _ := json.MarshalIndent(map[string]string{"key": "value"}, "", "  ")

    if err := SafeWriteFile("important.json", data, 0644); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}
```

---

## 3. File Operations

### 3.1 Checking If a File/Directory Exists

```go
package main

import (
    "errors"
    "fmt"
    "os"
)

func fileExists(path string) bool {
    _, err := os.Stat(path)
    return !errors.Is(err, os.ErrNotExist)
}

func main() {
    // os.Stat returns FileInfo and error
    info, err := os.Stat("config.json")
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("File does not exist")
        return
    }
    if err != nil {
        fmt.Fprintf(os.Stderr, "unexpected error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Name:    %s\n", info.Name())
    fmt.Printf("Size:    %d bytes\n", info.Size())
    fmt.Printf("Mode:    %s\n", info.Mode())
    fmt.Printf("ModTime: %s\n", info.ModTime())
    fmt.Printf("IsDir:   %t\n", info.IsDir())
}
```

**Node.js equivalent:**

```javascript
try {
    const stats = fs.statSync('config.json');
    console.log(`Size: ${stats.size}`);
    console.log(`IsDirectory: ${stats.isDirectory()}`);
} catch (err) {
    if (err.code === 'ENOENT') {
        console.log('File does not exist');
    }
}
```

### 3.2 Common File Operations

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
)

func main() {
    // --- Create directories ---
    // MkdirAll creates the full path (like mkdir -p)
    err := os.MkdirAll("output/reports/2024", 0755)
    if err != nil {
        fmt.Fprintf(os.Stderr, "mkdir error: %v\n", err)
        os.Exit(1)
    }

    // --- Rename / Move ---
    err = os.Rename("old-name.txt", "new-name.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "rename error: %v\n", err)
    }

    // --- Remove a single file ---
    err = os.Remove("temp.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "remove error: %v\n", err)
    }

    // --- Remove a directory tree (like rm -rf) ---
    // BE VERY CAREFUL with RemoveAll -- no confirmation, no trash bin
    err = os.RemoveAll("output/reports")
    if err != nil {
        fmt.Fprintf(os.Stderr, "removeAll error: %v\n", err)
    }

    // --- Copy a file (Go has no built-in copy -- you build it from io) ---
    // See Section 4 for how to do this with io.Copy

    // --- Get current working directory ---
    cwd, err := os.Getwd()
    if err != nil {
        fmt.Fprintf(os.Stderr, "getwd error: %v\n", err)
        os.Exit(1)
    }
    fmt.Println("Current directory:", cwd)

    // --- Change directory ---
    err = os.Chdir("/tmp")
    if err != nil {
        fmt.Fprintf(os.Stderr, "chdir error: %v\n", err)
    }
}

// --- filepath package: path manipulation ---
func filepathExamples() {
    // filepath works correctly across operating systems
    // (unlike path, which is for URL-style slash-separated paths)

    // Join parts into a path (uses OS-appropriate separator)
    p := filepath.Join("home", "user", "documents", "file.txt")
    fmt.Println(p) // home/user/documents/file.txt (on Unix)

    // Get the directory portion
    fmt.Println(filepath.Dir(p)) // home/user/documents

    // Get the filename
    fmt.Println(filepath.Base(p)) // file.txt

    // Get the extension
    fmt.Println(filepath.Ext(p)) // .txt

    // Get absolute path
    abs, _ := filepath.Abs("relative/path.txt")
    fmt.Println(abs) // /current/working/dir/relative/path.txt

    // Clean a path (resolve .., ., double slashes)
    fmt.Println(filepath.Clean("/a/b/../c/./d")) // /a/c/d

    // Match a glob pattern
    matched, _ := filepath.Match("*.go", "main.go")
    fmt.Println(matched) // true

    // Split into dir and file
    dir, file := filepath.Split("/home/user/config.json")
    fmt.Println(dir, file) // /home/user/ config.json
}
```

### 3.3 Temporary Files and Directories

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Create a temp file -- Go picks a unique name
    // First arg is directory ("" means OS default temp dir)
    // Second arg is a name pattern -- * is replaced with a random string
    tmpFile, err := os.CreateTemp("", "myapp-*.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer os.Remove(tmpFile.Name()) // Clean up when done
    defer tmpFile.Close()

    fmt.Println("Temp file:", tmpFile.Name())
    // e.g., /tmp/myapp-1234567890.txt

    tmpFile.WriteString("temporary data\n")

    // Create a temp directory
    tmpDir, err := os.MkdirTemp("", "myapp-*")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer os.RemoveAll(tmpDir) // Clean up the whole directory tree

    fmt.Println("Temp dir:", tmpDir)
}
```

---

## 4. The io.Reader/io.Writer Interfaces

This is arguably **the most important section in this chapter**. The `io.Reader` and `io.Writer` interfaces are the foundation of Go's I/O system, and understanding them unlocks the full power of Go's standard library.

### 4.1 The Interfaces

```go
// That's it. Two methods. Two interfaces. They run the world.
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

These are absurdly simple -- and that is exactly what makes them powerful.

### 4.2 WHY These Are the Most Important Interfaces in Go

Think about everything that can be read from or written to:

- Files on disk
- Network connections (TCP, HTTP request/response bodies)
- In-memory buffers (`bytes.Buffer`, `strings.Reader`)
- Compression streams (gzip, zlib)
- Encryption streams
- Hash computations (SHA256, MD5)
- Standard input/output (`os.Stdin`, `os.Stdout`)
- Base64 encoders/decoders

**Every single one of these implements `io.Reader`, `io.Writer`, or both.** This means any function that accepts an `io.Reader` can work with ALL of them. You write your function once, and it works with files, network data, in-memory strings, compressed streams -- anything.

This is the **Unix philosophy in Go**: small, composable pieces connected by a universal interface (in Unix it is byte streams via pipes; in Go it is `io.Reader`/`io.Writer`).

### 4.3 io.Reader in Practice

```go
package main

import (
    "bytes"
    "compress/gzip"
    "fmt"
    "io"
    "os"
    "strings"
)

// countLines works with ANY io.Reader -- file, network, string, anything
func countLines(r io.Reader) (int, error) {
    buf := make([]byte, 32*1024) // 32KB buffer
    count := 0

    for {
        n, err := r.Read(buf)
        for i := 0; i < n; i++ {
            if buf[i] == '\n' {
                count++
            }
        }
        if err == io.EOF {
            return count, nil
        }
        if err != nil {
            return count, err
        }
    }
}

func main() {
    // Use the SAME function with completely different sources:

    // 1. From a string
    lines, _ := countLines(strings.NewReader("hello\nworld\nfoo\n"))
    fmt.Printf("String: %d lines\n", lines) // 3

    // 2. From a byte slice
    data := []byte("line1\nline2\n")
    lines, _ = countLines(bytes.NewReader(data))
    fmt.Printf("Bytes: %d lines\n", lines) // 2

    // 3. From a file
    file, err := os.Open("main.go")
    if err == nil {
        defer file.Close()
        lines, _ = countLines(file)
        fmt.Printf("File: %d lines\n", lines)
    }

    // 4. From a gzipped file -- composing readers!
    gzFile, err := os.Open("data.gz")
    if err == nil {
        defer gzFile.Close()
        gzReader, err := gzip.NewReader(gzFile)
        if err == nil {
            defer gzReader.Close()
            lines, _ = countLines(gzReader)
            fmt.Printf("Gzipped file: %d lines\n", lines)
        }
    }
}
```

**Node.js comparison:**

```javascript
// Node.js uses different APIs for different sources:
// - fs.readFileSync for files
// - response.body for HTTP
// - Buffer for in-memory
// - stream.Readable for streaming

// Node.js streams are more complex with modes (flowing/paused),
// event-based API, and backpressure handling

// Go's io.Reader is synchronous and dead simple:
// call Read(), get bytes, check error. Done.
```

### 4.4 io.Copy -- Connecting Readers to Writers

`io.Copy` is the duct tape of Go I/O. It reads from a Reader and writes to a Writer until EOF.

```go
package main

import (
    "fmt"
    "io"
    "os"
)

// CopyFile copies a file from src to dst
func CopyFile(dst, src string) error {
    sourceFile, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("opening source: %w", err)
    }
    defer sourceFile.Close()

    destFile, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("creating destination: %w", err)
    }
    defer destFile.Close()

    // io.Copy reads from sourceFile and writes to destFile
    // It uses a 32KB internal buffer -- you never load the whole file into memory
    bytesCopied, err := io.Copy(destFile, sourceFile)
    if err != nil {
        return fmt.Errorf("copying: %w", err)
    }

    fmt.Printf("Copied %d bytes\n", bytesCopied)
    return nil
}

func main() {
    if err := CopyFile("backup.txt", "original.txt"); err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
}
```

**Why Go has no built-in file copy function:** Because `io.Copy` + `os.Open` + `os.Create` is already trivial. Go prefers composable primitives over specialized functions.

### 4.5 io.TeeReader -- Read and Tap

`io.TeeReader` is like a T-pipe: everything read from the reader is also written to a writer. Perfect for reading data while simultaneously hashing it, logging it, or saving a copy.

```go
package main

import (
    "crypto/sha256"
    "fmt"
    "io"
    "os"
)

func main() {
    file, err := os.Open("document.pdf")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    // Create a SHA256 hasher (implements io.Writer)
    hasher := sha256.New()

    // TeeReader: every byte read from tee also gets written to hasher
    tee := io.TeeReader(file, hasher)

    // Read the file content (the hash is computed simultaneously)
    content, err := io.ReadAll(tee)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error reading: %v\n", err)
        os.Exit(1)
    }

    // Both are done in a single pass through the file!
    fmt.Printf("Size: %d bytes\n", len(content))
    fmt.Printf("SHA256: %x\n", hasher.Sum(nil))
}
```

### 4.6 io.MultiWriter -- Write to Multiple Destinations

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func main() {
    logFile, err := os.Create("app.log")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer logFile.Close()

    // Write to both stdout AND the log file simultaneously
    multi := io.MultiWriter(os.Stdout, logFile)

    fmt.Fprintln(multi, "Application started")
    fmt.Fprintln(multi, "Processing data...")
    fmt.Fprintln(multi, "Done!")
    // Each line appears on screen AND in app.log
}
```

### 4.7 io.MultiReader -- Concatenate Readers

```go
package main

import (
    "io"
    "os"
    "strings"
)

func main() {
    header := strings.NewReader("=== REPORT HEADER ===\n\n")

    body, err := os.Open("report-body.txt")
    if err != nil {
        return
    }
    defer body.Close()

    footer := strings.NewReader("\n=== END OF REPORT ===\n")

    // Combine three readers into one continuous stream
    combined := io.MultiReader(header, body, footer)

    // Write the combined stream to a new file
    output, err := os.Create("full-report.txt")
    if err != nil {
        return
    }
    defer output.Close()

    io.Copy(output, combined)
}
```

### 4.8 io.LimitReader and io.SectionReader

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func main() {
    file, err := os.Open("huge-file.bin")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    defer file.Close()

    // LimitReader: only read the first 1024 bytes
    // Great for previewing files, protecting against large uploads, etc.
    limited := io.LimitReader(file, 1024)
    preview, _ := io.ReadAll(limited)
    fmt.Printf("First 1KB: %s\n", preview)

    // SectionReader: read a specific range from a file
    // (requires io.ReaderAt, which *os.File implements)
    section := io.NewSectionReader(file, 100, 50) // offset=100, length=50
    chunk, _ := io.ReadAll(section)
    fmt.Printf("Bytes 100-149: %x\n", chunk)
}
```

### 4.9 The Composability Diagram

Here is how Go's reader/writer ecosystem connects:

```
                    +----------------------------------------------+
                    |          Things that are io.Reader            |
                    |                                               |
                    |  os.File          strings.Reader              |
                    |  http.Response.Body  bytes.Buffer             |
                    |  gzip.Reader      cipher.StreamReader         |
                    |  net.Conn         bufio.Reader                |
                    |  base64.Decoder   os.Stdin                    |
                    +----------------------+------------------------+
                                           |
                                           v
                    +----------------------------------------------+
                    |        Functions that accept io.Reader        |
                    |                                               |
                    |  io.Copy          io.ReadAll                  |
                    |  io.TeeReader     bufio.NewScanner            |
                    |  json.NewDecoder  csv.NewReader               |
                    |  gzip.NewReader   io.LimitReader              |
                    |  fmt.Fscan        io.MultiReader              |
                    +----------------------------------------------+

                    +----------------------------------------------+
                    |          Things that are io.Writer            |
                    |                                               |
                    |  os.File          bytes.Buffer                |
                    |  http.ResponseWriter  net.Conn                |
                    |  gzip.Writer      cipher.StreamWriter         |
                    |  bufio.Writer     hash.Hash                   |
                    |  base64.Encoder   os.Stdout / os.Stderr       |
                    +----------------------+------------------------+
                                           |
                                           v
                    +----------------------------------------------+
                    |        Functions that accept io.Writer        |
                    |                                               |
                    |  io.Copy          fmt.Fprintf                 |
                    |  io.MultiWriter   json.NewEncoder             |
                    |  csv.NewWriter    gzip.NewWriter              |
                    |  io.TeeReader     bufio.NewWriter             |
                    |  log.New          template.Execute            |
                    +----------------------------------------------+
```

**This composability is Go's equivalent of Unix pipes.** Just as you can chain `cat file | grep pattern | sort | uniq` in a shell, you can chain Go readers and writers in infinite combinations.

---

## 5. Working with Directories

### 5.1 os.ReadDir -- List Directory Contents

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    entries, err := os.ReadDir(".")
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }

    for _, entry := range entries {
        if entry.IsDir() {
            fmt.Printf("[DIR]  %s\n", entry.Name())
        } else {
            info, err := entry.Info()
            if err != nil {
                continue
            }
            fmt.Printf("[FILE] %s (%d bytes)\n", entry.Name(), info.Size())
        }
    }
}
```

**Node.js equivalent:**

```javascript
const entries = fs.readdirSync('.', { withFileTypes: true });
for (const entry of entries) {
    if (entry.isDirectory()) {
        console.log(`[DIR]  ${entry.name}`);
    } else {
        const stats = fs.statSync(entry.name);
        console.log(`[FILE] ${entry.name} (${stats.size} bytes)`);
    }
}
```

### 5.2 filepath.WalkDir -- Recursively Walk a Directory Tree

`filepath.WalkDir` is the standard way to process an entire directory tree. It was introduced in Go 1.16 as a more efficient replacement for `filepath.Walk`.

```go
package main

import (
    "fmt"
    "io/fs"
    "os"
    "path/filepath"
    "strings"
)

func main() {
    root := "."
    var goFiles []string
    var totalSize int64

    err := filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error {
        // Handle errors accessing the path
        if err != nil {
            fmt.Fprintf(os.Stderr, "warning: %v\n", err)
            return nil // Return nil to continue walking; return err to stop
        }

        // Skip hidden directories (like .git)
        if d.IsDir() && strings.HasPrefix(d.Name(), ".") && d.Name() != "." {
            return filepath.SkipDir // Special return value: skip this directory
        }

        // Process Go files
        if !d.IsDir() && strings.HasSuffix(d.Name(), ".go") {
            goFiles = append(goFiles, path)

            info, err := d.Info()
            if err == nil {
                totalSize += info.Size()
            }
        }

        return nil
    })

    if err != nil {
        fmt.Fprintf(os.Stderr, "walk error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Found %d Go files (%d bytes total)\n", len(goFiles), totalSize)
    for _, f := range goFiles {
        fmt.Println(" ", f)
    }
}
```

**filepath.WalkDir vs filepath.Walk:**

- `WalkDir` (Go 1.16+): Receives `fs.DirEntry`, which is lazy -- it does NOT call `Stat` on every file. Much faster for large trees where you only need names and types.
- `Walk` (older): Receives `fs.FileInfo`, which calls `Stat` on every single file. Slower, especially on network filesystems.

Always prefer `WalkDir` unless you need the `FileInfo` for every entry.

**Node.js equivalent:**

```javascript
// Node.js 18.17+ has recursive readdir
const files = fs.readdirSync('.', { recursive: true, withFileTypes: true });
const goFiles = files
    .filter(f => f.isFile() && f.name.endsWith('.go'))
    .map(f => path.join(f.parentPath, f.name));
```

### 5.3 Practical Example: Find Large Files

```go
package main

import (
    "fmt"
    "io/fs"
    "os"
    "path/filepath"
    "sort"
)

type FileEntry struct {
    Path string
    Size int64
}

func findLargeFiles(root string, minSize int64) ([]FileEntry, error) {
    var files []FileEntry

    err := filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return nil // skip inaccessible files
        }
        if d.IsDir() {
            return nil
        }

        info, err := d.Info()
        if err != nil {
            return nil
        }

        if info.Size() >= minSize {
            files = append(files, FileEntry{Path: path, Size: info.Size()})
        }

        return nil
    })

    // Sort by size descending
    sort.Slice(files, func(i, j int) bool {
        return files[i].Size > files[j].Size
    })

    return files, err
}

func formatSize(bytes int64) string {
    const (
        KB = 1024
        MB = KB * 1024
        GB = MB * 1024
    )
    switch {
    case bytes >= GB:
        return fmt.Sprintf("%.2f GB", float64(bytes)/float64(GB))
    case bytes >= MB:
        return fmt.Sprintf("%.2f MB", float64(bytes)/float64(MB))
    case bytes >= KB:
        return fmt.Sprintf("%.2f KB", float64(bytes)/float64(KB))
    default:
        return fmt.Sprintf("%d B", bytes)
    }
}

func main() {
    root := "."
    if len(os.Args) > 1 {
        root = os.Args[1]
    }

    // Find files larger than 1MB
    files, err := findLargeFiles(root, 1024*1024)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Files larger than 1MB in %s:\n\n", root)
    for _, f := range files {
        fmt.Printf("  %10s  %s\n", formatSize(f.Size), f.Path)
    }
    fmt.Printf("\nTotal: %d files\n", len(files))
}
```

---

## 6. Environment Variables

### 6.1 Reading and Writing Environment Variables

```go
package main

import (
    "fmt"
    "os"
    "strings"
)

func main() {
    // Get a single environment variable
    // Returns empty string if not set (no error)
    home := os.Getenv("HOME")
    fmt.Println("HOME:", home)

    // LookupEnv distinguishes between "not set" and "set to empty string"
    value, exists := os.LookupEnv("DATABASE_URL")
    if !exists {
        fmt.Println("DATABASE_URL is not set")
    } else if value == "" {
        fmt.Println("DATABASE_URL is set but empty")
    } else {
        fmt.Println("DATABASE_URL:", value)
    }

    // Set an environment variable (affects this process and children)
    os.Setenv("MY_APP_MODE", "production")
    fmt.Println("MY_APP_MODE:", os.Getenv("MY_APP_MODE"))

    // Unset an environment variable
    os.Unsetenv("MY_APP_MODE")

    // List ALL environment variables
    for _, env := range os.Environ() {
        // Each entry is "KEY=VALUE"
        parts := strings.SplitN(env, "=", 2)
        key := parts[0]
        val := parts[1]

        // Print only PATH-like variables for demo
        if strings.Contains(key, "PATH") {
            fmt.Printf("%s = %s\n", key, val)
        }
    }
}
```

### 6.2 Pattern: Configuration from Environment with Defaults

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

type Config struct {
    Host     string
    Port     int
    Debug    bool
    LogLevel string
}

// getEnvOrDefault returns the env var value or a default
func getEnvOrDefault(key, defaultValue string) string {
    if value, exists := os.LookupEnv(key); exists {
        return value
    }
    return defaultValue
}

func getEnvAsInt(key string, defaultValue int) int {
    valueStr := os.Getenv(key)
    if valueStr == "" {
        return defaultValue
    }
    value, err := strconv.Atoi(valueStr)
    if err != nil {
        return defaultValue
    }
    return value
}

func getEnvAsBool(key string, defaultValue bool) bool {
    valueStr := os.Getenv(key)
    if valueStr == "" {
        return defaultValue
    }
    value, err := strconv.ParseBool(valueStr)
    if err != nil {
        return defaultValue
    }
    return value
}

func LoadConfig() Config {
    return Config{
        Host:     getEnvOrDefault("APP_HOST", "localhost"),
        Port:     getEnvAsInt("APP_PORT", 8080),
        Debug:    getEnvAsBool("APP_DEBUG", false),
        LogLevel: getEnvOrDefault("APP_LOG_LEVEL", "info"),
    }
}

func main() {
    config := LoadConfig()
    fmt.Printf("Config: %+v\n", config)
}
```

**Node.js equivalent:**

```javascript
// Node.js approach is essentially the same
const config = {
    host: process.env.APP_HOST || 'localhost',
    port: parseInt(process.env.APP_PORT) || 8080,
    debug: process.env.APP_DEBUG === 'true',
    logLevel: process.env.APP_LOG_LEVEL || 'info',
};
```

The key difference: Go's `os.Getenv` never returns `undefined` (it returns empty string), so there is no null/undefined confusion like in Node.js. `os.LookupEnv` is the explicit way to check existence.

---

## 7. Command Line Arguments

### 7.1 os.Args -- Raw Arguments

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // os.Args is a []string -- the raw command-line arguments
    // os.Args[0] is always the program name
    fmt.Println("Program:", os.Args[0])
    fmt.Println("All args:", os.Args)

    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "Usage: program <name>")
        os.Exit(1)
    }

    name := os.Args[1]
    fmt.Printf("Hello, %s!\n", name)
}
```

```bash
$ go run main.go Alice
Program: /tmp/go-build.../main
All args: [/tmp/go-build.../main Alice]
Hello, Alice!
```

**Node.js equivalent:**

```javascript
// process.argv[0] is the node binary path
// process.argv[1] is the script path
// process.argv[2+] are the actual arguments
const name = process.argv[2]; // Note: index 2, not 1!
```

### 7.2 The flag Package -- Standard Library Argument Parsing

For anything beyond the simplest arguments, use the `flag` package.

```go
package main

import (
    "flag"
    "fmt"
    "os"
    "strings"
)

func main() {
    // Define flags -- each returns a pointer
    host := flag.String("host", "localhost", "server hostname")
    port := flag.Int("port", 8080, "server port")
    verbose := flag.Bool("verbose", false, "enable verbose output")

    // You can also bind to existing variables
    var configFile string
    flag.StringVar(&configFile, "config", "", "path to config file")

    // Custom usage message
    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage: %s [options]\n\nOptions:\n", os.Args[0])
        flag.PrintDefaults()
    }

    // Parse the command line -- MUST be called after all flags are defined
    flag.Parse()

    // Remaining non-flag arguments
    args := flag.Args()

    fmt.Printf("Host:    %s\n", *host)
    fmt.Printf("Port:    %d\n", *port)
    fmt.Printf("Verbose: %t\n", *verbose)
    fmt.Printf("Config:  %s\n", configFile)
    fmt.Printf("Args:    %s\n", strings.Join(args, ", "))
}
```

```bash
$ go run main.go -host=example.com -port=3000 -verbose -config=/etc/app.conf extra1 extra2
Host:    example.com
Port:    3000
Verbose: true
Config:  /etc/app.conf
Args:    extra1, extra2

$ go run main.go -help
Usage: main [options]

Options:
  -config string
        path to config file
  -host string
        server hostname (default "localhost")
  -port int
        server port (default 8080)
  -verbose
        enable verbose output
```

**Important flag package behaviors:**

- Flags use a single dash: `-host`, not `--host` (the `flag` package accepts both `-host` and `--host`, but convention is single dash)
- Parsing stops at the first non-flag argument (or after `--`)
- Boolean flags: `-verbose` (no value needed), `-verbose=true`, `-verbose=false`
- Unrecognized flags cause the program to exit with usage info

### 7.3 Subcommands with flag.FlagSet

For CLI tools with subcommands (like `git commit`, `docker run`):

```go
package main

import (
    "flag"
    "fmt"
    "os"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "Usage: tool <command> [options]")
        fmt.Fprintln(os.Stderr, "Commands: add, list")
        os.Exit(1)
    }

    // Create separate FlagSets for each subcommand
    addCmd := flag.NewFlagSet("add", flag.ExitOnError)
    addName := addCmd.String("name", "", "item name (required)")
    addCount := addCmd.Int("count", 1, "number of items")

    listCmd := flag.NewFlagSet("list", flag.ExitOnError)
    listAll := listCmd.Bool("all", false, "show all items including hidden")

    switch os.Args[1] {
    case "add":
        addCmd.Parse(os.Args[2:])
        if *addName == "" {
            fmt.Fprintln(os.Stderr, "Error: -name is required")
            addCmd.Usage()
            os.Exit(1)
        }
        fmt.Printf("Adding %d of '%s'\n", *addCount, *addName)

    case "list":
        listCmd.Parse(os.Args[2:])
        fmt.Printf("Listing items (all=%t)\n", *listAll)

    default:
        fmt.Fprintf(os.Stderr, "Unknown command: %s\n", os.Args[1])
        os.Exit(1)
    }
}
```

```bash
$ tool add -name="widget" -count=5
Adding 5 of 'widget'

$ tool list -all
Listing items (all=true)
```

### 7.4 Popular Third-Party CLI Libraries

The standard `flag` package is fine for simple tools, but real-world CLI applications usually need more. Two libraries dominate:

**Cobra** (used by Kubernetes, Docker, Hugo, GitHub CLI):

```go
package main

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

func main() {
    var verbose bool

    rootCmd := &cobra.Command{
        Use:   "myapp",
        Short: "My awesome CLI tool",
        Long:  "A longer description of what the tool does.",
    }

    greetCmd := &cobra.Command{
        Use:   "greet [name]",
        Short: "Greet someone",
        Args:  cobra.ExactArgs(1), // Require exactly one argument
        Run: func(cmd *cobra.Command, args []string) {
            name := args[0]
            if verbose {
                fmt.Printf("About to greet %s...\n", name)
            }
            fmt.Printf("Hello, %s!\n", name)
        },
    }

    // Persistent flags are inherited by all subcommands
    rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")

    rootCmd.AddCommand(greetCmd)

    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

```bash
$ myapp greet -v Alice
About to greet Alice...
Hello, Alice!
```

**urfave/cli** (simpler API, less boilerplate):

```go
package main

import (
    "fmt"
    "os"

    "github.com/urfave/cli/v2"
)

func main() {
    app := &cli.App{
        Name:  "myapp",
        Usage: "A useful CLI tool",
        Flags: []cli.Flag{
            &cli.BoolFlag{Name: "verbose", Aliases: []string{"v"}},
        },
        Commands: []*cli.Command{
            {
                Name:  "greet",
                Usage: "Greet someone",
                Action: func(c *cli.Context) error {
                    name := c.Args().First()
                    if c.Bool("verbose") {
                        fmt.Printf("About to greet %s...\n", name)
                    }
                    fmt.Printf("Hello, %s!\n", name)
                    return nil
                },
            },
        },
    }

    if err := app.Run(os.Args); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

**Node.js equivalents:**
- `flag` package is similar to `minimist` (bare-bones)
- `cobra` is similar to `commander` (feature-rich, subcommands)
- `urfave/cli` is similar to `yargs` (declarative configuration)

---

## 8. Building CLI Tools

### WHY Go Is Excellent for CLI Tools

1. **Single binary:** `go build` produces one executable file. No runtime to install, no `node_modules`, no virtual environment. Just copy the binary and run it.
2. **Fast startup:** Go programs start in milliseconds. Node.js CLI tools have noticeable startup lag from loading the V8 engine and parsing JavaScript.
3. **Cross-compilation:** Build for Linux, macOS, and Windows from any platform with a single command.
4. **Static linking (default):** The binary includes everything it needs. No shared library headaches.
5. **Low memory footprint:** A typical Go CLI tool uses 5-10MB of RAM. Node.js starts at 30-50MB just for the runtime.

### 8.1 Complete Example: A File Search CLI Tool

Let us build `fsearch`, a practical CLI tool that searches for files by name pattern and optionally searches content, like a simplified combination of `find` and `grep`.

```go
package main

import (
    "bufio"
    "flag"
    "fmt"
    "io/fs"
    "os"
    "path/filepath"
    "regexp"
    "strings"
    "time"
)

// ANSI color codes for terminal output
const (
    colorReset  = "\033[0m"
    colorRed    = "\033[31m"
    colorGreen  = "\033[32m"
    colorYellow = "\033[33m"
    colorCyan   = "\033[36m"
)

// Config holds the parsed command-line options
type Config struct {
    Root       string
    Pattern    string         // filename pattern (glob)
    Content    string         // content search (regex)
    ContentRe  *regexp.Regexp // compiled content regex
    MaxDepth   int
    IgnoreDirs []string
    ShowSize   bool
    NoColor    bool
}

// Result represents a single search match
type Result struct {
    Path    string
    Size    int64
    Line    int    // 0 if not a content match
    Content string // the matching line
}

func main() {
    config := parseFlags()
    results, stats := search(config)
    printResults(config, results, stats)
}

func parseFlags() Config {
    root := flag.String("root", ".", "root directory to search")
    pattern := flag.String("name", "*", "filename glob pattern (e.g., '*.go')")
    content := flag.String("content", "", "search file contents (regex)")
    maxDepth := flag.Int("depth", -1, "maximum directory depth (-1 for unlimited)")
    ignoreDirs := flag.String("ignore", ".git,node_modules,vendor,.idea",
        "comma-separated directories to skip")
    showSize := flag.Bool("size", false, "show file sizes")
    noColor := flag.Bool("no-color", false, "disable colored output")

    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, `fsearch - Search for files by name and content

Usage:
  fsearch [options]

Examples:
  fsearch -name "*.go"                    Find all Go files
  fsearch -name "*.go" -content "TODO"    Find Go files containing "TODO"
  fsearch -name "*.log" -root /var/log    Search in /var/log for .log files
  fsearch -name "*.js" -ignore "node_modules,dist"

Options:
`)
        flag.PrintDefaults()
    }

    flag.Parse()

    config := Config{
        Root:       *root,
        Pattern:    *pattern,
        MaxDepth:   *maxDepth,
        IgnoreDirs: strings.Split(*ignoreDirs, ","),
        ShowSize:   *showSize,
        NoColor:    *noColor,
    }

    if *content != "" {
        re, err := regexp.Compile(*content)
        if err != nil {
            fmt.Fprintf(os.Stderr, "Invalid content regex: %v\n", err)
            os.Exit(1)
        }
        config.Content = *content
        config.ContentRe = re
    }

    return config
}

type SearchStats struct {
    FilesScanned int
    DirsScanned  int
    MatchCount   int
    Duration     time.Duration
}

func search(config Config) ([]Result, SearchStats) {
    var results []Result
    var stats SearchStats
    start := time.Now()

    rootDepth := strings.Count(filepath.Clean(config.Root), string(os.PathSeparator))

    err := filepath.WalkDir(config.Root, func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return nil // Skip inaccessible files
        }

        // Skip ignored directories
        if d.IsDir() {
            stats.DirsScanned++

            for _, ignore := range config.IgnoreDirs {
                if d.Name() == ignore {
                    return filepath.SkipDir
                }
            }

            // Check max depth
            if config.MaxDepth >= 0 {
                currentDepth := strings.Count(
                    filepath.Clean(path), string(os.PathSeparator)) - rootDepth
                if currentDepth > config.MaxDepth {
                    return filepath.SkipDir
                }
            }

            return nil
        }

        stats.FilesScanned++

        // Match filename pattern
        matched, _ := filepath.Match(config.Pattern, d.Name())
        if !matched {
            return nil
        }

        // If no content search, just add the file
        if config.ContentRe == nil {
            var size int64
            if config.ShowSize {
                info, err := d.Info()
                if err == nil {
                    size = info.Size()
                }
            }
            results = append(results, Result{Path: path, Size: size})
            stats.MatchCount++
            return nil
        }

        // Content search
        file, err := os.Open(path)
        if err != nil {
            return nil
        }
        defer file.Close()

        scanner := bufio.NewScanner(file)
        lineNum := 0
        for scanner.Scan() {
            lineNum++
            line := scanner.Text()
            if config.ContentRe.MatchString(line) {
                results = append(results, Result{
                    Path:    path,
                    Line:    lineNum,
                    Content: line,
                })
                stats.MatchCount++
            }
        }

        return nil
    })

    if err != nil {
        fmt.Fprintf(os.Stderr, "Walk error: %v\n", err)
    }

    stats.Duration = time.Since(start)
    return results, stats
}

func printResults(config Config, results []Result, stats SearchStats) {
    for _, r := range results {
        if config.ContentRe != nil {
            // Content match: show file:line:content
            if config.NoColor {
                fmt.Printf("%s:%d: %s\n", r.Path, r.Line, r.Content)
            } else {
                // Highlight the match in the content
                highlighted := config.ContentRe.ReplaceAllString(r.Content,
                    colorRed+"${0}"+colorReset)
                fmt.Printf("%s%s%s:%s%d%s: %s\n",
                    colorCyan, r.Path, colorReset,
                    colorGreen, r.Line, colorReset,
                    highlighted)
            }
        } else {
            // Filename match
            if config.ShowSize {
                fmt.Printf("%10s  %s\n", formatSize(r.Size), r.Path)
            } else {
                if config.NoColor {
                    fmt.Println(r.Path)
                } else {
                    fmt.Printf("%s%s%s\n", colorCyan, r.Path, colorReset)
                }
            }
        }
    }

    // Summary to stderr (so stdout can be piped)
    fmt.Fprintf(os.Stderr, "\n%s%d matches%s in %d files, %d dirs (%.2fms)\n",
        colorYellow, stats.MatchCount, colorReset,
        stats.FilesScanned, stats.DirsScanned,
        float64(stats.Duration.Microseconds())/1000)
}

func formatSize(bytes int64) string {
    const (
        KB = 1024
        MB = KB * 1024
        GB = MB * 1024
    )
    switch {
    case bytes >= GB:
        return fmt.Sprintf("%.1f GB", float64(bytes)/float64(GB))
    case bytes >= MB:
        return fmt.Sprintf("%.1f MB", float64(bytes)/float64(MB))
    case bytes >= KB:
        return fmt.Sprintf("%.1f KB", float64(bytes)/float64(KB))
    default:
        return fmt.Sprintf("%d B", bytes)
    }
}
```

```bash
# Build it
$ go build -o fsearch .

# Find all Go files
$ ./fsearch -name "*.go"

# Find TODOs in Go files
$ ./fsearch -name "*.go" -content "TODO|FIXME|HACK"

# Search with size info, ignoring vendor
$ ./fsearch -name "*.json" -size -ignore ".git,vendor,node_modules"

# Pipe results to other tools (summary goes to stderr, results to stdout)
$ ./fsearch -name "*.go" -no-color | xargs wc -l
```

### 8.2 Reading from stdin

Good CLI tools work with pipes. Here is how to detect and read from stdin:

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    // Check if stdin is a pipe (not a terminal)
    stat, _ := os.Stdin.Stat()
    isPiped := (stat.Mode() & os.ModeCharDevice) == 0

    if isPiped {
        // Reading from a pipe: cat file.txt | program
        scanner := bufio.NewScanner(os.Stdin)
        lineCount := 0
        for scanner.Scan() {
            lineCount++
            line := scanner.Text()
            fmt.Printf("%4d | %s\n", lineCount, strings.TrimRight(line, " \t"))
        }
    } else {
        // Interactive mode: user is typing directly
        fmt.Println("Enter text (Ctrl+D to finish):")
        scanner := bufio.NewScanner(os.Stdin)
        for scanner.Scan() {
            line := scanner.Text()
            fmt.Printf("You typed: %s\n", line)
        }
    }
}
```

### 8.3 Exit Codes

CLI tools communicate success/failure through exit codes. This matters for scripting and automation.

```go
package main

import (
    "fmt"
    "os"
)

const (
    ExitOK           = 0 // Success
    ExitUsageError   = 1 // Bad arguments
    ExitRuntimeError = 2 // Runtime failure
)

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "Usage: program <filename>")
        os.Exit(ExitUsageError)
    }

    filename := os.Args[1]
    _, err := os.Stat(filename)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(ExitRuntimeError)
    }

    fmt.Printf("File %s exists\n", filename)
    // Implicit os.Exit(0) when main returns
}
```

**Tip:** Always write error messages to `os.Stderr` (via `fmt.Fprintln(os.Stderr, ...)` or `fmt.Fprintf(os.Stderr, ...)`), not `os.Stdout`. This lets users pipe your stdout to other programs without error messages contaminating the data.

---

## 9. os/exec -- Running External Commands

### 9.1 Simple Command Execution

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
)

func main() {
    // Run a command and capture its output
    output, err := exec.Command("date").Output()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    fmt.Printf("Date: %s", output) // output includes trailing newline

    // Run with arguments
    output, err = exec.Command("ls", "-la", "/tmp").Output()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }
    fmt.Printf("Files:\n%s", output)
}
```

**Node.js equivalent:**

```javascript
const { execFileSync } = require('child_process');
const output = execFileSync('date').toString();
console.log(`Date: ${output}`);
```

### 9.2 Capturing stdout and stderr Separately

```go
package main

import (
    "bytes"
    "fmt"
    "os"
    "os/exec"
)

func main() {
    cmd := exec.Command("go", "build", "./...")

    var stdout, stderr bytes.Buffer
    cmd.Stdout = &stdout
    cmd.Stderr = &stderr

    err := cmd.Run()

    if err != nil {
        // The command failed -- stderr usually has the reason
        fmt.Fprintf(os.Stderr, "Build failed: %v\n", err)
        fmt.Fprintf(os.Stderr, "stderr: %s\n", stderr.String())
        os.Exit(1)
    }

    if stdout.Len() > 0 {
        fmt.Printf("stdout: %s\n", stdout.String())
    }
}
```

### 9.3 CombinedOutput -- When You Don't Care Which Stream

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
)

func main() {
    // CombinedOutput merges stdout and stderr
    output, err := exec.Command("git", "status").CombinedOutput()
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\noutput: %s\n", err, output)
        os.Exit(1)
    }
    fmt.Printf("%s", output)
}
```

### 9.4 Checking Exit Codes

```go
package main

import (
    "errors"
    "fmt"
    "os/exec"
)

func main() {
    cmd := exec.Command("grep", "-r", "pattern", "/nonexistent")
    err := cmd.Run()

    if err != nil {
        // Check if it's an ExitError (command ran but returned non-zero)
        var exitErr *exec.ExitError
        if errors.As(err, &exitErr) {
            fmt.Printf("Command exited with code: %d\n", exitErr.ExitCode())
        } else {
            // Command couldn't be found or couldn't start
            fmt.Printf("Failed to start command: %v\n", err)
        }
    }
}
```

### 9.5 Piping Between Commands

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
)

func main() {
    // Equivalent of: cat /etc/passwd | grep root | head -1

    cat := exec.Command("cat", "/etc/passwd")
    grep := exec.Command("grep", "root")
    head := exec.Command("head", "-1")

    // Connect them with pipes
    var err error
    grep.Stdin, err = cat.StdoutPipe()
    if err != nil {
        fmt.Fprintf(os.Stderr, "pipe error: %v\n", err)
        os.Exit(1)
    }

    head.Stdin, err = grep.StdoutPipe()
    if err != nil {
        fmt.Fprintf(os.Stderr, "pipe error: %v\n", err)
        os.Exit(1)
    }

    head.Stdout = os.Stdout

    // Start all commands (must start in reverse order of pipeline)
    head.Start()
    grep.Start()
    cat.Start()

    // Wait for all to finish
    cat.Wait()
    grep.Wait()
    head.Wait()
}

// simplePipe shows a simpler approach for two-command piping using StdoutPipe
func simplePipe() {
    cmd1 := exec.Command("ls", "-la")
    cmd2 := exec.Command("grep", ".go")

    // Get cmd1's stdout
    reader, err := cmd1.StdoutPipe()
    if err != nil {
        return
    }

    // Set cmd2's stdin
    cmd2.Stdin = reader
    cmd2.Stdout = os.Stdout

    cmd1.Start()
    cmd2.Start()

    cmd1.Wait()
    cmd2.Wait()
}

// shellPipe uses sh -c for complex pipelines
func shellPipe() {
    // When the pipeline is complex, just use sh -c
    cmd := exec.Command("sh", "-c",
        "ps aux | grep go | grep -v grep | awk '{print $2, $11}'")
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    cmd.Run()
}
```

**Important security note on `exec.Command` vs shell invocation:**

`exec.Command("ls", "-la")` runs `ls` directly with arguments as a list. This is safe from shell injection because arguments are never parsed by a shell. By contrast, `exec.Command("sh", "-c", userInput)` runs through the shell and is vulnerable if `userInput` is not sanitized. Always prefer the direct form with separate arguments when possible.

### 9.6 Running Commands with a Timeout

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "time"
)

func main() {
    // Create a context that cancels after 5 seconds
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // CommandContext will kill the process if the context expires
    cmd := exec.CommandContext(ctx, "sleep", "30")
    err := cmd.Run()

    if ctx.Err() == context.DeadlineExceeded {
        fmt.Fprintln(os.Stderr, "Command timed out")
        os.Exit(1)
    }
    if err != nil {
        fmt.Fprintf(os.Stderr, "Command failed: %v\n", err)
        os.Exit(1)
    }
}
```

### 9.7 Setting Working Directory and Environment

```go
package main

import (
    "fmt"
    "os"
    "os/exec"
)

func main() {
    cmd := exec.Command("go", "build", "-o", "myapp", ".")

    // Set working directory
    cmd.Dir = "/path/to/project"

    // Set environment (replaces entire environment)
    cmd.Env = append(os.Environ(),
        "CGO_ENABLED=0",
        "GOOS=linux",
        "GOARCH=amd64",
    )

    // Connect to parent's stdout/stderr for real-time output
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    if err := cmd.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Build failed: %v\n", err)
        os.Exit(1)
    }
}
```

### 9.8 Checking if a Command Exists

```go
package main

import (
    "fmt"
    "os/exec"
)

func commandExists(name string) bool {
    _, err := exec.LookPath(name)
    return err == nil
}

func main() {
    tools := []string{"git", "docker", "kubectl", "nonexistent-tool"}

    for _, tool := range tools {
        if commandExists(tool) {
            path, _ := exec.LookPath(tool)
            fmt.Printf("[OK]   %-20s -> %s\n", tool, path)
        } else {
            fmt.Printf("[MISS] %-20s (not found in PATH)\n", tool)
        }
    }
}
```

---

## 10. Cross-Compilation

This is one of Go's genuine superpowers. You can build binaries for any supported OS and architecture from any machine, with no extra tools required.

### 10.1 The Basics: GOOS and GOARCH

```bash
# Build for Linux on AMD64 (most common server target)
GOOS=linux GOARCH=amd64 go build -o myapp-linux-amd64 .

# Build for macOS on Apple Silicon
GOOS=darwin GOARCH=arm64 go build -o myapp-darwin-arm64 .

# Build for macOS on Intel
GOOS=darwin GOARCH=amd64 go build -o myapp-darwin-amd64 .

# Build for Windows
GOOS=windows GOARCH=amd64 go build -o myapp-windows-amd64.exe .

# Build for Linux on ARM (Raspberry Pi, AWS Graviton)
GOOS=linux GOARCH=arm64 go build -o myapp-linux-arm64 .
```

**That is it.** No cross-compilation toolchain. No docker. No VM. Just environment variables.

### 10.2 Common GOOS/GOARCH Combinations

| GOOS | GOARCH | Use Case |
|------|--------|----------|
| `linux` | `amd64` | Most servers, Docker containers, CI/CD |
| `linux` | `arm64` | AWS Graviton, newer Raspberry Pi, ARM servers |
| `linux` | `arm` | Older Raspberry Pi, embedded devices |
| `darwin` | `arm64` | macOS on Apple Silicon (M1/M2/M3/M4) |
| `darwin` | `amd64` | macOS on Intel |
| `windows` | `amd64` | Windows desktops and servers |
| `freebsd` | `amd64` | FreeBSD servers |

### 10.3 Build Script for Multiple Platforms

```go
//go:build ignore

// build-all.go - Cross-compile for all target platforms
// Run with: go run build-all.go

package main

import (
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
    "runtime"
    "sync"
    "time"
)

type Target struct {
    GOOS   string
    GOARCH string
}

func main() {
    appName := "fsearch"
    version := "1.0.0"
    outputDir := "dist"

    targets := []Target{
        {"linux", "amd64"},
        {"linux", "arm64"},
        {"darwin", "amd64"},
        {"darwin", "arm64"},
        {"windows", "amd64"},
    }

    // Create output directory
    os.MkdirAll(outputDir, 0755)

    start := time.Now()
    var wg sync.WaitGroup
    errors := make(chan error, len(targets))

    for _, target := range targets {
        wg.Add(1)
        go func(t Target) {
            defer wg.Done()

            ext := ""
            if t.GOOS == "windows" {
                ext = ".exe"
            }

            output := filepath.Join(outputDir,
                fmt.Sprintf("%s-%s-%s-%s%s",
                    appName, version, t.GOOS, t.GOARCH, ext))

            fmt.Printf("Building %s...\n", output)

            cmd := exec.Command("go", "build",
                "-ldflags", fmt.Sprintf("-s -w -X main.version=%s", version),
                "-o", output,
                ".",
            )
            cmd.Env = append(os.Environ(),
                "GOOS="+t.GOOS,
                "GOARCH="+t.GOARCH,
                "CGO_ENABLED=0", // Disable CGO for truly static binaries
            )
            cmd.Stdout = os.Stdout
            cmd.Stderr = os.Stderr

            if err := cmd.Run(); err != nil {
                errors <- fmt.Errorf("%s/%s: %w", t.GOOS, t.GOARCH, err)
            }
        }(target)
    }

    wg.Wait()
    close(errors)

    for err := range errors {
        fmt.Fprintf(os.Stderr, "ERROR: %v\n", err)
    }

    fmt.Printf("\nBuilt %d targets in %v (using %d CPUs)\n",
        len(targets), time.Since(start).Round(time.Millisecond), runtime.NumCPU())
}
```

### 10.4 Embedding Version Info with ldflags

```go
package main

import (
    "fmt"
    "runtime"
)

// These are set at build time via -ldflags
var (
    version   = "dev"
    commit    = "none"
    buildDate = "unknown"
)

func main() {
    fmt.Printf("myapp %s\n", version)
    fmt.Printf("  commit:  %s\n", commit)
    fmt.Printf("  built:   %s\n", buildDate)
    fmt.Printf("  go:      %s\n", runtime.Version())
    fmt.Printf("  os/arch: %s/%s\n", runtime.GOOS, runtime.GOARCH)
}
```

```bash
# Build with version info injected at compile time
go build \
  -ldflags "-X main.version=1.2.3 \
            -X main.commit=$(git rev-parse --short HEAD) \
            -X 'main.buildDate=$(date -u)'" \
  -o myapp .
```

### 10.5 CGO_ENABLED=0 -- Truly Static Binaries

By default, Go links against the system's C library for DNS resolution and some other OS interactions. Setting `CGO_ENABLED=0` produces a fully static binary that works on any Linux system, regardless of glibc version.

```bash
# This binary will run on ANY Linux system -- even Alpine, scratch Docker images, etc.
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o myapp .
```

This is why Go is so popular for Docker containers:

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o /myapp .

# Final stage -- scratch means literally empty, no OS at all
FROM scratch
COPY --from=builder /myapp /myapp
ENTRYPOINT ["/myapp"]
```

The resulting Docker image contains ONLY your binary. No shell, no libc, no package manager, nothing. Typical image size: 5-15MB.

**Node.js comparison:**

```dockerfile
# Node.js requires the entire runtime
FROM node:20-slim
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY . .
CMD ["node", "index.js"]
# Final image: 200-500MB typically
```

### 10.6 Listing All Supported Platforms

```bash
# See every OS/architecture combination Go can target
$ go tool dist list
aix/ppc64
android/386
android/amd64
android/arm
android/arm64
darwin/amd64
darwin/arm64
freebsd/386
freebsd/amd64
freebsd/arm
freebsd/arm64
linux/386
linux/amd64
linux/arm
linux/arm64
linux/mips
linux/mips64
...
# (60+ combinations)
```

There is no equivalent in the Node.js world. To distribute a Node.js application as a standalone binary, you need tools like `pkg`, `nexe`, or the newer Node.js Single Executable Applications (SEA) feature, and they all come with significant limitations.

---

## 11. Go vs Node.js Comparison Summary

### File I/O

| Task | Go | Node.js |
|------|-----|---------|
| Read whole file | `os.ReadFile(path)` | `fs.readFileSync(path, 'utf8')` |
| Read file streaming | `os.Open` + `bufio.Scanner` | `fs.createReadStream` + `readline` |
| Write whole file | `os.WriteFile(path, data, perm)` | `fs.writeFileSync(path, data)` |
| Write streaming | `os.Create` + `bufio.Writer` | `fs.createWriteStream` |
| Append to file | `os.OpenFile` with `O_APPEND` | `fs.appendFileSync` |
| Copy file | `io.Copy(dst, src)` | `fs.copyFileSync(src, dst)` |
| File exists | `os.Stat` + check error | `fs.existsSync(path)` |
| Delete file | `os.Remove(path)` | `fs.unlinkSync(path)` |
| Delete dir tree | `os.RemoveAll(path)` | `fs.rmSync(path, {recursive: true})` |
| Temp file | `os.CreateTemp("", "*.txt")` | `fs.mkdtempSync` + manual |
| List directory | `os.ReadDir(path)` | `fs.readdirSync(path)` |
| Walk directory | `filepath.WalkDir` | `fs.readdirSync({recursive: true})` |

### Stream/IO Composability

| Concept | Go | Node.js |
|---------|-----|---------|
| Universal read interface | `io.Reader` (1 method) | `stream.Readable` (complex class) |
| Universal write interface | `io.Writer` (1 method) | `stream.Writable` (complex class) |
| Connect reader to writer | `io.Copy(w, r)` | `readable.pipe(writable)` |
| Tee (read + copy) | `io.TeeReader` | `stream.PassThrough` + manual |
| Multi-destination write | `io.MultiWriter` | Custom or npm package |
| Buffered write | `bufio.Writer` | Built into `Writable` |
| Limit reading | `io.LimitReader` | Manual or npm package |

**Key difference:** Go's `io.Reader`/`io.Writer` are synchronous, explicit, and trivially composable. Node.js streams are event-driven, have flowing/paused modes, backpressure, and a steep learning curve. Go's approach is dramatically simpler for most use cases.

### CLI Tooling

| Aspect | Go | Node.js |
|--------|-----|---------|
| Argument parsing | `flag` (stdlib) | `process.argv` (manual) |
| Advanced CLI framework | `cobra`, `urfave/cli` | `commander`, `yargs` |
| Distribution | Single binary, `go install` | `npm install -g`, requires Node.js |
| Startup time | ~5ms | ~100-200ms |
| Memory usage | ~5-10 MB | ~30-50 MB |
| Cross-compilation | Built-in (`GOOS`/`GOARCH`) | Requires `pkg`, `nexe`, or `sea` |
| Binary size | ~5-15 MB | ~40-80 MB (bundled) |

---

## 12. Key Takeaways

1. **Use `os.ReadFile`/`os.WriteFile` for small files.** They are simple and correct. Switch to streaming only when you have a reason (large files, line-by-line processing, or long-running operations).

2. **Always `defer file.Close()` immediately after opening.** This is not optional -- it is the Go way to prevent resource leaks. Put it right after the error check, every single time.

3. **`io.Reader` and `io.Writer` are the heart of Go's I/O system.** Write functions that accept these interfaces, not concrete types. Your function that takes `io.Reader` works with files, network connections, compressed streams, strings, and things that have not been invented yet.

4. **`io.Copy` is your best friend.** Need to copy a file? `io.Copy`. Download a URL to disk? `io.Copy`. Pipe data between commands? `io.Copy`. It handles buffering, it handles EOF, it handles everything.

5. **Use `bufio.Scanner` for line-by-line reading, `bufio.Writer` for many small writes.** Buffering turns many system calls into few system calls, which is a massive performance win.

6. **Use `filepath` (not `path`) for filesystem paths.** The `filepath` package uses OS-appropriate separators. The `path` package is for URL-style forward-slash paths only.

7. **Write errors to stderr, data to stdout.** This is essential for CLI tools that participate in pipelines.

8. **`os.LookupEnv` vs `os.Getenv`:** Use `LookupEnv` when the distinction between "not set" and "set to empty" matters. Use `Getenv` when you just want the value (empty string if missing).

9. **Go's cross-compilation is trivial and powerful.** Two environment variables (`GOOS`, `GOARCH`) and you build for any platform. Add `CGO_ENABLED=0` for fully static binaries. This is why Go dominates the CLI tool ecosystem.

10. **Prefer `filepath.WalkDir` over `filepath.Walk`.** `WalkDir` is lazier (avoids unnecessary `Stat` calls) and was specifically designed as the improved replacement.

---

## 13. Practice Exercises

### Exercise 1: Line Counter (Beginner)
Build a CLI tool that counts lines, words, and characters in one or more files (like `wc`).

**Requirements:**
- Accept one or more filenames as arguments
- If no filenames, read from stdin
- Display lines, words, characters for each file
- Display totals if multiple files

**Hints:** Use `bufio.Scanner` with `ScanLines` for lines, `strings.Fields` for word counting.

---

### Exercise 2: File Deduplicator (Intermediate)
Build a tool that finds duplicate files in a directory tree by their content hash.

**Requirements:**
- Walk a directory recursively
- Compute SHA256 hash of each file
- Group files by hash
- Report groups with more than one file (duplicates)
- Add a `-delete` flag that removes duplicates (keeping the first)

**Hints:** Use `filepath.WalkDir`, `crypto/sha256`, `io.Copy` to a hasher, `map[string][]string` for grouping.

---

### Exercise 3: Log Tailer (Intermediate)
Build a tool that watches a log file for new lines (like `tail -f`) and optionally filters them.

**Requirements:**
- Accept a filename and optional regex filter
- Print new lines as they appear
- Handle the file being rotated (recreated)
- Add colorized output for different log levels (ERROR=red, WARN=yellow, INFO=green)

**Hints:** Use `os.Stat` to detect file changes, `time.Tick` for polling, `regexp` for filtering.

---

### Exercise 4: Multi-Platform Release Script (Intermediate)
Build a Go program that cross-compiles itself for multiple platforms and creates a release directory.

**Requirements:**
- Build for linux/amd64, linux/arm64, darwin/amd64, darwin/arm64, windows/amd64
- Embed version, commit, and build date using `-ldflags`
- Compute SHA256 checksums for each binary
- Write a `checksums.txt` file

**Hints:** Use `os/exec`, `crypto/sha256`, `sync.WaitGroup` for parallel builds.

---

### Exercise 5: Configuration File Merger (Advanced)
Build a CLI tool that merges configuration from multiple sources with precedence rules.

**Requirements:**
- Read a base config from a JSON/YAML file
- Override with values from environment variables (prefixed with `APP_`)
- Override with command-line flags
- Print the final merged configuration
- Support nested keys (e.g., `APP_DATABASE_HOST` maps to `database.host`)

**Hints:** Use `encoding/json`, `os.Environ`, `flag` package, `strings.Split` for key paths.

---

### Exercise 6: Directory Sync Tool (Advanced)
Build a tool that synchronizes two directories (like a simplified `rsync`).

**Requirements:**
- Compare source and destination directory trees
- Copy new files from source to destination
- Update files where source is newer (compare modification times)
- Add a `-dry-run` flag that shows what would happen without doing it
- Add a `-delete` flag that removes files in destination not present in source
- Display a summary of operations

**Hints:** Use `filepath.WalkDir` on both directories, `os.Stat` for modification times, `io.Copy` for file copying, `map[string]os.FileInfo` for building file inventories.

---

**Next Chapter:** Chapter 14 will cover Testing in Go -- unit tests, table-driven tests, benchmarks, mocking, and how Go's testing philosophy differs from the Node.js ecosystem.
