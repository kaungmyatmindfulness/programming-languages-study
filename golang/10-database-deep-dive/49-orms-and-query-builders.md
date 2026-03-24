# Chapter 49: ORMs and Query Builders in Go (GORM, ent, sqlc)

## Table of Contents

1. [Introduction](#introduction)
2. [ORM vs Query Builder vs Raw SQL](#orm-vs-query-builder-vs-raw-sql)
3. [GORM Fundamentals](#gorm-fundamentals)
4. [GORM Advanced Features](#gorm-advanced-features)
5. [ent by Facebook](#ent-by-facebook)
6. [ent Advanced Features](#ent-advanced-features)
7. [sqlc: Write SQL, Get Go](#sqlc-write-sql-get-go)
8. [Comparison: GORM vs ent vs sqlc](#comparison-gorm-vs-ent-vs-sqlc)
9. [When to Use Which Tool](#when-to-use-which-tool)
10. [Repository Pattern with Each Approach](#repository-pattern-with-each-approach)
11. [Testing Strategies](#testing-strategies)
12. [Performance Benchmarks and Common Pitfalls](#performance-benchmarks-and-common-pitfalls)
13. [Migration Strategies Between Approaches](#migration-strategies-between-approaches)
14. [Summary](#summary)

---

## Introduction

Go's database ecosystem offers a rich spectrum of tools ranging from raw `database/sql`
all the way up to full-featured ORMs. This chapter dives deep into the three most
prominent tools in the Go ecosystem:

| Tool | Category | Philosophy |
|------|----------|------------|
| **GORM** | Full ORM | Convention over configuration, ActiveRecord-style |
| **ent** | Entity framework + code gen | Schema-as-code with type-safe generated API |
| **sqlc** | Query builder / code gen | Write SQL, generate type-safe Go code |

Each tool makes fundamentally different tradeoffs between developer convenience,
type safety, performance, and control. Understanding these tradeoffs deeply is what
separates a productive Go backend engineer from one who fights their tools.

---

## ORM vs Query Builder vs Raw SQL

### The Spectrum

```
More Control                                              More Convenience
<-------------------------------------------------------------->
raw SQL    sqlc    squirrel/goqu    ent    GORM
```

### Raw SQL (`database/sql`)

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/lib/pq"
)

type User struct {
    ID        int64
    Name      string
    Email     string
    CreatedAt time.Time
}

func getUserByID(ctx context.Context, db *sql.DB, id int64) (*User, error) {
    var u User
    err := db.QueryRowContext(ctx,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
    if err != nil {
        return nil, fmt.Errorf("getUserByID %d: %w", id, err)
    }
    return &u, nil
}
```

**Pros:**
- Full control over every query
- No abstraction overhead
- No hidden queries or N+1 surprises
- Easy to optimize with `EXPLAIN ANALYZE`

**Cons:**
- Verbose boilerplate (`Scan` for every field)
- No compile-time safety for column names or types
- Manual mapping between rows and structs
- Refactoring is error-prone (rename a column, grep and pray)

### ORM (GORM)

```go
var user User
db.First(&user, id)
// SELECT * FROM users WHERE id = 1 ORDER BY id LIMIT 1
```

**Pros:**
- Minimal boilerplate
- Automatic migrations
- Built-in association handling
- Rapid prototyping

**Cons:**
- Implicit behavior can cause surprises
- N+1 query problems if you forget to preload
- Performance overhead from reflection
- Generated SQL may not be optimal

### Code-Generation (ent, sqlc)

```go
// ent - generated type-safe API
user, err := client.User.Get(ctx, id)

// sqlc - generated from SQL
user, err := queries.GetUser(ctx, id)
```

**Pros:**
- Compile-time type safety
- No runtime reflection
- Clear and predictable SQL
- Catches errors at build time, not at runtime

**Cons:**
- Extra build step (code generation)
- Learning curve for the generation toolchain
- Less flexible for truly dynamic queries

### Decision Matrix

| Factor | Raw SQL | sqlc | ent | GORM |
|--------|---------|------|-----|------|
| Type safety | None | Compile-time | Compile-time | Runtime |
| Boilerplate | High | Low | Low | Very Low |
| Performance | Best | Excellent | Good | Good* |
| Learning curve | SQL knowledge | SQL + config | Schema DSL | Struct tags |
| Dynamic queries | Manual string building | Limited | Good | Excellent |
| Migration support | Manual | External | Atlas integration | Built-in |
| Association handling | Manual JOINs | Manual JOINs | Graph traversal | Declarative |

> **Tip:** The asterisk on GORM's performance means "good when used correctly." Many
> GORM performance complaints come from misuse (N+1 queries, missing indexes) rather
> than inherent framework overhead.

---

## GORM Fundamentals

### Installation

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
go get -u gorm.io/driver/mysql
go get -u gorm.io/driver/sqlite
```

### Connecting to a Database

```go
package main

import (
    "fmt"
    "log"
    "os"
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func main() {
    dsn := fmt.Sprintf(
        "host=%s user=%s password=%s dbname=%s port=%s sslmode=disable",
        os.Getenv("DB_HOST"),
        os.Getenv("DB_USER"),
        os.Getenv("DB_PASS"),
        os.Getenv("DB_NAME"),
        os.Getenv("DB_PORT"),
    )

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // Log all SQL
        NowFunc: func() time.Time {
            return time.Now().UTC() // Always use UTC
        },
        PrepareStmt: true, // Cache prepared statements
    })
    if err != nil {
        log.Fatalf("failed to connect to database: %v", err)
    }

    // Get the underlying *sql.DB to configure connection pool
    sqlDB, err := db.DB()
    if err != nil {
        log.Fatalf("failed to get sql.DB: %v", err)
    }

    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)
    sqlDB.SetConnMaxIdleTime(10 * time.Minute)

    log.Println("Connected to database successfully")
}
```

### Model Definition

GORM models are Go structs with optional tags to control database mapping.

```go
import (
    "database/sql"
    "time"

    "gorm.io/gorm"
)

// gorm.Model provides ID, CreatedAt, UpdatedAt, DeletedAt (soft delete)
// type Model struct {
//     ID        uint           `gorm:"primarykey"`
//     CreatedAt time.Time
//     UpdatedAt time.Time
//     DeletedAt gorm.DeletedAt `gorm:"index"`
// }

// User demonstrates a fully-tagged GORM model.
type User struct {
    gorm.Model                          // Embeds ID, CreatedAt, UpdatedAt, DeletedAt
    Name      string                    `gorm:"type:varchar(100);not null;index"`
    Email     string                    `gorm:"type:varchar(255);uniqueIndex;not null"`
    Age       int                       `gorm:"default:0;check:age >= 0"`
    Bio       sql.NullString            `gorm:"type:text"`
    Role      string                    `gorm:"type:varchar(20);default:'user'"`
    IsActive  bool                      `gorm:"default:true"`

    // Associations
    Profile   Profile                   // Has One
    Orders    []Order                   // Has Many
    Languages []Language `gorm:"many2many:user_languages;"` // Many-to-Many
    CompanyID *uint                     // Belongs To (nullable FK)
    Company   Company                   // Belongs To
}

// Profile - has one relationship with User
type Profile struct {
    gorm.Model
    UserID    uint   `gorm:"uniqueIndex;not null"`
    AvatarURL string `gorm:"type:varchar(500)"`
    Phone     string `gorm:"type:varchar(20)"`
    Address   string `gorm:"type:text"`
}

// Order - has many relationship with User
type Order struct {
    gorm.Model
    UserID      uint           `gorm:"index;not null"`
    TotalAmount float64        `gorm:"type:decimal(10,2);not null"`
    Status      string         `gorm:"type:varchar(20);default:'pending';index"`
    Items       []OrderItem    // Has Many
}

// OrderItem - belongs to Order
type OrderItem struct {
    gorm.Model
    OrderID   uint    `gorm:"index;not null"`
    ProductID uint    `gorm:"index;not null"`
    Quantity  int     `gorm:"not null;check:quantity > 0"`
    Price     float64 `gorm:"type:decimal(10,2);not null"`
    Product   Product // Belongs To
}

// Product
type Product struct {
    gorm.Model
    Name        string    `gorm:"type:varchar(200);not null;index"`
    SKU         string    `gorm:"type:varchar(50);uniqueIndex;not null"`
    Price       float64   `gorm:"type:decimal(10,2);not null"`
    Description string    `gorm:"type:text"`
    Categories  []Category `gorm:"many2many:product_categories;"`
}

// Category
type Category struct {
    gorm.Model
    Name     string     `gorm:"type:varchar(100);uniqueIndex;not null"`
    ParentID *uint      `gorm:"index"`
    Parent   *Category  // Self-referential
    Children []Category `gorm:"foreignKey:ParentID"`
    Products []Product  `gorm:"many2many:product_categories;"`
}

// Language - many-to-many with User
type Language struct {
    gorm.Model
    Name  string `gorm:"type:varchar(50);uniqueIndex;not null"`
    Code  string `gorm:"type:varchar(10);uniqueIndex;not null"`
    Users []User `gorm:"many2many:user_languages;"`
}

// Company - one-to-many with User
type Company struct {
    gorm.Model
    Name      string `gorm:"type:varchar(200);not null"`
    Domain    string `gorm:"type:varchar(100);uniqueIndex"`
    Employees []User // Has Many
}

// TableName overrides the default table name.
func (User) TableName() string {
    return "users"
}
```

### Auto Migration

```go
func migrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &User{},
        &Profile{},
        &Order{},
        &OrderItem{},
        &Product{},
        &Category{},
        &Language{},
        &Company{},
    )
}
```

> **Warning:** `AutoMigrate` only creates tables, adds missing columns, and creates
> indexes. It does NOT delete unused columns, change column types, or drop tables.
> For production, use a proper migration tool like `golang-migrate` or Atlas.

### CRUD Operations

#### Create

```go
// Create a single record
user := User{
    Name:  "Alice",
    Email: "alice@example.com",
    Age:   30,
    Role:  "admin",
}
result := db.Create(&user)
// user.ID is populated after insert
fmt.Printf("Inserted user with ID: %d\n", user.ID)
fmt.Printf("Rows affected: %d\n", result.RowsAffected)
if result.Error != nil {
    log.Printf("Create failed: %v", result.Error)
}

// Create multiple records in batch
users := []User{
    {Name: "Bob", Email: "bob@example.com", Age: 25},
    {Name: "Charlie", Email: "charlie@example.com", Age: 35},
    {Name: "Diana", Email: "diana@example.com", Age: 28},
}
db.Create(&users) // Single INSERT with multiple rows

// Create in batches of N (useful for thousands of records)
db.CreateInBatches(&users, 100) // 100 records per INSERT
```

#### Read

```go
// Find by primary key
var user User
db.First(&user, 1)  // SELECT * FROM users WHERE id = 1 ORDER BY id LIMIT 1
db.First(&user, "id = ?", 1) // Same, explicit WHERE

// Find by other fields
db.Where("email = ?", "alice@example.com").First(&user)

// Find all matching records
var users []User
db.Where("age > ? AND role = ?", 25, "user").Find(&users)

// Find with struct condition (zero values are ignored!)
db.Where(&User{Name: "Alice", Age: 30}).Find(&users)
// WARNING: Age: 0 would be IGNORED because it is a zero value.
// Use map instead for zero-value conditions:
db.Where(map[string]interface{}{"name": "Alice", "age": 0}).Find(&users)

// Select specific columns
db.Select("name", "email").Find(&users)
// SELECT name, email FROM users

// Ordering, limit, offset
db.Order("created_at DESC").Limit(10).Offset(20).Find(&users)

// Count
var count int64
db.Model(&User{}).Where("role = ?", "admin").Count(&count)

// Exists check
var exists bool
db.Model(&User{}).
    Select("1").
    Where("email = ?", "alice@example.com").
    Find(&exists)

// Pluck - query single column into slice
var emails []string
db.Model(&User{}).Pluck("email", &emails)

// Distinct
var roles []string
db.Model(&User{}).Distinct("role").Pluck("role", &roles)
```

#### Update

```go
// Update single field
db.Model(&user).Update("name", "Alice Smith")
// UPDATE users SET name='Alice Smith', updated_at=NOW() WHERE id = 1

// Update multiple fields with struct (zero values ignored!)
db.Model(&user).Updates(User{Name: "Alice Smith", Age: 31})

// Update multiple fields with map (zero values included)
db.Model(&user).Updates(map[string]interface{}{
    "name":      "Alice Smith",
    "age":       31,
    "is_active": false, // false would be ignored with struct!
})

// Update with conditions
db.Model(&User{}).Where("role = ?", "user").Update("role", "member")

// Update column with expression
db.Model(&user).UpdateColumn("age", gorm.Expr("age + ?", 1))
// UPDATE users SET age = age + 1 WHERE id = 1

// Batch update without callbacks/hooks
db.Model(&User{}).Where("is_active = ?", false).
    UpdateColumn("role", "inactive")

// Save - updates ALL fields, including zero values
user.Name = "Alice Updated"
user.Age = 0 // This WILL be saved (unlike Updates with struct)
db.Save(&user)
```

> **Warning:** The difference between `Updates(struct)` and `Save` is critical.
> `Updates` with a struct ignores zero values. `Save` persists everything. When
> you need to set a field to its zero value (false, 0, ""), use `Save`, a map, or
> `Select` with `Updates`.

#### Delete

```go
// Soft delete (when model has DeletedAt field)
db.Delete(&user)
// UPDATE users SET deleted_at = NOW() WHERE id = 1

// Soft delete with conditions
db.Where("is_active = ?", false).Delete(&User{})

// Find soft-deleted records
db.Unscoped().Where("id = ?", 1).Find(&user)

// Permanently delete (skip soft delete)
db.Unscoped().Delete(&user)
// DELETE FROM users WHERE id = 1

// Delete by primary keys
db.Delete(&User{}, []int{1, 2, 3})
// DELETE FROM users WHERE id IN (1, 2, 3)
```

### Associations

#### Has One

```go
// Create user with profile
user := User{
    Name:  "Alice",
    Email: "alice@example.com",
    Profile: Profile{
        AvatarURL: "https://example.com/alice.jpg",
        Phone:     "+1234567890",
    },
}
db.Create(&user) // Creates both user and profile

// Find user with profile
var u User
db.Preload("Profile").First(&u, 1)
fmt.Println(u.Profile.Phone)

// Replace association
db.Model(&user).Association("Profile").Replace(&Profile{
    AvatarURL: "https://example.com/alice-new.jpg",
    Phone:     "+0987654321",
})
```

#### Has Many

```go
// Create user with orders
user := User{
    Name:  "Bob",
    Email: "bob@example.com",
    Orders: []Order{
        {TotalAmount: 99.99, Status: "completed"},
        {TotalAmount: 149.50, Status: "pending"},
    },
}
db.Create(&user)

// Append to association
db.Model(&user).Association("Orders").Append(&Order{
    TotalAmount: 50.00,
    Status:      "pending",
})

// Find with association count
var users []User
db.Preload("Orders").Find(&users)
for _, u := range users {
    fmt.Printf("%s has %d orders\n", u.Name, len(u.Orders))
}

// Count associations
count := db.Model(&user).Association("Orders").Count()
```

#### Many-to-Many

```go
// Create languages
golang := Language{Name: "Go", Code: "go"}
python := Language{Name: "Python", Code: "py"}
rust := Language{Name: "Rust", Code: "rs"}
db.Create(&golang)
db.Create(&python)
db.Create(&rust)

// Associate languages with user
db.Model(&user).Association("Languages").Append(&golang, &python)

// Find user with languages
var u User
db.Preload("Languages").First(&u, user.ID)
for _, lang := range u.Languages {
    fmt.Printf("  %s knows %s\n", u.Name, lang.Name)
}

// Remove from association (does not delete the language itself)
db.Model(&user).Association("Languages").Delete(&python)

// Replace all associations
db.Model(&user).Association("Languages").Replace(&golang, &rust)

// Clear all associations
db.Model(&user).Association("Languages").Clear()
```

#### Belongs To

```go
// Create company first
company := Company{Name: "Acme Corp", Domain: "acme.com"}
db.Create(&company)

// Create user belonging to company
user := User{
    Name:      "Eve",
    Email:     "eve@acme.com",
    CompanyID: &company.ID,
}
db.Create(&user)

// Preload the company
var u User
db.Preload("Company").First(&u, user.ID)
fmt.Printf("%s works at %s\n", u.Name, u.Company.Name)
```

### Hooks (Lifecycle Callbacks)

```go
// BeforeCreate hook - runs before INSERT
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.Role == "" {
        u.Role = "user"
    }
    // Validate email format
    if !strings.Contains(u.Email, "@") {
        return fmt.Errorf("invalid email: %s", u.Email)
    }
    return nil
}

// AfterCreate hook - runs after INSERT
func (u *User) AfterCreate(tx *gorm.DB) error {
    // Send welcome email, publish event, etc.
    log.Printf("User created: %s (ID: %d)", u.Name, u.ID)
    return nil
}

// BeforeUpdate hook
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // Audit log, validation, etc.
    if tx.Statement.Changed("Email") {
        log.Printf("Email changed for user %d", u.ID)
    }
    return nil
}

// AfterFind hook - runs after SELECT
func (u *User) AfterFind(tx *gorm.DB) error {
    // Post-processing: decrypt fields, compute derived values, etc.
    if u.Role == "" {
        u.Role = "user"
    }
    return nil
}

// BeforeDelete hook
func (u *User) BeforeDelete(tx *gorm.DB) error {
    // Prevent deletion of admin users
    if u.Role == "admin" {
        return fmt.Errorf("cannot delete admin user: %s", u.Name)
    }
    return nil
}

// Available hooks:
// BeforeSave / AfterSave     (called for both Create and Update)
// BeforeCreate / AfterCreate
// BeforeUpdate / AfterUpdate
// BeforeDelete / AfterDelete
// AfterFind
```

> **Tip:** Hooks execute within the same transaction. If a Before hook returns an
> error, the operation is rolled back. Use this for validation and enforcing
> business rules at the model layer.

### Scopes

Scopes are reusable query fragments.

```go
// Scope: active users
func ActiveUsers(db *gorm.DB) *gorm.DB {
    return db.Where("is_active = ?", true)
}

// Scope: users created in the last N days
func CreatedWithinDays(days int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        since := time.Now().AddDate(0, 0, -days)
        return db.Where("created_at >= ?", since)
    }
}

// Scope: paginate
func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page <= 0 {
            page = 1
        }
        if pageSize <= 0 || pageSize > 100 {
            pageSize = 20
        }
        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}

// Scope: order by field
func OrderBy(field, direction string) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        // Whitelist to prevent SQL injection
        allowed := map[string]bool{
            "id": true, "name": true, "created_at": true, "email": true,
        }
        if !allowed[field] {
            field = "id"
        }
        if direction != "ASC" && direction != "DESC" {
            direction = "ASC"
        }
        return db.Order(fmt.Sprintf("%s %s", field, direction))
    }
}

// Scope: filter by role
func WithRole(role string) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if role != "" {
            return db.Where("role = ?", role)
        }
        return db
    }
}

// Using scopes - they compose naturally
func ListUsers(db *gorm.DB, page, pageSize int, role string) ([]User, error) {
    var users []User
    err := db.Scopes(
        ActiveUsers,
        CreatedWithinDays(30),
        WithRole(role),
        Paginate(page, pageSize),
        OrderBy("created_at", "DESC"),
    ).Preload("Profile").Find(&users).Error
    return users, err
}
```

### Transactions

```go
// Method 1: Transaction closure (recommended - auto commit/rollback)
func CreateOrder(db *gorm.DB, userID uint, items []OrderItem) (*Order, error) {
    var order Order

    err := db.Transaction(func(tx *gorm.DB) error {
        // Check user exists
        var user User
        if err := tx.First(&user, userID).Error; err != nil {
            return fmt.Errorf("user not found: %w", err)
        }

        // Calculate total
        var total float64
        for _, item := range items {
            total += item.Price * float64(item.Quantity)
        }

        // Create order
        order = Order{
            UserID:      userID,
            TotalAmount: total,
            Status:      "pending",
        }
        if err := tx.Create(&order).Error; err != nil {
            return fmt.Errorf("create order: %w", err)
        }

        // Create order items
        for i := range items {
            items[i].OrderID = order.ID
        }
        if err := tx.Create(&items).Error; err != nil {
            return fmt.Errorf("create order items: %w", err)
        }

        // Update product stock (hypothetical)
        for _, item := range items {
            result := tx.Model(&Product{}).
                Where("id = ? AND stock >= ?", item.ProductID, item.Quantity).
                UpdateColumn("stock", gorm.Expr("stock - ?", item.Quantity))
            if result.RowsAffected == 0 {
                return fmt.Errorf("insufficient stock for product %d", item.ProductID)
            }
        }

        return nil // Commit
    })

    if err != nil {
        return nil, err
    }
    return &order, nil
}

// Method 2: Manual transaction control
func TransferFunds(db *gorm.DB, fromID, toID uint, amount float64) error {
    tx := db.Begin()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
        }
    }()

    if tx.Error != nil {
        return tx.Error
    }

    // Debit
    result := tx.Model(&Account{}).
        Where("id = ? AND balance >= ?", fromID, amount).
        UpdateColumn("balance", gorm.Expr("balance - ?", amount))
    if result.RowsAffected == 0 {
        tx.Rollback()
        return fmt.Errorf("insufficient funds or account not found: %d", fromID)
    }

    // Credit
    result = tx.Model(&Account{}).
        Where("id = ?", toID).
        UpdateColumn("balance", gorm.Expr("balance + ?", amount))
    if result.RowsAffected == 0 {
        tx.Rollback()
        return fmt.Errorf("destination account not found: %d", toID)
    }

    return tx.Commit().Error
}

// Nested transactions (savepoints)
func ComplexOperation(db *gorm.DB) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // Outer transaction work...
        tx.Create(&User{Name: "outer"})

        // Nested transaction (creates SAVEPOINT)
        err := tx.Transaction(func(tx2 *gorm.DB) error {
            tx2.Create(&User{Name: "inner"})
            // If this returns error, only inner work is rolled back
            return nil
        })
        if err != nil {
            // Inner transaction failed but outer continues
            log.Printf("inner transaction failed: %v", err)
        }

        return nil
    })
}
```

### Raw SQL Escape Hatch

```go
// Raw query
var users []User
db.Raw("SELECT * FROM users WHERE age > ? AND role = ?", 25, "admin").
    Scan(&users)

// Raw query with named arguments
db.Raw("SELECT * FROM users WHERE name = @name AND age = @age",
    sql.Named("name", "Alice"),
    sql.Named("age", 30),
).Scan(&users)

// Raw exec
db.Exec("UPDATE users SET is_active = ? WHERE last_login < ?",
    false, time.Now().AddDate(0, -6, 0))

// Scan into custom struct (not a GORM model)
type UserSummary struct {
    Role       string
    Count      int64
    AvgAge     float64
    MaxCreated time.Time
}
var summaries []UserSummary
db.Raw(`
    SELECT role,
           COUNT(*) as count,
           AVG(age) as avg_age,
           MAX(created_at) as max_created
    FROM users
    WHERE is_active = ?
    GROUP BY role
    ORDER BY count DESC
`, true).Scan(&summaries)

// Use raw SQL in specific clauses
db.Where("id IN (?)",
    db.Table("orders").Select("user_id").Where("status = ?", "vip"),
).Find(&users)
// SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE status = 'vip')

// Row-level access for maximum control
rows, err := db.Raw("SELECT name, email FROM users WHERE age > ?", 25).Rows()
if err != nil {
    log.Fatal(err)
}
defer rows.Close()
for rows.Next() {
    var name, email string
    rows.Scan(&name, &email)
    fmt.Printf("%s <%s>\n", name, email)
}
```

---

## GORM Advanced Features

### Preloading (Eager Loading)

```go
// Simple preload
var users []User
db.Preload("Profile").Preload("Orders").Find(&users)

// Nested preload
db.Preload("Orders.Items.Product").Find(&users)

// Conditional preload
db.Preload("Orders", "status = ?", "completed").Find(&users)

// Preload with custom query
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Where("status IN ?", []string{"completed", "shipped"}).
        Order("created_at DESC").
        Limit(5)
}).Find(&users)

// Preload All (use with caution - can be expensive)
db.Preload(clause.Associations).Find(&users)

// Joins preload (single query instead of N+1)
db.Joins("Profile").Find(&users)
// SELECT users.*, profiles.* FROM users LEFT JOIN profiles ON profiles.user_id = users.id

// Joins with conditions
db.Joins("Profile", db.Where(&Profile{Phone: "+1234567890"})).Find(&users)
```

> **Warning:** `Preload` fires separate SELECT queries per association.
> `Joins` uses SQL JOINs. For has-one/belongs-to, `Joins` is usually more
> efficient. For has-many and many-to-many, `Preload` prevents row duplication.

### Joins

```go
// Inner join
type UserOrder struct {
    UserName    string
    OrderTotal  float64
    OrderStatus string
}

var results []UserOrder
db.Model(&User{}).
    Select("users.name as user_name, orders.total_amount as order_total, orders.status as order_status").
    Joins("INNER JOIN orders ON orders.user_id = users.id").
    Where("orders.status = ?", "completed").
    Scan(&results)

// Left join with aggregation
type UserStats struct {
    UserID     uint
    Name       string
    OrderCount int64
    TotalSpent float64
}

var stats []UserStats
db.Model(&User{}).
    Select(`users.id as user_id,
            users.name,
            COUNT(orders.id) as order_count,
            COALESCE(SUM(orders.total_amount), 0) as total_spent`).
    Joins("LEFT JOIN orders ON orders.user_id = users.id").
    Group("users.id, users.name").
    Having("COUNT(orders.id) > ?", 0).
    Order("total_spent DESC").
    Scan(&stats)

// Subquery
subQuery := db.Model(&Order{}).
    Select("user_id, SUM(total_amount) as total").
    Group("user_id")

var richUsers []User
db.Joins("INNER JOIN (?) AS order_totals ON order_totals.user_id = users.id",
    subQuery).
    Where("order_totals.total > ?", 1000).
    Find(&richUsers)
```

### Custom Data Types

```go
import (
    "database/sql/driver"
    "encoding/json"
    "fmt"

    "gorm.io/gorm"
    "gorm.io/gorm/schema"
)

// JSON type - stores arbitrary JSON in a column
type JSON map[string]interface{}

// Value implements driver.Valuer
func (j JSON) Value() (driver.Value, error) {
    if j == nil {
        return nil, nil
    }
    bytes, err := json.Marshal(j)
    return string(bytes), err
}

// Scan implements sql.Scanner
func (j *JSON) Scan(value interface{}) error {
    if value == nil {
        *j = nil
        return nil
    }
    var bytes []byte
    switch v := value.(type) {
    case string:
        bytes = []byte(v)
    case []byte:
        bytes = v
    default:
        return fmt.Errorf("unsupported type: %T", value)
    }
    return json.Unmarshal(bytes, j)
}

// GormDataType implements schema.GormDataTypeInterface
func (JSON) GormDataType() string {
    return "json"
}

// StringSlice - stores []string as JSON array
type StringSlice []string

func (s StringSlice) Value() (driver.Value, error) {
    if s == nil {
        return nil, nil
    }
    bytes, err := json.Marshal(s)
    return string(bytes), err
}

func (s *StringSlice) Scan(value interface{}) error {
    if value == nil {
        *s = nil
        return nil
    }
    var bytes []byte
    switch v := value.(type) {
    case string:
        bytes = []byte(v)
    case []byte:
        bytes = v
    }
    return json.Unmarshal(bytes, s)
}

func (StringSlice) GormDataType() string {
    return "json"
}

// Encrypted string type
type EncryptedString string

func (e EncryptedString) Value() (driver.Value, error) {
    // In production, use a proper encryption library
    encrypted := encrypt(string(e))
    return encrypted, nil
}

func (e *EncryptedString) Scan(value interface{}) error {
    if value == nil {
        *e = ""
        return nil
    }
    decrypted := decrypt(value.(string))
    *e = EncryptedString(decrypted)
    return nil
}

// Using custom types in models
type UserProfile struct {
    gorm.Model
    UserID     uint
    Metadata   JSON           `gorm:"type:jsonb"`         // PostgreSQL JSONB
    Tags       StringSlice    `gorm:"type:json"`          // JSON array
    SSN        EncryptedString `gorm:"type:varchar(500)"` // Encrypted at rest
}
```

### Soft Delete

```go
// Basic soft delete is automatic with gorm.Model (includes DeletedAt)
// When you call db.Delete(&user), it sets deleted_at instead of removing the row.

// Query excluding soft-deleted records (default behavior)
var users []User
db.Find(&users) // WHERE deleted_at IS NULL

// Query including soft-deleted records
db.Unscoped().Find(&users) // No WHERE clause for deleted_at

// Permanently delete
db.Unscoped().Delete(&user)

// Restore soft-deleted record
db.Model(&user).Unscoped().Update("deleted_at", nil)

// Custom soft delete with Unix timestamp instead of nullable timestamp
type Flight struct {
    ID        uint
    Name      string
    DeletedAt soft_delete.DeletedAt `gorm:"softDelete:milli"` // Unix ms
    // or
    // IsDel soft_delete.DeletedAt `gorm:"softDelete:flag"` // Boolean flag
}

// Flagged soft delete (uses integer column: 0 = not deleted, 1 = deleted)
type Document struct {
    ID    uint
    Title string
    IsDel soft_delete.DeletedAt `gorm:"softDelete:flag;column:is_deleted"`
}
```

### Sharding Plugin

```go
import (
    "gorm.io/gorm"
    "gorm.io/sharding"
)

// Configure sharding
db.Use(sharding.Register(sharding.Config{
    ShardingKey:         "user_id",
    NumberOfShards:      4,
    PrimaryKeyGenerator: sharding.PKSnowflake,
}, "orders"))
// This will route queries to orders_0, orders_1, orders_2, orders_3
// based on user_id % 4

// Usage is transparent
db.Create(&Order{UserID: 42, TotalAmount: 99.99})
// Automatically inserts into orders_2 (42 % 4 = 2)

db.Where("user_id = ?", 42).Find(&orders)
// Automatically queries orders_2
```

### GORM Plugins and Middleware

```go
// Custom plugin interface
type MyPlugin struct{}

func (p *MyPlugin) Name() string {
    return "my_plugin"
}

func (p *MyPlugin) Initialize(db *gorm.DB) error {
    // Register callbacks
    db.Callback().Query().Before("gorm:query").Register("my_plugin:before_query",
        func(db *gorm.DB) {
            // Add tenant filtering, logging, metrics, etc.
            start := time.Now()
            db.InstanceSet("query_start", start)
        },
    )

    db.Callback().Query().After("gorm:query").Register("my_plugin:after_query",
        func(db *gorm.DB) {
            if start, ok := db.InstanceGet("query_start"); ok {
                duration := time.Since(start.(time.Time))
                log.Printf("Query took %v: %s", duration, db.Statement.SQL.String())
            }
        },
    )
    return nil
}

// Register plugin
db.Use(&MyPlugin{})

// Prometheus metrics plugin (using gorm's prometheus package)
import "gorm.io/plugin/prometheus"

db.Use(prometheus.New(prometheus.Config{
    DBName:          "mydb",
    RefreshInterval: 15, // seconds
    MetricsCollector: []prometheus.MetricsCollector{
        &prometheus.Postgres{},
    },
}))
```

---

## ent by Facebook

### Overview

`ent` is an entity framework for Go built by Facebook (Meta). It takes a
schema-as-code approach: you define your entities and their relationships in Go,
then `ent` generates a fully type-safe, statically-typed API for querying and
mutating your data.

### Installation

```bash
go get -d entgo.io/ent/cmd/ent
```

### Initializing a Schema

```bash
# Create a new schema
go run -mod=mod entgo.io/ent/cmd/ent new User
go run -mod=mod entgo.io/ent/cmd/ent new Post
go run -mod=mod entgo.io/ent/cmd/ent new Comment
go run -mod=mod entgo.io/ent/cmd/ent new Tag
go run -mod=mod entgo.io/ent/cmd/ent new Group
```

This creates schema files in `ent/schema/`.

### Schema Definition

```go
// ent/schema/user.go
package schema

import (
    "regexp"
    "time"

    "entgo.io/ent"
    "entgo.io/ent/dialect/entsql"
    "entgo.io/ent/schema"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/index"
    "entgo.io/ent/schema/mixin"
)

// TimeMixin adds created_at and updated_at to any schema.
type TimeMixin struct {
    mixin.Schema
}

func (TimeMixin) Fields() []ent.Field {
    return []ent.Field{
        field.Time("created_at").
            Default(time.Now).
            Immutable(),
        field.Time("updated_at").
            Default(time.Now).
            UpdateDefault(time.Now),
    }
}

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Mixin of the User.
func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        TimeMixin{},
    }
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            NotEmpty().
            MaxLen(100).
            Comment("The user's display name"),
        field.String("email").
            NotEmpty().
            MaxLen(255).
            Unique().
            Match(regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)).
            Comment("Primary email address"),
        field.Int("age").
            Positive().
            Optional().
            Nillable().
            Comment("User age in years"),
        field.Enum("role").
            Values("user", "admin", "moderator").
            Default("user"),
        field.Enum("status").
            Values("active", "inactive", "banned").
            Default("active"),
        field.String("password_hash").
            Sensitive(). // Won't appear in JSON or logs
            NotEmpty(),
        field.JSON("metadata", map[string]interface{}{}).
            Optional().
            Comment("Arbitrary JSON metadata"),
        field.Bool("is_verified").
            Default(false),
    }
}

// Edges of the User (relationships).
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        // User has one Profile (O2O)
        edge.To("profile", Profile.Type).
            Unique().
            Comment("User's profile"),
        // User has many Posts (O2M)
        edge.To("posts", Post.Type).
            Comment("Posts authored by this user"),
        // User has many Comments
        edge.To("comments", Comment.Type),
        // User belongs to many Groups (M2M)
        edge.To("groups", Group.Type).
            Comment("Groups the user is a member of"),
        // User follows other Users (M2M self-referential)
        edge.To("following", User.Type).
            From("followers"),
    }
}

// Indexes of the User.
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("email").
            Unique(),
        index.Fields("role", "status"),
        index.Fields("created_at"),
    }
}

// Annotations of the User.
func (User) Annotations() []schema.Annotation {
    return []schema.Annotation{
        entsql.Annotation{
            Table: "users",
        },
    }
}
```

```go
// ent/schema/post.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/index"
)

type Post struct {
    ent.Schema
}

func (Post) Mixin() []ent.Mixin {
    return []ent.Mixin{
        TimeMixin{},
    }
}

func (Post) Fields() []ent.Field {
    return []ent.Field{
        field.String("title").
            NotEmpty().
            MaxLen(200),
        field.Text("body").
            NotEmpty(),
        field.Enum("status").
            Values("draft", "published", "archived").
            Default("draft"),
        field.Int("view_count").
            Default(0).
            NonNegative(),
        field.Time("published_at").
            Optional().
            Nillable(),
    }
}

func (Post) Edges() []ent.Edge {
    return []ent.Edge{
        // Post belongs to User (back-reference of User.posts)
        edge.From("author", User.Type).
            Ref("posts").
            Unique().
            Required(),
        // Post has many Comments
        edge.To("comments", Comment.Type),
        // Post has many Tags (M2M)
        edge.To("tags", Tag.Type),
    }
}

func (Post) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("status", "created_at"),
    }
}
```

```go
// ent/schema/comment.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
)

type Comment struct {
    ent.Schema
}

func (Comment) Mixin() []ent.Mixin {
    return []ent.Mixin{
        TimeMixin{},
    }
}

func (Comment) Fields() []ent.Field {
    return []ent.Field{
        field.Text("body").
            NotEmpty(),
    }
}

func (Comment) Edges() []ent.Edge {
    return []ent.Edge{
        // Comment belongs to a Post
        edge.From("post", Post.Type).
            Ref("comments").
            Unique().
            Required(),
        // Comment belongs to a User
        edge.From("author", User.Type).
            Ref("comments").
            Unique().
            Required(),
        // Self-referential: replies
        edge.To("replies", Comment.Type).
            From("parent").
            Unique(),
    }
}
```

```go
// ent/schema/tag.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
)

type Tag struct {
    ent.Schema
}

func (Tag) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            NotEmpty().
            Unique().
            MaxLen(50),
        field.String("slug").
            NotEmpty().
            Unique().
            MaxLen(50),
    }
}

func (Tag) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("posts", Post.Type).
            Ref("tags"),
    }
}
```

```go
// ent/schema/group.go
package schema

import (
    "entgo.io/ent"
    "entgo.io/ent/schema/edge"
    "entgo.io/ent/schema/field"
)

type Group struct {
    ent.Schema
}

func (Group) Mixin() []ent.Mixin {
    return []ent.Mixin{
        TimeMixin{},
    }
}

func (Group) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").
            NotEmpty().
            MaxLen(100),
        field.String("description").
            Optional().
            MaxLen(500),
        field.Int("max_members").
            Default(100).
            Positive(),
    }
}

func (Group) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("members", User.Type).
            Ref("groups"),
    }
}
```

### Code Generation

```bash
# Generate the ent package
go generate ./ent

# Or run directly
go run -mod=mod entgo.io/ent/cmd/ent generate ./ent/schema
```

This generates a complete type-safe client in the `ent/` directory with:
- Client and transaction types
- CRUD builders for each entity
- Predicate functions for filtering
- Edge (relationship) traversal methods
- Migration code

### Using the Generated Client

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "yourproject/ent"
    "yourproject/ent/post"
    "yourproject/ent/user"
    "yourproject/ent/comment"

    _ "github.com/lib/pq"
)

func main() {
    // Create client
    client, err := ent.Open("postgres",
        "host=localhost port=5432 user=postgres dbname=myapp sslmode=disable",
    )
    if err != nil {
        log.Fatalf("failed opening connection: %v", err)
    }
    defer client.Close()

    ctx := context.Background()

    // Run auto migration
    if err := client.Schema.Create(ctx); err != nil {
        log.Fatalf("failed creating schema: %v", err)
    }

    // CRUD operations
    if err := crudExample(ctx, client); err != nil {
        log.Fatal(err)
    }
}

func crudExample(ctx context.Context, client *ent.Client) error {
    // CREATE
    alice, err := client.User.
        Create().
        SetName("Alice").
        SetEmail("alice@example.com").
        SetAge(30).
        SetRole(user.RoleAdmin).
        SetStatus(user.StatusActive).
        SetPasswordHash("$2a$10$...").
        SetIsVerified(true).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("creating user: %w", err)
    }
    fmt.Printf("Created user: %v\n", alice)

    // Bulk create
    bulk := make([]*ent.UserCreate, 3)
    names := []string{"Bob", "Charlie", "Diana"}
    for i, name := range names {
        bulk[i] = client.User.Create().
            SetName(name).
            SetEmail(fmt.Sprintf("%s@example.com", name)).
            SetPasswordHash("$2a$10$...").
            SetRole(user.RoleUser)
    }
    users, err := client.User.CreateBulk(bulk...).Save(ctx)
    if err != nil {
        return fmt.Errorf("bulk creating users: %w", err)
    }

    // READ - single
    u, err := client.User.Get(ctx, alice.ID)
    if err != nil {
        return fmt.Errorf("getting user: %w", err)
    }
    fmt.Printf("Got user: %s\n", u.Name)

    // READ - query with filters
    admins, err := client.User.
        Query().
        Where(
            user.RoleEQ(user.RoleAdmin),
            user.StatusEQ(user.StatusActive),
        ).
        Order(ent.Desc(user.FieldCreatedAt)).
        Limit(10).
        All(ctx)
    if err != nil {
        return fmt.Errorf("querying admins: %w", err)
    }
    fmt.Printf("Found %d admins\n", len(admins))

    // READ - query with OR conditions
    filtered, err := client.User.
        Query().
        Where(
            user.Or(
                user.RoleEQ(user.RoleAdmin),
                user.AgeGT(25),
            ),
        ).
        All(ctx)
    if err != nil {
        return fmt.Errorf("filtered query: %w", err)
    }
    _ = filtered

    // READ - exists check
    exists, err := client.User.
        Query().
        Where(user.EmailEQ("alice@example.com")).
        Exist(ctx)
    if err != nil {
        return fmt.Errorf("exists check: %w", err)
    }
    fmt.Printf("Alice exists: %v\n", exists)

    // READ - count
    count, err := client.User.
        Query().
        Where(user.StatusEQ(user.StatusActive)).
        Count(ctx)
    if err != nil {
        return fmt.Errorf("counting: %w", err)
    }
    fmt.Printf("Active users: %d\n", count)

    // UPDATE
    updated, err := client.User.
        UpdateOneID(alice.ID).
        SetName("Alice Smith").
        SetAge(31).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("updating user: %w", err)
    }
    fmt.Printf("Updated user: %s, age: %d\n", updated.Name, *updated.Age)

    // Bulk update
    affected, err := client.User.
        Update().
        Where(user.StatusEQ(user.StatusInactive)).
        SetStatus(user.StatusBanned).
        Save(ctx)
    if err != nil {
        return fmt.Errorf("bulk update: %w", err)
    }
    fmt.Printf("Bulk updated %d users\n", affected)

    // DELETE
    err = client.User.
        DeleteOneID(users[0].ID).
        Exec(ctx)
    if err != nil {
        return fmt.Errorf("deleting user: %w", err)
    }

    return nil
}
```

### Edges (Relations) and Graph Traversal

```go
func edgesExample(ctx context.Context, client *ent.Client) error {
    // Create user
    alice, err := client.User.
        Create().
        SetName("Alice").
        SetEmail("alice@example.com").
        SetPasswordHash("...").
        Save(ctx)
    if err != nil {
        return err
    }

    // Create posts for user
    post1, err := client.Post.
        Create().
        SetTitle("Introduction to ent").
        SetBody("ent is a powerful entity framework...").
        SetStatus(post.StatusPublished).
        SetPublishedAt(time.Now()).
        SetAuthor(alice). // Set the edge
        Save(ctx)
    if err != nil {
        return err
    }

    // Create tags
    goTag, _ := client.Tag.Create().SetName("Go").SetSlug("go").Save(ctx)
    dbTag, _ := client.Tag.Create().SetName("Database").SetSlug("database").Save(ctx)

    // Add tags to post (M2M)
    post1.Update().AddTags(goTag, dbTag).Save(ctx)

    // Create comment on post
    _, err = client.Comment.
        Create().
        SetBody("Great post!").
        SetPost(post1).
        SetAuthor(alice).
        Save(ctx)
    if err != nil {
        return err
    }

    // --- Graph Traversal ---

    // Get all posts by a user
    posts, err := alice.QueryPosts().
        Where(post.StatusEQ(post.StatusPublished)).
        Order(ent.Desc(post.FieldCreatedAt)).
        All(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("Alice has %d published posts\n", len(posts))

    // Get the author of a post
    author, err := post1.QueryAuthor().Only(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("Post author: %s\n", author.Name)

    // Complex traversal: get all comments on all posts by Alice
    comments, err := client.User.
        Query().
        Where(user.NameEQ("Alice")).
        QueryPosts().
        QueryComments().
        All(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("Total comments on Alice's posts: %d\n", len(comments))

    // Traverse: find all users who commented on posts tagged with "Go"
    commenters, err := client.Tag.
        Query().
        Where(tag.NameEQ("Go")).
        QueryPosts().
        QueryComments().
        QueryAuthor().
        All(ctx)
    if err != nil {
        return err
    }
    fmt.Printf("Users who commented on Go posts: %d\n", len(commenters))

    // Eager loading (like Preload in GORM)
    usersWithPosts, err := client.User.
        Query().
        WithPosts(func(q *ent.PostQuery) {
            q.WithComments() // Nested eager loading
            q.WithTags()
            q.Where(post.StatusEQ(post.StatusPublished))
            q.Order(ent.Desc(post.FieldCreatedAt))
            q.Limit(5)
        }).
        WithProfile().
        All(ctx)
    if err != nil {
        return err
    }
    for _, u := range usersWithPosts {
        fmt.Printf("User: %s\n", u.Name)
        for _, p := range u.Edges.Posts {
            fmt.Printf("  Post: %s (%d comments)\n", p.Title, len(p.Edges.Comments))
            for _, t := range p.Edges.Tags {
                fmt.Printf("    Tag: %s\n", t.Name)
            }
        }
    }

    return nil
}
```

### Aggregation

```go
func aggregationExample(ctx context.Context, client *ent.Client) error {
    // Count users by role
    var results []struct {
        Role  user.Role `json:"role"`
        Count int       `json:"count"`
    }
    err := client.User.
        Query().
        GroupBy(user.FieldRole).
        Aggregate(ent.Count()).
        Scan(ctx, &results)
    if err != nil {
        return err
    }
    for _, r := range results {
        fmt.Printf("Role %s: %d users\n", r.Role, r.Count)
    }

    // Aggregate: average age by role
    var ageStats []struct {
        Role   user.Role `json:"role"`
        AvgAge float64   `json:"avg_age"`
        MaxAge int       `json:"max_age"`
        MinAge int       `json:"min_age"`
    }
    err = client.User.
        Query().
        Where(user.AgeNotNil()).
        GroupBy(user.FieldRole).
        Aggregate(
            ent.Mean(user.FieldAge),
            ent.Max(user.FieldAge),
            ent.Min(user.FieldAge),
        ).
        Scan(ctx, &ageStats)
    if err != nil {
        return err
    }

    // Aggregate: total view count across posts
    var postStats []struct {
        Sum   int `json:"sum"`
        Count int `json:"count"`
    }
    err = client.Post.
        Query().
        Where(post.StatusEQ(post.StatusPublished)).
        Aggregate(
            ent.Sum(post.FieldViewCount),
            ent.Count(),
        ).
        Scan(ctx, &postStats)

    return err
}
```

### Privacy Layer

ent's privacy layer lets you define access control rules at the schema level.

```go
// ent/schema/user.go
import (
    "context"

    "entgo.io/ent"
    "entgo.io/ent/privacy"
)

// Policy defines the privacy policy of the User.
func (User) Policy() ent.Policy {
    return privacy.Policy{
        Mutation: privacy.MutationPolicy{
            // Only admins can create users
            rule.AllowIfAdmin(),
            // Users can update their own records
            rule.AllowIfOwnRecord(),
            // Deny everything else
            privacy.AlwaysDenyRule(),
        },
        Query: privacy.QueryPolicy{
            // Anyone can query users
            privacy.AlwaysAllowRule(),
        },
    }
}

// Viewer represents the authenticated user in context.
type Viewer struct {
    ID   int
    Role string
}

type viewerKey struct{}

func WithViewer(ctx context.Context, v *Viewer) context.Context {
    return context.WithValue(ctx, viewerKey{}, v)
}

func ViewerFromContext(ctx context.Context) *Viewer {
    v, _ := ctx.Value(viewerKey{}).(*Viewer)
    return v
}

// Privacy rules
package rule

import (
    "context"

    "entgo.io/ent/privacy"
    "yourproject/ent"
)

// AllowIfAdmin allows the operation if the viewer is an admin.
func AllowIfAdmin() privacy.MutationRule {
    return privacy.MutationRuleFunc(func(ctx context.Context, m ent.Mutation) error {
        viewer := ViewerFromContext(ctx)
        if viewer != nil && viewer.Role == "admin" {
            return privacy.Allow
        }
        return privacy.Skip // Let the next rule decide
    })
}

// AllowIfOwnRecord allows users to update their own records.
func AllowIfOwnRecord() privacy.MutationRule {
    return privacy.MutationRuleFunc(func(ctx context.Context, m ent.Mutation) error {
        viewer := ViewerFromContext(ctx)
        if viewer == nil {
            return privacy.Skip
        }
        // Check if the mutation is for the viewer's own record
        if um, ok := m.(*ent.UserMutation); ok {
            id, exists := um.ID()
            if exists && id == viewer.ID {
                return privacy.Allow
            }
        }
        return privacy.Skip
    })
}
```

### ent Hooks

```go
// ent/schema/user.go

// Hooks of the User.
func (User) Hooks() []ent.Hook {
    return []ent.Hook{
        // Audit log hook
        func(next ent.Mutator) ent.Mutator {
            return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
                start := time.Now()
                v, err := next.Mutate(ctx, m)
                duration := time.Since(start)

                log.Printf(
                    "Op=%s Type=%s Duration=%v Error=%v",
                    m.Op(), m.Type(), duration, err,
                )
                return v, err
            })
        },
        // Normalize email hook
        hook.On(
            func(next ent.Mutator) ent.Mutator {
                return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
                    if email, exists := m.Email(); exists {
                        m.SetEmail(strings.ToLower(strings.TrimSpace(email)))
                    }
                    return next.Mutate(ctx, m)
                })
            },
            ent.OpCreate|ent.OpUpdate|ent.OpUpdateOne,
        ),
    }
}
```

### Transactions in ent

```go
func createUserWithPosts(ctx context.Context, client *ent.Client) error {
    // Transaction with closure
    tx, err := client.Tx(ctx)
    if err != nil {
        return fmt.Errorf("starting transaction: %w", err)
    }

    // Use tx instead of client inside transaction
    alice, err := tx.User.
        Create().
        SetName("Alice").
        SetEmail("alice@example.com").
        SetPasswordHash("...").
        Save(ctx)
    if err != nil {
        return rollback(tx, fmt.Errorf("creating user: %w", err))
    }

    _, err = tx.Post.
        Create().
        SetTitle("My First Post").
        SetBody("Hello world!").
        SetAuthor(alice).
        Save(ctx)
    if err != nil {
        return rollback(tx, fmt.Errorf("creating post: %w", err))
    }

    return tx.Commit()
}

// Helper to rollback and wrap error
func rollback(tx *ent.Tx, err error) error {
    if rerr := tx.Rollback(); rerr != nil {
        err = fmt.Errorf("%w: rolling back: %v", err, rerr)
    }
    return err
}

// Alternative: WithTx helper for cleaner transaction usage
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    defer func() {
        if v := recover(); v != nil {
            tx.Rollback()
            panic(v)
        }
    }()
    if err := fn(tx); err != nil {
        if rerr := tx.Rollback(); rerr != nil {
            err = fmt.Errorf("%w: rolling back: %v", err, rerr)
        }
        return err
    }
    return tx.Commit()
}

// Usage
func createOrderWithTx(ctx context.Context, client *ent.Client) error {
    return WithTx(ctx, client, func(tx *ent.Tx) error {
        user, err := tx.User.Query().
            Where(user.EmailEQ("alice@example.com")).
            Only(ctx)
        if err != nil {
            return err
        }

        _, err = tx.Post.Create().
            SetTitle("Transactional Post").
            SetBody("This is atomic").
            SetAuthor(user).
            Save(ctx)
        return err
    })
}
```

---

## ent Advanced Features

### Custom Templates

ent supports Go templates for generating additional code.

```go
// ent/entc.go
//go:build ignore

package main

import (
    "log"

    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{
        Templates: []*gen.Template{
            // Add custom template for generating GraphQL resolvers
            gen.MustParse(gen.NewTemplate("graphql").
                ParseFiles("./template/graphql.tmpl")),
            // Add custom template for generating REST handlers
            gen.MustParse(gen.NewTemplate("rest").
                ParseFiles("./template/rest.tmpl")),
        },
        Features: []gen.Feature{
            gen.FeaturePrivacy,
            gen.FeatureEntQL,
            gen.FeatureSnapshot,
            gen.FeatureModifier,   // Allow raw SQL modifiers
            gen.FeatureUpsert,     // Upsert support
            gen.FeatureIntercept,  // Query interceptors
        },
    })
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

```bash
# Run code generation with entc.go
go generate ./ent
```

### Migration with Atlas

ent integrates with Atlas for versioned, production-grade migrations.

```go
// ent/entc.go
//go:build ignore

package main

import (
    "log"

    "ariga.io/atlas/sql/sqltool"
    "entgo.io/ent/entc"
    "entgo.io/ent/entc/gen"
    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    err := entc.Generate("./schema", &gen.Config{},
        entc.Extensions(
            // Use Atlas for migrations
            entc.FeatureSnapshot,
        ),
    )
    if err != nil {
        log.Fatalf("running ent codegen: %v", err)
    }
}
```

```bash
# Install Atlas CLI
curl -sSf https://atlasgo.sh | sh

# Generate migration files from ent schema changes
atlas migrate diff migration_name \
    --dir "file://ent/migrate/migrations" \
    --to "ent://ent/schema" \
    --dev-url "docker://postgres/15/test?search_path=public"

# Apply migrations
atlas migrate apply \
    --dir "file://ent/migrate/migrations" \
    --url "postgres://user:pass@localhost:5432/mydb?sslmode=disable"

# View migration status
atlas migrate status \
    --dir "file://ent/migrate/migrations" \
    --url "postgres://user:pass@localhost:5432/mydb?sslmode=disable"
```

Programmatic migration in Go:

```go
package main

import (
    "context"
    "log"

    atlas "ariga.io/atlas/sql/migrate"
    "entgo.io/ent/dialect/sql/schema"
    "yourproject/ent"
    "yourproject/ent/migrate"

    _ "github.com/lib/pq"
)

func runMigrations(ctx context.Context) error {
    client, err := ent.Open("postgres", "postgres://...")
    if err != nil {
        return err
    }
    defer client.Close()

    // Auto migration with diff-based approach
    err = client.Schema.Create(ctx,
        migrate.WithDropColumn(true),  // Allow dropping columns
        migrate.WithDropIndex(true),   // Allow dropping indexes
        migrate.WithGlobalUniqueID(true), // Use global unique IDs
        schema.WithAtlas(true),        // Use Atlas engine
    )
    return err
}
```

### Testing with ent

```go
package service_test

import (
    "context"
    "testing"

    "yourproject/ent"
    "yourproject/ent/enttest"
    "yourproject/ent/user"

    _ "github.com/mattn/go-sqlite3"
)

func TestUserCRUD(t *testing.T) {
    // Create a test client with SQLite in-memory database
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1")
    defer client.Close()

    ctx := context.Background()

    // Test Create
    u, err := client.User.
        Create().
        SetName("Test User").
        SetEmail("test@example.com").
        SetPasswordHash("hashed").
        Save(ctx)
    if err != nil {
        t.Fatalf("creating user: %v", err)
    }
    if u.Name != "Test User" {
        t.Errorf("expected name 'Test User', got %q", u.Name)
    }

    // Test Read
    found, err := client.User.Get(ctx, u.ID)
    if err != nil {
        t.Fatalf("getting user: %v", err)
    }
    if found.Email != "test@example.com" {
        t.Errorf("expected email 'test@example.com', got %q", found.Email)
    }

    // Test Update
    updated, err := client.User.
        UpdateOneID(u.ID).
        SetName("Updated Name").
        Save(ctx)
    if err != nil {
        t.Fatalf("updating user: %v", err)
    }
    if updated.Name != "Updated Name" {
        t.Errorf("expected name 'Updated Name', got %q", updated.Name)
    }

    // Test Query
    count, err := client.User.
        Query().
        Where(user.RoleEQ(user.RoleUser)).
        Count(ctx)
    if err != nil {
        t.Fatalf("counting users: %v", err)
    }
    if count != 1 {
        t.Errorf("expected 1 user, got %d", count)
    }

    // Test Delete
    err = client.User.DeleteOneID(u.ID).Exec(ctx)
    if err != nil {
        t.Fatalf("deleting user: %v", err)
    }

    exists, err := client.User.
        Query().
        Where(user.IDEQ(u.ID)).
        Exist(ctx)
    if err != nil {
        t.Fatalf("checking existence: %v", err)
    }
    if exists {
        t.Error("expected user to be deleted")
    }
}

func TestUserEdges(t *testing.T) {
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1")
    defer client.Close()
    ctx := context.Background()

    // Create user
    u, _ := client.User.Create().
        SetName("Alice").
        SetEmail("alice@test.com").
        SetPasswordHash("hash").
        Save(ctx)

    // Create posts
    p1, _ := client.Post.Create().
        SetTitle("Post 1").
        SetBody("Body 1").
        SetAuthor(u).
        Save(ctx)

    p2, _ := client.Post.Create().
        SetTitle("Post 2").
        SetBody("Body 2").
        SetAuthor(u).
        Save(ctx)

    // Query posts through user edge
    posts, err := u.QueryPosts().All(ctx)
    if err != nil {
        t.Fatalf("querying posts: %v", err)
    }
    if len(posts) != 2 {
        t.Errorf("expected 2 posts, got %d", len(posts))
    }

    // Verify back-reference
    author, err := p1.QueryAuthor().Only(ctx)
    if err != nil {
        t.Fatalf("querying author: %v", err)
    }
    if author.ID != u.ID {
        t.Errorf("expected author ID %d, got %d", u.ID, author.ID)
    }

    _ = p2 // Used above
}
```

---

## sqlc: Write SQL, Get Go

### Overview

sqlc takes the opposite approach to ORMs: you write SQL queries directly, and sqlc
generates type-safe Go code from them. The key insight is that SQL is already a
well-designed query language - so rather than building an abstraction over it, sqlc
just makes it type-safe.

### Installation

```bash
# Install sqlc
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest

# Or with Homebrew
brew install sqlc
```

### Configuration

```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "query/"
    schema: "schema/"
    gen:
      go:
        package: "db"
        out: "internal/db"
        sql_package: "pgx/v5"  # or "database/sql"
        emit_json_tags: true
        emit_prepared_queries: true
        emit_interface: true
        emit_exact_table_names: false
        emit_empty_slices: true
        emit_exported_queries: false
        emit_result_struct_pointers: false
        emit_params_struct_pointers: false
        emit_methods_with_db_argument: false
        emit_pointers_for_null_types: true
        emit_enum_valid_method: true
        emit_all_enum_values: true
        overrides:
          - db_type: "uuid"
            go_type: "github.com/google/uuid.UUID"
          - db_type: "timestamptz"
            go_type: "time.Time"
          - db_type: "text"
            go_type: "string"
          - db_type: "pg_catalog.numeric"
            go_type:
              import: "github.com/shopspring/decimal"
              type: "Decimal"
```

### Schema Definition

```sql
-- schema/001_users.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    age         INTEGER CHECK (age >= 0),
    role        VARCHAR(20) NOT NULL DEFAULT 'user',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role_active ON users(role, is_active);

-- schema/002_posts.sql
CREATE TABLE posts (
    id           BIGSERIAL PRIMARY KEY,
    author_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title        VARCHAR(200) NOT NULL,
    body         TEXT NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'draft',
    view_count   INTEGER NOT NULL DEFAULT 0,
    published_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_author ON posts(author_id);
CREATE INDEX idx_posts_status_created ON posts(status, created_at DESC);

-- schema/003_comments.sql
CREATE TABLE comments (
    id         BIGSERIAL PRIMARY KEY,
    post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id  BIGINT REFERENCES comments(id) ON DELETE CASCADE,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_comments_post ON comments(post_id);
CREATE INDEX idx_comments_author ON comments(author_id);

-- schema/004_tags.sql
CREATE TABLE tags (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    slug VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE post_tags (
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- schema/005_orders.sql
CREATE TABLE orders (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL REFERENCES users(id),
    total_amount NUMERIC(10, 2) NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id         BIGSERIAL PRIMARY KEY,
    order_id   BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL,
    quantity   INTEGER NOT NULL CHECK (quantity > 0),
    price      NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Query Definitions

```sql
-- query/users.sql

-- name: GetUser :one
SELECT * FROM users
WHERE id = $1 LIMIT 1;

-- name: GetUserByEmail :one
SELECT * FROM users
WHERE email = $1 LIMIT 1;

-- name: ListUsers :many
SELECT * FROM users
WHERE is_active = true
ORDER BY created_at DESC
LIMIT $1 OFFSET $2;

-- name: ListUsersByRole :many
SELECT * FROM users
WHERE role = $1 AND is_active = true
ORDER BY name ASC;

-- name: SearchUsers :many
SELECT * FROM users
WHERE
    (name ILIKE '%' || $1 || '%' OR email ILIKE '%' || $1 || '%')
    AND is_active = true
ORDER BY name ASC
LIMIT $2 OFFSET $3;

-- name: CreateUser :one
INSERT INTO users (name, email, age, role, metadata)
VALUES ($1, $2, $3, $4, $5)
RETURNING *;

-- name: UpdateUser :one
UPDATE users
SET name = $2,
    email = $3,
    age = $4,
    role = $5,
    updated_at = NOW()
WHERE id = $1
RETURNING *;

-- name: UpdateUserEmail :exec
UPDATE users
SET email = $2, updated_at = NOW()
WHERE id = $1;

-- name: DeactivateUser :exec
UPDATE users
SET is_active = false, updated_at = NOW()
WHERE id = $1;

-- name: DeleteUser :exec
DELETE FROM users WHERE id = $1;

-- name: CountUsersByRole :one
SELECT role, COUNT(*) as count
FROM users
WHERE is_active = true
GROUP BY role
HAVING role = $1;

-- name: GetUserStats :many
SELECT
    role,
    COUNT(*) as user_count,
    AVG(age)::FLOAT as avg_age,
    MIN(created_at) as earliest_signup
FROM users
WHERE is_active = true
GROUP BY role
ORDER BY user_count DESC;

-- name: UserExists :one
SELECT EXISTS(
    SELECT 1 FROM users WHERE email = $1
) AS exists;
```

```sql
-- query/posts.sql

-- name: GetPost :one
SELECT * FROM posts WHERE id = $1 LIMIT 1;

-- name: ListPostsByAuthor :many
SELECT * FROM posts
WHERE author_id = $1
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;

-- name: ListPublishedPosts :many
SELECT * FROM posts
WHERE status = 'published'
ORDER BY published_at DESC
LIMIT $1 OFFSET $2;

-- name: CreatePost :one
INSERT INTO posts (author_id, title, body, status, published_at)
VALUES ($1, $2, $3, $4, $5)
RETURNING *;

-- name: PublishPost :one
UPDATE posts
SET status = 'published',
    published_at = NOW(),
    updated_at = NOW()
WHERE id = $1
RETURNING *;

-- name: IncrementViewCount :exec
UPDATE posts
SET view_count = view_count + 1
WHERE id = $1;

-- name: DeletePost :exec
DELETE FROM posts WHERE id = $1;

-- name: GetPostWithAuthor :one
SELECT
    p.id,
    p.title,
    p.body,
    p.status,
    p.view_count,
    p.published_at,
    p.created_at,
    u.id AS author_id,
    u.name AS author_name,
    u.email AS author_email
FROM posts p
INNER JOIN users u ON u.id = p.author_id
WHERE p.id = $1;

-- name: ListPostsWithAuthors :many
SELECT
    p.id,
    p.title,
    p.body,
    p.status,
    p.view_count,
    p.published_at,
    p.created_at,
    u.id AS author_id,
    u.name AS author_name
FROM posts p
INNER JOIN users u ON u.id = p.author_id
WHERE p.status = 'published'
ORDER BY p.published_at DESC
LIMIT $1 OFFSET $2;

-- name: GetPostsByTag :many
SELECT p.*
FROM posts p
INNER JOIN post_tags pt ON pt.post_id = p.id
INNER JOIN tags t ON t.id = pt.tag_id
WHERE t.slug = $1 AND p.status = 'published'
ORDER BY p.published_at DESC
LIMIT $2 OFFSET $3;
```

```sql
-- query/tags.sql

-- name: GetTag :one
SELECT * FROM tags WHERE id = $1;

-- name: GetTagBySlug :one
SELECT * FROM tags WHERE slug = $1;

-- name: ListTags :many
SELECT * FROM tags ORDER BY name;

-- name: CreateTag :one
INSERT INTO tags (name, slug)
VALUES ($1, $2)
RETURNING *;

-- name: AddTagToPost :exec
INSERT INTO post_tags (post_id, tag_id)
VALUES ($1, $2)
ON CONFLICT DO NOTHING;

-- name: RemoveTagFromPost :exec
DELETE FROM post_tags
WHERE post_id = $1 AND tag_id = $2;

-- name: GetTagsForPost :many
SELECT t.*
FROM tags t
INNER JOIN post_tags pt ON pt.tag_id = t.id
WHERE pt.post_id = $1
ORDER BY t.name;

-- name: GetPopularTags :many
SELECT t.*, COUNT(pt.post_id) as post_count
FROM tags t
INNER JOIN post_tags pt ON pt.tag_id = t.id
INNER JOIN posts p ON p.id = pt.post_id AND p.status = 'published'
GROUP BY t.id
ORDER BY post_count DESC
LIMIT $1;
```

```sql
-- query/orders.sql

-- name: CreateOrder :one
INSERT INTO orders (user_id, total_amount, status)
VALUES ($1, $2, $3)
RETURNING *;

-- name: CreateOrderItem :one
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES ($1, $2, $3, $4)
RETURNING *;

-- name: GetOrderWithItems :many
SELECT
    o.id AS order_id,
    o.user_id,
    o.total_amount,
    o.status AS order_status,
    o.created_at AS order_date,
    oi.id AS item_id,
    oi.product_id,
    oi.quantity,
    oi.price
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
WHERE o.id = $1
ORDER BY oi.id;

-- name: GetUserOrders :many
SELECT * FROM orders
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;
```

### Batch Operations

```sql
-- query/batch.sql

-- name: CreateUsers :copyfrom
INSERT INTO users (name, email, age, role)
VALUES ($1, $2, $3, $4);

-- name: BatchUpdateUserRoles :exec
UPDATE users
SET role = $2, updated_at = NOW()
WHERE id = ANY($1::bigint[]);

-- name: BatchDeleteUsers :exec
DELETE FROM users WHERE id = ANY($1::bigint[]);

-- name: UpsertUser :one
INSERT INTO users (name, email, age, role)
VALUES ($1, $2, $3, $4)
ON CONFLICT (email)
DO UPDATE SET
    name = EXCLUDED.name,
    age = EXCLUDED.age,
    role = EXCLUDED.role,
    updated_at = NOW()
RETURNING *;
```

### Generate and Use

```bash
# Generate Go code
sqlc generate
```

This generates type-safe Go code in `internal/db/`:

```go
// internal/db/models.go (auto-generated)
package db

import (
    "time"
    "encoding/json"
)

type User struct {
    ID        int64            `json:"id"`
    Name      string           `json:"name"`
    Email     string           `json:"email"`
    Age       *int32           `json:"age"`
    Role      string           `json:"role"`
    IsActive  bool             `json:"is_active"`
    Metadata  json.RawMessage  `json:"metadata"`
    CreatedAt time.Time        `json:"created_at"`
    UpdatedAt time.Time        `json:"updated_at"`
}

type Post struct {
    ID          int64      `json:"id"`
    AuthorID    int64      `json:"author_id"`
    Title       string     `json:"title"`
    Body        string     `json:"body"`
    Status      string     `json:"status"`
    ViewCount   int32      `json:"view_count"`
    PublishedAt *time.Time `json:"published_at"`
    CreatedAt   time.Time  `json:"created_at"`
    UpdatedAt   time.Time  `json:"updated_at"`
}
// ... etc
```

```go
// Using the generated code
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    "yourproject/internal/db"

    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    ctx := context.Background()

    pool, err := pgxpool.New(ctx, "postgres://user:pass@localhost:5432/mydb")
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()

    queries := db.New(pool)

    // Create a user - type-safe parameters
    user, err := queries.CreateUser(ctx, db.CreateUserParams{
        Name: "Alice",
        Email: "alice@example.com",
        Age:  int32Ptr(30),
        Role: "admin",
        Metadata: json.RawMessage(`{"theme": "dark"}`),
    })
    if err != nil {
        log.Fatalf("create user: %v", err)
    }
    fmt.Printf("Created: %s (ID: %d)\n", user.Name, user.ID)

    // Read - returns strongly typed result
    found, err := queries.GetUser(ctx, user.ID)
    if err != nil {
        log.Fatalf("get user: %v", err)
    }
    fmt.Printf("Found: %s <%s>\n", found.Name, found.Email)

    // List with pagination
    users, err := queries.ListUsers(ctx, db.ListUsersParams{
        Limit:  20,
        Offset: 0,
    })
    if err != nil {
        log.Fatalf("list users: %v", err)
    }
    for _, u := range users {
        fmt.Printf("  %s (%s)\n", u.Name, u.Role)
    }

    // Join query - returns custom struct
    postWithAuthor, err := queries.GetPostWithAuthor(ctx, 1)
    if err != nil {
        log.Fatalf("get post with author: %v", err)
    }
    fmt.Printf("Post: %s by %s\n", postWithAuthor.Title, postWithAuthor.AuthorName)

    // Search
    results, err := queries.SearchUsers(ctx, db.SearchUsersParams{
        Column1: "alice",
        Limit:   10,
        Offset:  0,
    })
    if err != nil {
        log.Fatalf("search: %v", err)
    }
    fmt.Printf("Found %d users matching 'alice'\n", len(results))

    // Check existence
    exists, err := queries.UserExists(ctx, "alice@example.com")
    if err != nil {
        log.Fatalf("exists: %v", err)
    }
    fmt.Printf("User exists: %v\n", exists)
}

func int32Ptr(i int32) *int32 { return &i }
```

### Transactions with sqlc

```go
func createOrderTx(ctx context.Context, pool *pgxpool.Pool, userID int64, items []OrderItemInput) error {
    tx, err := pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback(ctx) // No-op if committed

    // Create a Queries instance bound to the transaction
    qtx := db.New(tx)

    // Calculate total
    var total float64
    for _, item := range items {
        total += item.Price * float64(item.Quantity)
    }

    // Create order
    order, err := qtx.CreateOrder(ctx, db.CreateOrderParams{
        UserID:      userID,
        TotalAmount: fmt.Sprintf("%.2f", total), // numeric as string
        Status:      "pending",
    })
    if err != nil {
        return fmt.Errorf("create order: %w", err)
    }

    // Create order items
    for _, item := range items {
        _, err := qtx.CreateOrderItem(ctx, db.CreateOrderItemParams{
            OrderID:   order.ID,
            ProductID: item.ProductID,
            Quantity:  item.Quantity,
            Price:     fmt.Sprintf("%.2f", item.Price),
        })
        if err != nil {
            return fmt.Errorf("create order item: %w", err)
        }
    }

    return tx.Commit(ctx)
}

type OrderItemInput struct {
    ProductID int64
    Quantity  int32
    Price     float64
}
```

---

## Comparison: GORM vs ent vs sqlc

### Feature Comparison Table

| Feature | GORM | ent | sqlc |
|---------|------|-----|------|
| **Approach** | Runtime ORM | Code-gen entity framework | Code-gen from SQL |
| **Type safety** | Runtime (reflection) | Compile-time | Compile-time |
| **Query language** | Go method chains | Go method chains (generated) | Raw SQL |
| **Schema definition** | Go structs + tags | Go schema DSL | SQL DDL files |
| **Code generation** | No | Yes (required) | Yes (required) |
| **Associations** | Declarative tags | Graph edges | Manual JOINs |
| **Migrations** | AutoMigrate (limited) | Atlas integration | External tool |
| **Hooks/middleware** | Yes (lifecycle callbacks) | Yes (mutation hooks) | No (use app layer) |
| **Transactions** | Yes | Yes | Yes (manual) |
| **Raw SQL** | Yes (escape hatch) | Yes (SQL modifiers) | Native |
| **Soft delete** | Built-in | Manual (via mixin) | Manual (in SQL) |
| **Plugins** | Rich ecosystem | Template-based extension | Limited |
| **Privacy/ACL** | No (manual) | Built-in privacy layer | No (manual) |
| **Testing** | Mock via interface | enttest package | Mock via interface |
| **Database support** | Postgres, MySQL, SQLite, SQL Server | Postgres, MySQL, SQLite, Gremlin | Postgres, MySQL, SQLite |
| **Maturity** | Very mature (since 2013) | Mature (since 2019) | Mature (since 2019) |

### Type Safety Comparison

```go
// GORM - runtime errors (compiles fine, fails at runtime)
var user User
db.Where("nonexistent_column = ?", "value").First(&user) // Runtime error
db.Where("name = ?", 123).First(&user) // Type mismatch, no compile error

// ent - compile-time errors
user, err := client.User.
    Query().
    Where(user.NonexistentColumn("value")). // COMPILE ERROR: undefined
    Only(ctx)

user, err := client.User.
    Query().
    Where(user.NameEQ(123)). // COMPILE ERROR: type mismatch
    Only(ctx)

// sqlc - compile-time errors (if SQL is valid, Go code is correct)
// If you reference a nonexistent column in SQL:
// -- name: Bad :one
// SELECT nonexistent FROM users WHERE id = $1;
// sqlc generate will FAIL with an error before you even compile Go code
```

### Performance Characteristics

```
Operation               GORM          ent           sqlc/raw
---------------------------------------------------------------
Single SELECT           ~1.2x raw     ~1.1x raw     ~1.0x raw
Bulk INSERT (1000)      ~1.5x raw     ~1.2x raw     ~1.0x raw
Complex JOIN            ~1.3x raw     ~1.2x raw     ~1.0x raw
N+1 (100 assoc)        10-100x raw*   1-2x raw**    1.0x raw
Memory (per query)      Higher***      Moderate      Lowest

*   GORM N+1 happens when you forget Preload
**  ent's eager loading generates efficient queries
*** GORM uses reflection which allocates more
```

> **Tip:** These are rough ballpark figures. Actual performance depends heavily on
> your specific queries, data volume, and how correctly you use each tool. Always
> benchmark with YOUR workload.

### Learning Curve

```
Difficulty      GORM              ent                   sqlc
------------------------------------------------------------------
Easy            Basic CRUD        Schema definition     Writing SQL
                Migrations        Basic queries         Configuration
                Simple queries    Code generation

Medium          Associations      Edges/relations       JOINs
                Preloading        Eager loading         Batch operations
                Scopes            Aggregation           Custom types
                Hooks             Hooks

Hard            Custom types      Privacy layer         Dynamic queries
                Performance       Custom templates      Complex aggregation
                Sharding          Atlas migrations      Transaction patterns
                Plugin writing    Graph traversal       Composing queries
```

---

## When to Use Which Tool

### Use GORM When

1. **Rapid prototyping** - You need to get something working fast
2. **Simple CRUD apps** - Standard business applications
3. **Team familiarity** - Team has ActiveRecord/Django ORM background
4. **Dynamic queries** - Need to build queries based on runtime conditions
5. **Quick migrations** - AutoMigrate is sufficient for your use case

```go
// GORM shines with dynamic query building
func SearchProducts(db *gorm.DB, filters ProductFilters) ([]Product, error) {
    query := db.Model(&Product{})

    if filters.Name != "" {
        query = query.Where("name ILIKE ?", "%"+filters.Name+"%")
    }
    if filters.MinPrice > 0 {
        query = query.Where("price >= ?", filters.MinPrice)
    }
    if filters.MaxPrice > 0 {
        query = query.Where("price <= ?", filters.MaxPrice)
    }
    if filters.CategoryID > 0 {
        query = query.Where("category_id = ?", filters.CategoryID)
    }
    if filters.InStock {
        query = query.Where("stock > 0")
    }

    var products []Product
    err := query.
        Order(filters.SortField + " " + filters.SortDir).
        Limit(filters.Limit).
        Offset(filters.Offset).
        Find(&products).Error
    return products, err
}
```

### Use ent When

1. **Graph-like data models** - Complex relationships and traversals
2. **Type safety is critical** - Financial, healthcare, regulated systems
3. **Access control** - Built-in privacy layer for multi-tenant apps
4. **GraphQL backends** - ent has excellent GraphQL integration
5. **Large teams** - Schema-as-code prevents drift and miscommunication

```go
// ent shines with graph traversal
// "Find all users who are members of groups that contain users who posted
// articles tagged with 'security'"
users, err := client.Tag.
    Query().
    Where(tag.NameEQ("security")).
    QueryPosts().
    QueryAuthor().
    QueryGroups().
    QueryMembers().
    All(ctx)
```

### Use sqlc When

1. **Performance-critical paths** - Zero abstraction overhead
2. **Complex SQL** - CTEs, window functions, lateral joins
3. **SQL expertise** - Team is comfortable writing SQL directly
4. **Existing schema** - Working with a legacy database
5. **Microservices** - Simple data access needs, minimal dependencies

```sql
-- sqlc shines with complex SQL that ORMs struggle with
-- name: GetUserActivityReport :many
WITH user_posts AS (
    SELECT
        author_id,
        COUNT(*) as post_count,
        AVG(view_count) as avg_views
    FROM posts
    WHERE published_at >= $1
    GROUP BY author_id
),
user_comments AS (
    SELECT
        author_id,
        COUNT(*) as comment_count
    FROM comments
    WHERE created_at >= $1
    GROUP BY author_id
)
SELECT
    u.id,
    u.name,
    u.email,
    COALESCE(up.post_count, 0) as post_count,
    COALESCE(up.avg_views, 0) as avg_views,
    COALESCE(uc.comment_count, 0) as comment_count,
    COALESCE(up.post_count, 0) + COALESCE(uc.comment_count, 0) as activity_score
FROM users u
LEFT JOIN user_posts up ON up.author_id = u.id
LEFT JOIN user_comments uc ON uc.author_id = u.id
WHERE u.is_active = true
ORDER BY activity_score DESC
LIMIT $2;
```

### Decision Flowchart

```
Start
  |
  v
Do you have complex relationships / graph traversals?
  |                    |
  YES                  NO
  |                    |
  v                    v
Need built-in      Is raw SQL performance critical?
privacy/ACL?          |              |
  |       |          YES             NO
  YES     NO          |              |
  |       |           v              v
  v       v       Is your team     How dynamic are
  ent     ent     strong in SQL?   your queries?
                    |        |       |           |
                   YES       NO    Very         Not very
                    |        |       |           |
                    v        v       v           v
                  sqlc    GORM     GORM       sqlc or ent
```

---

## Repository Pattern with Each Approach

### Interface Definition (Shared)

```go
// domain/repository.go
package domain

import (
    "context"
    "time"
)

type User struct {
    ID        int64
    Name      string
    Email     string
    Age       *int
    Role      string
    IsActive  bool
    CreatedAt time.Time
    UpdatedAt time.Time
}

type CreateUserInput struct {
    Name  string
    Email string
    Age   *int
    Role  string
}

type UpdateUserInput struct {
    Name  *string
    Email *string
    Age   *int
    Role  *string
}

type ListUsersFilter struct {
    Role     *string
    IsActive *bool
    Search   *string
    Limit    int
    Offset   int
}

type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    List(ctx context.Context, filter ListUsersFilter) ([]*User, error)
    Create(ctx context.Context, input CreateUserInput) (*User, error)
    Update(ctx context.Context, id int64, input UpdateUserInput) (*User, error)
    Delete(ctx context.Context, id int64) error
    Count(ctx context.Context, filter ListUsersFilter) (int64, error)
}
```

### GORM Repository Implementation

```go
// repository/gorm/user.go
package gormrepo

import (
    "context"
    "errors"
    "fmt"

    "yourproject/domain"

    "gorm.io/gorm"
)

type UserModel struct {
    gorm.Model
    Name     string `gorm:"type:varchar(100);not null"`
    Email    string `gorm:"type:varchar(255);uniqueIndex;not null"`
    Age      *int   `gorm:"check:age >= 0"`
    Role     string `gorm:"type:varchar(20);default:'user'"`
    IsActive bool   `gorm:"default:true"`
}

func (UserModel) TableName() string { return "users" }

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).First(&model, id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("user %d not found", id)
        }
        return nil, fmt.Errorf("get user by id: %w", err)
    }
    return toDomainUser(&model), nil
}

func (r *userRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).Where("email = ?", email).First(&model).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("user with email %s not found", email)
        }
        return nil, fmt.Errorf("get user by email: %w", err)
    }
    return toDomainUser(&model), nil
}

func (r *userRepository) List(ctx context.Context, filter domain.ListUsersFilter) ([]*domain.User, error) {
    query := r.db.WithContext(ctx).Model(&UserModel{})

    if filter.Role != nil {
        query = query.Where("role = ?", *filter.Role)
    }
    if filter.IsActive != nil {
        query = query.Where("is_active = ?", *filter.IsActive)
    }
    if filter.Search != nil && *filter.Search != "" {
        search := "%" + *filter.Search + "%"
        query = query.Where("name ILIKE ? OR email ILIKE ?", search, search)
    }

    if filter.Limit > 0 {
        query = query.Limit(filter.Limit)
    }
    if filter.Offset > 0 {
        query = query.Offset(filter.Offset)
    }

    var models []UserModel
    if err := query.Order("created_at DESC").Find(&models).Error; err != nil {
        return nil, fmt.Errorf("list users: %w", err)
    }

    users := make([]*domain.User, len(models))
    for i := range models {
        users[i] = toDomainUser(&models[i])
    }
    return users, nil
}

func (r *userRepository) Create(ctx context.Context, input domain.CreateUserInput) (*domain.User, error) {
    model := UserModel{
        Name:     input.Name,
        Email:    input.Email,
        Age:      input.Age,
        Role:     input.Role,
        IsActive: true,
    }
    if err := r.db.WithContext(ctx).Create(&model).Error; err != nil {
        return nil, fmt.Errorf("create user: %w", err)
    }
    return toDomainUser(&model), nil
}

func (r *userRepository) Update(ctx context.Context, id int64, input domain.UpdateUserInput) (*domain.User, error) {
    updates := map[string]interface{}{}
    if input.Name != nil {
        updates["name"] = *input.Name
    }
    if input.Email != nil {
        updates["email"] = *input.Email
    }
    if input.Age != nil {
        updates["age"] = *input.Age
    }
    if input.Role != nil {
        updates["role"] = *input.Role
    }

    if len(updates) == 0 {
        return r.GetByID(ctx, id)
    }

    result := r.db.WithContext(ctx).Model(&UserModel{}).Where("id = ?", id).Updates(updates)
    if result.Error != nil {
        return nil, fmt.Errorf("update user: %w", result.Error)
    }
    if result.RowsAffected == 0 {
        return nil, fmt.Errorf("user %d not found", id)
    }

    return r.GetByID(ctx, id)
}

func (r *userRepository) Delete(ctx context.Context, id int64) error {
    result := r.db.WithContext(ctx).Delete(&UserModel{}, id)
    if result.Error != nil {
        return fmt.Errorf("delete user: %w", result.Error)
    }
    if result.RowsAffected == 0 {
        return fmt.Errorf("user %d not found", id)
    }
    return nil
}

func (r *userRepository) Count(ctx context.Context, filter domain.ListUsersFilter) (int64, error) {
    query := r.db.WithContext(ctx).Model(&UserModel{})

    if filter.Role != nil {
        query = query.Where("role = ?", *filter.Role)
    }
    if filter.IsActive != nil {
        query = query.Where("is_active = ?", *filter.IsActive)
    }

    var count int64
    if err := query.Count(&count).Error; err != nil {
        return 0, fmt.Errorf("count users: %w", err)
    }
    return count, nil
}

// Mapper: GORM model -> domain entity
func toDomainUser(m *UserModel) *domain.User {
    return &domain.User{
        ID:        int64(m.ID),
        Name:      m.Name,
        Email:     m.Email,
        Age:       m.Age,
        Role:      m.Role,
        IsActive:  m.IsActive,
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }
}
```

### ent Repository Implementation

```go
// repository/ent/user.go
package entrepo

import (
    "context"
    "fmt"

    "yourproject/domain"
    "yourproject/ent"
    "yourproject/ent/user"
)

type userRepository struct {
    client *ent.Client
}

func NewUserRepository(client *ent.Client) domain.UserRepository {
    return &userRepository{client: client}
}

func (r *userRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    u, err := r.client.User.Get(ctx, int(id))
    if err != nil {
        if ent.IsNotFound(err) {
            return nil, fmt.Errorf("user %d not found", id)
        }
        return nil, fmt.Errorf("get user by id: %w", err)
    }
    return toDomainUser(u), nil
}

func (r *userRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    u, err := r.client.User.
        Query().
        Where(user.EmailEQ(email)).
        Only(ctx)
    if err != nil {
        if ent.IsNotFound(err) {
            return nil, fmt.Errorf("user with email %s not found", email)
        }
        return nil, fmt.Errorf("get user by email: %w", err)
    }
    return toDomainUser(u), nil
}

func (r *userRepository) List(ctx context.Context, filter domain.ListUsersFilter) ([]*domain.User, error) {
    query := r.client.User.Query()

    if filter.Role != nil {
        query = query.Where(user.RoleEQ(user.Role(*filter.Role)))
    }
    if filter.IsActive != nil {
        if *filter.IsActive {
            query = query.Where(user.StatusEQ(user.StatusActive))
        } else {
            query = query.Where(user.StatusNEQ(user.StatusActive))
        }
    }
    if filter.Search != nil && *filter.Search != "" {
        query = query.Where(
            user.Or(
                user.NameContainsFold(*filter.Search),
                user.EmailContainsFold(*filter.Search),
            ),
        )
    }

    if filter.Limit > 0 {
        query = query.Limit(filter.Limit)
    }
    if filter.Offset > 0 {
        query = query.Offset(filter.Offset)
    }

    entities, err := query.
        Order(ent.Desc(user.FieldCreatedAt)).
        All(ctx)
    if err != nil {
        return nil, fmt.Errorf("list users: %w", err)
    }

    users := make([]*domain.User, len(entities))
    for i, e := range entities {
        users[i] = toDomainUser(e)
    }
    return users, nil
}

func (r *userRepository) Create(ctx context.Context, input domain.CreateUserInput) (*domain.User, error) {
    create := r.client.User.
        Create().
        SetName(input.Name).
        SetEmail(input.Email).
        SetRole(user.Role(input.Role))

    if input.Age != nil {
        create = create.SetAge(*input.Age)
    }

    u, err := create.Save(ctx)
    if err != nil {
        return nil, fmt.Errorf("create user: %w", err)
    }
    return toDomainUser(u), nil
}

func (r *userRepository) Update(ctx context.Context, id int64, input domain.UpdateUserInput) (*domain.User, error) {
    update := r.client.User.UpdateOneID(int(id))

    if input.Name != nil {
        update = update.SetName(*input.Name)
    }
    if input.Email != nil {
        update = update.SetEmail(*input.Email)
    }
    if input.Age != nil {
        update = update.SetAge(*input.Age)
    }
    if input.Role != nil {
        update = update.SetRole(user.Role(*input.Role))
    }

    u, err := update.Save(ctx)
    if err != nil {
        if ent.IsNotFound(err) {
            return nil, fmt.Errorf("user %d not found", id)
        }
        return nil, fmt.Errorf("update user: %w", err)
    }
    return toDomainUser(u), nil
}

func (r *userRepository) Delete(ctx context.Context, id int64) error {
    err := r.client.User.DeleteOneID(int(id)).Exec(ctx)
    if err != nil {
        if ent.IsNotFound(err) {
            return fmt.Errorf("user %d not found", id)
        }
        return fmt.Errorf("delete user: %w", err)
    }
    return nil
}

func (r *userRepository) Count(ctx context.Context, filter domain.ListUsersFilter) (int64, error) {
    query := r.client.User.Query()

    if filter.Role != nil {
        query = query.Where(user.RoleEQ(user.Role(*filter.Role)))
    }

    count, err := query.Count(ctx)
    if err != nil {
        return 0, fmt.Errorf("count users: %w", err)
    }
    return int64(count), nil
}

func toDomainUser(e *ent.User) *domain.User {
    return &domain.User{
        ID:        int64(e.ID),
        Name:      e.Name,
        Email:     e.Email,
        Age:       e.Age,
        Role:      string(e.Role),
        IsActive:  e.Status == user.StatusActive,
        CreatedAt: e.CreatedAt,
        UpdatedAt: e.UpdatedAt,
    }
}
```

### sqlc Repository Implementation

```go
// repository/sqlc/user.go
package sqlcrepo

import (
    "context"
    "fmt"

    "yourproject/domain"
    "yourproject/internal/db"

    "github.com/jackc/pgx/v5/pgxpool"
)

type userRepository struct {
    pool    *pgxpool.Pool
    queries *db.Queries
}

func NewUserRepository(pool *pgxpool.Pool) domain.UserRepository {
    return &userRepository{
        pool:    pool,
        queries: db.New(pool),
    }
}

func (r *userRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    row, err := r.queries.GetUser(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user by id: %w", err)
    }
    return toDomainUser(row), nil
}

func (r *userRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    row, err := r.queries.GetUserByEmail(ctx, email)
    if err != nil {
        return nil, fmt.Errorf("get user by email: %w", err)
    }
    return toDomainUser(row), nil
}

func (r *userRepository) List(ctx context.Context, filter domain.ListUsersFilter) ([]*domain.User, error) {
    limit := int32(20)
    offset := int32(0)
    if filter.Limit > 0 {
        limit = int32(filter.Limit)
    }
    if filter.Offset > 0 {
        offset = int32(filter.Offset)
    }

    // sqlc generates different functions for different queries,
    // so we select the right one based on the filter.
    if filter.Search != nil && *filter.Search != "" {
        rows, err := r.queries.SearchUsers(ctx, db.SearchUsersParams{
            Column1: *filter.Search,
            Limit:   limit,
            Offset:  offset,
        })
        if err != nil {
            return nil, fmt.Errorf("search users: %w", err)
        }
        return toDomainUsers(rows), nil
    }

    if filter.Role != nil {
        rows, err := r.queries.ListUsersByRole(ctx, *filter.Role)
        if err != nil {
            return nil, fmt.Errorf("list users by role: %w", err)
        }
        return toDomainUsers(rows), nil
    }

    rows, err := r.queries.ListUsers(ctx, db.ListUsersParams{
        Limit:  limit,
        Offset: offset,
    })
    if err != nil {
        return nil, fmt.Errorf("list users: %w", err)
    }
    return toDomainUsers(rows), nil
}

func (r *userRepository) Create(ctx context.Context, input domain.CreateUserInput) (*domain.User, error) {
    var age *int32
    if input.Age != nil {
        a := int32(*input.Age)
        age = &a
    }

    row, err := r.queries.CreateUser(ctx, db.CreateUserParams{
        Name:     input.Name,
        Email:    input.Email,
        Age:      age,
        Role:     input.Role,
        Metadata: nil,
    })
    if err != nil {
        return nil, fmt.Errorf("create user: %w", err)
    }
    return toDomainUser(row), nil
}

func (r *userRepository) Update(ctx context.Context, id int64, input domain.UpdateUserInput) (*domain.User, error) {
    // With sqlc, partial updates require either:
    // 1. Reading the current record first and merging
    // 2. Having separate query per field combination
    // 3. Using COALESCE patterns in SQL

    current, err := r.queries.GetUser(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user for update: %w", err)
    }

    name := current.Name
    email := current.Email
    age := current.Age
    role := current.Role

    if input.Name != nil {
        name = *input.Name
    }
    if input.Email != nil {
        email = *input.Email
    }
    if input.Age != nil {
        a := int32(*input.Age)
        age = &a
    }
    if input.Role != nil {
        role = *input.Role
    }

    row, err := r.queries.UpdateUser(ctx, db.UpdateUserParams{
        ID:    id,
        Name:  name,
        Email: email,
        Age:   age,
        Role:  role,
    })
    if err != nil {
        return nil, fmt.Errorf("update user: %w", err)
    }
    return toDomainUser(row), nil
}

func (r *userRepository) Delete(ctx context.Context, id int64) error {
    return r.queries.DeleteUser(ctx, id)
}

func (r *userRepository) Count(ctx context.Context, filter domain.ListUsersFilter) (int64, error) {
    if filter.Role != nil {
        result, err := r.queries.CountUsersByRole(ctx, *filter.Role)
        if err != nil {
            return 0, fmt.Errorf("count users: %w", err)
        }
        return result.Count, nil
    }
    // Would need a separate query for count-all
    return 0, fmt.Errorf("count without role filter not implemented")
}

func toDomainUser(row db.User) *domain.User {
    var age *int
    if row.Age != nil {
        a := int(*row.Age)
        age = &a
    }
    return &domain.User{
        ID:        row.ID,
        Name:      row.Name,
        Email:     row.Email,
        Age:       age,
        Role:      row.Role,
        IsActive:  row.IsActive,
        CreatedAt: row.CreatedAt,
        UpdatedAt: row.UpdatedAt,
    }
}

func toDomainUsers[T db.User](rows []T) []*domain.User {
    users := make([]*domain.User, len(rows))
    for i, row := range rows {
        users[i] = toDomainUser(db.User(row))
    }
    return users
}
```

### Using the Repository in Service Layer

```go
// service/user_service.go
package service

import (
    "context"
    "fmt"

    "yourproject/domain"
)

type UserService struct {
    repo domain.UserRepository
}

func NewUserService(repo domain.UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Register(ctx context.Context, name, email string) (*domain.User, error) {
    // Check if email already exists
    existing, err := s.repo.GetByEmail(ctx, email)
    if err == nil && existing != nil {
        return nil, fmt.Errorf("email already registered: %s", email)
    }

    // Create user
    user, err := s.repo.Create(ctx, domain.CreateUserInput{
        Name:  name,
        Email: email,
        Role:  "user",
    })
    if err != nil {
        return nil, fmt.Errorf("register user: %w", err)
    }

    return user, nil
}

// Dependency injection - swap implementations freely
func main() {
    // Option A: GORM
    // repo := gormrepo.NewUserRepository(gormDB)

    // Option B: ent
    // repo := entrepo.NewUserRepository(entClient)

    // Option C: sqlc
    // repo := sqlcrepo.NewUserRepository(pgxPool)

    // The service doesn't care which implementation is used
    // svc := service.NewUserService(repo)
}
```

---

## Testing Strategies

### Testing GORM with SQLite

```go
// repository/gorm/user_test.go
package gormrepo_test

import (
    "context"
    "testing"

    gormrepo "yourproject/repository/gorm"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func setupTestDB(t *testing.T) *gorm.DB {
    t.Helper()

    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    })
    if err != nil {
        t.Fatalf("failed to open test db: %v", err)
    }

    // Auto-migrate
    err = db.AutoMigrate(&gormrepo.UserModel{})
    if err != nil {
        t.Fatalf("failed to migrate: %v", err)
    }

    return db
}

func TestUserRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    repo := gormrepo.NewUserRepository(db)
    ctx := context.Background()

    user, err := repo.Create(ctx, domain.CreateUserInput{
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if user.Name != "Alice" {
        t.Errorf("expected name Alice, got %s", user.Name)
    }
    if user.ID == 0 {
        t.Error("expected non-zero ID")
    }
}

func TestUserRepository_GetByID(t *testing.T) {
    db := setupTestDB(t)
    repo := gormrepo.NewUserRepository(db)
    ctx := context.Background()

    // Seed
    created, _ := repo.Create(ctx, domain.CreateUserInput{
        Name:  "Bob",
        Email: "bob@example.com",
        Role:  "admin",
    })

    // Test
    found, err := repo.GetByID(ctx, created.ID)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if found.Email != "bob@example.com" {
        t.Errorf("expected email bob@example.com, got %s", found.Email)
    }
}

func TestUserRepository_GetByID_NotFound(t *testing.T) {
    db := setupTestDB(t)
    repo := gormrepo.NewUserRepository(db)
    ctx := context.Background()

    _, err := repo.GetByID(ctx, 99999)
    if err == nil {
        t.Error("expected error for non-existent user")
    }
}

func TestUserRepository_List_WithFilters(t *testing.T) {
    db := setupTestDB(t)
    repo := gormrepo.NewUserRepository(db)
    ctx := context.Background()

    // Seed
    repo.Create(ctx, domain.CreateUserInput{Name: "Alice", Email: "alice@ex.com", Role: "admin"})
    repo.Create(ctx, domain.CreateUserInput{Name: "Bob", Email: "bob@ex.com", Role: "user"})
    repo.Create(ctx, domain.CreateUserInput{Name: "Charlie", Email: "charlie@ex.com", Role: "user"})

    // Filter by role
    role := "user"
    users, err := repo.List(ctx, domain.ListUsersFilter{Role: &role})
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if len(users) != 2 {
        t.Errorf("expected 2 users, got %d", len(users))
    }

    // Filter by search
    search := "alice"
    users, err = repo.List(ctx, domain.ListUsersFilter{Search: &search})
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if len(users) != 1 {
        t.Errorf("expected 1 user, got %d", len(users))
    }
}

func TestUserRepository_Update(t *testing.T) {
    db := setupTestDB(t)
    repo := gormrepo.NewUserRepository(db)
    ctx := context.Background()

    created, _ := repo.Create(ctx, domain.CreateUserInput{
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    })

    newName := "Alice Smith"
    updated, err := repo.Update(ctx, created.ID, domain.UpdateUserInput{
        Name: &newName,
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if updated.Name != "Alice Smith" {
        t.Errorf("expected name 'Alice Smith', got %s", updated.Name)
    }
    // Email should remain unchanged
    if updated.Email != "alice@example.com" {
        t.Errorf("email should not change, got %s", updated.Email)
    }
}

func TestUserRepository_Delete(t *testing.T) {
    db := setupTestDB(t)
    repo := gormrepo.NewUserRepository(db)
    ctx := context.Background()

    created, _ := repo.Create(ctx, domain.CreateUserInput{
        Name:  "ToDelete",
        Email: "delete@example.com",
        Role:  "user",
    })

    err := repo.Delete(ctx, created.ID)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    _, err = repo.GetByID(ctx, created.ID)
    if err == nil {
        t.Error("expected error after deletion")
    }
}
```

### Testing ent with enttest

```go
// repository/ent/user_test.go
package entrepo_test

import (
    "context"
    "testing"

    "yourproject/ent/enttest"
    entrepo "yourproject/repository/ent"

    _ "github.com/mattn/go-sqlite3"
)

func TestEntUserRepository_Create(t *testing.T) {
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1")
    defer client.Close()

    repo := entrepo.NewUserRepository(client)
    ctx := context.Background()

    user, err := repo.Create(ctx, domain.CreateUserInput{
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    })
    if err != nil {
        t.Fatalf("create: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}

func TestEntUserRepository_Edges(t *testing.T) {
    client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&_fk=1")
    defer client.Close()
    ctx := context.Background()

    // Create user with posts
    u, _ := client.User.Create().
        SetName("Alice").
        SetEmail("alice@test.com").
        SetPasswordHash("hash").
        Save(ctx)

    _, _ = client.Post.Create().
        SetTitle("Post 1").
        SetBody("Body 1").
        SetAuthor(u).
        Save(ctx)

    _, _ = client.Post.Create().
        SetTitle("Post 2").
        SetBody("Body 2").
        SetAuthor(u).
        Save(ctx)

    // Verify edge traversal
    posts, err := u.QueryPosts().All(ctx)
    if err != nil {
        t.Fatalf("query posts: %v", err)
    }
    if len(posts) != 2 {
        t.Errorf("expected 2 posts, got %d", len(posts))
    }
}
```

### Testing sqlc with pgx and testcontainers

```go
// repository/sqlc/user_test.go
package sqlcrepo_test

import (
    "context"
    "os"
    "testing"

    "yourproject/internal/db"
    sqlcrepo "yourproject/repository/sqlc"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

var testPool *pgxpool.Pool

func TestMain(m *testing.M) {
    ctx := context.Background()

    // Start PostgreSQL container
    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2),
        ),
    )
    if err != nil {
        panic(err)
    }
    defer pgContainer.Terminate(ctx)

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        panic(err)
    }

    testPool, err = pgxpool.New(ctx, connStr)
    if err != nil {
        panic(err)
    }
    defer testPool.Close()

    // Run schema migrations
    schema, _ := os.ReadFile("../../schema/001_users.sql")
    if _, err := testPool.Exec(ctx, string(schema)); err != nil {
        panic(err)
    }

    os.Exit(m.Run())
}

func TestSqlcUserRepository_Create(t *testing.T) {
    repo := sqlcrepo.NewUserRepository(testPool)
    ctx := context.Background()

    user, err := repo.Create(ctx, domain.CreateUserInput{
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    })
    if err != nil {
        t.Fatalf("create: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
    if user.ID == 0 {
        t.Error("expected non-zero ID")
    }

    // Clean up
    repo.Delete(ctx, user.ID)
}

func TestSqlcUserRepository_List(t *testing.T) {
    repo := sqlcrepo.NewUserRepository(testPool)
    ctx := context.Background()

    // Seed
    u1, _ := repo.Create(ctx, domain.CreateUserInput{Name: "A", Email: "a@test.com", Role: "admin"})
    u2, _ := repo.Create(ctx, domain.CreateUserInput{Name: "B", Email: "b@test.com", Role: "user"})
    u3, _ := repo.Create(ctx, domain.CreateUserInput{Name: "C", Email: "c@test.com", Role: "user"})

    defer func() {
        repo.Delete(ctx, u1.ID)
        repo.Delete(ctx, u2.ID)
        repo.Delete(ctx, u3.ID)
    }()

    users, err := repo.List(ctx, domain.ListUsersFilter{
        Limit:  10,
        Offset: 0,
    })
    if err != nil {
        t.Fatalf("list: %v", err)
    }
    if len(users) < 3 {
        t.Errorf("expected at least 3 users, got %d", len(users))
    }
}
```

### Mock-Based Testing (Works with All Approaches)

```go
// mock/user_repository.go
package mock

import (
    "context"
    "sync"

    "yourproject/domain"
)

type UserRepository struct {
    mu    sync.RWMutex
    users map[int64]*domain.User
    seq   int64
}

func NewUserRepository() *UserRepository {
    return &UserRepository{
        users: make(map[int64]*domain.User),
    }
}

func (m *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    u, ok := m.users[id]
    if !ok {
        return nil, fmt.Errorf("user %d not found", id)
    }
    return u, nil
}

func (m *UserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    for _, u := range m.users {
        if u.Email == email {
            return u, nil
        }
    }
    return nil, fmt.Errorf("user with email %s not found", email)
}

func (m *UserRepository) Create(ctx context.Context, input domain.CreateUserInput) (*domain.User, error) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.seq++
    u := &domain.User{
        ID:        m.seq,
        Name:      input.Name,
        Email:     input.Email,
        Age:       input.Age,
        Role:      input.Role,
        IsActive:  true,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    m.users[u.ID] = u
    return u, nil
}

func (m *UserRepository) List(ctx context.Context, filter domain.ListUsersFilter) ([]*domain.User, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    var result []*domain.User
    for _, u := range m.users {
        if filter.Role != nil && u.Role != *filter.Role {
            continue
        }
        result = append(result, u)
    }
    return result, nil
}

func (m *UserRepository) Update(ctx context.Context, id int64, input domain.UpdateUserInput) (*domain.User, error) {
    m.mu.Lock()
    defer m.mu.Unlock()
    u, ok := m.users[id]
    if !ok {
        return nil, fmt.Errorf("user %d not found", id)
    }
    if input.Name != nil {
        u.Name = *input.Name
    }
    if input.Email != nil {
        u.Email = *input.Email
    }
    u.UpdatedAt = time.Now()
    return u, nil
}

func (m *UserRepository) Delete(ctx context.Context, id int64) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.users, id)
    return nil
}

func (m *UserRepository) Count(ctx context.Context, filter domain.ListUsersFilter) (int64, error) {
    users, err := m.List(ctx, filter)
    return int64(len(users)), err
}

// Usage in service tests
func TestUserService_Register(t *testing.T) {
    repo := mock.NewUserRepository()
    svc := service.NewUserService(repo)
    ctx := context.Background()

    user, err := svc.Register(ctx, "Alice", "alice@example.com")
    if err != nil {
        t.Fatalf("register: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }

    // Duplicate registration should fail
    _, err = svc.Register(ctx, "Alice 2", "alice@example.com")
    if err == nil {
        t.Error("expected duplicate email error")
    }
}
```

### Testing Strategy Summary

| Strategy | Speed | Fidelity | Isolation | Best For |
|----------|-------|----------|-----------|----------|
| **In-memory mock** | Fastest | Low | Perfect | Service/business logic tests |
| **SQLite in-memory** | Fast | Medium | Good | GORM/ent unit tests |
| **Testcontainers** | Slow | High | Good | Integration tests, sqlc |
| **Shared test DB** | Fast (after setup) | High | Poor | Legacy codebases |
| **Transaction rollback** | Fast | High | Good | Tests that need real DB |

> **Tip:** Use a layered approach:
> 1. Mock repos for service-layer unit tests (fast, run always)
> 2. SQLite for repository-layer tests (fast, no Docker needed)
> 3. Testcontainers for integration tests (slower, run in CI)

---

## Performance Benchmarks and Common Pitfalls

### Benchmark Setup

```go
// bench_test.go
package bench_test

import (
    "context"
    "fmt"
    "testing"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

var benchDB *gorm.DB

func init() {
    var err error
    benchDB, err = gorm.Open(postgres.Open("postgres://..."), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Silent),
    })
    if err != nil {
        panic(err)
    }
}

func BenchmarkGORM_SingleInsert(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        user := User{
            Name:  fmt.Sprintf("user_%d", i),
            Email: fmt.Sprintf("user_%d@bench.com", i),
        }
        benchDB.Create(&user)
    }
}

func BenchmarkGORM_BatchInsert(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        users := make([]User, 100)
        for j := 0; j < 100; j++ {
            users[j] = User{
                Name:  fmt.Sprintf("user_%d_%d", i, j),
                Email: fmt.Sprintf("user_%d_%d@bench.com", i, j),
            }
        }
        benchDB.CreateInBatches(&users, 100)
    }
}

func BenchmarkGORM_SelectOne(b *testing.B) {
    // Seed first
    user := User{Name: "benchmark", Email: "bench@test.com"}
    benchDB.Create(&user)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var u User
        benchDB.First(&u, user.ID)
    }
}

func BenchmarkGORM_SelectWithPreload(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var users []User
        benchDB.Preload("Profile").Preload("Orders").Limit(100).Find(&users)
    }
}

func BenchmarkGORM_SelectWithJoins(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var users []User
        benchDB.Joins("Profile").Limit(100).Find(&users)
    }
}

// Equivalent raw SQL benchmark for comparison
func BenchmarkRawSQL_SelectOne(b *testing.B) {
    sqlDB, _ := benchDB.DB()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var u User
        sqlDB.QueryRow("SELECT id, name, email FROM users WHERE id = $1", 1).
            Scan(&u.ID, &u.Name, &u.Email)
    }
}
```

### Typical Benchmark Results

```
Benchmark                              ns/op       allocs/op
-----------------------------------------------------------
RawSQL_SelectOne                       45,200      3
Sqlc_SelectOne                         46,800      4
Ent_SelectOne                          52,100      8
GORM_SelectOne                         58,900      15
GORM_SelectOne_PreparedStmt            49,200      12

RawSQL_BulkInsert_1000                 8,200,000   2,010
Sqlc_BulkInsert_1000 (COPY)            3,100,000   1,005
Ent_BulkInsert_1000                    9,800,000   12,050
GORM_BulkInsert_1000                   12,500,000  18,200
GORM_BatchInsert_1000 (size=100)       9,900,000   14,100

RawSQL_Select100_WithJoin              380,000     310
GORM_Select100_WithPreload             1,200,000   1,850
GORM_Select100_WithJoins               420,000     580
Ent_Select100_WithEagerLoad            450,000     620
```

> **Key takeaway:** The raw performance difference between approaches is small for
> individual operations. The biggest performance gains come from avoiding N+1 queries
> and using batch operations correctly.

### Common Pitfalls

#### Pitfall 1: N+1 Queries (GORM)

```go
// BAD: N+1 problem - 1 query for users + N queries for profiles
var users []User
db.Find(&users)
for _, u := range users {
    var profile Profile
    db.Where("user_id = ?", u.ID).First(&profile) // N extra queries!
    fmt.Println(u.Name, profile.Phone)
}

// GOOD: Eager loading with Preload - 2 queries total
var users []User
db.Preload("Profile").Find(&users)
for _, u := range users {
    fmt.Println(u.Name, u.Profile.Phone)
}

// BEST: JOIN loading - 1 query total (for has-one/belongs-to)
var users []User
db.Joins("Profile").Find(&users)
```

#### Pitfall 2: SELECT * When You Only Need a Few Columns

```go
// BAD: Fetches all columns including large text/blob fields
var users []User
db.Find(&users)

// GOOD: Select only what you need
type UserListItem struct {
    ID    uint
    Name  string
    Email string
}
var items []UserListItem
db.Model(&User{}).Select("id", "name", "email").Find(&items)

// ent equivalent
users, _ := client.User.Query().
    Select(user.FieldID, user.FieldName, user.FieldEmail).
    All(ctx)
```

#### Pitfall 3: Missing Indexes

```go
// If you query by email frequently, make sure the index exists
type User struct {
    gorm.Model
    Email string `gorm:"uniqueIndex"` // Index defined in struct tag
}

// For composite indexes, use the Indexes method in ent:
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("role", "status", "created_at"),
    }
}

// For sqlc, define indexes in your schema SQL:
// CREATE INDEX idx_users_role_status ON users(role, status, created_at);
```

#### Pitfall 4: Unbounded Queries

```go
// BAD: Could return millions of rows
var users []User
db.Find(&users)

// GOOD: Always paginate
var users []User
db.Limit(100).Offset(0).Find(&users)

// BETTER: Use cursor-based pagination for large datasets
var users []User
db.Where("id > ?", lastSeenID).Order("id ASC").Limit(100).Find(&users)
```

#### Pitfall 5: Missing Context Propagation

```go
// BAD: No context - can't be cancelled, no timeout
db.Find(&users)

// GOOD: Always pass context
db.WithContext(ctx).Find(&users)

// Set query timeout via context
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
db.WithContext(ctx).Find(&users)
```

#### Pitfall 6: Transaction Scope Too Large

```go
// BAD: Holding a transaction while doing slow external calls
err := db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&order)
    sendEmail(order)            // Slow! Holds transaction open
    callPaymentAPI(order)       // Slow! Holds transaction open
    tx.Create(&payment)
    return nil
})

// GOOD: Keep transactions tight, do slow work outside
var order Order
err := db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&order)
    tx.Create(&payment)
    return nil
})
if err != nil {
    return err
}
// Slow work AFTER transaction commits
go sendEmail(order)
go callPaymentAPI(order)
```

#### Pitfall 7: GORM Zero-Value Gotcha

```go
// BAD: Trying to set IsActive to false using struct Updates
db.Model(&user).Updates(User{IsActive: false}) // false is zero value - IGNORED!

// GOOD: Use map for zero values
db.Model(&user).Updates(map[string]interface{}{"is_active": false})

// GOOD: Use Select to explicitly include fields
db.Model(&user).Select("is_active").Updates(User{IsActive: false})

// GOOD: Use Save (updates ALL fields)
user.IsActive = false
db.Save(&user)
```

#### Pitfall 8: Forgetting Unscoped for Soft-Deleted Records

```go
// This will NOT find soft-deleted users:
var user User
db.First(&user, deletedUserID) // Returns "record not found"

// You need Unscoped:
db.Unscoped().First(&user, deletedUserID) // Works

// Common trap in admin dashboards that need to show all records
var allUsers []User
db.Unscoped().Find(&allUsers) // Includes soft-deleted
```

#### Pitfall 9: sqlc Dynamic Queries Limitation

```go
// sqlc cannot generate dynamic WHERE clauses.
// You need separate queries or use CASE/COALESCE patterns.

-- Instead of dynamic filters, use COALESCE or CASE:
-- name: ListUsersFiltered :many
SELECT * FROM users
WHERE
    ($1::text IS NULL OR role = $1)
    AND ($2::bool IS NULL OR is_active = $2)
    AND ($3::text IS NULL OR name ILIKE '%' || $3 || '%')
ORDER BY created_at DESC
LIMIT $4 OFFSET $5;
```

#### Pitfall 10: Connection Pool Exhaustion

```go
// BAD: Default pool settings with high-concurrency workload
db, _ := gorm.Open(postgres.Open(dsn), &gorm.Config{})
// Default MaxOpenConns is 0 (unlimited) in GORM but underlying
// driver may have its own limits

// GOOD: Configure pool explicitly
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(25)         // Match your expected concurrency
sqlDB.SetMaxIdleConns(5)          // Keep some warm connections
sqlDB.SetConnMaxLifetime(time.Hour) // Rotate connections
sqlDB.SetConnMaxIdleTime(10 * time.Minute)

// For pgx pool:
config, _ := pgxpool.ParseConfig(connStr)
config.MaxConns = 25
config.MinConns = 5
config.MaxConnLifetime = time.Hour
config.MaxConnIdleTime = 10 * time.Minute
pool, _ := pgxpool.NewWithConfig(ctx, config)
```

---

## Migration Strategies Between Approaches

### GORM to sqlc

When your GORM application has performance-critical paths that need optimization,
or your team wants more control over SQL.

```
Step 1: Extract current schema
Step 2: Write SQL schema files matching current DB
Step 3: Write sqlc queries for hot paths
Step 4: Use both side-by-side via repository interface
Step 5: Gradually migrate remaining queries
```

```go
// Phase 1: Run both side by side

// Hybrid repository that uses sqlc for reads, GORM for writes
type HybridUserRepo struct {
    gormDB  *gorm.DB
    queries *db.Queries
}

func (r *HybridUserRepo) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    // Use sqlc for reads (faster, type-safe)
    row, err := r.queries.GetUser(ctx, id)
    if err != nil {
        return nil, err
    }
    return mapSqlcUser(row), nil
}

func (r *HybridUserRepo) Create(ctx context.Context, input domain.CreateUserInput) (*domain.User, error) {
    // Use GORM for writes (easier with associations, hooks)
    model := UserModel{
        Name:  input.Name,
        Email: input.Email,
    }
    if err := r.gormDB.WithContext(ctx).Create(&model).Error; err != nil {
        return nil, err
    }
    return mapGormUser(&model), nil
}
```

### GORM to ent

When you need better type safety and your data model has complex relationships.

```go
// Phase 1: Map GORM models to ent schemas
// GORM Model:
type User struct {
    gorm.Model
    Name  string `gorm:"type:varchar(100)"`
    Email string `gorm:"uniqueIndex"`
    Posts []Post
}

// Equivalent ent schema:
// ent/schema/user.go
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").MaxLen(100),
        field.String("email").Unique(),
        field.Time("created_at").Default(time.Now).Immutable(),
        field.Time("updated_at").Default(time.Now).UpdateDefault(time.Now),
    }
}
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("posts", Post.Type),
    }
}

// Phase 2: Use ent's Atlas migration to manage schema changes
// instead of GORM's AutoMigrate

// Phase 3: Swap repository implementations behind the interface
```

### sqlc to GORM

When you need more dynamic query building or faster development velocity.

```go
// This direction is less common, but may be needed when:
// - Dynamic query requirements increase
// - Team changes and SQL expertise decreases
// - Association handling becomes too complex in raw SQL

// Key migration steps:
// 1. Create GORM models matching your existing SQL schema
// 2. DON'T use AutoMigrate (schema already exists)
// 3. Implement the same repository interface
// 4. Switch implementations in your DI container
// 5. Run both in parallel with shadow-read testing

type GormUserRepo struct {
    db *gorm.DB
}

// Prevent AutoMigrate from running on existing tables
func NewGormUserRepo(db *gorm.DB) *GormUserRepo {
    // Do NOT call db.AutoMigrate here since schema is managed by sqlc/migrations
    return &GormUserRepo{db: db}
}
```

### Migration Checklist

```markdown
## Pre-Migration
- [ ] All repository access goes through interfaces (not direct DB calls)
- [ ] Comprehensive test suite exists for current implementation
- [ ] Schema is documented (or extractable from current tool)
- [ ] Performance benchmarks exist for critical paths

## During Migration
- [ ] New implementation satisfies the same interface
- [ ] Shadow-read testing (run both, compare results)
- [ ] No schema changes during migration
- [ ] Feature flags to toggle between implementations
- [ ] Monitor query latency and error rates

## Post-Migration
- [ ] Remove old implementation code
- [ ] Update CI/CD pipeline (code generation steps)
- [ ] Update documentation
- [ ] Train team on new tool
- [ ] Remove feature flags after stabilization period
```

### Shadow Read Pattern

```go
// shadow.go - Compare two implementations in production
type ShadowUserRepo struct {
    primary   domain.UserRepository // New implementation
    secondary domain.UserRepository // Old implementation (read-only)
    logger    *slog.Logger
}

func (s *ShadowUserRepo) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    // Always use primary for the response
    user, err := s.primary.GetByID(ctx, id)

    // Asynchronously compare with secondary
    go func() {
        shadowUser, shadowErr := s.secondary.GetByID(ctx, id)
        if (err == nil) != (shadowErr == nil) {
            s.logger.Warn("shadow read mismatch: error difference",
                "id", id,
                "primary_err", err,
                "shadow_err", shadowErr,
            )
        } else if err == nil && shadowErr == nil {
            if user.Name != shadowUser.Name || user.Email != shadowUser.Email {
                s.logger.Warn("shadow read mismatch: data difference",
                    "id", id,
                    "primary", user,
                    "shadow", shadowUser,
                )
            }
        }
    }()

    return user, err
}
```

---

## Summary

### Quick Reference

```
┌──────────────────────────────────────────────────────────────┐
│                    Choosing Your Tool                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  GORM                                                        │
│  ├── Best for: Rapid prototyping, simple CRUD, dynamic       │
│  │   queries, teams familiar with ActiveRecord patterns      │
│  ├── Install: go get gorm.io/gorm                            │
│  └── Key concept: Struct tags → table mapping                │
│                                                              │
│  ent                                                         │
│  ├── Best for: Complex relationships, graph traversals,      │
│  │   type safety, GraphQL backends, multi-tenant apps        │
│  ├── Install: go get entgo.io/ent                            │
│  └── Key concept: Schema-as-code → generated type-safe API   │
│                                                              │
│  sqlc                                                        │
│  ├── Best for: Performance-critical paths, complex SQL,      │
│  │   teams strong in SQL, microservices                      │
│  ├── Install: go install github.com/sqlc-dev/sqlc/cmd/sqlc   │
│  └── Key concept: Write SQL → generated type-safe Go         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **No tool is universally "best."** Each makes different tradeoffs. Pick the one
   that fits your team's skills, your project's requirements, and your performance
   constraints.

2. **Always use the repository pattern.** It decouples your business logic from your
   data access tool, making it possible to swap implementations, test with mocks,
   and migrate between tools incrementally.

3. **Type safety catches bugs early.** If your system handles money, PII, or
   regulated data, strongly prefer ent or sqlc over GORM.

4. **N+1 queries are the #1 performance killer** in ORM-based applications. Learn
   how to use `Preload`, `Joins`, and eager loading correctly.

5. **Test at multiple levels:** mock repositories for service tests, in-memory
   databases for repository tests, and real databases (via testcontainers) for
   integration tests.

6. **Migrations in production** should never use `AutoMigrate`. Use Atlas,
   golang-migrate, or another versioned migration tool.

7. **You can mix approaches.** Use GORM for admin dashboards where development speed
   matters, and sqlc for the hot path in your API. The repository interface makes
   this seamless.

8. **Context propagation and connection pool tuning** matter more for production
   performance than the choice of ORM vs raw SQL.

### Further Reading

- GORM documentation: https://gorm.io/docs/
- ent documentation: https://entgo.io/docs/getting-started
- sqlc documentation: https://docs.sqlc.dev/
- Atlas (migration tool): https://atlasgo.io/
- `database/sql` tutorial: https://go.dev/doc/database/
- Go database best practices: https://www.alexedwards.net/blog/organising-database-access
