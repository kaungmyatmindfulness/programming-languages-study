# Chapter 34: Storage Engines from Scratch

## Prerequisites

Before starting this chapter, you should be comfortable with:

- **Chapter 7** -- Pointers and memory (understanding how Go manages memory, heap vs stack)
- **Chapter 10** -- Goroutines and concurrency basics (goroutines, sync.Mutex, sync.RWMutex)
- **Chapter 13** -- File I/O and CLI tools (os.File, bufio, reading/writing files)
- **Chapter 14** -- Testing and benchmarking (writing benchmarks to measure performance)
- **Chapter 17** -- Advanced concurrency (sync.RWMutex patterns, concurrent data structures)
- **Chapter 20** -- Performance optimization (profiling, memory optimization, I/O optimization)

This chapter is based on **Chapter 3 of "Designing Data-Intensive Applications" by Martin Kleppmann** -- one of the most important chapters in one of the most important books in software engineering. DDIA Chapter 3 explains how databases store and retrieve data internally. We will take every concept from that chapter and implement it in Go, from the simplest append-only log to a complete LSM-tree key-value store.

If you come from Node.js, you have likely used databases as black boxes -- calling `db.get()` and `db.set()` without understanding what happens on disk. This chapter will change that. You will understand why some databases are fast for writes and slow for reads, why others are the opposite, and why the answer is always "it depends on the storage engine."

By the end of this chapter, you will have built a working key-value database from scratch. Not a toy. A real database with crash recovery, compaction, bloom filters, and efficient range queries.

---

## Table of Contents

1. [Why Understand Storage Engines?](#1-why-understand-storage-engines)
2. [The Simplest Database](#2-the-simplest-database)
3. [Hash Index](#3-hash-index)
4. [Log Segmentation and Compaction](#4-log-segmentation-and-compaction)
5. [SSTable (Sorted String Table)](#5-sstable-sorted-string-table)
6. [Memtable with Red-Black Tree](#6-memtable-with-red-black-tree)
7. [LSM Tree (Log-Structured Merge Tree)](#7-lsm-tree-log-structured-merge-tree)
8. [B-Tree Basics](#8-b-tree-basics)
9. [B-Tree Implementation](#9-b-tree-implementation)
10. [Write-Ahead Log (WAL)](#10-write-ahead-log-wal)
11. [Bloom Filters](#11-bloom-filters)
12. [LSM vs B-Tree Comparison](#12-lsm-vs-b-tree-comparison)
13. [Column-Oriented Storage Concepts](#13-column-oriented-storage-concepts)
14. [Real-World Example: Building a Complete Key-Value Store](#14-real-world-example-building-a-complete-key-value-store)
15. [Key Takeaways](#15-key-takeaways)
16. [Practice Exercises](#16-practice-exercises)

---

## 1. Why Understand Storage Engines?

### The Black Box Problem

Most developers treat databases as black boxes. You call `INSERT`, you call `SELECT`, and data appears. But when your production database starts slowing down at 3 AM and your pager goes off, "it's a black box" is not an answer your team wants to hear.

Understanding storage engines gives you three superpowers:

1. **Choosing the right database.** PostgreSQL, MySQL, LevelDB, RocksDB, Cassandra, MongoDB -- they all use different storage engines with different tradeoffs. If you understand the tradeoffs, you can pick the right tool before your system is built, not after it falls over.

2. **Debugging performance.** When reads are slow, is it because your index does not fit in memory? When writes are slow, is it because of write amplification? When disk usage keeps growing, is it because compaction is not keeping up? These questions only make sense if you understand the engine.

3. **Designing systems correctly.** Should you use a relational database or a key-value store? Should you index this column? Should you denormalize? The answers depend on how the storage engine works.

### Read Optimization vs Write Optimization

Every storage engine makes a fundamental choice: optimize for reads, or optimize for writes. You cannot have both. This is not a limitation of any particular database -- it is a fundamental tradeoff in computer science.

```
Write-Optimized (Log-Structured)        Read-Optimized (Page-Structured)
─────────────────────────────────        ──────────────────────────────────
Append data sequentially                 Update data in-place
Fast writes (sequential I/O)             Slower writes (random I/O)
Slower reads (must check multiple        Fast reads (go directly to the
  files and merge results)                 right page)
Must compact in background               No compaction needed
Examples: LSM-tree, LevelDB,             Examples: B-tree, PostgreSQL,
  RocksDB, Cassandra                       MySQL InnoDB, SQLite
```

### Go vs Node.js: Why Go Is Perfect for This

In Node.js, you would not normally build a storage engine. The single-threaded event loop, garbage collection pauses, and lack of control over memory layout make it impractical. V8 does not give you `mmap`, `fsync`, or fine-grained control over when bytes hit disk.

Go, on the other hand, gives you:

- **Direct file I/O** with `os.File`, `syscall.Mmap`, and `syscall.Fsync`
- **Goroutines** for background compaction without blocking reads/writes
- **sync.RWMutex** for concurrent read access with exclusive write access
- **Predictable memory layout** with structs and slices (no hidden object overhead)
- **No event loop** -- every goroutine can block on disk I/O independently

This is why LevelDB (C++), RocksDB (C++), Badger (Go), Pebble (Go), and BoltDB (Go) are all written in systems languages. Let us build our own.

---

## 2. The Simplest Database

### The Append-Only Log

The simplest possible database is two shell commands:

```bash
# Set a key-value pair (append to file)
echo "key,value" >> database.txt

# Get a value by key (scan the file)
grep "^key," database.txt | tail -1 | cut -d',' -f2
```

Writes are O(1) -- you just append to the end of the file. Reads are O(n) -- you must scan the entire file to find the latest value for a key. This is terrible for reads but surprisingly useful as a foundation. Every log-structured storage engine starts here.

### Go Implementation

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strings"
	"sync"
	"time"
)

// LogDB is the simplest possible database: an append-only log file.
// Writes are O(1) -- just append to the end.
// Reads are O(n) -- scan the entire file from the end.
type LogDB struct {
	file *os.File
	mu   sync.RWMutex
	path string
}

// NewLogDB creates or opens an append-only log database.
func NewLogDB(path string) (*LogDB, error) {
	// O_CREATE: create file if it does not exist
	// O_RDWR: open for both reading and writing
	// O_APPEND: writes always go to end of file
	file, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR|os.O_APPEND, 0644)
	if err != nil {
		return nil, fmt.Errorf("open database file: %w", err)
	}
	return &LogDB{file: file, path: path}, nil
}

// Set appends a key-value pair to the log. This is always O(1) because
// appending to a file is a sequential write -- the fastest kind of I/O.
func (db *LogDB) Set(key, value string) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	// Format: key\tvalue\n
	// We use tab as delimiter because it is unlikely to appear in keys/values.
	// A production system would use a binary format with length prefixes.
	line := fmt.Sprintf("%s\t%s\n", key, value)
	_, err := db.file.WriteString(line)
	return err
}

// Get scans the entire file to find the most recent value for a key.
// This is O(n) where n is the number of records in the file.
// We scan from beginning to end and keep the last match, because
// the most recent write is the current value.
func (db *LogDB) Get(key string) (string, bool, error) {
	db.mu.RLock()
	defer db.mu.RUnlock()

	// Seek to the beginning of the file for reading
	if _, err := db.file.Seek(0, 0); err != nil {
		return "", false, fmt.Errorf("seek to start: %w", err)
	}

	var lastValue string
	found := false

	scanner := bufio.NewScanner(db.file)
	for scanner.Scan() {
		line := scanner.Text()
		parts := strings.SplitN(line, "\t", 2)
		if len(parts) == 2 && parts[0] == key {
			lastValue = parts[1]
			found = true
		}
	}

	if err := scanner.Err(); err != nil {
		return "", false, fmt.Errorf("scan file: %w", err)
	}

	return lastValue, found, nil
}

// Delete marks a key as deleted by writing a special tombstone value.
// The key still exists in the log, but Get will return "not found".
func (db *LogDB) Delete(key string) error {
	return db.Set(key, "<<TOMBSTONE>>")
}

// Close closes the database file.
func (db *LogDB) Close() error {
	return db.file.Close()
}

func main() {
	// Create the database
	db, err := NewLogDB("/tmp/logdb_example.db")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create database: %v\n", err)
		os.Exit(1)
	}
	defer db.Close()
	defer os.Remove("/tmp/logdb_example.db")

	// Write some data
	fmt.Println("=== Append-Only Log Database ===")
	fmt.Println()

	db.Set("name", "Alice")
	db.Set("age", "30")
	db.Set("city", "Portland")
	db.Set("name", "Bob") // Overwrite -- both entries exist in the log

	// Read data
	name, found, _ := db.Get("name")
	fmt.Printf("name = %q (found: %v)\n", name, found) // "Bob" -- last write wins

	age, found, _ := db.Get("age")
	fmt.Printf("age  = %q (found: %v)\n", age, found)

	// Key that does not exist
	_, found, _ = db.Get("email")
	fmt.Printf("email found: %v\n", found) // false

	// Benchmark: write 100,000 records, then read one
	fmt.Println()
	fmt.Println("=== Performance ===")

	start := time.Now()
	for i := 0; i < 100_000; i++ {
		db.Set(fmt.Sprintf("key-%d", i), fmt.Sprintf("value-%d", i))
	}
	writeTime := time.Since(start)
	fmt.Printf("100,000 writes: %v (%.0f writes/sec)\n",
		writeTime, 100_000/writeTime.Seconds())

	// Reading the last key requires scanning the entire file
	start = time.Now()
	val, found, _ := db.Get("key-99999")
	readTime := time.Since(start)
	fmt.Printf("1 read (scan entire file): %v, value=%q, found=%v\n",
		readTime, val, found)

	// The problem is clear: writes are fast, reads are slow.
	// The file now has 100,004 lines, and every read scans all of them.
	fmt.Println()
	fmt.Println("Problem: reads are O(n). We need an index.")
}
```

### Node.js Comparison

In Node.js, the same concept looks like this:

```javascript
// Node.js -- append-only log database
const fs = require('fs');
const readline = require('readline');

class LogDB {
  constructor(path) {
    this.path = path;
    // Node.js has no O_APPEND + read mode in a single fd easily,
    // so we use separate operations
  }

  async set(key, value) {
    // fs.appendFile is async -- it goes through libuv's thread pool
    await fs.promises.appendFile(this.path, `${key}\t${value}\n`);
  }

  async get(key) {
    // Must read the ENTIRE file to find the latest value
    const stream = fs.createReadStream(this.path);
    const rl = readline.createInterface({ input: stream });
    let lastValue = undefined;

    for await (const line of rl) {
      const [k, v] = line.split('\t');
      if (k === key) lastValue = v;
    }
    return lastValue;
  }
}

// Usage:
// const db = new LogDB('/tmp/logdb.txt');
// await db.set('name', 'Alice');
// const name = await db.get('name'); // scans entire file
```

The key differences:

| Aspect | Go | Node.js |
|--------|------|---------|
| File access | Direct `os.File` with `O_APPEND` | `fs.appendFile` through libuv |
| Concurrency | `sync.RWMutex` -- multiple readers, one writer | Single-threaded -- no lock needed but no parallelism |
| Read performance | Blocks the goroutine (others continue) | Blocks the event loop during CPU-intensive scanning |
| Sync to disk | Can call `file.Sync()` (fsync) directly | `fs.fsync()` available but rarely used |

---

## 3. Hash Index

### From O(n) to O(1) Reads

The append-only log has terrible read performance because we must scan the entire file for every read. The fix is simple: keep an in-memory hash map that maps each key to the byte offset in the file where its most recent value was written.

```
In-memory hash index:
  "name"  -> offset 0
  "age"   -> offset 15
  "city"  -> offset 27
  "name"  -> offset 42    (updated after second write)

File on disk:
  Offset 0:  name\tAlice\n
  Offset 15: age\t30\n
  Offset 27: city\tPortland\n
  Offset 42: name\tBob\n
```

Now reads are O(1): look up the key in the hash map, seek to that offset, read the value. This is exactly how **Bitcask** (the default storage engine in Riak) works.

The constraint: the hash map must fit in memory. If you have billions of keys, this will not work. But for millions of keys with small key sizes, it is excellent.

### Go Implementation

```go
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"io"
	"os"
	"sync"
	"time"
)

// HashIndexDB keeps an in-memory hash map of key -> file offset.
// Writes append to a log file AND update the index.
// Reads look up the offset in the index and seek directly to it.
type HashIndexDB struct {
	index map[string]int64 // key -> byte offset in file
	file  *os.File
	mu    sync.RWMutex
	path  string
}

// record is the on-disk format for a key-value pair.
// Format: [keyLen:4][valueLen:4][key:keyLen][value:valueLen]
// Using fixed-size length prefixes makes it easy to read records
// without scanning for delimiters.
type record struct {
	Key   string
	Value string
}

// NewHashIndexDB creates a new hash-indexed database.
// If the file already exists, it rebuilds the index by scanning the file.
func NewHashIndexDB(path string) (*HashIndexDB, error) {
	file, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return nil, fmt.Errorf("open file: %w", err)
	}

	db := &HashIndexDB{
		index: make(map[string]int64),
		file:  file,
		path:  path,
	}

	// Rebuild index from existing data
	if err := db.rebuildIndex(); err != nil {
		file.Close()
		return nil, fmt.Errorf("rebuild index: %w", err)
	}

	return db, nil
}

// writeRecord writes a single record to the file and returns the offset
// where it was written.
func (db *HashIndexDB) writeRecord(key, value string) (int64, error) {
	// Get current position (this is where the record starts)
	offset, err := db.file.Seek(0, io.SeekEnd)
	if err != nil {
		return 0, err
	}

	// Write key length (4 bytes, little-endian)
	if err := binary.Write(db.file, binary.LittleEndian, uint32(len(key))); err != nil {
		return 0, err
	}

	// Write value length (4 bytes, little-endian)
	if err := binary.Write(db.file, binary.LittleEndian, uint32(len(value))); err != nil {
		return 0, err
	}

	// Write key bytes
	if _, err := db.file.WriteString(key); err != nil {
		return 0, err
	}

	// Write value bytes
	if _, err := db.file.WriteString(value); err != nil {
		return 0, err
	}

	return offset, nil
}

// readRecordAt reads a record starting at the given byte offset.
func (db *HashIndexDB) readRecordAt(offset int64) (record, error) {
	if _, err := db.file.Seek(offset, io.SeekStart); err != nil {
		return record{}, err
	}

	var keyLen, valueLen uint32

	if err := binary.Read(db.file, binary.LittleEndian, &keyLen); err != nil {
		return record{}, err
	}
	if err := binary.Read(db.file, binary.LittleEndian, &valueLen); err != nil {
		return record{}, err
	}

	keyBuf := make([]byte, keyLen)
	if _, err := io.ReadFull(db.file, keyBuf); err != nil {
		return record{}, err
	}

	valueBuf := make([]byte, valueLen)
	if _, err := io.ReadFull(db.file, valueBuf); err != nil {
		return record{}, err
	}

	return record{Key: string(keyBuf), Value: string(valueBuf)}, nil
}

// rebuildIndex scans the entire file and reconstructs the in-memory index.
// This runs once at startup. On a 1GB file with 10M records, this takes
// a few seconds -- acceptable for a database startup.
func (db *HashIndexDB) rebuildIndex() error {
	if _, err := db.file.Seek(0, io.SeekStart); err != nil {
		return err
	}

	reader := bufio.NewReader(db.file)
	var offset int64

	for {
		var keyLen, valueLen uint32

		if err := binary.Read(reader, binary.LittleEndian, &keyLen); err != nil {
			if err == io.EOF {
				break
			}
			return err
		}
		if err := binary.Read(reader, binary.LittleEndian, &valueLen); err != nil {
			return err
		}

		keyBuf := make([]byte, keyLen)
		if _, err := io.ReadFull(reader, keyBuf); err != nil {
			return err
		}

		// Skip value bytes -- we only need the key for the index
		if _, err := io.CopyN(io.Discard, reader, int64(valueLen)); err != nil {
			return err
		}

		// Map this key to its offset
		db.index[string(keyBuf)] = offset

		// Advance offset: 4 (keyLen) + 4 (valueLen) + keyLen + valueLen
		offset += 4 + 4 + int64(keyLen) + int64(valueLen)
	}

	return nil
}

// Set writes a key-value pair. The write is O(1) for the file append
// and O(1) for the hash map update.
func (db *HashIndexDB) Set(key, value string) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	offset, err := db.writeRecord(key, value)
	if err != nil {
		return err
	}

	db.index[key] = offset
	return nil
}

// Get reads a value by key. This is O(1): one hash map lookup + one disk seek.
func (db *HashIndexDB) Get(key string) (string, bool, error) {
	db.mu.RLock()
	defer db.mu.RUnlock()

	offset, exists := db.index[key]
	if !exists {
		return "", false, nil
	}

	rec, err := db.readRecordAt(offset)
	if err != nil {
		return "", false, err
	}

	return rec.Value, true, nil
}

// Delete removes a key by writing a tombstone record.
func (db *HashIndexDB) Delete(key string) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	// Write tombstone
	if _, err := db.writeRecord(key, ""); err != nil {
		return err
	}

	// Remove from index
	delete(db.index, key)
	return nil
}

// Keys returns all keys currently in the database.
func (db *HashIndexDB) Keys() []string {
	db.mu.RLock()
	defer db.mu.RUnlock()

	keys := make([]string, 0, len(db.index))
	for k := range db.index {
		keys = append(keys, k)
	}
	return keys
}

// Close closes the database file.
func (db *HashIndexDB) Close() error {
	return db.file.Close()
}

func main() {
	db, err := NewHashIndexDB("/tmp/hashdb_example.db")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create database: %v\n", err)
		os.Exit(1)
	}
	defer db.Close()
	defer os.Remove("/tmp/hashdb_example.db")

	fmt.Println("=== Hash Index Database ===")
	fmt.Println()

	// Basic operations
	db.Set("name", "Alice")
	db.Set("age", "30")
	db.Set("city", "Portland")
	db.Set("name", "Bob") // overwrite

	name, found, _ := db.Get("name")
	fmt.Printf("name = %q (found: %v)\n", name, found)

	age, found, _ := db.Get("age")
	fmt.Printf("age  = %q (found: %v)\n", age, found)

	fmt.Printf("Total keys: %d\n", len(db.Keys()))

	// Performance comparison with LogDB
	fmt.Println()
	fmt.Println("=== Performance ===")

	start := time.Now()
	for i := 0; i < 100_000; i++ {
		db.Set(fmt.Sprintf("key-%d", i), fmt.Sprintf("value-%d", i))
	}
	writeTime := time.Since(start)
	fmt.Printf("100,000 writes: %v (%.0f writes/sec)\n",
		writeTime, 100_000/writeTime.Seconds())

	// Now reads are O(1) instead of O(n)
	start = time.Now()
	for i := 0; i < 1000; i++ {
		db.Get(fmt.Sprintf("key-%d", i))
	}
	readTime := time.Since(start)
	fmt.Printf("1,000 reads: %v (%.0f reads/sec)\n",
		readTime, 1000/readTime.Seconds())

	// Compare: reads are now constant time regardless of database size
	start = time.Now()
	val, found, _ := db.Get("key-99999")
	singleRead := time.Since(start)
	fmt.Printf("Single read: %v, value=%q, found=%v\n", singleRead, val, found)
}
```

### Limitations of Hash Indexes

The hash index is fast but has important limitations:

1. **All keys must fit in memory.** If you have 10 billion keys, the hash map alone could use hundreds of gigabytes of RAM.
2. **Range queries are not efficient.** You cannot scan all keys between "user:1000" and "user:2000" without checking every key in the map.
3. **Crash recovery is slow.** On restart, you must scan the entire file to rebuild the index. For a 100GB file, this could take minutes.

These limitations motivate the more sophisticated structures we will build next.

---

## 4. Log Segmentation and Compaction

### The Problem: Unbounded File Growth

With the hash index approach, the log file grows forever. Even if you update the same key a million times, all million entries remain in the file. Only the last one matters. The rest is garbage.

### The Solution: Segments and Compaction

Instead of one giant file, we split the log into **segments**. When a segment reaches a threshold size, we close it and start writing to a new segment. In the background, we **compact** old segments by throwing away superseded values and merging segments together.

```
Before compaction:
  Segment 1: [a=1] [b=2] [a=3] [c=4] [a=5]
  Segment 2: [b=6] [d=7] [a=8]

After compaction:
  Merged segment: [a=8] [b=6] [c=4] [d=7]
```

### Go Implementation

```go
package main

import (
	"encoding/binary"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"sort"
	"sync"
	"time"
)

const maxSegmentSize = 1024 * 1024 // 1 MB per segment

// Segment represents a single log file with its own hash index.
type Segment struct {
	path   string
	file   *os.File
	index  map[string]int64
	size   int64
	frozen bool // true = no more writes, eligible for compaction
}

// SegmentedDB splits writes across multiple segment files and
// compacts old segments in the background.
type SegmentedDB struct {
	dir      string
	active   *Segment   // current segment receiving writes
	frozen   []*Segment // old segments, newest first
	mu       sync.RWMutex
	segCount int
}

// NewSegmentedDB creates a new segmented database in the given directory.
func NewSegmentedDB(dir string) (*SegmentedDB, error) {
	if err := os.MkdirAll(dir, 0755); err != nil {
		return nil, err
	}

	db := &SegmentedDB{dir: dir}

	// Create the first active segment
	seg, err := db.newSegment()
	if err != nil {
		return nil, err
	}
	db.active = seg

	return db, nil
}

func (db *SegmentedDB) newSegment() (*Segment, error) {
	db.segCount++
	path := filepath.Join(db.dir, fmt.Sprintf("segment-%06d.dat", db.segCount))
	file, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return nil, err
	}
	return &Segment{
		path:  path,
		file:  file,
		index: make(map[string]int64),
	}, nil
}

func writeRecordToFile(f *os.File, key, value string) (int64, int64, error) {
	offset, err := f.Seek(0, io.SeekEnd)
	if err != nil {
		return 0, 0, err
	}

	if err := binary.Write(f, binary.LittleEndian, uint32(len(key))); err != nil {
		return 0, 0, err
	}
	if err := binary.Write(f, binary.LittleEndian, uint32(len(value))); err != nil {
		return 0, 0, err
	}
	if _, err := f.WriteString(key); err != nil {
		return 0, 0, err
	}
	if _, err := f.WriteString(value); err != nil {
		return 0, 0, err
	}

	recordSize := int64(8 + len(key) + len(value))
	return offset, recordSize, nil
}

func readRecordFromFile(f *os.File, offset int64) (string, string, error) {
	if _, err := f.Seek(offset, io.SeekStart); err != nil {
		return "", "", err
	}

	var keyLen, valueLen uint32
	if err := binary.Read(f, binary.LittleEndian, &keyLen); err != nil {
		return "", "", err
	}
	if err := binary.Read(f, binary.LittleEndian, &valueLen); err != nil {
		return "", "", err
	}

	keyBuf := make([]byte, keyLen)
	if _, err := io.ReadFull(f, keyBuf); err != nil {
		return "", "", err
	}

	valueBuf := make([]byte, valueLen)
	if _, err := io.ReadFull(f, valueBuf); err != nil {
		return "", "", err
	}

	return string(keyBuf), string(valueBuf), nil
}

// Set writes a key-value pair. If the active segment is full, it is
// frozen and a new segment is started.
func (db *SegmentedDB) Set(key, value string) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	// Check if we need to rotate to a new segment
	if db.active.size >= maxSegmentSize {
		db.active.frozen = true
		db.frozen = append([]*Segment{db.active}, db.frozen...)

		newSeg, err := db.newSegment()
		if err != nil {
			return err
		}
		db.active = newSeg
	}

	offset, recordSize, err := writeRecordToFile(db.active.file, key, value)
	if err != nil {
		return err
	}

	db.active.index[key] = offset
	db.active.size += recordSize
	return nil
}

// Get searches for a key, starting with the active segment and then
// checking frozen segments from newest to oldest. First match wins.
func (db *SegmentedDB) Get(key string) (string, bool, error) {
	db.mu.RLock()
	defer db.mu.RUnlock()

	// Check active segment first
	if offset, ok := db.active.index[key]; ok {
		_, value, err := readRecordFromFile(db.active.file, offset)
		if err != nil {
			return "", false, err
		}
		return value, true, nil
	}

	// Check frozen segments (newest first)
	for _, seg := range db.frozen {
		if offset, ok := seg.index[key]; ok {
			_, value, err := readRecordFromFile(seg.file, offset)
			if err != nil {
				return "", false, err
			}
			return value, true, nil
		}
	}

	return "", false, nil
}

// Compact merges all frozen segments into a single new segment,
// keeping only the most recent value for each key.
func (db *SegmentedDB) Compact() error {
	db.mu.Lock()
	defer db.mu.Unlock()

	if len(db.frozen) < 2 {
		return nil // nothing to compact
	}

	// Collect all key-value pairs. Process from oldest to newest
	// so that newer values overwrite older ones.
	merged := make(map[string]string)

	for i := len(db.frozen) - 1; i >= 0; i-- {
		seg := db.frozen[i]
		for key, offset := range seg.index {
			_, value, err := readRecordFromFile(seg.file, offset)
			if err != nil {
				return fmt.Errorf("read from segment %s: %w", seg.path, err)
			}
			merged[key] = value
		}
	}

	// Create new compacted segment
	newSeg, err := db.newSegment()
	if err != nil {
		return err
	}

	for key, value := range merged {
		offset, _, err := writeRecordToFile(newSeg.file, key, value)
		if err != nil {
			return err
		}
		newSeg.index[key] = offset
	}
	newSeg.frozen = true

	// Close and remove old frozen segments
	oldSegments := db.frozen
	db.frozen = []*Segment{newSeg}

	for _, seg := range oldSegments {
		seg.file.Close()
		os.Remove(seg.path)
	}

	return nil
}

// Stats returns current database statistics.
func (db *SegmentedDB) Stats() (activeSize int64, frozenCount int, totalKeys int) {
	db.mu.RLock()
	defer db.mu.RUnlock()

	totalKeys = len(db.active.index)
	for _, seg := range db.frozen {
		totalKeys += len(seg.index)
	}

	return db.active.size, len(db.frozen), totalKeys
}

// Close closes all segment files.
func (db *SegmentedDB) Close() error {
	db.active.file.Close()
	for _, seg := range db.frozen {
		seg.file.Close()
	}
	return nil
}

func main() {
	dir := "/tmp/segmented_db_example"
	os.RemoveAll(dir)

	db, err := NewSegmentedDB(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed: %v\n", err)
		os.Exit(1)
	}
	defer db.Close()
	defer os.RemoveAll(dir)

	fmt.Println("=== Segmented Database with Compaction ===")
	fmt.Println()

	// Write enough data to create multiple segments
	// Each record is about 20-30 bytes, so ~40K records per 1MB segment
	for i := 0; i < 200_000; i++ {
		key := fmt.Sprintf("user:%05d", i%1000) // 1000 unique keys, overwritten 200 times
		value := fmt.Sprintf(`{"name":"user%d","version":%d}`, i%1000, i)
		if err := db.Set(key, value); err != nil {
			fmt.Fprintf(os.Stderr, "Write error: %v\n", err)
			os.Exit(1)
		}
	}

	activeSize, frozenCount, totalKeys := db.Stats()
	fmt.Printf("Before compaction:\n")
	fmt.Printf("  Active segment size: %d bytes\n", activeSize)
	fmt.Printf("  Frozen segments:     %d\n", frozenCount)
	fmt.Printf("  Total index entries: %d\n", totalKeys)

	// Count files on disk
	entries, _ := os.ReadDir(dir)
	fmt.Printf("  Files on disk:       %d\n", len(entries))

	// Compact
	start := time.Now()
	if err := db.Compact(); err != nil {
		fmt.Fprintf(os.Stderr, "Compaction error: %v\n", err)
		os.Exit(1)
	}
	compactTime := time.Since(start)

	activeSize, frozenCount, totalKeys = db.Stats()
	fmt.Printf("\nAfter compaction (%v):\n", compactTime)
	fmt.Printf("  Active segment size: %d bytes\n", activeSize)
	fmt.Printf("  Frozen segments:     %d\n", frozenCount)
	fmt.Printf("  Total index entries: %d\n", totalKeys)

	entries, _ = os.ReadDir(dir)
	fmt.Printf("  Files on disk:       %d\n", len(entries))

	// Verify data integrity
	val, found, _ := db.Get("user:00042")
	fmt.Printf("\nuser:00042 = %q (found: %v)\n", val, found)
}
```

### What Real Databases Do

Bitcask (Riak's storage engine) uses exactly this approach: hash index + segment files + compaction. It provides:

- Extremely fast writes (always sequential)
- Very fast reads (one disk seek)
- Bounded disk usage (compaction reclaims space)

The limitation remains: all keys must fit in RAM. For most use cases with millions of keys, this is fine. For billions of keys, we need a different approach.

---

## 5. SSTable (Sorted String Table)

### Why Sorted Order Matters

An SSTable is like a segment file, but with one crucial difference: **the key-value pairs are sorted by key**. This seemingly small change gives us enormous advantages:

1. **Efficient merging.** Merging two SSTables is the same as merging two sorted lists -- O(n) time, no extra memory. This is the merge step from merge sort.
2. **Sparse index.** You do not need to index every key. If you know that "apple" is at offset 0 and "banana" is at offset 1000, then "avocado" must be somewhere between 0 and 1000. You can binary-search within that range.
3. **Block compression.** Since nearby keys are grouped on disk, you can compress blocks of records together, saving disk space and I/O bandwidth.

### Go Implementation

```go
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"io"
	"os"
	"sort"
)

// IndexEntry maps a key to its byte offset in the SSTable file.
type IndexEntry struct {
	Key    string
	Offset int64
}

// SSTable represents an immutable, sorted key-value file on disk.
// Once written, an SSTable is never modified -- this is key to the
// entire log-structured design.
type SSTable struct {
	path  string
	index []IndexEntry // sparse index -- not every key, just every Nth
}

// SSTableWriter builds an SSTable from sorted key-value pairs.
type SSTableWriter struct {
	file       *os.File
	writer     *bufio.Writer
	index      []IndexEntry
	offset     int64
	entryCount int
	indexEvery int // add index entry every N records
}

// NewSSTableWriter creates a writer for a new SSTable file.
func NewSSTableWriter(path string, indexEvery int) (*SSTableWriter, error) {
	file, err := os.Create(path)
	if err != nil {
		return nil, err
	}
	return &SSTableWriter{
		file:       file,
		writer:     bufio.NewWriter(file),
		indexEvery: indexEvery,
	}, nil
}

// Write adds a key-value pair to the SSTable.
// IMPORTANT: keys must be written in sorted order.
func (w *SSTableWriter) Write(key, value string) error {
	// Add to sparse index every N entries
	if w.entryCount%w.indexEvery == 0 {
		w.index = append(w.index, IndexEntry{Key: key, Offset: w.offset})
	}

	// Write record: [keyLen:4][valueLen:4][key][value]
	keyBytes := []byte(key)
	valueBytes := []byte(value)

	if err := binary.Write(w.writer, binary.LittleEndian, uint32(len(keyBytes))); err != nil {
		return err
	}
	if err := binary.Write(w.writer, binary.LittleEndian, uint32(len(valueBytes))); err != nil {
		return err
	}
	if _, err := w.writer.Write(keyBytes); err != nil {
		return err
	}
	if _, err := w.writer.Write(valueBytes); err != nil {
		return err
	}

	w.offset += int64(8 + len(keyBytes) + len(valueBytes))
	w.entryCount++
	return nil
}

// Finish flushes all data and returns the SSTable.
func (w *SSTableWriter) Finish() (*SSTable, error) {
	if err := w.writer.Flush(); err != nil {
		return nil, err
	}
	if err := w.file.Sync(); err != nil {
		return nil, err
	}
	if err := w.file.Close(); err != nil {
		return nil, err
	}

	return &SSTable{
		path:  w.file.Name(),
		index: w.index,
	}, nil
}

// readRecordAt reads a key-value pair from the given offset in the file.
func readRecordAt(f *os.File, offset int64) (string, string, int64, error) {
	if _, err := f.Seek(offset, io.SeekStart); err != nil {
		return "", "", 0, err
	}

	var keyLen, valueLen uint32
	if err := binary.Read(f, binary.LittleEndian, &keyLen); err != nil {
		return "", "", 0, err
	}
	if err := binary.Read(f, binary.LittleEndian, &valueLen); err != nil {
		return "", "", 0, err
	}

	keyBuf := make([]byte, keyLen)
	if _, err := io.ReadFull(f, keyBuf); err != nil {
		return "", "", 0, err
	}

	valueBuf := make([]byte, valueLen)
	if _, err := io.ReadFull(f, valueBuf); err != nil {
		return "", "", 0, err
	}

	nextOffset := offset + int64(8+keyLen+valueLen)
	return string(keyBuf), string(valueBuf), nextOffset, nil
}

// Get searches for a key in the SSTable using the sparse index.
// 1. Binary search the sparse index to find the range.
// 2. Scan sequentially within that range.
func (sst *SSTable) Get(key string) (string, bool, error) {
	if len(sst.index) == 0 {
		return "", false, nil
	}

	// Binary search: find the largest index entry <= key
	pos := sort.Search(len(sst.index), func(i int) bool {
		return sst.index[i].Key > key
	}) - 1

	if pos < 0 {
		pos = 0
	}

	startOffset := sst.index[pos].Offset

	// Determine end of scan range
	var endOffset int64 = -1 // -1 means scan to EOF
	if pos+1 < len(sst.index) {
		endOffset = sst.index[pos+1].Offset
	}

	// Open file and scan the range
	f, err := os.Open(sst.path)
	if err != nil {
		return "", false, err
	}
	defer f.Close()

	offset := startOffset
	for {
		if endOffset >= 0 && offset >= endOffset {
			break
		}

		k, v, nextOffset, err := readRecordAt(f, offset)
		if err != nil {
			if err == io.EOF {
				break
			}
			return "", false, err
		}

		if k == key {
			return v, true, nil
		}
		if k > key {
			break // past the key in sorted order
		}

		offset = nextOffset
	}

	return "", false, nil
}

// Scan returns all key-value pairs in a range [startKey, endKey].
// This is efficient because the SSTable is sorted.
func (sst *SSTable) Scan(startKey, endKey string) ([]KeyValue, error) {
	if len(sst.index) == 0 {
		return nil, nil
	}

	// Find starting position using sparse index
	pos := sort.Search(len(sst.index), func(i int) bool {
		return sst.index[i].Key > startKey
	}) - 1
	if pos < 0 {
		pos = 0
	}

	f, err := os.Open(sst.path)
	if err != nil {
		return nil, err
	}
	defer f.Close()

	var results []KeyValue
	offset := sst.index[pos].Offset

	for {
		k, v, nextOffset, err := readRecordAt(f, offset)
		if err != nil {
			if err == io.EOF {
				break
			}
			return nil, err
		}

		if k > endKey {
			break
		}
		if k >= startKey {
			results = append(results, KeyValue{Key: k, Value: v})
		}

		offset = nextOffset
	}

	return results, nil
}

// KeyValue holds a single key-value pair.
type KeyValue struct {
	Key   string
	Value string
}

// MergeSSTables merges multiple SSTables into one, keeping only the
// latest value for each key. This is like the merge step of merge sort.
// Inputs must be ordered from newest to oldest.
func MergeSSTables(inputs []*SSTable, outputPath string, indexEvery int) (*SSTable, error) {
	type scanState struct {
		file   *os.File
		offset int64
		key    string
		value  string
		done   bool
	}

	// Open all input files
	states := make([]*scanState, len(inputs))
	for i, sst := range inputs {
		f, err := os.Open(sst.path)
		if err != nil {
			return nil, err
		}
		defer f.Close()

		states[i] = &scanState{file: f}

		// Read first record
		k, v, nextOff, err := readRecordAt(f, 0)
		if err != nil {
			states[i].done = true
		} else {
			states[i].key = k
			states[i].value = v
			states[i].offset = nextOff
		}
	}

	writer, err := NewSSTableWriter(outputPath, indexEvery)
	if err != nil {
		return nil, err
	}

	seen := make(map[string]bool)

	// Multi-way merge: always pick the smallest key
	for {
		// Find the state with the smallest key
		minIdx := -1
		for i, s := range states {
			if s.done {
				continue
			}
			if minIdx == -1 || s.key < states[minIdx].key {
				minIdx = i
			} else if s.key == states[minIdx].key {
				// Same key in multiple files -- advance the older one
				// (higher index = older, since inputs are newest-first)
				if i > minIdx {
					// Advance the older state
					k, v, nextOff, err := readRecordAt(states[i].file, states[i].offset)
					if err != nil {
						states[i].done = true
					} else {
						states[i].key = k
						states[i].value = v
						states[i].offset = nextOff
					}
				}
			}
		}

		if minIdx == -1 {
			break // all inputs exhausted
		}

		s := states[minIdx]

		// Only write the first occurrence (newest) of each key
		if !seen[s.key] {
			seen[s.key] = true
			if err := writer.Write(s.key, s.value); err != nil {
				return nil, err
			}
		}

		// Advance this state
		k, v, nextOff, err := readRecordAt(s.file, s.offset)
		if err != nil {
			s.done = true
		} else {
			s.key = k
			s.value = v
			s.offset = nextOff
		}
	}

	return writer.Finish()
}

func main() {
	fmt.Println("=== SSTable (Sorted String Table) ===")
	fmt.Println()

	// Create an SSTable with sorted data
	// In a real LSM tree, this data comes from a sorted memtable
	data := []KeyValue{
		{"apple", "red fruit"},
		{"avocado", "green fruit"},
		{"banana", "yellow fruit"},
		{"cherry", "small red fruit"},
		{"date", "sweet brown fruit"},
		{"elderberry", "dark purple berry"},
		{"fig", "soft sweet fruit"},
		{"grape", "vine fruit"},
		{"honeydew", "green melon"},
		{"kiwi", "brown fuzzy fruit"},
		{"lemon", "sour yellow citrus"},
		{"mango", "tropical fruit"},
		{"nectarine", "smooth peach"},
		{"orange", "citrus fruit"},
		{"papaya", "tropical melon-like"},
		{"quince", "hard yellow fruit"},
	}

	// Write SSTable with sparse index every 4 records
	sstPath := "/tmp/fruits.sst"
	defer os.Remove(sstPath)

	writer, err := NewSSTableWriter(sstPath, 4)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed: %v\n", err)
		os.Exit(1)
	}

	for _, kv := range data {
		if err := writer.Write(kv.Key, kv.Value); err != nil {
			fmt.Fprintf(os.Stderr, "Write failed: %v\n", err)
			os.Exit(1)
		}
	}

	sst, err := writer.Finish()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Finish failed: %v\n", err)
		os.Exit(1)
	}

	// Show the sparse index
	fmt.Println("Sparse index entries:")
	for _, entry := range sst.index {
		fmt.Printf("  %q -> offset %d\n", entry.Key, entry.Offset)
	}
	fmt.Println()

	// Point lookups
	for _, key := range []string{"banana", "grape", "mango", "zebra"} {
		val, found, err := sst.Get(key)
		if err != nil {
			fmt.Printf("  Error looking up %q: %v\n", key, err)
		} else if found {
			fmt.Printf("  %q = %q\n", key, val)
		} else {
			fmt.Printf("  %q = (not found)\n", key)
		}
	}

	// Range scan
	fmt.Println()
	fmt.Println("Range scan [cherry, honeydew]:")
	results, err := sst.Scan("cherry", "honeydew")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Scan failed: %v\n", err)
		os.Exit(1)
	}
	for _, kv := range results {
		fmt.Printf("  %q = %q\n", kv.Key, kv.Value)
	}
}
```

### Why SSTables Matter

SSTables are the foundation of LSM trees (used by LevelDB, RocksDB, Cassandra, HBase, and many more). The sorted order enables:

- **O(log n) lookups** using the sparse index instead of O(n)
- **Efficient range queries** -- just scan a contiguous region
- **O(n) merge** of multiple SSTables -- critical for compaction
- **Compression** of sorted blocks

---

## 6. Memtable with Red-Black Tree

### The Missing Piece

SSTables require data to be sorted, but writes arrive in random order. We cannot sort the data on disk (that would require rewriting the entire file). Instead, we sort it in memory using a balanced binary search tree.

A **memtable** is an in-memory sorted data structure. Writes go into the memtable. When the memtable reaches a threshold size, it is flushed to disk as an SSTable. The most common implementation uses a red-black tree (or a skip list), but any balanced BST works.

Go's standard library does not include a red-black tree, but we can build one or use a simpler sorted structure. For clarity, we will implement a skip list -- it is simpler than a red-black tree and has the same O(log n) complexity.

### Go Implementation

```go
package main

import (
	"fmt"
	"math/rand"
	"strings"
	"sync"
)

const maxLevel = 16

// skipListNode is a node in the skip list.
type skipListNode struct {
	key     string
	value   string
	forward []*skipListNode // forward[i] points to the next node at level i
}

// SkipList is a probabilistic data structure that provides O(log n)
// search, insert, and delete. It is used as the memtable in LSM trees.
// CockroachDB's Pebble storage engine uses a skip list for its memtable.
type SkipList struct {
	header *skipListNode
	level  int // current max level in use
	size   int // number of entries
	bytes  int // approximate memory usage in bytes
}

// NewSkipList creates an empty skip list.
func NewSkipList() *SkipList {
	header := &skipListNode{
		forward: make([]*skipListNode, maxLevel),
	}
	return &SkipList{
		header: header,
		level:  0,
	}
}

// randomLevel generates a random level for a new node.
// Each level has a 50% chance of being promoted to the next level.
// This gives us O(log n) expected height.
func randomLevel() int {
	level := 1
	for level < maxLevel && rand.Float64() < 0.5 {
		level++
	}
	return level
}

// Set inserts or updates a key-value pair. O(log n) expected time.
func (sl *SkipList) Set(key, value string) {
	// update[i] will hold the node whose forward[i] needs to be updated
	update := make([]*skipListNode, maxLevel)
	current := sl.header

	// Search from top level down
	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < key {
			current = current.forward[i]
		}
		update[i] = current
	}

	// Check if key already exists at level 0
	next := current.forward[0]
	if next != nil && next.key == key {
		// Update existing key
		sl.bytes -= len(next.value)
		sl.bytes += len(value)
		next.value = value
		return
	}

	// Insert new node
	newLevel := randomLevel()
	if newLevel > sl.level {
		for i := sl.level; i < newLevel; i++ {
			update[i] = sl.header
		}
		sl.level = newLevel
	}

	newNode := &skipListNode{
		key:     key,
		value:   value,
		forward: make([]*skipListNode, newLevel),
	}

	for i := 0; i < newLevel; i++ {
		newNode.forward[i] = update[i].forward[i]
		update[i].forward[i] = newNode
	}

	sl.size++
	sl.bytes += len(key) + len(value) + 64 // rough estimate including pointers
}

// Get retrieves a value by key. O(log n) expected time.
func (sl *SkipList) Get(key string) (string, bool) {
	current := sl.header

	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < key {
			current = current.forward[i]
		}
	}

	next := current.forward[0]
	if next != nil && next.key == key {
		return next.value, true
	}
	return "", false
}

// Delete removes a key from the skip list.
func (sl *SkipList) Delete(key string) bool {
	update := make([]*skipListNode, maxLevel)
	current := sl.header

	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < key {
			current = current.forward[i]
		}
		update[i] = current
	}

	target := current.forward[0]
	if target == nil || target.key != key {
		return false
	}

	for i := 0; i < sl.level; i++ {
		if update[i].forward[i] != target {
			break
		}
		update[i].forward[i] = target.forward[i]
	}

	// Reduce level if necessary
	for sl.level > 0 && sl.header.forward[sl.level-1] == nil {
		sl.level--
	}

	sl.size--
	sl.bytes -= len(target.key) + len(target.value) + 64
	return true
}

// Range returns all key-value pairs in [startKey, endKey] in sorted order.
func (sl *SkipList) Range(startKey, endKey string) []KeyValue {
	var results []KeyValue

	// Find the starting point
	current := sl.header
	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < startKey {
			current = current.forward[i]
		}
	}

	// Scan forward at level 0
	current = current.forward[0]
	for current != nil && current.key <= endKey {
		results = append(results, KeyValue{Key: current.key, Value: current.value})
		current = current.forward[0]
	}

	return results
}

// ForEach iterates over all entries in sorted order.
func (sl *SkipList) ForEach(fn func(key, value string)) {
	current := sl.header.forward[0]
	for current != nil {
		fn(current.key, current.value)
		current = current.forward[0]
	}
}

// KeyValue holds a key-value pair.
type KeyValue struct {
	Key   string
	Value string
}

// Memtable wraps a SkipList with a size threshold and thread safety.
type Memtable struct {
	sl        *SkipList
	mu        sync.RWMutex
	threshold int // flush to SSTable when bytes exceed this
}

// NewMemtable creates a memtable with the given size threshold in bytes.
func NewMemtable(threshold int) *Memtable {
	return &Memtable{
		sl:        NewSkipList(),
		threshold: threshold,
	}
}

// Set inserts or updates a key-value pair.
func (m *Memtable) Set(key, value string) bool {
	m.mu.Lock()
	defer m.mu.Unlock()
	m.sl.Set(key, value)
	return m.sl.bytes >= m.threshold // true = memtable is full, should flush
}

// Get retrieves a value by key.
func (m *Memtable) Get(key string) (string, bool) {
	m.mu.RLock()
	defer m.mu.RUnlock()
	return m.sl.Get(key)
}

// Size returns the approximate memory usage in bytes.
func (m *Memtable) Size() int {
	m.mu.RLock()
	defer m.mu.RUnlock()
	return m.sl.bytes
}

// Count returns the number of entries.
func (m *Memtable) Count() int {
	m.mu.RLock()
	defer m.mu.RUnlock()
	return m.sl.size
}

// Entries returns all entries in sorted order.
func (m *Memtable) Entries() []KeyValue {
	m.mu.RLock()
	defer m.mu.RUnlock()

	entries := make([]KeyValue, 0, m.sl.size)
	m.sl.ForEach(func(key, value string) {
		entries = append(entries, KeyValue{Key: key, Value: value})
	})
	return entries
}

func main() {
	fmt.Println("=== Memtable with Skip List ===")
	fmt.Println()

	// Demonstrate the skip list
	sl := NewSkipList()

	// Insert in random order -- the skip list keeps them sorted
	words := []string{"fig", "apple", "cherry", "banana", "elderberry", "date", "grape"}
	for _, w := range words {
		sl.Set(w, "fruit: "+w)
	}

	fmt.Println("All entries (sorted):")
	sl.ForEach(func(key, value string) {
		fmt.Printf("  %s = %s\n", key, value)
	})

	fmt.Println()
	fmt.Println("Range [cherry, fig]:")
	for _, kv := range sl.Range("cherry", "fig") {
		fmt.Printf("  %s = %s\n", kv.Key, kv.Value)
	}

	// Update a value
	sl.Set("banana", "yellow fruit")
	val, _ := sl.Get("banana")
	fmt.Printf("\nUpdated banana = %s\n", val)

	// Delete
	sl.Delete("date")
	_, found := sl.Get("date")
	fmt.Printf("date after delete: found=%v\n", found)

	// Demonstrate memtable with threshold
	fmt.Println()
	fmt.Println("=== Memtable with 1KB threshold ===")
	fmt.Println()

	mem := NewMemtable(1024) // 1KB threshold
	for i := 0; i < 100; i++ {
		key := fmt.Sprintf("key-%03d", i)
		value := fmt.Sprintf("value-%s", strings.Repeat("x", 10))
		isFull := mem.Set(key, value)
		if isFull {
			fmt.Printf("Memtable full at %d entries (%d bytes) -- should flush to SSTable\n",
				mem.Count(), mem.Size())
			break
		}
	}

	fmt.Printf("\nFinal memtable: %d entries, %d bytes\n", mem.Count(), mem.Size())

	// Show that entries come out sorted
	entries := mem.Entries()
	fmt.Printf("First 5 entries: ")
	for i := 0; i < 5 && i < len(entries); i++ {
		fmt.Printf("[%s] ", entries[i].Key)
	}
	fmt.Println()
}
```

### Node.js Comparison

In Node.js, you would typically use a sorted array or a third-party balanced BST library:

```javascript
// Node.js -- simple sorted map as memtable
class Memtable {
  constructor(threshold) {
    this.entries = new Map(); // NOT sorted -- Map preserves insertion order
    this.threshold = threshold;
    this.bytes = 0;
  }

  set(key, value) {
    if (this.entries.has(key)) {
      this.bytes -= this.entries.get(key).length;
    }
    this.entries.set(key, value);
    this.bytes += key.length + value.length;
    return this.bytes >= this.threshold;
  }

  get(key) {
    return this.entries.get(key);
  }

  // To flush as SSTable, we must sort the keys -- O(n log n)
  sortedEntries() {
    return [...this.entries.entries()]
      .sort(([a], [b]) => a.localeCompare(b));
  }
}
```

The critical difference: JavaScript's `Map` is unordered (or insertion-ordered), so flushing to a sorted SSTable requires an O(n log n) sort. Our Go skip list maintains sorted order at all times, making the flush a simple O(n) traversal.

---

## 7. LSM Tree (Log-Structured Merge Tree)

### Putting It All Together

An LSM tree combines everything we have built so far:

1. **Memtable** -- An in-memory sorted structure (our skip list) that receives all writes
2. **Write-Ahead Log (WAL)** -- A sequential log file for durability (if the process crashes before the memtable is flushed, we can recover from the WAL)
3. **SSTables** -- Immutable sorted files on disk, organized in levels
4. **Compaction** -- Background process that merges SSTables to reclaim space and improve read performance

```
Write path:
  1. Write to WAL (for durability)
  2. Write to memtable (in-memory skip list)
  3. When memtable is full, flush to Level 0 SSTable
  4. Background: compact Level 0 into Level 1, Level 1 into Level 2, etc.

Read path:
  1. Check memtable (most recent data)
  2. Check Level 0 SSTables (newest on-disk data)
  3. Check Level 1, Level 2, etc. (progressively older data)
  4. If not found in any level, the key does not exist

        ┌──────────────┐
        │   Memtable   │  ← All writes go here first
        │  (skip list) │
        └──────┬───────┘
               │ flush when full
        ┌──────▼───────┐
        │   Level 0    │  ← Recently flushed SSTables (may overlap)
        │  SST SST SST │
        └──────┬───────┘
               │ compact (merge + sort)
        ┌──────▼───────┐
        │   Level 1    │  ← Merged, non-overlapping SSTables
        │  SST SST SST │
        └──────┬───────┘
               │ compact
        ┌──────▼───────┐
        │   Level 2    │  ← Even larger, non-overlapping SSTables
        │  SST SST ... │
        └──────────────┘
```

### Go Implementation

```go
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"io"
	"math/rand"
	"os"
	"path/filepath"
	"sort"
	"sync"
	"sync/atomic"
	"time"
)

// ----- Skip List (Memtable backing structure) -----

const skipMaxLevel = 16

type slNode struct {
	key     string
	value   string
	deleted bool // tombstone marker
	forward []*slNode
}

type skipList struct {
	header *slNode
	level  int
	size   int
	bytes  int
}

func newSkipList() *skipList {
	return &skipList{
		header: &slNode{forward: make([]*slNode, skipMaxLevel)},
	}
}

func slRandomLevel() int {
	lvl := 1
	for lvl < skipMaxLevel && rand.Float64() < 0.5 {
		lvl++
	}
	return lvl
}

func (sl *skipList) set(key, value string, deleted bool) {
	update := make([]*slNode, skipMaxLevel)
	cur := sl.header
	for i := sl.level - 1; i >= 0; i-- {
		for cur.forward[i] != nil && cur.forward[i].key < key {
			cur = cur.forward[i]
		}
		update[i] = cur
	}

	next := cur.forward[0]
	if next != nil && next.key == key {
		sl.bytes -= len(next.value)
		next.value = value
		next.deleted = deleted
		sl.bytes += len(value)
		return
	}

	newLvl := slRandomLevel()
	if newLvl > sl.level {
		for i := sl.level; i < newLvl; i++ {
			update[i] = sl.header
		}
		sl.level = newLvl
	}

	node := &slNode{
		key:     key,
		value:   value,
		deleted: deleted,
		forward: make([]*slNode, newLvl),
	}
	for i := 0; i < newLvl; i++ {
		node.forward[i] = update[i].forward[i]
		update[i].forward[i] = node
	}
	sl.size++
	sl.bytes += len(key) + len(value) + 80
}

func (sl *skipList) get(key string) (string, bool, bool) {
	cur := sl.header
	for i := sl.level - 1; i >= 0; i-- {
		for cur.forward[i] != nil && cur.forward[i].key < key {
			cur = cur.forward[i]
		}
	}
	next := cur.forward[0]
	if next != nil && next.key == key {
		return next.value, next.deleted, true
	}
	return "", false, false
}

type kvEntry struct {
	key     string
	value   string
	deleted bool
}

func (sl *skipList) entries() []kvEntry {
	result := make([]kvEntry, 0, sl.size)
	cur := sl.header.forward[0]
	for cur != nil {
		result = append(result, kvEntry{key: cur.key, value: cur.value, deleted: cur.deleted})
		cur = cur.forward[0]
	}
	return result
}

// ----- SSTable -----

type sstIndex struct {
	key    string
	offset int64
}

type sstable struct {
	path  string
	index []sstIndex
	size  int64
}

func writeSSTable(path string, entries []kvEntry, indexEvery int) (*sstable, error) {
	f, err := os.Create(path)
	if err != nil {
		return nil, err
	}
	w := bufio.NewWriter(f)

	var idx []sstIndex
	var offset int64

	for i, e := range entries {
		if i%indexEvery == 0 {
			idx = append(idx, sstIndex{key: e.key, offset: offset})
		}

		keyBytes := []byte(e.key)
		valBytes := []byte(e.value)

		// Flag byte: 0 = normal, 1 = tombstone
		flag := byte(0)
		if e.deleted {
			flag = 1
		}

		if err := w.WriteByte(flag); err != nil {
			f.Close()
			return nil, err
		}
		if err := binary.Write(w, binary.LittleEndian, uint32(len(keyBytes))); err != nil {
			f.Close()
			return nil, err
		}
		if err := binary.Write(w, binary.LittleEndian, uint32(len(valBytes))); err != nil {
			f.Close()
			return nil, err
		}
		if _, err := w.Write(keyBytes); err != nil {
			f.Close()
			return nil, err
		}
		if _, err := w.Write(valBytes); err != nil {
			f.Close()
			return nil, err
		}

		offset += 1 + 4 + 4 + int64(len(keyBytes)) + int64(len(valBytes))
	}

	if err := w.Flush(); err != nil {
		f.Close()
		return nil, err
	}
	if err := f.Sync(); err != nil {
		f.Close()
		return nil, err
	}
	f.Close()

	return &sstable{path: path, index: idx, size: offset}, nil
}

func readSSRecord(f *os.File, offset int64) (kvEntry, int64, error) {
	if _, err := f.Seek(offset, io.SeekStart); err != nil {
		return kvEntry{}, 0, err
	}

	flagBuf := make([]byte, 1)
	if _, err := io.ReadFull(f, flagBuf); err != nil {
		return kvEntry{}, 0, err
	}

	var keyLen, valLen uint32
	if err := binary.Read(f, binary.LittleEndian, &keyLen); err != nil {
		return kvEntry{}, 0, err
	}
	if err := binary.Read(f, binary.LittleEndian, &valLen); err != nil {
		return kvEntry{}, 0, err
	}

	keyBuf := make([]byte, keyLen)
	if _, err := io.ReadFull(f, keyBuf); err != nil {
		return kvEntry{}, 0, err
	}

	valBuf := make([]byte, valLen)
	if _, err := io.ReadFull(f, valBuf); err != nil {
		return kvEntry{}, 0, err
	}

	next := offset + 1 + 4 + 4 + int64(keyLen) + int64(valLen)
	return kvEntry{
		key:     string(keyBuf),
		value:   string(valBuf),
		deleted: flagBuf[0] == 1,
	}, next, nil
}

func (sst *sstable) get(key string) (string, bool, bool, error) {
	if len(sst.index) == 0 {
		return "", false, false, nil
	}

	pos := sort.Search(len(sst.index), func(i int) bool {
		return sst.index[i].key > key
	}) - 1
	if pos < 0 {
		pos = 0
	}

	var endOffset int64 = sst.size
	if pos+1 < len(sst.index) {
		endOffset = sst.index[pos+1].offset
	}

	f, err := os.Open(sst.path)
	if err != nil {
		return "", false, false, err
	}
	defer f.Close()

	offset := sst.index[pos].offset
	for offset < endOffset {
		entry, next, err := readSSRecord(f, offset)
		if err != nil {
			break
		}
		if entry.key == key {
			return entry.value, entry.deleted, true, nil
		}
		if entry.key > key {
			break
		}
		offset = next
	}

	return "", false, false, nil
}

// ----- Write-Ahead Log -----

type wal struct {
	file *os.File
	w    *bufio.Writer
}

func newWAL(path string) (*wal, error) {
	f, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR|os.O_APPEND, 0644)
	if err != nil {
		return nil, err
	}
	return &wal{file: f, w: bufio.NewWriter(f)}, nil
}

func (w *wal) append(key, value string, deleted bool) error {
	flag := byte(0)
	if deleted {
		flag = 1
	}
	if err := w.w.WriteByte(flag); err != nil {
		return err
	}
	if err := binary.Write(w.w, binary.LittleEndian, uint32(len(key))); err != nil {
		return err
	}
	if err := binary.Write(w.w, binary.LittleEndian, uint32(len(value))); err != nil {
		return err
	}
	if _, err := w.w.WriteString(key); err != nil {
		return err
	}
	if _, err := w.w.WriteString(value); err != nil {
		return err
	}
	return w.w.Flush()
}

func (w *wal) sync() error {
	if err := w.w.Flush(); err != nil {
		return err
	}
	return w.file.Sync()
}

func (w *wal) close() error {
	w.w.Flush()
	return w.file.Close()
}

func recoverWAL(path string) ([]kvEntry, error) {
	f, err := os.Open(path)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, nil
		}
		return nil, err
	}
	defer f.Close()

	r := bufio.NewReader(f)
	var entries []kvEntry

	for {
		flagBuf := make([]byte, 1)
		if _, err := io.ReadFull(r, flagBuf); err != nil {
			break
		}

		var keyLen, valLen uint32
		if err := binary.Read(r, binary.LittleEndian, &keyLen); err != nil {
			break
		}
		if err := binary.Read(r, binary.LittleEndian, &valLen); err != nil {
			break
		}

		keyBuf := make([]byte, keyLen)
		if _, err := io.ReadFull(r, keyBuf); err != nil {
			break
		}
		valBuf := make([]byte, valLen)
		if _, err := io.ReadFull(r, valBuf); err != nil {
			break
		}

		entries = append(entries, kvEntry{
			key:     string(keyBuf),
			value:   string(valBuf),
			deleted: flagBuf[0] == 1,
		})
	}

	return entries, nil
}

// ----- LSM Tree -----

// LSMTree is a Log-Structured Merge Tree. This is the storage engine
// used by LevelDB, RocksDB, Cassandra, and many other databases.
type LSMTree struct {
	dir            string
	memtable       *skipList
	memtableWAL    *wal
	immutable      []*skipList // memtables being flushed
	levels         [][]*sstable
	mu             sync.RWMutex
	sstSeq         int64 // atomic counter for SSTable filenames
	memThreshold   int   // flush memtable when it exceeds this many bytes
	compactTrigger int   // compact a level when it has this many SSTables
}

// NewLSMTree creates a new LSM tree in the given directory.
func NewLSMTree(dir string, memThreshold int) (*LSMTree, error) {
	if err := os.MkdirAll(dir, 0755); err != nil {
		return nil, err
	}

	walPath := filepath.Join(dir, "wal.log")

	// Recover from WAL if it exists
	recovered, err := recoverWAL(walPath)
	if err != nil {
		return nil, fmt.Errorf("recover WAL: %w", err)
	}

	w, err := newWAL(walPath)
	if err != nil {
		return nil, fmt.Errorf("create WAL: %w", err)
	}

	mem := newSkipList()
	for _, e := range recovered {
		mem.set(e.key, e.value, e.deleted)
	}

	lsm := &LSMTree{
		dir:            dir,
		memtable:       mem,
		memtableWAL:    w,
		levels:         make([][]*sstable, 4), // 4 levels
		memThreshold:   memThreshold,
		compactTrigger: 4,
	}

	return lsm, nil
}

// Set writes a key-value pair to the LSM tree.
func (lsm *LSMTree) Set(key, value string) error {
	lsm.mu.Lock()
	defer lsm.mu.Unlock()

	// Step 1: Write to WAL for durability
	if err := lsm.memtableWAL.append(key, value, false); err != nil {
		return fmt.Errorf("WAL append: %w", err)
	}

	// Step 2: Write to memtable
	lsm.memtable.set(key, value, false)

	// Step 3: Check if memtable needs flushing
	if lsm.memtable.bytes >= lsm.memThreshold {
		if err := lsm.flushMemtable(); err != nil {
			return fmt.Errorf("flush memtable: %w", err)
		}
	}

	return nil
}

// Delete writes a tombstone marker for the key.
func (lsm *LSMTree) Delete(key string) error {
	lsm.mu.Lock()
	defer lsm.mu.Unlock()

	if err := lsm.memtableWAL.append(key, "", true); err != nil {
		return fmt.Errorf("WAL append: %w", err)
	}

	lsm.memtable.set(key, "", true)

	if lsm.memtable.bytes >= lsm.memThreshold {
		if err := lsm.flushMemtable(); err != nil {
			return fmt.Errorf("flush memtable: %w", err)
		}
	}

	return nil
}

// Get reads a value by key.
// Search order: memtable -> immutable memtables -> Level 0 -> Level 1 -> ...
func (lsm *LSMTree) Get(key string) (string, bool, error) {
	lsm.mu.RLock()
	defer lsm.mu.RUnlock()

	// Check active memtable
	if val, deleted, found := lsm.memtable.get(key); found {
		if deleted {
			return "", false, nil // tombstone
		}
		return val, true, nil
	}

	// Check immutable memtables (newest first)
	for _, imm := range lsm.immutable {
		if val, deleted, found := imm.get(key); found {
			if deleted {
				return "", false, nil
			}
			return val, true, nil
		}
	}

	// Check SSTables level by level
	for _, level := range lsm.levels {
		// Search from newest to oldest within each level
		for i := len(level) - 1; i >= 0; i-- {
			val, deleted, found, err := level[i].get(key)
			if err != nil {
				return "", false, err
			}
			if found {
				if deleted {
					return "", false, nil
				}
				return val, true, nil
			}
		}
	}

	return "", false, nil
}

// flushMemtable writes the current memtable to disk as an SSTable.
// Must be called with lsm.mu held.
func (lsm *LSMTree) flushMemtable() error {
	entries := lsm.memtable.entries()
	if len(entries) == 0 {
		return nil
	}

	// Create SSTable
	seq := atomic.AddInt64(&lsm.sstSeq, 1)
	path := filepath.Join(lsm.dir, fmt.Sprintf("sst-L0-%06d.dat", seq))

	sst, err := writeSSTable(path, entries, 16)
	if err != nil {
		return err
	}

	// Add to Level 0
	lsm.levels[0] = append(lsm.levels[0], sst)

	// Reset memtable and WAL
	lsm.memtable = newSkipList()
	lsm.memtableWAL.close()
	os.Remove(filepath.Join(lsm.dir, "wal.log"))
	w, err := newWAL(filepath.Join(lsm.dir, "wal.log"))
	if err != nil {
		return err
	}
	lsm.memtableWAL = w

	// Trigger compaction if Level 0 has too many SSTables
	if len(lsm.levels[0]) >= lsm.compactTrigger {
		return lsm.compactLevel(0)
	}

	return nil
}

// compactLevel merges all SSTables from one level into the next level.
func (lsm *LSMTree) compactLevel(level int) error {
	if level+1 >= len(lsm.levels) {
		return nil
	}

	// Collect all entries from this level's SSTables
	allEntries := make(map[string]kvEntry)

	// First, add entries from the next level (older data)
	for _, sst := range lsm.levels[level+1] {
		f, err := os.Open(sst.path)
		if err != nil {
			return err
		}
		var offset int64
		for offset < sst.size {
			entry, next, err := readSSRecord(f, offset)
			if err != nil {
				break
			}
			allEntries[entry.key] = entry
			offset = next
		}
		f.Close()
	}

	// Then, add entries from the current level (newer data overwrites)
	for _, sst := range lsm.levels[level] {
		f, err := os.Open(sst.path)
		if err != nil {
			return err
		}
		var offset int64
		for offset < sst.size {
			entry, next, err := readSSRecord(f, offset)
			if err != nil {
				break
			}
			allEntries[entry.key] = entry
			offset = next
		}
		f.Close()
	}

	// Sort entries by key
	sorted := make([]kvEntry, 0, len(allEntries))
	for _, e := range allEntries {
		// Drop tombstones at the lowest level
		if e.deleted && level+1 == len(lsm.levels)-1 {
			continue
		}
		sorted = append(sorted, e)
	}
	sort.Slice(sorted, func(i, j int) bool {
		return sorted[i].key < sorted[j].key
	})

	// Write new SSTable at next level
	seq := atomic.AddInt64(&lsm.sstSeq, 1)
	path := filepath.Join(lsm.dir, fmt.Sprintf("sst-L%d-%06d.dat", level+1, seq))
	newSST, err := writeSSTable(path, sorted, 16)
	if err != nil {
		return err
	}

	// Remove old SSTables
	for _, sst := range lsm.levels[level] {
		os.Remove(sst.path)
	}
	for _, sst := range lsm.levels[level+1] {
		os.Remove(sst.path)
	}

	lsm.levels[level] = nil
	lsm.levels[level+1] = []*sstable{newSST}

	return nil
}

// Stats returns database statistics.
func (lsm *LSMTree) Stats() map[string]interface{} {
	lsm.mu.RLock()
	defer lsm.mu.RUnlock()

	stats := map[string]interface{}{
		"memtable_size":    lsm.memtable.bytes,
		"memtable_entries": lsm.memtable.size,
	}
	for i, level := range lsm.levels {
		stats[fmt.Sprintf("level_%d_sstables", i)] = len(level)
	}
	return stats
}

// Close flushes the memtable and closes all files.
func (lsm *LSMTree) Close() error {
	lsm.mu.Lock()
	defer lsm.mu.Unlock()

	// Flush remaining memtable
	if lsm.memtable.size > 0 {
		if err := lsm.flushMemtable(); err != nil {
			return err
		}
	}

	lsm.memtableWAL.close()
	return nil
}

func main() {
	dir := "/tmp/lsm_tree_example"
	os.RemoveAll(dir)

	fmt.Println("=== LSM Tree (Log-Structured Merge Tree) ===")
	fmt.Println()

	// Create LSM tree with 4KB memtable threshold
	lsm, err := NewLSMTree(dir, 4096)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed: %v\n", err)
		os.Exit(1)
	}
	defer os.RemoveAll(dir)

	// Write data -- this will trigger multiple memtable flushes
	start := time.Now()
	for i := 0; i < 10_000; i++ {
		key := fmt.Sprintf("key-%06d", i)
		value := fmt.Sprintf(`{"id":%d,"data":"value-%d"}`, i, i)
		if err := lsm.Set(key, value); err != nil {
			fmt.Fprintf(os.Stderr, "Set error: %v\n", err)
			os.Exit(1)
		}
	}
	writeTime := time.Since(start)

	fmt.Printf("Wrote 10,000 records in %v\n", writeTime)
	fmt.Printf("Stats: %v\n", lsm.Stats())

	// Read data
	start = time.Now()
	for i := 0; i < 1000; i++ {
		key := fmt.Sprintf("key-%06d", rand.Intn(10_000))
		_, _, err := lsm.Get(key)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Get error: %v\n", err)
		}
	}
	readTime := time.Since(start)

	fmt.Printf("1,000 random reads in %v\n", readTime)

	// Verify specific values
	val, found, _ := lsm.Get("key-000042")
	fmt.Printf("\nkey-000042 = %q (found: %v)\n", val, found)

	// Test updates
	lsm.Set("key-000042", `{"id":42,"data":"UPDATED"}`)
	val, found, _ = lsm.Get("key-000042")
	fmt.Printf("key-000042 (after update) = %q (found: %v)\n", val, found)

	// Test deletes
	lsm.Delete("key-000042")
	_, found, _ = lsm.Get("key-000042")
	fmt.Printf("key-000042 (after delete): found=%v\n", found)

	// Close and reopen to test WAL recovery
	fmt.Println()
	fmt.Println("=== WAL Recovery Test ===")

	lsm.Set("recovery-test", "this should survive a restart")
	// Do NOT call Close (simulating a crash -- the memtable is lost)
	lsm.memtableWAL.sync() // but the WAL is synced

	// Reopen
	lsm2, err := NewLSMTree(dir, 4096)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Reopen failed: %v\n", err)
		os.Exit(1)
	}

	val, found, _ = lsm2.Get("recovery-test")
	fmt.Printf("After recovery: recovery-test = %q (found: %v)\n", val, found)

	lsm2.Close()
}
```

### How This Maps to Real Databases

| Our Implementation | LevelDB/RocksDB | Cassandra |
|---|---|---|
| `skipList` memtable | SkipList or Arena | ConcurrentSkipListMap |
| `wal` | WAL with CRC checksums | CommitLog |
| `sstable` | Table format with bloom filters, compression | SSTable with bloom filters |
| `compactLevel` | Leveled or Size-Tiered compaction | Size-Tiered or Leveled |
| 4 levels | 7 levels (configurable) | Configurable |

---

## 8. B-Tree Basics

### A Different Approach

While LSM trees optimize for writes, B-trees optimize for reads. A B-tree is a self-balancing tree where each node can contain multiple keys and has multiple children. B-trees are the standard storage engine for relational databases (PostgreSQL, MySQL InnoDB, SQLite).

Key properties:
- Each node is a fixed-size **page** (typically 4KB or 16KB, matching the OS page size)
- All leaf nodes are at the same depth (perfectly balanced)
- Each internal node has between `t-1` and `2t-1` keys (where `t` is the minimum degree)
- Searches are O(log n) with a very small base (high branching factor)

```
                    [  15  |  30  ]
                   /       |       \
          [5|10]       [20|25]       [35|40|45]
         / | \         / | \         / | | \
       [1] [7] [12]  [17] [22] [27] [32] [37] [42] [48]

  - Root has 2 keys, 3 children
  - Internal nodes have 2-3 keys
  - Leaf nodes store actual values
  - All leaves at the same level (depth 2)
  - To find key 22: root -> middle child -> middle child -> found!
    Only 3 page reads for ANY key in the tree
```

### Why B-Trees Use Pages

On a hard drive, reading a single byte is almost as slow as reading 4KB -- the disk head must seek to the right position regardless. So B-trees store many keys per node, filling an entire 4KB page. This means each level of the tree can hold hundreds of keys, so a tree with millions of records only needs 3-4 levels.

```
Branching factor 500 (typical for 4KB pages with small keys):
  Level 0 (root):     1 page     = up to 500 keys
  Level 1:          500 pages    = up to 250,000 keys
  Level 2:       250,000 pages   = up to 125,000,000 keys
  Level 3:   125,000,000 pages   = up to 62,500,000,000 keys

  To find any key among 125 MILLION records: only 3 disk reads!
```

### Go Implementation: In-Memory B-Tree

We start with an in-memory B-tree to understand the algorithms, then move to disk-based in the next section.

```go
package main

import (
	"fmt"
	"strings"
)

const minDegree = 3 // minimum degree t: each node has at most 2t-1 keys

// BTreeNode represents a single node in the B-tree.
type BTreeNode struct {
	keys     []string     // sorted keys
	values   []string     // values corresponding to keys
	children []*BTreeNode // child pointers (len = len(keys) + 1 for internal nodes)
	isLeaf   bool
}

// BTree is a self-balancing tree optimized for disk-based storage.
type BTree struct {
	root *BTreeNode
	t    int // minimum degree
}

// NewBTree creates an empty B-tree with the given minimum degree.
func NewBTree(t int) *BTree {
	return &BTree{
		root: &BTreeNode{isLeaf: true},
		t:    t,
	}
}

// search looks for a key in the subtree rooted at this node.
func (n *BTreeNode) search(key string) (string, bool) {
	// Binary search for the key position
	i := 0
	for i < len(n.keys) && n.keys[i] < key {
		i++
	}

	// Check if we found the key
	if i < len(n.keys) && n.keys[i] == key {
		return n.values[i], true
	}

	// If this is a leaf, the key is not in the tree
	if n.isLeaf {
		return "", false
	}

	// Recurse into the appropriate child
	return n.children[i].search(key)
}

// Get searches for a key in the B-tree.
func (bt *BTree) Get(key string) (string, bool) {
	return bt.root.search(key)
}

// splitChild splits a full child node into two nodes.
// This is the core operation that keeps the B-tree balanced.
func (n *BTreeNode) splitChild(i int, t int) {
	child := n.children[i]

	// Create new node that will hold the right half of child's keys
	newNode := &BTreeNode{
		isLeaf: child.isLeaf,
	}

	midIdx := t - 1

	// The median key moves up to the parent
	medianKey := child.keys[midIdx]
	medianValue := child.values[midIdx]

	// Right half goes to new node
	newNode.keys = append(newNode.keys, child.keys[midIdx+1:]...)
	newNode.values = append(newNode.values, child.values[midIdx+1:]...)

	if !child.isLeaf {
		newNode.children = append(newNode.children, child.children[midIdx+1:]...)
	}

	// Left half stays in original child
	child.keys = child.keys[:midIdx]
	child.values = child.values[:midIdx]
	if !child.isLeaf {
		child.children = child.children[:midIdx+1]
	}

	// Insert median key into parent at position i
	n.keys = append(n.keys, "")
	copy(n.keys[i+1:], n.keys[i:])
	n.keys[i] = medianKey

	n.values = append(n.values, "")
	copy(n.values[i+1:], n.values[i:])
	n.values[i] = medianValue

	// Insert new child pointer
	n.children = append(n.children, nil)
	copy(n.children[i+2:], n.children[i+1:])
	n.children[i+1] = newNode
}

// insertNonFull inserts a key into a node that is guaranteed to not be full.
func (n *BTreeNode) insertNonFull(key, value string, t int) {
	i := len(n.keys) - 1

	if n.isLeaf {
		// Find the position and insert
		n.keys = append(n.keys, "")
		n.values = append(n.values, "")

		for i >= 0 && n.keys[i] > key {
			n.keys[i+1] = n.keys[i]
			n.values[i+1] = n.values[i]
			i--
		}

		// Check for update (key already exists)
		if i >= 0 && n.keys[i] == key {
			n.values[i] = value
			// Remove the extra slots we added
			n.keys = n.keys[:len(n.keys)-1]
			n.values = n.values[:len(n.values)-1]
			return
		}

		n.keys[i+1] = key
		n.values[i+1] = value
	} else {
		// Find which child to recurse into
		for i >= 0 && n.keys[i] > key {
			i--
		}

		// Check for update at this node
		if i >= 0 && n.keys[i] == key {
			n.values[i] = value
			return
		}

		i++

		// If the child is full, split it first
		if len(n.children[i].keys) == 2*t-1 {
			n.splitChild(i, t)
			// After splitting, check which side the key goes to
			if key > n.keys[i] {
				i++
			} else if key == n.keys[i] {
				n.values[i] = value
				return
			}
		}

		n.children[i].insertNonFull(key, value, t)
	}
}

// Set inserts or updates a key-value pair in the B-tree.
func (bt *BTree) Set(key, value string) {
	root := bt.root

	// If root is full, split it
	if len(root.keys) == 2*bt.t-1 {
		newRoot := &BTreeNode{
			children: []*BTreeNode{root},
		}
		newRoot.splitChild(0, bt.t)
		bt.root = newRoot

		// Insert into the correct child
		newRoot.insertNonFull(key, value, bt.t)
	} else {
		root.insertNonFull(key, value, bt.t)
	}
}

// Range returns all key-value pairs in [startKey, endKey] in sorted order.
func (bt *BTree) Range(startKey, endKey string) []struct{ Key, Value string } {
	var results []struct{ Key, Value string }
	bt.root.rangeSearch(startKey, endKey, &results)
	return results
}

func (n *BTreeNode) rangeSearch(startKey, endKey string, results *[]struct{ Key, Value string }) {
	i := 0
	for i < len(n.keys) && n.keys[i] < startKey {
		i++
	}

	for i < len(n.keys) {
		if !n.isLeaf {
			n.children[i].rangeSearch(startKey, endKey, results)
		}
		if n.keys[i] > endKey {
			return
		}
		if n.keys[i] >= startKey {
			*results = append(*results, struct{ Key, Value string }{n.keys[i], n.values[i]})
		}
		i++
	}

	if !n.isLeaf && i < len(n.children) {
		n.children[i].rangeSearch(startKey, endKey, results)
	}
}

// printTree prints the B-tree structure for debugging.
func (bt *BTree) printTree() {
	type queueItem struct {
		node  *BTreeNode
		depth int
	}

	queue := []queueItem{{bt.root, 0}}
	currentDepth := -1

	for len(queue) > 0 {
		item := queue[0]
		queue = queue[1:]

		if item.depth != currentDepth {
			if currentDepth >= 0 {
				fmt.Println()
			}
			currentDepth = item.depth
			fmt.Printf("Level %d: ", currentDepth)
		}

		fmt.Printf("[%s] ", strings.Join(item.node.keys, "|"))

		for _, child := range item.node.children {
			if child != nil {
				queue = append(queue, queueItem{child, item.depth + 1})
			}
		}
	}
	fmt.Println()
}

func main() {
	fmt.Println("=== B-Tree Basics ===")
	fmt.Println()

	bt := NewBTree(minDegree) // minimum degree 3: nodes have 2-5 keys

	// Insert data
	keys := []string{
		"10", "20", "05", "06", "12", "30", "07", "17",
		"15", "25", "35", "40", "01", "03", "08", "09",
		"11", "13", "14", "16", "18", "19", "21", "22",
	}

	for _, k := range keys {
		bt.Set(k, "val-"+k)
	}

	fmt.Println("Tree structure after inserting 24 keys:")
	bt.printTree()

	// Lookups
	fmt.Println()
	for _, key := range []string{"05", "17", "35", "99"} {
		val, found := bt.Get(key)
		if found {
			fmt.Printf("  Get(%q) = %q\n", key, val)
		} else {
			fmt.Printf("  Get(%q) = (not found)\n", key)
		}
	}

	// Range query
	fmt.Println()
	fmt.Println("Range [10, 20]:")
	for _, kv := range bt.Range("10", "20") {
		fmt.Printf("  %s = %s\n", kv.Key, kv.Value)
	}

	// Update
	bt.Set("17", "UPDATED-val-17")
	val, _ := bt.Get("17")
	fmt.Printf("\nAfter update: Get(\"17\") = %q\n", val)

	// Performance characteristics
	fmt.Println()
	fmt.Println("=== B-Tree Performance ===")
	fmt.Println()

	largeBT := NewBTree(50) // minimum degree 50: nodes have 49-99 keys
	for i := 0; i < 1_000_000; i++ {
		largeBT.Set(fmt.Sprintf("%08d", i), fmt.Sprintf("v%d", i))
	}

	// With branching factor ~100, tree depth is about 3
	// So any lookup takes about 3 comparisons
	val, found := largeBT.Get("00500000")
	fmt.Printf("Lookup in 1M-entry B-tree: found=%v, value=%q\n", found, val)

	count := 0
	for _, kv := range largeBT.Range("00100000", "00100010") {
		fmt.Printf("  %s = %s\n", kv.Key, kv.Value)
		count++
	}
	fmt.Printf("Range query returned %d results\n", count)
}
```

---

## 9. B-Tree Implementation

### Disk-Based B-Tree with Page Management

Real B-trees store each node as a fixed-size page on disk. The page manager handles reading and writing pages, and the B-tree uses page IDs instead of pointers. This is how PostgreSQL, MySQL InnoDB, and SQLite work internally.

```go
package main

import (
	"encoding/binary"
	"fmt"
	"io"
	"os"
	"sync"
)

const (
	pageSize    = 4096 // 4KB pages, matching OS page size
	maxKeys     = 50   // maximum keys per node (determines branching factor)
	maxKeySize  = 32
	maxValSize  = 64
	headerSize  = 16 // page header: [isLeaf:1][keyCount:2][padding:13]
)

// PageID uniquely identifies a page on disk.
type PageID uint32

const invalidPage PageID = 0

// DiskBTreePage represents a B-tree node stored on disk.
type DiskBTreePage struct {
	id       PageID
	isLeaf   bool
	keyCount uint16
	keys     [maxKeys]string
	values   [maxKeys]string
	children [maxKeys + 1]PageID
	dirty    bool
}

// PageManager handles reading and writing fixed-size pages to a file.
type PageManager struct {
	file     *os.File
	mu       sync.Mutex
	nextPage PageID
	cache    map[PageID]*DiskBTreePage // simple page cache
}

// NewPageManager creates a new page manager backed by the given file.
func NewPageManager(path string) (*PageManager, error) {
	file, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return nil, err
	}

	// Determine next page ID from file size
	info, err := file.Stat()
	if err != nil {
		file.Close()
		return nil, err
	}

	nextPage := PageID(1) // page 0 is reserved
	if info.Size() > 0 {
		nextPage = PageID(info.Size()/pageSize) + 1
	}

	return &PageManager{
		file:     file,
		nextPage: nextPage,
		cache:    make(map[PageID]*DiskBTreePage),
	}, nil
}

// AllocatePage creates a new empty page.
func (pm *PageManager) AllocatePage(isLeaf bool) *DiskBTreePage {
	pm.mu.Lock()
	defer pm.mu.Unlock()

	page := &DiskBTreePage{
		id:     pm.nextPage,
		isLeaf: isLeaf,
		dirty:  true,
	}
	pm.nextPage++
	pm.cache[page.id] = page
	return page
}

// serializePage converts a page to bytes for writing to disk.
func serializePage(page *DiskBTreePage) []byte {
	buf := make([]byte, pageSize)

	// Header
	if page.isLeaf {
		buf[0] = 1
	}
	binary.LittleEndian.PutUint16(buf[1:3], page.keyCount)

	offset := headerSize

	// Write keys and values
	for i := 0; i < int(page.keyCount); i++ {
		// Key length + key
		keyBytes := []byte(page.keys[i])
		binary.LittleEndian.PutUint16(buf[offset:offset+2], uint16(len(keyBytes)))
		offset += 2
		copy(buf[offset:], keyBytes)
		offset += maxKeySize

		// Value length + value
		valBytes := []byte(page.values[i])
		binary.LittleEndian.PutUint16(buf[offset:offset+2], uint16(len(valBytes)))
		offset += 2
		copy(buf[offset:], valBytes)
		offset += maxValSize
	}

	// Write child page IDs for internal nodes
	if !page.isLeaf {
		childOffset := pageSize - (maxKeys+1)*4
		for i := 0; i <= int(page.keyCount); i++ {
			binary.LittleEndian.PutUint32(buf[childOffset:], uint32(page.children[i]))
			childOffset += 4
		}
	}

	return buf
}

// deserializePage converts bytes back to a page.
func deserializePage(id PageID, buf []byte) *DiskBTreePage {
	page := &DiskBTreePage{id: id}
	page.isLeaf = buf[0] == 1
	page.keyCount = binary.LittleEndian.Uint16(buf[1:3])

	offset := headerSize

	for i := 0; i < int(page.keyCount); i++ {
		keyLen := binary.LittleEndian.Uint16(buf[offset : offset+2])
		offset += 2
		page.keys[i] = string(buf[offset : offset+int(keyLen)])
		offset += maxKeySize

		valLen := binary.LittleEndian.Uint16(buf[offset : offset+2])
		offset += 2
		page.values[i] = string(buf[offset : offset+int(valLen)])
		offset += maxValSize
	}

	if !page.isLeaf {
		childOffset := pageSize - (maxKeys+1)*4
		for i := 0; i <= int(page.keyCount); i++ {
			page.children[i] = PageID(binary.LittleEndian.Uint32(buf[childOffset:]))
			childOffset += 4
		}
	}

	return page
}

// ReadPage reads a page from disk (or cache).
func (pm *PageManager) ReadPage(id PageID) (*DiskBTreePage, error) {
	pm.mu.Lock()
	defer pm.mu.Unlock()

	if page, ok := pm.cache[id]; ok {
		return page, nil
	}

	buf := make([]byte, pageSize)
	offset := int64(id) * int64(pageSize)
	if _, err := pm.file.ReadAt(buf, offset); err != nil && err != io.EOF {
		return nil, err
	}

	page := deserializePage(id, buf)
	pm.cache[id] = page
	return page, nil
}

// WritePage writes a page to disk.
func (pm *PageManager) WritePage(page *DiskBTreePage) error {
	pm.mu.Lock()
	defer pm.mu.Unlock()

	buf := serializePage(page)
	offset := int64(page.id) * int64(pageSize)
	if _, err := pm.file.WriteAt(buf, offset); err != nil {
		return err
	}

	page.dirty = false
	pm.cache[page.id] = page
	return nil
}

// Flush writes all dirty pages to disk and calls fsync.
func (pm *PageManager) Flush() error {
	pm.mu.Lock()
	defer pm.mu.Unlock()

	for _, page := range pm.cache {
		if page.dirty {
			buf := serializePage(page)
			offset := int64(page.id) * int64(pageSize)
			if _, err := pm.file.WriteAt(buf, offset); err != nil {
				return err
			}
			page.dirty = false
		}
	}

	return pm.file.Sync()
}

// Close closes the underlying file.
func (pm *PageManager) Close() error {
	pm.Flush()
	return pm.file.Close()
}

// DiskBTree is a B-tree backed by disk pages.
type DiskBTree struct {
	pm     *PageManager
	rootID PageID
}

// NewDiskBTree creates a new disk-backed B-tree.
func NewDiskBTree(path string) (*DiskBTree, error) {
	pm, err := NewPageManager(path)
	if err != nil {
		return nil, err
	}

	root := pm.AllocatePage(true)
	if err := pm.WritePage(root); err != nil {
		return nil, err
	}

	return &DiskBTree{pm: pm, rootID: root.id}, nil
}

// Get searches for a key in the disk-backed B-tree.
func (bt *DiskBTree) Get(key string) (string, bool, error) {
	return bt.searchNode(bt.rootID, key)
}

func (bt *DiskBTree) searchNode(pageID PageID, key string) (string, bool, error) {
	page, err := bt.pm.ReadPage(pageID)
	if err != nil {
		return "", false, err
	}

	// Find the key or the child to recurse into
	i := 0
	for i < int(page.keyCount) && page.keys[i] < key {
		i++
	}

	if i < int(page.keyCount) && page.keys[i] == key {
		return page.values[i], true, nil
	}

	if page.isLeaf {
		return "", false, nil
	}

	return bt.searchNode(page.children[i], key)
}

// Set inserts or updates a key-value pair.
func (bt *DiskBTree) Set(key, value string) error {
	root, err := bt.pm.ReadPage(bt.rootID)
	if err != nil {
		return err
	}

	if int(root.keyCount) == maxKeys-1 {
		// Root is full -- create new root and split
		newRoot := bt.pm.AllocatePage(false)
		newRoot.children[0] = bt.rootID
		bt.splitChild(newRoot, 0)
		bt.rootID = newRoot.id
		return bt.insertNonFull(newRoot, key, value)
	}

	return bt.insertNonFull(root, key, value)
}

func (bt *DiskBTree) splitChild(parent *DiskBTreePage, idx int) error {
	child, err := bt.pm.ReadPage(parent.children[idx])
	if err != nil {
		return err
	}

	newNode := bt.pm.AllocatePage(child.isLeaf)
	mid := int(child.keyCount) / 2

	// Copy right half to new node
	newNode.keyCount = uint16(int(child.keyCount) - mid - 1)
	for j := 0; j < int(newNode.keyCount); j++ {
		newNode.keys[j] = child.keys[mid+1+j]
		newNode.values[j] = child.values[mid+1+j]
	}

	if !child.isLeaf {
		for j := 0; j <= int(newNode.keyCount); j++ {
			newNode.children[j] = child.children[mid+1+j]
		}
	}

	// Median key moves to parent
	medianKey := child.keys[mid]
	medianValue := child.values[mid]

	// Shrink child to left half
	child.keyCount = uint16(mid)

	// Shift parent's keys and children to make room
	for j := int(parent.keyCount); j > idx; j-- {
		parent.keys[j] = parent.keys[j-1]
		parent.values[j] = parent.values[j-1]
		parent.children[j+1] = parent.children[j]
	}

	parent.keys[idx] = medianKey
	parent.values[idx] = medianValue
	parent.children[idx+1] = newNode.id
	parent.keyCount++

	// Mark all modified pages as dirty and write
	parent.dirty = true
	child.dirty = true
	newNode.dirty = true

	bt.pm.WritePage(parent)
	bt.pm.WritePage(child)
	bt.pm.WritePage(newNode)

	return nil
}

func (bt *DiskBTree) insertNonFull(node *DiskBTreePage, key, value string) error {
	i := int(node.keyCount) - 1

	if node.isLeaf {
		// Check for update
		for j := 0; j < int(node.keyCount); j++ {
			if node.keys[j] == key {
				node.values[j] = value
				node.dirty = true
				return bt.pm.WritePage(node)
			}
		}

		// Shift keys right and insert
		node.keyCount++
		for i >= 0 && node.keys[i] > key {
			node.keys[i+1] = node.keys[i]
			node.values[i+1] = node.values[i]
			i--
		}
		node.keys[i+1] = key
		node.values[i+1] = value
		node.dirty = true
		return bt.pm.WritePage(node)
	}

	// Find child to recurse into
	for i >= 0 && node.keys[i] > key {
		i--
	}
	if i >= 0 && node.keys[i] == key {
		node.values[i] = value
		node.dirty = true
		return bt.pm.WritePage(node)
	}
	i++

	child, err := bt.pm.ReadPage(node.children[i])
	if err != nil {
		return err
	}

	if int(child.keyCount) == maxKeys-1 {
		if err := bt.splitChild(node, i); err != nil {
			return err
		}
		if key > node.keys[i] {
			i++
		} else if key == node.keys[i] {
			node.values[i] = value
			node.dirty = true
			return bt.pm.WritePage(node)
		}
		child, err = bt.pm.ReadPage(node.children[i])
		if err != nil {
			return err
		}
	}

	return bt.insertNonFull(child, key, value)
}

// Close flushes all pages and closes the file.
func (bt *DiskBTree) Close() error {
	return bt.pm.Close()
}

func main() {
	path := "/tmp/disk_btree_example.db"
	os.Remove(path)
	defer os.Remove(path)

	fmt.Println("=== Disk-Based B-Tree ===")
	fmt.Println()

	bt, err := NewDiskBTree(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed: %v\n", err)
		os.Exit(1)
	}

	// Insert data
	for i := 0; i < 10_000; i++ {
		key := fmt.Sprintf("key-%06d", i)
		value := fmt.Sprintf("val-%d", i)
		if err := bt.Set(key, value); err != nil {
			fmt.Fprintf(os.Stderr, "Insert error at %d: %v\n", i, err)
			os.Exit(1)
		}
	}

	// Read back
	val, found, err := bt.Get("key-005000")
	fmt.Printf("Get(\"key-005000\") = %q, found=%v, err=%v\n", val, found, err)

	val, found, err = bt.Get("key-009999")
	fmt.Printf("Get(\"key-009999\") = %q, found=%v, err=%v\n", val, found, err)

	_, found, err = bt.Get("nonexistent")
	fmt.Printf("Get(\"nonexistent\"): found=%v, err=%v\n", found, err)

	// Check file size
	bt.Close()
	info, _ := os.Stat(path)
	fmt.Printf("\nFile size: %d bytes (%.2f MB)\n", info.Size(), float64(info.Size())/(1024*1024))
	fmt.Printf("Pages used: %d\n", info.Size()/pageSize)
	fmt.Printf("Page size: %d bytes\n", pageSize)
}
```

### B-Tree vs Hash Index

| Feature | B-Tree | Hash Index |
|---------|--------|------------|
| Point lookup | O(log n) -- 3-4 disk reads | O(1) -- 1 disk read |
| Range query | Excellent -- leaf nodes are linked | Terrible -- must scan all keys |
| Disk usage | Efficient (pages ~2/3 full) | Growing log + compaction |
| Memory | Only root/top levels cached | ALL keys must fit in RAM |
| Write speed | Slower (random I/O, in-place update) | Fast (sequential append) |

---

## 10. Write-Ahead Log (WAL)

### The Crash Recovery Problem

Consider what happens when your database process crashes mid-write:

1. **Hash index / LSM memtable**: Data in memory is lost. Without a WAL, all unwritten data vanishes.
2. **B-tree in-place update**: A half-written page could corrupt the tree. Without a WAL, the database could become unreadable.

A Write-Ahead Log solves both problems: before making any change to the main data structure, write the change to a sequential log file. If the process crashes, replay the log to recover.

The rule is simple: **the WAL record must be on disk before the corresponding data change is considered committed.**

### Go Implementation

```go
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"hash/crc32"
	"io"
	"os"
	"sync"
	"time"
)

// WALEntryType indicates what kind of operation the entry represents.
type WALEntryType byte

const (
	WALSet    WALEntryType = 1
	WALDelete WALEntryType = 2
)

// WALEntry is a single entry in the write-ahead log.
// On-disk format:
//
//	[CRC32:4][Type:1][KeyLen:4][ValLen:4][Key:KeyLen][Value:ValLen]
//
// The CRC32 checksum covers Type+KeyLen+ValLen+Key+Value, allowing
// us to detect corrupted entries during recovery.
type WALEntry struct {
	Type  WALEntryType
	Key   string
	Value string
}

// WriteAheadLog provides durable, sequential write logging with
// CRC32 checksums for data integrity.
type WriteAheadLog struct {
	file   *os.File
	writer *bufio.Writer
	mu     sync.Mutex
	path   string
	size   int64
}

// NewWriteAheadLog creates or opens a WAL file.
func NewWriteAheadLog(path string) (*WriteAheadLog, error) {
	file, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR|os.O_APPEND, 0644)
	if err != nil {
		return nil, err
	}

	info, _ := file.Stat()
	size := int64(0)
	if info != nil {
		size = info.Size()
	}

	return &WriteAheadLog{
		file:   file,
		writer: bufio.NewWriter(file),
		path:   path,
		size:   size,
	}, nil
}

// Append writes an entry to the WAL.
func (w *WriteAheadLog) Append(entry WALEntry) error {
	w.mu.Lock()
	defer w.mu.Unlock()

	// Build the payload (everything after the CRC)
	keyBytes := []byte(entry.Key)
	valBytes := []byte(entry.Value)

	payloadSize := 1 + 4 + 4 + len(keyBytes) + len(valBytes)
	payload := make([]byte, payloadSize)
	offset := 0

	payload[offset] = byte(entry.Type)
	offset++

	binary.LittleEndian.PutUint32(payload[offset:], uint32(len(keyBytes)))
	offset += 4

	binary.LittleEndian.PutUint32(payload[offset:], uint32(len(valBytes)))
	offset += 4

	copy(payload[offset:], keyBytes)
	offset += len(keyBytes)

	copy(payload[offset:], valBytes)

	// Calculate CRC32 of the payload
	checksum := crc32.ChecksumIEEE(payload)

	// Write CRC32 first, then payload
	if err := binary.Write(w.writer, binary.LittleEndian, checksum); err != nil {
		return err
	}
	if _, err := w.writer.Write(payload); err != nil {
		return err
	}

	w.size += int64(4 + payloadSize)
	return nil
}

// Sync flushes the buffer and calls fsync to ensure data is on disk.
// This is the critical durability guarantee.
func (w *WriteAheadLog) Sync() error {
	w.mu.Lock()
	defer w.mu.Unlock()

	if err := w.writer.Flush(); err != nil {
		return err
	}
	return w.file.Sync()
}

// Recover reads all valid entries from the WAL.
// Entries with invalid CRC32 checksums are skipped (they represent
// incomplete writes from a crash).
func (w *WriteAheadLog) Recover() ([]WALEntry, int, error) {
	w.mu.Lock()
	defer w.mu.Unlock()

	if _, err := w.file.Seek(0, io.SeekStart); err != nil {
		return nil, 0, err
	}

	reader := bufio.NewReader(w.file)
	var entries []WALEntry
	corrupted := 0

	for {
		// Read CRC32
		var checksum uint32
		if err := binary.Read(reader, binary.LittleEndian, &checksum); err != nil {
			break // EOF or incomplete entry
		}

		// Read type
		typeByte, err := reader.ReadByte()
		if err != nil {
			corrupted++
			break
		}

		// Read key length
		var keyLen, valLen uint32
		if err := binary.Read(reader, binary.LittleEndian, &keyLen); err != nil {
			corrupted++
			break
		}
		if err := binary.Read(reader, binary.LittleEndian, &valLen); err != nil {
			corrupted++
			break
		}

		// Read key and value
		keyBuf := make([]byte, keyLen)
		if _, err := io.ReadFull(reader, keyBuf); err != nil {
			corrupted++
			break
		}
		valBuf := make([]byte, valLen)
		if _, err := io.ReadFull(reader, valBuf); err != nil {
			corrupted++
			break
		}

		// Verify CRC32
		payloadSize := 1 + 4 + 4 + int(keyLen) + int(valLen)
		payload := make([]byte, payloadSize)
		offset := 0
		payload[offset] = typeByte
		offset++
		binary.LittleEndian.PutUint32(payload[offset:], keyLen)
		offset += 4
		binary.LittleEndian.PutUint32(payload[offset:], valLen)
		offset += 4
		copy(payload[offset:], keyBuf)
		offset += int(keyLen)
		copy(payload[offset:], valBuf)

		expectedCRC := crc32.ChecksumIEEE(payload)
		if checksum != expectedCRC {
			corrupted++
			continue // skip this corrupted entry
		}

		entries = append(entries, WALEntry{
			Type:  WALEntryType(typeByte),
			Key:   string(keyBuf),
			Value: string(valBuf),
		})
	}

	return entries, corrupted, nil
}

// Truncate clears the WAL (called after a successful checkpoint).
func (w *WriteAheadLog) Truncate() error {
	w.mu.Lock()
	defer w.mu.Unlock()

	if err := w.writer.Flush(); err != nil {
		return err
	}
	if err := w.file.Truncate(0); err != nil {
		return err
	}
	if _, err := w.file.Seek(0, io.SeekStart); err != nil {
		return err
	}
	w.writer.Reset(w.file)
	w.size = 0
	return nil
}

// Size returns the current WAL size in bytes.
func (w *WriteAheadLog) Size() int64 {
	w.mu.Lock()
	defer w.mu.Unlock()
	return w.size
}

// Close closes the WAL file.
func (w *WriteAheadLog) Close() error {
	w.mu.Lock()
	defer w.mu.Unlock()
	w.writer.Flush()
	return w.file.Close()
}

func main() {
	walPath := "/tmp/wal_example.log"
	os.Remove(walPath)
	defer os.Remove(walPath)

	fmt.Println("=== Write-Ahead Log (WAL) ===")
	fmt.Println()

	// Create WAL and write entries
	wal1, err := NewWriteAheadLog(walPath)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed: %v\n", err)
		os.Exit(1)
	}

	entries := []WALEntry{
		{Type: WALSet, Key: "user:1", Value: `{"name":"Alice","age":30}`},
		{Type: WALSet, Key: "user:2", Value: `{"name":"Bob","age":25}`},
		{Type: WALSet, Key: "user:3", Value: `{"name":"Charlie","age":35}`},
		{Type: WALDelete, Key: "user:2", Value: ""},
		{Type: WALSet, Key: "user:1", Value: `{"name":"Alice","age":31}`},
	}

	start := time.Now()
	for _, e := range entries {
		if err := wal1.Append(e); err != nil {
			fmt.Fprintf(os.Stderr, "Append error: %v\n", err)
			os.Exit(1)
		}
	}

	// Sync to guarantee durability
	if err := wal1.Sync(); err != nil {
		fmt.Fprintf(os.Stderr, "Sync error: %v\n", err)
		os.Exit(1)
	}
	writeTime := time.Since(start)

	fmt.Printf("Wrote %d entries in %v\n", len(entries), writeTime)
	fmt.Printf("WAL size: %d bytes\n", wal1.Size())

	// Simulate crash: close without clean shutdown
	wal1.Close()

	// Recover from WAL
	fmt.Println()
	fmt.Println("=== Simulating crash recovery ===")

	wal2, err := NewWriteAheadLog(walPath)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Reopen failed: %v\n", err)
		os.Exit(1)
	}

	recovered, corrupted, err := wal2.Recover()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Recovery failed: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Recovered %d entries, %d corrupted\n", len(recovered), corrupted)

	// Replay entries to rebuild state
	state := make(map[string]string)
	for _, e := range recovered {
		switch e.Type {
		case WALSet:
			state[e.Key] = e.Value
			fmt.Printf("  REPLAY SET %s = %s\n", e.Key, e.Value)
		case WALDelete:
			delete(state, e.Key)
			fmt.Printf("  REPLAY DELETE %s\n", e.Key)
		}
	}

	fmt.Println()
	fmt.Println("Recovered state:")
	for k, v := range state {
		fmt.Printf("  %s = %s\n", k, v)
	}

	// After successful recovery, truncate the WAL
	wal2.Truncate()
	fmt.Printf("\nWAL truncated. Size: %d bytes\n", wal2.Size())

	// Benchmark: WAL write throughput
	fmt.Println()
	fmt.Println("=== WAL Throughput Benchmark ===")

	benchWAL, _ := NewWriteAheadLog("/tmp/wal_bench.log")
	defer os.Remove("/tmp/wal_bench.log")

	n := 100_000
	start = time.Now()
	for i := 0; i < n; i++ {
		benchWAL.Append(WALEntry{
			Type:  WALSet,
			Key:   fmt.Sprintf("key-%d", i),
			Value: fmt.Sprintf("value-%d", i),
		})
	}
	benchWAL.Sync()
	dur := time.Since(start)

	fmt.Printf("Wrote %d entries in %v\n", n, dur)
	fmt.Printf("Throughput: %.0f entries/sec\n", float64(n)/dur.Seconds())
	fmt.Printf("WAL size: %d bytes (%.2f MB)\n", benchWAL.Size(),
		float64(benchWAL.Size())/(1024*1024))

	benchWAL.Close()
	wal2.Close()
}
```

### fsync: The Durability Guarantee

The `file.Sync()` call (which maps to `fsync` on Linux and macOS) is what makes the WAL durable. Without it, the OS may buffer writes in memory and lose them on crash.

```
                Without fsync                    With fsync
                ─────────────                    ──────────
App writes      → OS buffer (RAM)                → OS buffer (RAM)
                  ↓ (when OS decides)              ↓ (immediately forced)
                → Disk                            → Disk

Crash safety:   Data may be lost                  Data is guaranteed on disk
Performance:    Fast (OS batches writes)           Slower (waits for disk)
```

In production databases, the tradeoff between `fsync` frequency and performance is critical. Some databases `fsync` every write (maximum durability). Others batch writes and `fsync` periodically (higher throughput, small window of data loss).

---

## 11. Bloom Filters

### The Problem: Wasted Disk Reads

In an LSM tree, a read must check multiple SSTables to find a key. If the key does not exist, the read checks EVERY SSTable before determining it is missing. This is expensive -- each check involves disk I/O.

A **Bloom filter** is a probabilistic data structure that can tell you:
- **Definitely not in the set** (100% certain)
- **Probably in the set** (small chance of false positive)

By attaching a Bloom filter to each SSTable, we can skip SSTables that definitely do not contain the key, dramatically reducing disk reads for missing keys.

### How Bloom Filters Work

A Bloom filter is a bit array of size `m` with `k` hash functions. To add a key:
1. Hash the key with each of the `k` hash functions
2. Set the corresponding bits to 1

To check if a key might be in the set:
1. Hash the key with each of the `k` hash functions
2. If ALL corresponding bits are 1, the key MIGHT be in the set
3. If ANY bit is 0, the key is DEFINITELY NOT in the set

```
Bloom filter with 16 bits and 3 hash functions:

Add "apple":    hash1("apple")=2, hash2("apple")=7, hash3("apple")=13
  [0 0 1 0 0 0 0 1 0 0 0 0 0 1 0 0]

Add "banana":   hash1("banana")=4, hash2("banana")=7, hash3("banana")=11
  [0 0 1 0 1 0 0 1 0 0 0 1 0 1 0 0]

Check "cherry": hash1("cherry")=2, hash2("cherry")=9, hash3("cherry")=13
  bit 2=1, bit 9=0, bit 13=1 → NOT in set (bit 9 is 0)

Check "apple":  hash1=2, hash2=7, hash3=13
  bit 2=1, bit 7=1, bit 13=1 → PROBABLY in set (all bits are 1)
```

### Go Implementation

```go
package main

import (
	"fmt"
	"hash"
	"hash/fnv"
	"math"
)

// BloomFilter is a space-efficient probabilistic data structure for set
// membership testing. It can have false positives but never false negatives.
type BloomFilter struct {
	bits    []bool
	size    uint   // number of bits
	numHash uint   // number of hash functions
	count   int    // number of items added
}

// NewBloomFilter creates a Bloom filter sized for the expected number of
// items and desired false positive rate.
//
// Formula:
//
//	m = -(n * ln(p)) / (ln(2)^2)  -- optimal number of bits
//	k = (m/n) * ln(2)             -- optimal number of hash functions
//
// where n = expected items, p = false positive rate
func NewBloomFilter(expectedItems int, falsePositiveRate float64) *BloomFilter {
	// Calculate optimal size
	m := -float64(expectedItems) * math.Log(falsePositiveRate) / (math.Log(2) * math.Log(2))
	size := uint(math.Ceil(m))

	// Calculate optimal number of hash functions
	k := float64(size) / float64(expectedItems) * math.Log(2)
	numHash := uint(math.Ceil(k))

	if numHash < 1 {
		numHash = 1
	}

	return &BloomFilter{
		bits:    make([]bool, size),
		size:    size,
		numHash: numHash,
	}
}

// hashValues generates k hash values for the given key using double hashing.
// We use two independent hashes (h1 and h2) and derive k hashes as:
//
//	hash_i = h1 + i * h2  (mod m)
//
// This technique is described in "Less Hashing, Same Performance:
// Building a Better Bloom Filter" by Kirsch and Mitzenmacher.
func (bf *BloomFilter) hashValues(key string) []uint {
	h1 := fnv32(key)
	h2 := fnv32a(key)

	hashes := make([]uint, bf.numHash)
	for i := uint(0); i < bf.numHash; i++ {
		hashes[i] = uint(h1+i*h2) % bf.size
	}
	return hashes
}

func fnv32(key string) uint32 {
	h := fnv.New32()
	h.Write([]byte(key))
	return h.Sum32()
}

func fnv32a(key string) uint32 {
	h := fnv.New32a()
	h.Write([]byte(key))
	return h.Sum32()
}

// Add inserts a key into the Bloom filter.
func (bf *BloomFilter) Add(key string) {
	for _, idx := range bf.hashValues(key) {
		bf.bits[idx] = true
	}
	bf.count++
}

// MayContain tests whether a key MIGHT be in the set.
// Returns false = definitely not in the set.
// Returns true  = probably in the set (with false positive probability).
func (bf *BloomFilter) MayContain(key string) bool {
	for _, idx := range bf.hashValues(key) {
		if !bf.bits[idx] {
			return false // definitely not present
		}
	}
	return true // probably present
}

// FillRatio returns the fraction of bits that are set.
// If this exceeds ~50%, the false positive rate increases rapidly.
func (bf *BloomFilter) FillRatio() float64 {
	set := 0
	for _, b := range bf.bits {
		if b {
			set++
		}
	}
	return float64(set) / float64(bf.size)
}

// Stats returns Bloom filter statistics.
func (bf *BloomFilter) Stats() (size uint, numHash uint, count int, fillRatio float64) {
	return bf.size, bf.numHash, bf.count, bf.FillRatio()
}

// TheoreticalFPR returns the theoretical false positive rate.
func (bf *BloomFilter) TheoreticalFPR() float64 {
	// p = (1 - e^(-kn/m))^k
	exponent := -float64(bf.numHash) * float64(bf.count) / float64(bf.size)
	return math.Pow(1-math.Exp(exponent), float64(bf.numHash))
}

// --- Bloom filter for SSTable integration ---

// BloomSSTable simulates an SSTable with a Bloom filter attached.
type BloomSSTable struct {
	bloom *BloomFilter
	data  map[string]string // simulated on-disk data
	reads int               // counter for disk reads
}

// NewBloomSSTable creates an SSTable with Bloom filter.
func NewBloomSSTable(entries map[string]string) *BloomSSTable {
	bf := NewBloomFilter(len(entries), 0.01) // 1% false positive rate
	for key := range entries {
		bf.Add(key)
	}
	return &BloomSSTable{
		bloom: bf,
		data:  entries,
	}
}

// Get checks the Bloom filter before doing a (simulated) disk read.
func (sst *BloomSSTable) Get(key string) (string, bool) {
	// Step 1: Check Bloom filter (in memory -- free)
	if !sst.bloom.MayContain(key) {
		return "", false // definitely not here -- saved a disk read!
	}

	// Step 2: Bloom filter says "maybe" -- do the actual disk read
	sst.reads++
	val, ok := sst.data[key]
	return val, ok
}

// --- Custom hasher to avoid the hash.Hash interface overhead ---
var _ hash.Hash32 = fnv.New32()

func main() {
	fmt.Println("=== Bloom Filters ===")
	fmt.Println()

	// Create a Bloom filter with 1% false positive rate
	bf := NewBloomFilter(10000, 0.01)

	// Add 10,000 keys
	for i := 0; i < 10000; i++ {
		bf.Add(fmt.Sprintf("key-%d", i))
	}

	size, numHash, count, fillRatio := bf.Stats()
	fmt.Printf("Bloom filter stats:\n")
	fmt.Printf("  Bits:            %d (%.2f KB)\n", size, float64(size)/8/1024)
	fmt.Printf("  Hash functions:  %d\n", numHash)
	fmt.Printf("  Items added:     %d\n", count)
	fmt.Printf("  Fill ratio:      %.2f%%\n", fillRatio*100)
	fmt.Printf("  Theoretical FPR: %.4f%%\n", bf.TheoreticalFPR()*100)
	fmt.Println()

	// Test: all inserted keys should return true
	allFound := true
	for i := 0; i < 10000; i++ {
		if !bf.MayContain(fmt.Sprintf("key-%d", i)) {
			allFound = false
			break
		}
	}
	fmt.Printf("All 10,000 inserted keys found: %v (should be true)\n", allFound)

	// Test: measure actual false positive rate with keys NOT in the set
	falsePositives := 0
	testCount := 100000
	for i := 0; i < testCount; i++ {
		if bf.MayContain(fmt.Sprintf("missing-%d", i)) {
			falsePositives++
		}
	}
	actualFPR := float64(falsePositives) / float64(testCount)
	fmt.Printf("False positive rate: %.4f%% (%d/%d)\n",
		actualFPR*100, falsePositives, testCount)
	fmt.Println()

	// Demonstrate disk read savings with SSTable + Bloom filter
	fmt.Println("=== Bloom Filter + SSTable Integration ===")
	fmt.Println()

	data := make(map[string]string)
	for i := 0; i < 10000; i++ {
		data[fmt.Sprintf("user:%d", i)] = fmt.Sprintf(`{"id":%d}`, i)
	}

	sst := NewBloomSSTable(data)

	// Read existing keys -- Bloom filter says "maybe", then we do a disk read
	for i := 0; i < 1000; i++ {
		sst.Get(fmt.Sprintf("user:%d", i))
	}
	fmt.Printf("1,000 reads for EXISTING keys: %d disk reads\n", sst.reads)

	// Read missing keys -- Bloom filter catches most of them
	sst.reads = 0
	for i := 10000; i < 11000; i++ {
		sst.Get(fmt.Sprintf("user:%d", i)) // these keys do not exist
	}
	fmt.Printf("1,000 reads for MISSING keys:  %d disk reads (%.1f%% saved)\n",
		sst.reads, (1-float64(sst.reads)/1000)*100)

	// Without Bloom filter, ALL 1000 reads would hit disk
	fmt.Printf("Without Bloom filter:          1000 disk reads (0%% saved)\n")
}
```

### Bloom Filter Sizing

The relationship between bits per key and false positive rate:

| Bits per key | False positive rate |
|---|---|
| 5 | ~10% |
| 7 | ~1% |
| 10 | ~0.1% |
| 13 | ~0.01% |
| 17 | ~0.001% |

RocksDB defaults to 10 bits per key (~0.1% false positive rate). For 1 billion keys, this costs about 1.2 GB of RAM -- a good tradeoff for avoiding billions of unnecessary disk reads.

---

## 12. LSM vs B-Tree Comparison

### The Three Amplification Factors

Every storage engine has three kinds of amplification. Understanding them is the key to choosing the right engine.

1. **Write Amplification** -- How many bytes of disk I/O per byte of user data written?
   - B-tree: A single key update requires reading a page, modifying it, and writing it back. With WAL, that is 2 writes. Worst case with splits: many pages rewritten.
   - LSM: A key is written to the WAL, then the memtable, then to SSTable during flush, then again during each level of compaction. Write amplification can be 10-30x.

2. **Read Amplification** -- How many disk reads per user read?
   - B-tree: O(log n) page reads, typically 3-4 for millions of records.
   - LSM: Must check memtable + each level. With Bloom filters, typically 1-2 disk reads. Without them, potentially many.

3. **Space Amplification** -- How much disk space used vs actual data size?
   - B-tree: Pages are typically 50-70% full (fragmentation after splits). Space amplification ~1.5x.
   - LSM: During compaction, old and new data coexist temporarily. Space amplification can be 2x or more.

### Go Benchmark

```go
package main

import (
	"fmt"
	"math/rand"
	"os"
	"runtime"
	"time"
)

// SimpleBTree is a minimal in-memory B-tree for benchmarking.
type SimpleBTree struct {
	root *simpleNode
	t    int
}

type simpleNode struct {
	keys     []string
	values   []string
	children []*simpleNode
	isLeaf   bool
}

func newSimpleBTree(t int) *SimpleBTree {
	return &SimpleBTree{
		root: &simpleNode{isLeaf: true},
		t:    t,
	}
}

func (n *simpleNode) search(key string) (string, bool) {
	i := 0
	for i < len(n.keys) && n.keys[i] < key {
		i++
	}
	if i < len(n.keys) && n.keys[i] == key {
		return n.values[i], true
	}
	if n.isLeaf {
		return "", false
	}
	return n.children[i].search(key)
}

func (bt *SimpleBTree) Get(key string) (string, bool) {
	return bt.root.search(key)
}

func (n *simpleNode) splitChild(i int, t int) {
	child := n.children[i]
	newNode := &simpleNode{isLeaf: child.isLeaf}
	mid := t - 1

	newNode.keys = append([]string{}, child.keys[mid+1:]...)
	newNode.values = append([]string{}, child.values[mid+1:]...)
	if !child.isLeaf {
		newNode.children = append([]*simpleNode{}, child.children[mid+1:]...)
	}

	medKey := child.keys[mid]
	medVal := child.values[mid]

	child.keys = child.keys[:mid]
	child.values = child.values[:mid]
	if !child.isLeaf {
		child.children = child.children[:mid+1]
	}

	n.keys = append(n.keys[:i+1], n.keys[i:]...)
	n.keys[i] = medKey
	n.values = append(n.values[:i+1], n.values[i:]...)
	n.values[i] = medVal
	n.children = append(n.children[:i+2], n.children[i+1:]...)
	n.children[i+1] = newNode
}

func (n *simpleNode) insertNonFull(key, value string, t int) {
	i := len(n.keys) - 1
	if n.isLeaf {
		n.keys = append(n.keys, "")
		n.values = append(n.values, "")
		for i >= 0 && n.keys[i] > key {
			n.keys[i+1] = n.keys[i]
			n.values[i+1] = n.values[i]
			i--
		}
		if i >= 0 && n.keys[i] == key {
			n.values[i] = value
			n.keys = n.keys[:len(n.keys)-1]
			n.values = n.values[:len(n.values)-1]
			return
		}
		n.keys[i+1] = key
		n.values[i+1] = value
	} else {
		for i >= 0 && n.keys[i] > key {
			i--
		}
		if i >= 0 && n.keys[i] == key {
			n.values[i] = value
			return
		}
		i++
		if len(n.children[i].keys) == 2*t-1 {
			n.splitChild(i, t)
			if key > n.keys[i] {
				i++
			} else if key == n.keys[i] {
				n.values[i] = value
				return
			}
		}
		n.children[i].insertNonFull(key, value, t)
	}
}

func (bt *SimpleBTree) Set(key, value string) {
	if len(bt.root.keys) == 2*bt.t-1 {
		newRoot := &simpleNode{children: []*simpleNode{bt.root}}
		newRoot.splitChild(0, bt.t)
		bt.root = newRoot
		newRoot.insertNonFull(key, value, bt.t)
	} else {
		bt.root.insertNonFull(key, value, bt.t)
	}
}

// SimpleHashDB is a hash-indexed log for benchmarking.
type SimpleHashDB struct {
	data map[string]string
}

func newSimpleHashDB() *SimpleHashDB {
	return &SimpleHashDB{data: make(map[string]string)}
}

func (db *SimpleHashDB) Set(key, value string) {
	db.data[key] = value
}

func (db *SimpleHashDB) Get(key string) (string, bool) {
	v, ok := db.data[key]
	return v, ok
}

func benchmark(name string, n int, setFn func(string, string), getFn func(string) (string, bool)) {
	// Generate keys
	keys := make([]string, n)
	for i := 0; i < n; i++ {
		keys[i] = fmt.Sprintf("key-%08d", i)
	}
	values := make([]string, n)
	for i := 0; i < n; i++ {
		values[i] = fmt.Sprintf("value-%08d", i)
	}

	// Benchmark writes
	runtime.GC()
	start := time.Now()
	for i := 0; i < n; i++ {
		setFn(keys[i], values[i])
	}
	writeTime := time.Since(start)

	// Benchmark random reads
	readKeys := make([]string, n)
	for i := 0; i < n; i++ {
		readKeys[i] = keys[rand.Intn(n)]
	}

	runtime.GC()
	start = time.Now()
	found := 0
	for i := 0; i < n; i++ {
		_, ok := getFn(readKeys[i])
		if ok {
			found++
		}
	}
	readTime := time.Since(start)

	// Benchmark sequential reads (range-like)
	start = time.Now()
	for i := 0; i < n; i++ {
		getFn(keys[i])
	}
	seqReadTime := time.Since(start)

	// Memory usage
	var m runtime.MemStats
	runtime.GC()
	runtime.ReadMemStats(&m)

	fmt.Printf("%-20s | %d records\n", name, n)
	fmt.Printf("  Writes:          %12v (%8.0f ops/sec)\n",
		writeTime, float64(n)/writeTime.Seconds())
	fmt.Printf("  Random reads:    %12v (%8.0f ops/sec)\n",
		readTime, float64(n)/readTime.Seconds())
	fmt.Printf("  Sequential reads:%12v (%8.0f ops/sec)\n",
		seqReadTime, float64(n)/seqReadTime.Seconds())
	fmt.Printf("  Heap in use:     %12d bytes (%.2f MB)\n",
		m.HeapInuse, float64(m.HeapInuse)/(1024*1024))
	fmt.Println()
}

func main() {
	fmt.Println("=== LSM vs B-Tree vs Hash Index Comparison ===")
	fmt.Println()

	n := 100_000

	// Benchmark B-Tree
	bt := newSimpleBTree(64)
	benchmark("B-Tree (t=64)", n, bt.Set, bt.Get)

	// Benchmark Hash Index
	hdb := newSimpleHashDB()
	benchmark("Hash Index", n, hdb.Set, hdb.Get)

	// Benchmark smaller B-Tree (simulates different branching factors)
	bt2 := newSimpleBTree(4)
	benchmark("B-Tree (t=4)", n, bt2.Set, bt2.Get)

	fmt.Println("=== Analysis ===")
	fmt.Println()
	fmt.Println("Summary of tradeoffs:")
	fmt.Println()
	fmt.Println("  Hash Index:")
	fmt.Println("    + O(1) point lookups")
	fmt.Println("    + O(1) writes")
	fmt.Println("    - No range queries")
	fmt.Println("    - All keys must fit in RAM")
	fmt.Println()
	fmt.Println("  B-Tree:")
	fmt.Println("    + Excellent range queries")
	fmt.Println("    + O(log n) reads with high branching factor")
	fmt.Println("    + Well-understood, predictable performance")
	fmt.Println("    - Write amplification from in-place updates")
	fmt.Println("    - Page splits cause random I/O")
	fmt.Println()
	fmt.Println("  LSM Tree:")
	fmt.Println("    + Fast sequential writes")
	fmt.Println("    + Good space efficiency after compaction")
	fmt.Println("    + Supports range queries")
	fmt.Println("    - Read amplification (check multiple levels)")
	fmt.Println("    - Write amplification from compaction")
	fmt.Println("    - Compaction can cause latency spikes")
	fmt.Println()
	fmt.Println("  Rule of thumb:")
	fmt.Println("    Write-heavy workload → LSM Tree (Cassandra, RocksDB)")
	fmt.Println("    Read-heavy workload  → B-Tree (PostgreSQL, MySQL)")
	fmt.Println("    Simple key-value     → Hash Index (Redis, Memcached)")

	os.Exit(0)
}
```

### When to Use Which

| Workload | Best Engine | Real-World Database |
|----------|-------------|-------------------|
| Write-heavy, time-series | LSM Tree | Cassandra, InfluxDB, ScyllaDB |
| Read-heavy, transactions | B-Tree | PostgreSQL, MySQL InnoDB |
| Simple key-value, all in RAM | Hash Index | Redis, Memcached |
| Mixed read-write | LSM or B-Tree | RocksDB (LSM), SQLite (B-Tree) |
| Analytical (OLAP) | Column store | ClickHouse, DuckDB |

---

## 13. Column-Oriented Storage Concepts

### Row-Oriented vs Column-Oriented

Everything we have built so far is **row-oriented**: each record stores all fields together. This is great for transactional workloads (OLTP) where you read or write entire records.

But for analytical workloads (OLAP), you often only need a few columns from millions of rows. For example: "What is the average order total for each product category in 2024?" This query only needs two columns (`category` and `total`) out of perhaps twenty.

```
Row-oriented (PostgreSQL, MySQL):
  Row 1: [id=1, name="Widget", category="Tools", price=9.99, ...]
  Row 2: [id=2, name="Gadget", category="Electronics", price=24.99, ...]
  Row 3: [id=3, name="Hammer", category="Tools", price=12.99, ...]

  To compute AVG(price) WHERE category="Tools":
  → Must read ALL columns of ALL rows, then filter and compute
  → Reads ~20 columns × 1M rows = 20M field reads

Column-oriented (ClickHouse, DuckDB, Parquet):
  Column "id":       [1, 2, 3, ...]
  Column "name":     ["Widget", "Gadget", "Hammer", ...]
  Column "category": ["Tools", "Electronics", "Tools", ...]
  Column "price":    [9.99, 24.99, 12.99, ...]

  To compute AVG(price) WHERE category="Tools":
  → Read ONLY the "category" and "price" columns
  → Reads 2 columns × 1M rows = 2M field reads (10x less I/O)
```

### Go Implementation

```go
package main

import (
	"fmt"
	"math"
	"math/rand"
	"runtime"
	"strings"
	"time"
)

// ColumnStore stores data in a columnar layout where each column is
// stored contiguously. This is how analytical databases (ClickHouse,
// DuckDB, BigQuery) and file formats (Parquet, ORC) work.
type ColumnStore struct {
	columns    map[string]*Column
	rowCount   int
	columnOrder []string
}

// Column represents a single column of data.
type Column struct {
	Name   string
	Values []interface{} // in production, this would be typed ([]int64, []float64, etc.)
}

// NewColumnStore creates an empty column store.
func NewColumnStore(columnNames []string) *ColumnStore {
	cs := &ColumnStore{
		columns:     make(map[string]*Column),
		columnOrder: columnNames,
	}
	for _, name := range columnNames {
		cs.columns[name] = &Column{Name: name}
	}
	return cs
}

// AddRow adds a row of values (one per column, in column order).
func (cs *ColumnStore) AddRow(values ...interface{}) {
	for i, name := range cs.columnOrder {
		if i < len(values) {
			cs.columns[name].Values = append(cs.columns[name].Values, values[i])
		}
	}
	cs.rowCount++
}

// ScanColumn returns all values for a single column.
// This is extremely fast because the column is stored contiguously.
func (cs *ColumnStore) ScanColumn(name string) []interface{} {
	col, ok := cs.columns[name]
	if !ok {
		return nil
	}
	return col.Values
}

// FilteredAggregate computes an aggregate (sum, count, avg) on one column
// filtered by a condition on another column.
func (cs *ColumnStore) FilteredAggregate(
	filterCol string,
	filterFn func(interface{}) bool,
	aggCol string,
	aggFn string, // "sum", "count", "avg", "min", "max"
) float64 {
	filter := cs.columns[filterCol]
	agg := cs.columns[aggCol]

	var sum float64
	var count int
	minVal := math.MaxFloat64
	maxVal := -math.MaxFloat64

	for i := 0; i < cs.rowCount; i++ {
		if filterFn(filter.Values[i]) {
			val := toFloat64(agg.Values[i])
			sum += val
			count++
			if val < minVal {
				minVal = val
			}
			if val > maxVal {
				maxVal = val
			}
		}
	}

	switch aggFn {
	case "sum":
		return sum
	case "count":
		return float64(count)
	case "avg":
		if count == 0 {
			return 0
		}
		return sum / float64(count)
	case "min":
		return minVal
	case "max":
		return maxVal
	default:
		return 0
	}
}

// GroupByAggregate groups by one column and computes an aggregate on another.
func (cs *ColumnStore) GroupByAggregate(
	groupCol string,
	aggCol string,
	aggFn string,
) map[string]float64 {
	group := cs.columns[groupCol]
	agg := cs.columns[aggCol]

	sums := make(map[string]float64)
	counts := make(map[string]int)

	for i := 0; i < cs.rowCount; i++ {
		key := fmt.Sprint(group.Values[i])
		val := toFloat64(agg.Values[i])
		sums[key] += val
		counts[key]++
	}

	results := make(map[string]float64)
	for key, sum := range sums {
		switch aggFn {
		case "sum":
			results[key] = sum
		case "count":
			results[key] = float64(counts[key])
		case "avg":
			results[key] = sum / float64(counts[key])
		}
	}
	return results
}

func toFloat64(v interface{}) float64 {
	switch val := v.(type) {
	case float64:
		return val
	case int:
		return float64(val)
	case int64:
		return float64(val)
	default:
		return 0
	}
}

// --- Row-oriented store for comparison ---

type RowStore struct {
	columns []string
	rows    [][]interface{}
}

func NewRowStore(columns []string) *RowStore {
	return &RowStore{columns: columns}
}

func (rs *RowStore) AddRow(values ...interface{}) {
	rs.rows = append(rs.rows, values)
}

func (rs *RowStore) FilteredAggregate(
	filterCol string,
	filterFn func(interface{}) bool,
	aggCol string,
	aggFn string,
) float64 {
	filterIdx := -1
	aggIdx := -1
	for i, name := range rs.columns {
		if name == filterCol {
			filterIdx = i
		}
		if name == aggCol {
			aggIdx = i
		}
	}

	var sum float64
	var count int

	// Must read ENTIRE rows even though we only need 2 columns
	for _, row := range rs.rows {
		if filterFn(row[filterIdx]) {
			sum += toFloat64(row[aggIdx])
			count++
		}
	}

	switch aggFn {
	case "sum":
		return sum
	case "avg":
		if count == 0 {
			return 0
		}
		return sum / float64(count)
	default:
		return 0
	}
}

func main() {
	fmt.Println("=== Column-Oriented Storage ===")
	fmt.Println()

	categories := []string{"Electronics", "Clothing", "Food", "Tools", "Books"}

	n := 1_000_000

	// Create both stores with the same data
	cols := []string{"id", "product", "category", "price", "quantity"}
	colStore := NewColumnStore(cols)
	rowStore := NewRowStore(cols)

	fmt.Printf("Generating %d rows of data...\n", n)
	start := time.Now()

	for i := 0; i < n; i++ {
		id := i
		product := fmt.Sprintf("Product-%d", i)
		category := categories[rand.Intn(len(categories))]
		price := rand.Float64()*100 + 1
		quantity := rand.Intn(100) + 1

		colStore.AddRow(id, product, category, price, quantity)
		rowStore.AddRow(id, product, category, price, quantity)
	}
	fmt.Printf("Data generation: %v\n\n", time.Since(start))

	// Query: AVG(price) WHERE category = "Electronics"
	filterFn := func(v interface{}) bool {
		return v.(string) == "Electronics"
	}

	// Column store query
	runtime.GC()
	start = time.Now()
	colResult := colStore.FilteredAggregate("category", filterFn, "price", "avg")
	colTime := time.Since(start)

	// Row store query
	runtime.GC()
	start = time.Now()
	rowResult := rowStore.FilteredAggregate("category", filterFn, "price", "avg")
	rowTime := time.Since(start)

	fmt.Println("Query: AVG(price) WHERE category = 'Electronics'")
	fmt.Printf("  Column store: %.2f (took %v)\n", colResult, colTime)
	fmt.Printf("  Row store:    %.2f (took %v)\n", rowResult, rowTime)
	if rowTime > 0 {
		fmt.Printf("  Speedup:      %.1fx\n", float64(rowTime)/float64(colTime))
	}
	fmt.Println()

	// Group-by query
	start = time.Now()
	grouped := colStore.GroupByAggregate("category", "price", "avg")
	groupTime := time.Since(start)

	fmt.Println("Query: AVG(price) GROUP BY category")
	fmt.Printf("  Took: %v\n", groupTime)
	for cat, avg := range grouped {
		fmt.Printf("  %-15s: $%.2f\n", cat, avg)
	}

	fmt.Println()
	fmt.Println("=== Why Column Stores Are Faster for Analytics ===")
	fmt.Println()
	fmt.Println("1. Read only the columns you need (less I/O)")
	fmt.Println("2. Same-type data is contiguous (CPU cache friendly)")
	fmt.Println("3. Columnar data compresses much better (same type, similar values)")
	fmt.Println("4. Vectorized operations (SIMD) work on contiguous arrays")
	fmt.Println()
	fmt.Println("Real-world column stores: ClickHouse, DuckDB, BigQuery, Redshift")
	fmt.Println("File formats: Apache Parquet, Apache ORC")

	// Demonstrate compression potential
	fmt.Println()
	fmt.Println("=== Compression Potential ===")
	categories_col := colStore.ScanColumn("category")
	uniqueCategories := make(map[string]bool)
	for _, v := range categories_col {
		uniqueCategories[v.(string)] = true
	}
	fmt.Printf("Category column: %d values, %d unique\n",
		len(categories_col), len(uniqueCategories))
	fmt.Println("With dictionary encoding: each value stored as 1 byte (index into dictionary)")
	rawSize := 0
	for _, v := range categories_col {
		rawSize += len(v.(string))
	}
	compressedSize := len(categories_col) * 1 // 1 byte per dictionary index
	for cat := range uniqueCategories {
		compressedSize += len(cat) // dictionary overhead
	}
	fmt.Printf("Raw size:        %d bytes (%.2f MB)\n", rawSize, float64(rawSize)/(1024*1024))
	fmt.Printf("Compressed size: %d bytes (%.2f MB)\n", compressedSize, float64(compressedSize)/(1024*1024))
	fmt.Printf("Compression:     %.1fx\n", float64(rawSize)/float64(compressedSize))

	_ = strings.Builder{}
}
```

### Node.js Comparison

In Node.js, columnar storage is typically handled by libraries like Apache Arrow JS or by using Parquet files through DuckDB-WASM:

```javascript
// Node.js with DuckDB for columnar queries
const duckdb = require('duckdb');
const db = new duckdb.Database(':memory:');

// DuckDB is a column-oriented database that runs in-process
db.run(`CREATE TABLE orders (
  id INTEGER,
  category VARCHAR,
  price DOUBLE
)`);

// Insert data...

// This query only reads the 'category' and 'price' columns
const result = db.all(`
  SELECT category, AVG(price) as avg_price
  FROM orders
  GROUP BY category
`);
```

The difference: in Go, we built the column store from scratch to understand how it works. In Node.js, you typically use a library that hides the implementation. Understanding the internals helps you write better queries and choose better tools.

---

## 14. Real-World Example: Building a Complete Key-Value Store

### Combining Everything

Now we combine everything -- WAL, memtable, SSTables, Bloom filters, and compaction -- into a complete, working key-value database. This is a simplified version of what LevelDB, RocksDB, or Go's Pebble database does internally.

```go
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"hash/crc32"
	"hash/fnv"
	"io"
	"math"
	"math/rand"
	"os"
	"path/filepath"
	"sort"
	"sync"
	"sync/atomic"
	"time"
)

// ============================================================
// Bloom Filter
// ============================================================

type bloomFilter struct {
	bits    []uint64
	size    uint
	numHash uint
}

func newBloomFilter(expectedItems int, fpRate float64) *bloomFilter {
	m := uint(math.Ceil(-float64(expectedItems) * math.Log(fpRate) / (math.Log(2) * math.Log(2))))
	k := uint(math.Ceil(float64(m) / float64(expectedItems) * math.Log(2)))
	if k < 1 {
		k = 1
	}
	words := (m + 63) / 64
	return &bloomFilter{bits: make([]uint64, words), size: m, numHash: k}
}

func (bf *bloomFilter) add(key string) {
	h1 := fnvHash32(key)
	h2 := fnvHash32a(key)
	for i := uint(0); i < bf.numHash; i++ {
		idx := uint(h1+uint32(i)*h2) % bf.size
		bf.bits[idx/64] |= 1 << (idx % 64)
	}
}

func (bf *bloomFilter) mayContain(key string) bool {
	h1 := fnvHash32(key)
	h2 := fnvHash32a(key)
	for i := uint(0); i < bf.numHash; i++ {
		idx := uint(h1+uint32(i)*h2) % bf.size
		if bf.bits[idx/64]&(1<<(idx%64)) == 0 {
			return false
		}
	}
	return true
}

func fnvHash32(key string) uint32 {
	h := fnv.New32()
	h.Write([]byte(key))
	return h.Sum32()
}

func fnvHash32a(key string) uint32 {
	h := fnv.New32a()
	h.Write([]byte(key))
	return h.Sum32()
}

// ============================================================
// Skip List (Memtable)
// ============================================================

const slMaxLevel = 20

type slNode2 struct {
	key     string
	value   string
	deleted bool
	forward []*slNode2
}

type skipList2 struct {
	header *slNode2
	level  int
	size   int
	bytes  int
}

func newSkipList2() *skipList2 {
	return &skipList2{header: &slNode2{forward: make([]*slNode2, slMaxLevel)}}
}

func (sl *skipList2) set(key, value string, deleted bool) {
	update := make([]*slNode2, slMaxLevel)
	cur := sl.header
	for i := sl.level - 1; i >= 0; i-- {
		for cur.forward[i] != nil && cur.forward[i].key < key {
			cur = cur.forward[i]
		}
		update[i] = cur
	}
	next := cur.forward[0]
	if next != nil && next.key == key {
		sl.bytes -= len(next.value)
		next.value = value
		next.deleted = deleted
		sl.bytes += len(value)
		return
	}
	lvl := 1
	for lvl < slMaxLevel && rand.Float64() < 0.25 {
		lvl++
	}
	if lvl > sl.level {
		for i := sl.level; i < lvl; i++ {
			update[i] = sl.header
		}
		sl.level = lvl
	}
	node := &slNode2{key: key, value: value, deleted: deleted, forward: make([]*slNode2, lvl)}
	for i := 0; i < lvl; i++ {
		node.forward[i] = update[i].forward[i]
		update[i].forward[i] = node
	}
	sl.size++
	sl.bytes += len(key) + len(value) + 100
}

func (sl *skipList2) get(key string) (string, bool, bool) {
	cur := sl.header
	for i := sl.level - 1; i >= 0; i-- {
		for cur.forward[i] != nil && cur.forward[i].key < key {
			cur = cur.forward[i]
		}
	}
	next := cur.forward[0]
	if next != nil && next.key == key {
		return next.value, next.deleted, true
	}
	return "", false, false
}

type kvEntry2 struct {
	key     string
	value   string
	deleted bool
}

func (sl *skipList2) entries() []kvEntry2 {
	result := make([]kvEntry2, 0, sl.size)
	cur := sl.header.forward[0]
	for cur != nil {
		result = append(result, kvEntry2{cur.key, cur.value, cur.deleted})
		cur = cur.forward[0]
	}
	return result
}

// ============================================================
// SSTable with Bloom Filter
// ============================================================

type sstIndex2 struct {
	key    string
	offset int64
}

type ssTable2 struct {
	path  string
	index []sstIndex2
	bloom *bloomFilter
	size  int64
}

func writeSST(path string, entries []kvEntry2, indexEvery int) (*ssTable2, error) {
	f, err := os.Create(path)
	if err != nil {
		return nil, err
	}
	w := bufio.NewWriter(f)
	var idx []sstIndex2
	var offset int64

	bf := newBloomFilter(len(entries)+1, 0.01)

	for i, e := range entries {
		if i%indexEvery == 0 {
			idx = append(idx, sstIndex2{e.key, offset})
		}
		bf.add(e.key)

		flag := byte(0)
		if e.deleted {
			flag = 1
		}
		w.WriteByte(flag)
		binary.Write(w, binary.LittleEndian, uint32(len(e.key)))
		binary.Write(w, binary.LittleEndian, uint32(len(e.value)))
		w.WriteString(e.key)
		w.WriteString(e.value)
		offset += 1 + 4 + 4 + int64(len(e.key)) + int64(len(e.value))
	}
	w.Flush()
	f.Sync()
	f.Close()

	return &ssTable2{path: path, index: idx, bloom: bf, size: offset}, nil
}

func readSSTRecord(f *os.File, offset int64) (kvEntry2, int64, error) {
	if _, err := f.Seek(offset, io.SeekStart); err != nil {
		return kvEntry2{}, 0, err
	}
	buf := make([]byte, 1)
	if _, err := io.ReadFull(f, buf); err != nil {
		return kvEntry2{}, 0, err
	}
	var kl, vl uint32
	binary.Read(f, binary.LittleEndian, &kl)
	binary.Read(f, binary.LittleEndian, &vl)
	kb := make([]byte, kl)
	io.ReadFull(f, kb)
	vb := make([]byte, vl)
	io.ReadFull(f, vb)
	next := offset + 1 + 4 + 4 + int64(kl) + int64(vl)
	return kvEntry2{string(kb), string(vb), buf[0] == 1}, next, nil
}

func (sst *ssTable2) get(key string) (string, bool, bool, error) {
	if !sst.bloom.mayContain(key) {
		return "", false, false, nil
	}
	if len(sst.index) == 0 {
		return "", false, false, nil
	}
	pos := sort.Search(len(sst.index), func(i int) bool { return sst.index[i].key > key }) - 1
	if pos < 0 {
		pos = 0
	}
	end := sst.size
	if pos+1 < len(sst.index) {
		end = sst.index[pos+1].offset
	}
	f, err := os.Open(sst.path)
	if err != nil {
		return "", false, false, err
	}
	defer f.Close()

	off := sst.index[pos].offset
	for off < end {
		e, next, err := readSSTRecord(f, off)
		if err != nil {
			break
		}
		if e.key == key {
			return e.value, e.deleted, true, nil
		}
		if e.key > key {
			break
		}
		off = next
	}
	return "", false, false, nil
}

func (sst *ssTable2) allEntries() ([]kvEntry2, error) {
	f, err := os.Open(sst.path)
	if err != nil {
		return nil, err
	}
	defer f.Close()
	var entries []kvEntry2
	var off int64
	for off < sst.size {
		e, next, err := readSSTRecord(f, off)
		if err != nil {
			break
		}
		entries = append(entries, e)
		off = next
	}
	return entries, nil
}

// ============================================================
// Write-Ahead Log with CRC
// ============================================================

type wal2 struct {
	file *os.File
	w    *bufio.Writer
	mu   sync.Mutex
}

func newWAL2(path string) (*wal2, error) {
	f, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR|os.O_APPEND, 0644)
	if err != nil {
		return nil, err
	}
	return &wal2{file: f, w: bufio.NewWriter(f)}, nil
}

func (w *wal2) append(key, value string, deleted bool) error {
	flag := byte(0)
	if deleted {
		flag = 1
	}
	payload := make([]byte, 1+4+4+len(key)+len(value))
	off := 0
	payload[off] = flag
	off++
	binary.LittleEndian.PutUint32(payload[off:], uint32(len(key)))
	off += 4
	binary.LittleEndian.PutUint32(payload[off:], uint32(len(value)))
	off += 4
	copy(payload[off:], key)
	off += len(key)
	copy(payload[off:], value)

	crc := crc32.ChecksumIEEE(payload)
	if err := binary.Write(w.w, binary.LittleEndian, crc); err != nil {
		return err
	}
	_, err := w.w.Write(payload)
	return err
}

func (w *wal2) sync() error {
	if err := w.w.Flush(); err != nil {
		return err
	}
	return w.file.Sync()
}

func (w *wal2) close() error {
	w.w.Flush()
	return w.file.Close()
}

func recoverWAL2(path string) ([]kvEntry2, error) {
	f, err := os.Open(path)
	if err != nil {
		if os.IsNotExist(err) {
			return nil, nil
		}
		return nil, err
	}
	defer f.Close()
	r := bufio.NewReader(f)
	var entries []kvEntry2
	for {
		var crc uint32
		if err := binary.Read(r, binary.LittleEndian, &crc); err != nil {
			break
		}
		flag, err := r.ReadByte()
		if err != nil {
			break
		}
		var kl, vl uint32
		if err := binary.Read(r, binary.LittleEndian, &kl); err != nil {
			break
		}
		if err := binary.Read(r, binary.LittleEndian, &vl); err != nil {
			break
		}
		kb := make([]byte, kl)
		if _, err := io.ReadFull(r, kb); err != nil {
			break
		}
		vb := make([]byte, vl)
		if _, err := io.ReadFull(r, vb); err != nil {
			break
		}
		// Verify CRC
		payload := make([]byte, 1+4+4+int(kl)+int(vl))
		off := 0
		payload[off] = flag
		off++
		binary.LittleEndian.PutUint32(payload[off:], kl)
		off += 4
		binary.LittleEndian.PutUint32(payload[off:], vl)
		off += 4
		copy(payload[off:], kb)
		off += int(kl)
		copy(payload[off:], vb)
		if crc32.ChecksumIEEE(payload) != crc {
			continue
		}
		entries = append(entries, kvEntry2{string(kb), string(vb), flag == 1})
	}
	return entries, nil
}

// ============================================================
// Complete Key-Value Store (MiniDB)
// ============================================================

// MiniDB is a complete key-value store combining:
//   - Write-Ahead Log for durability
//   - Skip List memtable for in-memory writes
//   - SSTables with Bloom filters for on-disk storage
//   - Multi-level compaction for space management
//
// This is a simplified but functional version of LevelDB/RocksDB.
type MiniDB struct {
	dir          string
	memtable     *skipList2
	wal          *wal2
	levels       [][]*ssTable2 // levels[0] = most recent SSTables
	mu           sync.RWMutex
	sstSeq       int64
	memThreshold int
	compactAt    int

	// Statistics
	walWrites     int64
	memWrites     int64
	flushCount    int64
	compactCount  int64
	bloomSaved    int64
	diskReads     int64
}

// Open opens or creates a MiniDB at the given directory.
func Open(dir string) (*MiniDB, error) {
	if err := os.MkdirAll(dir, 0755); err != nil {
		return nil, err
	}

	walPath := filepath.Join(dir, "current.wal")
	recovered, err := recoverWAL2(walPath)
	if err != nil {
		return nil, fmt.Errorf("WAL recovery: %w", err)
	}

	w, err := newWAL2(walPath)
	if err != nil {
		return nil, err
	}

	mem := newSkipList2()
	for _, e := range recovered {
		mem.set(e.key, e.value, e.deleted)
	}

	db := &MiniDB{
		dir:          dir,
		memtable:     mem,
		wal:          w,
		levels:       make([][]*ssTable2, 4),
		memThreshold: 64 * 1024, // 64KB
		compactAt:    4,
	}

	if len(recovered) > 0 {
		fmt.Printf("[MiniDB] Recovered %d entries from WAL\n", len(recovered))
	}

	return db, nil
}

// Put writes a key-value pair.
func (db *MiniDB) Put(key, value string) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	if err := db.wal.append(key, value, false); err != nil {
		return err
	}
	atomic.AddInt64(&db.walWrites, 1)

	db.memtable.set(key, value, false)
	atomic.AddInt64(&db.memWrites, 1)

	if db.memtable.bytes >= db.memThreshold {
		return db.flush()
	}
	return nil
}

// Delete removes a key by writing a tombstone.
func (db *MiniDB) Delete(key string) error {
	db.mu.Lock()
	defer db.mu.Unlock()

	if err := db.wal.append(key, "", true); err != nil {
		return err
	}
	db.memtable.set(key, "", true)

	if db.memtable.bytes >= db.memThreshold {
		return db.flush()
	}
	return nil
}

// Get reads a value by key.
func (db *MiniDB) Get(key string) (string, bool, error) {
	db.mu.RLock()
	defer db.mu.RUnlock()

	// Check memtable first
	if val, deleted, found := db.memtable.get(key); found {
		if deleted {
			return "", false, nil
		}
		return val, true, nil
	}

	// Check SSTables level by level
	for _, level := range db.levels {
		for i := len(level) - 1; i >= 0; i-- {
			sst := level[i]
			// Bloom filter check
			if !sst.bloom.mayContain(key) {
				atomic.AddInt64(&db.bloomSaved, 1)
				continue
			}
			atomic.AddInt64(&db.diskReads, 1)
			val, deleted, found, err := sst.get(key)
			if err != nil {
				return "", false, err
			}
			if found {
				if deleted {
					return "", false, nil
				}
				return val, true, nil
			}
		}
	}

	return "", false, nil
}

// flush writes the memtable to disk as an SSTable.
func (db *MiniDB) flush() error {
	entries := db.memtable.entries()
	if len(entries) == 0 {
		return nil
	}

	seq := atomic.AddInt64(&db.sstSeq, 1)
	path := filepath.Join(db.dir, fmt.Sprintf("L0-%06d.sst", seq))
	sst, err := writeSST(path, entries, 32)
	if err != nil {
		return err
	}

	db.levels[0] = append(db.levels[0], sst)
	db.memtable = newSkipList2()
	atomic.AddInt64(&db.flushCount, 1)

	// Reset WAL
	db.wal.close()
	os.Remove(filepath.Join(db.dir, "current.wal"))
	w, err := newWAL2(filepath.Join(db.dir, "current.wal"))
	if err != nil {
		return err
	}
	db.wal = w

	if len(db.levels[0]) >= db.compactAt {
		return db.compact(0)
	}
	return nil
}

// compact merges SSTables from one level into the next.
func (db *MiniDB) compact(level int) error {
	if level+1 >= len(db.levels) {
		return nil
	}

	all := make(map[string]kvEntry2)

	// Older data first
	for _, sst := range db.levels[level+1] {
		entries, err := sst.allEntries()
		if err != nil {
			return err
		}
		for _, e := range entries {
			all[e.key] = e
		}
	}

	// Newer data overwrites
	for _, sst := range db.levels[level] {
		entries, err := sst.allEntries()
		if err != nil {
			return err
		}
		for _, e := range entries {
			all[e.key] = e
		}
	}

	sorted := make([]kvEntry2, 0, len(all))
	for _, e := range all {
		if e.deleted && level+1 == len(db.levels)-1 {
			continue // drop tombstones at bottom level
		}
		sorted = append(sorted, e)
	}
	sort.Slice(sorted, func(i, j int) bool { return sorted[i].key < sorted[j].key })

	seq := atomic.AddInt64(&db.sstSeq, 1)
	path := filepath.Join(db.dir, fmt.Sprintf("L%d-%06d.sst", level+1, seq))
	newSST, err := writeSST(path, sorted, 32)
	if err != nil {
		return err
	}

	for _, sst := range db.levels[level] {
		os.Remove(sst.path)
	}
	for _, sst := range db.levels[level+1] {
		os.Remove(sst.path)
	}

	db.levels[level] = nil
	db.levels[level+1] = []*ssTable2{newSST}
	atomic.AddInt64(&db.compactCount, 1)

	return nil
}

// Stats returns database statistics.
func (db *MiniDB) Stats() string {
	db.mu.RLock()
	defer db.mu.RUnlock()

	var totalSSTs int
	var totalSSTSize int64
	for _, level := range db.levels {
		totalSSTs += len(level)
		for _, sst := range level {
			totalSSTSize += sst.size
		}
	}

	return fmt.Sprintf(
		"Memtable: %d entries (%d bytes) | SSTables: %d (%.2f KB) | "+
			"Flushes: %d | Compactions: %d | Bloom saved: %d | Disk reads: %d",
		db.memtable.size, db.memtable.bytes,
		totalSSTs, float64(totalSSTSize)/1024,
		atomic.LoadInt64(&db.flushCount),
		atomic.LoadInt64(&db.compactCount),
		atomic.LoadInt64(&db.bloomSaved),
		atomic.LoadInt64(&db.diskReads),
	)
}

// Close flushes remaining data and closes all files.
func (db *MiniDB) Close() error {
	db.mu.Lock()
	defer db.mu.Unlock()

	if db.memtable.size > 0 {
		if err := db.flush(); err != nil {
			return err
		}
	}
	return db.wal.close()
}

func main() {
	dir := "/tmp/minidb_example"
	os.RemoveAll(dir)
	defer os.RemoveAll(dir)

	fmt.Println("=== MiniDB: A Complete Key-Value Store ===")
	fmt.Println()

	db, err := Open(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to open: %v\n", err)
		os.Exit(1)
	}

	// ---- Write Performance ----
	n := 50_000
	start := time.Now()
	for i := 0; i < n; i++ {
		key := fmt.Sprintf("user:%06d", i)
		value := fmt.Sprintf(`{"id":%d,"name":"User %d","email":"user%d@example.com"}`, i, i, i)
		if err := db.Put(key, value); err != nil {
			fmt.Fprintf(os.Stderr, "Put error: %v\n", err)
			os.Exit(1)
		}
	}
	writeTime := time.Since(start)
	fmt.Printf("Wrote %d records in %v (%.0f writes/sec)\n",
		n, writeTime, float64(n)/writeTime.Seconds())
	fmt.Printf("Stats: %s\n\n", db.Stats())

	// ---- Read Performance (existing keys) ----
	start = time.Now()
	found := 0
	for i := 0; i < 10_000; i++ {
		key := fmt.Sprintf("user:%06d", rand.Intn(n))
		_, ok, err := db.Get(key)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Get error: %v\n", err)
		}
		if ok {
			found++
		}
	}
	readTime := time.Since(start)
	fmt.Printf("10,000 random reads: %v (%.0f reads/sec), found=%d\n",
		readTime, 10_000/readTime.Seconds(), found)

	// ---- Read Performance (missing keys -- Bloom filter saves disk reads) ----
	start = time.Now()
	for i := 0; i < 10_000; i++ {
		key := fmt.Sprintf("nonexistent:%06d", i)
		db.Get(key)
	}
	missingReadTime := time.Since(start)
	fmt.Printf("10,000 missing key reads: %v (Bloom filter saves most disk reads)\n",
		missingReadTime)
	fmt.Printf("Stats: %s\n\n", db.Stats())

	// ---- Update ----
	db.Put("user:000042", `{"id":42,"name":"UPDATED USER","email":"updated@example.com"}`)
	val, ok, _ := db.Get("user:000042")
	fmt.Printf("After update: user:000042 = %q (found=%v)\n", val, ok)

	// ---- Delete ----
	db.Delete("user:000042")
	_, ok, _ = db.Get("user:000042")
	fmt.Printf("After delete: user:000042 found=%v\n\n", ok)

	// ---- Crash Recovery ----
	fmt.Println("=== Crash Recovery Test ===")
	db.Put("crash-test-key", "this-must-survive")
	db.wal.sync() // ensure WAL is on disk

	// Simulate crash: do not call Close()
	db.wal.close()

	// Reopen
	db2, err := Open(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Reopen failed: %v\n", err)
		os.Exit(1)
	}
	val, ok, _ = db2.Get("crash-test-key")
	fmt.Printf("After crash recovery: crash-test-key = %q (found=%v)\n\n", val, ok)

	// ---- Final Stats ----
	fmt.Println("=== Final Stats ===")
	fmt.Printf("%s\n", db2.Stats())

	// Check disk usage
	var totalDisk int64
	filepath.Walk(dir, func(path string, info os.FileInfo, err error) error {
		if err == nil && !info.IsDir() {
			totalDisk += info.Size()
		}
		return nil
	})
	fmt.Printf("Total disk usage: %.2f MB\n", float64(totalDisk)/(1024*1024))

	db2.Close()

	fmt.Println()
	fmt.Println("=== What We Built ===")
	fmt.Println()
	fmt.Println("MiniDB implements the core of an LSM-tree storage engine:")
	fmt.Println("  1. Write-Ahead Log (WAL) for crash durability (CRC32 checksums)")
	fmt.Println("  2. Skip List memtable for fast in-memory writes")
	fmt.Println("  3. SSTables with sparse indexes for on-disk storage")
	fmt.Println("  4. Bloom filters to skip unnecessary disk reads")
	fmt.Println("  5. Multi-level compaction to reclaim space")
	fmt.Println("  6. Tombstone-based deletion")
	fmt.Println("  7. Concurrent read access with sync.RWMutex")
	fmt.Println()
	fmt.Println("This is the same architecture used by:")
	fmt.Println("  - LevelDB (Google)")
	fmt.Println("  - RocksDB (Meta)")
	fmt.Println("  - Pebble (CockroachDB)")
	fmt.Println("  - Cassandra (Apache)")
	fmt.Println("  - HBase (Apache)")
}
```

---

## 15. Key Takeaways

### Storage Engine Fundamentals

1. **Every database is built on a storage engine.** The engine determines the fundamental tradeoffs of the database -- whether reads or writes are fast, whether range queries are efficient, and how much disk space is used.

2. **The append-only log is the simplest storage engine.** O(1) writes, O(n) reads. It is the foundation of all log-structured storage.

3. **Hash indexes make reads O(1) but require all keys in memory.** This is how Bitcask (Riak) works. It is excellent for workloads with a moderate number of keys and high read throughput.

4. **SSTables (Sorted String Tables) enable efficient merging and range queries.** Sorting the data on disk is the key insight that makes LSM trees work.

5. **Memtables (skip lists or red-black trees) sort data in memory before flushing.** They bridge the gap between random writes and sorted on-disk storage.

6. **LSM trees combine memtable + WAL + SSTables + compaction.** They optimize for writes at the cost of read amplification. Used by LevelDB, RocksDB, Cassandra.

7. **B-trees store data in fixed-size pages with high branching factors.** They optimize for reads -- any key can be found in 3-4 disk reads even with millions of records. Used by PostgreSQL, MySQL, SQLite.

8. **Write-Ahead Logs (WAL) provide crash durability.** The rule: write to WAL first, then update the main data structure. On crash, replay the WAL.

9. **Bloom filters eliminate unnecessary disk reads.** A small amount of memory (10 bits per key) can eliminate 99.9% of unnecessary disk reads for missing keys.

10. **Column-oriented storage is better for analytics.** When queries touch only a few columns out of many, reading data column-by-column is dramatically more efficient than row-by-row.

### The Three Amplification Factors

Every storage engine design is a tradeoff between:

- **Write amplification**: How many times is data written to disk? (LSM: 10-30x, B-tree: 2-3x)
- **Read amplification**: How many disk reads per query? (LSM: 1-5, B-tree: 3-4)
- **Space amplification**: How much extra disk space is needed? (LSM: ~2x during compaction, B-tree: ~1.5x from fragmentation)

No engine can minimize all three simultaneously. Understanding this tradeoff is the most important lesson from DDIA Chapter 3.

### Go's Strengths for Storage Engines

Go is an excellent language for implementing storage engines because:

- **Direct file I/O** with `os.File`, `syscall.Mmap`, `file.Sync()`
- **Goroutines** for background compaction without blocking reads
- **`sync.RWMutex`** for concurrent readers with exclusive writers
- **Predictable memory layout** with structs and slices
- **No GC pressure** when using `[]byte` for large buffers
- **Excellent benchmarking** with `testing.B`

Real-world Go storage engines: **Pebble** (CockroachDB), **Badger** (Dgraph), **BoltDB** (etcd), **bbolt** (Consul).

---

## 16. Practice Exercises

### Exercise 1: Crash Safety Audit

Take the `MiniDB` implementation and add a crash simulation test:

1. Write 10,000 records
2. Kill the process at a random point (simulate with `os.Exit(1)` inside the `Put` method)
3. Reopen the database
4. Verify that all committed data (everything before the last `Sync`) is intact
5. Count how many records were lost (between the last sync and the crash)

Think about: What guarantees does the WAL provide? What happens to data that was written to the WAL but not yet synced? What happens to data in the memtable that was not flushed?

### Exercise 2: Range Query Support

Add a `Range(startKey, endKey string) ([]KeyValue, error)` method to MiniDB:

1. Collect matching entries from the memtable (the skip list already supports range queries)
2. Collect matching entries from each SSTable level
3. Merge all results, keeping only the newest value for each key
4. Exclude tombstoned entries
5. Return results in sorted order

Test with: Insert 100,000 user records with keys like `user:000001` through `user:100000`. Verify that `Range("user:050000", "user:050010")` returns exactly 11 records.

### Exercise 3: Configurable Sync Mode

Add a sync mode configuration to MiniDB:

- `SyncEveryWrite`: Call `fsync` after every WAL append (maximum durability, slowest)
- `SyncEveryN(n)`: Call `fsync` every N writes (tradeoff)
- `SyncNever`: Never call `fsync`, rely on OS to flush (fastest, least durable)

Benchmark all three modes with 100,000 writes and measure:
- Throughput (writes per second)
- Durability window (maximum data loss on crash)

### Exercise 4: Size-Tiered Compaction

The current compaction strategy is simple: merge all SSTables from level N into level N+1. Implement **size-tiered compaction** instead:

1. When level 0 has 4 SSTables of similar size, merge them into 1 SSTable at level 1
2. When level 1 has 4 SSTables, merge them into 1 at level 2
3. SSTables at higher levels are progressively larger

Compare the write amplification between the two strategies using counters that track total bytes written.

### Exercise 5: Implement a B-Tree Delete

The B-tree implementation in Section 8-9 supports insert and search but not delete. Implement `Delete(key string)`:

1. Find the key in the tree
2. If the key is in a leaf node with more than `t-1` keys, simply remove it
3. If the key is in an internal node, replace it with its in-order predecessor or successor
4. If a node becomes too small after deletion, merge it with a sibling or redistribute keys

This is the hardest exercise -- B-tree deletion is notoriously tricky. Test with: insert 10,000 keys, delete every other key, verify the remaining keys are still accessible and the tree invariants hold.

### Exercise 6: Build a Bloom Filter with Counting

The standard Bloom filter does not support deletion (you cannot unset a bit because it might be shared by multiple keys). Implement a **counting Bloom filter** that stores a 4-bit counter at each position instead of a single bit:

- `Add(key)`: increment the counters
- `Remove(key)`: decrement the counters (but never below zero)
- `MayContain(key)`: return true if all counters are > 0

Test that after adding 1000 keys and removing 500, the remaining 500 are still detected and the removed 500 are (mostly) not.

### Exercise 7: Build a Column Store with Dictionary Encoding

Extend the column store from Section 13 with dictionary encoding:

1. For columns with low cardinality (few unique values), store an array of integer indices instead of strings
2. Maintain a dictionary mapping indices to actual values
3. Implement a query that filters on a dictionary-encoded column
4. Benchmark the memory usage and query speed against the uncompressed version

Target: a `category` column with 10 unique values and 10 million rows should use ~10MB with dictionary encoding vs ~100MB without.

### Exercise 8: Full-Text Search Index

Build a simple inverted index on top of MiniDB:

1. For each document (value), tokenize it into words
2. For each word, store a posting list: `word -> [doc1, doc2, doc3]`
3. Implement `Search(query string) []string` that finds all documents containing all query words
4. Use MiniDB as the storage backend for both documents and the inverted index

This combines everything: the LSM tree stores the data, and the inverted index is built on top of it using the same storage engine.
