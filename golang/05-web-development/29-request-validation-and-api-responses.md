# Chapter 29: Request Validation & API Responses

Every API has a trust boundary -- the line where untrusted external input enters your system. In a production API, this boundary is your HTTP handler layer. Every byte that arrives over the wire must be treated as hostile until proven otherwise. This chapter covers the full lifecycle of request validation and response formatting in a production Go API, drawn from real patterns used in production systems like "krafty-core". We will build a complete, reusable validation and response framework from scratch, compare every approach to its Node.js/Express equivalent, and end with a fully integrated handler that ties everything together.

Chapters 1-22 covered Go fundamentals. This chapter assumes you are comfortable with structs, interfaces, JSON, HTTP handlers, error handling, and context. If you have built APIs in Express with Joi, Zod, or express-validator, the comparisons throughout will be especially valuable.

## Prerequisites

- Comfortable with Go structs, interfaces, and methods (Chapters 4, 6)
- Understanding of pointers and memory (Chapter 7)
- Familiar with error handling patterns (Chapter 8)
- Basic HTTP server and JSON knowledge (Chapter 12)
- Familiar with context package (Chapter 16)
- Understanding of the middleware pattern (Chapter 12, Section 7)

---

## Table of Contents

1. [Why Validate?](#1-why-validate)
2. [JSON Request Parsing](#2-json-request-parsing)
3. [Struct Validation with go-playground/validator](#3-struct-validation-with-go-playgroundvalidator)
4. [Custom Type Validation](#4-custom-type-validation)
5. [Parsing Validation Errors](#5-parsing-validation-errors)
6. [Query Parameter Validation](#6-query-parameter-validation)
7. [Path Parameter Validation](#7-path-parameter-validation)
8. [Standardized Response Format](#8-standardized-response-format)
9. [Response Helper Functions](#9-response-helper-functions)
10. [Error Codes](#10-error-codes)
11. [Pointer Fields for Optional Values](#11-pointer-fields-for-optional-values)
12. [Pagination](#12-pagination)
13. [Content-Type Negotiation](#13-content-type-negotiation)
14. [Request Size Limits](#14-request-size-limits)
15. [Real-World Handler Example](#15-real-world-handler-example)
16. [Key Takeaways](#16-key-takeaways)
17. [Practice Exercises](#17-practice-exercises)

---

## 1. Why Validate?

### The Trust Boundary

Every HTTP request that hits your server crosses a **trust boundary**. Outside that boundary: the wild internet, malicious actors, broken clients, misconfigured frontends, fuzzing bots. Inside that boundary: your business logic, your database, your users' data. Validation is the gatekeeper.

There are three fundamental reasons to validate:

**1. Security (Defense in Depth).** SQL injection, XSS, buffer overflows, and type confusion attacks all start with unvalidated input. Validation is your first line of defense. Even if your database driver uses parameterized queries (and it should), validation prevents garbage from ever reaching the query layer.

**2. Data Integrity.** If your database schema says `session_type` must be one of `"practice"`, `"exam"`, or `"review"`, your API layer should enforce that before the request reaches the database. Otherwise you get constraint violation errors from PostgreSQL that look terrible to users and are harder to debug.

**3. User Experience.** When a user submits invalid data, they deserve clear, actionable error messages: "The `timeLimit` field must be between 1 and 7200 seconds", not "Internal Server Error" or "pq: violates check constraint".

### Where to Validate

Validate at every layer, but differently:

```
Client (browser/mobile)  -->  API Handler  -->  Service Layer  -->  Repository  -->  Database
        |                         |                  |                  |              |
   UX validation           Input validation    Business rules     Query safety    Constraints
   (immediate              (format, types,     (authorization,    (parameterized  (NOT NULL,
    feedback)               ranges, required)   limits, state)     queries)        CHECK, FK)
```

The API handler layer performs **structural validation**: Is the JSON valid? Are required fields present? Are values within acceptable ranges? Is the type correct? The service layer performs **business validation**: Does this user have permission? Does the referenced certification exist? Is the session state valid for this operation?

This chapter focuses on structural validation in the handler layer and the response formatting that communicates validation results back to clients.

### Node.js Comparison

In the Express/Node.js world, validation typically uses one of:

```javascript
// Express + Joi (classic)
const Joi = require('joi');
const schema = Joi.object({
  certificationId: Joi.string().min(1).max(100).required(),
  type: Joi.string().valid('practice', 'exam', 'review').required(),
  questionIds: Joi.array().items(Joi.number().integer().min(1)).min(1).max(500).required(),
  timeLimit: Joi.number().integer().min(1).max(7200).optional(),
});

app.post('/sessions', (req, res) => {
  const { error, value } = schema.validate(req.body, { abortEarly: false });
  if (error) {
    return res.status(422).json({
      status: 'error',
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid request',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message,
        })),
      },
    });
  }
  // ... business logic
});

// Express + Zod (modern)
const { z } = require('zod');
const CreateSessionSchema = z.object({
  certificationId: z.string().min(1).max(100),
  type: z.enum(['practice', 'exam', 'review']),
  questionIds: z.array(z.number().int().min(1)).min(1).max(500),
  timeLimit: z.number().int().min(1).max(7200).optional(),
});
```

Go takes a different approach: validation rules are declared as **struct tags**, and a validator library reads those tags at runtime using reflection. The struct itself is the schema, the tags are the rules, and the validator is the engine. This is more similar to Java's Bean Validation (JSR 380) or Python's Pydantic than to Joi/Zod.

---

## 2. JSON Request Parsing

Before you can validate a request, you must parse it from raw bytes into a Go struct. The `encoding/json` package offers two approaches, and choosing the right one matters for production APIs.

### json.NewDecoder vs json.Unmarshal

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"strings"
)

type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
	Age   int    `json:"age"`
}

func main() {
	jsonBody := `{"name": "Alice", "email": "alice@example.com", "age": 30}`

	// ── Approach 1: json.Unmarshal (buffered) ──
	// Reads the entire body into memory first, then parses.
	bodyBytes := []byte(jsonBody)
	var req1 CreateUserRequest
	if err := json.Unmarshal(bodyBytes, &req1); err != nil {
		fmt.Println("Unmarshal error:", err)
		return
	}
	fmt.Printf("Unmarshal: %+v\n", req1)

	// ── Approach 2: json.NewDecoder (streaming) ──
	// Reads from an io.Reader, parsing as bytes arrive.
	reader := strings.NewReader(jsonBody)
	var req2 CreateUserRequest
	if err := json.NewDecoder(reader).Decode(&req2); err != nil {
		fmt.Println("Decode error:", err)
		return
	}
	fmt.Printf("Decoder:   %+v\n", req2)

	// ── When to use which? ──
	// json.Unmarshal: When you already have []byte (e.g., reading from a file, test cases).
	// json.NewDecoder: When you have an io.Reader (e.g., http.Request.Body).
	//                  More memory-efficient for large payloads because it doesn't
	//                  buffer the entire body before parsing.

	// ── IMPORTANT: Detecting extra data after JSON ──
	// json.NewDecoder does NOT error on trailing garbage by default.
	// For strict parsing, check that nothing remains after Decode:
	trailingGarbage := `{"name": "Bob"}  garbage here`
	reader2 := strings.NewReader(trailingGarbage)
	decoder := json.NewDecoder(reader2)

	var req3 CreateUserRequest
	if err := decoder.Decode(&req3); err != nil {
		fmt.Println("Decode error:", err)
		return
	}
	// Check for trailing content
	if decoder.More() {
		fmt.Println("WARNING: trailing data after JSON object")
	}
	// Even more strict: try to decode again, expect EOF
	if err := decoder.Decode(&struct{}{}); err != io.EOF {
		fmt.Println("WARNING: request body must contain a single JSON object")
	}

	_ = bytes.NewReader(nil) // just to use the bytes import in this example
}
```

### DisallowUnknownFields

By default, Go's JSON decoder silently ignores fields in the JSON that do not match any struct field. This is usually fine, but for strict APIs where you want to catch client typos (e.g., `"emial"` instead of `"email"`), you can disallow unknown fields.

```go
package main

import (
	"encoding/json"
	"fmt"
	"strings"
)

type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

func main() {
	// The JSON has a typo: "emial" instead of "email"
	jsonBody := `{"name": "Alice", "emial": "alice@example.com"}`

	// ── Default behavior: silently ignores unknown "emial" field ──
	var req1 CreateUserRequest
	decoder1 := json.NewDecoder(strings.NewReader(jsonBody))
	if err := decoder1.Decode(&req1); err != nil {
		fmt.Println("Error:", err)
	} else {
		fmt.Printf("Default:  %+v\n", req1) // Email will be ""
	}

	// ── Strict behavior: rejects unknown fields ──
	var req2 CreateUserRequest
	decoder2 := json.NewDecoder(strings.NewReader(jsonBody))
	decoder2.DisallowUnknownFields()
	if err := decoder2.Decode(&req2); err != nil {
		fmt.Printf("Strict:   error: %v\n", err)
		// Output: Strict:   error: json: unknown field "emial"
	}
}
```

### Production JSON Parsing Function

In a real API, you want a single function that handles all the edge cases of JSON parsing. Here is a production-grade implementation:

```go
package request

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"strings"
)

// MaxBodySize is the default maximum request body size (1 MB).
const MaxBodySize = 1_048_576

// DecodeJSON reads and parses a JSON request body into dst.
// It enforces:
//   - A maximum body size to prevent resource exhaustion.
//   - A single JSON object (no trailing content).
//   - Optionally: disallowing unknown fields.
//
// Returns a human-readable error string if parsing fails.
func DecodeJSON(r *http.Request, dst interface{}) error {
	// Enforce Content-Type
	ct := r.Header.Get("Content-Type")
	if ct != "" {
		mediaType := strings.ToLower(strings.TrimSpace(strings.Split(ct, ";")[0]))
		if mediaType != "application/json" {
			return fmt.Errorf("Content-Type header must be application/json, got %q", mediaType)
		}
	}

	// Limit request body size
	r.Body = http.MaxBytesReader(nil, r.Body, MaxBodySize)

	// Create decoder with strict settings
	decoder := json.NewDecoder(r.Body)
	decoder.DisallowUnknownFields()

	// Decode the JSON
	if err := decoder.Decode(dst); err != nil {
		return categorizeJSONError(err)
	}

	// Ensure only one JSON value in the body
	if err := decoder.Decode(&struct{}{}); err != io.EOF {
		return errors.New("request body must contain a single JSON object")
	}

	return nil
}

// categorizeJSONError converts raw json errors into human-friendly messages.
func categorizeJSONError(err error) error {
	var syntaxError *json.SyntaxError
	var unmarshalTypeError *json.UnmarshalTypeError
	var maxBytesError *http.MaxBytesError

	switch {
	case errors.As(err, &syntaxError):
		return fmt.Errorf("malformed JSON at position %d", syntaxError.Offset)

	case errors.Is(err, io.ErrUnexpectedEOF):
		return errors.New("malformed JSON: unexpected end of input")

	case errors.As(err, &unmarshalTypeError):
		return fmt.Errorf("invalid value for field %q: expected %s",
			unmarshalTypeError.Field, unmarshalTypeError.Type)

	case strings.HasPrefix(err.Error(), "json: unknown field "):
		fieldName := strings.TrimPrefix(err.Error(), "json: unknown field ")
		return fmt.Errorf("unknown field %s", fieldName)

	case errors.Is(err, io.EOF):
		return errors.New("request body must not be empty")

	case errors.As(err, &maxBytesError):
		return fmt.Errorf("request body must not be larger than %d bytes", maxBytesError.Limit)

	default:
		return fmt.Errorf("invalid JSON: %s", err.Error())
	}
}
```

### Node.js Comparison

Express automatically parses JSON with `express.json()` middleware:

```javascript
// Express handles JSON parsing automatically
app.use(express.json({ limit: '1mb' }));

app.post('/users', (req, res) => {
  // req.body is already parsed -- no manual decoding needed
  const { name, email } = req.body;
  // But: no type safety, no unknown field detection, no streaming
});
```

Go requires explicit parsing, but gives you control over every aspect: body size limits, unknown field handling, streaming, and precise error categorization. Express gives you convenience at the cost of control.

---

## 3. Struct Validation with go-playground/validator

The `go-playground/validator` package is the de facto standard for struct validation in Go. It uses struct tags to declare validation rules, similar to how `json` tags declare JSON field names.

### Installation

```bash
go get github.com/go-playground/validator/v10
```

### Basic Usage

```go
package main

import (
	"fmt"

	"github.com/go-playground/validator/v10"
)

// Use the "validate" struct tag to declare rules.
type CreateUserRequest struct {
	Name     string `json:"name"     validate:"required,min=2,max=100"`
	Email    string `json:"email"    validate:"required,email"`
	Age      int    `json:"age"      validate:"required,min=13,max=150"`
	Password string `json:"password" validate:"required,min=8,max=72"`
	Role     string `json:"role"     validate:"required,oneof=admin user moderator"`
}

func main() {
	// Create a single validator instance and reuse it.
	// The validator is safe for concurrent use.
	validate := validator.New()

	// ── Valid request ──
	validReq := CreateUserRequest{
		Name:     "Alice",
		Email:    "alice@example.com",
		Age:      30,
		Password: "secureP@ss1",
		Role:     "admin",
	}
	if err := validate.Struct(validReq); err != nil {
		fmt.Println("Unexpected error:", err)
	} else {
		fmt.Println("Valid request: OK")
	}

	// ── Invalid request ──
	invalidReq := CreateUserRequest{
		Name:     "", // required, will fail
		Email:    "not-an-email",
		Age:      5, // min=13, will fail
		Password: "short",
		Role:     "superadmin", // not in oneof
	}
	if err := validate.Struct(invalidReq); err != nil {
		// err is of type validator.ValidationErrors
		fmt.Println("\nValidation errors:")
		for _, e := range err.(validator.ValidationErrors) {
			fmt.Printf("  Field: %-10s Tag: %-10s Value: %v\n",
				e.Field(), e.Tag(), e.Value())
		}
	}
}
```

Output:

```
Valid request: OK

Validation errors:
  Field: Name       Tag: required   Value:
  Field: Email      Tag: email      Value: not-an-email
  Field: Age        Tag: min        Value: 5
  Field: Password   Tag: min        Value: short
  Field: Role       Tag: oneof      Value: superadmin
```

### Commonly Used Built-in Validators

Here is a reference of the most useful built-in validation tags:

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-playground/validator/v10"
)

// This struct demonstrates the most commonly used validation tags.
type ValidationShowcase struct {
	// ── Presence ──
	Required    string `validate:"required"`          // Must not be zero value
	OmitEmpty   string `validate:"omitempty,min=5"`   // Skip validation if empty

	// ── String length ──
	MinLen      string `validate:"min=3"`             // Minimum 3 characters
	MaxLen      string `validate:"max=100"`           // Maximum 100 characters
	ExactLen    string `validate:"len=5"`             // Exactly 5 characters

	// ── Numeric range ──
	MinNum      int    `validate:"min=1"`             // >= 1
	MaxNum      int    `validate:"max=100"`           // <= 100
	GTNum       int    `validate:"gt=0"`              // > 0 (strictly greater than)
	LTNum       int    `validate:"lt=100"`            // < 100

	// ── String formats ──
	Email       string `validate:"email"`             // Valid email format
	URL         string `validate:"url"`               // Valid URL
	URI         string `validate:"uri"`               // Valid URI
	UUID        string `validate:"uuid"`              // Valid UUID (v1-v5)
	UUIDv4      string `validate:"uuid4"`             // Valid UUID v4 specifically
	Alpha       string `validate:"alpha"`             // Letters only
	AlphaNum    string `validate:"alphanum"`          // Letters and digits only
	Numeric     string `validate:"numeric"`           // Numeric string
	IP          string `validate:"ip"`                // Valid IP (v4 or v6)
	IPv4        string `validate:"ipv4"`              // Valid IPv4
	CIDRv4      string `validate:"cidrv4"`            // Valid CIDR notation

	// ── Enum / choices ──
	OneOf       string `validate:"oneof=red green blue"`

	// ── Slice / array ──
	MinItems    []int  `validate:"min=1"`             // At least 1 item
	MaxItems    []int  `validate:"max=10"`            // At most 10 items
	UniqueItems []int  `validate:"unique"`            // All items must be unique

	// ── Dive: validate slice elements ──
	Tags        []string `validate:"required,min=1,max=10,dive,min=1,max=50"`
	// The above means: Tags is required, between 1 and 10 items,
	// and each item (after "dive") must be between 1 and 50 characters.

	// ── Cross-field ──
	Password    string `validate:"required,min=8"`
	ConfirmPass string `validate:"required,eqfield=Password"` // Must equal Password

	// ── Time ──
	StartDate   time.Time `validate:"required"`
	EndDate     time.Time `validate:"required,gtfield=StartDate"` // Must be after StartDate

	// ── Conditional ──
	Country     string `validate:"required"`
	ZipCode     string `validate:"required_if=Country US"` // Required only if Country is "US"
	PostalCode  string `validate:"required_if=Country CA"` // Required only if Country is "CA"
}

func main() {
	validate := validator.New()

	// Quick demo of dive validation
	type TagRequest struct {
		Tags []string `validate:"required,min=1,max=5,dive,min=1,max=20"`
	}

	// Valid
	req := TagRequest{Tags: []string{"golang", "api", "validation"}}
	if err := validate.Struct(req); err != nil {
		fmt.Println("Error:", err)
	} else {
		fmt.Println("Tags valid: OK")
	}

	// Invalid: empty string in slice
	badReq := TagRequest{Tags: []string{"golang", "", "validation"}}
	if err := validate.Struct(badReq); err != nil {
		for _, e := range err.(validator.ValidationErrors) {
			fmt.Printf("  %s failed on '%s' (value: %q)\n",
				e.Namespace(), e.Tag(), e.Value())
		}
	}
	// Output: TagRequest.Tags[1] failed on 'min' (value: "")
}
```

### Registering Custom Validators

The built-in validators cover most cases, but production APIs often need custom validation logic. You can register custom validators that integrate seamlessly with the tag-based system.

```go
package main

import (
	"fmt"
	"strings"
	"unicode"

	"github.com/go-playground/validator/v10"
)

// Custom validator: password must contain at least one uppercase,
// one lowercase, one digit, and one special character.
func strongPassword(fl validator.FieldLevel) bool {
	password := fl.Field().String()
	var (
		hasUpper   bool
		hasLower   bool
		hasDigit   bool
		hasSpecial bool
	)
	for _, ch := range password {
		switch {
		case unicode.IsUpper(ch):
			hasUpper = true
		case unicode.IsLower(ch):
			hasLower = true
		case unicode.IsDigit(ch):
			hasDigit = true
		case unicode.IsPunct(ch) || unicode.IsSymbol(ch):
			hasSpecial = true
		}
	}
	return hasUpper && hasLower && hasDigit && hasSpecial
}

// Custom validator: no leading or trailing whitespace.
func noWhitespace(fl validator.FieldLevel) bool {
	s := fl.Field().String()
	return s == strings.TrimSpace(s)
}

// Custom validator: slug format (lowercase letters, digits, hyphens).
func slug(fl validator.FieldLevel) bool {
	s := fl.Field().String()
	if s == "" {
		return true // Let "required" handle presence
	}
	for _, ch := range s {
		if !unicode.IsLower(ch) && !unicode.IsDigit(ch) && ch != '-' {
			return false
		}
	}
	// No leading/trailing hyphens, no consecutive hyphens
	return !strings.HasPrefix(s, "-") && !strings.HasSuffix(s, "-") && !strings.Contains(s, "--")
}

type RegisterRequest struct {
	Username string `json:"username" validate:"required,min=3,max=30,slug"`
	Email    string `json:"email"    validate:"required,email,nowhitespace"`
	Password string `json:"password" validate:"required,min=8,max=72,strongpassword"`
}

func main() {
	validate := validator.New()

	// Register custom validators
	validate.RegisterValidation("strongpassword", strongPassword)
	validate.RegisterValidation("nowhitespace", noWhitespace)
	validate.RegisterValidation("slug", slug)

	// ── Test cases ──
	tests := []struct {
		name string
		req  RegisterRequest
	}{
		{
			name: "valid",
			req: RegisterRequest{
				Username: "alice-bob",
				Email:    "alice@example.com",
				Password: "Str0ng!Pass",
			},
		},
		{
			name: "weak password",
			req: RegisterRequest{
				Username: "alice",
				Email:    "alice@example.com",
				Password: "password",
			},
		},
		{
			name: "bad slug",
			req: RegisterRequest{
				Username: "Alice Bob",
				Email:    "alice@example.com",
				Password: "Str0ng!Pass",
			},
		},
		{
			name: "whitespace email",
			req: RegisterRequest{
				Username: "alice",
				Email:    " alice@example.com ",
				Password: "Str0ng!Pass",
			},
		},
	}

	for _, tc := range tests {
		fmt.Printf("\n=== %s ===\n", tc.name)
		if err := validate.Struct(tc.req); err != nil {
			for _, e := range err.(validator.ValidationErrors) {
				fmt.Printf("  FAIL: %s -> %s\n", e.Field(), e.Tag())
			}
		} else {
			fmt.Println("  PASS")
		}
	}
}
```

### Using JSON Field Names in Validation

By default, `validator` reports Go struct field names (`CertificationID`), not JSON field names (`certificationId`). You can fix this by registering a tag name function:

```go
package main

import (
	"fmt"
	"reflect"
	"strings"

	"github.com/go-playground/validator/v10"
)

func main() {
	validate := validator.New()

	// Register a function that extracts JSON tag names for error messages.
	validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
		if name == "-" {
			return ""
		}
		return name
	})

	type Request struct {
		CertificationID string `json:"certificationId" validate:"required"`
		QuestionCount   int    `json:"questionCount"   validate:"required,min=1"`
	}

	req := Request{} // All zero values
	if err := validate.Struct(req); err != nil {
		for _, e := range err.(validator.ValidationErrors) {
			// Now reports "certificationId" instead of "CertificationID"
			fmt.Printf("  Field: %s, Tag: %s\n", e.Field(), e.Tag())
		}
	}
}
```

Output:

```
  Field: certificationId, Tag: required
  Field: questionCount, Tag: min
```

---

## 4. Custom Type Validation

Production APIs often have **enumerated types** -- fields that accept only a fixed set of values. In Go, the idiomatic way to handle this is with a custom type backed by a string, combined with custom JSON unmarshaling.

### The Pattern

```go
package main

import (
	"encoding/json"
	"fmt"
)

// ── Step 1: Define the type ──
type SessionType string

const (
	SessionTypePractice SessionType = "practice"
	SessionTypeExam     SessionType = "exam"
	SessionTypeReview   SessionType = "review"
)

// ── Step 2: IsValid method ──
// This is the single source of truth for valid values.
func (s SessionType) IsValid() bool {
	switch s {
	case SessionTypePractice, SessionTypeExam, SessionTypeReview:
		return true
	}
	return false
}

// ── Step 3: String method for display ──
func (s SessionType) String() string {
	return string(s)
}

// ── Step 4: AllSessionTypes for documentation and error messages ──
func AllSessionTypes() []SessionType {
	return []SessionType{
		SessionTypePractice,
		SessionTypeExam,
		SessionTypeReview,
	}
}

// ── Step 5: Custom JSON unmarshaling ──
// This ensures invalid values are rejected at parse time,
// BEFORE validation even runs.
func (s *SessionType) UnmarshalJSON(data []byte) error {
	var str string
	if err := json.Unmarshal(data, &str); err != nil {
		return fmt.Errorf("session type must be a string")
	}
	st := SessionType(str)
	if !st.IsValid() {
		return fmt.Errorf("invalid session type %q: must be one of %v", str, AllSessionTypes())
	}
	*s = st
	return nil
}

// ── Step 6: Custom JSON marshaling (optional but good practice) ──
func (s SessionType) MarshalJSON() ([]byte, error) {
	return json.Marshal(string(s))
}

// ── Using it in a request struct ──
type CreateSessionRequest struct {
	CertificationID string      `json:"certificationId"`
	Type            SessionType `json:"type"`
}

func main() {
	// Valid
	validJSON := `{"certificationId": "aws-saa-c03", "type": "practice"}`
	var req1 CreateSessionRequest
	if err := json.Unmarshal([]byte(validJSON), &req1); err != nil {
		fmt.Println("Error:", err)
	} else {
		fmt.Printf("Valid:   %+v\n", req1)
	}

	// Invalid type
	invalidJSON := `{"certificationId": "aws-saa-c03", "type": "speedrun"}`
	var req2 CreateSessionRequest
	if err := json.Unmarshal([]byte(invalidJSON), &req2); err != nil {
		fmt.Println("Invalid:", err)
	}

	// Wrong type entirely
	wrongTypeJSON := `{"certificationId": "aws-saa-c03", "type": 42}`
	var req3 CreateSessionRequest
	if err := json.Unmarshal([]byte(wrongTypeJSON), &req3); err != nil {
		fmt.Println("Wrong:  ", err)
	}
}
```

Output:

```
Valid:   {CertificationID:aws-saa-c03 Type:practice}
Invalid: invalid session type "speedrun": must be one of [practice exam review]
Wrong:   session type must be a string
```

### Why Custom UnmarshalJSON Instead of Just Validator Tags?

You might wonder: why not just use `validate:"oneof=practice exam review"` and skip the custom unmarshaling? There are two important reasons:

1. **Parse-time rejection.** With `oneof`, the invalid string `"speedrun"` would be accepted by `json.Unmarshal` and stored in the struct as `SessionType("speedrun")`. It would only be caught later when you call `validate.Struct()`. With custom `UnmarshalJSON`, the invalid value is caught immediately during parsing. This means the struct can never contain an invalid value -- a much stronger invariant.

2. **Type safety.** Any function that receives a `SessionType` value can trust that it is valid. Without custom unmarshaling, you have to remember to validate everywhere.

3. **Better error messages.** The `UnmarshalJSON` error message includes the invalid value and the list of valid values. The validator's `oneof` just says "failed on 'oneof'".

### Combining Custom Types with Validator

You can still use `validate:"required"` alongside custom `UnmarshalJSON`. The `required` tag ensures the field is present (not the zero value), while `UnmarshalJSON` ensures the value is valid:

```go
package main

import (
	"encoding/json"
	"fmt"

	"github.com/go-playground/validator/v10"
)

type Difficulty string

const (
	DifficultyEasy   Difficulty = "easy"
	DifficultyMedium Difficulty = "medium"
	DifficultyHard   Difficulty = "hard"
)

func (d Difficulty) IsValid() bool {
	switch d {
	case DifficultyEasy, DifficultyMedium, DifficultyHard:
		return true
	}
	return false
}

func (d *Difficulty) UnmarshalJSON(data []byte) error {
	var str string
	if err := json.Unmarshal(data, &str); err != nil {
		return fmt.Errorf("difficulty must be a string")
	}
	diff := Difficulty(str)
	if !diff.IsValid() {
		return fmt.Errorf("invalid difficulty %q: must be one of [easy, medium, hard]", str)
	}
	*d = diff
	return nil
}

type CreateQuestionRequest struct {
	Text       string     `json:"text"       validate:"required,min=10,max=5000"`
	Difficulty Difficulty `json:"difficulty"  validate:"required"`
	// "required" here ensures the field was present in the JSON.
	// If the JSON has "difficulty": "easy", UnmarshalJSON validates the value.
	// If the JSON omits "difficulty" entirely, the validator catches the zero value.
}

func main() {
	validate := validator.New()

	// Case 1: Valid
	j1 := `{"text": "What is the capital of France?", "difficulty": "easy"}`
	var req1 CreateQuestionRequest
	if err := json.Unmarshal([]byte(j1), &req1); err != nil {
		fmt.Println("Parse error:", err)
		return
	}
	if err := validate.Struct(req1); err != nil {
		fmt.Println("Validation error:", err)
	} else {
		fmt.Println("Case 1: PASS")
	}

	// Case 2: Missing difficulty
	j2 := `{"text": "What is the capital of France?"}`
	var req2 CreateQuestionRequest
	if err := json.Unmarshal([]byte(j2), &req2); err != nil {
		fmt.Println("Parse error:", err)
		return
	}
	if err := validate.Struct(req2); err != nil {
		fmt.Println("Case 2:", err)
		// Output: Key: 'CreateQuestionRequest.Difficulty' Error:Field validation for 'Difficulty' failed on the 'required' tag
	}

	// Case 3: Invalid difficulty value
	j3 := `{"text": "What is the capital of France?", "difficulty": "nightmare"}`
	var req3 CreateQuestionRequest
	if err := json.Unmarshal([]byte(j3), &req3); err != nil {
		fmt.Println("Case 3:", err)
		// Output: invalid difficulty "nightmare": must be one of [easy, medium, hard]
	}
}
```

### More Complex Custom Types: Sort Direction

```go
package main

import (
	"encoding/json"
	"fmt"
)

type SortDirection string

const (
	SortAsc  SortDirection = "asc"
	SortDesc SortDirection = "desc"
)

func (s SortDirection) IsValid() bool {
	switch s {
	case SortAsc, SortDesc:
		return true
	}
	return false
}

func (s *SortDirection) UnmarshalJSON(data []byte) error {
	var str string
	if err := json.Unmarshal(data, &str); err != nil {
		return fmt.Errorf("sort direction must be a string")
	}
	sd := SortDirection(str)
	if !sd.IsValid() {
		return fmt.Errorf("invalid sort direction %q: must be 'asc' or 'desc'", str)
	}
	*s = sd
	return nil
}

// UnmarshalText allows this type to work with query parameters too.
// url.Values uses encoding.TextUnmarshaler.
func (s *SortDirection) UnmarshalText(data []byte) error {
	str := string(data)
	sd := SortDirection(str)
	if !sd.IsValid() {
		return fmt.Errorf("invalid sort direction %q: must be 'asc' or 'desc'", str)
	}
	*s = sd
	return nil
}

func main() {
	j := `{"direction": "asc"}`
	var result struct {
		Direction SortDirection `json:"direction"`
	}
	if err := json.Unmarshal([]byte(j), &result); err != nil {
		fmt.Println("Error:", err)
	} else {
		fmt.Printf("Direction: %s\n", result.Direction) // asc
	}
}
```

### Node.js Comparison

In TypeScript/Zod, you get similar type safety with enums:

```typescript
// TypeScript + Zod
const SessionType = z.enum(['practice', 'exam', 'review']);
type SessionType = z.infer<typeof SessionType>;

// The Go approach is more verbose but provides:
// 1. Compile-time type safety (not just runtime)
// 2. Methods on the type (IsValid, String)
// 3. Parse-time validation (UnmarshalJSON)
// 4. The type carries its constraint everywhere it's used
```

---

## 5. Parsing Validation Errors

The `validator.ValidationErrors` type is a slice of `validator.FieldError` values. Each `FieldError` contains the field name, the tag that failed, the parameter (e.g., `100` from `max=100`), and the actual value. Raw `ValidationErrors` look terrible in API responses. We need to convert them to structured, user-friendly messages.

### The Validation Detail Type

```go
package validation

// ValidationDetail represents a single field validation error
// in a user-friendly format suitable for API responses.
type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}
```

### The Full Error Parser

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"reflect"
	"strings"

	"github.com/go-playground/validator/v10"
)

// ValidationDetail is a user-friendly validation error for a single field.
type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// ParseValidationErrors converts validator.ValidationErrors into a slice
// of user-friendly ValidationDetail values.
func ParseValidationErrors(err error) []ValidationDetail {
	var details []ValidationDetail
	var ve validator.ValidationErrors

	if !errors.As(err, &ve) {
		return nil
	}

	for _, fe := range ve {
		details = append(details, ValidationDetail{
			Field:   toJSONFieldName(fe.Namespace()),
			Message: buildMessage(fe),
		})
	}

	return details
}

// toJSONFieldName converts "CreateSessionRequest.CertificationID" to "certificationId".
// It strips the top-level struct name and converts to the JSON field path.
func toJSONFieldName(namespace string) string {
	// Remove the top-level struct name
	// "CreateSessionRequest.CertificationID" -> "CertificationID"
	// "CreateSessionRequest.QuestionIDs[0]" -> "QuestionIDs[0]"
	parts := strings.SplitN(namespace, ".", 2)
	if len(parts) == 2 {
		return parts[1]
	}
	return namespace
}

// buildMessage creates a human-readable message from a FieldError.
func buildMessage(fe validator.FieldError) string {
	field := fe.Field()

	switch fe.Tag() {
	case "required":
		return fmt.Sprintf("%s is required", field)

	case "min":
		if fe.Kind() == reflect.String {
			return fmt.Sprintf("%s must be at least %s characters", field, fe.Param())
		}
		if fe.Kind() == reflect.Slice || fe.Kind() == reflect.Array {
			return fmt.Sprintf("%s must have at least %s items", field, fe.Param())
		}
		return fmt.Sprintf("%s must be at least %s", field, fe.Param())

	case "max":
		if fe.Kind() == reflect.String {
			return fmt.Sprintf("%s must be at most %s characters", field, fe.Param())
		}
		if fe.Kind() == reflect.Slice || fe.Kind() == reflect.Array {
			return fmt.Sprintf("%s must have at most %s items", field, fe.Param())
		}
		return fmt.Sprintf("%s must be at most %s", field, fe.Param())

	case "len":
		return fmt.Sprintf("%s must be exactly %s characters", field, fe.Param())

	case "email":
		return fmt.Sprintf("%s must be a valid email address", field)

	case "url":
		return fmt.Sprintf("%s must be a valid URL", field)

	case "uuid", "uuid4":
		return fmt.Sprintf("%s must be a valid UUID", field)

	case "oneof":
		return fmt.Sprintf("%s must be one of: %s", field, fe.Param())

	case "gt":
		return fmt.Sprintf("%s must be greater than %s", field, fe.Param())

	case "gte":
		return fmt.Sprintf("%s must be greater than or equal to %s", field, fe.Param())

	case "lt":
		return fmt.Sprintf("%s must be less than %s", field, fe.Param())

	case "lte":
		return fmt.Sprintf("%s must be less than or equal to %s", field, fe.Param())

	case "eqfield":
		return fmt.Sprintf("%s must match %s", field, fe.Param())

	case "unique":
		return fmt.Sprintf("%s must contain unique values", field)

	case "alphanum":
		return fmt.Sprintf("%s must contain only letters and numbers", field)

	case "alpha":
		return fmt.Sprintf("%s must contain only letters", field)

	default:
		return fmt.Sprintf("%s failed validation: %s", field, fe.Tag())
	}
}

// ── Demo ──

type CreateSessionRequest struct {
	CertificationID string   `json:"certificationId" validate:"required,min=1,max=100"`
	Type            string   `json:"type"            validate:"required,oneof=practice exam review"`
	QuestionIDs     []int    `json:"questionIds"     validate:"required,min=1,max=500,dive,min=1"`
	Domains         []string `json:"domains"         validate:"omitempty,dive,min=1,max=100"`
	TimeLimit       *int     `json:"timeLimit"       validate:"omitempty,min=1,max=7200"`
}

func main() {
	validate := validator.New()

	// Use JSON tag names in error messages
	validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
		if name == "-" {
			return ""
		}
		return name
	})

	// An invalid request with multiple errors
	tooSmall := 0
	req := CreateSessionRequest{
		CertificationID: "",                        // required
		Type:            "speedrun",                 // not in oneof
		QuestionIDs:     []int{5, 0, 3},            // 0 fails dive,min=1
		Domains:         []string{"networking", ""}, // "" fails dive,min=1
		TimeLimit:       &tooSmall,                  // 0 fails min=1
	}

	err := validate.Struct(req)
	if err != nil {
		details := ParseValidationErrors(err)

		// Pretty-print the details as JSON
		output, _ := json.MarshalIndent(details, "", "  ")
		fmt.Println(string(output))
	}
}
```

Output:

```json
[
  {
    "field": "certificationId",
    "message": "certificationId is required"
  },
  {
    "field": "type",
    "message": "type must be one of: practice exam review"
  },
  {
    "field": "questionIds[1]",
    "message": "questionIds[1] must be at least 1"
  },
  {
    "field": "domains[1]",
    "message": "domains[1] must be at least 1 characters"
  },
  {
    "field": "timeLimit",
    "message": "timeLimit must be at least 1"
  }
]
```

### Node.js Comparison

Joi and Zod produce similar structured errors:

```javascript
// Joi
const { error } = schema.validate(body, { abortEarly: false });
error.details.map(d => ({
  field: d.path.join('.'),
  message: d.message,
}));
// [
//   { field: "certificationId", message: "\"certificationId\" is required" },
//   { field: "type", message: "\"type\" must be one of [practice, exam, review]" },
// ]

// Zod
const result = schema.safeParse(body);
if (!result.success) {
  result.error.issues.map(issue => ({
    field: issue.path.join('.'),
    message: issue.message,
  }));
}
```

The Go approach requires more manual work to produce friendly messages, but gives you full control over wording and formatting. In a production codebase, you write the `ParseValidationErrors` function once and reuse it in every handler.

---

## 6. Query Parameter Validation

Not all input comes in the request body. List endpoints typically use query parameters for pagination, sorting, and filtering. These parameters need just as much validation as JSON bodies.

### Parsing Query Parameters

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
	"strings"
)

// ── Query Parameter Parsing Helpers ──

// QueryParams wraps url.Values with convenient typed accessors.
type QueryParams struct {
	values map[string][]string
	errors []ValidationDetail
}

type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// NewQueryParams creates a QueryParams from an HTTP request.
func NewQueryParams(r *http.Request) *QueryParams {
	return &QueryParams{
		values: r.URL.Query(),
	}
}

// String returns a string query parameter with a default value.
func (q *QueryParams) String(key, defaultValue string) string {
	v := q.values.Get(key)
	if v == "" {
		return defaultValue
	}
	return v
}

// Int returns an integer query parameter with validation.
func (q *QueryParams) Int(key string, defaultValue, min, max int) int {
	v := q.values.Get(key)
	if v == "" {
		return defaultValue
	}

	n, err := strconv.Atoi(v)
	if err != nil {
		q.errors = append(q.errors, ValidationDetail{
			Field:   key,
			Message: fmt.Sprintf("%s must be a valid integer", key),
		})
		return defaultValue
	}

	if n < min {
		q.errors = append(q.errors, ValidationDetail{
			Field:   key,
			Message: fmt.Sprintf("%s must be at least %d", key, min),
		})
		return defaultValue
	}

	if n > max {
		q.errors = append(q.errors, ValidationDetail{
			Field:   key,
			Message: fmt.Sprintf("%s must be at most %d", key, max),
		})
		return defaultValue
	}

	return n
}

// OneOf returns a string query parameter that must be one of the allowed values.
func (q *QueryParams) OneOf(key, defaultValue string, allowed []string) string {
	v := q.values.Get(key)
	if v == "" {
		return defaultValue
	}

	for _, a := range allowed {
		if v == a {
			return v
		}
	}

	q.errors = append(q.errors, ValidationDetail{
		Field:   key,
		Message: fmt.Sprintf("%s must be one of: %s", key, strings.Join(allowed, ", ")),
	})
	return defaultValue
}

// StringSlice returns a comma-separated query parameter as a slice.
func (q *QueryParams) StringSlice(key string, maxItems int) []string {
	v := q.values.Get(key)
	if v == "" {
		return nil
	}

	items := strings.Split(v, ",")
	if len(items) > maxItems {
		q.errors = append(q.errors, ValidationDetail{
			Field:   key,
			Message: fmt.Sprintf("%s must have at most %d items", key, maxItems),
		})
		return items[:maxItems]
	}

	// Trim whitespace from each item
	for i, item := range items {
		items[i] = strings.TrimSpace(item)
	}

	return items
}

// Bool returns a boolean query parameter.
func (q *QueryParams) Bool(key string, defaultValue bool) bool {
	v := q.values.Get(key)
	if v == "" {
		return defaultValue
	}

	switch strings.ToLower(v) {
	case "true", "1", "yes":
		return true
	case "false", "0", "no":
		return false
	default:
		q.errors = append(q.errors, ValidationDetail{
			Field:   key,
			Message: fmt.Sprintf("%s must be a boolean (true/false)", key),
		})
		return defaultValue
	}
}

// HasErrors returns true if any validation errors were collected.
func (q *QueryParams) HasErrors() bool {
	return len(q.errors) > 0
}

// Errors returns accumulated validation errors.
func (q *QueryParams) Errors() []ValidationDetail {
	return q.errors
}

// ── List Query ──

// ListQuery is a standard set of parsed list/pagination query parameters.
type ListQuery struct {
	Limit   int
	Offset  int
	Sort    string
	Order   string
	Search  string
	Domains []string
	Active  bool
}

// ParseListQuery extracts and validates standard list query parameters.
func ParseListQuery(r *http.Request) (*ListQuery, []ValidationDetail) {
	q := NewQueryParams(r)

	query := &ListQuery{
		Limit:   q.Int("limit", 20, 1, 100),
		Offset:  q.Int("offset", 0, 0, 100000),
		Sort:    q.OneOf("sort", "created_at", []string{"created_at", "updated_at", "name", "score"}),
		Order:   q.OneOf("order", "desc", []string{"asc", "desc"}),
		Search:  q.String("search", ""),
		Domains: q.StringSlice("domains", 20),
		Active:  q.Bool("active", true),
	}

	if q.HasErrors() {
		return nil, q.Errors()
	}

	return query, nil
}

func main() {
	// Simulate a request: GET /api/sessions?limit=50&offset=0&sort=score&order=asc&domains=networking,security
	req, _ := http.NewRequest("GET",
		"/api/sessions?limit=50&offset=0&sort=score&order=asc&domains=networking,security&active=true",
		nil,
	)

	query, errs := ParseListQuery(req)
	if errs != nil {
		output, _ := json.MarshalIndent(errs, "", "  ")
		fmt.Println("Errors:", string(output))
		return
	}

	fmt.Printf("Parsed query: %+v\n", query)

	// Simulate an invalid request
	badReq, _ := http.NewRequest("GET",
		"/api/sessions?limit=abc&offset=-5&sort=invalid&order=maybe",
		nil,
	)

	_, badErrs := ParseListQuery(badReq)
	if badErrs != nil {
		output, _ := json.MarshalIndent(badErrs, "", "  ")
		fmt.Println("\nValidation errors:")
		fmt.Println(string(output))
	}
}
```

Output:

```
Parsed query: &{Limit:50 Offset:0 Sort:score Order:asc Search: Domains:[networking security] Active:true}

Validation errors:
[
  {
    "field": "limit",
    "message": "limit must be a valid integer"
  },
  {
    "field": "offset",
    "message": "offset must be at least 0"
  },
  {
    "field": "sort",
    "message": "sort must be one of: created_at, updated_at, name, score"
  },
  {
    "field": "order",
    "message": "order must be one of: asc, desc"
  }
]
```

### Using Struct Tags for Query Parameters

For a more declarative approach, you can use the validator package with query parameters too by parsing into a struct first:

```go
package main

import (
	"fmt"
	"net/http"
	"reflect"
	"strconv"
	"strings"

	"github.com/go-playground/validator/v10"
)

// ListParams is a struct that represents list query parameters.
// The `query` tag specifies the query parameter name.
type ListParams struct {
	Limit  int    `query:"limit"  validate:"min=1,max=100"`
	Offset int    `query:"offset" validate:"min=0,max=100000"`
	Sort   string `query:"sort"   validate:"oneof=created_at updated_at name score"`
	Order  string `query:"order"  validate:"oneof=asc desc"`
	Search string `query:"search" validate:"max=200"`
}

// DefaultListParams returns sensible defaults.
func DefaultListParams() ListParams {
	return ListParams{
		Limit:  20,
		Offset: 0,
		Sort:   "created_at",
		Order:  "desc",
		Search: "",
	}
}

// BindQuery populates a struct from query parameters using the `query` tag.
func BindQuery(r *http.Request, dst interface{}) error {
	v := reflect.ValueOf(dst)
	if v.Kind() != reflect.Ptr || v.Elem().Kind() != reflect.Struct {
		return fmt.Errorf("dst must be a pointer to a struct")
	}

	v = v.Elem()
	t := v.Type()
	q := r.URL.Query()

	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		tag := field.Tag.Get("query")
		if tag == "" || tag == "-" {
			continue
		}

		value := q.Get(tag)
		if value == "" {
			continue // Keep the default value
		}

		fv := v.Field(i)
		switch fv.Kind() {
		case reflect.String:
			fv.SetString(value)
		case reflect.Int, reflect.Int64:
			n, err := strconv.ParseInt(value, 10, 64)
			if err != nil {
				return fmt.Errorf("invalid value for %s: must be an integer", tag)
			}
			fv.SetInt(n)
		case reflect.Bool:
			b, err := strconv.ParseBool(value)
			if err != nil {
				return fmt.Errorf("invalid value for %s: must be a boolean", tag)
			}
			fv.SetBool(b)
		}
	}

	return nil
}

func main() {
	validate := validator.New()
	validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := fld.Tag.Get("query")
		if name == "" || name == "-" {
			return fld.Name
		}
		return name
	})

	// Simulate: GET /api/items?limit=50&sort=name&order=asc
	req, _ := http.NewRequest("GET",
		"/api/items?limit=50&sort=name&order=asc",
		nil,
	)

	params := DefaultListParams()
	if err := BindQuery(req, &params); err != nil {
		fmt.Println("Bind error:", err)
		return
	}

	if err := validate.Struct(params); err != nil {
		for _, e := range err.(validator.ValidationErrors) {
			fmt.Printf("  %s: %s\n", e.Field(), e.Tag())
		}
		return
	}

	fmt.Printf("Params: %+v\n", params)
	// Output: Params: {Limit:50 Offset:0 Sort:name Order:asc Search:}

	// Invalid params
	badReq, _ := http.NewRequest("GET",
		"/api/items?limit=500&sort=invalid",
		nil,
	)

	params2 := DefaultListParams()
	if err := BindQuery(badReq, &params2); err != nil {
		fmt.Println("Bind error:", err)
		return
	}

	if err := validate.Struct(params2); err != nil {
		fmt.Println("\nValidation errors:")
		for _, e := range err.(validator.ValidationErrors) {
			fmt.Printf("  %s failed '%s' (value: %v, param: %s)\n",
				e.Field(), e.Tag(), e.Value(), e.Param())
		}
	}
}
```

### Node.js Comparison

Express query parameter validation with Joi:

```javascript
// Express + Joi for query parameters
const listSchema = Joi.object({
  limit: Joi.number().integer().min(1).max(100).default(20),
  offset: Joi.number().integer().min(0).max(100000).default(0),
  sort: Joi.string().valid('created_at', 'updated_at', 'name', 'score').default('created_at'),
  order: Joi.string().valid('asc', 'desc').default('desc'),
  search: Joi.string().max(200).default(''),
});

app.get('/api/items', (req, res) => {
  const { error, value } = listSchema.validate(req.query, { abortEarly: false });
  if (error) { /* return 422 */ }
  // value.limit, value.offset, etc.
});
```

Joi and Zod handle query parameters the same as body -- just pass `req.query` instead of `req.body`. In Go, query parameters arrive as `url.Values` (map of string slices), so you need manual type conversion or a binding helper.

---

## 7. Path Parameter Validation

Route parameters like `/api/sessions/:id` or `/api/certifications/:slug` also need validation. The specific approach depends on your router, but the validation patterns are universal.

### Validating UUID Path Parameters

```go
package main

import (
	"fmt"
	"net/http"
	"regexp"
	"strconv"
	"strings"
)

// ── UUID validation ──

var uuidRegex = regexp.MustCompile(
	`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`,
)

// IsValidUUID checks if a string is a valid UUID format.
func IsValidUUID(s string) bool {
	return uuidRegex.MatchString(strings.ToLower(s))
}

// ParseUUID validates and normalizes a UUID string.
// Returns the lowercase UUID or an error.
func ParseUUID(s string) (string, error) {
	lower := strings.ToLower(strings.TrimSpace(s))
	if !uuidRegex.MatchString(lower) {
		return "", fmt.Errorf("invalid UUID format: %q", s)
	}
	return lower, nil
}

// ── Numeric ID validation ──

// ParseID validates a numeric ID path parameter.
func ParseID(s string) (int64, error) {
	id, err := strconv.ParseInt(s, 10, 64)
	if err != nil {
		return 0, fmt.Errorf("invalid ID: must be a positive integer, got %q", s)
	}
	if id <= 0 {
		return 0, fmt.Errorf("invalid ID: must be a positive integer, got %d", id)
	}
	return id, nil
}

// ── Slug validation ──

var slugRegex = regexp.MustCompile(`^[a-z0-9]+(?:-[a-z0-9]+)*$`)

// ParseSlug validates a URL slug.
func ParseSlug(s string) (string, error) {
	if len(s) < 1 || len(s) > 200 {
		return "", fmt.Errorf("slug must be between 1 and 200 characters")
	}
	if !slugRegex.MatchString(s) {
		return "", fmt.Errorf("invalid slug format: %q (must be lowercase alphanumeric with hyphens)", s)
	}
	return s, nil
}

// ── Path parameter extraction ──
// In production, you'd use your router's built-in param extraction.
// This demonstrates the concept with Go 1.22+ net/http routing.

// PathParam extracts a named path parameter from the request.
// For Go 1.22+ with built-in routing: r.PathValue("id")
// For chi router: chi.URLParam(r, "id")
// For gorilla/mux: mux.Vars(r)["id"]
func PathParam(r *http.Request, name string) string {
	// Go 1.22+ built-in routing
	return r.PathValue(name)
}

func main() {
	// ── UUID examples ──
	uuids := []string{
		"550e8400-e29b-41d4-a716-446655440000", // valid
		"550E8400-E29B-41D4-A716-446655440000", // valid (uppercase, will be normalized)
		"not-a-uuid",                            // invalid
		"550e8400-e29b-41d4-a716",               // invalid (too short)
		"",                                       // invalid (empty)
	}

	fmt.Println("=== UUID Validation ===")
	for _, u := range uuids {
		parsed, err := ParseUUID(u)
		if err != nil {
			fmt.Printf("  FAIL: %s -> %s\n", u, err)
		} else {
			fmt.Printf("  OK:   %s -> %s\n", u, parsed)
		}
	}

	// ── Numeric ID examples ──
	ids := []string{"42", "0", "-1", "abc", "9999999999"}

	fmt.Println("\n=== ID Validation ===")
	for _, id := range ids {
		parsed, err := ParseID(id)
		if err != nil {
			fmt.Printf("  FAIL: %s -> %s\n", id, err)
		} else {
			fmt.Printf("  OK:   %s -> %d\n", id, parsed)
		}
	}

	// ── Slug examples ──
	slugs := []string{"aws-saa-c03", "my-certification", "CamelCase", "no--double", "-leading"}

	fmt.Println("\n=== Slug Validation ===")
	for _, s := range slugs {
		parsed, err := ParseSlug(s)
		if err != nil {
			fmt.Printf("  FAIL: %s -> %s\n", s, err)
		} else {
			fmt.Printf("  OK:   %s -> %s\n", s, parsed)
		}
	}
}
```

### Using Path Validation in Handlers

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"regexp"
	"strconv"
	"strings"
)

var uuidRegex = regexp.MustCompile(
	`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`,
)

func ParseUUID(s string) (string, error) {
	lower := strings.ToLower(strings.TrimSpace(s))
	if !uuidRegex.MatchString(lower) {
		return "", fmt.Errorf("invalid UUID format: %q", s)
	}
	return lower, nil
}

func ParseID(s string) (int64, error) {
	id, err := strconv.ParseInt(s, 10, 64)
	if err != nil || id <= 0 {
		return 0, fmt.Errorf("invalid ID: must be a positive integer")
	}
	return id, nil
}

// writeJSON is a simple JSON response helper.
func writeJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(data)
}

// ── Handlers using Go 1.22+ routing ──

func getSessionHandler(w http.ResponseWriter, r *http.Request) {
	// Extract and validate the UUID path parameter
	rawID := r.PathValue("id")
	sessionID, err := ParseUUID(rawID)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{
			"error": fmt.Sprintf("Invalid session ID: %s", err),
		})
		return
	}

	// Use sessionID in business logic...
	writeJSON(w, http.StatusOK, map[string]string{
		"sessionId": sessionID,
		"message":   "Session retrieved",
	})
}

func getQuestionHandler(w http.ResponseWriter, r *http.Request) {
	// Extract and validate the numeric path parameter
	rawID := r.PathValue("id")
	questionID, err := ParseID(rawID)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{
			"error": "Invalid question ID: must be a positive integer",
		})
		return
	}

	writeJSON(w, http.StatusOK, map[string]interface{}{
		"questionId": questionID,
		"message":    "Question retrieved",
	})
}

func main() {
	mux := http.NewServeMux()

	// Go 1.22+ pattern-based routing
	mux.HandleFunc("GET /api/sessions/{id}", getSessionHandler)
	mux.HandleFunc("GET /api/questions/{id}", getQuestionHandler)

	fmt.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Middleware for Path Parameter Validation

For routes that consistently use the same parameter type (e.g., all resource IDs are UUIDs), you can create middleware that validates path parameters before the handler runs:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"regexp"
	"strings"
)

var uuidRegex = regexp.MustCompile(
	`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`,
)

type contextKey string

const validatedIDKey contextKey = "validatedID"

// RequireUUIDParam is middleware that validates a path parameter is a valid UUID.
// On success, the normalized UUID is stored in the request context.
func RequireUUIDParam(paramName string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		raw := r.PathValue(paramName)
		lower := strings.ToLower(strings.TrimSpace(raw))

		if !uuidRegex.MatchString(lower) {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusBadRequest)
			json.NewEncoder(w).Encode(map[string]interface{}{
				"status": "error",
				"error": map[string]interface{}{
					"code":    "INVALID_PARAMETER",
					"message": fmt.Sprintf("Invalid %s: must be a valid UUID", paramName),
				},
			})
			return
		}

		// Store the validated UUID in context
		ctx := context.WithValue(r.Context(), validatedIDKey, lower)
		next(w, r.WithContext(ctx))
	}
}

// GetValidatedID retrieves the validated UUID from context.
func GetValidatedID(ctx context.Context) string {
	id, _ := ctx.Value(validatedIDKey).(string)
	return id
}

// ── Handler (no need to validate ID -- middleware already did it) ──

func getSession(w http.ResponseWriter, r *http.Request) {
	sessionID := GetValidatedID(r.Context())

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status": "ok",
		"data": map[string]string{
			"id":     sessionID,
			"status": "active",
		},
	})
}

func deleteSession(w http.ResponseWriter, r *http.Request) {
	sessionID := GetValidatedID(r.Context())

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status": "ok",
		"data": map[string]string{
			"id":      sessionID,
			"deleted": "true",
		},
	})
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/sessions/{id}", RequireUUIDParam("id", getSession))
	mux.HandleFunc("DELETE /api/sessions/{id}", RequireUUIDParam("id", deleteSession))

	fmt.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 8. Standardized Response Format

Every production API needs a consistent response format. When every endpoint returns the same envelope structure, clients can write generic error handling, and developers can reason about any endpoint without reading its documentation.

### The Response Envelope

```go
package response

import "encoding/json"

// ── Success Response ──

// SuccessResponse is the standard envelope for successful API responses.
type SuccessResponse struct {
	Status string      `json:"status"` // Always "ok"
	Data   interface{} `json:"data"`
	Meta   *Meta       `json:"meta,omitempty"`
}

// Meta contains response metadata, typically for list endpoints.
type Meta struct {
	Total      int    `json:"total,omitempty"`
	Limit      int    `json:"limit,omitempty"`
	Offset     int    `json:"offset,omitempty"`
	HasMore    bool   `json:"hasMore,omitempty"`
	NextCursor string `json:"nextCursor,omitempty"`
}

// ── Error Response ──

// ErrorResponse is the standard envelope for error API responses.
type ErrorResponse struct {
	Status string      `json:"status"` // Always "error"
	Error  ErrorDetail `json:"error"`
}

// ErrorDetail contains structured error information.
type ErrorDetail struct {
	Code    string             `json:"code"`              // Machine-readable error code
	Message string             `json:"message"`           // Human-readable message
	Details []ValidationDetail `json:"details,omitempty"` // Field-level errors
}

// ValidationDetail describes a single field validation error.
type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}
```

### What Responses Look Like

A successful single-resource response:

```json
{
  "status": "ok",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "certificationId": "aws-saa-c03",
    "type": "practice",
    "score": 85,
    "createdAt": "2025-01-15T10:30:00Z"
  }
}
```

A successful list response with metadata:

```json
{
  "status": "ok",
  "data": [
    {"id": "...", "name": "Session 1"},
    {"id": "...", "name": "Session 2"}
  ],
  "meta": {
    "total": 47,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  }
}
```

A validation error response:

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {"field": "certificationId", "message": "certificationId is required"},
      {"field": "type", "message": "type must be one of: practice, exam, review"},
      {"field": "questionIds", "message": "questionIds must have at least 1 items"}
    ]
  }
}
```

A not-found error:

```json
{
  "status": "error",
  "error": {
    "code": "NOT_FOUND",
    "message": "Session not found"
  }
}
```

An unauthorized error:

```json
{
  "status": "error",
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}
```

### Complete Runnable Example

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

// ── Response Types ──

type SuccessResponse struct {
	Status string      `json:"status"`
	Data   interface{} `json:"data"`
	Meta   *Meta       `json:"meta,omitempty"`
}

type Meta struct {
	Total      int    `json:"total,omitempty"`
	Limit      int    `json:"limit,omitempty"`
	Offset     int    `json:"offset,omitempty"`
	HasMore    bool   `json:"hasMore,omitempty"`
	NextCursor string `json:"nextCursor,omitempty"`
}

type ErrorResponse struct {
	Status string      `json:"status"`
	Error  ErrorDetail `json:"error"`
}

type ErrorDetail struct {
	Code    string             `json:"code"`
	Message string             `json:"message"`
	Details []ValidationDetail `json:"details,omitempty"`
}

type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

func main() {
	// ── Demonstrate serialization ──

	// Success response (single item)
	success := SuccessResponse{
		Status: "ok",
		Data: map[string]interface{}{
			"id":              "550e8400-e29b-41d4-a716-446655440000",
			"certificationId": "aws-saa-c03",
			"type":            "practice",
			"score":           85,
		},
	}

	out, _ := json.MarshalIndent(success, "", "  ")
	fmt.Println("=== Success (single) ===")
	fmt.Println(string(out))

	// Success response (list with meta)
	listSuccess := SuccessResponse{
		Status: "ok",
		Data: []map[string]interface{}{
			{"id": "aaa", "name": "Session 1"},
			{"id": "bbb", "name": "Session 2"},
		},
		Meta: &Meta{
			Total:   47,
			Limit:   20,
			Offset:  0,
			HasMore: true,
		},
	}

	out, _ = json.MarshalIndent(listSuccess, "", "  ")
	fmt.Println("\n=== Success (list) ===")
	fmt.Println(string(out))

	// Validation error
	validationErr := ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "VALIDATION_ERROR",
			Message: "Invalid request data",
			Details: []ValidationDetail{
				{Field: "certificationId", Message: "certificationId is required"},
				{Field: "type", Message: "type must be one of: practice, exam, review"},
			},
		},
	}

	out, _ = json.MarshalIndent(validationErr, "", "  ")
	fmt.Println("\n=== Validation Error ===")
	fmt.Println(string(out))

	// Not found error (no details)
	notFound := ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "NOT_FOUND",
			Message: "Session not found",
		},
	}

	out, _ = json.MarshalIndent(notFound, "", "  ")
	fmt.Println("\n=== Not Found ===")
	fmt.Println(string(out))

	_ = http.StatusOK // just to reference net/http
}
```

### Node.js Comparison

Express APIs typically don't have a standardized response format unless you build one:

```javascript
// Common Express pattern -- ad hoc responses
app.get('/api/users/:id', (req, res) => {
  // Some endpoints return: { user: { ... } }
  // Others return: { data: { ... } }
  // Others return: { result: { ... }, success: true }
  // No consistency!
});

// Better: standardized response helpers (similar to what we're building)
const ok = (res, data, meta) => res.json({ status: 'ok', data, meta });
const error = (res, code, message, details) =>
  res.status(code).json({ status: 'error', error: { code, message, details } });
```

The Go approach naturally encourages structure because Go's type system makes you define the response shape upfront. In JavaScript, it is too easy to return whatever shape you feel like in the moment.

---

## 9. Response Helper Functions

Writing `w.Header().Set("Content-Type", "application/json")` and `w.WriteHeader(...)` and `json.NewEncoder(w).Encode(...)` in every handler is tedious and error-prone. Response helper functions eliminate this boilerplate.

### The Complete Response Package

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
)

// ── Types (normally in a separate package) ──

type SuccessResponse struct {
	Status string      `json:"status"`
	Data   interface{} `json:"data"`
	Meta   *Meta       `json:"meta,omitempty"`
}

type Meta struct {
	Total      int    `json:"total,omitempty"`
	Limit      int    `json:"limit,omitempty"`
	Offset     int    `json:"offset,omitempty"`
	HasMore    bool   `json:"hasMore,omitempty"`
	NextCursor string `json:"nextCursor,omitempty"`
}

type ErrorResponse struct {
	Status string      `json:"status"`
	Error  ErrorDetail `json:"error"`
}

type ErrorDetail struct {
	Code    string             `json:"code"`
	Message string             `json:"message"`
	Details []ValidationDetail `json:"details,omitempty"`
}

type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// ── Core write function ──

// writeJSON writes a JSON response with the given status code.
// It logs encoding errors rather than panicking.
func writeJSON(w http.ResponseWriter, statusCode int, payload interface{}) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("X-Content-Type-Options", "nosniff")
	w.WriteHeader(statusCode)

	if err := json.NewEncoder(w).Encode(payload); err != nil {
		// At this point, the status code is already sent.
		// We can only log the error; we cannot change the response.
		log.Printf("ERROR: failed to encode JSON response: %v", err)
	}
}

// ── Success Helpers ──

// OK sends a 200 response with data.
func OK(w http.ResponseWriter, data interface{}) {
	writeJSON(w, http.StatusOK, SuccessResponse{
		Status: "ok",
		Data:   data,
	})
}

// OKWithMeta sends a 200 response with data and metadata.
func OKWithMeta(w http.ResponseWriter, data interface{}, meta *Meta) {
	writeJSON(w, http.StatusOK, SuccessResponse{
		Status: "ok",
		Data:   data,
		Meta:   meta,
	})
}

// OKList sends a 200 response for list endpoints with pagination metadata.
func OKList(w http.ResponseWriter, data interface{}, total, limit, offset int) {
	writeJSON(w, http.StatusOK, SuccessResponse{
		Status: "ok",
		Data:   data,
		Meta: &Meta{
			Total:   total,
			Limit:   limit,
			Offset:  offset,
			HasMore: offset+limit < total,
		},
	})
}

// Created sends a 201 response with the created resource.
func Created(w http.ResponseWriter, data interface{}) {
	writeJSON(w, http.StatusCreated, SuccessResponse{
		Status: "ok",
		Data:   data,
	})
}

// NoContent sends a 204 response with no body.
func NoContent(w http.ResponseWriter) {
	w.WriteHeader(http.StatusNoContent)
}

// ── Error Helpers ──

// BadRequest sends a 400 response.
func BadRequest(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusBadRequest, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "BAD_REQUEST",
			Message: message,
		},
	})
}

// ValidationError sends a 422 response with field-level validation details.
func ValidationError(w http.ResponseWriter, message string, details []ValidationDetail) {
	writeJSON(w, http.StatusUnprocessableEntity, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "VALIDATION_ERROR",
			Message: message,
			Details: details,
		},
	})
}

// Unauthorized sends a 401 response.
func Unauthorized(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusUnauthorized, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "UNAUTHORIZED",
			Message: message,
		},
	})
}

// Forbidden sends a 403 response.
func Forbidden(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusForbidden, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "FORBIDDEN",
			Message: message,
		},
	})
}

// NotFound sends a 404 response.
func NotFound(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusNotFound, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "NOT_FOUND",
			Message: message,
		},
	})
}

// Conflict sends a 409 response.
func Conflict(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusConflict, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "CONFLICT",
			Message: message,
		},
	})
}

// TooManyRequests sends a 429 response.
func TooManyRequests(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusTooManyRequests, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "RATE_LIMITED",
			Message: message,
		},
	})
}

// InternalError sends a 500 response.
// IMPORTANT: Never expose internal error details to clients.
// Log the real error server-side; send a generic message to the client.
func InternalError(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusInternalServerError, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "INTERNAL_ERROR",
			Message: message,
		},
	})
}

// ServiceUnavailable sends a 503 response.
func ServiceUnavailable(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusServiceUnavailable, ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    "SERVICE_UNAVAILABLE",
			Message: message,
		},
	})
}

// ── Demo Server ──

type Session struct {
	ID              string `json:"id"`
	CertificationID string `json:"certificationId"`
	Type            string `json:"type"`
	Score           int    `json:"score"`
}

func main() {
	mux := http.NewServeMux()

	// GET /api/sessions - list with pagination
	mux.HandleFunc("GET /api/sessions", func(w http.ResponseWriter, r *http.Request) {
		sessions := []Session{
			{ID: "aaa-111", CertificationID: "aws-saa-c03", Type: "practice", Score: 85},
			{ID: "bbb-222", CertificationID: "aws-saa-c03", Type: "exam", Score: 92},
		}
		OKList(w, sessions, 47, 20, 0)
	})

	// GET /api/sessions/{id} - single resource
	mux.HandleFunc("GET /api/sessions/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")
		if id == "not-found" {
			NotFound(w, "Session not found")
			return
		}
		OK(w, Session{ID: id, CertificationID: "aws-saa-c03", Type: "practice", Score: 85})
	})

	// POST /api/sessions - create
	mux.HandleFunc("POST /api/sessions", func(w http.ResponseWriter, r *http.Request) {
		// Simulate validation error
		ValidationError(w, "Invalid request data", []ValidationDetail{
			{Field: "certificationId", Message: "certificationId is required"},
			{Field: "type", Message: "type must be one of: practice, exam, review"},
		})
	})

	// DELETE /api/sessions/{id} - no content
	mux.HandleFunc("DELETE /api/sessions/{id}", func(w http.ResponseWriter, r *http.Request) {
		NoContent(w)
	})

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl localhost:8080/api/sessions")
	fmt.Println("  curl localhost:8080/api/sessions/abc-123")
	fmt.Println("  curl localhost:8080/api/sessions/not-found")
	fmt.Println("  curl -X POST localhost:8080/api/sessions")
	fmt.Println("  curl -X DELETE localhost:8080/api/sessions/abc-123")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Node.js Comparison

```javascript
// Express response helpers -- similar concept
class ApiResponse {
  static ok(res, data, meta) {
    res.json({ status: 'ok', data, ...(meta && { meta }) });
  }

  static created(res, data) {
    res.status(201).json({ status: 'ok', data });
  }

  static validationError(res, message, details) {
    res.status(422).json({
      status: 'error',
      error: { code: 'VALIDATION_ERROR', message, details },
    });
  }

  static notFound(res, message = 'Resource not found') {
    res.status(404).json({
      status: 'error',
      error: { code: 'NOT_FOUND', message },
    });
  }

  static internalError(res, message = 'Internal server error') {
    res.status(500).json({
      status: 'error',
      error: { code: 'INTERNAL_ERROR', message },
    });
  }
}

// Usage:
app.get('/api/sessions', (req, res) => {
  ApiResponse.ok(res, sessions, { total: 47, limit: 20, offset: 0 });
});
```

The Go and Node.js patterns are nearly identical here -- the difference is that Go's response types are explicitly defined structs, while JavaScript's are ad-hoc objects. The Go approach catches structural mistakes at compile time.

---

## 10. Error Codes

Error codes are the bridge between your API and your client applications. HTTP status codes tell the client *category* of error (4xx = client error, 5xx = server error). Error codes tell the client *what specifically went wrong* so it can take programmatic action.

### Defining Error Codes

```go
package main

import "fmt"

// ErrorCode is a machine-readable error identifier.
// Clients use these to determine how to handle errors programmatically.
type ErrorCode string

const (
	// ── Client Errors (4xx) ──

	// ErrBadRequest indicates malformed input (bad JSON, wrong content type).
	ErrBadRequest ErrorCode = "BAD_REQUEST"

	// ErrValidation indicates the input was well-formed but semantically invalid.
	ErrValidation ErrorCode = "VALIDATION_ERROR"

	// ErrUnauthorized indicates missing or invalid authentication.
	ErrUnauthorized ErrorCode = "UNAUTHORIZED"

	// ErrForbidden indicates the user is authenticated but lacks permission.
	ErrForbidden ErrorCode = "FORBIDDEN"

	// ErrNotFound indicates the requested resource does not exist.
	ErrNotFound ErrorCode = "NOT_FOUND"

	// ErrConflict indicates a state conflict (e.g., duplicate email, concurrent edit).
	ErrConflict ErrorCode = "CONFLICT"

	// ErrGone indicates a resource that existed but has been permanently deleted.
	ErrGone ErrorCode = "GONE"

	// ErrRateLimited indicates the client has sent too many requests.
	ErrRateLimited ErrorCode = "RATE_LIMITED"

	// ErrPayloadTooLarge indicates the request body exceeds the size limit.
	ErrPayloadTooLarge ErrorCode = "PAYLOAD_TOO_LARGE"

	// ── Server Errors (5xx) ──

	// ErrInternal indicates an unexpected server error.
	ErrInternal ErrorCode = "INTERNAL_ERROR"

	// ErrServiceUnavailable indicates the server is temporarily unable to handle requests.
	ErrServiceUnavailable ErrorCode = "SERVICE_UNAVAILABLE"

	// ErrDatabaseError indicates a database operation failed.
	// NOTE: Use this for logging, but return INTERNAL_ERROR to clients.
	ErrDatabaseError ErrorCode = "DATABASE_ERROR"

	// ── Business Logic Errors ──

	// ErrSessionComplete indicates an operation on a completed session.
	ErrSessionComplete ErrorCode = "SESSION_ALREADY_COMPLETE"

	// ErrInsufficientQuestions indicates not enough questions match the criteria.
	ErrInsufficientQuestions ErrorCode = "INSUFFICIENT_QUESTIONS"

	// ErrSubscriptionRequired indicates the operation requires a paid subscription.
	ErrSubscriptionRequired ErrorCode = "SUBSCRIPTION_REQUIRED"

	// ErrQuotaExceeded indicates the user has exceeded their usage quota.
	ErrQuotaExceeded ErrorCode = "QUOTA_EXCEEDED"
)

// String returns the error code as a string.
func (e ErrorCode) String() string {
	return string(e)
}

func main() {
	// Error codes are constants -- they can be used in switch statements
	code := ErrNotFound

	switch code {
	case ErrBadRequest, ErrValidation:
		fmt.Println("Client sent bad data")
	case ErrUnauthorized:
		fmt.Println("Need to log in")
	case ErrForbidden:
		fmt.Println("Not allowed")
	case ErrNotFound:
		fmt.Println("Resource not found")
	case ErrInternal:
		fmt.Println("Server error")
	default:
		fmt.Println("Unknown error:", code)
	}
}
```

### Mapping Error Codes to HTTP Status Codes

```go
package main

import (
	"fmt"
	"net/http"
)

type ErrorCode string

const (
	ErrBadRequest           ErrorCode = "BAD_REQUEST"
	ErrValidation           ErrorCode = "VALIDATION_ERROR"
	ErrUnauthorized         ErrorCode = "UNAUTHORIZED"
	ErrForbidden            ErrorCode = "FORBIDDEN"
	ErrNotFound             ErrorCode = "NOT_FOUND"
	ErrConflict             ErrorCode = "CONFLICT"
	ErrGone                 ErrorCode = "GONE"
	ErrRateLimited          ErrorCode = "RATE_LIMITED"
	ErrPayloadTooLarge      ErrorCode = "PAYLOAD_TOO_LARGE"
	ErrInternal             ErrorCode = "INTERNAL_ERROR"
	ErrServiceUnavailable   ErrorCode = "SERVICE_UNAVAILABLE"
	ErrSessionComplete      ErrorCode = "SESSION_ALREADY_COMPLETE"
	ErrInsufficientQuestions ErrorCode = "INSUFFICIENT_QUESTIONS"
	ErrSubscriptionRequired ErrorCode = "SUBSCRIPTION_REQUIRED"
	ErrQuotaExceeded        ErrorCode = "QUOTA_EXCEEDED"
)

// HTTPStatus returns the appropriate HTTP status code for an error code.
func (e ErrorCode) HTTPStatus() int {
	switch e {
	case ErrBadRequest:
		return http.StatusBadRequest // 400
	case ErrValidation:
		return http.StatusUnprocessableEntity // 422
	case ErrUnauthorized:
		return http.StatusUnauthorized // 401
	case ErrForbidden:
		return http.StatusForbidden // 403
	case ErrNotFound:
		return http.StatusNotFound // 404
	case ErrConflict:
		return http.StatusConflict // 409
	case ErrGone:
		return http.StatusGone // 410
	case ErrRateLimited:
		return http.StatusTooManyRequests // 429
	case ErrPayloadTooLarge:
		return http.StatusRequestEntityTooLarge // 413
	case ErrServiceUnavailable:
		return http.StatusServiceUnavailable // 503
	case ErrSessionComplete, ErrInsufficientQuestions:
		return http.StatusUnprocessableEntity // 422
	case ErrSubscriptionRequired:
		return http.StatusPaymentRequired // 402
	case ErrQuotaExceeded:
		return http.StatusTooManyRequests // 429
	default:
		return http.StatusInternalServerError // 500
	}
}

func main() {
	codes := []ErrorCode{
		ErrBadRequest, ErrValidation, ErrUnauthorized, ErrForbidden,
		ErrNotFound, ErrConflict, ErrRateLimited, ErrInternal,
		ErrSessionComplete, ErrSubscriptionRequired,
	}

	for _, code := range codes {
		fmt.Printf("  %-30s -> %d %s\n", code, code.HTTPStatus(), http.StatusText(code.HTTPStatus()))
	}
}
```

### Application-Level Error Type

For maximum ergonomics, define an `AppError` type that carries the error code, message, and optional details. Service functions return `*AppError`, and the handler layer converts it to an HTTP response:

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

// ── Error Code ──

type ErrorCode string

const (
	ErrBadRequest    ErrorCode = "BAD_REQUEST"
	ErrValidation    ErrorCode = "VALIDATION_ERROR"
	ErrUnauthorized  ErrorCode = "UNAUTHORIZED"
	ErrForbidden     ErrorCode = "FORBIDDEN"
	ErrNotFound      ErrorCode = "NOT_FOUND"
	ErrConflict      ErrorCode = "CONFLICT"
	ErrInternal      ErrorCode = "INTERNAL_ERROR"
)

func (e ErrorCode) HTTPStatus() int {
	switch e {
	case ErrBadRequest:
		return 400
	case ErrValidation:
		return 422
	case ErrUnauthorized:
		return 401
	case ErrForbidden:
		return 403
	case ErrNotFound:
		return 404
	case ErrConflict:
		return 409
	default:
		return 500
	}
}

// ── Application Error ──

type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// AppError is a structured error that carries enough context
// to generate a complete API error response.
type AppError struct {
	Code    ErrorCode
	Message string
	Details []ValidationDetail
	Err     error // The underlying error (for logging, NOT for clients)
}

// Error implements the error interface.
func (e *AppError) Error() string {
	if e.Err != nil {
		return fmt.Sprintf("%s: %s: %v", e.Code, e.Message, e.Err)
	}
	return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

// Unwrap supports errors.Is and errors.As.
func (e *AppError) Unwrap() error {
	return e.Err
}

// ── Constructor functions for common errors ──

func NewBadRequestError(message string) *AppError {
	return &AppError{Code: ErrBadRequest, Message: message}
}

func NewValidationError(message string, details []ValidationDetail) *AppError {
	return &AppError{Code: ErrValidation, Message: message, Details: details}
}

func NewUnauthorizedError(message string) *AppError {
	return &AppError{Code: ErrUnauthorized, Message: message}
}

func NewForbiddenError(message string) *AppError {
	return &AppError{Code: ErrForbidden, Message: message}
}

func NewNotFoundError(resource string) *AppError {
	return &AppError{Code: ErrNotFound, Message: fmt.Sprintf("%s not found", resource)}
}

func NewConflictError(message string) *AppError {
	return &AppError{Code: ErrConflict, Message: message}
}

func NewInternalError(message string, err error) *AppError {
	return &AppError{Code: ErrInternal, Message: message, Err: err}
}

// ── Response Writer ──

type ErrorResponse struct {
	Status string      `json:"status"`
	Error  ErrorDetail `json:"error"`
}

type ErrorDetail struct {
	Code    string             `json:"code"`
	Message string             `json:"message"`
	Details []ValidationDetail `json:"details,omitempty"`
}

// WriteAppError converts an AppError to an HTTP response.
func WriteAppError(w http.ResponseWriter, appErr *AppError) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(appErr.Code.HTTPStatus())

	json.NewEncoder(w).Encode(ErrorResponse{
		Status: "error",
		Error: ErrorDetail{
			Code:    string(appErr.Code),
			Message: appErr.Message,
			Details: appErr.Details,
		},
	})
}

// ── Example: Service that returns AppError ──

type SessionService struct{}

func (s *SessionService) GetSession(id string) (*map[string]string, *AppError) {
	if id == "not-found" {
		return nil, NewNotFoundError("Session")
	}
	if id == "forbidden" {
		return nil, NewForbiddenError("You do not have access to this session")
	}
	data := map[string]string{"id": id, "status": "active"}
	return &data, nil
}

// ── Example: Handler using AppError ──

func main() {
	service := &SessionService{}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /api/sessions/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id")

		session, appErr := service.GetSession(id)
		if appErr != nil {
			WriteAppError(w, appErr)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]interface{}{
			"status": "ok",
			"data":   session,
		})
	})

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl localhost:8080/api/sessions/abc-123")
	fmt.Println("  curl localhost:8080/api/sessions/not-found")
	fmt.Println("  curl localhost:8080/api/sessions/forbidden")
	http.ListenAndServe(":8080", mux)
}
```

### Node.js Comparison

```javascript
// Express -- custom error class pattern
class AppError extends Error {
  constructor(code, message, statusCode, details = []) {
    super(message);
    this.code = code;
    this.statusCode = statusCode;
    this.details = details;
  }
}

// Express error-handling middleware
app.use((err, req, res, next) => {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      status: 'error',
      error: { code: err.code, message: err.message, details: err.details },
    });
  } else {
    res.status(500).json({
      status: 'error',
      error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' },
    });
  }
});
```

Express has centralized error-handling middleware that catches thrown errors. Go does not have this -- each handler explicitly checks for errors and writes the response. This is more verbose but also more explicit: you always know exactly what error handling a handler does, because it is right there in the code.

---

## 11. Pointer Fields for Optional Values

One of the trickiest aspects of JSON parsing in Go is distinguishing between "the field was not provided" and "the field was provided with the zero value." This matters for:

- **PATCH requests:** You need to know which fields the client wants to update.
- **Optional fields with meaningful zero values:** A `timeLimit` of `0` might mean "no limit" or might mean the field was omitted.
- **Nullable database fields:** A field that can be `NULL` in the database.

### The Problem

```go
package main

import (
	"encoding/json"
	"fmt"
)

// ── Without pointers: cannot distinguish missing from zero ──

type UpdateRequestBad struct {
	Name    string `json:"name"`
	Score   int    `json:"score"`
	IsActive bool  `json:"isActive"`
}

// ── With pointers: nil means "not provided" ──

type UpdateRequestGood struct {
	Name     *string `json:"name"`
	Score    *int    `json:"score"`
	IsActive *bool   `json:"isActive"`
}

func main() {
	// JSON that only sets "name"
	jsonBody := `{"name": "Alice"}`

	// ── Without pointers ──
	var bad UpdateRequestBad
	json.Unmarshal([]byte(jsonBody), &bad)
	fmt.Printf("Bad:  Name=%q Score=%d IsActive=%v\n", bad.Name, bad.Score, bad.IsActive)
	// Bad:  Name="Alice" Score=0 IsActive=false
	// We cannot tell: did the client send score=0 or omit score entirely?
	// Did the client send isActive=false or omit isActive?

	// ── With pointers ──
	var good UpdateRequestGood
	json.Unmarshal([]byte(jsonBody), &good)
	fmt.Printf("Good: Name=%v Score=%v IsActive=%v\n",
		derefStr(good.Name), good.Score, good.IsActive)
	// Good: Name=Alice Score=<nil> IsActive=<nil>
	// Now we KNOW: Name was provided, Score was NOT, IsActive was NOT.

	// JSON that explicitly sets zero values
	jsonZero := `{"name": "", "score": 0, "isActive": false}`

	var good2 UpdateRequestGood
	json.Unmarshal([]byte(jsonZero), &good2)
	fmt.Printf("Zero: Name=%v Score=%v IsActive=%v\n",
		good2.Name, good2.Score, good2.IsActive)
	// Zero: Name=0xc... (points to "") Score=0xc... (points to 0) IsActive=0xc... (points to false)
	// All pointers are non-nil, so we know the client explicitly sent these values.
}

func derefStr(s *string) string {
	if s == nil {
		return "<nil>"
	}
	return *s
}
```

### Pointer Helper Functions

Working with pointers is verbose. Helper functions make it bearable:

```go
package main

import "fmt"

// ── Pointer constructors (to-pointer helpers) ──

// Ptr returns a pointer to the given value.
// This is a generic function (Go 1.18+).
func Ptr[T any](v T) *T {
	return &v
}

// Pre-1.18 alternatives for common types:
func StringPtr(s string) *string { return &s }
func IntPtr(i int) *int          { return &i }
func BoolPtr(b bool) *bool       { return &b }
func Float64Ptr(f float64) *float64 { return &f }

// ── Dereference helpers (from-pointer with defaults) ──

// Deref returns the value pointed to, or the zero value if nil.
func Deref[T any](p *T) T {
	if p == nil {
		var zero T
		return zero
	}
	return *p
}

// DerefOr returns the value pointed to, or a default if nil.
func DerefOr[T any](p *T, defaultVal T) T {
	if p == nil {
		return defaultVal
	}
	return *p
}

// Pre-1.18 alternatives:
func DerefString(s *string, def string) string {
	if s == nil {
		return def
	}
	return *s
}

func DerefInt(i *int, def int) int {
	if i == nil {
		return def
	}
	return *i
}

func DerefBool(b *bool, def bool) bool {
	if b == nil {
		return def
	}
	return *b
}

func main() {
	// Creating pointer values inline (without helpers: ugly)
	// s := "hello"
	// req.Name = &s   // Two lines for one field!

	// With helpers: clean
	type UpdateRequest struct {
		Name     *string `json:"name"`
		Score    *int    `json:"score"`
		IsActive *bool   `json:"isActive"`
	}

	req := UpdateRequest{
		Name:     Ptr("Alice"),
		Score:    Ptr(95),
		IsActive: Ptr(true),
	}

	// Dereferencing with defaults
	name := DerefOr(req.Name, "Unknown")
	score := DerefOr(req.Score, 0)
	active := DerefOr(req.IsActive, false)

	fmt.Printf("Name: %s, Score: %d, Active: %v\n", name, score, active)

	// Nil pointer handling
	var emptyReq UpdateRequest
	name2 := DerefOr(emptyReq.Name, "Default Name")
	score2 := DerefOr(emptyReq.Score, -1)
	fmt.Printf("Empty: Name: %s, Score: %d\n", name2, score2)
}
```

### PATCH Request Pattern

Here is a complete pattern for handling PATCH requests where only provided fields should be updated:

```go
package main

import (
	"encoding/json"
	"fmt"
)

// Ptr returns a pointer to the given value.
func Ptr[T any](v T) *T {
	return &v
}

// ── The model (what's in the database) ──

type User struct {
	ID       string `json:"id"`
	Name     string `json:"name"`
	Email    string `json:"email"`
	Bio      string `json:"bio"`
	Age      int    `json:"age"`
	IsPublic bool   `json:"isPublic"`
}

// ── The PATCH request (all fields are pointers) ──

type UpdateUserRequest struct {
	Name     *string `json:"name"     validate:"omitempty,min=2,max=100"`
	Email    *string `json:"email"    validate:"omitempty,email"`
	Bio      *string `json:"bio"      validate:"omitempty,max=500"`
	Age      *int    `json:"age"      validate:"omitempty,min=13,max=150"`
	IsPublic *bool   `json:"isPublic"`
}

// ApplyTo applies the non-nil fields of the update request to the user.
func (req *UpdateUserRequest) ApplyTo(user *User) {
	if req.Name != nil {
		user.Name = *req.Name
	}
	if req.Email != nil {
		user.Email = *req.Email
	}
	if req.Bio != nil {
		user.Bio = *req.Bio
	}
	if req.Age != nil {
		user.Age = *req.Age
	}
	if req.IsPublic != nil {
		user.IsPublic = *req.IsPublic
	}
}

func main() {
	// Existing user in database
	user := User{
		ID:       "user-123",
		Name:     "Alice",
		Email:    "alice@example.com",
		Bio:      "Go developer",
		Age:      30,
		IsPublic: true,
	}

	fmt.Println("Before:", user)

	// PATCH request: only update name and make profile private
	patchJSON := `{"name": "Alice Smith", "isPublic": false}`

	var req UpdateUserRequest
	if err := json.Unmarshal([]byte(patchJSON), &req); err != nil {
		fmt.Println("Error:", err)
		return
	}

	// Show which fields were provided
	fmt.Printf("\nProvided fields:\n")
	fmt.Printf("  Name:     %v (provided: %v)\n", req.Name, req.Name != nil)
	fmt.Printf("  Email:    %v (provided: %v)\n", req.Email, req.Email != nil)
	fmt.Printf("  Bio:      %v (provided: %v)\n", req.Bio, req.Bio != nil)
	fmt.Printf("  Age:      %v (provided: %v)\n", req.Age, req.Age != nil)
	fmt.Printf("  IsPublic: %v (provided: %v)\n", req.IsPublic, req.IsPublic != nil)

	// Apply only the provided fields
	req.ApplyTo(&user)

	fmt.Println("\nAfter:", user)
	// Name is updated, IsPublic is updated, everything else is unchanged.

	// ── Edge case: explicitly setting to zero value ──
	zeroJSON := `{"bio": "", "age": 0}`
	var req2 UpdateUserRequest
	json.Unmarshal([]byte(zeroJSON), &req2)

	fmt.Printf("\nZero value patch:\n")
	fmt.Printf("  Bio: %v (is nil: %v, value: %q)\n", req2.Bio, req2.Bio == nil, safeDeref(req2.Bio))
	fmt.Printf("  Age: %v (is nil: %v)\n", req2.Age, req2.Age == nil)
	// Both are non-nil! The client explicitly sent "" and 0.
	// Without pointers, we could not distinguish this from omission.

	req2.ApplyTo(&user)
	fmt.Println("\nAfter zero patch:", user)
	// Bio is now "", Age is now 0 -- because the client explicitly set them.
}

func safeDeref(s *string) string {
	if s == nil {
		return "<nil>"
	}
	return *s
}
```

### Node.js Comparison

In JavaScript/TypeScript, `undefined` vs explicit `null` serves a similar purpose:

```typescript
// TypeScript PATCH pattern
interface UpdateUserRequest {
  name?: string;       // undefined = not provided
  email?: string;
  bio?: string | null; // null = explicitly set to null
  age?: number;
}

function applyPatch(user: User, patch: UpdateUserRequest) {
  // Only update fields that are present in the request
  if (patch.name !== undefined) user.name = patch.name;
  if (patch.email !== undefined) user.email = patch.email;
  if (patch.bio !== undefined) user.bio = patch.bio; // Could be null
  if (patch.age !== undefined) user.age = patch.age;
}
```

JavaScript's `undefined` is more natural for "not provided" than Go's nil pointers. But Go's approach is explicit and type-safe -- the pointer type in the struct declaration tells you immediately that the field is optional.

---

## 12. Pagination

Every list endpoint needs pagination. Without it, returning 10,000 rows brings your API (and client) to its knees. There are two major approaches: offset-based and cursor-based.

### Offset-Based Pagination

The simpler approach. The client sends `limit` (page size) and `offset` (skip count). Easy to understand, easy to implement, works well for most use cases.

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
)

// ── Pagination Types ──

type PaginationParams struct {
	Limit  int
	Offset int
}

type PaginationMeta struct {
	Total   int  `json:"total"`
	Limit   int  `json:"limit"`
	Offset  int  `json:"offset"`
	HasMore bool `json:"hasMore"`
}

type PaginatedResponse struct {
	Status string         `json:"status"`
	Data   interface{}    `json:"data"`
	Meta   PaginationMeta `json:"meta"`
}

// ParsePagination extracts and validates pagination parameters.
func ParsePagination(r *http.Request) PaginationParams {
	limit := parseIntParam(r, "limit", 20, 1, 100)
	offset := parseIntParam(r, "offset", 0, 0, 100000)
	return PaginationParams{Limit: limit, Offset: offset}
}

func parseIntParam(r *http.Request, key string, defaultVal, min, max int) int {
	s := r.URL.Query().Get(key)
	if s == "" {
		return defaultVal
	}
	n, err := strconv.Atoi(s)
	if err != nil || n < min {
		return min
	}
	if n > max {
		return max
	}
	return n
}

// ── Simulated Database ──

type Session struct {
	ID    string `json:"id"`
	Name  string `json:"name"`
	Score int    `json:"score"`
}

var allSessions []Session

func init() {
	// Generate 47 fake sessions
	for i := 1; i <= 47; i++ {
		allSessions = append(allSessions, Session{
			ID:    fmt.Sprintf("session-%03d", i),
			Name:  fmt.Sprintf("Practice Session %d", i),
			Score: 50 + (i * 7 % 50),
		})
	}
}

func listSessions(params PaginationParams) ([]Session, int) {
	total := len(allSessions)

	start := params.Offset
	if start > total {
		return []Session{}, total
	}

	end := start + params.Limit
	if end > total {
		end = total
	}

	return allSessions[start:end], total
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/sessions", func(w http.ResponseWriter, r *http.Request) {
		params := ParsePagination(r)
		sessions, total := listSessions(params)

		resp := PaginatedResponse{
			Status: "ok",
			Data:   sessions,
			Meta: PaginationMeta{
				Total:   total,
				Limit:   params.Limit,
				Offset:  params.Offset,
				HasMore: params.Offset+params.Limit < total,
			},
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(resp)
	})

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl 'localhost:8080/api/sessions'                  # First page (default)")
	fmt.Println("  curl 'localhost:8080/api/sessions?limit=10&offset=0'  # Page 1")
	fmt.Println("  curl 'localhost:8080/api/sessions?limit=10&offset=10' # Page 2")
	fmt.Println("  curl 'localhost:8080/api/sessions?limit=10&offset=40' # Last page")
	http.ListenAndServe(":8080", mux)
}
```

### Cursor-Based Pagination

For datasets that change frequently (social feeds, real-time data), offset-based pagination has a problem: if new items are inserted between page requests, items can be duplicated or skipped. Cursor-based pagination solves this by using a stable pointer (cursor) into the dataset.

```go
package main

import (
	"encoding/base64"
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
	"time"
)

// ── Cursor Encoding ──
// A cursor is an opaque string that encodes the position in the dataset.
// Typically it encodes the sort key of the last item returned.

type Cursor struct {
	ID        string    `json:"id"`
	CreatedAt time.Time `json:"createdAt"`
}

// EncodeCursor creates an opaque cursor string from a cursor struct.
func EncodeCursor(c Cursor) string {
	data, _ := json.Marshal(c)
	return base64.URLEncoding.EncodeToString(data)
}

// DecodeCursor parses an opaque cursor string back into a Cursor.
func DecodeCursor(s string) (*Cursor, error) {
	data, err := base64.URLEncoding.DecodeString(s)
	if err != nil {
		return nil, fmt.Errorf("invalid cursor format")
	}
	var c Cursor
	if err := json.Unmarshal(data, &c); err != nil {
		return nil, fmt.Errorf("invalid cursor data")
	}
	return &c, nil
}

// ── Response Types ──

type CursorPaginatedResponse struct {
	Status string      `json:"status"`
	Data   interface{} `json:"data"`
	Meta   CursorMeta  `json:"meta"`
}

type CursorMeta struct {
	HasMore    bool   `json:"hasMore"`
	NextCursor string `json:"nextCursor,omitempty"`
	Count      int    `json:"count"` // Number of items in this page
}

// ── Simulated Data ──

type Post struct {
	ID        string    `json:"id"`
	Title     string    `json:"title"`
	CreatedAt time.Time `json:"createdAt"`
}

var allPosts []Post

func init() {
	base := time.Date(2025, 1, 1, 0, 0, 0, 0, time.UTC)
	for i := 0; i < 25; i++ {
		allPosts = append(allPosts, Post{
			ID:        fmt.Sprintf("post-%03d", i+1),
			Title:     fmt.Sprintf("Blog Post #%d", i+1),
			CreatedAt: base.Add(time.Duration(i) * time.Hour),
		})
	}
}

func listPostsAfterCursor(cursor *Cursor, limit int) ([]Post, bool) {
	var startIdx int
	if cursor != nil {
		for i, p := range allPosts {
			if p.ID == cursor.ID {
				startIdx = i + 1
				break
			}
		}
	}

	end := startIdx + limit
	hasMore := end < len(allPosts)
	if end > len(allPosts) {
		end = len(allPosts)
	}

	return allPosts[startIdx:end], hasMore
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/posts", func(w http.ResponseWriter, r *http.Request) {
		// Parse limit
		limit := 5
		if l := r.URL.Query().Get("limit"); l != "" {
			if n, err := strconv.Atoi(l); err == nil && n >= 1 && n <= 50 {
				limit = n
			}
		}

		// Parse cursor
		var cursor *Cursor
		if c := r.URL.Query().Get("cursor"); c != "" {
			var err error
			cursor, err = DecodeCursor(c)
			if err != nil {
				w.Header().Set("Content-Type", "application/json")
				w.WriteHeader(http.StatusBadRequest)
				json.NewEncoder(w).Encode(map[string]interface{}{
					"status": "error",
					"error": map[string]string{
						"code":    "BAD_REQUEST",
						"message": "Invalid cursor parameter",
					},
				})
				return
			}
		}

		// Fetch data
		posts, hasMore := listPostsAfterCursor(cursor, limit)

		// Build response
		meta := CursorMeta{
			HasMore: hasMore,
			Count:   len(posts),
		}
		if hasMore && len(posts) > 0 {
			lastPost := posts[len(posts)-1]
			meta.NextCursor = EncodeCursor(Cursor{
				ID:        lastPost.ID,
				CreatedAt: lastPost.CreatedAt,
			})
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(CursorPaginatedResponse{
			Status: "ok",
			Data:   posts,
			Meta:   meta,
		})
	})

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl 'localhost:8080/api/posts?limit=5'")
	fmt.Println("  # Copy nextCursor from response, then:")
	fmt.Println("  curl 'localhost:8080/api/posts?limit=5&cursor=<nextCursor>'")
	http.ListenAndServe(":8080", mux)
}
```

### When to Use Which

```
+---------------------+---------------------------+---------------------------+
| Feature             | Offset-Based              | Cursor-Based              |
+---------------------+---------------------------+---------------------------+
| Jump to page N      | Yes (offset = N * limit)  | No (must traverse)        |
| Total count         | Easy (SELECT COUNT)       | Expensive (extra query)   |
| Stable with inserts | No (items shift)          | Yes (cursor is stable)    |
| Implementation      | Simple                    | More complex              |
| SQL complexity      | LIMIT + OFFSET            | WHERE + LIMIT             |
| Performance at depth| Degrades (OFFSET 10000)   | Constant                  |
| Use case            | Admin panels, search      | Feeds, real-time data     |
+---------------------+---------------------------+---------------------------+
```

**Recommendation:** Use offset-based for most CRUD APIs. Switch to cursor-based when you have feeds, real-time data, or you know clients will paginate deeply (offset > 10,000).

---

## 13. Content-Type Negotiation

A production API must ensure clients send the correct `Content-Type` and receive the correct one in responses.

### Enforcing Request Content-Type

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"strings"
)

// RequireJSON is middleware that ensures the request has a JSON content type.
// It only checks requests that are expected to have a body (POST, PUT, PATCH).
func RequireJSON(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Only check methods that should have a body
		if r.Method == http.MethodPost || r.Method == http.MethodPut || r.Method == http.MethodPatch {
			ct := r.Header.Get("Content-Type")
			if ct == "" {
				writeError(w, http.StatusUnsupportedMediaType,
					"Content-Type header is required. Use application/json")
				return
			}

			// Parse the media type (ignore parameters like charset)
			mediaType := strings.ToLower(strings.TrimSpace(strings.Split(ct, ";")[0]))
			if mediaType != "application/json" {
				writeError(w, http.StatusUnsupportedMediaType,
					fmt.Sprintf("Unsupported Content-Type %q. Use application/json", mediaType))
				return
			}
		}

		next.ServeHTTP(w, r)
	})
}

// SetJSONContentType is middleware that sets the Content-Type header for responses.
func SetJSONContentType(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json; charset=utf-8")
		next.ServeHTTP(w, r)
	})
}

func writeError(w http.ResponseWriter, status int, message string) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(map[string]interface{}{
		"status": "error",
		"error": map[string]string{
			"code":    "UNSUPPORTED_MEDIA_TYPE",
			"message": message,
		},
	})
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("POST /api/sessions", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})

	mux.HandleFunc("GET /api/sessions", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})

	// Wrap with middleware
	handler := RequireJSON(SetJSONContentType(mux))

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl -X POST localhost:8080/api/sessions                           # 415 (no Content-Type)")
	fmt.Println("  curl -X POST -H 'Content-Type: text/plain' localhost:8080/api/sessions  # 415")
	fmt.Println("  curl -X POST -H 'Content-Type: application/json' -d '{}' localhost:8080/api/sessions  # 200")
	fmt.Println("  curl localhost:8080/api/sessions                                    # 200 (GET, no body check)")
	log.Fatal(http.ListenAndServe(":8080", handler))
}
```

### Accept Header Handling

Some APIs also check the `Accept` header to ensure the client can handle JSON responses:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"strings"
)

// CheckAcceptJSON is middleware that verifies the client accepts JSON.
func CheckAcceptJSON(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		accept := r.Header.Get("Accept")

		// If no Accept header, assume the client accepts anything (RFC 7231)
		if accept == "" || accept == "*/*" {
			next.ServeHTTP(w, r)
			return
		}

		// Check if the client accepts JSON
		acceptsJSON := false
		for _, mediaType := range strings.Split(accept, ",") {
			mt := strings.ToLower(strings.TrimSpace(strings.Split(mediaType, ";")[0]))
			if mt == "application/json" || mt == "*/*" || mt == "application/*" {
				acceptsJSON = true
				break
			}
		}

		if !acceptsJSON {
			w.Header().Set("Content-Type", "application/json; charset=utf-8")
			w.WriteHeader(http.StatusNotAcceptable)
			json.NewEncoder(w).Encode(map[string]interface{}{
				"status": "error",
				"error": map[string]string{
					"code":    "NOT_ACCEPTABLE",
					"message": "This API only produces application/json responses",
				},
			})
			return
		}

		next.ServeHTTP(w, r)
	})
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /api/hello", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"message": "hello"})
	})

	handler := CheckAcceptJSON(mux)

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl localhost:8080/api/hello                                    # 200 (no Accept header)")
	fmt.Println("  curl -H 'Accept: application/json' localhost:8080/api/hello      # 200")
	fmt.Println("  curl -H 'Accept: text/xml' localhost:8080/api/hello              # 406")
	log.Fatal(http.ListenAndServe(":8080", handler))
}
```

---

## 14. Request Size Limits

Without request size limits, a malicious client can send a 10 GB request body and exhaust your server's memory. Always limit request body size.

### http.MaxBytesReader

Go's standard library provides `http.MaxBytesReader` to enforce body size limits:

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"log"
	"net/http"
)

// MaxBodySize limits for different endpoint types.
const (
	DefaultMaxBody   = 1 << 20  // 1 MB
	SmallMaxBody     = 64 << 10 // 64 KB (for simple JSON payloads)
	LargeMaxBody     = 10 << 20 // 10 MB (for file metadata, bulk operations)
)

// LimitBody is middleware that limits request body size.
func LimitBody(maxBytes int64) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Only limit methods that have a body
			if r.Method == http.MethodPost || r.Method == http.MethodPut || r.Method == http.MethodPatch {
				r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
			}
			next.ServeHTTP(w, r)
		})
	}
}

// isMaxBytesError checks if an error is due to body size exceeding the limit.
func isMaxBytesError(err error) bool {
	var maxBytesErr *http.MaxBytesError
	return errors.As(err, &maxBytesErr)
}

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("POST /api/sessions", func(w http.ResponseWriter, r *http.Request) {
		var body map[string]interface{}
		if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
			if isMaxBytesError(err) {
				w.Header().Set("Content-Type", "application/json")
				w.WriteHeader(http.StatusRequestEntityTooLarge)
				json.NewEncoder(w).Encode(map[string]interface{}{
					"status": "error",
					"error": map[string]string{
						"code":    "PAYLOAD_TOO_LARGE",
						"message": "Request body too large. Maximum size is 64 KB",
					},
				})
				return
			}
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusBadRequest)
			json.NewEncoder(w).Encode(map[string]interface{}{
				"status": "error",
				"error": map[string]string{
					"code":    "BAD_REQUEST",
					"message": "Invalid JSON",
				},
			})
			return
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]interface{}{
			"status":  "ok",
			"data":    body,
			"message": "Parsed successfully",
		})
	})

	// Apply 64 KB limit
	handler := LimitBody(SmallMaxBody)(mux)

	fmt.Println("Server starting on :8080")
	fmt.Println("Try:")
	fmt.Println("  curl -X POST -H 'Content-Type: application/json' -d '{\"name\":\"test\"}' localhost:8080/api/sessions")
	fmt.Println("  # To test limit: send a body larger than 64KB")
	log.Fatal(http.ListenAndServe(":8080", handler))
}
```

### Per-Route Size Limits

Different endpoints may need different size limits. A simple JSON creation endpoint needs 64 KB; a bulk import endpoint might need 10 MB:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
)

// WithBodyLimit wraps a handler with a specific body size limit.
func WithBodyLimit(maxBytes int64, handler http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
		handler(w, r)
	}
}

func main() {
	mux := http.NewServeMux()

	// Small limit for simple JSON endpoints (64 KB)
	mux.HandleFunc("POST /api/sessions",
		WithBodyLimit(64<<10, func(w http.ResponseWriter, r *http.Request) {
			var body map[string]interface{}
			if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
				http.Error(w, "Bad request", http.StatusBadRequest)
				return
			}
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
		}),
	)

	// Larger limit for bulk operations (5 MB)
	mux.HandleFunc("POST /api/questions/bulk",
		WithBodyLimit(5<<20, func(w http.ResponseWriter, r *http.Request) {
			var body map[string]interface{}
			if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
				http.Error(w, "Bad request", http.StatusBadRequest)
				return
			}
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
		}),
	)

	fmt.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Node.js Comparison

```javascript
// Express body size limit
app.use(express.json({ limit: '1mb' })); // Global default

// Per-route limit
app.post('/api/sessions', express.json({ limit: '64kb' }), createSession);
app.post('/api/questions/bulk', express.json({ limit: '5mb' }), bulkImport);
```

Express handles this with a middleware option. Go requires wrapping `r.Body` with `http.MaxBytesReader`. Both achieve the same result.

---

## 15. Real-World Handler Example

Now let us bring everything together into a complete, production-grade handler. This example shows how all the pieces -- JSON parsing, struct validation, custom types, error parsing, response formatting -- work together in a real handler.

### Project Structure

```
krafty-core/
  internal/
    handler/
      session.go          // HTTP handlers
    middleware/
      auth.go             // Authentication middleware
    model/
      session.go          // Domain types
    request/
      decode.go           // JSON parsing
    response/
      response.go         // Response helpers
    service/
      session.go          // Business logic
    validation/
      validator.go        // Validator setup
      errors.go           // Error parsing
```

### The Complete Example

This is a full, runnable program that demonstrates the complete flow:

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"log"
	"log/slog"
	"net/http"
	"os"
	"reflect"
	"regexp"
	"strings"
	"time"

	"github.com/go-playground/validator/v10"
)

// ============================================================
// model/session.go - Domain Types
// ============================================================

// SessionType represents the kind of study session.
type SessionType string

const (
	SessionTypePractice SessionType = "practice"
	SessionTypeExam     SessionType = "exam"
	SessionTypeReview   SessionType = "review"
)

func (s SessionType) IsValid() bool {
	switch s {
	case SessionTypePractice, SessionTypeExam, SessionTypeReview:
		return true
	}
	return false
}

func AllSessionTypes() []SessionType {
	return []SessionType{SessionTypePractice, SessionTypeExam, SessionTypeReview}
}

func (s *SessionType) UnmarshalJSON(data []byte) error {
	var str string
	if err := json.Unmarshal(data, &str); err != nil {
		return fmt.Errorf("session type must be a string")
	}
	st := SessionType(str)
	if !st.IsValid() {
		return fmt.Errorf("invalid session type %q: must be one of %v", str, AllSessionTypes())
	}
	*s = st
	return nil
}

// Session is the domain model for a study session.
type Session struct {
	ID              string      `json:"id"`
	UserID          string      `json:"userId"`
	CertificationID string      `json:"certificationId"`
	Type            SessionType `json:"type"`
	QuestionIDs     []int       `json:"questionIds"`
	Domains         []string    `json:"domains,omitempty"`
	TimeLimit       *int        `json:"timeLimit,omitempty"`
	Score           *int        `json:"score,omitempty"`
	Status          string      `json:"status"`
	CreatedAt       time.Time   `json:"createdAt"`
}

// CreateSessionRequest is the validated request for creating a session.
type CreateSessionRequest struct {
	CertificationID string      `json:"certificationId" validate:"required,min=1,max=100"`
	Type            SessionType `json:"type"            validate:"required"`
	QuestionIDs     []int       `json:"questionIds"     validate:"required,min=1,max=500,dive,min=1"`
	Domains         []string    `json:"domains"         validate:"omitempty,dive,min=1,max=100"`
	TimeLimit       *int        `json:"timeLimit"       validate:"omitempty,min=1,max=7200"`
}

// ============================================================
// validation/validator.go - Validator Setup
// ============================================================

// Validate is the package-level validator instance.
// Created once, reused everywhere. Safe for concurrent use.
var Validate *validator.Validate

func init() {
	Validate = validator.New()

	// Use JSON tag names in error messages
	Validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
		if name == "-" {
			return ""
		}
		return name
	})
}

// ============================================================
// validation/errors.go - Validation Error Parsing
// ============================================================

// ValidationDetail is a user-friendly validation error for one field.
type ValidationDetail struct {
	Field   string `json:"field"`
	Message string `json:"message"`
}

// ParseValidationErrors converts validator.ValidationErrors to user-friendly details.
func ParseValidationErrors(err error) []ValidationDetail {
	var details []ValidationDetail
	var ve validator.ValidationErrors

	if !errors.As(err, &ve) {
		return nil
	}

	for _, fe := range ve {
		details = append(details, ValidationDetail{
			Field:   toJSONFieldName(fe.Namespace()),
			Message: buildMessage(fe),
		})
	}

	return details
}

func toJSONFieldName(namespace string) string {
	parts := strings.SplitN(namespace, ".", 2)
	if len(parts) == 2 {
		return parts[1]
	}
	return namespace
}

func buildMessage(fe validator.FieldError) string {
	field := fe.Field()
	switch fe.Tag() {
	case "required":
		return fmt.Sprintf("%s is required", field)
	case "min":
		if fe.Kind() == reflect.String {
			return fmt.Sprintf("%s must be at least %s characters", field, fe.Param())
		}
		if fe.Kind() == reflect.Slice {
			return fmt.Sprintf("%s must have at least %s items", field, fe.Param())
		}
		return fmt.Sprintf("%s must be at least %s", field, fe.Param())
	case "max":
		if fe.Kind() == reflect.String {
			return fmt.Sprintf("%s must be at most %s characters", field, fe.Param())
		}
		if fe.Kind() == reflect.Slice {
			return fmt.Sprintf("%s must have at most %s items", field, fe.Param())
		}
		return fmt.Sprintf("%s must be at most %s", field, fe.Param())
	case "email":
		return fmt.Sprintf("%s must be a valid email address", field)
	case "oneof":
		return fmt.Sprintf("%s must be one of: %s", field, fe.Param())
	case "uuid", "uuid4":
		return fmt.Sprintf("%s must be a valid UUID", field)
	default:
		return fmt.Sprintf("%s failed validation: %s", field, fe.Tag())
	}
}

// ============================================================
// request/decode.go - JSON Parsing
// ============================================================

const MaxBodySize = 1_048_576 // 1 MB

func DecodeJSON(r *http.Request, dst interface{}) error {
	ct := r.Header.Get("Content-Type")
	if ct != "" {
		mediaType := strings.ToLower(strings.TrimSpace(strings.Split(ct, ";")[0]))
		if mediaType != "application/json" {
			return fmt.Errorf("Content-Type must be application/json")
		}
	}

	r.Body = http.MaxBytesReader(nil, r.Body, MaxBodySize)
	decoder := json.NewDecoder(r.Body)
	decoder.DisallowUnknownFields()

	if err := decoder.Decode(dst); err != nil {
		return categorizeJSONError(err)
	}

	if err := decoder.Decode(&struct{}{}); err != io.EOF {
		return fmt.Errorf("request body must contain a single JSON object")
	}

	return nil
}

func categorizeJSONError(err error) error {
	var syntaxError *json.SyntaxError
	var unmarshalTypeError *json.UnmarshalTypeError
	var maxBytesError *http.MaxBytesError

	switch {
	case errors.As(err, &syntaxError):
		return fmt.Errorf("malformed JSON at position %d", syntaxError.Offset)
	case errors.Is(err, io.ErrUnexpectedEOF):
		return fmt.Errorf("malformed JSON: unexpected end of input")
	case errors.As(err, &unmarshalTypeError):
		return fmt.Errorf("invalid value for field %q: expected %s",
			unmarshalTypeError.Field, unmarshalTypeError.Type)
	case strings.HasPrefix(err.Error(), "json: unknown field "):
		fieldName := strings.TrimPrefix(err.Error(), "json: unknown field ")
		return fmt.Errorf("unknown field %s", fieldName)
	case errors.Is(err, io.EOF):
		return fmt.Errorf("request body must not be empty")
	case errors.As(err, &maxBytesError):
		return fmt.Errorf("request body must not be larger than %d bytes", maxBytesError.Limit)
	default:
		return fmt.Errorf("invalid JSON: %s", err.Error())
	}
}

// ============================================================
// response/response.go - Response Helpers
// ============================================================

type SuccessResponse struct {
	Status string      `json:"status"`
	Data   interface{} `json:"data"`
	Meta   *Meta       `json:"meta,omitempty"`
}

type Meta struct {
	Total int `json:"total,omitempty"`
}

type ErrorResponse struct {
	Status string      `json:"status"`
	Error  ErrorDetail `json:"error"`
}

type ErrorDetail struct {
	Code    string             `json:"code"`
	Message string             `json:"message"`
	Details []ValidationDetail `json:"details,omitempty"`
}

func writeJSON(w http.ResponseWriter, status int, payload interface{}) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Header().Set("X-Content-Type-Options", "nosniff")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(payload); err != nil {
		slog.Error("failed to encode response", "error", err)
	}
}

func RespondOK(w http.ResponseWriter, data interface{}) {
	writeJSON(w, http.StatusOK, SuccessResponse{Status: "ok", Data: data})
}

func RespondCreated(w http.ResponseWriter, data interface{}) {
	writeJSON(w, http.StatusCreated, SuccessResponse{Status: "ok", Data: data})
}

func RespondBadRequest(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusBadRequest, ErrorResponse{
		Status: "error",
		Error:  ErrorDetail{Code: "BAD_REQUEST", Message: message},
	})
}

func RespondValidationError(w http.ResponseWriter, message string, details []ValidationDetail) {
	writeJSON(w, http.StatusUnprocessableEntity, ErrorResponse{
		Status: "error",
		Error:  ErrorDetail{Code: "VALIDATION_ERROR", Message: message, Details: details},
	})
}

func RespondUnauthorized(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusUnauthorized, ErrorResponse{
		Status: "error",
		Error:  ErrorDetail{Code: "UNAUTHORIZED", Message: message},
	})
}

func RespondNotFound(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusNotFound, ErrorResponse{
		Status: "error",
		Error:  ErrorDetail{Code: "NOT_FOUND", Message: message},
	})
}

func RespondInternalError(w http.ResponseWriter, message string) {
	writeJSON(w, http.StatusInternalServerError, ErrorResponse{
		Status: "error",
		Error:  ErrorDetail{Code: "INTERNAL_ERROR", Message: message},
	})
}

// ============================================================
// middleware/auth.go - Authentication Middleware
// ============================================================

type contextKey string

const userIDKey contextKey = "userID"

// AuthMiddleware is a simplified auth middleware that extracts user ID from a header.
// In production, this would validate a JWT token.
func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		userID := r.Header.Get("X-User-ID")
		if userID == "" {
			RespondUnauthorized(w, "Authentication required")
			return
		}

		ctx := context.WithValue(r.Context(), userIDKey, userID)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// GetUserID retrieves the authenticated user ID from context.
func GetUserID(ctx context.Context) string {
	userID, _ := ctx.Value(userIDKey).(string)
	return userID
}

// ============================================================
// service/session.go - Business Logic
// ============================================================

type SessionService struct {
	logger *slog.Logger
}

func NewSessionService(logger *slog.Logger) *SessionService {
	return &SessionService{logger: logger}
}

func (s *SessionService) CreateSession(
	ctx context.Context,
	userID string,
	req *CreateSessionRequest,
) (*Session, error) {
	s.logger.Info("creating session",
		"userId", userID,
		"certificationId", req.CertificationID,
		"type", req.Type,
		"questionCount", len(req.QuestionIDs),
	)

	// In production: validate certification exists, user has access,
	// questions exist and belong to the certification, etc.

	session := &Session{
		ID:              "generated-uuid-here",
		UserID:          userID,
		CertificationID: req.CertificationID,
		Type:            req.Type,
		QuestionIDs:     req.QuestionIDs,
		Domains:         req.Domains,
		TimeLimit:       req.TimeLimit,
		Status:          "active",
		CreatedAt:       time.Now().UTC(),
	}

	return session, nil
}

func (s *SessionService) GetSession(ctx context.Context, userID, sessionID string) (*Session, error) {
	// In production: query database
	if sessionID == "not-found-id" {
		return nil, nil // Not found
	}

	return &Session{
		ID:              sessionID,
		UserID:          userID,
		CertificationID: "aws-saa-c03",
		Type:            SessionTypePractice,
		QuestionIDs:     []int{1, 2, 3, 4, 5},
		Status:          "active",
		CreatedAt:       time.Now().UTC().Add(-1 * time.Hour),
	}, nil
}

// ============================================================
// handler/session.go - HTTP Handlers
// ============================================================

type SessionHandler struct {
	service *SessionService
	logger  *slog.Logger
}

func NewSessionHandler(service *SessionService, logger *slog.Logger) *SessionHandler {
	return &SessionHandler{service: service, logger: logger}
}

// CreateSession handles POST /api/sessions
func (h *SessionHandler) CreateSession(w http.ResponseWriter, r *http.Request) {
	// Step 1: Parse JSON body
	var req CreateSessionRequest
	if err := DecodeJSON(r, &req); err != nil {
		h.logger.Warn("invalid JSON in create session request", "error", err)
		RespondBadRequest(w, err.Error())
		return
	}

	// Step 2: Validate the parsed struct
	if err := Validate.Struct(req); err != nil {
		details := ParseValidationErrors(err)
		h.logger.Warn("validation failed for create session request",
			"errors", len(details),
		)
		RespondValidationError(w, "Invalid request data", details)
		return
	}

	// Step 3: Extract authenticated user
	userID := GetUserID(r.Context())

	// Step 4: Call business logic
	session, err := h.service.CreateSession(r.Context(), userID, &req)
	if err != nil {
		h.logger.Error("failed to create session",
			"userId", userID,
			"error", err,
		)
		RespondInternalError(w, "Failed to create session")
		return
	}

	// Step 5: Return success response
	h.logger.Info("session created",
		"sessionId", session.ID,
		"userId", userID,
	)
	RespondCreated(w, session)
}

var uuidRegex = regexp.MustCompile(
	`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`,
)

// GetSession handles GET /api/sessions/{id}
func (h *SessionHandler) GetSession(w http.ResponseWriter, r *http.Request) {
	// Step 1: Validate path parameter
	sessionID := r.PathValue("id")
	if !uuidRegex.MatchString(strings.ToLower(sessionID)) {
		RespondBadRequest(w, "Invalid session ID: must be a valid UUID")
		return
	}
	sessionID = strings.ToLower(sessionID)

	// Step 2: Extract authenticated user
	userID := GetUserID(r.Context())

	// Step 3: Call business logic
	session, err := h.service.GetSession(r.Context(), userID, sessionID)
	if err != nil {
		h.logger.Error("failed to get session",
			"sessionId", sessionID,
			"error", err,
		)
		RespondInternalError(w, "Failed to retrieve session")
		return
	}
	if session == nil {
		RespondNotFound(w, "Session not found")
		return
	}

	// Step 4: Return success response
	RespondOK(w, session)
}

// ============================================================
// main.go - Wiring
// ============================================================

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	sessionService := NewSessionService(logger)
	sessionHandler := NewSessionHandler(sessionService, logger)

	mux := http.NewServeMux()

	// Register routes
	mux.HandleFunc("POST /api/sessions", sessionHandler.CreateSession)
	mux.HandleFunc("GET /api/sessions/{id}", sessionHandler.GetSession)

	// Apply middleware
	handler := AuthMiddleware(mux)

	fmt.Println("===========================================")
	fmt.Println("  krafty-core API - Request Validation Demo")
	fmt.Println("===========================================")
	fmt.Println()
	fmt.Println("Server starting on :8080")
	fmt.Println()
	fmt.Println("Test commands:")
	fmt.Println()
	fmt.Println("  # Create session (valid)")
	fmt.Println(`  curl -s -X POST http://localhost:8080/api/sessions \`)
	fmt.Println(`    -H "Content-Type: application/json" \`)
	fmt.Println(`    -H "X-User-ID: user-123" \`)
	fmt.Println(`    -d '{"certificationId":"aws-saa-c03","type":"practice","questionIds":[1,2,3,4,5]}' | jq .`)
	fmt.Println()
	fmt.Println("  # Create session (validation error)")
	fmt.Println(`  curl -s -X POST http://localhost:8080/api/sessions \`)
	fmt.Println(`    -H "Content-Type: application/json" \`)
	fmt.Println(`    -H "X-User-ID: user-123" \`)
	fmt.Println(`    -d '{"certificationId":"","type":"speedrun","questionIds":[]}' | jq .`)
	fmt.Println()
	fmt.Println("  # Create session (bad JSON)")
	fmt.Println(`  curl -s -X POST http://localhost:8080/api/sessions \`)
	fmt.Println(`    -H "Content-Type: application/json" \`)
	fmt.Println(`    -H "X-User-ID: user-123" \`)
	fmt.Println(`    -d 'not json' | jq .`)
	fmt.Println()
	fmt.Println("  # Create session (no auth)")
	fmt.Println(`  curl -s -X POST http://localhost:8080/api/sessions \`)
	fmt.Println(`    -H "Content-Type: application/json" \`)
	fmt.Println(`    -d '{}' | jq .`)
	fmt.Println()
	fmt.Println("  # Get session (valid UUID)")
	fmt.Println(`  curl -s http://localhost:8080/api/sessions/550e8400-e29b-41d4-a716-446655440000 \`)
	fmt.Println(`    -H "X-User-ID: user-123" | jq .`)
	fmt.Println()
	fmt.Println("  # Get session (invalid UUID)")
	fmt.Println(`  curl -s http://localhost:8080/api/sessions/not-a-uuid \`)
	fmt.Println(`    -H "X-User-ID: user-123" | jq .`)
	fmt.Println()
	fmt.Println("  # Get session (not found)")
	fmt.Println(`  curl -s http://localhost:8080/api/sessions/not-found-id \`)
	fmt.Println(`    -H "X-User-ID: user-123" | jq .`)
	fmt.Println()

	log.Fatal(http.ListenAndServe(":8080", handler))
}
```

### Expected Responses

**Valid creation request:**

```json
{
  "status": "ok",
  "data": {
    "id": "generated-uuid-here",
    "userId": "user-123",
    "certificationId": "aws-saa-c03",
    "type": "practice",
    "questionIds": [1, 2, 3, 4, 5],
    "status": "active",
    "createdAt": "2025-01-15T10:30:00Z"
  }
}
```

**Validation error:**

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request data",
    "details": [
      {
        "field": "certificationId",
        "message": "certificationId is required"
      },
      {
        "field": "questionIds",
        "message": "questionIds must have at least 1 items"
      }
    ]
  }
}
```

**Bad JSON:**

```json
{
  "status": "error",
  "error": {
    "code": "BAD_REQUEST",
    "message": "invalid JSON: invalid session type \"speedrun\": must be one of [practice exam review]"
  }
}
```

**Unauthorized:**

```json
{
  "status": "error",
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}
```

### The Handler Pattern Summarized

Every production handler follows this pattern:

```
1. Parse the request body (DecodeJSON)
   -> On failure: 400 Bad Request with human-readable message

2. Validate the parsed struct (Validate.Struct)
   -> On failure: 422 Unprocessable Entity with field-level details

3. Validate path/query parameters
   -> On failure: 400 Bad Request

4. Extract context values (user ID, request ID, etc.)

5. Call business logic (service layer)
   -> On failure: map to appropriate error response
   -> Not found: 404
   -> Forbidden: 403
   -> Conflict: 409
   -> Internal: 500 (log real error, generic message to client)

6. Return success response
   -> 200 OK for reads
   -> 201 Created for creates
   -> 204 No Content for deletes
```

### Node.js Comparison: Complete Handler

```javascript
// Express equivalent of the CreateSession handler
const createSession = async (req, res) => {
  // Step 1 + 2: Parse + validate (Zod does both at once)
  const result = CreateSessionSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      status: 'error',
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid request data',
        details: result.error.issues.map(i => ({
          field: i.path.join('.'),
          message: i.message,
        })),
      },
    });
  }

  // Step 3: User ID from auth middleware
  const userId = req.user.id; // Set by passport or custom middleware

  // Step 4: Business logic
  try {
    const session = await sessionService.createSession(userId, result.data);
    return res.status(201).json({ status: 'ok', data: session });
  } catch (err) {
    logger.error('Failed to create session', { userId, error: err });
    return res.status(500).json({
      status: 'error',
      error: { code: 'INTERNAL_ERROR', message: 'Failed to create session' },
    });
  }
};
```

Key differences:

| Aspect | Go | Express/Node.js |
|--------|------|-----------------|
| JSON parsing | Explicit (`DecodeJSON`) | Implicit (`express.json()` middleware) |
| Validation | Separate step (`Validate.Struct`) | Can be combined with parsing (Zod `safeParse`) |
| Type safety | Compile-time struct types | Runtime only (TypeScript adds compile-time) |
| Error handling | Explicit `if err != nil` checks | try/catch or error middleware |
| Custom types | `UnmarshalJSON` on custom types | Zod `.refine()` or custom transforms |
| Response format | Explicit struct types | Ad-hoc objects (unless you enforce structure) |

---

## 16. Key Takeaways

1. **Validate at the trust boundary.** Every HTTP handler that accepts input should parse and validate that input before passing it to the service layer. The handler is your API's gatekeeper.

2. **Use `json.NewDecoder` for HTTP request bodies.** It reads from `io.Reader` (streaming), works naturally with `r.Body`, and supports `DisallowUnknownFields()`. Use `json.Unmarshal` only when you already have `[]byte`.

3. **Always limit request body size.** Use `http.MaxBytesReader` to prevent memory exhaustion from oversized payloads. Set different limits for different endpoints.

4. **Use `go-playground/validator` with struct tags.** Declare validation rules as struct tags (`validate:"required,min=1,max=100"`). Register a tag name function to use JSON field names in error messages. Create a single validator instance and reuse it.

5. **Use custom `UnmarshalJSON` for enum types.** Custom types with `IsValid()` methods and `UnmarshalJSON` implementations catch invalid values at parse time, ensuring your structs can never contain invalid enum values. This is stronger than runtime-only validation.

6. **Convert validation errors to user-friendly messages.** Never expose raw `validator.FieldError` to API clients. Write a `ParseValidationErrors` function once and reuse it everywhere. Map each tag to a human-readable message.

7. **Use pointer fields (`*string`, `*int`) for optional/nullable values.** This lets you distinguish between "the client did not send this field" (nil) and "the client sent the zero value" ("", 0, false). This is essential for PATCH endpoints.

8. **Standardize your response format.** Define `SuccessResponse` and `ErrorResponse` structs and use them consistently across every endpoint. Clients should never have to guess the response shape.

9. **Use response helper functions.** `OK()`, `Created()`, `BadRequest()`, `ValidationError()`, `NotFound()`, `InternalError()` eliminate boilerplate and ensure consistency. Write them once in a `response` package.

10. **Define semantic error codes.** HTTP status codes categorize errors; error codes like `VALIDATION_ERROR`, `NOT_FOUND`, `UNAUTHORIZED` tell clients exactly what happened so they can take programmatic action.

11. **Never expose internal errors to clients.** Log the real error server-side with `slog`. Send a generic message to the client. `"Failed to create session"` is fine; `"pq: duplicate key value violates unique constraint users_email_key"` is not.

12. **Choose the right pagination strategy.** Offset-based for most CRUD APIs (simple, supports "jump to page N"). Cursor-based for feeds and real-time data (stable under inserts, constant performance at depth).

13. **Validate query and path parameters with the same rigor as body data.** Parse integers, validate ranges, check UUIDs, enforce allowed values. Do not trust `r.URL.Query()` or `r.PathValue()` any more than you trust `r.Body`.

14. **Every handler follows the same pattern:** Parse -> Validate -> Extract context -> Business logic -> Respond. This consistency makes handlers predictable, testable, and easy to review.

---

## 17. Practice Exercises

### Exercise 1: Validation Package

Build a reusable `validation` package that:
- Creates a singleton `*validator.Validate` instance.
- Registers a JSON tag name function.
- Registers at least 3 custom validators: `slug`, `strongpassword`, and `nohtml` (rejects strings containing HTML tags).
- Exports a `ParseValidationErrors(err error) []ValidationDetail` function.
- Write table-driven tests for each custom validator and for `ParseValidationErrors`.

### Exercise 2: Request Structs

Define complete request structs with validation tags for these endpoints:
- `POST /api/certifications` (name, slug, description, provider, difficulty, domains list).
- `PATCH /api/certifications/:id` (all fields optional, pointer types).
- `POST /api/questions` (text, options list with at least 2 items, correct answer index, explanation, domain, difficulty).
- `POST /api/sessions/:id/answers` (questionId, selectedOptionIndex, timeSpent in seconds).

Each struct should have appropriate JSON tags, validation tags, and custom types where needed.

### Exercise 3: Response Package

Build a complete `response` package with:
- `SuccessResponse`, `ErrorResponse`, `Meta`, `ErrorDetail`, `ValidationDetail` types.
- Helper functions: `OK`, `OKList`, `Created`, `NoContent`, `BadRequest`, `ValidationError`, `Unauthorized`, `Forbidden`, `NotFound`, `Conflict`, `InternalError`.
- A `WriteAppError` function that accepts an `*AppError` and writes the correct HTTP response.
- Write tests that verify each helper produces the correct status code, Content-Type header, and JSON body.

### Exercise 4: Query Parameter Parser

Build a `QueryParams` parser that:
- Supports `Int`, `String`, `Bool`, `StringSlice`, `OneOf`, `UUID`, and `Time` (RFC3339) parameter types.
- Accumulates errors instead of failing on the first one.
- Returns all errors at once for a better client experience.
- Write a `ParseSessionListQuery(r *http.Request)` function that extracts `limit`, `offset`, `sort`, `order`, `certificationId`, `type`, `status`, `minScore`, `maxScore`, and `domains` from query parameters.
- Write table-driven tests with valid and invalid query strings.

### Exercise 5: Custom Types

Implement 3 custom types with full `IsValid()`, `UnmarshalJSON()`, `MarshalJSON()`, and `String()` methods:
- `Difficulty` (easy, medium, hard, expert).
- `QuestionStatus` (draft, published, archived).
- `SortOrder` (asc, desc).

Write tests that verify:
- Valid values are accepted.
- Invalid values produce clear error messages.
- JSON round-trip (marshal then unmarshal) preserves the value.
- The zero value is handled correctly.

### Exercise 6: Complete CRUD Handlers

Build a complete set of handlers for a "Question" resource:
- `POST /api/questions` -- Create question (validate body, return 201).
- `GET /api/questions` -- List questions (validate query params, return paginated 200).
- `GET /api/questions/{id}` -- Get question (validate UUID path param, return 200 or 404).
- `PATCH /api/questions/{id}` -- Update question (validate UUID + partial body, return 200 or 404).
- `DELETE /api/questions/{id}` -- Delete question (validate UUID, return 204 or 404).

Use an in-memory store (a map or slice). Every handler should use the validation package, response package, and follow the Parse -> Validate -> Logic -> Respond pattern.

### Exercise 7: Middleware Chain

Build a middleware chain that applies to all API routes:
1. Request ID middleware (generates or extracts `X-Request-ID`).
2. Logging middleware (logs method, path, status, duration).
3. Recovery middleware (catches panics, returns 500).
4. Content-Type enforcement middleware.
5. Body size limit middleware.
6. Auth middleware (validates a bearer token header).

Wire them together and verify the chain works by writing integration tests using `httptest.NewServer`.

### Exercise 8: Error Handling Deep Dive

Implement the `AppError` pattern:
- Define an `AppError` struct with `Code`, `Message`, `Details`, and `Err` fields.
- Implement the `error` and `Unwrap` interfaces.
- Create constructor functions for every error type.
- Modify the service layer to return `*AppError` instead of `error`.
- Modify handlers to use `WriteAppError` for service errors.
- Write tests that verify: (a) `errors.Is` and `errors.As` work with `AppError`, (b) internal errors do not leak to clients, (c) the correct HTTP status code is set for each error code.

### Exercise 9: Integration Test Suite

Write a full integration test suite for the session creation handler:
- Test valid request returns 201.
- Test empty body returns 400.
- Test malformed JSON returns 400.
- Test unknown fields returns 400 (with `DisallowUnknownFields`).
- Test missing required fields returns 422 with correct details.
- Test invalid enum value (bad `SessionType`) returns 400.
- Test boundary values (`timeLimit` of 0, 1, 7200, 7201).
- Test oversized body returns 413.
- Test missing auth returns 401.
- Test wrong Content-Type returns 415.

Use `httptest.NewRecorder` and verify status code, Content-Type, and parse the JSON response body to check every field.
