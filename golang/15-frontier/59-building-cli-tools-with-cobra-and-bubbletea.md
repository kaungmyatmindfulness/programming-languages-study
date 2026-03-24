# Chapter 59: Building CLI Tools with Cobra and Bubbletea (TUI) in Go

## Table of Contents

1. [Why Go Is Great for CLI Tools](#1-why-go-is-great-for-cli-tools)
2. [Foundations: os.Args and the flag Package](#2-foundations-osargs-and-the-flag-package)
3. [Cobra Framework](#3-cobra-framework)
4. [Bubbletea TUI Framework](#4-bubbletea-tui-framework)
5. [The Charm Ecosystem](#5-the-charm-ecosystem)
6. [Terminal Colors and Formatting](#6-terminal-colors-and-formatting)
7. [Progress Bars and Spinners](#7-progress-bars-and-spinners)
8. [Interactive Prompts](#8-interactive-prompts)
9. [Cross-Compilation for Multiple Platforms](#9-cross-compilation-for-multiple-platforms)
10. [Distribution](#10-distribution)
11. [Testing CLI Applications](#11-testing-cli-applications)
12. [Best Practices for CLI UX](#12-best-practices-for-cli-ux)

---

## 1. Why Go Is Great for CLI Tools

Go has become the de facto language for modern CLI tools. Many of the most popular command-line
programs in active use today -- Docker, Kubernetes (kubectl), Terraform, Hugo, GitHub CLI (gh),
Caddy, lazygit, and hundreds more -- are written in Go. This is no accident. The language has
properties that make it uniquely suited to this domain.

### Static Binaries with Zero Dependencies

Go compiles to a single, statically linked binary. There is no runtime to install, no virtual
machine, no shared libraries to track. You hand someone a file, they run it, and it works. Compare
this to Python (requires interpreter + pip packages), Node.js (requires node + npm), or Java
(requires JVM). For CLI tools that need to run on servers, in containers, and on developer machines,
this is an enormous advantage.

### Fast Startup Time

Go binaries launch in milliseconds. There is no JIT warmup, no module loading phase, no garbage
collector pre-initialization. When a user types a command and presses Enter, the output appears
instantly. This matters for CLI tools that are invoked hundreds of times per day.

### Cross-Compilation Is Built In

Go supports cross-compilation out of the box. From a single machine you can build binaries for
Linux, macOS, Windows, FreeBSD, and more -- for amd64, arm64, and other architectures. A single
`GOOS=linux GOARCH=arm64 go build` produces an ARM Linux binary from your Mac.

### Excellent Standard Library

The `os`, `flag`, `io`, `fmt`, `path/filepath`, `os/exec`, `text/template`, and `encoding/json`
packages in the standard library cover the vast majority of what CLI tools need. You can build
useful tools without any third-party dependencies at all.

### Concurrency for Parallel Operations

Go's goroutines and channels make it trivial to perform work concurrently -- downloading files in
parallel, spinning up progress indicators alongside real work, polling multiple APIs at once. This
is particularly valuable in tools that perform I/O-heavy operations.

### Mature Ecosystem

The Go ecosystem has best-in-class libraries for CLI development:

| Library | Purpose |
|---------|---------|
| `cobra` | Command and subcommand framework |
| `viper` | Configuration management |
| `bubbletea` | Terminal User Interface (TUI) framework |
| `lipgloss` | Terminal styling |
| `bubbles` | Reusable TUI components |
| `fatih/color` | Terminal colors |
| `promptui` | Interactive prompts |
| `goreleaser` | Build and release automation |

---

## 2. Foundations: os.Args and the flag Package

Before diving into frameworks, it is important to understand the primitives Go provides for
command-line argument parsing.

### 2.1 os.Args

The simplest way to access command-line arguments is through `os.Args`, a string slice that
contains every argument passed to the program. The first element is always the program name.

```go
// cmd/args-demo/main.go
package main

import (
    "fmt"
    "os"
    "strings"
)

func main() {
    // os.Args[0] is the program name
    fmt.Printf("Program: %s\n", os.Args[0])
    fmt.Printf("Number of arguments: %d\n", len(os.Args)-1)

    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "Usage: args-demo <name> [greeting]")
        os.Exit(1)
    }

    name := os.Args[1]
    greeting := "Hello"
    if len(os.Args) >= 3 {
        greeting = strings.Join(os.Args[2:], " ")
    }

    fmt.Printf("%s, %s!\n", greeting, name)
}
```

```bash
$ go run main.go Alice
Hello, Alice!

$ go run main.go Alice Good morning
Good morning, Alice!
```

> **Tip:** `os.Args` is fine for the simplest scripts, but it quickly becomes unmanageable when
> you need flags, defaults, help text, or subcommands. Graduate to the `flag` package or Cobra
> as soon as your needs grow.

### 2.2 The flag Package

The standard library `flag` package provides proper flag parsing with types, defaults, and
automatic help generation.

```go
// cmd/flag-demo/main.go
package main

import (
    "flag"
    "fmt"
    "os"
    "strings"
)

func main() {
    // Define flags with default values and help text
    var (
        name    = flag.String("name", "World", "Name to greet")
        count   = flag.Int("count", 1, "Number of times to greet")
        upper   = flag.Bool("upper", false, "Print greeting in uppercase")
        verbose = flag.Bool("v", false, "Enable verbose output")
    )

    // Custom usage message
    flag.Usage = func() {
        fmt.Fprintf(os.Stderr, "Usage: greet [options]\n\nOptions:\n")
        flag.PrintDefaults()
    }

    // Parse command-line flags
    flag.Parse()

    // flag.Args() returns non-flag arguments
    remaining := flag.Args()
    if *verbose {
        fmt.Printf("Remaining args: %v\n", remaining)
    }

    greeting := fmt.Sprintf("Hello, %s!", *name)
    if *upper {
        greeting = strings.ToUpper(greeting)
    }

    for i := 0; i < *count; i++ {
        fmt.Println(greeting)
    }
}
```

```bash
$ go run main.go -name Alice -count 3
Hello, Alice!
Hello, Alice!
Hello, Alice!

$ go run main.go -upper -name "Go Developer"
HELLO, GO DEVELOPER!

$ go run main.go -h
Usage: greet [options]

Options:
  -count int
        Number of times to greet (default 1)
  -name string
        Name to greet (default "World")
  -upper
        Print greeting in uppercase
  -v    Enable verbose output
```

### 2.3 Flag Binding to Struct Fields

For more organized code, bind flags directly to struct fields:

```go
package main

import (
    "flag"
    "fmt"
    "time"
)

type Config struct {
    Host    string
    Port    int
    Timeout time.Duration
    Debug   bool
}

func main() {
    cfg := Config{}

    flag.StringVar(&cfg.Host, "host", "localhost", "Server host")
    flag.IntVar(&cfg.Port, "port", 8080, "Server port")
    flag.DurationVar(&cfg.Timeout, "timeout", 30*time.Second, "Request timeout")
    flag.BoolVar(&cfg.Debug, "debug", false, "Enable debug mode")

    flag.Parse()

    fmt.Printf("Config: %+v\n", cfg)
}
```

### 2.4 Custom Flag Types

Implement the `flag.Value` interface to create flags with custom parsing logic:

```go
package main

import (
    "flag"
    "fmt"
    "strings"
)

// StringSlice implements flag.Value for a comma-separated list of strings.
type StringSlice []string

func (s *StringSlice) String() string {
    return strings.Join(*s, ",")
}

func (s *StringSlice) Set(value string) error {
    *s = strings.Split(value, ",")
    return nil
}

// LogLevel implements flag.Value for validated log levels.
type LogLevel string

func (l *LogLevel) String() string {
    return string(*l)
}

func (l *LogLevel) Set(value string) error {
    valid := map[string]bool{
        "debug": true, "info": true, "warn": true, "error": true,
    }
    lower := strings.ToLower(value)
    if !valid[lower] {
        return fmt.Errorf("invalid log level %q; must be one of: debug, info, warn, error", value)
    }
    *l = LogLevel(lower)
    return nil
}

func main() {
    var tags StringSlice
    var level LogLevel = "info"

    flag.Var(&tags, "tags", "Comma-separated list of tags")
    flag.Var(&level, "level", "Log level (debug, info, warn, error)")
    flag.Parse()

    fmt.Printf("Tags:  %v\n", tags)
    fmt.Printf("Level: %s\n", level)
}
```

```bash
$ go run main.go -tags "web,api,backend" -level debug
Tags:  [web api backend]
Level: debug

$ go run main.go -level critical
invalid value "critical" for flag -level: invalid log level "critical"; must be one of: debug, info, warn, error
```

### 2.5 Limitations of the flag Package

The standard `flag` package has limitations that motivate the use of Cobra:

| Limitation | Impact |
|-----------|--------|
| No subcommand support | Cannot build `git commit`, `docker run` style CLIs |
| Only `-flag` syntax | No POSIX `--long-flag` or `-s` short flags |
| No required flags | Must validate manually |
| No flag grouping | All flags are flat |
| No shell completions | Users must remember flag names |
| No automatic man pages | No generated documentation |

---

## 3. Cobra Framework

[Cobra](https://github.com/spf13/cobra) is the most widely used CLI framework in the Go ecosystem.
It powers kubectl, Hugo, GitHub CLI, Helm, and many other major projects. It provides subcommands,
POSIX-compliant flags, shell completions, automatic help generation, and integration with Viper for
configuration.

### 3.1 Installation and Project Setup

```bash
# Install cobra-cli (the scaffolding tool)
go install github.com/spf13/cobra-cli@latest

# Create a new project
mkdir taskctl && cd taskctl
go mod init github.com/yourname/taskctl

# Initialize cobra project structure
cobra-cli init

# Add subcommands
cobra-cli add add
cobra-cli add list
cobra-cli add complete
cobra-cli add delete
```

This generates the following structure:

```
taskctl/
├── cmd/
│   ├── root.go       # Root command, global flags, initialization
│   ├── add.go        # 'add' subcommand
│   ├── list.go       # 'list' subcommand
│   ├── complete.go   # 'complete' subcommand
│   └── delete.go     # 'delete' subcommand
├── main.go           # Entry point (calls cmd.Execute)
├── go.mod
└── go.sum
```

### 3.2 Understanding the Root Command

The root command is the entry point of your application and the parent of all subcommands:

```go
// cmd/root.go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var cfgFile string

// rootCmd represents the base command when called without any subcommands.
var rootCmd = &cobra.Command{
    Use:   "taskctl",
    Short: "A task management CLI tool",
    Long: `taskctl is a command-line task manager that helps you organize
your work. It supports adding, listing, completing, and deleting tasks,
with priorities, due dates, and tags.

Examples:
  taskctl add "Buy groceries" --priority high --due tomorrow
  taskctl list --status pending
  taskctl complete 42
  taskctl delete 42`,
    // If the root command has its own action (uncommon):
    // Run: func(cmd *cobra.Command, args []string) { },
}

// Execute adds all child commands to the root command and sets flags.
// This is called by main.main(). It only needs to happen once.
func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func init() {
    // Run before any command executes
    cobra.OnInitialize(initConfig)

    // Persistent flags are available to this command and all subcommands
    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "",
        "config file (default is $HOME/.taskctl.yaml)")
    rootCmd.PersistentFlags().Bool("verbose", false, "Enable verbose output")
    rootCmd.PersistentFlags().String("format", "table",
        "Output format (table, json, yaml)")

    // Bind flags to viper keys
    viper.BindPFlag("verbose", rootCmd.PersistentFlags().Lookup("verbose"))
    viper.BindPFlag("format", rootCmd.PersistentFlags().Lookup("format"))
}

func initConfig() {
    if cfgFile != "" {
        viper.SetConfigFile(cfgFile)
    } else {
        home, err := os.UserHomeDir()
        cobra.CheckErr(err)

        viper.AddConfigPath(home)
        viper.AddConfigPath(".")
        viper.SetConfigType("yaml")
        viper.SetConfigName(".taskctl")
    }

    // Read environment variables with TASKCTL_ prefix
    viper.SetEnvPrefix("TASKCTL")
    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err == nil {
        if viper.GetBool("verbose") {
            fmt.Fprintln(os.Stderr, "Using config file:", viper.ConfigFileUsed())
        }
    }
}
```

### 3.3 The main.go Entry Point

```go
// main.go
package main

import "github.com/yourname/taskctl/cmd"

func main() {
    cmd.Execute()
}
```

This is intentionally minimal. All logic lives in the `cmd` package.

### 3.4 Commands and Subcommands

Each command is a `*cobra.Command` struct. Let's build a complete task manager.

#### The Data Layer

```go
// internal/task/task.go
package task

import (
    "encoding/json"
    "fmt"
    "os"
    "path/filepath"
    "sort"
    "time"
)

type Priority string

const (
    PriorityLow    Priority = "low"
    PriorityMedium Priority = "medium"
    PriorityHigh   Priority = "high"
)

type Status string

const (
    StatusPending   Status = "pending"
    StatusCompleted Status = "completed"
)

type Task struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description,omitempty"`
    Priority    Priority  `json:"priority"`
    Status      Status    `json:"status"`
    Tags        []string  `json:"tags,omitempty"`
    DueDate     string    `json:"due_date,omitempty"`
    CreatedAt   time.Time `json:"created_at"`
    CompletedAt time.Time `json:"completed_at,omitempty"`
}

type Store struct {
    path  string
    Tasks []Task `json:"tasks"`
    NextID int   `json:"next_id"`
}

func NewStore(path string) (*Store, error) {
    s := &Store{path: path, NextID: 1}

    // Ensure directory exists
    dir := filepath.Dir(path)
    if err := os.MkdirAll(dir, 0o755); err != nil {
        return nil, fmt.Errorf("creating store directory: %w", err)
    }

    // Load existing data
    data, err := os.ReadFile(path)
    if err != nil {
        if os.IsNotExist(err) {
            return s, nil
        }
        return nil, fmt.Errorf("reading store: %w", err)
    }

    if err := json.Unmarshal(data, s); err != nil {
        return nil, fmt.Errorf("parsing store: %w", err)
    }

    return s, nil
}

func (s *Store) Save() error {
    data, err := json.MarshalIndent(s, "", "  ")
    if err != nil {
        return fmt.Errorf("marshaling store: %w", err)
    }
    return os.WriteFile(s.path, data, 0o644)
}

func (s *Store) Add(title, description string, priority Priority, tags []string, dueDate string) Task {
    t := Task{
        ID:          s.NextID,
        Title:       title,
        Description: description,
        Priority:    priority,
        Status:      StatusPending,
        Tags:        tags,
        DueDate:     dueDate,
        CreatedAt:   time.Now(),
    }
    s.Tasks = append(s.Tasks, t)
    s.NextID++
    return t
}

func (s *Store) Complete(id int) (*Task, error) {
    for i := range s.Tasks {
        if s.Tasks[i].ID == id {
            if s.Tasks[i].Status == StatusCompleted {
                return nil, fmt.Errorf("task %d is already completed", id)
            }
            s.Tasks[i].Status = StatusCompleted
            s.Tasks[i].CompletedAt = time.Now()
            return &s.Tasks[i], nil
        }
    }
    return nil, fmt.Errorf("task %d not found", id)
}

func (s *Store) Delete(id int) (*Task, error) {
    for i := range s.Tasks {
        if s.Tasks[i].ID == id {
            removed := s.Tasks[i]
            s.Tasks = append(s.Tasks[:i], s.Tasks[i+1:]...)
            return &removed, nil
        }
    }
    return nil, fmt.Errorf("task %d not found", id)
}

func (s *Store) List(status Status, priority Priority, tag string) []Task {
    var result []Task
    for _, t := range s.Tasks {
        if status != "" && t.Status != status {
            continue
        }
        if priority != "" && t.Priority != priority {
            continue
        }
        if tag != "" && !containsTag(t.Tags, tag) {
            continue
        }
        result = append(result, t)
    }

    // Sort by priority (high > medium > low), then by creation date
    priorityOrder := map[Priority]int{
        PriorityHigh:   0,
        PriorityMedium: 1,
        PriorityLow:    2,
    }
    sort.Slice(result, func(i, j int) bool {
        if result[i].Priority != result[j].Priority {
            return priorityOrder[result[i].Priority] < priorityOrder[result[j].Priority]
        }
        return result[i].CreatedAt.Before(result[j].CreatedAt)
    })

    return result
}

func containsTag(tags []string, tag string) bool {
    for _, t := range tags {
        if t == tag {
            return true
        }
    }
    return false
}
```

#### The Add Command

```go
// cmd/add.go
package cmd

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
    "github.com/yourname/taskctl/internal/task"
)

var addCmd = &cobra.Command{
    Use:   "add <title>",
    Short: "Add a new task",
    Long: `Add a new task with an optional description, priority, tags, and due date.

Examples:
  taskctl add "Buy groceries"
  taskctl add "Write report" --priority high --due 2026-04-01
  taskctl add "Review PR" --tags "code,review" --desc "Review PR #42"`,

    // Require at least one argument (the title)
    Args: cobra.MinimumNArgs(1),

    RunE: func(cmd *cobra.Command, args []string) error {
        store, err := getStore()
        if err != nil {
            return err
        }

        title := strings.Join(args, " ")
        desc, _ := cmd.Flags().GetString("desc")
        priority, _ := cmd.Flags().GetString("priority")
        tagsStr, _ := cmd.Flags().GetString("tags")
        due, _ := cmd.Flags().GetString("due")

        var tags []string
        if tagsStr != "" {
            tags = strings.Split(tagsStr, ",")
        }

        t := store.Add(title, desc, task.Priority(priority), tags, due)
        if err := store.Save(); err != nil {
            return fmt.Errorf("saving task: %w", err)
        }

        format := viper.GetString("format")
        switch format {
        case "json":
            // JSON output handled elsewhere
            fmt.Printf("{\"id\": %d, \"title\": %q}\n", t.ID, t.Title)
        default:
            fmt.Printf("Created task #%d: %s\n", t.ID, t.Title)
        }

        return nil
    },
}

func init() {
    rootCmd.AddCommand(addCmd)

    // Local flags -- only available on this command
    addCmd.Flags().StringP("desc", "d", "", "Task description")
    addCmd.Flags().StringP("priority", "p", "medium",
        "Task priority (low, medium, high)")
    addCmd.Flags().StringP("tags", "t", "", "Comma-separated tags")
    addCmd.Flags().String("due", "", "Due date (YYYY-MM-DD)")
}

func getStore() (*task.Store, error) {
    home, err := os.UserHomeDir()
    if err != nil {
        return nil, err
    }
    storePath := viper.GetString("store_path")
    if storePath == "" {
        storePath = filepath.Join(home, ".taskctl", "tasks.json")
    }
    return task.NewStore(storePath)
}
```

#### The List Command

```go
// cmd/list.go
package cmd

import (
    "encoding/json"
    "fmt"
    "os"
    "text/tabwriter"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
    "github.com/yourname/taskctl/internal/task"
)

var listCmd = &cobra.Command{
    Use:     "list",
    Aliases: []string{"ls", "l"},
    Short:   "List tasks",
    Long:    `List tasks with optional filters for status, priority, and tags.`,

    RunE: func(cmd *cobra.Command, args []string) error {
        store, err := getStore()
        if err != nil {
            return err
        }

        status, _ := cmd.Flags().GetString("status")
        priority, _ := cmd.Flags().GetString("priority")
        tag, _ := cmd.Flags().GetString("tag")

        tasks := store.List(
            task.Status(status),
            task.Priority(priority),
            tag,
        )

        if len(tasks) == 0 {
            fmt.Println("No tasks found.")
            return nil
        }

        format := viper.GetString("format")
        switch format {
        case "json":
            return printJSON(tasks)
        default:
            return printTable(tasks)
        }
    },
}

func init() {
    rootCmd.AddCommand(listCmd)

    listCmd.Flags().StringP("status", "s", "", "Filter by status (pending, completed)")
    listCmd.Flags().StringP("priority", "p", "", "Filter by priority (low, medium, high)")
    listCmd.Flags().StringP("tag", "t", "", "Filter by tag")
}

func printTable(tasks []task.Task) error {
    w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
    fmt.Fprintln(w, "ID\tSTATUS\tPRIORITY\tTITLE\tDUE")
    fmt.Fprintln(w, "--\t------\t--------\t-----\t---")

    for _, t := range tasks {
        statusIcon := "[ ]"
        if t.Status == task.StatusCompleted {
            statusIcon = "[x]"
        }

        priorityLabel := string(t.Priority)
        due := t.DueDate
        if due == "" {
            due = "-"
        }

        fmt.Fprintf(w, "%d\t%s\t%s\t%s\t%s\n",
            t.ID, statusIcon, priorityLabel, t.Title, due)
    }

    return w.Flush()
}

func printJSON(tasks []task.Task) error {
    enc := json.NewEncoder(os.Stdout)
    enc.SetIndent("", "  ")
    return enc.Encode(tasks)
}
```

#### The Complete and Delete Commands

```go
// cmd/complete.go
package cmd

import (
    "fmt"
    "strconv"

    "github.com/spf13/cobra"
)

var completeCmd = &cobra.Command{
    Use:   "complete <task-id>",
    Short: "Mark a task as completed",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        id, err := strconv.Atoi(args[0])
        if err != nil {
            return fmt.Errorf("invalid task ID: %s", args[0])
        }

        store, err := getStore()
        if err != nil {
            return err
        }

        t, err := store.Complete(id)
        if err != nil {
            return err
        }

        if err := store.Save(); err != nil {
            return fmt.Errorf("saving: %w", err)
        }

        fmt.Printf("Completed task #%d: %s\n", t.ID, t.Title)
        return nil
    },
}

func init() {
    rootCmd.AddCommand(completeCmd)
}
```

```go
// cmd/delete.go
package cmd

import (
    "fmt"
    "strconv"

    "github.com/spf13/cobra"
)

var deleteCmd = &cobra.Command{
    Use:     "delete <task-id>",
    Aliases: []string{"rm", "remove"},
    Short:   "Delete a task",
    Args:    cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        id, err := strconv.Atoi(args[0])
        if err != nil {
            return fmt.Errorf("invalid task ID: %s", args[0])
        }

        store, err := getStore()
        if err != nil {
            return err
        }

        force, _ := cmd.Flags().GetBool("force")
        t, err := store.Delete(id)
        if err != nil {
            return err
        }

        if !force {
            // In a real app, you'd prompt for confirmation here
        }

        if err := store.Save(); err != nil {
            return fmt.Errorf("saving: %w", err)
        }

        fmt.Printf("Deleted task #%d: %s\n", t.ID, t.Title)
        return nil
    },
}

func init() {
    rootCmd.AddCommand(deleteCmd)
    deleteCmd.Flags().BoolP("force", "f", false, "Skip confirmation prompt")
}
```

### 3.5 Flags: Persistent, Local, and Required

Cobra supports three categories of flags. Understanding when to use each is critical.

```go
// Persistent flags: inherited by ALL subcommands
rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "Config file path")
rootCmd.PersistentFlags().BoolP("verbose", "v", false, "Verbose output")
rootCmd.PersistentFlags().String("format", "table", "Output format")

// Local flags: available ONLY on the command they are defined on
addCmd.Flags().StringP("priority", "p", "medium", "Task priority")
addCmd.Flags().StringP("tags", "t", "", "Comma-separated tags")

// Required flags: the command will fail if these are not provided
addCmd.Flags().String("assignee", "", "Person to assign this task to")
addCmd.MarkFlagRequired("assignee")

// Required flag group: at least one of these must be provided
cmd.MarkFlagsOneRequired("output", "file", "stdout")

// Mutually exclusive flags: at most one can be provided
cmd.MarkFlagsMutuallyExclusive("json", "yaml", "table")

// Flag with custom completion
addCmd.RegisterFlagCompletionFunc("priority", func(
    cmd *cobra.Command, args []string, toComplete string,
) ([]string, cobra.ShellCompDirective) {
    return []string{"low", "medium", "high"}, cobra.ShellCompDirectiveNoFileComp
})
```

### 3.6 Arguments Validation

Cobra provides built-in validators for positional arguments:

```go
// No arguments allowed
var statusCmd = &cobra.Command{
    Use:  "status",
    Args: cobra.NoArgs,
    // ...
}

// Exactly N arguments
var completeCmd = &cobra.Command{
    Use:  "complete <task-id>",
    Args: cobra.ExactArgs(1),
    // ...
}

// Between min and max arguments
var tagCmd = &cobra.Command{
    Use:  "tag <task-id> <tag1> [tag2...]",
    Args: cobra.RangeArgs(2, 10),
    // ...
}

// At least N arguments
var addCmd = &cobra.Command{
    Use:  "add <title-words...>",
    Args: cobra.MinimumNArgs(1),
    // ...
}

// Custom argument validation
var deployCmd = &cobra.Command{
    Use:  "deploy <environment>",
    Args: func(cmd *cobra.Command, args []string) error {
        if len(args) != 1 {
            return fmt.Errorf("requires exactly one argument: environment name")
        }
        valid := map[string]bool{
            "dev": true, "staging": true, "production": true,
        }
        if !valid[args[0]] {
            return fmt.Errorf("invalid environment %q; must be dev, staging, or production", args[0])
        }
        return nil
    },
    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Printf("Deploying to %s...\n", args[0])
        return nil
    },
}

// Combine validators
var importCmd = &cobra.Command{
    Use: "import <file1> [file2...]",
    Args: cobra.MatchAll(
        cobra.MinimumNArgs(1),
        func(cmd *cobra.Command, args []string) error {
            for _, arg := range args {
                if filepath.Ext(arg) != ".json" {
                    return fmt.Errorf("file %q must have .json extension", arg)
                }
            }
            return nil
        },
    ),
    // ...
}
```

### 3.7 Command Groups and Nested Subcommands

For larger CLIs, organize commands into groups with nested subcommands:

```go
// cmd/config.go -- parent command (no Run, just a container)
var configCmd = &cobra.Command{
    Use:   "config",
    Short: "Manage configuration",
    Long:  "View and modify taskctl configuration settings.",
}

func init() {
    rootCmd.AddCommand(configCmd)
}

// cmd/config_get.go
var configGetCmd = &cobra.Command{
    Use:   "get <key>",
    Short: "Get a configuration value",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        key := args[0]
        value := viper.Get(key)
        if value == nil {
            return fmt.Errorf("key %q not found", key)
        }
        fmt.Printf("%s = %v\n", key, value)
        return nil
    },
}

func init() {
    configCmd.AddCommand(configGetCmd)
}

// cmd/config_set.go
var configSetCmd = &cobra.Command{
    Use:   "set <key> <value>",
    Short: "Set a configuration value",
    Args:  cobra.ExactArgs(2),
    RunE: func(cmd *cobra.Command, args []string) error {
        viper.Set(args[0], args[1])
        return viper.WriteConfig()
    },
}

func init() {
    configCmd.AddCommand(configSetCmd)
}

// cmd/config_list.go
var configListCmd = &cobra.Command{
    Use:     "list",
    Aliases: []string{"ls"},
    Short:   "List all configuration values",
    Args:    cobra.NoArgs,
    Run: func(cmd *cobra.Command, args []string) {
        for _, key := range viper.AllKeys() {
            fmt.Printf("%s = %v\n", key, viper.Get(key))
        }
    },
}

func init() {
    configCmd.AddCommand(configListCmd)
}
```

This produces the following command tree:

```
taskctl config get <key>
taskctl config set <key> <value>
taskctl config list
```

### 3.8 PreRun and PostRun Hooks

Cobra commands support lifecycle hooks that run before and after the main `Run` function:

```go
var deployCmd = &cobra.Command{
    Use:   "deploy <environment>",
    Short: "Deploy the application",

    // PersistentPreRun runs for this command AND all its children
    PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
        // Check authentication
        token := viper.GetString("auth_token")
        if token == "" {
            return fmt.Errorf("not authenticated; run 'taskctl auth login' first")
        }
        return nil
    },

    // PreRun runs only for this specific command, after PersistentPreRun
    PreRunE: func(cmd *cobra.Command, args []string) error {
        env := args[0]
        if env == "production" {
            fmt.Print("Are you sure you want to deploy to production? (y/N): ")
            var answer string
            fmt.Scanln(&answer)
            if answer != "y" && answer != "Y" {
                return fmt.Errorf("deployment cancelled")
            }
        }
        return nil
    },

    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Printf("Deploying to %s...\n", args[0])
        // actual deployment logic
        return nil
    },

    // PostRun runs after Run completes successfully
    PostRun: func(cmd *cobra.Command, args []string) {
        fmt.Println("Deployment complete. Running post-deploy checks...")
    },
}
```

### 3.9 Shell Completions

Cobra generates shell completions for Bash, Zsh, Fish, and PowerShell:

```go
// cmd/completion.go
var completionCmd = &cobra.Command{
    Use:   "completion [bash|zsh|fish|powershell]",
    Short: "Generate shell completion scripts",
    Long: `Generate shell completion scripts for taskctl.

To load completions:

Bash:
  $ source <(taskctl completion bash)

  # To install permanently:
  $ taskctl completion bash > /etc/bash_completion.d/taskctl

Zsh:
  $ source <(taskctl completion zsh)

  # To install permanently (oh-my-zsh):
  $ taskctl completion zsh > ~/.oh-my-zsh/completions/_taskctl

Fish:
  $ taskctl completion fish | source

  # To install permanently:
  $ taskctl completion fish > ~/.config/fish/completions/taskctl.fish

PowerShell:
  PS> taskctl completion powershell | Out-String | Invoke-Expression

  # To install permanently, add the output to your profile.`,

    DisableFlagsInUseLine: true,
    ValidArgs:             []string{"bash", "zsh", "fish", "powershell"},
    Args:                  cobra.MatchAll(cobra.ExactArgs(1), cobra.OnlyValidArgs),

    Run: func(cmd *cobra.Command, args []string) {
        switch args[0] {
        case "bash":
            cmd.Root().GenBashCompletion(os.Stdout)
        case "zsh":
            cmd.Root().GenZshCompletion(os.Stdout)
        case "fish":
            cmd.Root().GenFishCompletion(os.Stdout, true)
        case "powershell":
            cmd.Root().GenPowerShellCompletionWithDesc(os.Stdout)
        }
    },
}

func init() {
    rootCmd.AddCommand(completionCmd)
}
```

#### Custom Completions for Arguments

```go
var addCmd = &cobra.Command{
    Use: "add <title>",
    // ValidArgsFunction provides completions for positional arguments
    ValidArgsFunction: func(
        cmd *cobra.Command, args []string, toComplete string,
    ) ([]string, cobra.ShellCompDirective) {
        // For the first argument, suggest nothing (free-form title)
        if len(args) == 0 {
            return nil, cobra.ShellCompDirectiveNoFileComp
        }
        return nil, cobra.ShellCompDirectiveNoFileComp
    },
    // ...
}

// Flag completion
func init() {
    addCmd.Flags().StringP("priority", "p", "medium", "Task priority")
    addCmd.RegisterFlagCompletionFunc("priority",
        func(cmd *cobra.Command, args []string, toComplete string,
        ) ([]string, cobra.ShellCompDirective) {
            priorities := []string{
                "low\tLow priority",
                "medium\tMedium priority (default)",
                "high\tHigh priority - needs attention",
            }
            return priorities, cobra.ShellCompDirectiveNoFileComp
        })

    addCmd.Flags().String("assignee", "", "Assign to team member")
    addCmd.RegisterFlagCompletionFunc("assignee",
        func(cmd *cobra.Command, args []string, toComplete string,
        ) ([]string, cobra.ShellCompDirective) {
            // In a real app, you'd load team members from a config or API
            members := []string{"alice", "bob", "charlie", "diana"}
            return members, cobra.ShellCompDirectiveNoFileComp
        })
}
```

### 3.10 Configuration with Viper Integration

Viper handles configuration from files, environment variables, flags, and remote systems.
The canonical precedence order (highest to lowest):

1. Explicit flag value on the command line
2. Environment variable
3. Configuration file
4. Key/value store (remote)
5. Default value

```go
// cmd/root.go -- extended configuration setup
func initConfig() {
    if cfgFile != "" {
        viper.SetConfigFile(cfgFile)
    } else {
        home, err := os.UserHomeDir()
        cobra.CheckErr(err)

        // Search for config in multiple locations
        viper.AddConfigPath(home)
        viper.AddConfigPath(".")
        viper.AddConfigPath("/etc/taskctl/")
        viper.SetConfigType("yaml")
        viper.SetConfigName(".taskctl")
    }

    // Environment variable binding
    viper.SetEnvPrefix("TASKCTL")
    viper.AutomaticEnv()

    // Replace hyphens with underscores in env var names
    // e.g., TASKCTL_STORE_PATH maps to "store-path" or "store_path"
    viper.SetEnvKeyReplacer(strings.NewReplacer("-", "_", ".", "_"))

    // Set defaults
    viper.SetDefault("format", "table")
    viper.SetDefault("store_path", filepath.Join(home(), ".taskctl", "tasks.json"))
    viper.SetDefault("editor", "vim")
    viper.SetDefault("color", true)
    viper.SetDefault("max_results", 50)

    // Read the config file
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            fmt.Fprintf(os.Stderr, "Error reading config: %s\n", err)
        }
    }
}
```

Example configuration file (`~/.taskctl.yaml`):

```yaml
# ~/.taskctl.yaml
format: table
color: true
verbose: false
store_path: ~/.taskctl/tasks.json
editor: nvim
max_results: 100

# API settings (for a hypothetical remote sync feature)
api:
  base_url: https://api.taskctl.dev
  timeout: 30s

# Default values for new tasks
defaults:
  priority: medium
  tags: []
```

#### Unmarshaling Config into Structs

```go
type APIConfig struct {
    BaseURL string        `mapstructure:"base_url"`
    Timeout time.Duration `mapstructure:"timeout"`
    Token   string        `mapstructure:"token"`
}

type AppConfig struct {
    Format     string    `mapstructure:"format"`
    Color      bool      `mapstructure:"color"`
    Verbose    bool      `mapstructure:"verbose"`
    StorePath  string    `mapstructure:"store_path"`
    MaxResults int       `mapstructure:"max_results"`
    API        APIConfig `mapstructure:"api"`
}

func loadConfig() (*AppConfig, error) {
    var cfg AppConfig
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshaling config: %w", err)
    }
    return &cfg, nil
}
```

### 3.11 Generating Documentation

Cobra can auto-generate documentation in multiple formats:

```go
// cmd/docs.go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
    "github.com/spf13/cobra/doc"
)

var docsCmd = &cobra.Command{
    Use:    "docs",
    Short:  "Generate documentation",
    Hidden: true, // Hide from normal help output
    RunE: func(cmd *cobra.Command, args []string) error {
        format, _ := cmd.Flags().GetString("format")
        dir, _ := cmd.Flags().GetString("dir")

        switch format {
        case "markdown":
            return doc.GenMarkdownTree(rootCmd, dir)
        case "man":
            header := &doc.GenManHeader{
                Title:   "TASKCTL",
                Section: "1",
            }
            return doc.GenManTree(rootCmd, header, dir)
        case "yaml":
            return doc.GenYamlTree(rootCmd, dir)
        case "rst":
            return doc.GenReSTTree(rootCmd, dir)
        default:
            return fmt.Errorf("unsupported format: %s", format)
        }
    },
}

func init() {
    rootCmd.AddCommand(docsCmd)
    docsCmd.Flags().String("format", "markdown", "Output format (markdown, man, yaml, rst)")
    docsCmd.Flags().String("dir", "./docs", "Output directory")
}
```

```bash
# Generate markdown documentation
taskctl docs --format markdown --dir ./docs

# Generate man pages
taskctl docs --format man --dir /usr/local/share/man/man1/
```

### 3.12 Building a Real CLI Tool: API Client

Here is a more complex example -- a REST API client CLI:

```go
// cmd/api.go
package cmd

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strings"
    "time"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var apiCmd = &cobra.Command{
    Use:   "api",
    Short: "Make API requests",
    Long:  "Interact with the taskctl remote API.",
}

var apiGetCmd = &cobra.Command{
    Use:   "get <path>",
    Short: "Make a GET request",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        return doRequest("GET", args[0], nil)
    },
}

var apiPostCmd = &cobra.Command{
    Use:   "post <path>",
    Short: "Make a POST request",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        data, _ := cmd.Flags().GetString("data")
        file, _ := cmd.Flags().GetString("file")

        var body io.Reader
        if file != "" {
            f, err := os.Open(file)
            if err != nil {
                return fmt.Errorf("opening file: %w", err)
            }
            defer f.Close()
            body = f
        } else if data != "" {
            body = strings.NewReader(data)
        }

        return doRequest("POST", args[0], body)
    },
}

func doRequest(method, path string, body io.Reader) error {
    baseURL := viper.GetString("api.base_url")
    token := viper.GetString("api.token")
    timeout := viper.GetDuration("api.timeout")

    if baseURL == "" {
        return fmt.Errorf("api.base_url not configured")
    }

    url := fmt.Sprintf("%s/%s", strings.TrimRight(baseURL, "/"), strings.TrimLeft(path, "/"))

    req, err := http.NewRequest(method, url, body)
    if err != nil {
        return fmt.Errorf("creating request: %w", err)
    }

    req.Header.Set("Content-Type", "application/json")
    if token != "" {
        req.Header.Set("Authorization", "Bearer "+token)
    }

    client := &http.Client{Timeout: timeout}
    if timeout == 0 {
        client.Timeout = 30 * time.Second
    }

    resp, err := client.Do(req)
    if err != nil {
        return fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return fmt.Errorf("reading response: %w", err)
    }

    // Pretty-print JSON response
    var prettyJSON bytes.Buffer
    if err := json.Indent(&prettyJSON, respBody, "", "  "); err != nil {
        // Not JSON, print raw
        fmt.Println(string(respBody))
    } else {
        fmt.Println(prettyJSON.String())
    }

    if resp.StatusCode >= 400 {
        return fmt.Errorf("API returned status %d", resp.StatusCode)
    }

    return nil
}

func init() {
    apiCmd.AddCommand(apiGetCmd)
    apiCmd.AddCommand(apiPostCmd)
    rootCmd.AddCommand(apiCmd)

    apiPostCmd.Flags().StringP("data", "d", "", "Request body (JSON string)")
    apiPostCmd.Flags().StringP("file", "f", "", "Read request body from file")
    apiPostCmd.MarkFlagsMutuallyExclusive("data", "file")
}
```

### 3.13 Version Command Pattern

A clean version command with build-time injection:

```go
// internal/version/version.go
package version

import "fmt"

// These are set at build time using -ldflags.
var (
    Version   = "dev"
    Commit    = "none"
    Date      = "unknown"
    GoVersion = "unknown"
)

func Full() string {
    return fmt.Sprintf("taskctl %s (commit: %s, built: %s, go: %s)",
        Version, Commit, Date, GoVersion)
}
```

```go
// cmd/version.go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
    "github.com/yourname/taskctl/internal/version"
)

var versionCmd = &cobra.Command{
    Use:   "version",
    Short: "Print the version",
    Run: func(cmd *cobra.Command, args []string) {
        short, _ := cmd.Flags().GetBool("short")
        if short {
            fmt.Println(version.Version)
        } else {
            fmt.Println(version.Full())
        }
    },
}

func init() {
    rootCmd.AddCommand(versionCmd)
    versionCmd.Flags().Bool("short", false, "Print version number only")
}
```

Build with version info:

```bash
go build -ldflags "\
  -X github.com/yourname/taskctl/internal/version.Version=1.2.3 \
  -X github.com/yourname/taskctl/internal/version.Commit=$(git rev-parse --short HEAD) \
  -X github.com/yourname/taskctl/internal/version.Date=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -X github.com/yourname/taskctl/internal/version.GoVersion=$(go version | cut -d' ' -f3)" \
  -o taskctl .
```

---

## 4. Bubbletea TUI Framework

[Bubbletea](https://github.com/charmbracelet/bubbletea) is a Go framework for building terminal
user interfaces (TUIs) based on the Elm Architecture. It provides a clean, functional approach to
managing state and rendering in the terminal.

### 4.1 The Elm Architecture

Bubbletea is built on three concepts from the Elm language:

1. **Model** -- The application state. A struct that holds everything your UI needs.
2. **Update** -- A function that takes the current model and a message, and returns a new model.
   This is the only place where state changes.
3. **View** -- A function that takes the current model and returns a string to render.

```
User Input (KeyPress, MouseClick, etc.)
    |
    v
+--------+     +--------+     +------+
| Update | --> |  Model | --> | View | --> Terminal Output
+--------+     +--------+     +------+
    ^                                        |
    |________________________________________|
              (next message from user)
```

This architecture makes TUIs predictable and testable. There is no mutable shared state, no
callbacks modifying global variables, and no race conditions.

### 4.2 Hello Bubbletea

```go
// Install: go get github.com/charmbracelet/bubbletea

package main

import (
    "fmt"
    "os"

    tea "github.com/charmbracelet/bubbletea"
)

// Model holds the application state.
type model struct {
    choices  []string
    cursor   int
    selected map[int]struct{}
}

// initialModel returns the starting state.
func initialModel() model {
    return model{
        choices: []string{
            "Buy groceries",
            "Walk the dog",
            "Write code",
            "Read a book",
            "Exercise",
        },
        selected: make(map[int]struct{}),
    }
}

// Init returns an initial command (or nil for no command).
func (m model) Init() tea.Cmd {
    return nil // No initial I/O
}

// Update handles messages and returns the updated model.
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "ctrl+c", "q":
            return m, tea.Quit

        case "up", "k":
            if m.cursor > 0 {
                m.cursor--
            }

        case "down", "j":
            if m.cursor < len(m.choices)-1 {
                m.cursor++
            }

        case " ", "enter":
            if _, ok := m.selected[m.cursor]; ok {
                delete(m.selected, m.cursor)
            } else {
                m.selected[m.cursor] = struct{}{}
            }
        }
    }

    return m, nil
}

// View renders the UI as a string.
func (m model) View() string {
    s := "What would you like to do today?\n\n"

    for i, choice := range m.choices {
        cursor := "  " // no cursor
        if m.cursor == i {
            cursor = "> " // cursor
        }

        checked := "[ ]"
        if _, ok := m.selected[i]; ok {
            checked = "[x]"
        }

        s += fmt.Sprintf("%s%s %s\n", cursor, checked, choice)
    }

    s += "\nPress space to select, q to quit.\n"

    return s
}

func main() {
    p := tea.NewProgram(initialModel())
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

### 4.3 Commands and Messages

In Bubbletea, side effects (I/O, timers, HTTP requests) are handled through **Commands** (`tea.Cmd`).
A command is a function that returns a **Message** (`tea.Msg`). The runtime executes commands
asynchronously and delivers their messages to `Update`.

```go
package main

import (
    "fmt"
    "net/http"
    "os"
    "time"

    tea "github.com/charmbracelet/bubbletea"
)

// Custom message types
type statusMsg int
type errMsg struct{ err error }

func (e errMsg) Error() string { return e.err.Error() }

type model struct {
    url      string
    status   int
    err      error
    loading  bool
    quitting bool
}

// checkStatus is a command that makes an HTTP request.
func checkStatus(url string) tea.Cmd {
    return func() tea.Msg {
        client := &http.Client{Timeout: 10 * time.Second}
        resp, err := client.Get(url)
        if err != nil {
            return errMsg{err}
        }
        defer resp.Body.Close()
        return statusMsg(resp.StatusCode)
    }
}

func initialModel(url string) model {
    return model{url: url, loading: true}
}

func (m model) Init() tea.Cmd {
    // Start the HTTP request when the program starts
    return checkStatus(m.url)
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case statusMsg:
        m.status = int(msg)
        m.loading = false
        return m, nil

    case errMsg:
        m.err = msg.err
        m.loading = false
        return m, nil

    case tea.KeyMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            m.quitting = true
            return m, tea.Quit
        case "r":
            m.loading = true
            return m, checkStatus(m.url)
        }
    }

    return m, nil
}

func (m model) View() string {
    if m.quitting {
        return ""
    }

    if m.loading {
        return fmt.Sprintf("Checking %s...\n", m.url)
    }

    if m.err != nil {
        return fmt.Sprintf("Error: %v\n\nPress r to retry, q to quit.\n", m.err)
    }

    return fmt.Sprintf("Status: %d\n\nPress r to refresh, q to quit.\n", m.status)
}

func main() {
    url := "https://httpbin.org/status/200"
    if len(os.Args) > 1 {
        url = os.Args[1]
    }

    p := tea.NewProgram(initialModel(url))
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

### 4.4 Tick Commands (Timers and Animation)

```go
package main

import (
    "fmt"
    "os"
    "time"

    tea "github.com/charmbracelet/bubbletea"
)

type tickMsg time.Time

func tickEvery(d time.Duration) tea.Cmd {
    return tea.Every(d, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}

type model struct {
    elapsed  time.Duration
    running  bool
    quitting bool
}

func (m model) Init() tea.Cmd {
    return tickEvery(time.Second)
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tickMsg:
        if m.running {
            m.elapsed += time.Second
        }
        return m, tickEvery(time.Second)

    case tea.KeyMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            m.quitting = true
            return m, tea.Quit
        case " ":
            m.running = !m.running
        case "r":
            m.elapsed = 0
            m.running = false
        }
    }

    return m, nil
}

func (m model) View() string {
    if m.quitting {
        return fmt.Sprintf("Final time: %s\n", m.elapsed)
    }

    status := "PAUSED"
    if m.running {
        status = "RUNNING"
    }

    return fmt.Sprintf(
        "\n  Stopwatch [%s]\n\n  %s\n\n  space: start/stop  r: reset  q: quit\n",
        status, m.elapsed,
    )
}

func main() {
    p := tea.NewProgram(model{})
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

### 4.5 Bubbles: The Component Library

[Bubbles](https://github.com/charmbracelet/bubbles) is the official companion library of reusable
components for Bubbletea. Each component follows the same Model/Update/View pattern.

#### Text Input

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/charmbracelet/bubbles/textinput"
    tea "github.com/charmbracelet/bubbletea"
)

type model struct {
    textInput textinput.Model
    err       error
}

func initialModel() model {
    ti := textinput.New()
    ti.Placeholder = "Enter your name"
    ti.Focus()
    ti.CharLimit = 50
    ti.Width = 30

    return model{textInput: ti}
}

func (m model) Init() tea.Cmd {
    return textinput.Blink
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd

    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.Type {
        case tea.KeyCtrlC, tea.KeyEsc:
            return m, tea.Quit
        case tea.KeyEnter:
            return m, tea.Quit
        }
    }

    m.textInput, cmd = m.textInput.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return fmt.Sprintf(
        "What's your name?\n\n%s\n\n%s",
        m.textInput.View(),
        "(esc to quit)",
    ) + "\n"
}

func main() {
    p := tea.NewProgram(initialModel())
    m, err := p.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    finalModel := m.(model)
    name := strings.TrimSpace(finalModel.textInput.Value())
    if name != "" {
        fmt.Printf("\nHello, %s!\n", name)
    }
}
```

#### Multi-Field Form

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/charmbracelet/bubbles/textinput"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

var (
    focusedStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))
    blurredStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("240"))
    noStyle      = lipgloss.NewStyle()
)

type model struct {
    inputs     []textinput.Model
    focusIndex int
    submitted  bool
}

func initialModel() model {
    inputs := make([]textinput.Model, 4)

    inputs[0] = textinput.New()
    inputs[0].Placeholder = "Task title"
    inputs[0].Focus()
    inputs[0].PromptStyle = focusedStyle
    inputs[0].TextStyle = focusedStyle

    inputs[1] = textinput.New()
    inputs[1].Placeholder = "Description"

    inputs[2] = textinput.New()
    inputs[2].Placeholder = "Priority (low/medium/high)"

    inputs[3] = textinput.New()
    inputs[3].Placeholder = "Tags (comma-separated)"

    return model{inputs: inputs}
}

func (m model) Init() tea.Cmd {
    return textinput.Blink
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "ctrl+c", "esc":
            return m, tea.Quit

        case "tab", "shift+tab", "up", "down", "enter":
            s := msg.String()

            // Submit on Enter at the last field
            if s == "enter" && m.focusIndex == len(m.inputs)-1 {
                m.submitted = true
                return m, tea.Quit
            }

            // Move focus
            if s == "up" || s == "shift+tab" {
                m.focusIndex--
            } else {
                m.focusIndex++
            }

            // Wrap around
            if m.focusIndex >= len(m.inputs) {
                m.focusIndex = 0
            } else if m.focusIndex < 0 {
                m.focusIndex = len(m.inputs) - 1
            }

            cmds := make([]tea.Cmd, len(m.inputs))
            for i := range m.inputs {
                if i == m.focusIndex {
                    cmds[i] = m.inputs[i].Focus()
                    m.inputs[i].PromptStyle = focusedStyle
                    m.inputs[i].TextStyle = focusedStyle
                } else {
                    m.inputs[i].Blur()
                    m.inputs[i].PromptStyle = noStyle
                    m.inputs[i].TextStyle = noStyle
                }
            }

            return m, tea.Batch(cmds...)
        }
    }

    // Update the focused input
    cmd := m.updateInputs(msg)
    return m, cmd
}

func (m *model) updateInputs(msg tea.Msg) tea.Cmd {
    cmds := make([]tea.Cmd, len(m.inputs))
    for i := range m.inputs {
        m.inputs[i], cmds[i] = m.inputs[i].Update(msg)
    }
    return tea.Batch(cmds...)
}

func (m model) View() string {
    var b strings.Builder

    b.WriteString("\n  New Task\n\n")

    for i := range m.inputs {
        b.WriteString("  " + m.inputs[i].View() + "\n")
    }

    b.WriteString("\n  tab: next field  enter: submit  esc: cancel\n")

    return b.String()
}

func main() {
    p := tea.NewProgram(initialModel())
    result, err := p.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    m := result.(model)
    if m.submitted {
        fmt.Printf("\nTask created:\n")
        fmt.Printf("  Title:       %s\n", m.inputs[0].Value())
        fmt.Printf("  Description: %s\n", m.inputs[1].Value())
        fmt.Printf("  Priority:    %s\n", m.inputs[2].Value())
        fmt.Printf("  Tags:        %s\n", m.inputs[3].Value())
    }
}
```

#### List Component

```go
package main

import (
    "fmt"
    "os"

    "github.com/charmbracelet/bubbles/list"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

var docStyle = lipgloss.NewStyle().Margin(1, 2)

// item implements list.Item interface
type item struct {
    title       string
    description string
}

func (i item) Title() string       { return i.title }
func (i item) Description() string { return i.description }
func (i item) FilterValue() string { return i.title }

type model struct {
    list list.Model
}

func initialModel() model {
    items := []list.Item{
        item{title: "Buy groceries", description: "Milk, eggs, bread, cheese"},
        item{title: "Walk the dog", description: "At least 30 minutes in the park"},
        item{title: "Write code", description: "Finish the CLI chapter"},
        item{title: "Read a book", description: "Continue 'Designing Data-Intensive Applications'"},
        item{title: "Exercise", description: "30 min cardio + strength training"},
        item{title: "Cook dinner", description: "Try the new pasta recipe"},
        item{title: "Clean house", description: "Vacuum and mop the floors"},
        item{title: "Call dentist", description: "Schedule annual checkup"},
    }

    l := list.New(items, list.NewDefaultDelegate(), 0, 0)
    l.Title = "My Tasks"
    l.SetShowStatusBar(true)
    l.SetFilteringEnabled(true)

    return model{list: l}
}

func (m model) Init() tea.Cmd {
    return nil
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        h, v := docStyle.GetFrameSize()
        m.list.SetSize(msg.Width-h, msg.Height-v)

    case tea.KeyMsg:
        switch msg.String() {
        case "ctrl+c":
            return m, tea.Quit
        case "enter":
            selected := m.list.SelectedItem()
            if selected != nil {
                i := selected.(item)
                fmt.Printf("Selected: %s\n", i.title)
            }
            return m, tea.Quit
        }
    }

    var cmd tea.Cmd
    m.list, cmd = m.list.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return docStyle.Render(m.list.View())
}

func main() {
    p := tea.NewProgram(initialModel(), tea.WithAltScreen())
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

#### Table Component

```go
package main

import (
    "fmt"
    "os"

    "github.com/charmbracelet/bubbles/table"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

var baseStyle = lipgloss.NewStyle().
    BorderStyle(lipgloss.NormalBorder()).
    BorderForeground(lipgloss.Color("240"))

type model struct {
    table table.Model
}

func initialModel() model {
    columns := []table.Column{
        {Title: "ID", Width: 5},
        {Title: "Status", Width: 10},
        {Title: "Priority", Width: 10},
        {Title: "Title", Width: 30},
        {Title: "Due Date", Width: 12},
    }

    rows := []table.Row{
        {"1", "pending", "high", "Fix authentication bug", "2026-03-25"},
        {"2", "pending", "high", "Deploy v2.0", "2026-03-26"},
        {"3", "done", "medium", "Write unit tests", "2026-03-20"},
        {"4", "pending", "medium", "Update documentation", "2026-03-28"},
        {"5", "pending", "low", "Refactor logger", "2026-04-01"},
        {"6", "done", "high", "Security audit", "2026-03-18"},
        {"7", "pending", "medium", "Add metrics endpoint", "2026-03-30"},
        {"8", "pending", "low", "Clean up dead code", "2026-04-05"},
    }

    t := table.New(
        table.WithColumns(columns),
        table.WithRows(rows),
        table.WithFocused(true),
        table.WithHeight(10),
    )

    s := table.DefaultStyles()
    s.Header = s.Header.
        BorderStyle(lipgloss.NormalBorder()).
        BorderForeground(lipgloss.Color("240")).
        BorderBottom(true).
        Bold(true)
    s.Selected = s.Selected.
        Foreground(lipgloss.Color("229")).
        Background(lipgloss.Color("57")).
        Bold(false)
    t.SetStyles(s)

    return model{table: t}
}

func (m model) Init() tea.Cmd {
    return nil
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            return m, tea.Quit
        case "enter":
            row := m.table.SelectedRow()
            fmt.Printf("Selected task #%s: %s\n", row[0], row[3])
            return m, tea.Quit
        }
    }

    var cmd tea.Cmd
    m.table, cmd = m.table.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return baseStyle.Render(m.table.View()) + "\n  q: quit  enter: select\n"
}

func main() {
    p := tea.NewProgram(initialModel())
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

#### Spinner Component

```go
package main

import (
    "fmt"
    "os"
    "time"

    "github.com/charmbracelet/bubbles/spinner"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

type doneMsg struct{}

type model struct {
    spinner  spinner.Model
    message  string
    done     bool
    quitting bool
}

func initialModel() model {
    s := spinner.New()
    s.Spinner = spinner.Dot
    s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))

    return model{
        spinner: s,
        message: "Loading data...",
    }
}

// Simulated long-running operation
func doWork() tea.Cmd {
    return func() tea.Msg {
        time.Sleep(3 * time.Second) // Simulate work
        return doneMsg{}
    }
}

func (m model) Init() tea.Cmd {
    return tea.Batch(m.spinner.Tick, doWork())
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case doneMsg:
        m.done = true
        return m, tea.Quit

    case tea.KeyMsg:
        if msg.String() == "ctrl+c" {
            m.quitting = true
            return m, tea.Quit
        }

    case spinner.TickMsg:
        var cmd tea.Cmd
        m.spinner, cmd = m.spinner.Update(msg)
        return m, cmd
    }

    return m, nil
}

func (m model) View() string {
    if m.done {
        return "Done! Data loaded successfully.\n"
    }
    if m.quitting {
        return "Cancelled.\n"
    }
    return fmt.Sprintf("\n  %s %s\n\n", m.spinner.View(), m.message)
}

func main() {
    p := tea.NewProgram(initialModel())
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

### 4.6 Lip Gloss: Terminal Styling

[Lip Gloss](https://github.com/charmbracelet/lipgloss) is a CSS-like styling library for terminal
output. It handles colors, borders, padding, margins, alignment, and layout composition.

```go
package main

import (
    "fmt"

    "github.com/charmbracelet/lipgloss"
)

func main() {
    // Basic styles
    title := lipgloss.NewStyle().
        Bold(true).
        Foreground(lipgloss.Color("#FAFAFA")).
        Background(lipgloss.Color("#7D56F4")).
        PaddingLeft(2).
        PaddingRight(2)

    subtitle := lipgloss.NewStyle().
        Foreground(lipgloss.Color("#888888")).
        Italic(true)

    highlight := lipgloss.NewStyle().
        Foreground(lipgloss.Color("#FF6B6B")).
        Bold(true)

    // Box with borders
    infoBox := lipgloss.NewStyle().
        Border(lipgloss.RoundedBorder()).
        BorderForeground(lipgloss.Color("#874BFD")).
        Padding(1, 2).
        Width(50)

    // Success/Error/Warning styles
    success := lipgloss.NewStyle().
        Foreground(lipgloss.Color("#04B575")).
        Bold(true).
        SetString("SUCCESS")

    errorStyle := lipgloss.NewStyle().
        Foreground(lipgloss.Color("#FF0000")).
        Bold(true).
        SetString("ERROR")

    warning := lipgloss.NewStyle().
        Foreground(lipgloss.Color("#FFAA00")).
        Bold(true).
        SetString("WARNING")

    fmt.Println(title.Render("Task Manager Dashboard"))
    fmt.Println(subtitle.Render("Manage your tasks efficiently"))
    fmt.Println()

    fmt.Println(infoBox.Render(
        highlight.Render("5 tasks") + " pending\n" +
            highlight.Render("3 tasks") + " completed today\n" +
            highlight.Render("2 tasks") + " overdue",
    ))
    fmt.Println()

    fmt.Printf("[%s] All tests passed\n", success)
    fmt.Printf("[%s] Connection timeout\n", errorStyle)
    fmt.Printf("[%s] Disk usage above 80%%\n", warning)
}
```

#### Layout Composition with Lip Gloss

```go
package main

import (
    "fmt"
    "strings"

    "github.com/charmbracelet/lipgloss"
)

func main() {
    width := 80

    // Define column styles
    leftCol := lipgloss.NewStyle().
        Width(width / 2).
        Padding(1, 2).
        Border(lipgloss.RoundedBorder()).
        BorderForeground(lipgloss.Color("63"))

    rightCol := lipgloss.NewStyle().
        Width(width / 2).
        Padding(1, 2).
        Border(lipgloss.RoundedBorder()).
        BorderForeground(lipgloss.Color("205"))

    header := lipgloss.NewStyle().
        Bold(true).
        Foreground(lipgloss.Color("229")).
        Background(lipgloss.Color("63")).
        Width(width).
        Align(lipgloss.Center).
        Padding(0, 1)

    footer := lipgloss.NewStyle().
        Foreground(lipgloss.Color("240")).
        Width(width).
        Align(lipgloss.Center)

    // Content
    leftContent := "Pending Tasks\n\n" +
        "1. Fix authentication bug\n" +
        "2. Deploy v2.0\n" +
        "3. Update documentation\n" +
        "4. Add metrics endpoint"

    rightContent := "Completed Today\n\n" +
        "1. Write unit tests\n" +
        "2. Security audit\n" +
        "3. Code review"

    // Compose layout
    // JoinHorizontal places blocks side by side
    columns := lipgloss.JoinHorizontal(
        lipgloss.Top,
        leftCol.Render(leftContent),
        rightCol.Render(rightContent),
    )

    // JoinVertical stacks blocks
    page := lipgloss.JoinVertical(
        lipgloss.Left,
        header.Render("Task Dashboard"),
        columns,
        footer.Render(strings.Repeat("-", width)),
        footer.Render("Press q to quit | Tab to switch panels"),
    )

    fmt.Println(page)
}
```

#### Adaptive Colors

Lip Gloss can adapt colors based on the terminal's background:

```go
// AdaptiveColor picks a color based on whether the terminal has a light or
// dark background.
adaptive := lipgloss.NewStyle().
    Foreground(lipgloss.AdaptiveColor{
        Light: "#333333", // Used on light backgrounds
        Dark:  "#DDDDDD", // Used on dark backgrounds
    })

// CompleteAdaptiveColor allows specifying ANSI, ANSI256, and TrueColor
// variants for maximum compatibility.
compatible := lipgloss.NewStyle().
    Foreground(lipgloss.CompleteAdaptiveColor{
        Light: lipgloss.CompleteColor{
            TrueColor: "#333333",
            ANSI256:   "235",
            ANSI:      "0",
        },
        Dark: lipgloss.CompleteColor{
            TrueColor: "#DDDDDD",
            ANSI256:   "252",
            ANSI:      "7",
        },
    })
```

### 4.7 Building a Complete TUI App: File Browser

Here is a full-featured file browser TUI combining Bubbletea, Bubbles, and Lip Gloss:

```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
    "sort"
    "strings"

    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

// Styles
var (
    titleStyle = lipgloss.NewStyle().
            Bold(true).
            Foreground(lipgloss.Color("#FAFAFA")).
            Background(lipgloss.Color("#7D56F4")).
            Padding(0, 1).
            MarginBottom(1)

    selectedStyle = lipgloss.NewStyle().
            Foreground(lipgloss.Color("#FFFFFF")).
            Background(lipgloss.Color("#7D56F4")).
            Bold(true)

    dirStyle = lipgloss.NewStyle().
            Foreground(lipgloss.Color("#7D56F4")).
            Bold(true)

    fileStyle = lipgloss.NewStyle().
            Foreground(lipgloss.Color("#FAFAFA"))

    infoStyle = lipgloss.NewStyle().
            Foreground(lipgloss.Color("#888888"))

    helpStyle = lipgloss.NewStyle().
            Foreground(lipgloss.Color("#626262")).
            MarginTop(1)

    errorStyle = lipgloss.NewStyle().
            Foreground(lipgloss.Color("#FF0000"))
)

type fileEntry struct {
    name  string
    isDir bool
    size  int64
    info  os.FileInfo
}

type model struct {
    currentDir string
    entries    []fileEntry
    cursor     int
    offset     int // for scrolling
    height     int
    width      int
    err        error
    history    []string // directory history for "back"
}

func initialModel(dir string) model {
    m := model{
        currentDir: dir,
        height:     20,
        width:      80,
    }
    m.loadDir()
    return m
}

func (m *model) loadDir() {
    entries, err := os.ReadDir(m.currentDir)
    if err != nil {
        m.err = err
        return
    }

    m.entries = nil
    m.err = nil

    // Add parent directory entry
    if m.currentDir != "/" {
        m.entries = append(m.entries, fileEntry{
            name:  "..",
            isDir: true,
        })
    }

    var dirs, files []fileEntry

    for _, e := range entries {
        info, err := e.Info()
        if err != nil {
            continue
        }
        fe := fileEntry{
            name:  e.Name(),
            isDir: e.IsDir(),
            size:  info.Size(),
            info:  info,
        }
        if e.IsDir() {
            dirs = append(dirs, fe)
        } else {
            files = append(files, fe)
        }
    }

    // Sort: directories first, then files, both alphabetical
    sort.Slice(dirs, func(i, j int) bool {
        return strings.ToLower(dirs[i].name) < strings.ToLower(dirs[j].name)
    })
    sort.Slice(files, func(i, j int) bool {
        return strings.ToLower(files[i].name) < strings.ToLower(files[j].name)
    })

    m.entries = append(m.entries, dirs...)
    m.entries = append(m.entries, files...)
    m.cursor = 0
    m.offset = 0
}

func (m model) Init() tea.Cmd {
    return nil
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.height = msg.Height - 6 // Leave room for header and footer
        m.width = msg.Width

    case tea.KeyMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            return m, tea.Quit

        case "up", "k":
            if m.cursor > 0 {
                m.cursor--
                if m.cursor < m.offset {
                    m.offset = m.cursor
                }
            }

        case "down", "j":
            if m.cursor < len(m.entries)-1 {
                m.cursor++
                if m.cursor >= m.offset+m.height {
                    m.offset = m.cursor - m.height + 1
                }
            }

        case "enter", "right", "l":
            if len(m.entries) == 0 {
                break
            }
            entry := m.entries[m.cursor]
            if entry.isDir {
                var newDir string
                if entry.name == ".." {
                    newDir = filepath.Dir(m.currentDir)
                } else {
                    newDir = filepath.Join(m.currentDir, entry.name)
                }
                m.history = append(m.history, m.currentDir)
                m.currentDir = newDir
                m.loadDir()
            }

        case "left", "h", "backspace":
            if len(m.history) > 0 {
                m.currentDir = m.history[len(m.history)-1]
                m.history = m.history[:len(m.history)-1]
                m.loadDir()
            } else if m.currentDir != "/" {
                m.history = append(m.history, m.currentDir)
                m.currentDir = filepath.Dir(m.currentDir)
                m.loadDir()
            }

        case "home", "g":
            m.cursor = 0
            m.offset = 0

        case "end", "G":
            m.cursor = len(m.entries) - 1
            if m.cursor >= m.height {
                m.offset = m.cursor - m.height + 1
            }
        }
    }

    return m, nil
}

func (m model) View() string {
    var b strings.Builder

    // Title bar
    b.WriteString(titleStyle.Render(fmt.Sprintf(" %s ", m.currentDir)))
    b.WriteString("\n")

    if m.err != nil {
        b.WriteString(errorStyle.Render(fmt.Sprintf("Error: %v", m.err)))
        b.WriteString("\n")
        b.WriteString(helpStyle.Render("backspace: go back  q: quit"))
        return b.String()
    }

    if len(m.entries) == 0 {
        b.WriteString(infoStyle.Render("(empty directory)"))
        b.WriteString("\n")
    }

    // File list with scrolling
    end := m.offset + m.height
    if end > len(m.entries) {
        end = len(m.entries)
    }

    for i := m.offset; i < end; i++ {
        entry := m.entries[i]

        // Format the entry
        var line string
        if entry.isDir {
            line = dirStyle.Render(fmt.Sprintf("  %s/", entry.name))
        } else {
            size := formatSize(entry.size)
            name := fileStyle.Render(fmt.Sprintf("  %s", entry.name))
            line = fmt.Sprintf("%-40s %s", name, infoStyle.Render(size))
        }

        // Highlight selected entry
        if i == m.cursor {
            if entry.isDir {
                line = selectedStyle.Render(fmt.Sprintf("> %s/", entry.name))
            } else {
                line = selectedStyle.Render(fmt.Sprintf("> %s", entry.name))
            }
        }

        b.WriteString(line + "\n")
    }

    // Scroll indicator
    if len(m.entries) > m.height {
        pct := float64(m.cursor) / float64(len(m.entries)-1) * 100
        b.WriteString(infoStyle.Render(
            fmt.Sprintf("\n  %d/%d (%.0f%%)", m.cursor+1, len(m.entries), pct)))
    }

    // Help
    b.WriteString(helpStyle.Render(
        "\n  arrows/hjkl: navigate  enter: open dir  backspace: back  q: quit"))

    return b.String()
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

func main() {
    startDir, err := os.Getwd()
    if err != nil {
        startDir = "/"
    }
    if len(os.Args) > 1 {
        startDir = os.Args[1]
    }

    p := tea.NewProgram(
        initialModel(startDir),
        tea.WithAltScreen(),       // Use alternate screen buffer
        tea.WithMouseCellMotion(), // Enable mouse support
    )

    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

### 4.8 Building a Dashboard TUI

A more complex example combining multiple Bubbles components:

```go
package main

import (
    "fmt"
    "math/rand"
    "os"
    "strings"
    "time"

    "github.com/charmbracelet/bubbles/progress"
    "github.com/charmbracelet/bubbles/spinner"
    "github.com/charmbracelet/bubbles/table"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
)

// Styles
var (
    panelStyle = lipgloss.NewStyle().
            Border(lipgloss.RoundedBorder()).
            BorderForeground(lipgloss.Color("62")).
            Padding(1, 2)

    headerStyle = lipgloss.NewStyle().
            Bold(true).
            Foreground(lipgloss.Color("229")).
            Background(lipgloss.Color("62")).
            Width(80).
            Align(lipgloss.Center).
            Padding(0, 1)

    statLabel = lipgloss.NewStyle().
            Foreground(lipgloss.Color("241"))

    statValue = lipgloss.NewStyle().
            Bold(true).
            Foreground(lipgloss.Color("229"))

    greenText = lipgloss.NewStyle().
            Foreground(lipgloss.Color("42"))

    redText = lipgloss.NewStyle().
            Foreground(lipgloss.Color("196"))
)

type tickMsg time.Time

type serverStatus struct {
    name       string
    status     string
    cpu        float64
    memory     float64
    requests   int
    errorRate  float64
}

type model struct {
    servers    []serverStatus
    spinner    spinner.Model
    cpuBar     progress.Model
    memBar     progress.Model
    table      table.Model
    width      int
    height     int
    lastUpdate time.Time
}

func tick() tea.Cmd {
    return tea.Every(2*time.Second, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}

func initialModel() model {
    // Spinner
    s := spinner.New()
    s.Spinner = spinner.Pulse
    s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))

    // Progress bars
    cpuBar := progress.New(progress.WithDefaultGradient())
    memBar := progress.New(progress.WithGradient("#5A56E0", "#EE6FF8"))

    // Table
    columns := []table.Column{
        {Title: "Server", Width: 15},
        {Title: "Status", Width: 10},
        {Title: "CPU", Width: 8},
        {Title: "Memory", Width: 8},
        {Title: "Req/s", Width: 8},
        {Title: "Errors", Width: 8},
    }

    t := table.New(
        table.WithColumns(columns),
        table.WithFocused(true),
        table.WithHeight(6),
    )

    tStyle := table.DefaultStyles()
    tStyle.Header = tStyle.Header.
        BorderStyle(lipgloss.NormalBorder()).
        BorderBottom(true).
        Bold(true)
    t.SetStyles(tStyle)

    servers := generateServers()

    m := model{
        servers:    servers,
        spinner:    s,
        cpuBar:     cpuBar,
        memBar:     memBar,
        table:      t,
        width:      80,
        height:     24,
        lastUpdate: time.Now(),
    }
    m.updateTable()

    return m
}

func generateServers() []serverStatus {
    names := []string{"web-01", "web-02", "api-01", "api-02", "db-01", "cache-01"}
    servers := make([]serverStatus, len(names))
    for i, name := range names {
        servers[i] = serverStatus{
            name:      name,
            status:    "healthy",
            cpu:       rand.Float64() * 100,
            memory:    30 + rand.Float64()*60,
            requests:  rand.Intn(5000),
            errorRate: rand.Float64() * 5,
        }
        if rand.Float64() < 0.1 {
            servers[i].status = "degraded"
        }
    }
    return servers
}

func (m *model) updateTable() {
    rows := make([]table.Row, len(m.servers))
    for i, s := range m.servers {
        rows[i] = table.Row{
            s.name,
            s.status,
            fmt.Sprintf("%.1f%%", s.cpu),
            fmt.Sprintf("%.1f%%", s.memory),
            fmt.Sprintf("%d", s.requests),
            fmt.Sprintf("%.2f%%", s.errorRate),
        }
    }
    m.table.SetRows(rows)
}

func (m model) Init() tea.Cmd {
    return tea.Batch(m.spinner.Tick, tick())
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height

    case tickMsg:
        m.servers = generateServers()
        m.updateTable()
        m.lastUpdate = time.Now()
        return m, tick()

    case spinner.TickMsg:
        var cmd tea.Cmd
        m.spinner, cmd = m.spinner.Update(msg)
        return m, cmd

    case tea.KeyMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            return m, tea.Quit
        }
    }

    var cmd tea.Cmd
    m.table, cmd = m.table.Update(msg)
    return m, cmd
}

func (m model) View() string {
    var b strings.Builder

    // Header
    b.WriteString(headerStyle.Render("Server Dashboard"))
    b.WriteString("\n\n")

    // Stats summary
    totalRequests := 0
    avgCPU := 0.0
    avgMem := 0.0
    healthy := 0
    for _, s := range m.servers {
        totalRequests += s.requests
        avgCPU += s.cpu
        avgMem += s.memory
        if s.status == "healthy" {
            healthy++
        }
    }
    avgCPU /= float64(len(m.servers))
    avgMem /= float64(len(m.servers))

    stats := lipgloss.JoinHorizontal(lipgloss.Top,
        panelStyle.Width(18).Render(
            statLabel.Render("Servers")+"\n"+
                statValue.Render(fmt.Sprintf("%d/%d", healthy, len(m.servers)))),
        panelStyle.Width(18).Render(
            statLabel.Render("Requests/s")+"\n"+
                statValue.Render(fmt.Sprintf("%d", totalRequests))),
        panelStyle.Width(18).Render(
            statLabel.Render("Avg CPU")+"\n"+
                statValue.Render(fmt.Sprintf("%.1f%%", avgCPU))),
        panelStyle.Width(18).Render(
            statLabel.Render("Avg Memory")+"\n"+
                statValue.Render(fmt.Sprintf("%.1f%%", avgMem))),
    )
    b.WriteString(stats)
    b.WriteString("\n\n")

    // Progress bars
    b.WriteString(fmt.Sprintf("  CPU  %s\n", m.cpuBar.ViewAs(avgCPU/100)))
    b.WriteString(fmt.Sprintf("  MEM  %s\n\n", m.memBar.ViewAs(avgMem/100)))

    // Server table
    b.WriteString(panelStyle.Render(m.table.View()))
    b.WriteString("\n\n")

    // Footer
    updateStr := m.lastUpdate.Format("15:04:05")
    b.WriteString(fmt.Sprintf("  %s Live  |  Updated: %s  |  q: quit\n",
        m.spinner.View(), updateStr))

    return b.String()
}

func main() {
    p := tea.NewProgram(initialModel(), tea.WithAltScreen())
    if _, err := p.Run(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

---

## 5. The Charm Ecosystem

[Charm](https://charm.sh) is the company behind Bubbletea, Lip Gloss, and Bubbles. They maintain
a broader ecosystem of tools for building beautiful terminal applications.

### 5.1 Wish: SSH Apps

[Wish](https://github.com/charmbracelet/wish) lets you serve Bubbletea applications over SSH.
Users connect via `ssh` and get a full TUI experience -- no installation required.

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/charmbracelet/bubbles/list"
    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/lipgloss"
    "github.com/charmbracelet/wish"
    "github.com/charmbracelet/wish/activeterm"
    "github.com/charmbracelet/wish/bubbletea"
    "github.com/charmbracelet/ssh"
)

// item implements list.Item
type item struct {
    title, desc string
}

func (i item) Title() string       { return i.title }
func (i item) Description() string { return i.desc }
func (i item) FilterValue() string { return i.title }

// model is our Bubbletea model served over SSH
type model struct {
    list   list.Model
    user   string
}

func newModel(user string) model {
    items := []list.Item{
        item{title: "Welcome", desc: "Thanks for connecting, " + user + "!"},
        item{title: "Tasks", desc: "View your assigned tasks"},
        item{title: "Status", desc: "Check system status"},
        item{title: "Logs", desc: "View recent logs"},
    }

    l := list.New(items, list.NewDefaultDelegate(), 60, 20)
    l.Title = fmt.Sprintf("Dashboard for %s", user)

    return model{list: l, user: user}
}

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.list.SetSize(msg.Width, msg.Height)
    case tea.KeyMsg:
        if msg.String() == "q" {
            return m, tea.Quit
        }
    }
    var cmd tea.Cmd
    m.list, cmd = m.list.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return lipgloss.NewStyle().Margin(1, 2).Render(m.list.View())
}

// teaHandler returns a Bubbletea handler for Wish
func teaHandler(s ssh.Session) (tea.Model, []tea.ProgramOption) {
    return newModel(s.User()), []tea.ProgramOption{tea.WithAltScreen()}
}

func main() {
    host := "0.0.0.0"
    port := 2222

    srv, err := wish.NewServer(
        wish.WithAddress(fmt.Sprintf("%s:%d", host, port)),
        wish.WithHostKeyPath(".ssh/id_ed25519"),
        wish.WithMiddleware(
            bubbletea.Middleware(teaHandler),
            activeterm.Middleware(),
        ),
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "Could not start server: %v\n", err)
        os.Exit(1)
    }

    done := make(chan os.Signal, 1)
    signal.Notify(done, os.Interrupt, syscall.SIGTERM)

    fmt.Printf("Starting SSH server on %s:%d\n", host, port)
    go func() {
        if err := srv.ListenAndServe(); err != nil {
            fmt.Fprintf(os.Stderr, "Server error: %v\n", err)
        }
    }()

    <-done
    fmt.Println("\nShutting down...")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        fmt.Fprintf(os.Stderr, "Shutdown error: %v\n", err)
    }
}
```

Users can then connect:

```bash
ssh -p 2222 alice@yourserver.com
```

### 5.2 Glow: Markdown Rendering

[Glow](https://github.com/charmbracelet/glow) is a terminal Markdown renderer. While it is
primarily used as a standalone CLI tool, its underlying library (`glamour`) can be embedded in
your own programs:

```go
package main

import (
    "fmt"
    "os"

    "github.com/charmbracelet/glamour"
)

func main() {
    markdown := `
# Project Status Report

## Summary

The project is **on track** for the Q1 release.

## Completed Tasks

- [x] Authentication module
- [x] Database migrations
- [x] API endpoints

## Remaining Tasks

- [ ] Frontend integration
- [ ] Load testing
- [ ] Documentation

## Metrics

| Metric | Value |
|--------|-------|
| Test Coverage | 87% |
| Open Issues | 12 |
| Sprint Velocity | 42 |

> Note: All critical issues have been resolved.

` + "```go\nfmt.Println(\"Hello, World!\")\n```"

    renderer, err := glamour.NewTermRenderer(
        glamour.WithAutoStyle(),     // Auto-detect light/dark background
        glamour.WithWordWrap(80),    // Wrap at 80 columns
    )
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    rendered, err := renderer.Render(markdown)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    fmt.Print(rendered)
}
```

### 5.3 Log: Structured Logging with Style

```go
package main

import (
    "os"

    "github.com/charmbracelet/log"
)

func main() {
    // Create a styled logger
    logger := log.NewWithOptions(os.Stderr, log.Options{
        ReportTimestamp: true,
        ReportCaller:   true,
        Level:          log.DebugLevel,
    })

    logger.Debug("Starting application")
    logger.Info("Server started", "host", "localhost", "port", 8080)
    logger.Warn("High memory usage", "usage", "87%")
    logger.Error("Connection failed", "err", "timeout", "retries", 3)

    // Child loggers with prefixed fields
    dbLogger := logger.With("component", "database")
    dbLogger.Info("Connected", "host", "db.example.com")
    dbLogger.Debug("Query executed", "duration", "12ms", "rows", 42)
}
```

### 5.4 Huh: Form Library

[Huh](https://github.com/charmbracelet/huh) is a higher-level form/prompt library built on
Bubbletea:

```go
package main

import (
    "fmt"
    "os"

    "github.com/charmbracelet/huh"
)

func main() {
    var (
        name     string
        email    string
        role     string
        confirm  bool
        language string
        bio      string
    )

    form := huh.NewForm(
        // Page 1: Basic info
        huh.NewGroup(
            huh.NewInput().
                Title("Name").
                Placeholder("John Doe").
                Value(&name).
                Validate(func(s string) error {
                    if len(s) < 2 {
                        return fmt.Errorf("name must be at least 2 characters")
                    }
                    return nil
                }),

            huh.NewInput().
                Title("Email").
                Placeholder("john@example.com").
                Value(&email),
        ),

        // Page 2: Role selection
        huh.NewGroup(
            huh.NewSelect[string]().
                Title("Role").
                Options(
                    huh.NewOption("Developer", "developer"),
                    huh.NewOption("Designer", "designer"),
                    huh.NewOption("Manager", "manager"),
                    huh.NewOption("DevOps", "devops"),
                ).
                Value(&role),

            huh.NewSelect[string]().
                Title("Preferred Language").
                Options(
                    huh.NewOption("Go", "go"),
                    huh.NewOption("Rust", "rust"),
                    huh.NewOption("Python", "python"),
                    huh.NewOption("TypeScript", "typescript"),
                ).
                Value(&language),
        ),

        // Page 3: Bio and confirmation
        huh.NewGroup(
            huh.NewText().
                Title("Bio").
                Placeholder("Tell us about yourself...").
                CharLimit(500).
                Value(&bio),

            huh.NewConfirm().
                Title("Submit?").
                Affirmative("Yes!").
                Negative("No.").
                Value(&confirm),
        ),
    )

    err := form.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    if confirm {
        fmt.Printf("\nProfile created:\n")
        fmt.Printf("  Name:     %s\n", name)
        fmt.Printf("  Email:    %s\n", email)
        fmt.Printf("  Role:     %s\n", role)
        fmt.Printf("  Language: %s\n", language)
        fmt.Printf("  Bio:      %s\n", bio)
    } else {
        fmt.Println("Cancelled.")
    }
}
```

---

## 6. Terminal Colors and Formatting

### 6.1 fatih/color

The [fatih/color](https://github.com/fatih/color) package is the most popular Go library for
terminal colors. It supports ANSI colors, bold/italic/underline, and automatic detection of
color support.

```go
package main

import (
    "fmt"
    "os"

    "github.com/fatih/color"
)

func main() {
    // Simple color functions
    color.Red("This is red text")
    color.Green("This is green text")
    color.Yellow("Warning: %s", "something happened")
    color.Blue("Info: running on port %d", 8080)

    // Bold and underline
    color.New(color.Bold).Println("Bold text")
    color.New(color.FgRed, color.Bold).Println("Bold red text")
    color.New(color.FgWhite, color.BgRed).Println("White on red")

    // Create reusable color functions
    success := color.New(color.FgGreen, color.Bold).SprintFunc()
    warning := color.New(color.FgYellow).SprintFunc()
    errorFn := color.New(color.FgRed, color.Bold).SprintFunc()
    info := color.New(color.FgCyan).SprintFunc()

    fmt.Printf("[%s] All tests passed\n", success("PASS"))
    fmt.Printf("[%s] Deprecated API used\n", warning("WARN"))
    fmt.Printf("[%s] Connection refused\n", errorFn("FAIL"))
    fmt.Printf("[%s] Server started on :8080\n", info("INFO"))

    // Sprintf variants
    successMsg := color.GreenString("Deployed %s to %s", "v1.2.3", "production")
    fmt.Println(successMsg)

    // Disable colors globally (for non-TTY output)
    color.NoColor = false // Set to true to force no color

    // Writing to specific writers
    errorWriter := color.New(color.FgRed).FprintFunc()
    errorWriter(os.Stderr, "Error: something went wrong\n")

    // Hi-intensity colors
    color.New(color.FgHiGreen).Println("Hi-intensity green")
    color.New(color.FgHiYellow, color.Bold).Println("Hi-intensity bold yellow")

    // Color pairs for table-like output
    header := color.New(color.FgWhite, color.Bold, color.Underline)
    header.Println("Name            Status    Priority")

    row1 := color.New(color.FgGreen)
    row1.Println("Fix auth bug    Done      High")

    row2 := color.New(color.FgYellow)
    row2.Println("Update docs     Pending   Medium")

    row3 := color.New(color.FgRed)
    row3.Println("Deploy v2.0     Blocked   High")
}
```

### 6.2 termenv

[termenv](https://github.com/muesli/termenv) provides advanced terminal capabilities including
true color support, hyperlinks, and sophisticated color profiles.

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/muesli/termenv"
)

func main() {
    // Get the terminal output profile
    output := termenv.NewOutput(os.Stdout)
    p := output.ColorProfile()

    fmt.Printf("Terminal: %s\n", output.EnvNoColor())
    fmt.Printf("Color profile: %v\n", p)

    // Style text
    bold := output.String("Bold text").Bold().String()
    italic := output.String("Italic text").Italic().String()
    underline := output.String("Underlined").Underline().String()
    strikethrough := output.String("Strikethrough").Strikethrough().String()

    fmt.Println(bold)
    fmt.Println(italic)
    fmt.Println(underline)
    fmt.Println(strikethrough)

    // Colors
    red := output.String("Red text").Foreground(p.Color("#FF0000")).String()
    greenBg := output.String("Green background").Background(p.Color("#00FF00")).String()
    fmt.Println(red)
    fmt.Println(greenBg)

    // Hyperlinks (supported in modern terminals like iTerm2, Windows Terminal)
    link := output.Hyperlink("https://charm.sh", "Visit Charm")
    fmt.Println(link)

    // Copy to clipboard
    output.Copy("text to copy")

    // Terminal status
    fg := output.ForegroundColor()
    bg := output.BackgroundColor()
    fmt.Printf("Foreground: %v\n", fg)
    fmt.Printf("Background: %v\n", bg)
    fmt.Printf("Dark background: %v\n", output.HasDarkBackground())

    // Create a gradient
    colors := []string{"#FF0000", "#FF7700", "#FFFF00", "#00FF00", "#0000FF", "#8B00FF"}
    text := "RAINBOW TEXT"

    var rainbow strings.Builder
    for i, ch := range text {
        colorIdx := i % len(colors)
        s := output.String(string(ch)).Foreground(p.Color(colors[colorIdx]))
        rainbow.WriteString(s.String())
    }
    fmt.Println(rainbow.String())
}
```

### 6.3 Detecting Color Support

It is important to respect the user's environment:

```go
package main

import (
    "fmt"
    "os"

    "github.com/mattn/go-isatty"
)

func main() {
    // Check if stdout is a terminal (not piped or redirected)
    isTTY := isatty.IsTerminal(os.Stdout.Fd()) ||
        isatty.IsCygwinTerminal(os.Stdout.Fd())

    if isTTY {
        fmt.Println("Running in a terminal -- colors are available")
    } else {
        fmt.Println("Output is redirected -- using plain text")
    }

    // Many color libraries check NO_COLOR env var automatically
    // https://no-color.org/
    if os.Getenv("NO_COLOR") != "" {
        fmt.Println("NO_COLOR is set -- respecting user preference")
    }

    // TERM=dumb also indicates no color support
    if os.Getenv("TERM") == "dumb" {
        fmt.Println("TERM=dumb -- disabling colors")
    }
}
```

> **Best Practice:** Always respect the `NO_COLOR` environment variable
> (see [no-color.org](https://no-color.org/)). If `NO_COLOR` is set (to any value), your CLI
> should not output ANSI color codes.

---

## 7. Progress Bars and Spinners

### 7.1 schollz/progressbar

[schollz/progressbar](https://github.com/schollz/progressbar) provides configurable progress bars:

```go
package main

import (
    "fmt"
    "time"

    "github.com/schollz/progressbar/v3"
)

func main() {
    // Simple progress bar
    bar := progressbar.Default(100, "Processing")
    for i := 0; i < 100; i++ {
        bar.Add(1)
        time.Sleep(50 * time.Millisecond)
    }

    fmt.Println()

    // Download-style progress bar with bytes
    bar2 := progressbar.DefaultBytes(1024*1024*100, "Downloading")
    for i := 0; i < 100; i++ {
        bar2.Add(1024 * 1024) // 1 MB at a time
        time.Sleep(30 * time.Millisecond)
    }

    fmt.Println()

    // Customized progress bar
    bar3 := progressbar.NewOptions(1000,
        progressbar.OptionSetDescription("Building"),
        progressbar.OptionSetWidth(40),
        progressbar.OptionShowCount(),
        progressbar.OptionShowIts(),
        progressbar.OptionSetPredictTime(true),
        progressbar.OptionSetTheme(progressbar.Theme{
            Saucer:        "=",
            SaucerHead:    ">",
            SaucerPadding: "-",
            BarStart:      "[",
            BarEnd:        "]",
        }),
        progressbar.OptionOnCompletion(func() {
            fmt.Println("\nBuild complete!")
        }),
    )

    for i := 0; i < 1000; i++ {
        bar3.Add(1)
        time.Sleep(5 * time.Millisecond)
    }
}
```

#### Spinner Without Progress

```go
package main

import (
    "fmt"
    "time"

    "github.com/schollz/progressbar/v3"
)

func main() {
    // Indeterminate spinner (use -1 for max)
    bar := progressbar.NewOptions(-1,
        progressbar.OptionSetDescription("Connecting to server..."),
        progressbar.OptionSpinnerType(14), // Spinner style (0-75 available)
        progressbar.OptionSetWidth(10),
    )

    // Simulate work
    done := make(chan bool)
    go func() {
        time.Sleep(3 * time.Second)
        done <- true
    }()

    for {
        select {
        case <-done:
            bar.Finish()
            fmt.Println("\nConnected!")
            return
        default:
            bar.Add(1)
            time.Sleep(100 * time.Millisecond)
        }
    }
}
```

### 7.2 Multi-Progress Bars

For concurrent operations, use multiple progress bars:

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"

    "github.com/schollz/progressbar/v3"
)

func main() {
    files := []struct {
        name string
        size int64
    }{
        {"ubuntu-22.04.iso", 3_600_000_000},
        {"golang-1.22.tar.gz", 150_000_000},
        {"node-v20.tar.xz", 45_000_000},
        {"rust-1.75-src.tar.gz", 250_000_000},
    }

    var wg sync.WaitGroup

    for _, file := range files {
        wg.Add(1)
        go func(name string, size int64) {
            defer wg.Done()

            bar := progressbar.DefaultBytes(size, "Downloading "+name)

            downloaded := int64(0)
            for downloaded < size {
                chunk := int64(rand.Intn(10_000_000)) + 1_000_000
                if downloaded+chunk > size {
                    chunk = size - downloaded
                }
                bar.Add64(chunk)
                downloaded += chunk
                time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
            }
        }(file.name, file.size)
    }

    wg.Wait()
    fmt.Println("\nAll downloads complete!")
}
```

---

## 8. Interactive Prompts

### 8.1 promptui

[promptui](https://github.com/manifoldco/promptui) provides interactive prompts for selections and
text input:

```go
package main

import (
    "fmt"
    "os"
    "strconv"
    "strings"

    "github.com/manifoldco/promptui"
)

func main() {
    // Text prompt with validation
    namePrompt := promptui.Prompt{
        Label: "Project name",
        Validate: func(input string) error {
            if len(input) < 2 {
                return fmt.Errorf("name must be at least 2 characters")
            }
            if strings.Contains(input, " ") {
                return fmt.Errorf("name must not contain spaces")
            }
            return nil
        },
        Default: "my-project",
    }

    name, err := namePrompt.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Prompt failed: %v\n", err)
        os.Exit(1)
    }

    // Number prompt
    portPrompt := promptui.Prompt{
        Label:   "Port",
        Default: "8080",
        Validate: func(input string) error {
            n, err := strconv.Atoi(input)
            if err != nil {
                return fmt.Errorf("must be a number")
            }
            if n < 1024 || n > 65535 {
                return fmt.Errorf("must be between 1024 and 65535")
            }
            return nil
        },
    }

    port, err := portPrompt.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Prompt failed: %v\n", err)
        os.Exit(1)
    }

    // Password prompt (masked input)
    passwordPrompt := promptui.Prompt{
        Label: "API Key",
        Mask:  '*',
        Validate: func(input string) error {
            if len(input) < 8 {
                return fmt.Errorf("key must be at least 8 characters")
            }
            return nil
        },
    }

    apiKey, err := passwordPrompt.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Prompt failed: %v\n", err)
        os.Exit(1)
    }

    // Selection prompt
    frameworkSelect := promptui.Select{
        Label: "Web framework",
        Items: []string{"gin", "echo", "chi", "fiber", "net/http"},
        Size:  5,
    }

    _, framework, err := frameworkSelect.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Selection failed: %v\n", err)
        os.Exit(1)
    }

    // Yes/No confirmation
    confirmPrompt := promptui.Prompt{
        Label:     "Create project",
        IsConfirm: true,
    }

    _, err = confirmPrompt.Run()
    if err != nil {
        fmt.Println("Cancelled.")
        os.Exit(0)
    }

    fmt.Printf("\nCreating project:\n")
    fmt.Printf("  Name:      %s\n", name)
    fmt.Printf("  Port:      %s\n", port)
    fmt.Printf("  API Key:   %s***\n", apiKey[:3])
    fmt.Printf("  Framework: %s\n", framework)
}
```

#### Selection with Custom Templates

```go
package main

import (
    "fmt"
    "os"

    "github.com/manifoldco/promptui"
)

type database struct {
    Name        string
    Description string
    License     string
}

func main() {
    databases := []database{
        {Name: "PostgreSQL", Description: "Advanced relational database", License: "PostgreSQL License"},
        {Name: "MySQL", Description: "Popular relational database", License: "GPLv2"},
        {Name: "SQLite", Description: "Embedded relational database", License: "Public Domain"},
        {Name: "MongoDB", Description: "Document-oriented NoSQL", License: "SSPL"},
        {Name: "Redis", Description: "In-memory key-value store", License: "BSD"},
    }

    templates := &promptui.SelectTemplates{
        Label:    "{{ . }}?",
        Active:   "\U0001F449 {{ .Name | cyan }} ({{ .License | red }})",
        Inactive: "  {{ .Name | cyan }} ({{ .License | red }})",
        Selected: "\U0001F44D {{ .Name | green }}",
        Details: `
--------- Details ----------
{{ "Name:" | faint }}	{{ .Name }}
{{ "Description:" | faint }}	{{ .Description }}
{{ "License:" | faint }}	{{ .License }}`,
    }

    searcher := func(input string, index int) bool {
        db := databases[index]
        name := fmt.Sprintf("%s %s", db.Name, db.Description)
        return promptui.ContainsIgnoreCase(name, input)
    }

    prompt := promptui.Select{
        Label:     "Select database",
        Items:     databases,
        Templates: templates,
        Size:      5,
        Searcher:  searcher,
    }

    idx, _, err := prompt.Run()
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Selected: %s\n", databases[idx].Name)
}
```

### 8.2 AlecAivazis/survey (v2)

[survey](https://github.com/AlecAivazis/survey) provides another style of interactive prompts:

```go
package main

import (
    "fmt"
    "os"

    "github.com/AlecAivazis/survey/v2"
)

func main() {
    // All questions at once
    answers := struct {
        Name      string
        Language  string
        Features  []string
        Continue  bool
    }{}

    questions := []*survey.Question{
        {
            Name: "name",
            Prompt: &survey.Input{
                Message: "Project name:",
                Default: "my-app",
            },
            Validate: survey.Required,
        },
        {
            Name: "language",
            Prompt: &survey.Select{
                Message: "Choose a language:",
                Options: []string{"Go", "Rust", "Python", "TypeScript"},
                Default: "Go",
            },
        },
        {
            Name: "features",
            Prompt: &survey.MultiSelect{
                Message: "Select features:",
                Options: []string{
                    "REST API",
                    "GraphQL",
                    "WebSocket",
                    "Database",
                    "Authentication",
                    "Docker",
                    "CI/CD",
                },
            },
        },
        {
            Name: "continue",
            Prompt: &survey.Confirm{
                Message: "Create project?",
                Default: true,
            },
        },
    }

    err := survey.Ask(questions, &answers)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }

    if answers.Continue {
        fmt.Printf("\nCreating %s with %s\n", answers.Name, answers.Language)
        fmt.Printf("Features: %v\n", answers.Features)
    }
}

// Individual prompts
func individualPrompts() {
    // Editor prompt (opens $EDITOR)
    var bio string
    survey.AskOne(&survey.Editor{
        Message:  "Enter your bio:",
        FileName: "*.md",
    }, &bio)

    // Password
    var password string
    survey.AskOne(&survey.Password{
        Message: "Enter password:",
    }, &password)

    // Multi-line input
    var notes string
    survey.AskOne(&survey.Multiline{
        Message: "Additional notes:",
    }, &notes)
}
```

---

## 9. Cross-Compilation for Multiple Platforms

### 9.1 Basic Cross-Compilation

Go makes cross-compilation trivial with `GOOS` and `GOARCH` environment variables:

```bash
# Build for Linux (amd64)
GOOS=linux GOARCH=amd64 go build -o taskctl-linux-amd64 .

# Build for Linux (arm64 -- Raspberry Pi 4, AWS Graviton)
GOOS=linux GOARCH=arm64 go build -o taskctl-linux-arm64 .

# Build for macOS (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o taskctl-darwin-arm64 .

# Build for macOS (Intel)
GOOS=darwin GOARCH=amd64 go build -o taskctl-darwin-amd64 .

# Build for Windows
GOOS=windows GOARCH=amd64 go build -o taskctl-windows-amd64.exe .

# Build for FreeBSD
GOOS=freebsd GOARCH=amd64 go build -o taskctl-freebsd-amd64 .
```

### 9.2 Build Script for All Platforms

```bash
#!/bin/bash
# scripts/build-all.sh

set -euo pipefail

APP_NAME="taskctl"
VERSION="${1:-dev}"
COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
OUTPUT_DIR="dist"

LDFLAGS="-s -w \
  -X github.com/yourname/taskctl/internal/version.Version=${VERSION} \
  -X github.com/yourname/taskctl/internal/version.Commit=${COMMIT} \
  -X github.com/yourname/taskctl/internal/version.Date=${DATE}"

PLATFORMS=(
    "linux/amd64"
    "linux/arm64"
    "linux/arm"
    "darwin/amd64"
    "darwin/arm64"
    "windows/amd64"
    "windows/arm64"
    "freebsd/amd64"
)

rm -rf "${OUTPUT_DIR}"
mkdir -p "${OUTPUT_DIR}"

for platform in "${PLATFORMS[@]}"; do
    GOOS="${platform%/*}"
    GOARCH="${platform#*/}"

    output="${OUTPUT_DIR}/${APP_NAME}-${VERSION}-${GOOS}-${GOARCH}"
    if [ "${GOOS}" = "windows" ]; then
        output="${output}.exe"
    fi

    echo "Building ${platform}..."
    GOOS="${GOOS}" GOARCH="${GOARCH}" go build \
        -ldflags "${LDFLAGS}" \
        -o "${output}" \
        .
done

echo "Build complete. Artifacts in ${OUTPUT_DIR}/:"
ls -lh "${OUTPUT_DIR}/"
```

### 9.3 CGO Considerations

By default, `CGO_ENABLED=0` when cross-compiling. This is usually what you want for CLI tools.
If your application requires cgo (e.g., for SQLite via `go-sqlite3`), you need a cross-compiler
toolchain:

```bash
# Force CGO off (pure Go, fully static)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o taskctl .

# If you need CGO for cross-compilation, use tools like:
# - zig cc (as a drop-in cross-compiler)
# - docker with target-platform toolchains
# - xgo (https://github.com/techknowlogick/xgo)

# Using zig as cross-compiler (popular approach)
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 \
    CC="zig cc -target x86_64-linux" \
    go build -o taskctl-linux .
```

### 9.4 Supported GOOS/GOARCH Combinations

| GOOS | GOARCH values |
|------|---------------|
| `linux` | `amd64`, `arm64`, `arm`, `386`, `mips`, `mips64`, `ppc64le`, `riscv64`, `s390x` |
| `darwin` | `amd64`, `arm64` |
| `windows` | `amd64`, `arm64`, `386`, `arm` |
| `freebsd` | `amd64`, `arm64`, `386`, `arm` |
| `openbsd` | `amd64`, `arm64`, `386` |
| `netbsd` | `amd64`, `arm64`, `386` |
| `js` | `wasm` |
| `wasip1` | `wasm` |
| `ios` | `arm64`, `amd64` |
| `android` | `arm64`, `amd64`, `arm`, `386` |

List all supported pairs:

```bash
go tool dist list
```

---

## 10. Distribution

### 10.1 GoReleaser

[GoReleaser](https://goreleaser.com) automates the entire release pipeline: building, archiving,
checksumming, signing, publishing to GitHub, Homebrew, Docker registries, and more.

#### Installation

```bash
# macOS
brew install goreleaser

# Go
go install github.com/goreleaser/goreleaser/v2@latest
```

#### Configuration

```yaml
# .goreleaser.yaml
version: 2

project_name: taskctl

before:
  hooks:
    - go mod tidy
    - go generate ./...

builds:
  - id: taskctl
    main: .
    binary: taskctl
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
      - freebsd
    goarch:
      - amd64
      - arm64
      - arm
    goarm:
      - "7"
    ignore:
      - goos: freebsd
        goarch: arm
      - goos: windows
        goarch: arm
    ldflags:
      - -s -w
      - -X github.com/yourname/taskctl/internal/version.Version={{.Version}}
      - -X github.com/yourname/taskctl/internal/version.Commit={{.Commit}}
      - -X github.com/yourname/taskctl/internal/version.Date={{.Date}}

archives:
  - id: default
    formats:
      - tar.gz
    format_overrides:
      - goos: windows
        formats:
          - zip
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    files:
      - LICENSE
      - README.md
      - completions/*

checksum:
  name_template: "checksums.txt"
  algorithm: sha256

snapshot:
  version_template: "{{ incpatch .Version }}-next"

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^ci:"
      - "merge conflict"
      - Merge pull request
      - Merge branch

release:
  github:
    owner: yourname
    name: taskctl
  draft: false
  prerelease: auto
  name_template: "v{{ .Version }}"

brews:
  - name: taskctl
    repository:
      owner: yourname
      name: homebrew-tap
    homepage: "https://github.com/yourname/taskctl"
    description: "A task management CLI tool"
    license: "MIT"
    install: |
      bin.install "taskctl"
      # Install shell completions
      bash_completion.install "completions/taskctl.bash" => "taskctl"
      zsh_completion.install "completions/taskctl.zsh" => "_taskctl"
      fish_completion.install "completions/taskctl.fish"
    test: |
      system "#{bin}/taskctl", "version"

scoops:
  - name: taskctl
    repository:
      owner: yourname
      name: scoop-bucket
    homepage: "https://github.com/yourname/taskctl"
    description: "A task management CLI tool"
    license: MIT

nfpms:
  - id: packages
    package_name: taskctl
    vendor: Your Name
    homepage: "https://github.com/yourname/taskctl"
    maintainer: "Your Name <you@example.com>"
    description: "A task management CLI tool"
    license: MIT
    formats:
      - deb
      - rpm
      - apk
    bindir: /usr/bin
    contents:
      - src: ./completions/taskctl.bash
        dst: /usr/share/bash-completion/completions/taskctl
      - src: ./completions/taskctl.zsh
        dst: /usr/share/zsh/vendor-completions/_taskctl
      - src: ./completions/taskctl.fish
        dst: /usr/share/fish/vendor_completions.d/taskctl.fish

dockers:
  - image_templates:
      - "ghcr.io/yourname/taskctl:{{ .Version }}-amd64"
    use: buildx
    build_flag_templates:
      - "--platform=linux/amd64"
    goarch: amd64
    dockerfile: Dockerfile

  - image_templates:
      - "ghcr.io/yourname/taskctl:{{ .Version }}-arm64"
    use: buildx
    build_flag_templates:
      - "--platform=linux/arm64"
    goarch: arm64
    dockerfile: Dockerfile

docker_manifests:
  - name_template: "ghcr.io/yourname/taskctl:{{ .Version }}"
    image_templates:
      - "ghcr.io/yourname/taskctl:{{ .Version }}-amd64"
      - "ghcr.io/yourname/taskctl:{{ .Version }}-arm64"

  - name_template: "ghcr.io/yourname/taskctl:latest"
    image_templates:
      - "ghcr.io/yourname/taskctl:{{ .Version }}-amd64"
      - "ghcr.io/yourname/taskctl:{{ .Version }}-arm64"

signs:
  - artifacts: checksum
    args:
      - "--batch"
      - "--local-user"
      - "{{ .Env.GPG_FINGERPRINT }}"
      - "--output"
      - "${signature}"
      - "--detach-sign"
      - "${artifact}"
```

#### Usage

```bash
# Test the release locally (does not publish)
goreleaser release --snapshot --clean

# Create a release from a git tag
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
goreleaser release --clean

# Check configuration
goreleaser check
```

#### GitHub Actions Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - name: Run Tests
        run: go test ./...

      - name: Generate Completions
        run: |
          mkdir -p completions
          go run . completion bash > completions/taskctl.bash
          go run . completion zsh > completions/taskctl.zsh
          go run . completion fish > completions/taskctl.fish

      - uses: goreleaser/goreleaser-action@v6
        with:
          version: "~> v2"
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 10.2 Homebrew Distribution

If you do not want GoReleaser to manage your Homebrew formula, create one manually:

```ruby
# Formula/taskctl.rb
class Taskctl < Formula
  desc "A task management CLI tool"
  homepage "https://github.com/yourname/taskctl"
  version "1.0.0"
  license "MIT"

  on_macos do
    on_arm do
      url "https://github.com/yourname/taskctl/releases/download/v1.0.0/taskctl_1.0.0_darwin_arm64.tar.gz"
      sha256 "abc123..."
    end
    on_intel do
      url "https://github.com/yourname/taskctl/releases/download/v1.0.0/taskctl_1.0.0_darwin_amd64.tar.gz"
      sha256 "def456..."
    end
  end

  on_linux do
    on_arm do
      url "https://github.com/yourname/taskctl/releases/download/v1.0.0/taskctl_1.0.0_linux_arm64.tar.gz"
      sha256 "ghi789..."
    end
    on_intel do
      url "https://github.com/yourname/taskctl/releases/download/v1.0.0/taskctl_1.0.0_linux_amd64.tar.gz"
      sha256 "jkl012..."
    end
  end

  def install
    bin.install "taskctl"
    bash_completion.install "completions/taskctl.bash" => "taskctl"
    zsh_completion.install "completions/taskctl.zsh" => "_taskctl"
    fish_completion.install "completions/taskctl.fish"
  end

  test do
    assert_match "taskctl version", shell_output("#{bin}/taskctl version")
  end
end
```

Users install via:

```bash
brew tap yourname/tap
brew install taskctl
```

### 10.3 Snap Distribution

```yaml
# snap/snapcraft.yaml
name: taskctl
version: "1.0.0"
summary: A task management CLI tool
description: |
  taskctl is a command-line task manager that helps you organize
  your work with priorities, due dates, and tags.

grade: stable
confinement: strict
base: core22

architectures:
  - build-on: amd64
  - build-on: arm64

parts:
  taskctl:
    plugin: go
    source: .
    build-snaps:
      - go/1.22/stable

apps:
  taskctl:
    command: bin/taskctl
    plugs:
      - home
      - network
```

### 10.4 Docker Distribution

```dockerfile
# Dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build \
    -ldflags="-s -w" \
    -o /taskctl .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
COPY --from=builder /taskctl /usr/local/bin/taskctl

ENTRYPOINT ["taskctl"]
```

```bash
docker build -t taskctl .
docker run --rm taskctl version
docker run --rm -v ~/.taskctl:/root/.taskctl taskctl list
```

### 10.5 Install Script

Many Go CLI tools provide a one-liner install script:

```bash
#!/bin/sh
# install.sh -- Download and install taskctl
set -e

REPO="yourname/taskctl"
INSTALL_DIR="${INSTALL_DIR:-/usr/local/bin}"

# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
case "${ARCH}" in
    x86_64)  ARCH="amd64" ;;
    aarch64) ARCH="arm64" ;;
    armv7l)  ARCH="arm"   ;;
esac

# Get latest version
VERSION=$(curl -sL "https://api.github.com/repos/${REPO}/releases/latest" | \
    grep '"tag_name"' | sed -E 's/.*"v([^"]+)".*/\1/')

if [ -z "${VERSION}" ]; then
    echo "Error: Could not determine latest version"
    exit 1
fi

# Download
URL="https://github.com/${REPO}/releases/download/v${VERSION}/taskctl_${VERSION}_${OS}_${ARCH}.tar.gz"
echo "Downloading taskctl v${VERSION} for ${OS}/${ARCH}..."

TMP_DIR=$(mktemp -d)
curl -sL "${URL}" | tar xz -C "${TMP_DIR}"

# Install
echo "Installing to ${INSTALL_DIR}..."
sudo install -m 755 "${TMP_DIR}/taskctl" "${INSTALL_DIR}/taskctl"

# Cleanup
rm -rf "${TMP_DIR}"

echo "taskctl v${VERSION} installed successfully!"
echo "Run 'taskctl --help' to get started."
```

Users install with:

```bash
curl -sSL https://raw.githubusercontent.com/yourname/taskctl/main/install.sh | sh
```

---

## 11. Testing CLI Applications

### 11.1 Testing Cobra Commands

```go
// cmd/add_test.go
package cmd

import (
    "bytes"
    "os"
    "path/filepath"
    "testing"
)

func TestAddCommand(t *testing.T) {
    // Create a temporary store
    tmpDir := t.TempDir()
    storePath := filepath.Join(tmpDir, "tasks.json")
    t.Setenv("TASKCTL_STORE_PATH", storePath)

    tests := []struct {
        name       string
        args       []string
        wantErr    bool
        wantOutput string
    }{
        {
            name:       "add simple task",
            args:       []string{"add", "Buy groceries"},
            wantOutput: "Created task #1: Buy groceries\n",
        },
        {
            name:       "add task with priority",
            args:       []string{"add", "Fix bug", "--priority", "high"},
            wantOutput: "Created task #2: Fix bug\n",
        },
        {
            name:       "add task with tags",
            args:       []string{"add", "Review PR", "--tags", "code,review"},
            wantOutput: "Created task #3: Review PR\n",
        },
        {
            name:    "add without title fails",
            args:    []string{"add"},
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Capture output
            buf := new(bytes.Buffer)
            rootCmd.SetOut(buf)
            rootCmd.SetErr(buf)
            rootCmd.SetArgs(tt.args)

            err := rootCmd.Execute()

            if tt.wantErr {
                if err == nil {
                    t.Error("expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Errorf("unexpected error: %v", err)
                return
            }

            got := buf.String()
            if got != tt.wantOutput {
                t.Errorf("output = %q, want %q", got, tt.wantOutput)
            }
        })
    }
}
```

### 11.2 Testing with Execution Isolation

For more robust tests, create a fresh root command for each test to avoid shared state:

```go
// cmd/testing_helpers_test.go
package cmd

import (
    "bytes"
    "path/filepath"
    "testing"
)

type cmdResult struct {
    stdout string
    stderr string
    err    error
}

func executeCommand(t *testing.T, args ...string) cmdResult {
    t.Helper()

    // Create a temporary store for each test
    tmpDir := t.TempDir()
    storePath := filepath.Join(tmpDir, "tasks.json")

    // Build a fresh command tree
    root := newRootCmd()
    root.PersistentFlags().String("store-path", storePath, "")

    stdout := new(bytes.Buffer)
    stderr := new(bytes.Buffer)
    root.SetOut(stdout)
    root.SetErr(stderr)
    root.SetArgs(args)

    err := root.Execute()

    return cmdResult{
        stdout: stdout.String(),
        stderr: stderr.String(),
        err:    err,
    }
}

func TestCommandIntegration(t *testing.T) {
    // Add a task
    result := executeCommand(t, "add", "Test task", "--priority", "high")
    if result.err != nil {
        t.Fatalf("add failed: %v", result.err)
    }

    // List tasks
    result = executeCommand(t, "list", "--format", "json")
    if result.err != nil {
        t.Fatalf("list failed: %v", result.err)
    }

    // Note: in real tests, you would parse the JSON output and
    // verify the task was created correctly.
}
```

### 11.3 Testing Bubbletea Programs

Bubbletea provides a `teatest` package for testing TUI programs:

```go
package main

import (
    "bytes"
    "testing"
    "time"

    tea "github.com/charmbracelet/bubbletea"
    "github.com/charmbracelet/x/exp/teatest"
)

func TestTUIModel(t *testing.T) {
    m := initialModel()

    // Test initial state
    if m.cursor != 0 {
        t.Errorf("initial cursor = %d, want 0", m.cursor)
    }
    if len(m.selected) != 0 {
        t.Errorf("initial selected = %d, want 0", len(m.selected))
    }

    // Test Update with key messages
    t.Run("move cursor down", func(t *testing.T) {
        msg := tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}}
        updated, _ := m.Update(msg)
        newModel := updated.(model)
        if newModel.cursor != 1 {
            t.Errorf("cursor = %d, want 1", newModel.cursor)
        }
    })

    t.Run("select item", func(t *testing.T) {
        msg := tea.KeyMsg{Type: tea.KeySpace}
        updated, _ := m.Update(msg)
        newModel := updated.(model)
        if _, ok := newModel.selected[0]; !ok {
            t.Error("item 0 should be selected")
        }
    })

    t.Run("quit on q", func(t *testing.T) {
        msg := tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'q'}}
        _, cmd := m.Update(msg)
        if cmd == nil {
            t.Error("expected quit command")
        }
    })
}

func TestTUIView(t *testing.T) {
    m := initialModel()
    view := m.View()

    // Verify view contains expected elements
    if !bytes.Contains([]byte(view), []byte("What would you like to do today?")) {
        t.Error("view should contain title")
    }

    for _, choice := range m.choices {
        if !bytes.Contains([]byte(view), []byte(choice)) {
            t.Errorf("view should contain choice %q", choice)
        }
    }
}

// Integration test using teatest
func TestTUIIntegration(t *testing.T) {
    m := initialModel()
    tm := teatest.NewTestModel(t, m, teatest.WithInitialTermSize(80, 24))

    // Wait for initial render
    tm.WaitForOutput(t, func(bts []byte) bool {
        return bytes.Contains(bts, []byte("What would you like to do today?"))
    })

    // Navigate down
    tm.Send(tea.KeyMsg{Type: tea.KeyDown})
    tm.Send(tea.KeyMsg{Type: tea.KeyDown})

    // Select item
    tm.Send(tea.KeyMsg{Type: tea.KeySpace})

    // Quit
    tm.Send(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'q'}})

    // Wait for the program to finish
    tm.WaitFinished(t, teatest.WithFinalTimeout(3*time.Second))

    // Check final output
    out := tm.FinalOutput(t)
    if !bytes.Contains(out, []byte("[x]")) {
        t.Error("expected at least one selected item in final output")
    }
}
```

### 11.4 Golden File Tests

Golden file tests compare output against a saved "golden" file:

```go
package main

import (
    "flag"
    "os"
    "path/filepath"
    "testing"
)

var update = flag.Bool("update", false, "update golden files")

func TestGoldenOutput(t *testing.T) {
    tests := []struct {
        name string
        args []string
    }{
        {"help", []string{"--help"}},
        {"version", []string{"version"}},
        {"list_empty", []string{"list"}},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := executeCommand(t, tt.args...)

            goldenPath := filepath.Join("testdata", tt.name+".golden")

            if *update {
                os.MkdirAll("testdata", 0o755)
                os.WriteFile(goldenPath, []byte(result.stdout), 0o644)
                return
            }

            expected, err := os.ReadFile(goldenPath)
            if err != nil {
                t.Fatalf("reading golden file: %v", err)
            }

            if result.stdout != string(expected) {
                t.Errorf("output mismatch.\nGot:\n%s\nWant:\n%s",
                    result.stdout, string(expected))
            }
        })
    }
}
```

Run with `go test -update` to regenerate golden files, then `go test` to verify against them.

### 11.5 Testing CLI I/O Patterns

```go
package main

import (
    "bytes"
    "io"
    "os"
    "strings"
    "testing"
)

// captureOutput captures stdout and stderr during a function call.
func captureOutput(t *testing.T, fn func()) (stdout, stderr string) {
    t.Helper()

    oldStdout := os.Stdout
    oldStderr := os.Stderr

    rOut, wOut, _ := os.Pipe()
    rErr, wErr, _ := os.Pipe()

    os.Stdout = wOut
    os.Stderr = wErr

    fn()

    wOut.Close()
    wErr.Close()

    var bufOut, bufErr bytes.Buffer
    io.Copy(&bufOut, rOut)
    io.Copy(&bufErr, rErr)

    os.Stdout = oldStdout
    os.Stderr = oldStderr

    return bufOut.String(), bufErr.String()
}

// provideInput simulates stdin input.
func provideInput(t *testing.T, input string, fn func()) {
    t.Helper()

    oldStdin := os.Stdin
    r, w, _ := os.Pipe()

    go func() {
        defer w.Close()
        io.Copy(w, strings.NewReader(input))
    }()

    os.Stdin = r
    defer func() { os.Stdin = oldStdin }()

    fn()
}

func TestWithIO(t *testing.T) {
    stdout, stderr := captureOutput(t, func() {
        // Your CLI function here
        os.Stdout.WriteString("hello stdout\n")
        os.Stderr.WriteString("hello stderr\n")
    })

    if stdout != "hello stdout\n" {
        t.Errorf("stdout = %q", stdout)
    }
    if stderr != "hello stderr\n" {
        t.Errorf("stderr = %q", stderr)
    }
}
```

### 11.6 End-to-End Testing with the Binary

```go
package e2e_test

import (
    "os"
    "os/exec"
    "path/filepath"
    "strings"
    "testing"
)

var binaryPath string

func TestMain(m *testing.M) {
    // Build the binary once before all tests
    tmpDir, err := os.MkdirTemp("", "taskctl-e2e-*")
    if err != nil {
        panic(err)
    }
    defer os.RemoveAll(tmpDir)

    binaryPath = filepath.Join(tmpDir, "taskctl")
    cmd := exec.Command("go", "build", "-o", binaryPath, ".")
    cmd.Dir = ".." // assuming tests are in e2e/ subdirectory
    if out, err := cmd.CombinedOutput(); err != nil {
        panic(string(out) + ": " + err.Error())
    }

    os.Exit(m.Run())
}

func runCLI(t *testing.T, args ...string) (string, error) {
    t.Helper()
    cmd := exec.Command(binaryPath, args...)
    cmd.Env = append(os.Environ(),
        "TASKCTL_STORE_PATH="+filepath.Join(t.TempDir(), "tasks.json"),
        "NO_COLOR=1",
    )
    out, err := cmd.CombinedOutput()
    return string(out), err
}

func TestE2E_AddAndList(t *testing.T) {
    // Add a task
    out, err := runCLI(t, "add", "Buy groceries", "--priority", "high")
    if err != nil {
        t.Fatalf("add failed: %v\nOutput: %s", err, out)
    }
    if !strings.Contains(out, "Created task #1") {
        t.Errorf("unexpected output: %s", out)
    }

    // List tasks
    out, err = runCLI(t, "list")
    if err != nil {
        t.Fatalf("list failed: %v\nOutput: %s", err, out)
    }
    if !strings.Contains(out, "Buy groceries") {
        t.Errorf("task not found in list output: %s", out)
    }
}

func TestE2E_HelpOutput(t *testing.T) {
    out, err := runCLI(t, "--help")
    if err != nil {
        t.Fatalf("help failed: %v", err)
    }

    expected := []string{
        "taskctl",
        "add",
        "list",
        "complete",
        "delete",
    }

    for _, s := range expected {
        if !strings.Contains(out, s) {
            t.Errorf("help output missing %q", s)
        }
    }
}

func TestE2E_VersionOutput(t *testing.T) {
    out, err := runCLI(t, "version")
    if err != nil {
        t.Fatalf("version failed: %v", err)
    }
    if !strings.Contains(out, "taskctl") {
        t.Errorf("version output missing 'taskctl': %s", out)
    }
}

func TestE2E_ExitCodes(t *testing.T) {
    // Invalid command should fail
    _, err := runCLI(t, "nonexistent")
    if err == nil {
        t.Error("expected error for nonexistent command")
    }

    // Complete with invalid ID should fail
    _, err = runCLI(t, "complete", "999")
    if err == nil {
        t.Error("expected error for invalid task ID")
    }
}
```

---

## 12. Best Practices for CLI UX

### 12.1 Follow Established Conventions

| Convention | Description |
|-----------|-------------|
| `-v` / `--verbose` | Increase output verbosity |
| `-q` / `--quiet` | Suppress output |
| `-o` / `--output` | Output file or format |
| `-f` / `--force` | Skip confirmation prompts |
| `-n` / `--dry-run` | Show what would happen without doing it |
| `--no-color` | Disable color output |
| `--version` | Print version and exit |
| `--help` or `-h` | Print help and exit |
| `--config` | Path to configuration file |

### 12.2 Exit Codes

Use meaningful exit codes. This is critical for scripting:

```go
package main

import "os"

const (
    ExitOK              = 0  // Successful execution
    ExitError           = 1  // General error
    ExitUsageError      = 2  // Invalid command-line usage
    ExitDataError       = 65 // Input data was incorrect
    ExitNoInput         = 66 // Input file did not exist
    ExitCannotCreate    = 73 // Cannot create output file
    ExitTempFail        = 75 // Temporary failure (retry might succeed)
    ExitConfigError     = 78 // Configuration error
)

func main() {
    if err := run(); err != nil {
        switch err.(type) {
        case *UsageError:
            os.Exit(ExitUsageError)
        case *ConfigError:
            os.Exit(ExitConfigError)
        default:
            os.Exit(ExitError)
        }
    }
    os.Exit(ExitOK)
}
```

### 12.3 Respect the User's Environment

```go
package main

import (
    "io"
    "os"

    "github.com/mattn/go-isatty"
)

type OutputConfig struct {
    Color   bool
    Unicode bool
    Pager   string
    Width   int
}

func detectOutputConfig() OutputConfig {
    cfg := OutputConfig{
        Width: 80, // safe default
    }

    isTTY := isatty.IsTerminal(os.Stdout.Fd())

    // Enable color only if:
    // 1. stdout is a TTY
    // 2. NO_COLOR is not set
    // 3. TERM is not "dumb"
    cfg.Color = isTTY &&
        os.Getenv("NO_COLOR") == "" &&
        os.Getenv("TERM") != "dumb"

    // Enable Unicode in modern terminals
    cfg.Unicode = isTTY && os.Getenv("TERM") != "dumb"

    // Use pager for long output
    cfg.Pager = os.Getenv("PAGER")
    if cfg.Pager == "" {
        cfg.Pager = "less"
    }

    return cfg
}

// pageOutput pipes output through a pager for long content.
func pageOutput(content string) error {
    if !isatty.IsTerminal(os.Stdout.Fd()) {
        _, err := io.WriteString(os.Stdout, content)
        return err
    }

    // Use less with helpful options
    pager := os.Getenv("PAGER")
    if pager == "" {
        pager = "less -R" // -R preserves ANSI color codes
    }

    // ... pipe to pager (implementation depends on OS)
    _, err := io.WriteString(os.Stdout, content)
    return err
}
```

### 12.4 Stderr vs Stdout

This is one of the most important and most frequently violated conventions:

- **stdout** -- Program output. Data that can be piped to another program.
- **stderr** -- Progress, status messages, diagnostics, errors, warnings.

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type Task struct {
    ID    int    `json:"id"`
    Title string `json:"title"`
}

func main() {
    // Status messages go to stderr so they don't pollute piped output
    fmt.Fprintln(os.Stderr, "Loading tasks...")

    tasks := []Task{
        {ID: 1, Title: "Buy groceries"},
        {ID: 2, Title: "Write code"},
    }

    fmt.Fprintln(os.Stderr, "Found", len(tasks), "tasks")

    // Actual data goes to stdout so it can be piped
    enc := json.NewEncoder(os.Stdout)
    enc.SetIndent("", "  ")
    enc.Encode(tasks)

    // This works correctly with pipes:
    // taskctl list 2>/dev/null | jq '.[] | .title'
    // taskctl list --json | jq '.[]'
}
```

### 12.5 Machine-Readable Output

Always provide a machine-readable format for scripting:

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "text/tabwriter"

    "gopkg.in/yaml.v3"
)

type Task struct {
    ID       int    `json:"id" yaml:"id"`
    Title    string `json:"title" yaml:"title"`
    Status   string `json:"status" yaml:"status"`
    Priority string `json:"priority" yaml:"priority"`
}

func printTasks(tasks []Task, format string) error {
    switch format {
    case "json":
        enc := json.NewEncoder(os.Stdout)
        enc.SetIndent("", "  ")
        return enc.Encode(tasks)

    case "jsonl":
        // JSON Lines -- one JSON object per line (great for streaming)
        enc := json.NewEncoder(os.Stdout)
        for _, t := range tasks {
            if err := enc.Encode(t); err != nil {
                return err
            }
        }
        return nil

    case "yaml":
        enc := yaml.NewEncoder(os.Stdout)
        return enc.Encode(tasks)

    case "tsv":
        // Tab-separated values (easy to parse with awk/cut)
        for _, t := range tasks {
            fmt.Printf("%d\t%s\t%s\t%s\n", t.ID, t.Title, t.Status, t.Priority)
        }
        return nil

    case "ids":
        // Just IDs, one per line (for piping: taskctl list --format ids | xargs ...)
        for _, t := range tasks {
            fmt.Println(t.ID)
        }
        return nil

    default: // "table"
        w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
        fmt.Fprintln(w, "ID\tTITLE\tSTATUS\tPRIORITY")
        for _, t := range tasks {
            fmt.Fprintf(w, "%d\t%s\t%s\t%s\n",
                t.ID, t.Title, t.Status, t.Priority)
        }
        return w.Flush()
    }
}
```

### 12.6 Helpful Error Messages

```go
package main

import (
    "fmt"
    "os"
    "strings"
)

// CLIError represents a user-facing error with suggestions.
type CLIError struct {
    Message     string
    Suggestions []string
    ExitCode    int
}

func (e *CLIError) Error() string {
    var b strings.Builder
    fmt.Fprintf(&b, "Error: %s\n", e.Message)

    if len(e.Suggestions) > 0 {
        b.WriteString("\nDid you mean:\n")
        for _, s := range e.Suggestions {
            fmt.Fprintf(&b, "  %s\n", s)
        }
    }

    return b.String()
}

// Example usage
func handleError(err error) {
    if err == nil {
        return
    }

    if cliErr, ok := err.(*CLIError); ok {
        fmt.Fprint(os.Stderr, cliErr.Error())
        os.Exit(cliErr.ExitCode)
    }

    // Generic error
    fmt.Fprintf(os.Stderr, "Error: %v\n", err)
    fmt.Fprintf(os.Stderr, "Run 'taskctl --help' for usage information.\n")
    os.Exit(1)
}

// Example: unknown command with suggestions
func unknownCommand(cmd string, available []string) error {
    var suggestions []string
    for _, a := range available {
        if strings.HasPrefix(a, cmd[:1]) || levenshtein(cmd, a) <= 2 {
            suggestions = append(suggestions, "taskctl "+a)
        }
    }

    return &CLIError{
        Message:     fmt.Sprintf("unknown command %q", cmd),
        Suggestions: suggestions,
        ExitCode:    2,
    }
}

// Simple Levenshtein distance for command suggestions.
func levenshtein(a, b string) int {
    la, lb := len(a), len(b)
    d := make([][]int, la+1)
    for i := range d {
        d[i] = make([]int, lb+1)
        d[i][0] = i
    }
    for j := 0; j <= lb; j++ {
        d[0][j] = j
    }
    for i := 1; i <= la; i++ {
        for j := 1; j <= lb; j++ {
            cost := 1
            if a[i-1] == b[j-1] {
                cost = 0
            }
            d[i][j] = min(
                d[i-1][j]+1,      // deletion
                d[i][j-1]+1,      // insertion
                d[i-1][j-1]+cost, // substitution
            )
        }
    }
    return d[la][lb]
}
```

### 12.7 Dry-Run Mode

```go
var dryRun bool

func init() {
    rootCmd.PersistentFlags().BoolVar(&dryRun, "dry-run", false,
        "Print what would happen without making changes")
}

func deleteTask(id int) error {
    if dryRun {
        fmt.Fprintf(os.Stderr, "[dry-run] Would delete task #%d\n", id)
        return nil
    }

    // Actually delete the task
    store, err := getStore()
    if err != nil {
        return err
    }
    _, err = store.Delete(id)
    if err != nil {
        return err
    }
    return store.Save()
}
```

### 12.8 Configuration File Discovery

Follow the XDG Base Directory specification on Linux and platform conventions elsewhere:

```go
package config

import (
    "os"
    "path/filepath"
    "runtime"
)

// ConfigDir returns the platform-appropriate configuration directory.
func ConfigDir(appName string) string {
    switch runtime.GOOS {
    case "darwin":
        // macOS: ~/Library/Application Support/<appName>
        home, _ := os.UserHomeDir()
        return filepath.Join(home, "Library", "Application Support", appName)

    case "windows":
        // Windows: %APPDATA%\<appName>
        return filepath.Join(os.Getenv("APPDATA"), appName)

    default:
        // Linux/BSD: $XDG_CONFIG_HOME/<appName> or ~/.config/<appName>
        if xdg := os.Getenv("XDG_CONFIG_HOME"); xdg != "" {
            return filepath.Join(xdg, appName)
        }
        home, _ := os.UserHomeDir()
        return filepath.Join(home, ".config", appName)
    }
}

// DataDir returns the platform-appropriate data directory.
func DataDir(appName string) string {
    switch runtime.GOOS {
    case "darwin":
        home, _ := os.UserHomeDir()
        return filepath.Join(home, "Library", "Application Support", appName)

    case "windows":
        return filepath.Join(os.Getenv("LOCALAPPDATA"), appName)

    default:
        if xdg := os.Getenv("XDG_DATA_HOME"); xdg != "" {
            return filepath.Join(xdg, appName)
        }
        home, _ := os.UserHomeDir()
        return filepath.Join(home, ".local", "share", appName)
    }
}

// CacheDir returns the platform-appropriate cache directory.
func CacheDir(appName string) string {
    switch runtime.GOOS {
    case "darwin":
        home, _ := os.UserHomeDir()
        return filepath.Join(home, "Library", "Caches", appName)

    case "windows":
        return filepath.Join(os.Getenv("LOCALAPPDATA"), appName, "Cache")

    default:
        if xdg := os.Getenv("XDG_CACHE_HOME"); xdg != "" {
            return filepath.Join(xdg, appName)
        }
        home, _ := os.UserHomeDir()
        return filepath.Join(home, ".cache", appName)
    }
}

// ConfigSearchPaths returns all paths to check for configuration, in priority order.
func ConfigSearchPaths(appName string) []string {
    paths := []string{}

    // 1. Explicit config file (from --config flag or env var)
    if envConfig := os.Getenv("TASKCTL_CONFIG"); envConfig != "" {
        paths = append(paths, envConfig)
    }

    // 2. Current directory
    cwd, _ := os.Getwd()
    paths = append(paths, filepath.Join(cwd, "."+appName+".yaml"))

    // 3. User config directory
    paths = append(paths, filepath.Join(ConfigDir(appName), "config.yaml"))

    // 4. Home directory dotfile
    home, _ := os.UserHomeDir()
    paths = append(paths, filepath.Join(home, "."+appName+".yaml"))

    // 5. System-wide config
    if runtime.GOOS != "windows" {
        paths = append(paths, filepath.Join("/etc", appName, "config.yaml"))
    }

    return paths
}
```

### 12.9 Signal Handling and Graceful Shutdown

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // Create a context that is cancelled on SIGINT or SIGTERM
    ctx, stop := signal.NotifyContext(context.Background(),
        os.Interrupt, syscall.SIGTERM)
    defer stop()

    fmt.Println("Running... Press Ctrl+C to stop.")

    // Simulate long-running work
    select {
    case <-time.After(60 * time.Second):
        fmt.Println("Work completed.")
    case <-ctx.Done():
        fmt.Fprintln(os.Stderr, "\nInterrupted. Cleaning up...")

        // Perform cleanup with a deadline
        cleanupCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()

        if err := cleanup(cleanupCtx); err != nil {
            fmt.Fprintf(os.Stderr, "Cleanup error: %v\n", err)
            os.Exit(1)
        }

        fmt.Fprintln(os.Stderr, "Cleanup complete.")
    }
}

func cleanup(ctx context.Context) error {
    // Save state, close connections, remove temp files, etc.
    select {
    case <-time.After(1 * time.Second): // simulate cleanup work
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### 12.10 Summary of CLI UX Principles

| Principle | Guidance |
|-----------|----------|
| **Be quiet by default** | Only output what the user asked for. Use `-v` for more. |
| **Fail fast and loud** | Return clear errors with actionable suggestions immediately. |
| **stdout is data, stderr is status** | Never mix data output with progress/status messages. |
| **Respect NO_COLOR** | Check `NO_COLOR` env var. Also check if stdout is a TTY. |
| **Support piping** | Output should be usable with `|`, `>`, and `xargs`. |
| **Provide --json** | Give machine-readable output for scripting. |
| **Use exit codes** | 0 = success, 1 = general error, 2 = usage error. |
| **Offer --dry-run** | For destructive operations, show what would happen. |
| **Ask before destroying** | Require `--force` or confirmation for dangerous actions. |
| **Show progress** | For operations longer than ~1 second, show a spinner or bar. |
| **Be discoverable** | Good `--help`, shell completions, man pages. |
| **Be consistent** | Same flag names across all subcommands (`--output`, `--verbose`). |

---

## Summary

This chapter covered the complete stack for building professional CLI tools in Go:

- **Foundations**: `os.Args` and the `flag` package provide the primitives. Use them for simple
  scripts, but graduate to Cobra for anything with subcommands.

- **Cobra** is the dominant CLI framework. It gives you subcommands, POSIX flags, shell
  completions, auto-generated documentation, and Viper integration for configuration. Most
  major Go CLI tools use it.

- **Bubbletea** brings the Elm Architecture to the terminal. The Model/Update/View loop provides
  a clean, testable way to build interactive TUIs. Combined with **Bubbles** (reusable components)
  and **Lip Gloss** (styling), you can build rich terminal applications.

- **The Charm ecosystem** extends this with SSH-served apps (Wish), markdown rendering (Glow/Glamour),
  structured logging (Log), and high-level forms (Huh).

- **Colors and formatting** are handled by fatih/color (simple), termenv (advanced), and Lip Gloss
  (layout composition). Always respect `NO_COLOR`.

- **Distribution** with GoReleaser automates the release pipeline. Targets Homebrew, Scoop, Docker,
  Snap, and direct downloads. One YAML file, one command.

- **Testing** CLI tools requires testing commands (with captured output), TUI models (with teatest),
  and end-to-end binaries. Golden file tests catch regressions.

- **CLI UX** demands that you use stdout for data, stderr for status, meaningful exit codes,
  machine-readable output formats, and respect for the user's terminal environment.

The combination of Go's static binaries, fast startup, cross-compilation, and this ecosystem of
libraries makes it the strongest platform for CLI tool development available today.
