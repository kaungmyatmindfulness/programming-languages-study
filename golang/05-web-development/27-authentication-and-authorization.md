# Chapter 27: Authentication & Authorization

Building production authentication systems is one of the most security-critical tasks in backend development. A single mistake -- storing tokens in plain text, using a weak signing algorithm, forgetting to set `HttpOnly` on a cookie -- can expose every user in your system. This chapter walks through building a real, production-grade authentication system in Go, inspired by patterns from a hybrid auth system (external identity provider validation + internally issued JWTs) as seen in production Go APIs.

We will build everything from first principles: how JWTs work, why Ed25519 is the modern signing choice, how access and refresh token pairs operate, how to deliver tokens securely via cookies, how to write middleware that protects routes, and how to implement role-based authorization. Every code example is complete and runnable. Every decision is explained with the "why" before the "how."

If you have built authentication in Node.js with Passport.js, jsonwebtoken, or express-session, the comparisons throughout will ground each concept in familiar territory.

---

## Prerequisites

Before starting this chapter, you should be comfortable with:

- **HTTP servers and middleware** (Chapter 12: JSON, HTTP & REST APIs)
- **Context package** (Chapter 16: Context Package)
- **Interfaces and structs** (Chapter 6: Structs & Interfaces)
- **Error handling** (Chapter 8: Error Handling)
- **Testing** (Chapter 14: Testing, Benchmarking & Fuzzing)
- **Concurrency basics** -- goroutines, channels, tickers (Chapters 10-11)
- **Database basics** -- SQL queries, connection pools (Chapter 21: Building Microservices)

---

## Table of Contents

1. [Authentication Landscape](#1-authentication-landscape)
2. [JWT Fundamentals](#2-jwt-fundamentals)
3. [JWT with Ed25519 Signing](#3-jwt-with-ed25519-signing)
4. [Access & Refresh Token Pair](#4-access--refresh-token-pair)
5. [Issuing Tokens](#5-issuing-tokens)
6. [Validating Tokens](#6-validating-tokens)
7. [OIDC Token Validation](#7-oidc-token-validation)
8. [Token Exchange Flow](#8-token-exchange-flow)
9. [Cookie-Based Token Delivery](#9-cookie-based-token-delivery)
10. [Token Refresh Endpoint](#10-token-refresh-endpoint)
11. [Refresh Token Storage](#11-refresh-token-storage)
12. [Auth Middleware](#12-auth-middleware)
13. [Extracting User Info from Context](#13-extracting-user-info-from-context)
14. [Role-Based Authorization](#14-role-based-authorization)
15. [Security Best Practices](#15-security-best-practices)
16. [Testing Authentication](#16-testing-authentication)
17. [Real-World Example: Complete Auth System](#17-real-world-example-complete-auth-system)
18. [Key Takeaways](#18-key-takeaways)
19. [Practice Exercises](#19-practice-exercises)

---

## 1. Authentication Landscape

### Authentication vs Authorization

These two terms are used interchangeably in casual conversation, but they are fundamentally different:

- **Authentication (AuthN):** "Who are you?" -- Verifying the identity of a user. When a user logs in with their email and password, or signs in with Google, the system authenticates them.
- **Authorization (AuthZ):** "What can you do?" -- Determining what an authenticated user is allowed to access. An authenticated user might be an `admin` who can delete any resource, or a `viewer` who can only read.

```
+------------------------------------------------------------------+
|                    Authentication Flow                            |
|                                                                   |
|  User Credentials ──► Verify Identity ──► Authenticated User     |
|  (email/password,      (check DB,          (known identity)       |
|   OAuth token,          verify token,                              |
|   API key)              validate cert)                             |
+------------------------------------------------------------------+
|                    Authorization Flow                              |
|                                                                   |
|  Authenticated User ──► Check Permissions ──► Allow/Deny         |
|  (user ID, roles)       (RBAC, ABAC,         (proceed or         |
|                          policy engine)        return 403)        |
+------------------------------------------------------------------+
```

### Session-Based Authentication

The traditional approach, dominant in server-rendered web applications:

1. User submits credentials (email + password)
2. Server verifies credentials against the database
3. Server creates a session (stored in memory, Redis, or database)
4. Server sends back a session ID in a cookie
5. Browser automatically sends the cookie with every request
6. Server looks up the session by ID to identify the user

```
Browser                          Server                    Session Store
  |                                |                           |
  |── POST /login (credentials) ──►|                           |
  |                                |── Create session ────────►|
  |                                |◄── Session ID ────────────|
  |◄── Set-Cookie: sid=abc123 ─────|                           |
  |                                |                           |
  |── GET /api/data ──────────────►|                           |
  |   Cookie: sid=abc123           |── Lookup session ────────►|
  |                                |◄── User data ─────────────|
  |◄── 200 OK (data) ─────────────|                           |
```

**Advantages:**
- Simple to implement and understand
- Easy to revoke: delete the session from the store
- Server has full control over session lifetime

**Disadvantages:**
- Server must store state (memory, Redis, or database)
- Difficult to scale horizontally: session affinity or shared session store required
- Not suitable for mobile apps or third-party API consumers
- CSRF vulnerabilities if not carefully handled

**Node.js equivalent:**
```javascript
// Node.js session-based auth with express-session
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'your-secret',
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 86400000 }
}));

app.post('/login', async (req, res) => {
  const user = await verifyCredentials(req.body.email, req.body.password);
  req.session.userId = user.id;
  res.json({ message: 'Logged in' });
});
```

### Token-Based Authentication (JWT)

The modern approach, dominant in APIs and single-page applications:

1. User submits credentials or external identity token
2. Server verifies and issues a signed token (JWT)
3. Client stores the token (cookie, localStorage, or memory)
4. Client sends the token with every request (cookie or Authorization header)
5. Server validates the token's signature and claims -- no database lookup needed

```
Browser/Client                   Server
  |                                |
  |── POST /auth/login ──────────►|
  |   (credentials or IdP token)   |
  |                                |── Verify + sign JWT
  |◄── JWT (access + refresh) ─────|
  |                                |
  |── GET /api/data ──────────────►|
  |   Authorization: Bearer <JWT>  |
  |                                |── Verify JWT signature
  |                                |── Check expiry
  |                                |── Extract claims (no DB lookup)
  |◄── 200 OK (data) ─────────────|
```

**Advantages:**
- Stateless: server does not store sessions. Any server in a cluster can validate the token.
- Scales horizontally without shared session stores
- Works for APIs, mobile apps, microservices, and third-party consumers
- Self-contained: the token carries the user's identity and permissions

**Disadvantages:**
- Cannot be revoked easily (until expiry) without a blocklist
- Token size is larger than a session ID
- Requires careful security handling (secure storage, short expiry)

### OAuth 2.0 Overview

OAuth 2.0 is an **authorization framework** that allows a third-party application to access a user's resources on another service without exposing their password. It defines four grant types, but the most common for web apps is the **Authorization Code flow:**

```
+--------+                               +---------------+
|        |── (1) Authorization Request ──►|               |
|        |                                | Authorization |
|        |◄── (2) Authorization Code ─────|    Server     |
| Client |                                |  (Google,     |
|        |── (3) Code + Client Secret ───►|   GitHub,     |
|        |                                |   etc.)       |
|        |◄── (4) Access Token ───────────|               |
|        |                                +---------------+
|        |
|        |── (5) API Request + Token ────►+---------------+
|        |                                |   Resource    |
|        |◄── (6) Protected Resource ─────|    Server     |
+--------+                                +---------------+
```

OAuth 2.0 **does not define how to authenticate the user** -- it only defines how to authorize access. That is where OIDC comes in.

### OpenID Connect (OIDC)

OpenID Connect is an **identity layer on top of OAuth 2.0**. It adds:

- An **ID Token** (a JWT) that contains user identity information (email, name, picture)
- A standardized **UserInfo endpoint** for fetching additional profile data
- A **discovery mechanism** (`.well-known/openid-configuration`) for automatic provider configuration
- Standardized **scopes** (`openid`, `profile`, `email`)

When a user clicks "Sign in with Google," the flow is:

1. Frontend redirects user to Google's authorization endpoint
2. User authenticates with Google and consents
3. Google redirects back with an authorization code
4. Frontend (or backend) exchanges the code for tokens
5. Google returns an **ID Token** (JWT) + Access Token
6. Your backend validates the ID Token and extracts user info

This is the foundation of the hybrid auth pattern we will build: the frontend handles the OAuth/OIDC flow with an identity provider (Google, Apple, etc.), then sends the ID Token to your backend, which validates it and issues its own JWT pair.

### The Hybrid Approach (What We Will Build)

```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│          │ (1)     │              │ (2)     │              │
│  Client  │────────►│   Identity   │────────►│   Client     │
│  (SPA)   │ OAuth   │   Provider   │ ID Token│   (SPA)      │
│          │ flow    │  (Google)    │         │              │
└──────────┘         └──────────────┘         └──────┬───────┘
                                                     │
                                              (3) POST /auth/token-exchange
                                                  Body: { id_token: "..." }
                                                     │
                                                     ▼
                                              ┌──────────────┐
                                              │              │
                                              │  Your Go     │
                                              │  Backend     │
                                              │              │
                                              │ (4) Validate │
                                              │  IdP token   │
                                              │              │
                                              │ (5) Issue    │
                                              │  own JWT     │
                                              │  pair        │
                                              │              │
                                              │ (6) Set      │
                                              │  cookies     │
                                              └──────────────┘
```

**Why this pattern?**

- The frontend handles the complex OAuth redirect flow with the identity provider
- Your backend never touches user passwords
- Your backend issues its own short-lived tokens with exactly the claims you need
- You control token lifetime, refresh logic, and revocation
- Tokens are delivered via httpOnly cookies (not accessible to JavaScript, preventing XSS)

---

## 2. JWT Fundamentals

### What is a JWT?

A JSON Web Token (JWT, pronounced "jot") is a compact, URL-safe token format defined by [RFC 7519](https://tools.ietf.org/html/rfc7519). It contains three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoiYWxpY2VAZXhhbXBsZS5jb20iLCJpc3MiOiJteS1hcGkiLCJleHAiOjE3MDkzMjk2MDB9.signature_bytes_here
|___________________________________|  |______________________________________________________________________________|  |_____________________|
           Header                                            Payload                                                      Signature
```

### Header

The header declares the token type and signing algorithm:

```json
{
  "alg": "EdDSA",
  "typ": "JWT"
}
```

Common algorithms:
- `HS256` -- HMAC with SHA-256 (symmetric: same key signs and verifies)
- `RS256` -- RSA with SHA-256 (asymmetric: private key signs, public key verifies)
- `ES256` -- ECDSA with P-256 curve (asymmetric, smaller keys than RSA)
- `EdDSA` -- Edwards-curve Digital Signature Algorithm (asymmetric, fastest, smallest signatures)

### Payload (Claims)

The payload contains **claims** -- statements about the user and the token itself:

```json
{
  "sub": "user_123",
  "email": "alice@example.com",
  "name": "Alice Johnson",
  "iss": "my-api",
  "aud": ["my-api"],
  "exp": 1709329600,
  "iat": 1709328700,
  "nbf": 1709328700
}
```

**Registered claims** (defined by RFC 7519):

| Claim | Name | Description |
|-------|------|-------------|
| `sub` | Subject | The user ID or entity the token represents |
| `iss` | Issuer | Who issued the token (your service name) |
| `aud` | Audience | Who the token is intended for |
| `exp` | Expiration Time | Unix timestamp after which the token is invalid |
| `iat` | Issued At | Unix timestamp when the token was created |
| `nbf` | Not Before | Unix timestamp before which the token is invalid |
| `jti` | JWT ID | Unique identifier for the token (for revocation) |

**Custom claims** -- you can add any application-specific data:

| Claim | Description |
|-------|-------------|
| `email` | User's email address |
| `name` | User's display name |
| `picture` | URL to user's avatar |
| `roles` | Array of roles (e.g., `["admin", "editor"]`) |

### Signature

The signature ensures the token has not been tampered with. It is computed over the header and payload:

```
SIGNATURE = Sign(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  signingKey
)
```

Anyone with the verification key (public key for asymmetric algorithms, shared secret for HMAC) can verify the signature. If even one byte of the header or payload is changed, the signature becomes invalid.

### Important: JWTs Are Not Encrypted

A common misconception: JWTs are **signed**, not **encrypted**. Anyone can decode the header and payload by Base64-decoding them. **Never put secrets in a JWT.** The signature only guarantees **integrity** (the data has not been modified) and **authenticity** (the token was issued by someone with the signing key).

```go
package main

import (
	"encoding/base64"
	"fmt"
	"strings"
)

func main() {
	// A JWT can be decoded by anyone -- it is NOT encrypted
	token := "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoiYWxpY2VAZXhhbXBsZS5jb20ifQ.signature"

	parts := strings.Split(token, ".")
	if len(parts) != 3 {
		fmt.Println("Invalid JWT format")
		return
	}

	// Decode header
	header, err := base64.RawURLEncoding.DecodeString(parts[0])
	if err != nil {
		fmt.Println("Error decoding header:", err)
		return
	}
	fmt.Println("Header:", string(header))
	// Output: Header: {"alg":"EdDSA","typ":"JWT"}

	// Decode payload
	payload, err := base64.RawURLEncoding.DecodeString(parts[1])
	if err != nil {
		fmt.Println("Error decoding payload:", err)
		return
	}
	fmt.Println("Payload:", string(payload))
	// Output: Payload: {"sub":"user_123","email":"alice@example.com"}

	// The signature part cannot be forged without the private key,
	// but the header and payload are readable by anyone.
}
```

### Node.js Comparison

In Node.js, the most common JWT library is `jsonwebtoken`:

```javascript
// Node.js -- signing a JWT
const jwt = require('jsonwebtoken');

const token = jwt.sign(
  { sub: 'user_123', email: 'alice@example.com' },
  'your-secret-key',
  { algorithm: 'HS256', expiresIn: '15m' }
);

// Node.js -- verifying a JWT
try {
  const decoded = jwt.verify(token, 'your-secret-key');
  console.log(decoded.sub); // 'user_123'
} catch (err) {
  console.error('Invalid token:', err.message);
}
```

In Go, the equivalent library is `github.com/golang-jwt/jwt/v5`, which we will use throughout this chapter. The key difference: Go's type system means claims are defined as structs with explicit types, not arbitrary objects.

---

## 3. JWT with Ed25519 Signing

### Why Ed25519?

Ed25519 is an Edwards-curve Digital Signature Algorithm that provides:

1. **Speed:** Ed25519 signing is ~20-30x faster than RSA-2048. Verification is ~5-10x faster.
2. **Small keys:** 32-byte private seed, 32-byte public key (vs 256+ bytes for RSA-2048).
3. **Small signatures:** 64 bytes (vs 256 bytes for RSA-2048).
4. **Security:** 128-bit security level, resistant to timing attacks by design.
5. **Deterministic:** Same input always produces the same signature (no randomness needed).
6. **No configuration pitfalls:** Unlike RSA, there is no padding scheme to choose wrong.

```
+---------------------+----------+------------+------------------+-------------+
| Algorithm           | Key Size | Sig Size   | Sign Speed       | Security    |
+---------------------+----------+------------+------------------+-------------+
| HMAC-SHA256 (HS256) | 32 bytes | 32 bytes   | Fastest          | Symmetric   |
| RSA-2048 (RS256)    | 256 bytes| 256 bytes  | Slow             | 112-bit     |
| ECDSA P-256 (ES256) | 32 bytes | 64 bytes   | Fast             | 128-bit     |
| Ed25519 (EdDSA)     | 32 bytes | 64 bytes   | Fastest (asym)   | 128-bit     |
+---------------------+----------+------------+------------------+-------------+
```

**Why not HMAC (HS256)?** HMAC is symmetric -- the same secret signs and verifies. If you have multiple services that need to verify tokens, each one needs the secret, increasing the attack surface. With Ed25519, only the issuing service holds the private key; all other services verify with the public key.

**Why not RSA (RS256)?** RSA keys are large, signatures are large, and signing is slow. RSA-2048 is still secure, but Ed25519 is faster, smaller, and simpler. RSA also has footguns (padding oracles, small exponent attacks) that Ed25519 does not.

### Go's Built-In Ed25519 Support

Go's standard library includes Ed25519 in `crypto/ed25519`. No third-party cryptography libraries are needed.

```go
package main

import (
	"crypto/ed25519"
	"encoding/hex"
	"fmt"
)

func main() {
	// Generate a new Ed25519 key pair
	publicKey, privateKey, err := ed25519.GenerateKey(nil) // nil uses crypto/rand
	if err != nil {
		panic(err)
	}

	fmt.Printf("Private key (seed): %s\n", hex.EncodeToString(privateKey.Seed()))
	fmt.Printf("Public key:         %s\n", hex.EncodeToString(publicKey))

	// Sign a message
	message := []byte("hello, world")
	signature := ed25519.Sign(privateKey, message)
	fmt.Printf("Signature:          %s\n", hex.EncodeToString(signature))

	// Verify the signature
	valid := ed25519.Verify(publicKey, message, signature)
	fmt.Printf("Signature valid:    %v\n", valid) // true

	// Tamper with the message
	tampered := []byte("hello, world!")
	valid = ed25519.Verify(publicKey, tampered, signature)
	fmt.Printf("Tampered valid:     %v\n", valid) // false
}
```

### Deterministic Key Generation from a Seed

In production, you do not call `GenerateKey` each time the server starts. Instead, you store a 32-byte **seed** in your secrets manager (AWS Secrets Manager, HashiCorp Vault, environment variable) and derive the key pair deterministically:

```go
package main

import (
	"crypto/ed25519"
	"encoding/hex"
	"fmt"
)

func main() {
	// In production, this comes from a secrets manager or environment variable.
	// The seed is exactly 32 bytes.
	seedHex := "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"
	seedBytes, err := hex.DecodeString(seedHex)
	if err != nil {
		panic(fmt.Sprintf("invalid seed hex: %v", err))
	}

	if len(seedBytes) != ed25519.SeedSize {
		panic(fmt.Sprintf("seed must be %d bytes, got %d", ed25519.SeedSize, len(seedBytes)))
	}

	// Derive the key pair from the seed -- deterministic, same seed = same keys
	privateKey := ed25519.NewKeyFromSeed(seedBytes)
	publicKey := privateKey.Public().(ed25519.PublicKey)

	fmt.Printf("Public key: %s\n", hex.EncodeToString(publicKey))

	// The same seed always produces the same key pair.
	// This means you can store just the 32-byte seed and reconstruct the full
	// 64-byte private key at startup.
}
```

### Building the JWTIssuer

Now we combine Ed25519 with `golang-jwt/jwt/v5` to create a reusable JWT issuer:

```go
package auth

import (
	"crypto/ed25519"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// JWTIssuer handles creating and validating JWTs using Ed25519.
type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

// NewJWTIssuer creates a new JWTIssuer from a 32-byte Ed25519 seed.
// The seed should come from a secrets manager or secure environment variable.
func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) (*JWTIssuer, error) {
	if len(seed) != ed25519.SeedSize {
		return nil, fmt.Errorf("seed must be exactly %d bytes, got %d", ed25519.SeedSize, len(seed))
	}

	privateKey := ed25519.NewKeyFromSeed(seed)
	publicKey := privateKey.Public().(ed25519.PublicKey)

	return &JWTIssuer{
		privateKey: privateKey,
		publicKey:  publicKey,
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}, nil
}

// PublicKey returns the Ed25519 public key for external verification.
// Other services can use this to verify tokens without the private key.
func (j *JWTIssuer) PublicKey() ed25519.PublicKey {
	return j.publicKey
}

// AccessTTL returns the access token time-to-live.
func (j *JWTIssuer) AccessTTL() time.Duration {
	return j.accessTTL
}

// RefreshTTL returns the refresh token time-to-live.
func (j *JWTIssuer) RefreshTTL() time.Duration {
	return j.refreshTTL
}
```

**Node.js comparison -- there is no built-in Ed25519 JWT support in `jsonwebtoken`:**

```javascript
// Node.js: Ed25519 JWT signing requires a different library
// jsonwebtoken does NOT support EdDSA.
// You need jose (by panva) or similar:
const jose = require('jose');

const { privateKey, publicKey } = await jose.generateKeyPair('EdDSA');

const jwt = await new jose.SignJWT({ sub: 'user_123', email: 'alice@example.com' })
  .setProtectedHeader({ alg: 'EdDSA' })
  .setIssuer('my-api')
  .setExpirationTime('15m')
  .sign(privateKey);

// Go advantage: Ed25519 is in the standard library.
// Node.js requires third-party crypto libraries for EdDSA support.
```

---

## 4. Access & Refresh Token Pair

### Why Two Tokens?

A single long-lived token is a security risk: if stolen, the attacker has access for the entire token lifetime. A single short-lived token is a UX problem: the user has to re-authenticate every 15 minutes.

The solution is a **token pair:**

- **Access Token:** Short-lived (5-15 minutes). Used for every API request. Contains user claims. If stolen, the window of abuse is small.
- **Refresh Token:** Long-lived (7-30 days). Used only to obtain a new access token. Stored more securely. Can be revoked server-side.

```
┌─────────────────────────────────────────────────────────────┐
│                    Token Lifecycle                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Login ──► [Access Token: 15min] + [Refresh Token: 7 days] │
│                      │                       │              │
│            Use for API calls          Store securely        │
│                      │                       │              │
│            Access token expires               │              │
│                      │                       │              │
│              ┌───────▼───────┐               │              │
│              │ POST /refresh │◄──────────────┘              │
│              └───────┬───────┘                              │
│                      │                                      │
│            New [Access Token: 15min]                        │
│            (+ optionally rotate refresh token)              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Token Pair Structure

```go
package auth

import (
	"github.com/golang-jwt/jwt/v5"
)

// TokenPair holds an access token and a refresh token.
type TokenPair struct {
	AccessToken  string `json:"access_token"`
	RefreshToken string `json:"refresh_token"`
}

// AccessClaims represents the claims embedded in an access token.
// The access token is sent with every API request and contains
// user identity information that the server needs for authorization.
type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

// RefreshClaims represents the claims embedded in a refresh token.
// The refresh token contains minimal information -- just enough
// to identify the user and issue a new access token.
type RefreshClaims struct {
	jwt.RegisteredClaims
}
```

**Why `*string` for Name and Picture?**

Some identity providers do not return a name or profile picture. Using pointer types (`*string`) allows these fields to be `nil`, which serializes as either absent (with `omitempty`) or `null` in JSON. A bare `string` would serialize as `""`, which is ambiguous -- does the user have no name, or did we not receive one?

### Claims Design Decisions

**What goes in the access token:**
- User ID (`sub` claim) -- always
- Email -- for display and audit logging
- Name, picture -- for frontend display without extra API calls
- Roles/permissions -- for authorization decisions (Section 14)

**What does NOT go in the access token:**
- Passwords or password hashes -- obviously
- Full user profiles -- tokens should be small
- Sensitive PII beyond what is needed for authorization
- Anything that changes frequently -- token claims are fixed until expiry

**What goes in the refresh token:**
- User ID (`sub` claim) -- to identify who is refreshing
- A unique token ID (`jti` claim) -- for revocation (Section 11)
- Minimal other data -- the refresh token is not used for authorization

---

## 5. Issuing Tokens

### Complete Token Issuance

This is the core operation: given a user's identity, produce a signed token pair.

```go
package auth

import (
	"crypto/ed25519"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

// JWTIssuer handles creating and validating JWTs using Ed25519.
type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

// NewJWTIssuer creates a JWTIssuer from a 32-byte Ed25519 seed.
func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) (*JWTIssuer, error) {
	if len(seed) != ed25519.SeedSize {
		return nil, fmt.Errorf("seed must be exactly %d bytes, got %d", ed25519.SeedSize, len(seed))
	}

	privateKey := ed25519.NewKeyFromSeed(seed)
	publicKey := privateKey.Public().(ed25519.PublicKey)

	return &JWTIssuer{
		privateKey: privateKey,
		publicKey:  publicKey,
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}, nil
}

// IssueTokenPair creates a signed access token and refresh token for the given user.
func (j *JWTIssuer) IssueTokenPair(userID, email string, name, picture *string) (*TokenPair, error) {
	now := time.Now()

	// --- Access Token ---
	accessClaims := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
		},
		Email:   email,
		Name:    name,
		Picture: picture,
	}

	accessToken := jwt.NewWithClaims(jwt.SigningMethodEdDSA, accessClaims)
	signedAccess, err := accessToken.SignedString(j.privateKey)
	if err != nil {
		return nil, fmt.Errorf("signing access token: %w", err)
	}

	// --- Refresh Token ---
	refreshClaims := &RefreshClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.refreshTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
			ID:        uuid.New().String(), // Unique ID for revocation tracking
		},
	}

	refreshToken := jwt.NewWithClaims(jwt.SigningMethodEdDSA, refreshClaims)
	signedRefresh, err := refreshToken.SignedString(j.privateKey)
	if err != nil {
		return nil, fmt.Errorf("signing refresh token: %w", err)
	}

	return &TokenPair{
		AccessToken:  signedAccess,
		RefreshToken: signedRefresh,
	}, nil
}
```

### Complete Runnable Example

```go
package main

import (
	"crypto/ed25519"
	"encoding/hex"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

// --- Types ---

type TokenPair struct {
	AccessToken  string
	RefreshToken string
}

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

type RefreshClaims struct {
	jwt.RegisteredClaims
}

// --- JWTIssuer ---

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) (*JWTIssuer, error) {
	if len(seed) != ed25519.SeedSize {
		return nil, fmt.Errorf("seed must be %d bytes, got %d", ed25519.SeedSize, len(seed))
	}
	privateKey := ed25519.NewKeyFromSeed(seed)
	publicKey := privateKey.Public().(ed25519.PublicKey)
	return &JWTIssuer{
		privateKey: privateKey,
		publicKey:  publicKey,
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}, nil
}

func (j *JWTIssuer) IssueTokenPair(userID, email string, name, picture *string) (*TokenPair, error) {
	now := time.Now()

	accessClaims := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
		},
		Email:   email,
		Name:    name,
		Picture: picture,
	}

	accessToken := jwt.NewWithClaims(jwt.SigningMethodEdDSA, accessClaims)
	signedAccess, err := accessToken.SignedString(j.privateKey)
	if err != nil {
		return nil, fmt.Errorf("signing access token: %w", err)
	}

	refreshClaims := &RefreshClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.refreshTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
			ID:        uuid.New().String(),
		},
	}

	refreshToken := jwt.NewWithClaims(jwt.SigningMethodEdDSA, refreshClaims)
	signedRefresh, err := refreshToken.SignedString(j.privateKey)
	if err != nil {
		return nil, fmt.Errorf("signing refresh token: %w", err)
	}

	return &TokenPair{
		AccessToken:  signedAccess,
		RefreshToken: signedRefresh,
	}, nil
}

func main() {
	// Create a deterministic seed (in production, load from secrets manager)
	seedHex := "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"
	seed, _ := hex.DecodeString(seedHex)

	issuer, err := NewJWTIssuer(seed, "my-api", 15*time.Minute, 7*24*time.Hour)
	if err != nil {
		panic(err)
	}

	name := "Alice Johnson"
	picture := "https://example.com/alice.jpg"
	pair, err := issuer.IssueTokenPair("user_abc123", "alice@example.com", &name, &picture)
	if err != nil {
		panic(err)
	}

	fmt.Println("Access Token:")
	fmt.Println(pair.AccessToken)
	fmt.Println()
	fmt.Println("Refresh Token:")
	fmt.Println(pair.RefreshToken)

	// Access token is ~250 bytes (small thanks to Ed25519's 64-byte signatures)
	// Compare with RS256 tokens which are ~500+ bytes
	fmt.Printf("\nAccess token length:  %d bytes\n", len(pair.AccessToken))
	fmt.Printf("Refresh token length: %d bytes\n", len(pair.RefreshToken))
}
```

---

## 6. Validating Tokens

### Token Validation is Not Just Signature Checking

Validating a JWT requires multiple checks:

1. **Signature verification:** Was the token signed by our private key?
2. **Expiration check:** Is `exp` in the future?
3. **Not-before check:** Is `nbf` in the past?
4. **Issuer check:** Does `iss` match our expected issuer?
5. **Audience check:** Does `aud` include our expected audience?
6. **Algorithm check:** Was the correct signing algorithm used? (Prevents algorithm confusion attacks)

### The Algorithm Confusion Attack

One of the most dangerous JWT vulnerabilities is the **algorithm confusion attack.** If your server accepts tokens signed with `HS256` (HMAC) and also has an RSA public key, an attacker can:

1. Take your RSA **public** key (which is, well, public)
2. Sign a forged JWT using HMAC with the public key as the secret
3. Set `"alg": "HS256"` in the header
4. The server mistakenly uses the public key as the HMAC secret and the signature validates

**Prevention:** Always specify the expected signing algorithm when parsing. Never let the token's header dictate which algorithm to use.

### ValidateAccessToken Implementation

```go
package auth

import (
	"crypto/ed25519"
	"fmt"

	"github.com/golang-jwt/jwt/v5"
)

// ValidateAccessToken parses and validates an access token, returning the claims.
func (j *JWTIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}

	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		// CRITICAL: Verify the signing algorithm is what we expect.
		// This prevents algorithm confusion attacks.
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		// These options enforce claim validation:
		jwt.WithIssuer(j.issuer),                       // iss must match
		jwt.WithAudience(j.issuer),                     // aud must contain our issuer
		jwt.WithExpirationRequired(),                    // exp must be present
		jwt.WithIssuedAt(),                              // iat must be present
		jwt.WithValidMethods([]string{"EdDSA"}),         // Only accept EdDSA
	)

	if err != nil {
		return nil, fmt.Errorf("invalid access token: %w", err)
	}

	if !token.Valid {
		return nil, fmt.Errorf("token is not valid")
	}

	return claims, nil
}

// ValidateRefreshToken parses and validates a refresh token, returning the claims.
func (j *JWTIssuer) ValidateRefreshToken(tokenString string) (*RefreshClaims, error) {
	claims := &RefreshClaims{}

	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)

	if err != nil {
		return nil, fmt.Errorf("invalid refresh token: %w", err)
	}

	if !token.Valid {
		return nil, fmt.Errorf("token is not valid")
	}

	// Refresh tokens MUST have a JTI (JWT ID) for revocation tracking
	if claims.ID == "" {
		return nil, fmt.Errorf("refresh token missing jti claim")
	}

	return claims, nil
}
```

### Complete Validation Example

```go
package main

import (
	"crypto/ed25519"
	"encoding/hex"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

type RefreshClaims struct {
	jwt.RegisteredClaims
}

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) *JWTIssuer {
	privateKey := ed25519.NewKeyFromSeed(seed)
	publicKey := privateKey.Public().(ed25519.PublicKey)
	return &JWTIssuer{
		privateKey: privateKey,
		publicKey:  publicKey,
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}
}

func (j *JWTIssuer) IssueAccessToken(userID, email string, name *string) (string, error) {
	now := time.Now()
	claims := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
			ID:        uuid.New().String(),
		},
		Email: email,
		Name:  name,
	}
	token := jwt.NewWithClaims(jwt.SigningMethodEdDSA, claims)
	return token.SignedString(j.privateKey)
}

func (j *JWTIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, fmt.Errorf("invalid token: %w", err)
	}
	if !token.Valid {
		return nil, fmt.Errorf("token not valid")
	}
	return claims, nil
}

func main() {
	seedHex := "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"
	seed, _ := hex.DecodeString(seedHex)

	issuer := NewJWTIssuer(seed, "my-api", 15*time.Minute, 7*24*time.Hour)

	name := "Alice"
	tokenStr, err := issuer.IssueAccessToken("user_123", "alice@example.com", &name)
	if err != nil {
		panic(err)
	}
	fmt.Println("Token:", tokenStr[:50]+"...")

	// Validate the token
	claims, err := issuer.ValidateAccessToken(tokenStr)
	if err != nil {
		fmt.Println("Validation failed:", err)
		return
	}
	fmt.Println("Subject:", claims.Subject)
	fmt.Println("Email:  ", claims.Email)
	fmt.Println("Name:   ", *claims.Name)
	fmt.Println("Expires:", claims.ExpiresAt.Time)

	// Try validating a tampered token
	tampered := tokenStr[:len(tokenStr)-5] + "XXXXX"
	_, err = issuer.ValidateAccessToken(tampered)
	fmt.Println("\nTampered token error:", err)

	// Try validating with wrong issuer
	wrongIssuer := NewJWTIssuer(seed, "other-api", 15*time.Minute, 7*24*time.Hour)
	_, err = wrongIssuer.ValidateAccessToken(tokenStr)
	fmt.Println("Wrong issuer error: ", err)
}
```

### Validation with a Public-Key-Only Verifier

In microservice architectures, services that need to verify tokens should only have the public key:

```go
package auth

import (
	"crypto/ed25519"
	"fmt"

	"github.com/golang-jwt/jwt/v5"
)

// JWTVerifier can only verify tokens, not issue them.
// Distribute this to services that need to authenticate requests
// but should not be able to create tokens.
type JWTVerifier struct {
	publicKey ed25519.PublicKey
	issuer    string
}

// NewJWTVerifier creates a verifier from a public key.
func NewJWTVerifier(publicKey ed25519.PublicKey, issuer string) *JWTVerifier {
	return &JWTVerifier{
		publicKey: publicKey,
		issuer:    issuer,
	}
}

// VerifyAccessToken validates an access token using only the public key.
func (v *JWTVerifier) VerifyAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return v.publicKey, nil
	},
		jwt.WithIssuer(v.issuer),
		jwt.WithAudience(v.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, fmt.Errorf("verification failed: %w", err)
	}
	if !token.Valid {
		return nil, fmt.Errorf("token not valid")
	}
	return claims, nil
}
```

---

## 7. OIDC Token Validation

### What is OIDC Validation?

When your frontend authenticates with Google (or Apple, Microsoft, Auth0, etc.) using OAuth 2.0 + OIDC, it receives an **ID Token** -- a JWT signed by the identity provider. Your backend must validate this token to trust the user's identity.

Validating an external OIDC token is different from validating your own tokens:

- You do not have the identity provider's private key
- The provider publishes its public keys at a JWKS (JSON Web Key Set) endpoint
- Keys rotate periodically -- your server must fetch and cache them
- You must verify the `iss` (issuer) and `aud` (audience/client ID) claims

### OIDC Discovery

Every OIDC provider publishes a discovery document at `/.well-known/openid-configuration`:

```
https://accounts.google.com/.well-known/openid-configuration
```

This document contains:
- `issuer`: The provider's identifier (e.g., `https://accounts.google.com`)
- `jwks_uri`: URL to fetch the provider's public keys
- `authorization_endpoint`: Where to redirect for authentication
- `token_endpoint`: Where to exchange authorization codes for tokens
- `id_token_signing_alg_values_supported`: Which algorithms the provider uses

### Using go-oidc for Validation

The `github.com/coreos/go-oidc/v3/oidc` package handles the entire OIDC validation flow:

```go
package auth

import (
	"context"
	"fmt"

	"github.com/coreos/go-oidc/v3/oidc"
)

// OIDCValidator validates ID tokens from an external identity provider.
type OIDCValidator struct {
	verifier *oidc.IDTokenVerifier
	provider *oidc.Provider
}

// OIDCClaims represents the user info extracted from an OIDC ID token.
type OIDCClaims struct {
	Email         string `json:"email"`
	EmailVerified bool   `json:"email_verified"`
	Name          string `json:"name"`
	Picture       string `json:"picture"`
	Subject       string `json:"sub"` // The user's ID at the identity provider
}

// NewOIDCValidator creates a validator for the given OIDC provider.
// issuerURL is the provider's issuer URL (e.g., "https://accounts.google.com")
// clientID is your application's OAuth client ID.
func NewOIDCValidator(ctx context.Context, issuerURL, clientID string) (*OIDCValidator, error) {
	// NewProvider fetches the discovery document and JWKS automatically.
	// It caches the keys and refreshes them when they expire.
	provider, err := oidc.NewProvider(ctx, issuerURL)
	if err != nil {
		return nil, fmt.Errorf("creating OIDC provider for %s: %w", issuerURL, err)
	}

	verifier := provider.Verifier(&oidc.Config{
		ClientID: clientID,
	})

	return &OIDCValidator{
		verifier: verifier,
		provider: provider,
	}, nil
}

// Validate verifies an ID token and extracts the user claims.
func (v *OIDCValidator) Validate(ctx context.Context, rawToken string) (*OIDCClaims, error) {
	// Verify performs:
	// 1. Signature verification using the provider's JWKS
	// 2. Expiration check
	// 3. Issuer validation
	// 4. Audience validation (must contain our clientID)
	idToken, err := v.verifier.Verify(ctx, rawToken)
	if err != nil {
		return nil, fmt.Errorf("verifying ID token: %w", err)
	}

	// Extract claims from the verified token
	var claims OIDCClaims
	if err := idToken.Claims(&claims); err != nil {
		return nil, fmt.Errorf("extracting claims: %w", err)
	}

	// Validate required fields
	if claims.Email == "" {
		return nil, fmt.Errorf("ID token missing email claim")
	}

	if !claims.EmailVerified {
		return nil, fmt.Errorf("email not verified by identity provider")
	}

	return &claims, nil
}
```

### Supporting Multiple Identity Providers

Production systems often support multiple providers (Google, Apple, Microsoft). Use a registry pattern:

```go
package auth

import (
	"context"
	"fmt"
	"sync"
)

// ProviderConfig holds the configuration for a single OIDC provider.
type ProviderConfig struct {
	IssuerURL string
	ClientID  string
}

// OIDCRegistry manages multiple OIDC validators, one per identity provider.
type OIDCRegistry struct {
	validators map[string]*OIDCValidator
	mu         sync.RWMutex
}

// NewOIDCRegistry creates a registry and initializes validators for all providers.
func NewOIDCRegistry(ctx context.Context, providers map[string]ProviderConfig) (*OIDCRegistry, error) {
	registry := &OIDCRegistry{
		validators: make(map[string]*OIDCValidator),
	}

	for name, cfg := range providers {
		validator, err := NewOIDCValidator(ctx, cfg.IssuerURL, cfg.ClientID)
		if err != nil {
			return nil, fmt.Errorf("initializing provider %s: %w", name, err)
		}
		registry.validators[name] = validator
	}

	return registry, nil
}

// Validate validates a token against the specified provider.
func (r *OIDCRegistry) Validate(ctx context.Context, providerName, rawToken string) (*OIDCClaims, error) {
	r.mu.RLock()
	validator, ok := r.validators[providerName]
	r.mu.RUnlock()

	if !ok {
		return nil, fmt.Errorf("unknown identity provider: %s", providerName)
	}

	return validator.Validate(ctx, rawToken)
}

// Usage:
// registry, err := NewOIDCRegistry(ctx, map[string]ProviderConfig{
//     "google": {IssuerURL: "https://accounts.google.com", ClientID: "your-google-client-id"},
//     "apple":  {IssuerURL: "https://appleid.apple.com", ClientID: "your-apple-client-id"},
// })
//
// claims, err := registry.Validate(ctx, "google", rawIDToken)
```

### Node.js Comparison

```javascript
// Node.js OIDC validation with google-auth-library
const { OAuth2Client } = require('google-auth-library');

const client = new OAuth2Client('your-google-client-id');

async function verifyGoogleToken(idToken) {
  const ticket = await client.verifyIdToken({
    idToken,
    audience: 'your-google-client-id',
  });
  const payload = ticket.getPayload();
  return {
    email: payload.email,
    emailVerified: payload.email_verified,
    name: payload.name,
    picture: payload.picture,
    subject: payload.sub,
  };
}

// Or with the generic openid-client library:
const { Issuer } = require('openid-client');

const googleIssuer = await Issuer.discover('https://accounts.google.com');
const client = new googleIssuer.Client({ client_id: 'your-id' });
// ...

// Go's go-oidc package is the direct equivalent of openid-client.
// Both handle discovery, key rotation, and validation automatically.
```

---

## 8. Token Exchange Flow

### The Pattern

The token exchange flow is the bridge between "user authenticated with Google" and "user has a session with your API." Here is the complete flow:

```
Frontend                        Your Go Backend                     Google
   |                                  |                                |
   |── (1) OAuth flow ───────────────────────────────────────────────►|
   |                                  |                                |
   |◄── (2) ID Token (JWT) ──────────────────────────────────────────|
   |                                  |                                |
   |── (3) POST /auth/token-exchange ►|                                |
   |   Body: { provider: "google",    |                                |
   |           id_token: "eyJ..." }   |                                |
   |                                  |── (4) Validate ID token ──────►|
   |                                  |   (fetch JWKS, verify sig)     |
   |                                  |◄── (5) Valid claims ──────────|
   |                                  |                                |
   |                                  |── (6) Find or create user     |
   |                                  |   in your database             |
   |                                  |                                |
   |                                  |── (7) Issue your own JWT pair |
   |                                  |                                |
   |◄── (8) Set-Cookie: access_token  |                                |
   |    Set-Cookie: refresh_token     |                                |
   |                                  |                                |
   |── (9) GET /api/me ──────────────►|                                |
   |   Cookie: access_token=eyJ...    |                                |
   |                                  |── (10) Validate YOUR token    |
   |◄── (11) { user data } ──────────|                                |
```

### Token Exchange Handler

```go
package main

import (
	"context"
	"crypto/ed25519"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/coreos/go-oidc/v3/oidc"
	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

// --- Configuration ---

type AuthConfig struct {
	JWTSeedHex   string
	JWTIssuer    string
	AccessTTL    time.Duration
	RefreshTTL   time.Duration
	CookieDomain string
	CookieSecure bool
	GoogleClient string
}

// --- Token Exchange Request/Response ---

type TokenExchangeRequest struct {
	Provider string `json:"provider"` // "google", "apple", etc.
	IDToken  string `json:"id_token"`
}

type TokenExchangeResponse struct {
	User UserInfo `json:"user"`
}

type UserInfo struct {
	ID      string  `json:"id"`
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

// --- Auth Service ---

type AuthService struct {
	jwtIssuer     *JWTIssuer
	oidcValidator *OIDCValidator
	userRepo      UserRepository
	tokenRepo     RefreshTokenRepository
	config        AuthConfig
}

// UserRepository defines the interface for user persistence.
type UserRepository interface {
	FindByEmail(ctx context.Context, email string) (*User, error)
	Create(ctx context.Context, user *User) error
	UpdateLastLogin(ctx context.Context, userID string) error
}

// RefreshTokenRepository defines the interface for refresh token persistence.
type RefreshTokenRepository interface {
	Store(ctx context.Context, tokenID, userID string, hashedToken []byte, expiresAt time.Time) error
	Verify(ctx context.Context, tokenID string, hashedToken []byte) (bool, error)
	Revoke(ctx context.Context, tokenID string) error
	RevokeAllForUser(ctx context.Context, userID string) error
}

type User struct {
	ID        string
	Email     string
	Name      *string
	Picture   *string
	CreatedAt time.Time
	LastLogin time.Time
}

// HandleTokenExchange handles the POST /auth/token-exchange endpoint.
func (s *AuthService) HandleTokenExchange(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	// Parse request body
	var req TokenExchangeRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{
			"error": "invalid request body",
		})
		return
	}

	// Validate required fields
	if req.Provider == "" || req.IDToken == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{
			"error": "provider and id_token are required",
		})
		return
	}

	ctx := r.Context()

	// Step 1: Validate the identity provider's ID token
	oidcClaims, err := s.oidcValidator.Validate(ctx, req.IDToken)
	if err != nil {
		log.Printf("OIDC validation failed for provider %s: %v", req.Provider, err)
		writeJSON(w, http.StatusUnauthorized, map[string]string{
			"error": "invalid identity token",
		})
		return
	}

	// Step 2: Find or create the user in our database
	user, err := s.userRepo.FindByEmail(ctx, oidcClaims.Email)
	if err != nil {
		// User does not exist -- create them
		user = &User{
			ID:        uuid.New().String(),
			Email:     oidcClaims.Email,
			CreatedAt: time.Now(),
		}
		if oidcClaims.Name != "" {
			user.Name = &oidcClaims.Name
		}
		if oidcClaims.Picture != "" {
			user.Picture = &oidcClaims.Picture
		}
		if err := s.userRepo.Create(ctx, user); err != nil {
			log.Printf("Failed to create user: %v", err)
			writeJSON(w, http.StatusInternalServerError, map[string]string{
				"error": "internal server error",
			})
			return
		}
	} else {
		// User exists -- update last login
		_ = s.userRepo.UpdateLastLogin(ctx, user.ID)
	}

	// Step 3: Issue our own JWT pair
	pair, err := s.jwtIssuer.IssueTokenPair(user.ID, user.Email, user.Name, user.Picture)
	if err != nil {
		log.Printf("Failed to issue tokens: %v", err)
		writeJSON(w, http.StatusInternalServerError, map[string]string{
			"error": "internal server error",
		})
		return
	}

	// Step 4: Store the refresh token hash in the database
	// (See Section 11 for the full implementation)

	// Step 5: Set tokens as httpOnly cookies
	setTokenCookies(w, pair, s.config)

	// Step 6: Return user info
	writeJSON(w, http.StatusOK, TokenExchangeResponse{
		User: UserInfo{
			ID:      user.ID,
			Email:   user.Email,
			Name:    user.Name,
			Picture: user.Picture,
		},
	})
}

// setTokenCookies sets access and refresh tokens as httpOnly cookies.
func setTokenCookies(w http.ResponseWriter, pair *TokenPair, cfg AuthConfig) {
	http.SetCookie(w, &http.Cookie{
		Name:     "access_token",
		Value:    pair.AccessToken,
		MaxAge:   int(cfg.AccessTTL.Seconds()),
		HttpOnly: true,
		Secure:   cfg.CookieSecure,
		SameSite: http.SameSiteLaxMode,
		Path:     "/",
		Domain:   cfg.CookieDomain,
	})

	http.SetCookie(w, &http.Cookie{
		Name:     "refresh_token",
		Value:    pair.RefreshToken,
		MaxAge:   int(cfg.RefreshTTL.Seconds()),
		HttpOnly: true,
		Secure:   cfg.CookieSecure,
		SameSite: http.SameSiteLaxMode,
		Path:     "/auth", // Only sent to auth endpoints
		Domain:   cfg.CookieDomain,
	})
}

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

// --- Placeholder types for compilation (full implementations in other sections) ---

type TokenPair struct {
	AccessToken  string
	RefreshToken string
}

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

type RefreshClaims struct {
	jwt.RegisteredClaims
}

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) *JWTIssuer {
	pk := ed25519.NewKeyFromSeed(seed)
	return &JWTIssuer{
		privateKey: pk,
		publicKey:  pk.Public().(ed25519.PublicKey),
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}
}

func (j *JWTIssuer) IssueTokenPair(userID, email string, name, picture *string) (*TokenPair, error) {
	now := time.Now()
	ac := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject: userID, Issuer: j.issuer, Audience: jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt: jwt.NewNumericDate(now), NotBefore: jwt.NewNumericDate(now),
		},
		Email: email, Name: name, Picture: picture,
	}
	at := jwt.NewWithClaims(jwt.SigningMethodEdDSA, ac)
	sa, err := at.SignedString(j.privateKey)
	if err != nil {
		return nil, err
	}
	rc := &RefreshClaims{RegisteredClaims: jwt.RegisteredClaims{
		Subject: userID, Issuer: j.issuer, Audience: jwt.ClaimStrings{j.issuer},
		ExpiresAt: jwt.NewNumericDate(now.Add(j.refreshTTL)),
		IssuedAt: jwt.NewNumericDate(now), NotBefore: jwt.NewNumericDate(now),
		ID: uuid.New().String(),
	}}
	rt := jwt.NewWithClaims(jwt.SigningMethodEdDSA, rc)
	sr, err := rt.SignedString(j.privateKey)
	if err != nil {
		return nil, err
	}
	return &TokenPair{AccessToken: sa, RefreshToken: sr}, nil
}

type OIDCValidator struct{}
type OIDCClaims struct {
	Email         string
	EmailVerified bool
	Name          string
	Picture       string
}

func (v *OIDCValidator) Validate(ctx context.Context, rawToken string) (*OIDCClaims, error) {
	// In production, this uses go-oidc as shown in Section 7
	return nil, fmt.Errorf("not implemented in this example")
}

func main() {
	seedHex := "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"
	seed, _ := hex.DecodeString(seedHex)

	cfg := AuthConfig{
		JWTIssuer:    "my-api",
		AccessTTL:    15 * time.Minute,
		RefreshTTL:   7 * 24 * time.Hour,
		CookieDomain: "localhost",
		CookieSecure: false, // true in production
		GoogleClient: "your-google-client-id",
	}

	jwtIss := NewJWTIssuer(seed, cfg.JWTIssuer, cfg.AccessTTL, cfg.RefreshTTL)

	service := &AuthService{
		jwtIssuer:     jwtIss,
		oidcValidator: &OIDCValidator{},
		config:        cfg,
	}

	mux := http.NewServeMux()
	mux.HandleFunc("/auth/token-exchange", service.HandleTokenExchange)

	fmt.Println("Auth server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

---

## 9. Cookie-Based Token Delivery

### Why Cookies Instead of localStorage?

This is one of the most consequential security decisions in web authentication:

| Storage Method | XSS Vulnerable | CSRF Vulnerable | httpOnly |
|----------------|---------------|-----------------|----------|
| localStorage | **YES** -- any JS can read it | No | No |
| sessionStorage | **YES** -- any JS can read it | No | No |
| httpOnly Cookie | **No** -- JS cannot access it | Yes (mitigated) | **YES** |
| In-memory (JS variable) | **YES** -- if XSS can run code | No | No |

**The critical insight:** If an attacker injects JavaScript into your page (XSS), they can steal any token stored in localStorage or sessionStorage. An `httpOnly` cookie is invisible to JavaScript -- even injected JavaScript cannot read it. The browser automatically sends it with every request.

CSRF (Cross-Site Request Forgery) is a concern with cookies, but it is mitigable with `SameSite` attributes and CSRF tokens. XSS is much harder to fully prevent, making httpOnly cookies the superior choice.

### Cookie Configuration

```go
package main

import (
	"net/http"
	"time"
)

// CookieConfig holds cookie configuration that varies by environment.
type CookieConfig struct {
	Domain   string // e.g., ".example.com" or "localhost"
	Secure   bool   // true in production (HTTPS only)
	SameSite http.SameSite
}

// ProductionCookieConfig returns secure cookie settings for production.
func ProductionCookieConfig(domain string) CookieConfig {
	return CookieConfig{
		Domain:   domain,
		Secure:   true,               // HTTPS only
		SameSite: http.SameSiteLaxMode, // Allows top-level navigation
	}
}

// DevelopmentCookieConfig returns permissive cookie settings for local development.
func DevelopmentCookieConfig() CookieConfig {
	return CookieConfig{
		Domain:   "localhost",
		Secure:   false,               // HTTP allowed
		SameSite: http.SameSiteLaxMode,
	}
}

// SetAccessTokenCookie sets the access token as an httpOnly cookie.
func SetAccessTokenCookie(w http.ResponseWriter, token string, ttl time.Duration, cfg CookieConfig) {
	http.SetCookie(w, &http.Cookie{
		Name:     "access_token",
		Value:    token,
		MaxAge:   int(ttl.Seconds()),
		HttpOnly: true,                   // NOT accessible by JavaScript
		Secure:   cfg.Secure,             // Only sent over HTTPS in production
		SameSite: cfg.SameSite,           // CSRF protection
		Path:     "/",                    // Sent with ALL requests
		Domain:   cfg.Domain,
	})
}

// SetRefreshTokenCookie sets the refresh token as an httpOnly cookie.
// The Path is restricted to /auth so the refresh token is only sent
// to authentication endpoints, reducing exposure.
func SetRefreshTokenCookie(w http.ResponseWriter, token string, ttl time.Duration, cfg CookieConfig) {
	http.SetCookie(w, &http.Cookie{
		Name:     "refresh_token",
		Value:    token,
		MaxAge:   int(ttl.Seconds()),
		HttpOnly: true,
		Secure:   cfg.Secure,
		SameSite: cfg.SameSite,
		Path:     "/auth",               // Only sent to auth endpoints
		Domain:   cfg.Domain,
	})
}

// ClearAuthCookies removes both auth cookies by setting MaxAge to -1.
// Used during logout.
func ClearAuthCookies(w http.ResponseWriter, cfg CookieConfig) {
	http.SetCookie(w, &http.Cookie{
		Name:     "access_token",
		Value:    "",
		MaxAge:   -1,                    // Delete immediately
		HttpOnly: true,
		Secure:   cfg.Secure,
		SameSite: cfg.SameSite,
		Path:     "/",
		Domain:   cfg.Domain,
	})

	http.SetCookie(w, &http.Cookie{
		Name:     "refresh_token",
		Value:    "",
		MaxAge:   -1,
		HttpOnly: true,
		Secure:   cfg.Secure,
		SameSite: cfg.SameSite,
		Path:     "/auth",
		Domain:   cfg.Domain,
	})
}

func main() {
	cfg := DevelopmentCookieConfig()

	http.HandleFunc("/auth/login", func(w http.ResponseWriter, r *http.Request) {
		// After successful authentication...
		accessToken := "eyJ..." // issued by JWTIssuer
		refreshToken := "eyJ..."

		SetAccessTokenCookie(w, accessToken, 15*time.Minute, cfg)
		SetRefreshTokenCookie(w, refreshToken, 7*24*time.Hour, cfg)

		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"message":"logged in"}`))
	})

	http.HandleFunc("/auth/logout", func(w http.ResponseWriter, r *http.Request) {
		ClearAuthCookies(w, cfg)
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"message":"logged out"}`))
	})

	http.ListenAndServe(":8080", nil)
}
```

### SameSite Cookie Attribute Deep Dive

The `SameSite` attribute controls when cookies are sent with cross-origin requests:

```
+------------------+------------------------------------------+-------------------+
| SameSite Value   | Behavior                                 | CSRF Protection   |
+------------------+------------------------------------------+-------------------+
| Strict           | Only sent on same-site requests.         | Maximum           |
|                  | NOT sent when navigating from external   |                   |
|                  | sites (e.g., clicking a link in email)   |                   |
+------------------+------------------------------------------+-------------------+
| Lax (recommended)| Sent on same-site requests AND           | Good              |
|                  | top-level navigations (GET only).        |                   |
|                  | NOT sent on cross-site POST/AJAX.        |                   |
+------------------+------------------------------------------+-------------------+
| None             | Always sent (requires Secure flag).      | None (use CSRF    |
|                  | Needed for cross-domain auth.            | tokens instead)   |
+------------------+------------------------------------------+-------------------+
```

**Recommendation:** Use `SameSite=Lax` for most applications. It prevents CSRF on POST requests while still allowing users to navigate to your site from external links without losing their session.

### Node.js Comparison

```javascript
// Express.js cookie-based auth
const express = require('express');
const cookieParser = require('cookie-parser');

const app = express();
app.use(cookieParser());

app.post('/auth/login', (req, res) => {
  const accessToken = generateJWT(user);
  const refreshToken = generateRefreshJWT(user);

  res.cookie('access_token', accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 15 * 60 * 1000, // 15 minutes in ms
    path: '/',
    domain: process.env.COOKIE_DOMAIN,
  });

  res.cookie('refresh_token', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days in ms
    path: '/auth',
    domain: process.env.COOKIE_DOMAIN,
  });

  res.json({ user });
});

// Go advantage: http.SetCookie is in the standard library.
// Go's MaxAge is in seconds (simpler), Node's is in milliseconds.
// Go's SameSite is a typed constant, Node's is a string (typo-prone).
```

---

## 10. Token Refresh Endpoint

### How Token Refresh Works

When the access token expires (after 15 minutes), the client receives a 401 Unauthorized response. Instead of forcing the user to log in again, the client sends the refresh token to get a new access token:

```
Client                          Server
  |                                |
  |── GET /api/data ──────────────►|
  |   Cookie: access_token (expired)|
  |                                |── Validate access token
  |◄── 401 Unauthorized ──────────|   ← EXPIRED
  |                                |
  |── POST /auth/refresh ─────────►|
  |   Cookie: refresh_token         |
  |                                |── Validate refresh token
  |                                |── Check refresh token in DB
  |                                |── Issue new access token
  |                                |── (Optionally rotate refresh token)
  |◄── Set-Cookie: access_token ───|
  |                                |
  |── GET /api/data ──────────────►|
  |   Cookie: access_token (new)    |
  |◄── 200 OK (data) ─────────────|
```

### Refresh Handler Implementation

```go
package main

import (
	"context"
	"crypto/ed25519"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

// --- Types (repeated for completeness) ---

type TokenPair struct {
	AccessToken  string
	RefreshToken string
}

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

type RefreshClaims struct {
	jwt.RegisteredClaims
}

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

type CookieConfig struct {
	Domain   string
	Secure   bool
	SameSite http.SameSite
}

type UserRepository interface {
	FindByID(ctx context.Context, id string) (*User, error)
}

type RefreshTokenRepository interface {
	Store(ctx context.Context, tokenID, userID string, hashedToken []byte, expiresAt time.Time) error
	Verify(ctx context.Context, tokenID string, hashedToken []byte) (bool, error)
	Revoke(ctx context.Context, tokenID string) error
	RevokeAllForUser(ctx context.Context, userID string) error
}

type User struct {
	ID      string
	Email   string
	Name    *string
	Picture *string
}

// --- Refresh Handler ---

type RefreshHandler struct {
	jwtIssuer *JWTIssuer
	userRepo  UserRepository
	tokenRepo RefreshTokenRepository
	cookieCfg CookieConfig
}

// HandleRefresh handles POST /auth/refresh.
// It validates the refresh token, issues a new access token,
// and optionally rotates the refresh token.
func (h *RefreshHandler) HandleRefresh(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	ctx := r.Context()

	// Step 1: Extract refresh token from cookie
	cookie, err := r.Cookie("refresh_token")
	if err != nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{
			"error": "no refresh token",
		})
		return
	}

	rawRefreshToken := cookie.Value

	// Step 2: Validate the refresh token's signature and claims
	refreshClaims, err := h.jwtIssuer.ValidateRefreshToken(rawRefreshToken)
	if err != nil {
		log.Printf("Invalid refresh token: %v", err)
		writeJSON(w, http.StatusUnauthorized, map[string]string{
			"error": "invalid refresh token",
		})
		return
	}

	// Step 3: Verify the refresh token exists in our database (not revoked)
	hashedToken := hashToken(rawRefreshToken)
	valid, err := h.tokenRepo.Verify(ctx, refreshClaims.ID, hashedToken)
	if err != nil || !valid {
		log.Printf("Refresh token not found in database or revoked: jti=%s", refreshClaims.ID)

		// SECURITY: If a refresh token is used but not found in the DB,
		// it may have been stolen and already rotated. Revoke ALL tokens
		// for this user as a precaution (refresh token reuse detection).
		_ = h.tokenRepo.RevokeAllForUser(ctx, refreshClaims.Subject)

		writeJSON(w, http.StatusUnauthorized, map[string]string{
			"error": "refresh token revoked",
		})
		return
	}

	// Step 4: Fetch current user data (to get up-to-date name, picture, etc.)
	user, err := h.userRepo.FindByID(ctx, refreshClaims.Subject)
	if err != nil {
		log.Printf("User not found during refresh: %v", err)
		writeJSON(w, http.StatusUnauthorized, map[string]string{
			"error": "user not found",
		})
		return
	}

	// Step 5: Revoke the old refresh token (one-time use / rotation)
	if err := h.tokenRepo.Revoke(ctx, refreshClaims.ID); err != nil {
		log.Printf("Failed to revoke old refresh token: %v", err)
		// Continue anyway -- better to issue new tokens than to lock the user out
	}

	// Step 6: Issue a new token pair
	pair, err := h.jwtIssuer.IssueTokenPair(user.ID, user.Email, user.Name, user.Picture)
	if err != nil {
		log.Printf("Failed to issue new tokens: %v", err)
		writeJSON(w, http.StatusInternalServerError, map[string]string{
			"error": "internal server error",
		})
		return
	}

	// Step 7: Store the new refresh token hash in the database
	newRefreshClaims, _ := h.jwtIssuer.ValidateRefreshToken(pair.RefreshToken)
	newHash := hashToken(pair.RefreshToken)
	expiresAt := newRefreshClaims.ExpiresAt.Time

	if err := h.tokenRepo.Store(ctx, newRefreshClaims.ID, user.ID, newHash, expiresAt); err != nil {
		log.Printf("Failed to store new refresh token: %v", err)
	}

	// Step 8: Set new cookies
	setTokenCookies(w, pair, h.cookieCfg, h.jwtIssuer.accessTTL, h.jwtIssuer.refreshTTL)

	writeJSON(w, http.StatusOK, map[string]string{
		"message": "tokens refreshed",
	})
}

// hashToken creates a SHA-256 hash of a token string.
// We store hashes, not raw tokens, in the database.
func hashToken(token string) []byte {
	h := sha256.Sum256([]byte(token))
	return h[:]
}

func setTokenCookies(w http.ResponseWriter, pair *TokenPair, cfg CookieConfig, accessTTL, refreshTTL time.Duration) {
	http.SetCookie(w, &http.Cookie{
		Name:     "access_token",
		Value:    pair.AccessToken,
		MaxAge:   int(accessTTL.Seconds()),
		HttpOnly: true,
		Secure:   cfg.Secure,
		SameSite: cfg.SameSite,
		Path:     "/",
		Domain:   cfg.Domain,
	})
	http.SetCookie(w, &http.Cookie{
		Name:     "refresh_token",
		Value:    pair.RefreshToken,
		MaxAge:   int(refreshTTL.Seconds()),
		HttpOnly: true,
		Secure:   cfg.Secure,
		SameSite: cfg.SameSite,
		Path:     "/auth",
		Domain:   cfg.Domain,
	})
}

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

// ValidateRefreshToken validates and parses a refresh token.
func (j *JWTIssuer) ValidateRefreshToken(tokenString string) (*RefreshClaims, error) {
	claims := &RefreshClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, fmt.Errorf("token not valid")
	}
	if claims.ID == "" {
		return nil, fmt.Errorf("missing jti")
	}
	return claims, nil
}

func main() {
	seedHex := "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"
	seed, _ := hex.DecodeString(seedHex)

	jwtIss := &JWTIssuer{
		privateKey: ed25519.NewKeyFromSeed(seed),
		issuer:     "my-api",
		accessTTL:  15 * time.Minute,
		refreshTTL: 7 * 24 * time.Hour,
	}
	jwtIss.publicKey = jwtIss.privateKey.Public().(ed25519.PublicKey)

	handler := &RefreshHandler{
		jwtIssuer: jwtIss,
		cookieCfg: CookieConfig{
			Domain:   "localhost",
			Secure:   false,
			SameSite: http.SameSiteLaxMode,
		},
		// userRepo and tokenRepo would be real implementations
	}

	mux := http.NewServeMux()
	mux.HandleFunc("/auth/refresh", handler.HandleRefresh)

	fmt.Println("Refresh server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Refresh Token Rotation

**Rotation** means that every time a refresh token is used, it is invalidated and a new one is issued. This is a critical security measure:

- If an attacker steals a refresh token and uses it, the legitimate user's next refresh attempt will fail (the old token was already consumed)
- When the server detects reuse of an already-consumed refresh token, it revokes ALL tokens for that user
- This is known as **automatic reuse detection**

```
Without Rotation:
  Attacker steals refresh token → Can refresh indefinitely for 7 days

With Rotation:
  Attacker steals refresh token → Uses it → Gets new tokens
  Legitimate user tries to refresh → Old token rejected → ALL tokens revoked
  Attacker's new tokens also become useless
```

---

## 11. Refresh Token Storage

### Why Store Refresh Tokens?

Access tokens are stateless -- the server validates them using only the signature and claims. Refresh tokens need server-side storage because:

1. **Revocation:** You need to invalidate specific refresh tokens (e.g., when a user logs out)
2. **Rotation detection:** You need to know if a refresh token has already been used
3. **Audit trail:** You want to know which devices/sessions are active

### Never Store Raw Tokens

Just like passwords, refresh tokens must be **hashed** before storage. If the database is compromised, raw tokens would let an attacker impersonate any user. Hashed tokens are useless without the original.

We use SHA-256 for hashing refresh tokens (not bcrypt). Why?

- Refresh tokens are long, random strings with high entropy -- not guessable passwords
- SHA-256 is a one-way hash that is fast and deterministic
- bcrypt is designed for low-entropy passwords and is intentionally slow -- unnecessary here

### Database Schema

```sql
CREATE TABLE refresh_tokens (
    id          TEXT PRIMARY KEY,          -- JWT ID (jti claim)
    user_id     TEXT NOT NULL,             -- Foreign key to users table
    token_hash  BYTEA NOT NULL,           -- SHA-256 hash of the refresh token
    expires_at  TIMESTAMP NOT NULL,        -- When the token expires
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    revoked_at  TIMESTAMP,                -- NULL if active, set when revoked

    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at);
```

### Repository Implementation

```go
package auth

import (
	"context"
	"crypto/sha256"
	"database/sql"
	"fmt"
	"time"
)

// RefreshTokenRepo implements refresh token storage using a SQL database.
type RefreshTokenRepo struct {
	db *sql.DB
}

// NewRefreshTokenRepo creates a new repository.
func NewRefreshTokenRepo(db *sql.DB) *RefreshTokenRepo {
	return &RefreshTokenRepo{db: db}
}

// HashToken computes the SHA-256 hash of a raw token string.
func HashToken(rawToken string) []byte {
	h := sha256.Sum256([]byte(rawToken))
	return h[:]
}

// Store saves a hashed refresh token in the database.
func (r *RefreshTokenRepo) Store(ctx context.Context, tokenID, userID string, hashedToken []byte, expiresAt time.Time) error {
	query := `
		INSERT INTO refresh_tokens (id, user_id, token_hash, expires_at, created_at)
		VALUES ($1, $2, $3, $4, NOW())
	`
	_, err := r.db.ExecContext(ctx, query, tokenID, userID, hashedToken, expiresAt)
	if err != nil {
		return fmt.Errorf("storing refresh token: %w", err)
	}
	return nil
}

// Verify checks that a refresh token exists, is not revoked, and matches the hash.
func (r *RefreshTokenRepo) Verify(ctx context.Context, tokenID string, hashedToken []byte) (bool, error) {
	query := `
		SELECT token_hash FROM refresh_tokens
		WHERE id = $1
		  AND revoked_at IS NULL
		  AND expires_at > NOW()
	`

	var storedHash []byte
	err := r.db.QueryRowContext(ctx, query, tokenID).Scan(&storedHash)
	if err == sql.ErrNoRows {
		return false, nil // Token not found or revoked
	}
	if err != nil {
		return false, fmt.Errorf("querying refresh token: %w", err)
	}

	// Constant-time comparison to prevent timing attacks
	if len(storedHash) != len(hashedToken) {
		return false, nil
	}
	match := true
	for i := range storedHash {
		if storedHash[i] != hashedToken[i] {
			match = false
			// Do NOT break early -- constant time
		}
	}
	return match, nil
}

// Revoke marks a specific refresh token as revoked.
func (r *RefreshTokenRepo) Revoke(ctx context.Context, tokenID string) error {
	query := `
		UPDATE refresh_tokens SET revoked_at = NOW()
		WHERE id = $1 AND revoked_at IS NULL
	`
	_, err := r.db.ExecContext(ctx, query, tokenID)
	if err != nil {
		return fmt.Errorf("revoking refresh token: %w", err)
	}
	return nil
}

// RevokeAllForUser revokes all refresh tokens for a user.
// Used when suspicious activity is detected (refresh token reuse).
func (r *RefreshTokenRepo) RevokeAllForUser(ctx context.Context, userID string) error {
	query := `
		UPDATE refresh_tokens SET revoked_at = NOW()
		WHERE user_id = $1 AND revoked_at IS NULL
	`
	result, err := r.db.ExecContext(ctx, query, userID)
	if err != nil {
		return fmt.Errorf("revoking all tokens for user: %w", err)
	}
	count, _ := result.RowsAffected()
	if count > 0 {
		fmt.Printf("SECURITY: Revoked %d refresh tokens for user %s\n", count, userID)
	}
	return nil
}

// DeleteExpired removes expired tokens from the database.
// Run periodically as a background cleanup task.
func (r *RefreshTokenRepo) DeleteExpired(ctx context.Context) (int64, error) {
	query := `DELETE FROM refresh_tokens WHERE expires_at < NOW()`
	result, err := r.db.ExecContext(ctx, query)
	if err != nil {
		return 0, fmt.Errorf("deleting expired tokens: %w", err)
	}
	return result.RowsAffected()
}
```

### Background Cleanup Goroutine

Expired tokens accumulate in the database. A background goroutine periodically cleans them up:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"
)

// RefreshTokenRepository is the interface for token storage.
type RefreshTokenRepository interface {
	DeleteExpired(ctx context.Context) (int64, error)
}

// StartTokenCleanup launches a background goroutine that periodically
// deletes expired refresh tokens from the database.
// It respects context cancellation for graceful shutdown.
func StartTokenCleanup(ctx context.Context, repo RefreshTokenRepository, interval time.Duration) {
	go func() {
		ticker := time.NewTicker(interval)
		defer ticker.Stop()

		// Run once at startup
		deleted, err := repo.DeleteExpired(ctx)
		if err != nil {
			log.Printf("Token cleanup error: %v", err)
		} else if deleted > 0 {
			log.Printf("Token cleanup: deleted %d expired tokens", deleted)
		}

		for {
			select {
			case <-ctx.Done():
				log.Println("Token cleanup goroutine shutting down")
				return
			case <-ticker.C:
				deleted, err := repo.DeleteExpired(ctx)
				if err != nil {
					log.Printf("Token cleanup error: %v", err)
					continue
				}
				if deleted > 0 {
					log.Printf("Token cleanup: deleted %d expired tokens", deleted)
				}
			}
		}
	}()
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// In production, repo would be a real RefreshTokenRepo connected to your database.
	// var repo RefreshTokenRepository = auth.NewRefreshTokenRepo(db)
	// StartTokenCleanup(ctx, repo, 1*time.Hour)

	fmt.Println("Token cleanup goroutine started")
	fmt.Println("Press Ctrl+C to stop")

	// Block until context is cancelled (in a real server, this would be
	// your HTTP server's ListenAndServe call)
	<-ctx.Done()
}
```

### Using `crypto/subtle` for Constant-Time Comparison

The `Verify` method above does manual constant-time comparison. In production, use the standard library:

```go
package main

import (
	"crypto/sha256"
	"crypto/subtle"
	"fmt"
)

func main() {
	token := "eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9..."

	// Hash the token
	hash := sha256.Sum256([]byte(token))
	storedHash := hash[:] // Simulating what is stored in DB

	// Later, when verifying:
	incomingHash := sha256.Sum256([]byte(token))

	// Constant-time comparison prevents timing attacks.
	// A timing attack could reveal the correct hash byte by byte
	// by measuring response time differences.
	match := subtle.ConstantTimeCompare(storedHash, incomingHash[:])
	fmt.Println("Match:", match == 1) // true
}
```

---

## 12. Auth Middleware

### Middleware Pattern Review

Go HTTP middleware wraps a handler, performing actions before and/or after the inner handler executes. For authentication, the middleware:

1. Extracts the token from the request (cookie or Authorization header)
2. Validates the token
3. Injects the user claims into the request context
4. Calls the next handler (or returns 401)

```
Request ──► [Auth Middleware] ──► [Route Handler]
                │                       │
                ├─ Extract token         ├─ Access ctx.Value(UserClaimsKey)
                ├─ Validate              │
                ├─ Inject claims         │
                │  into context          │
                │                       │
                └─ OR return 401         │
```

### Context Keys

```go
package auth

// contextKey is an unexported type for context keys in this package.
// Using a custom type prevents collisions with keys from other packages.
type contextKey string

const (
	// UserClaimsKey is the context key for storing user claims.
	UserClaimsKey contextKey = "user_claims"
)

// UserClaims represents the authenticated user's information
// extracted from a validated JWT.
type UserClaims struct {
	UserID  string
	Email   string
	Name    *string
	Picture *string
	Roles   []string // For role-based authorization (Section 14)
}
```

### Auth Middleware Implementation

```go
package main

import (
	"context"
	"crypto/ed25519"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// --- Context Key ---

type contextKey string

const UserClaimsKey contextKey = "user_claims"

// --- User Claims ---

type UserClaims struct {
	UserID  string
	Email   string
	Name    *string
	Picture *string
	Roles   []string
}

// --- JWT Types ---

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string   `json:"email"`
	Name    *string  `json:"name,omitempty"`
	Picture *string  `json:"picture,omitempty"`
	Roles   []string `json:"roles,omitempty"`
}

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func (j *JWTIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, fmt.Errorf("invalid token")
	}
	return claims, nil
}

// --- Auth Middleware ---

// AuthMiddleware validates the access token and injects user claims into the context.
// It checks for tokens in this order:
// 1. Cookie named "access_token"
// 2. Authorization header with "Bearer " prefix
func AuthMiddleware(jwtIssuer *JWTIssuer) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Step 1: Extract the token
			tokenString := extractToken(r)
			if tokenString == "" {
				writeJSON(w, http.StatusUnauthorized, map[string]string{
					"error": "authentication required",
				})
				return
			}

			// Step 2: Validate the token
			claims, err := jwtIssuer.ValidateAccessToken(tokenString)
			if err != nil {
				log.Printf("Token validation failed: %v", err)
				writeJSON(w, http.StatusUnauthorized, map[string]string{
					"error": "invalid or expired token",
				})
				return
			}

			// Step 3: Create UserClaims and inject into context
			userClaims := &UserClaims{
				UserID:  claims.Subject,
				Email:   claims.Email,
				Name:    claims.Name,
				Picture: claims.Picture,
				Roles:   claims.Roles,
			}

			ctx := context.WithValue(r.Context(), UserClaimsKey, userClaims)

			// Step 4: Call the next handler with the enriched context
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// extractToken gets the JWT from the request, checking cookie first, then header.
func extractToken(r *http.Request) string {
	// Priority 1: Cookie (preferred for web apps)
	if cookie, err := r.Cookie("access_token"); err == nil {
		return cookie.Value
	}

	// Priority 2: Authorization header (for API clients, mobile apps)
	authHeader := r.Header.Get("Authorization")
	if strings.HasPrefix(authHeader, "Bearer ") {
		return strings.TrimPrefix(authHeader, "Bearer ")
	}

	return ""
}

// --- Optional Auth Middleware ---

// OptionalAuthMiddleware is like AuthMiddleware but does not reject
// unauthenticated requests. If a valid token is present, claims
// are injected into the context. Otherwise, the request proceeds
// without claims. Useful for endpoints that behave differently
// for authenticated vs unauthenticated users.
func OptionalAuthMiddleware(jwtIssuer *JWTIssuer) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			tokenString := extractToken(r)
			if tokenString != "" {
				claims, err := jwtIssuer.ValidateAccessToken(tokenString)
				if err == nil {
					userClaims := &UserClaims{
						UserID:  claims.Subject,
						Email:   claims.Email,
						Name:    claims.Name,
						Picture: claims.Picture,
						Roles:   claims.Roles,
					}
					ctx := context.WithValue(r.Context(), UserClaimsKey, userClaims)
					r = r.WithContext(ctx)
				}
				// If token is invalid, just proceed without claims
			}
			next.ServeHTTP(w, r)
		})
	}
}

// --- Helper ---

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

// --- Example Usage ---

func main() {
	seedHex := "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"
	seed, _ := hex.DecodeString(seedHex)

	jwtIss := &JWTIssuer{
		privateKey: ed25519.NewKeyFromSeed(seed),
		issuer:     "my-api",
		accessTTL:  15 * time.Minute,
		refreshTTL: 7 * 24 * time.Hour,
	}
	jwtIss.publicKey = jwtIss.privateKey.Public().(ed25519.PublicKey)

	mux := http.NewServeMux()

	// Protected endpoint -- requires authentication
	protectedHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims := r.Context().Value(UserClaimsKey).(*UserClaims)
		writeJSON(w, http.StatusOK, map[string]string{
			"message": fmt.Sprintf("Hello, %s!", claims.Email),
			"user_id": claims.UserID,
		})
	})
	mux.Handle("/api/me", AuthMiddleware(jwtIss)(protectedHandler))

	// Public endpoint with optional auth
	publicHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims, ok := r.Context().Value(UserClaimsKey).(*UserClaims)
		if ok && claims != nil {
			writeJSON(w, http.StatusOK, map[string]string{
				"message": fmt.Sprintf("Welcome back, %s!", claims.Email),
			})
		} else {
			writeJSON(w, http.StatusOK, map[string]string{
				"message": "Welcome, guest!",
			})
		}
	})
	mux.Handle("/api/greeting", OptionalAuthMiddleware(jwtIss)(publicHandler))

	fmt.Println("Server listening on :8080")
	fmt.Println("  GET /api/me       -- requires auth")
	fmt.Println("  GET /api/greeting -- optional auth")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Node.js Comparison: Passport.js vs Go Middleware

```javascript
// Node.js with Passport.js
const passport = require('passport');
const { Strategy: JwtStrategy, ExtractJwt } = require('passport-jwt');

passport.use(new JwtStrategy({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: publicKey,
  algorithms: ['EdDSA'],
  issuer: 'my-api',
  audience: 'my-api',
}, (payload, done) => {
  return done(null, {
    userId: payload.sub,
    email: payload.email,
    roles: payload.roles,
  });
}));

// Protected route
app.get('/api/me',
  passport.authenticate('jwt', { session: false }),
  (req, res) => {
    res.json({ message: `Hello, ${req.user.email}!` });
  }
);

// Go advantage: No framework needed. The middleware is ~30 lines of code.
// Passport.js requires passport + passport-jwt + strategy configuration.
// Go's approach is explicit -- you can read exactly what happens.
```

---

## 13. Extracting User Info from Context

### Helper Functions

After the auth middleware injects claims into the context, downstream handlers need convenient ways to access them:

```go
package auth

import (
	"context"
)

// contextKey is an unexported type for context keys in this package.
type contextKey string

// UserClaimsKey is the context key for storing user claims.
const UserClaimsKey contextKey = "user_claims"

// UserClaims represents the authenticated user's information.
type UserClaims struct {
	UserID  string
	Email   string
	Name    *string
	Picture *string
	Roles   []string
}

// GetUserClaims extracts user claims from the context.
// Returns nil if the context does not contain claims (unauthenticated request).
func GetUserClaims(ctx context.Context) *UserClaims {
	claims, ok := ctx.Value(UserClaimsKey).(*UserClaims)
	if !ok {
		return nil
	}
	return claims
}

// GetUserID extracts the user ID from the context.
// Returns an empty string if not authenticated.
func GetUserID(ctx context.Context) string {
	claims := GetUserClaims(ctx)
	if claims == nil {
		return ""
	}
	return claims.UserID
}

// GetUserEmail extracts the user's email from the context.
// Returns an empty string if not authenticated.
func GetUserEmail(ctx context.Context) string {
	claims := GetUserClaims(ctx)
	if claims == nil {
		return ""
	}
	return claims.Email
}

// GetUserRoles extracts the user's roles from the context.
// Returns nil if not authenticated.
func GetUserRoles(ctx context.Context) []string {
	claims := GetUserClaims(ctx)
	if claims == nil {
		return nil
	}
	return claims.Roles
}

// IsAuthenticated checks if the request context contains valid user claims.
func IsAuthenticated(ctx context.Context) bool {
	return GetUserClaims(ctx) != nil
}

// HasRole checks if the authenticated user has a specific role.
func HasRole(ctx context.Context, role string) bool {
	roles := GetUserRoles(ctx)
	for _, r := range roles {
		if r == role {
			return true
		}
	}
	return false
}

// HasAnyRole checks if the authenticated user has any of the specified roles.
func HasAnyRole(ctx context.Context, roles ...string) bool {
	userRoles := GetUserRoles(ctx)
	roleSet := make(map[string]struct{}, len(userRoles))
	for _, r := range userRoles {
		roleSet[r] = struct{}{}
	}
	for _, r := range roles {
		if _, ok := roleSet[r]; ok {
			return true
		}
	}
	return false
}
```

### Usage in Handlers

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
)

type contextKey string

const UserClaimsKey contextKey = "user_claims"

type UserClaims struct {
	UserID  string
	Email   string
	Name    *string
	Picture *string
	Roles   []string
}

func GetUserClaims(ctx context.Context) *UserClaims {
	claims, ok := ctx.Value(UserClaimsKey).(*UserClaims)
	if !ok {
		return nil
	}
	return claims
}

func GetUserID(ctx context.Context) string {
	claims := GetUserClaims(ctx)
	if claims == nil {
		return ""
	}
	return claims.UserID
}

func IsAuthenticated(ctx context.Context) bool {
	return GetUserClaims(ctx) != nil
}

// Example: A handler that uses the auth context helpers
func handleGetProfile(w http.ResponseWriter, r *http.Request) {
	claims := GetUserClaims(r.Context())
	if claims == nil {
		// This should not happen if AuthMiddleware is applied,
		// but defensive coding is good practice.
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}

	response := map[string]interface{}{
		"id":    claims.UserID,
		"email": claims.Email,
	}

	if claims.Name != nil {
		response["name"] = *claims.Name
	}
	if claims.Picture != nil {
		response["picture"] = *claims.Picture
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

// Example: A service layer function that gets the user ID from context
type OrderService struct{}

func (s *OrderService) CreateOrder(ctx context.Context, items []string) error {
	userID := GetUserID(ctx)
	if userID == "" {
		return fmt.Errorf("user not authenticated")
	}

	// Use userID to associate the order with the user
	fmt.Printf("Creating order for user %s with items: %v\n", userID, items)
	return nil
}

// Example: Audit logging with context
func auditLog(ctx context.Context, action, resource string) {
	userID := GetUserID(ctx)
	if userID == "" {
		userID = "anonymous"
	}
	fmt.Printf("AUDIT: user=%s action=%s resource=%s\n", userID, action, resource)
}

func main() {
	// Simulate a request with auth context
	ctx := context.Background()
	name := "Alice"
	claims := &UserClaims{
		UserID: "user_123",
		Email:  "alice@example.com",
		Name:   &name,
		Roles:  []string{"admin", "editor"},
	}
	ctx = context.WithValue(ctx, UserClaimsKey, claims)

	// Use the helpers
	fmt.Println("Authenticated:", IsAuthenticated(ctx))
	fmt.Println("User ID:      ", GetUserID(ctx))

	// Service layer
	svc := &OrderService{}
	svc.CreateOrder(ctx, []string{"item1", "item2"})

	// Audit logging
	auditLog(ctx, "create", "order/456")
}
```

---

## 14. Role-Based Authorization

### RBAC Overview

Role-Based Access Control (RBAC) assigns permissions based on roles. A user has one or more roles, and each role has specific permissions:

```
+--------+     +---------+     +-------------+
| User   |────►| Role    |────►| Permission  |
+--------+     +---------+     +-------------+
| Alice  |     | admin   |     | create:*    |
| Bob    |     | editor  |     | read:*      |
| Carol  |     | viewer  |     | update:own  |
+--------+     +---------+     | delete:*    |
                               +-------------+

Alice ──► admin  ──► create:*, read:*, update:*, delete:*
Bob   ──► editor ──► create:*, read:*, update:own
Carol ──► viewer ──► read:*
```

### Adding Roles to JWT Claims

```go
package auth

import (
	"github.com/golang-jwt/jwt/v5"
)

// AccessClaims with roles for RBAC.
type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string   `json:"email"`
	Name    *string  `json:"name,omitempty"`
	Picture *string  `json:"picture,omitempty"`
	Roles   []string `json:"roles,omitempty"`
}
```

### Authorization Middleware

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"log"
)

type contextKey string

const UserClaimsKey contextKey = "user_claims"

type UserClaims struct {
	UserID  string
	Email   string
	Name    *string
	Picture *string
	Roles   []string
}

func GetUserClaims(ctx context.Context) *UserClaims {
	claims, ok := ctx.Value(UserClaimsKey).(*UserClaims)
	if !ok {
		return nil
	}
	return claims
}

// RequireRole creates middleware that requires the user to have a specific role.
func RequireRole(role string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims := GetUserClaims(r.Context())
			if claims == nil {
				writeJSON(w, http.StatusUnauthorized, map[string]string{
					"error": "authentication required",
				})
				return
			}

			if !hasRole(claims.Roles, role) {
				log.Printf("Authorization denied: user %s lacks role %s (has: %v)",
					claims.UserID, role, claims.Roles)
				writeJSON(w, http.StatusForbidden, map[string]string{
					"error": fmt.Sprintf("role '%s' required", role),
				})
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// RequireAnyRole creates middleware that requires the user to have at least
// one of the specified roles.
func RequireAnyRole(roles ...string) func(http.Handler) http.Handler {
	roleSet := make(map[string]struct{}, len(roles))
	for _, r := range roles {
		roleSet[r] = struct{}{}
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims := GetUserClaims(r.Context())
			if claims == nil {
				writeJSON(w, http.StatusUnauthorized, map[string]string{
					"error": "authentication required",
				})
				return
			}

			for _, userRole := range claims.Roles {
				if _, ok := roleSet[userRole]; ok {
					next.ServeHTTP(w, r)
					return
				}
			}

			log.Printf("Authorization denied: user %s has roles %v, needs one of %v",
				claims.UserID, claims.Roles, roles)
			writeJSON(w, http.StatusForbidden, map[string]string{
				"error": "insufficient permissions",
			})
		})
	}
}

// RequireAllRoles creates middleware that requires the user to have ALL
// of the specified roles.
func RequireAllRoles(roles ...string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims := GetUserClaims(r.Context())
			if claims == nil {
				writeJSON(w, http.StatusUnauthorized, map[string]string{
					"error": "authentication required",
				})
				return
			}

			userRoleSet := make(map[string]struct{}, len(claims.Roles))
			for _, r := range claims.Roles {
				userRoleSet[r] = struct{}{}
			}

			for _, required := range roles {
				if _, ok := userRoleSet[required]; !ok {
					log.Printf("Authorization denied: user %s missing role %s",
						claims.UserID, required)
					writeJSON(w, http.StatusForbidden, map[string]string{
						"error": "insufficient permissions",
					})
					return
				}
			}

			next.ServeHTTP(w, r)
		})
	}
}

// ResourceOwnerMiddleware checks that the authenticated user owns the resource
// being accessed. The ownerIDFunc extracts the resource owner's ID from the request.
func ResourceOwnerMiddleware(ownerIDFunc func(*http.Request) string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims := GetUserClaims(r.Context())
			if claims == nil {
				writeJSON(w, http.StatusUnauthorized, map[string]string{
					"error": "authentication required",
				})
				return
			}

			ownerID := ownerIDFunc(r)
			if ownerID != claims.UserID {
				// Check if user is admin (admins can access any resource)
				if !hasRole(claims.Roles, "admin") {
					writeJSON(w, http.StatusForbidden, map[string]string{
						"error": "you can only access your own resources",
					})
					return
				}
			}

			next.ServeHTTP(w, r)
		})
	}
}

func hasRole(userRoles []string, target string) bool {
	for _, r := range userRoles {
		if r == target {
			return true
		}
	}
	return false
}

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

// Chain applies middleware in order (left to right).
func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
	for i := len(middlewares) - 1; i >= 0; i-- {
		handler = middlewares[i](handler)
	}
	return handler
}

// --- Example Usage ---

// Dummy auth middleware for this example
func dummyAuth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Simulate an authenticated admin user
		claims := &UserClaims{
			UserID: "user_123",
			Email:  "admin@example.com",
			Roles:  []string{"admin", "editor"},
		}
		ctx := context.WithValue(r.Context(), UserClaimsKey, claims)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func main() {
	mux := http.NewServeMux()

	// Public endpoint -- no auth needed
	mux.HandleFunc("/api/health", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	// Authenticated endpoint -- any authenticated user
	mux.Handle("/api/me", Chain(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims := GetUserClaims(r.Context())
			writeJSON(w, http.StatusOK, map[string]string{
				"user_id": claims.UserID,
				"email":   claims.Email,
			})
		}),
		dummyAuth,
	))

	// Admin-only endpoint
	mux.Handle("/api/admin/users", Chain(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			writeJSON(w, http.StatusOK, map[string]string{
				"message": "admin users list",
			})
		}),
		dummyAuth,
		RequireRole("admin"),
	))

	// Editor or admin endpoint
	mux.Handle("/api/posts", Chain(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			writeJSON(w, http.StatusOK, map[string]string{
				"message": "posts management",
			})
		}),
		dummyAuth,
		RequireAnyRole("admin", "editor"),
	))

	fmt.Println("Server listening on :8080")
	fmt.Println("  GET /api/health       -- public")
	fmt.Println("  GET /api/me           -- authenticated")
	fmt.Println("  GET /api/admin/users  -- admin only")
	fmt.Println("  GET /api/posts        -- admin or editor")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Node.js Comparison: RBAC Middleware

```javascript
// Node.js / Express RBAC middleware
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'authentication required' });
    }

    const hasRole = req.user.roles.some(role => allowedRoles.includes(role));
    if (!hasRole) {
      return res.status(403).json({ error: 'insufficient permissions' });
    }

    next();
  };
}

// Usage
app.get('/api/admin/users',
  passport.authenticate('jwt', { session: false }),
  requireRole('admin'),
  (req, res) => {
    res.json({ message: 'admin users list' });
  }
);

// Very similar pattern. The key difference is type safety:
// Go enforces the roles are strings at compile time.
// Node.js relies on runtime behavior.
```

---

## 15. Security Best Practices

### 1. Token Expiry Strategy

```
Access Token:
  ├── TTL: 5-15 minutes
  ├── Why short: Limits damage window if stolen
  ├── Tradeoff: More frequent refreshes
  └── Recommendation: 15 minutes for most applications

Refresh Token:
  ├── TTL: 7-30 days
  ├── Why longer: User convenience (stay logged in)
  ├── Must be: Stored server-side (hashed), rotated on use
  └── Recommendation: 7 days with rotation
```

### 2. Token Rotation

Always rotate refresh tokens on use. Implement reuse detection:

```go
package main

import (
	"context"
	"fmt"
	"log"
)

// RefreshTokenRepository defines the interface for token storage.
type RefreshTokenRepository interface {
	MarkUsed(ctx context.Context, tokenID string) (alreadyUsed bool, err error)
	RevokeAllForUser(ctx context.Context, userID string) error
}

// ValidateAndRotate validates a refresh token and detects reuse.
// If a token has already been used, ALL tokens for the user are revoked.
func ValidateAndRotate(ctx context.Context, repo RefreshTokenRepository, tokenID, userID string) error {
	alreadyUsed, err := repo.MarkUsed(ctx, tokenID)
	if err != nil {
		return fmt.Errorf("checking token usage: %w", err)
	}

	if alreadyUsed {
		// SECURITY ALERT: This refresh token was already used.
		// This means either:
		// 1. An attacker stole the token and used it
		// 2. A race condition caused the client to send it twice
		//
		// In either case, the safest action is to revoke all tokens
		// for this user, forcing re-authentication.
		log.Printf("SECURITY: Refresh token reuse detected for user %s, token %s", userID, tokenID)

		if err := repo.RevokeAllForUser(ctx, userID); err != nil {
			log.Printf("Failed to revoke tokens for user %s: %v", userID, err)
		}

		return fmt.Errorf("refresh token reuse detected -- all sessions revoked")
	}

	return nil
}
```

### 3. CSRF Protection with SameSite Cookies

When using cookies, SameSite=Lax prevents most CSRF attacks. For extra security with SameSite=None or legacy browsers, add a CSRF token:

```go
package main

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
)

// GenerateCSRFToken creates a cryptographically random CSRF token.
func GenerateCSRFToken() (string, error) {
	bytes := make([]byte, 32)
	if _, err := rand.Read(bytes); err != nil {
		return "", err
	}
	return hex.EncodeToString(bytes), nil
}

// CSRFMiddleware validates the CSRF token for state-changing requests.
// The token is set as a non-httpOnly cookie (so JavaScript can read it)
// and must be sent back in the X-CSRF-Token header.
func CSRFMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Skip CSRF check for safe methods
		if r.Method == http.MethodGet || r.Method == http.MethodHead || r.Method == http.MethodOptions {
			next.ServeHTTP(w, r)
			return
		}

		// Get the token from the cookie (set during page load)
		cookie, err := r.Cookie("csrf_token")
		if err != nil {
			http.Error(w, "CSRF token missing", http.StatusForbidden)
			return
		}

		// Get the token from the request header (set by JavaScript)
		headerToken := r.Header.Get("X-CSRF-Token")
		if headerToken == "" {
			http.Error(w, "CSRF header missing", http.StatusForbidden)
			return
		}

		// Compare
		if cookie.Value != headerToken {
			http.Error(w, "CSRF token mismatch", http.StatusForbidden)
			return
		}

		next.ServeHTTP(w, r)
	})
}

// SetCSRFCookie sets a new CSRF token cookie.
// This cookie is NOT httpOnly, so JavaScript can read it
// and include it in the X-CSRF-Token header.
func SetCSRFCookie(w http.ResponseWriter) error {
	token, err := GenerateCSRFToken()
	if err != nil {
		return err
	}

	http.SetCookie(w, &http.Cookie{
		Name:     "csrf_token",
		Value:    token,
		MaxAge:   3600,            // 1 hour
		HttpOnly: false,           // JavaScript MUST be able to read this
		Secure:   true,
		SameSite: http.SameSiteLaxMode,
		Path:     "/",
	})

	return nil
}

func main() {
	mux := http.NewServeMux()

	// Page load sets CSRF cookie
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		SetCSRFCookie(w)
		w.Write([]byte("CSRF cookie set"))
	})

	// Protected endpoint
	mux.Handle("/api/data", CSRFMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Data updated"))
	})))

	fmt.Println("CSRF example server on :8080")
	http.ListenAndServe(":8080", mux)
}
```

### 4. Secure Headers

```go
package main

import (
	"fmt"
	"net/http"
)

// SecurityHeadersMiddleware adds security headers to every response.
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Prevent MIME type sniffing
		w.Header().Set("X-Content-Type-Options", "nosniff")

		// Prevent clickjacking
		w.Header().Set("X-Frame-Options", "DENY")

		// Enable XSS protection (legacy browsers)
		w.Header().Set("X-XSS-Protection", "1; mode=block")

		// Strict Transport Security (HTTPS only)
		w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

		// Content Security Policy
		w.Header().Set("Content-Security-Policy", "default-src 'self'")

		// Referrer Policy
		w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")

		// Permissions Policy (formerly Feature-Policy)
		w.Header().Set("Permissions-Policy", "camera=(), microphone=(), geolocation=()")

		next.ServeHTTP(w, r)
	})
}

func main() {
	handler := SecurityHeadersMiddleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		fmt.Fprintf(w, `{"status":"ok"}`)
	}))

	http.ListenAndServe(":8080", handler)
}
```

### 5. Rate Limiting Login Endpoints

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"sync"
	"time"
)

// RateLimiter implements a simple sliding window rate limiter.
type RateLimiter struct {
	mu       sync.Mutex
	attempts map[string][]time.Time
	limit    int
	window   time.Duration
}

// NewRateLimiter creates a rate limiter that allows `limit` attempts per `window`.
func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
	rl := &RateLimiter{
		attempts: make(map[string][]time.Time),
		limit:    limit,
		window:   window,
	}

	// Periodic cleanup of expired entries
	go func() {
		ticker := time.NewTicker(5 * time.Minute)
		defer ticker.Stop()
		for range ticker.C {
			rl.cleanup()
		}
	}()

	return rl
}

// Allow checks if a request from the given key should be allowed.
func (rl *RateLimiter) Allow(key string) bool {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	now := time.Now()
	windowStart := now.Add(-rl.window)

	// Filter to only recent attempts
	recent := make([]time.Time, 0)
	for _, t := range rl.attempts[key] {
		if t.After(windowStart) {
			recent = append(recent, t)
		}
	}

	if len(recent) >= rl.limit {
		rl.attempts[key] = recent
		return false
	}

	rl.attempts[key] = append(recent, now)
	return true
}

func (rl *RateLimiter) cleanup() {
	rl.mu.Lock()
	defer rl.mu.Unlock()

	windowStart := time.Now().Add(-rl.window)
	for key, attempts := range rl.attempts {
		recent := make([]time.Time, 0)
		for _, t := range attempts {
			if t.After(windowStart) {
				recent = append(recent, t)
			}
		}
		if len(recent) == 0 {
			delete(rl.attempts, key)
		} else {
			rl.attempts[key] = recent
		}
	}
}

// RateLimitMiddleware wraps a handler with rate limiting based on client IP.
func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Use the client's IP as the rate limit key
			clientIP := r.RemoteAddr // In production, use X-Forwarded-For or X-Real-IP

			if !limiter.Allow(clientIP) {
				w.Header().Set("Retry-After", "60")
				w.Header().Set("Content-Type", "application/json")
				w.WriteHeader(http.StatusTooManyRequests)
				json.NewEncoder(w).Encode(map[string]string{
					"error": "too many requests, please try again later",
				})
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

func main() {
	// Allow 5 login attempts per minute per IP
	loginLimiter := NewRateLimiter(5, 1*time.Minute)

	mux := http.NewServeMux()

	// Rate-limited login endpoint
	loginHandler := RateLimitMiddleware(loginLimiter)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{
			"message": "login endpoint",
		})
	}))
	mux.Handle("/auth/login", loginHandler)

	fmt.Println("Rate limiting example on :8080")
	fmt.Println("  POST /auth/login -- 5 requests/minute")
	http.ListenAndServe(":8080", mux)
}
```

### Security Checklist

```
Token Security:
  ✓ Use asymmetric signing (Ed25519) -- separate signing and verification keys
  ✓ Short access token TTL (5-15 minutes)
  ✓ Refresh token rotation with reuse detection
  ✓ Store refresh tokens hashed (SHA-256) in the database
  ✓ Validate algorithm, issuer, audience on EVERY token parse
  ✓ Use unique JTI claims for revocation tracking

Cookie Security:
  ✓ HttpOnly flag on ALL auth cookies
  ✓ Secure flag in production (HTTPS only)
  ✓ SameSite=Lax (or Strict) to prevent CSRF
  ✓ Restrict refresh token cookie path to /auth
  ✓ Set appropriate Domain attribute
  ✓ Clear cookies on logout (MaxAge=-1)

Transport Security:
  ✓ HTTPS everywhere in production
  ✓ HSTS header (Strict-Transport-Security)
  ✓ TLS 1.2+ minimum

API Security:
  ✓ Rate limit authentication endpoints
  ✓ Log failed authentication attempts
  ✓ Do NOT reveal whether a user exists (use generic error messages)
  ✓ Set security response headers (X-Content-Type-Options, etc.)
  ✓ Validate Content-Type on POST requests
  ✓ Set response Content-Type explicitly

Key Management:
  ✓ Store signing keys in a secrets manager (Vault, AWS Secrets Manager)
  ✓ Never commit keys to version control
  ✓ Rotate keys periodically (implement key ID / kid header)
  ✓ Separate keys per environment (dev, staging, production)
```

---

## 16. Testing Authentication

### Testing Strategy

Authentication code is security-critical and must be thoroughly tested:

1. **Unit tests:** Token issuance and validation logic
2. **Middleware tests:** Auth middleware with valid, invalid, and missing tokens
3. **Integration tests:** Full token exchange and refresh flows
4. **Security tests:** Expired tokens, tampered tokens, wrong algorithms

### Test Helper: Creating Test JWTIssuers

```go
package auth_test

import (
	"crypto/ed25519"
	"testing"
	"time"
)

// TestJWTIssuer creates a JWTIssuer with a deterministic key for testing.
// The key is hardcoded, so tests are reproducible.
func TestJWTIssuer(t *testing.T) *JWTIssuer {
	t.Helper()

	// Deterministic seed for reproducible tests
	seed := make([]byte, ed25519.SeedSize)
	for i := range seed {
		seed[i] = byte(i)
	}

	issuer, err := NewJWTIssuer(seed, "test-api", 15*time.Minute, 7*24*time.Hour)
	if err != nil {
		t.Fatalf("Failed to create test JWTIssuer: %v", err)
	}
	return issuer
}
```

### Testing Token Issuance and Validation

```go
package main

import (
	"crypto/ed25519"
	"fmt"
	"testing"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

// --- Production Types (same as before, included for completeness) ---

type TokenPair struct {
	AccessToken  string
	RefreshToken string
}

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string  `json:"email"`
	Name    *string `json:"name,omitempty"`
	Picture *string `json:"picture,omitempty"`
}

type RefreshClaims struct {
	jwt.RegisteredClaims
}

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) (*JWTIssuer, error) {
	if len(seed) != ed25519.SeedSize {
		return nil, fmt.Errorf("seed must be %d bytes", ed25519.SeedSize)
	}
	pk := ed25519.NewKeyFromSeed(seed)
	return &JWTIssuer{
		privateKey: pk,
		publicKey:  pk.Public().(ed25519.PublicKey),
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}, nil
}

func (j *JWTIssuer) IssueTokenPair(userID, email string, name, picture *string) (*TokenPair, error) {
	now := time.Now()
	ac := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject: userID, Issuer: j.issuer, Audience: jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt: jwt.NewNumericDate(now), NotBefore: jwt.NewNumericDate(now),
		},
		Email: email, Name: name, Picture: picture,
	}
	at := jwt.NewWithClaims(jwt.SigningMethodEdDSA, ac)
	sa, err := at.SignedString(j.privateKey)
	if err != nil {
		return nil, err
	}
	rc := &RefreshClaims{RegisteredClaims: jwt.RegisteredClaims{
		Subject: userID, Issuer: j.issuer, Audience: jwt.ClaimStrings{j.issuer},
		ExpiresAt: jwt.NewNumericDate(now.Add(j.refreshTTL)),
		IssuedAt: jwt.NewNumericDate(now), NotBefore: jwt.NewNumericDate(now),
		ID: uuid.New().String(),
	}}
	rt := jwt.NewWithClaims(jwt.SigningMethodEdDSA, rc)
	sr, err := rt.SignedString(j.privateKey)
	if err != nil {
		return nil, err
	}
	return &TokenPair{AccessToken: sa, RefreshToken: sr}, nil
}

func (j *JWTIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, fmt.Errorf("invalid token")
	}
	return claims, nil
}

// --- Test Helper ---

func newTestIssuer(t *testing.T) *JWTIssuer {
	t.Helper()
	seed := make([]byte, ed25519.SeedSize)
	for i := range seed {
		seed[i] = byte(i)
	}
	issuer, err := NewJWTIssuer(seed, "test-api", 15*time.Minute, 7*24*time.Hour)
	if err != nil {
		t.Fatalf("Failed to create test issuer: %v", err)
	}
	return issuer
}

// --- Tests ---

func TestIssueTokenPair(t *testing.T) {
	issuer := newTestIssuer(t)

	name := "Alice"
	picture := "https://example.com/alice.jpg"
	pair, err := issuer.IssueTokenPair("user_123", "alice@example.com", &name, &picture)
	if err != nil {
		t.Fatalf("IssueTokenPair failed: %v", err)
	}

	if pair.AccessToken == "" {
		t.Error("Access token is empty")
	}
	if pair.RefreshToken == "" {
		t.Error("Refresh token is empty")
	}
	if pair.AccessToken == pair.RefreshToken {
		t.Error("Access and refresh tokens should be different")
	}
}

func TestValidateAccessToken_Valid(t *testing.T) {
	issuer := newTestIssuer(t)

	name := "Bob"
	pair, err := issuer.IssueTokenPair("user_456", "bob@example.com", &name, nil)
	if err != nil {
		t.Fatalf("IssueTokenPair failed: %v", err)
	}

	claims, err := issuer.ValidateAccessToken(pair.AccessToken)
	if err != nil {
		t.Fatalf("ValidateAccessToken failed: %v", err)
	}

	if claims.Subject != "user_456" {
		t.Errorf("Expected subject 'user_456', got '%s'", claims.Subject)
	}
	if claims.Email != "bob@example.com" {
		t.Errorf("Expected email 'bob@example.com', got '%s'", claims.Email)
	}
	if claims.Name == nil || *claims.Name != "Bob" {
		t.Errorf("Expected name 'Bob', got %v", claims.Name)
	}
	if claims.Picture != nil {
		t.Errorf("Expected picture nil, got %v", claims.Picture)
	}
}

func TestValidateAccessToken_Tampered(t *testing.T) {
	issuer := newTestIssuer(t)

	pair, _ := issuer.IssueTokenPair("user_123", "alice@example.com", nil, nil)

	// Tamper with the token
	tampered := pair.AccessToken[:len(pair.AccessToken)-5] + "XXXXX"

	_, err := issuer.ValidateAccessToken(tampered)
	if err == nil {
		t.Error("Expected validation to fail for tampered token")
	}
}

func TestValidateAccessToken_WrongIssuer(t *testing.T) {
	issuer := newTestIssuer(t)
	pair, _ := issuer.IssueTokenPair("user_123", "alice@example.com", nil, nil)

	// Create a verifier with a different issuer name but same keys
	seed := make([]byte, ed25519.SeedSize)
	for i := range seed {
		seed[i] = byte(i)
	}
	wrongIssuer, _ := NewJWTIssuer(seed, "wrong-api", 15*time.Minute, 7*24*time.Hour)

	_, err := wrongIssuer.ValidateAccessToken(pair.AccessToken)
	if err == nil {
		t.Error("Expected validation to fail for wrong issuer")
	}
}

func TestValidateAccessToken_DifferentKey(t *testing.T) {
	issuer := newTestIssuer(t)
	pair, _ := issuer.IssueTokenPair("user_123", "alice@example.com", nil, nil)

	// Create an issuer with different keys
	differentSeed := make([]byte, ed25519.SeedSize)
	for i := range differentSeed {
		differentSeed[i] = byte(i + 100)
	}
	differentIssuer, _ := NewJWTIssuer(differentSeed, "test-api", 15*time.Minute, 7*24*time.Hour)

	_, err := differentIssuer.ValidateAccessToken(pair.AccessToken)
	if err == nil {
		t.Error("Expected validation to fail for different signing key")
	}
}

func TestNewJWTIssuer_InvalidSeedSize(t *testing.T) {
	_, err := NewJWTIssuer([]byte("too-short"), "test-api", 15*time.Minute, 7*24*time.Hour)
	if err == nil {
		t.Error("Expected error for invalid seed size")
	}
}
```

### Testing Middleware

```go
package main

import (
	"context"
	"crypto/ed25519"
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

// --- Minimal types for middleware testing ---

type contextKey string

const UserClaimsKey contextKey = "user_claims"

type UserClaims struct {
	UserID string
	Email  string
	Roles  []string
}

type AccessClaims struct {
	jwt.RegisteredClaims
	Email string   `json:"email"`
	Roles []string `json:"roles,omitempty"`
}

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
}

func (j *JWTIssuer) IssueAccessToken(userID, email string, roles []string) (string, error) {
	now := time.Now()
	claims := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject: userID, Issuer: j.issuer, Audience: jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt: jwt.NewNumericDate(now), NotBefore: jwt.NewNumericDate(now),
		},
		Email: email, Roles: roles,
	}
	token := jwt.NewWithClaims(jwt.SigningMethodEdDSA, claims)
	return token.SignedString(j.privateKey)
}

func (j *JWTIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method")
		}
		return j.publicKey, nil
	}, jwt.WithIssuer(j.issuer), jwt.WithAudience(j.issuer), jwt.WithExpirationRequired(), jwt.WithValidMethods([]string{"EdDSA"}))
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, fmt.Errorf("invalid")
	}
	return claims, nil
}

// AuthMiddleware for testing
func AuthMiddleware(jwtIssuer *JWTIssuer) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			var tokenString string
			if cookie, err := r.Cookie("access_token"); err == nil {
				tokenString = cookie.Value
			}
			if tokenString == "" {
				auth := r.Header.Get("Authorization")
				if strings.HasPrefix(auth, "Bearer ") {
					tokenString = strings.TrimPrefix(auth, "Bearer ")
				}
			}
			if tokenString == "" {
				w.WriteHeader(http.StatusUnauthorized)
				json.NewEncoder(w).Encode(map[string]string{"error": "authentication required"})
				return
			}
			claims, err := jwtIssuer.ValidateAccessToken(tokenString)
			if err != nil {
				w.WriteHeader(http.StatusUnauthorized)
				json.NewEncoder(w).Encode(map[string]string{"error": "invalid token"})
				return
			}
			userClaims := &UserClaims{
				UserID: claims.Subject, Email: claims.Email, Roles: claims.Roles,
			}
			ctx := context.WithValue(r.Context(), UserClaimsKey, userClaims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

// --- Test Helper ---

func newTestIssuerForMiddleware(t *testing.T) *JWTIssuer {
	t.Helper()
	seed := make([]byte, ed25519.SeedSize)
	for i := range seed {
		seed[i] = byte(i)
	}
	pk := ed25519.NewKeyFromSeed(seed)
	return &JWTIssuer{
		privateKey: pk,
		publicKey:  pk.Public().(ed25519.PublicKey),
		issuer:     "test-api",
		accessTTL:  15 * time.Minute,
	}
}

// --- Middleware Tests ---

func TestAuthMiddleware_ValidTokenInCookie(t *testing.T) {
	issuer := newTestIssuerForMiddleware(t)

	token, err := issuer.IssueAccessToken("user_123", "alice@example.com", []string{"admin"})
	if err != nil {
		t.Fatalf("Failed to issue token: %v", err)
	}

	handler := AuthMiddleware(issuer)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims := r.Context().Value(UserClaimsKey).(*UserClaims)
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]string{
			"user_id": claims.UserID,
			"email":   claims.Email,
		})
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/me", nil)
	req.AddCookie(&http.Cookie{Name: "access_token", Value: token})
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("Expected 200, got %d", rec.Code)
	}

	var body map[string]string
	json.NewDecoder(rec.Body).Decode(&body)
	if body["user_id"] != "user_123" {
		t.Errorf("Expected user_id 'user_123', got '%s'", body["user_id"])
	}
}

func TestAuthMiddleware_ValidTokenInHeader(t *testing.T) {
	issuer := newTestIssuerForMiddleware(t)

	token, _ := issuer.IssueAccessToken("user_456", "bob@example.com", nil)

	handler := AuthMiddleware(issuer)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims := r.Context().Value(UserClaimsKey).(*UserClaims)
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, claims.Email)
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/me", nil)
	req.Header.Set("Authorization", "Bearer "+token)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Errorf("Expected 200, got %d", rec.Code)
	}
	if rec.Body.String() != "bob@example.com" {
		t.Errorf("Expected 'bob@example.com', got '%s'", rec.Body.String())
	}
}

func TestAuthMiddleware_NoToken(t *testing.T) {
	issuer := newTestIssuerForMiddleware(t)

	handler := AuthMiddleware(issuer)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t.Error("Handler should not be called without auth")
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/me", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusUnauthorized {
		t.Errorf("Expected 401, got %d", rec.Code)
	}
}

func TestAuthMiddleware_InvalidToken(t *testing.T) {
	issuer := newTestIssuerForMiddleware(t)

	handler := AuthMiddleware(issuer)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		t.Error("Handler should not be called with invalid token")
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/me", nil)
	req.Header.Set("Authorization", "Bearer invalid.token.here")
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusUnauthorized {
		t.Errorf("Expected 401, got %d", rec.Code)
	}
}

func TestAuthMiddleware_CookiePriorityOverHeader(t *testing.T) {
	issuer := newTestIssuerForMiddleware(t)

	cookieToken, _ := issuer.IssueAccessToken("cookie_user", "cookie@example.com", nil)
	headerToken, _ := issuer.IssueAccessToken("header_user", "header@example.com", nil)

	handler := AuthMiddleware(issuer)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims := r.Context().Value(UserClaimsKey).(*UserClaims)
		fmt.Fprintf(w, claims.UserID)
	}))

	req := httptest.NewRequest(http.MethodGet, "/api/me", nil)
	req.AddCookie(&http.Cookie{Name: "access_token", Value: cookieToken})
	req.Header.Set("Authorization", "Bearer "+headerToken)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	// Cookie should take priority
	if rec.Body.String() != "cookie_user" {
		t.Errorf("Expected 'cookie_user' (cookie priority), got '%s'", rec.Body.String())
	}
}
```

### Mocking the JWT Issuer with Interfaces

For testing handlers that depend on token issuance, define an interface:

```go
package auth

import (
	"time"
)

// TokenIssuer defines the interface for issuing and validating tokens.
// This makes it easy to mock in tests.
type TokenIssuer interface {
	IssueTokenPair(userID, email string, name, picture *string) (*TokenPair, error)
	ValidateAccessToken(tokenString string) (*AccessClaims, error)
	ValidateRefreshToken(tokenString string) (*RefreshClaims, error)
	AccessTTL() time.Duration
	RefreshTTL() time.Duration
}

// --- Mock for testing ---

// MockTokenIssuer is a test double for TokenIssuer.
type MockTokenIssuer struct {
	IssueTokenPairFunc      func(userID, email string, name, picture *string) (*TokenPair, error)
	ValidateAccessTokenFunc func(tokenString string) (*AccessClaims, error)
	ValidateRefreshFunc     func(tokenString string) (*RefreshClaims, error)
	MockAccessTTL           time.Duration
	MockRefreshTTL          time.Duration
}

func (m *MockTokenIssuer) IssueTokenPair(userID, email string, name, picture *string) (*TokenPair, error) {
	if m.IssueTokenPairFunc != nil {
		return m.IssueTokenPairFunc(userID, email, name, picture)
	}
	return &TokenPair{
		AccessToken:  "mock-access-token",
		RefreshToken: "mock-refresh-token",
	}, nil
}

func (m *MockTokenIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	if m.ValidateAccessTokenFunc != nil {
		return m.ValidateAccessTokenFunc(tokenString)
	}
	return &AccessClaims{Email: "mock@example.com"}, nil
}

func (m *MockTokenIssuer) ValidateRefreshToken(tokenString string) (*RefreshClaims, error) {
	if m.ValidateRefreshFunc != nil {
		return m.ValidateRefreshFunc(tokenString)
	}
	return &RefreshClaims{}, nil
}

func (m *MockTokenIssuer) AccessTTL() time.Duration  { return m.MockAccessTTL }
func (m *MockTokenIssuer) RefreshTTL() time.Duration { return m.MockRefreshTTL }
```

---

## 17. Real-World Example: Complete Auth System

This section ties everything together into a complete, production-ready authentication system. The code below is structured as a real Go project would be, with clean separation between the auth package, handlers, middleware, and main server.

### Project Structure

```
myapp/
├── cmd/
│   └── server/
│       └── main.go              # Application entry point
├── internal/
│   ├── auth/
│   │   ├── claims.go            # Token claim types
│   │   ├── context.go           # Context helpers
│   │   ├── cookies.go           # Cookie management
│   │   ├── issuer.go            # JWTIssuer (Ed25519)
│   │   ├── middleware.go         # Auth middleware
│   │   ├── oidc.go              # OIDC validation
│   │   └── rbac.go              # Role-based authorization
│   ├── handler/
│   │   ├── auth_handler.go      # Token exchange, refresh, logout
│   │   └── user_handler.go      # User endpoints
│   └── repository/
│       ├── user_repo.go         # User database operations
│       └── token_repo.go        # Refresh token storage
├── go.mod
└── go.sum
```

### Complete Runnable Server

The following is a single-file version of the complete system for clarity. In production, split it as shown above.

```go
package main

import (
	"context"
	"crypto/ed25519"
	"crypto/sha256"
	"crypto/subtle"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"github.com/google/uuid"
)

// =============================================================================
// Configuration
// =============================================================================

type Config struct {
	JWTSeedHex   string
	JWTIssuer    string
	AccessTTL    time.Duration
	RefreshTTL   time.Duration
	CookieDomain string
	CookieSecure bool
	ListenAddr   string
}

func LoadConfig() Config {
	return Config{
		JWTSeedHex:   getEnv("JWT_SEED_HEX", "4f1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b"),
		JWTIssuer:    getEnv("JWT_ISSUER", "myapp-api"),
		AccessTTL:    15 * time.Minute,
		RefreshTTL:   7 * 24 * time.Hour,
		CookieDomain: getEnv("COOKIE_DOMAIN", "localhost"),
		CookieSecure: getEnv("COOKIE_SECURE", "false") == "true",
		ListenAddr:   getEnv("LISTEN_ADDR", ":8080"),
	}
}

func getEnv(key, fallback string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return fallback
}

// =============================================================================
// Auth Types
// =============================================================================

type contextKey string

const UserClaimsKey contextKey = "user_claims"

type UserClaims struct {
	UserID  string   `json:"user_id"`
	Email   string   `json:"email"`
	Name    *string  `json:"name,omitempty"`
	Picture *string  `json:"picture,omitempty"`
	Roles   []string `json:"roles,omitempty"`
}

type TokenPair struct {
	AccessToken  string `json:"access_token"`
	RefreshToken string `json:"refresh_token"`
}

type AccessClaims struct {
	jwt.RegisteredClaims
	Email   string   `json:"email"`
	Name    *string  `json:"name,omitempty"`
	Picture *string  `json:"picture,omitempty"`
	Roles   []string `json:"roles,omitempty"`
}

type RefreshClaims struct {
	jwt.RegisteredClaims
}

// =============================================================================
// JWT Issuer
// =============================================================================

type JWTIssuer struct {
	privateKey ed25519.PrivateKey
	publicKey  ed25519.PublicKey
	issuer     string
	accessTTL  time.Duration
	refreshTTL time.Duration
}

func NewJWTIssuer(seed []byte, issuer string, accessTTL, refreshTTL time.Duration) (*JWTIssuer, error) {
	if len(seed) != ed25519.SeedSize {
		return nil, fmt.Errorf("seed must be exactly %d bytes, got %d", ed25519.SeedSize, len(seed))
	}
	pk := ed25519.NewKeyFromSeed(seed)
	return &JWTIssuer{
		privateKey: pk,
		publicKey:  pk.Public().(ed25519.PublicKey),
		issuer:     issuer,
		accessTTL:  accessTTL,
		refreshTTL: refreshTTL,
	}, nil
}

func (j *JWTIssuer) IssueTokenPair(userID, email string, name, picture *string, roles []string) (*TokenPair, error) {
	now := time.Now()

	// Access token
	accessClaims := &AccessClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.accessTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
		},
		Email:   email,
		Name:    name,
		Picture: picture,
		Roles:   roles,
	}
	accessToken := jwt.NewWithClaims(jwt.SigningMethodEdDSA, accessClaims)
	signedAccess, err := accessToken.SignedString(j.privateKey)
	if err != nil {
		return nil, fmt.Errorf("signing access token: %w", err)
	}

	// Refresh token
	refreshClaims := &RefreshClaims{
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   userID,
			Issuer:    j.issuer,
			Audience:  jwt.ClaimStrings{j.issuer},
			ExpiresAt: jwt.NewNumericDate(now.Add(j.refreshTTL)),
			IssuedAt:  jwt.NewNumericDate(now),
			NotBefore: jwt.NewNumericDate(now),
			ID:        uuid.New().String(),
		},
	}
	refreshToken := jwt.NewWithClaims(jwt.SigningMethodEdDSA, refreshClaims)
	signedRefresh, err := refreshToken.SignedString(j.privateKey)
	if err != nil {
		return nil, fmt.Errorf("signing refresh token: %w", err)
	}

	return &TokenPair{
		AccessToken:  signedAccess,
		RefreshToken: signedRefresh,
	}, nil
}

func (j *JWTIssuer) ValidateAccessToken(tokenString string) (*AccessClaims, error) {
	claims := &AccessClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, fmt.Errorf("invalid access token: %w", err)
	}
	if !token.Valid {
		return nil, fmt.Errorf("token is not valid")
	}
	return claims, nil
}

func (j *JWTIssuer) ValidateRefreshToken(tokenString string) (*RefreshClaims, error) {
	claims := &RefreshClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodEd25519); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
		}
		return j.publicKey, nil
	},
		jwt.WithIssuer(j.issuer),
		jwt.WithAudience(j.issuer),
		jwt.WithExpirationRequired(),
		jwt.WithValidMethods([]string{"EdDSA"}),
	)
	if err != nil {
		return nil, fmt.Errorf("invalid refresh token: %w", err)
	}
	if !token.Valid {
		return nil, fmt.Errorf("token is not valid")
	}
	if claims.ID == "" {
		return nil, fmt.Errorf("refresh token missing jti claim")
	}
	return claims, nil
}

// =============================================================================
// Context Helpers
// =============================================================================

func GetUserClaims(ctx context.Context) *UserClaims {
	claims, ok := ctx.Value(UserClaimsKey).(*UserClaims)
	if !ok {
		return nil
	}
	return claims
}

func GetUserID(ctx context.Context) string {
	if c := GetUserClaims(ctx); c != nil {
		return c.UserID
	}
	return ""
}

func IsAuthenticated(ctx context.Context) bool {
	return GetUserClaims(ctx) != nil
}

func HasRole(ctx context.Context, role string) bool {
	c := GetUserClaims(ctx)
	if c == nil {
		return false
	}
	for _, r := range c.Roles {
		if r == role {
			return true
		}
	}
	return false
}

// =============================================================================
// Cookie Helpers
// =============================================================================

func SetTokenCookies(w http.ResponseWriter, pair *TokenPair, cfg Config) {
	http.SetCookie(w, &http.Cookie{
		Name:     "access_token",
		Value:    pair.AccessToken,
		MaxAge:   int(cfg.AccessTTL.Seconds()),
		HttpOnly: true,
		Secure:   cfg.CookieSecure,
		SameSite: http.SameSiteLaxMode,
		Path:     "/",
		Domain:   cfg.CookieDomain,
	})
	http.SetCookie(w, &http.Cookie{
		Name:     "refresh_token",
		Value:    pair.RefreshToken,
		MaxAge:   int(cfg.RefreshTTL.Seconds()),
		HttpOnly: true,
		Secure:   cfg.CookieSecure,
		SameSite: http.SameSiteLaxMode,
		Path:     "/auth",
		Domain:   cfg.CookieDomain,
	})
}

func ClearTokenCookies(w http.ResponseWriter, cfg Config) {
	http.SetCookie(w, &http.Cookie{
		Name: "access_token", Value: "", MaxAge: -1, HttpOnly: true,
		Secure: cfg.CookieSecure, SameSite: http.SameSiteLaxMode, Path: "/", Domain: cfg.CookieDomain,
	})
	http.SetCookie(w, &http.Cookie{
		Name: "refresh_token", Value: "", MaxAge: -1, HttpOnly: true,
		Secure: cfg.CookieSecure, SameSite: http.SameSiteLaxMode, Path: "/auth", Domain: cfg.CookieDomain,
	})
}

// =============================================================================
// In-Memory Repositories (replace with database in production)
// =============================================================================

type User struct {
	ID        string
	Email     string
	Name      *string
	Picture   *string
	Roles     []string
	CreatedAt time.Time
}

type InMemoryUserRepo struct {
	mu    sync.RWMutex
	users map[string]*User // keyed by ID
	byEmail map[string]string // email -> ID
}

func NewInMemoryUserRepo() *InMemoryUserRepo {
	return &InMemoryUserRepo{
		users:   make(map[string]*User),
		byEmail: make(map[string]string),
	}
}

func (r *InMemoryUserRepo) FindByEmail(email string) (*User, bool) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	id, ok := r.byEmail[email]
	if !ok {
		return nil, false
	}
	return r.users[id], true
}

func (r *InMemoryUserRepo) FindByID(id string) (*User, bool) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	u, ok := r.users[id]
	return u, ok
}

func (r *InMemoryUserRepo) Create(user *User) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.users[user.ID] = user
	r.byEmail[user.Email] = user.ID
}

type StoredRefreshToken struct {
	TokenID   string
	UserID    string
	TokenHash []byte
	ExpiresAt time.Time
	Revoked   bool
}

type InMemoryTokenRepo struct {
	mu     sync.RWMutex
	tokens map[string]*StoredRefreshToken // keyed by token ID (jti)
}

func NewInMemoryTokenRepo() *InMemoryTokenRepo {
	return &InMemoryTokenRepo{
		tokens: make(map[string]*StoredRefreshToken),
	}
}

func (r *InMemoryTokenRepo) Store(tokenID, userID string, hashedToken []byte, expiresAt time.Time) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.tokens[tokenID] = &StoredRefreshToken{
		TokenID:   tokenID,
		UserID:    userID,
		TokenHash: hashedToken,
		ExpiresAt: expiresAt,
	}
}

func (r *InMemoryTokenRepo) Verify(tokenID string, hashedToken []byte) bool {
	r.mu.RLock()
	defer r.mu.RUnlock()
	stored, ok := r.tokens[tokenID]
	if !ok || stored.Revoked || time.Now().After(stored.ExpiresAt) {
		return false
	}
	return subtle.ConstantTimeCompare(stored.TokenHash, hashedToken) == 1
}

func (r *InMemoryTokenRepo) Revoke(tokenID string) {
	r.mu.Lock()
	defer r.mu.Unlock()
	if t, ok := r.tokens[tokenID]; ok {
		t.Revoked = true
	}
}

func (r *InMemoryTokenRepo) RevokeAllForUser(userID string) int {
	r.mu.Lock()
	defer r.mu.Unlock()
	count := 0
	for _, t := range r.tokens {
		if t.UserID == userID && !t.Revoked {
			t.Revoked = true
			count++
		}
	}
	return count
}

func (r *InMemoryTokenRepo) DeleteExpired() int {
	r.mu.Lock()
	defer r.mu.Unlock()
	count := 0
	now := time.Now()
	for id, t := range r.tokens {
		if now.After(t.ExpiresAt) {
			delete(r.tokens, id)
			count++
		}
	}
	return count
}

// =============================================================================
// Token Hashing
// =============================================================================

func HashToken(rawToken string) []byte {
	h := sha256.Sum256([]byte(rawToken))
	return h[:]
}

// =============================================================================
// Middleware
// =============================================================================

func AuthMiddleware(jwtIssuer *JWTIssuer) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			tokenString := extractToken(r)
			if tokenString == "" {
				writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "authentication required"})
				return
			}

			claims, err := jwtIssuer.ValidateAccessToken(tokenString)
			if err != nil {
				log.Printf("Token validation failed: %v", err)
				writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid or expired token"})
				return
			}

			userClaims := &UserClaims{
				UserID:  claims.Subject,
				Email:   claims.Email,
				Name:    claims.Name,
				Picture: claims.Picture,
				Roles:   claims.Roles,
			}
			ctx := context.WithValue(r.Context(), UserClaimsKey, userClaims)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}

func RequireRole(role string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if !HasRole(r.Context(), role) {
				writeJSON(w, http.StatusForbidden, map[string]string{
					"error": fmt.Sprintf("role '%s' required", role),
				})
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}

func extractToken(r *http.Request) string {
	if cookie, err := r.Cookie("access_token"); err == nil {
		return cookie.Value
	}
	if auth := r.Header.Get("Authorization"); strings.HasPrefix(auth, "Bearer ") {
		return strings.TrimPrefix(auth, "Bearer ")
	}
	return ""
}

// =============================================================================
// Handlers
// =============================================================================

type AuthHandler struct {
	jwtIssuer *JWTIssuer
	userRepo  *InMemoryUserRepo
	tokenRepo *InMemoryTokenRepo
	config    Config
}

// HandleDemoLogin is a simplified login for demonstration.
// In production, this would validate an OIDC token (see Section 7-8).
func (h *AuthHandler) HandleDemoLogin(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeJSON(w, http.StatusMethodNotAllowed, map[string]string{"error": "method not allowed"})
		return
	}

	var req struct {
		Email string `json:"email"`
		Name  string `json:"name"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid request body"})
		return
	}
	if req.Email == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "email is required"})
		return
	}

	// Find or create user
	user, found := h.userRepo.FindByEmail(req.Email)
	if !found {
		user = &User{
			ID:        uuid.New().String(),
			Email:     req.Email,
			Roles:     []string{"viewer"},
			CreatedAt: time.Now(),
		}
		if req.Name != "" {
			user.Name = &req.Name
		}
		h.userRepo.Create(user)
		log.Printf("Created new user: %s (%s)", user.ID, user.Email)
	}

	// Issue token pair
	pair, err := h.jwtIssuer.IssueTokenPair(user.ID, user.Email, user.Name, user.Picture, user.Roles)
	if err != nil {
		log.Printf("Failed to issue tokens: %v", err)
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal server error"})
		return
	}

	// Store refresh token hash
	refreshClaims, _ := h.jwtIssuer.ValidateRefreshToken(pair.RefreshToken)
	h.tokenRepo.Store(refreshClaims.ID, user.ID, HashToken(pair.RefreshToken), refreshClaims.ExpiresAt.Time)

	// Set cookies
	SetTokenCookies(w, pair, h.config)

	writeJSON(w, http.StatusOK, map[string]interface{}{
		"user": map[string]interface{}{
			"id":    user.ID,
			"email": user.Email,
			"name":  user.Name,
			"roles": user.Roles,
		},
		"message": "logged in",
	})
}

func (h *AuthHandler) HandleRefresh(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeJSON(w, http.StatusMethodNotAllowed, map[string]string{"error": "method not allowed"})
		return
	}

	// Extract refresh token from cookie
	cookie, err := r.Cookie("refresh_token")
	if err != nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "no refresh token"})
		return
	}

	// Validate JWT signature and claims
	refreshClaims, err := h.jwtIssuer.ValidateRefreshToken(cookie.Value)
	if err != nil {
		log.Printf("Invalid refresh token: %v", err)
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "invalid refresh token"})
		return
	}

	// Verify token exists in store and is not revoked
	if !h.tokenRepo.Verify(refreshClaims.ID, HashToken(cookie.Value)) {
		log.Printf("SECURITY: Refresh token reuse detected for user %s", refreshClaims.Subject)
		revoked := h.tokenRepo.RevokeAllForUser(refreshClaims.Subject)
		log.Printf("Revoked %d tokens for user %s", revoked, refreshClaims.Subject)
		ClearTokenCookies(w, h.config)
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "refresh token revoked"})
		return
	}

	// Fetch user
	user, found := h.userRepo.FindByID(refreshClaims.Subject)
	if !found {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "user not found"})
		return
	}

	// Revoke old refresh token
	h.tokenRepo.Revoke(refreshClaims.ID)

	// Issue new pair
	pair, err := h.jwtIssuer.IssueTokenPair(user.ID, user.Email, user.Name, user.Picture, user.Roles)
	if err != nil {
		log.Printf("Failed to issue tokens: %v", err)
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal server error"})
		return
	}

	// Store new refresh token
	newRefreshClaims, _ := h.jwtIssuer.ValidateRefreshToken(pair.RefreshToken)
	h.tokenRepo.Store(newRefreshClaims.ID, user.ID, HashToken(pair.RefreshToken), newRefreshClaims.ExpiresAt.Time)

	// Set new cookies
	SetTokenCookies(w, pair, h.config)

	writeJSON(w, http.StatusOK, map[string]string{"message": "tokens refreshed"})
}

func (h *AuthHandler) HandleLogout(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		writeJSON(w, http.StatusMethodNotAllowed, map[string]string{"error": "method not allowed"})
		return
	}

	// Try to revoke the refresh token if present
	if cookie, err := r.Cookie("refresh_token"); err == nil {
		if claims, err := h.jwtIssuer.ValidateRefreshToken(cookie.Value); err == nil {
			h.tokenRepo.Revoke(claims.ID)
			log.Printf("Revoked refresh token %s for user %s", claims.ID, claims.Subject)
		}
	}

	ClearTokenCookies(w, h.config)
	writeJSON(w, http.StatusOK, map[string]string{"message": "logged out"})
}

// HandleMe returns the authenticated user's information.
func HandleMe(w http.ResponseWriter, r *http.Request) {
	claims := GetUserClaims(r.Context())
	if claims == nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": "not authenticated"})
		return
	}
	writeJSON(w, http.StatusOK, claims)
}

// HandleAdminDashboard is an admin-only endpoint.
func HandleAdminDashboard(w http.ResponseWriter, r *http.Request) {
	claims := GetUserClaims(r.Context())
	writeJSON(w, http.StatusOK, map[string]interface{}{
		"message":  "welcome to the admin dashboard",
		"admin_id": claims.UserID,
	})
}

// =============================================================================
// Helpers
// =============================================================================

func writeJSON(w http.ResponseWriter, status int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(v)
}

func chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
	for i := len(middlewares) - 1; i >= 0; i-- {
		handler = middlewares[i](handler)
	}
	return handler
}

// =============================================================================
// Main
// =============================================================================

func main() {
	cfg := LoadConfig()

	// Initialize JWT issuer
	seed, err := hex.DecodeString(cfg.JWTSeedHex)
	if err != nil {
		log.Fatalf("Invalid JWT seed hex: %v", err)
	}

	jwtIssuer, err := NewJWTIssuer(seed, cfg.JWTIssuer, cfg.AccessTTL, cfg.RefreshTTL)
	if err != nil {
		log.Fatalf("Failed to create JWT issuer: %v", err)
	}

	// Initialize repositories
	userRepo := NewInMemoryUserRepo()
	tokenRepo := NewInMemoryTokenRepo()

	// Seed an admin user for demo
	adminName := "Admin User"
	userRepo.Create(&User{
		ID:        "admin_001",
		Email:     "admin@example.com",
		Name:      &adminName,
		Roles:     []string{"admin", "editor", "viewer"},
		CreatedAt: time.Now(),
	})

	// Initialize handlers
	authHandler := &AuthHandler{
		jwtIssuer: jwtIssuer,
		userRepo:  userRepo,
		tokenRepo: tokenRepo,
		config:    cfg,
	}

	// Start token cleanup goroutine
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	go func() {
		ticker := time.NewTicker(1 * time.Hour)
		defer ticker.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-ticker.C:
				deleted := tokenRepo.DeleteExpired()
				if deleted > 0 {
					log.Printf("Cleaned up %d expired refresh tokens", deleted)
				}
			}
		}
	}()

	// Set up routes
	mux := http.NewServeMux()

	// Public endpoints
	mux.HandleFunc("/auth/login", authHandler.HandleDemoLogin)
	mux.HandleFunc("/auth/refresh", authHandler.HandleRefresh)
	mux.HandleFunc("/auth/logout", authHandler.HandleLogout)

	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
	})

	// Authenticated endpoints
	mux.Handle("/api/me", chain(
		http.HandlerFunc(HandleMe),
		AuthMiddleware(jwtIssuer),
	))

	// Admin-only endpoints
	mux.Handle("/api/admin/dashboard", chain(
		http.HandlerFunc(HandleAdminDashboard),
		AuthMiddleware(jwtIssuer),
		RequireRole("admin"),
	))

	// Create server with timeouts
	server := &http.Server{
		Addr:         cfg.ListenAddr,
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  120 * time.Second,
	}

	// Graceful shutdown
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
		<-sigCh

		log.Println("Shutting down...")
		cancel() // Stop token cleanup goroutine

		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer shutdownCancel()

		if err := server.Shutdown(shutdownCtx); err != nil {
			log.Fatalf("Shutdown error: %v", err)
		}
	}()

	// Start server
	log.Printf("Server starting on %s", cfg.ListenAddr)
	log.Println()
	log.Println("Available endpoints:")
	log.Println("  POST /auth/login          -- Login (demo: send {\"email\": \"...\", \"name\": \"...\"})")
	log.Println("  POST /auth/refresh        -- Refresh access token")
	log.Println("  POST /auth/logout         -- Logout")
	log.Println("  GET  /api/me              -- Get current user (requires auth)")
	log.Println("  GET  /api/admin/dashboard -- Admin dashboard (requires admin role)")
	log.Println("  GET  /health              -- Health check")
	log.Println()
	log.Println("Example usage:")
	log.Println("  curl -X POST http://localhost:8080/auth/login \\")
	log.Println("    -H 'Content-Type: application/json' \\")
	log.Println("    -d '{\"email\":\"admin@example.com\",\"name\":\"Admin\"}' \\")
	log.Println("    -c cookies.txt")
	log.Println()
	log.Println("  curl http://localhost:8080/api/me -b cookies.txt")
	log.Println("  curl http://localhost:8080/api/admin/dashboard -b cookies.txt")
	log.Println()

	if err := server.ListenAndServe(); err != http.ErrServerClosed {
		log.Fatalf("Server error: %v", err)
	}
	log.Println("Server stopped")
}
```

### Testing the Complete System

```bash
# 1. Start the server
go run main.go

# 2. Login (creates user + issues tokens as cookies)
curl -X POST http://localhost:8080/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@example.com","name":"Admin"}' \
  -c cookies.txt -v

# 3. Access protected endpoint
curl http://localhost:8080/api/me -b cookies.txt

# 4. Access admin endpoint
curl http://localhost:8080/api/admin/dashboard -b cookies.txt

# 5. Refresh tokens
curl -X POST http://localhost:8080/auth/refresh -b cookies.txt -c cookies.txt

# 6. Logout
curl -X POST http://localhost:8080/auth/logout -b cookies.txt -c cookies.txt

# 7. Verify access is revoked
curl http://localhost:8080/api/me -b cookies.txt
# Should return 401
```

---

## 18. Key Takeaways

### Authentication Architecture

1. **The hybrid pattern** (external IdP + internal JWT) separates concerns cleanly. The identity provider handles user passwords and MFA. Your backend only handles authorization tokens.

2. **Token pairs** (access + refresh) balance security and usability. Short-lived access tokens limit the damage window. Long-lived refresh tokens avoid forcing users to re-login constantly.

3. **Ed25519** is the modern choice for JWT signing. It is faster, smaller, and more secure than RSA. It is asymmetric, so verification keys can be safely distributed.

### Security Essentials

4. **Always validate the signing algorithm** when parsing JWTs. The algorithm confusion attack is one of the most common JWT vulnerabilities.

5. **Store refresh tokens hashed** (SHA-256) in the database. If the database is compromised, raw tokens would let attackers impersonate any user.

6. **Rotate refresh tokens** on every use and implement reuse detection. If a previously-used token is presented, revoke all tokens for that user.

7. **Use httpOnly, Secure, SameSite cookies** for token delivery. This prevents XSS from stealing tokens and mitigates CSRF.

### Go-Specific Patterns

8. **Context is king** for propagating user identity. Auth middleware injects claims into the context; handlers and service layers extract them without coupling.

9. **Interfaces for testability.** Define `TokenIssuer` and `TokenRepository` interfaces so you can mock them in tests without hitting real infrastructure.

10. **Middleware composition** in Go is explicit and readable. Unlike Passport.js strategies or Express middleware chains, Go middleware is just function wrapping -- no magic, no hidden state.

### Comparison with Node.js

| Aspect | Go | Node.js |
|--------|-----|---------|
| JWT library | `golang-jwt/jwt/v5` | `jsonwebtoken` (no EdDSA) / `jose` |
| Ed25519 | Standard library `crypto/ed25519` | Requires `jose` or `noble-ed25519` |
| OIDC | `coreos/go-oidc/v3` | `openid-client` or `google-auth-library` |
| Cookie handling | `net/http` (standard library) | `cookie-parser` middleware |
| Auth middleware | ~30 lines, explicit | Passport.js (framework + strategies) |
| Context propagation | `context.Context` (built-in) | `req.user` (attached by middleware) |
| Type safety | Claims are typed structs | Claims are `any` objects |
| Testing | `httptest` (standard library) | `supertest` + `jest` |

---

## 19. Practice Exercises

### Exercise 1: Token Blacklist

Implement a token blacklist that allows revoking access tokens before they expire. Requirements:

- Add a `jti` (JWT ID) to access tokens
- Create an `AccessTokenBlacklist` that stores revoked token IDs
- Modify the auth middleware to check the blacklist
- Add a `POST /auth/revoke` endpoint that blacklists the current access token
- Use a sync.Map or mutex-protected map for concurrent access
- Include a TTL-based cleanup to remove entries for tokens that have naturally expired

**Hint:** The blacklist only needs to store token IDs, not the full tokens. Entries can be removed after the token's expiry time passes.

### Exercise 2: Multi-Device Session Management

Build a session management system where users can see and revoke individual sessions. Requirements:

- Track each refresh token with metadata: device name, IP address, last used time
- Add a `GET /api/sessions` endpoint that lists all active sessions for the current user
- Add a `DELETE /api/sessions/:id` endpoint that revokes a specific session
- Add a `DELETE /api/sessions` endpoint that revokes all sessions except the current one
- Include the session ID in the access token claims so handlers know which session is current

### Exercise 3: API Key Authentication

Add a second authentication method: API keys for programmatic access. Requirements:

- Design an `api_keys` table with: key hash, user ID, scopes, created at, expires at, last used
- Generate API keys as `myapp_sk_<random-hex>` (with a recognizable prefix)
- Modify the auth middleware to check for API keys in the `X-API-Key` header
- API key requests should set `UserClaims` in the context just like JWT auth
- Implement rate limiting per API key
- Add CRUD endpoints for managing API keys (create, list, revoke)

### Exercise 4: Scope-Based Authorization

Extend the role-based system with fine-grained scopes. Requirements:

- Define scopes like `posts:read`, `posts:write`, `users:read`, `users:admin`
- Map roles to sets of scopes (e.g., `admin` has all scopes, `editor` has `posts:*`)
- Add scopes to the access token claims
- Create a `RequireScope("posts:write")` middleware
- Support wildcard scopes (`posts:*` matches `posts:read` and `posts:write`)
- Write table-driven tests for scope matching logic

### Exercise 5: Key Rotation

Implement JWT signing key rotation. Requirements:

- Support multiple active signing keys, each identified by a `kid` (Key ID) in the JWT header
- The most recent key is used for signing new tokens
- All active keys can be used for verification (so tokens signed with old keys still work)
- Add a `/well-known/jwks.json` endpoint that publishes public keys in JWKS format
- Implement a `RotateKey()` method that generates a new key pair and makes it the active signing key
- Old keys should be kept for verification until all tokens signed with them have expired
- Write tests for the rotation lifecycle: issue with key A, rotate to key B, verify tokens from both

### Exercise 6: Integration Test Suite

Write a comprehensive integration test for the complete auth system from Section 17. Requirements:

- Use `httptest.NewServer` to run the full server in tests
- Test the complete flow: login -> access protected resource -> token refresh -> access again -> logout -> verify access denied
- Test edge cases: expired tokens, tampered tokens, revoked refresh tokens, reuse detection
- Test RBAC: viewer cannot access admin endpoints, admin can
- Use table-driven tests for the middleware test cases
- Achieve >90% code coverage on the auth package

---

*This chapter covered building a production authentication system from the ground up. The patterns shown here -- hybrid OIDC validation, Ed25519 JWT signing, access/refresh token pairs, httpOnly cookie delivery, middleware-based auth, and role-based authorization -- form the foundation of secure Go API authentication. In the next chapter, we will build on this foundation to implement real-time features with WebSockets and Server-Sent Events.*
