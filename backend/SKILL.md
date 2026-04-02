---
name: backend
description: "Build backend services in Go using stdlib net/http. Takes an OpenAPI spec from the software-architecture skill and scaffolds a working Go project. Covers REST APIs, database layer (PostgreSQL + JSONB), object storage (SeaweedFS or cloud), auth (PKCE, client credentials, API keys), and deployment (Docker, K8s, or cloud). Evaluates every default against project requirements before committing. Use this skill when the user asks to implement a backend, build an API, or scaffold a Go service."
---

# Backend Skill

## When to Skip

- The app is pure frontend (static site, client-side only)
- Apps Script handles all logic — go to `google-apps-script`
- The backend already exists and the user just needs frontend or mobile work

## When to Use

- New API or backend service from scratch
- Scaffolding a Go project from an OpenAPI spec
- Adding a new backend service to an existing system

---

Build production backend services in Go. Default stack: Go stdlib `net/http` (1.22+), PostgreSQL, Docker. Always evaluate whether alternatives are a better fit before committing to defaults.

## Step 0: Check for Architecture Output

If the `software-architecture` skill has already produced an OpenAPI spec and design document, read them first:
- The OpenAPI spec defines your routes, request/response schemas, and auth requirements
- The design document defines your database choice, deployment target, and integration points

If no architecture output exists, gather requirements directly from the user before proceeding.

## Step 1: Evaluate Every Default

Defaults exist to move fast — but every default must be explicitly evaluated against the project's actual requirements before committing. Walk through each decision below. State your evaluation briefly: confirm the default fits or recommend an alternative with a reason.

### API Style

| Option | Default? | When to use |
|--------|----------|-------------|
| **REST** | Yes | CRUD operations, broad client support, OpenAPI-driven. Most apps. |
| **GraphQL** | No | Multiple clients needing very different data shapes, and a BFF isn't justified |
| **gRPC** | No | Internal service-to-service where latency matters. Not for browser clients without a proxy. |
| **WebSocket** | Alongside REST | Real-time features (chat, live updates, notifications) |

### Language

| Option | Default? | When to use |
|--------|----------|-------------|
| **Go** | Yes | APIs, microservices, CLI tools, anything deploying to containers/K8s |
| **Python (FastAPI)** | No | ML/data-heavy backends, rapid prototyping, when the team is Python-only |
| **Apps Script** | No | Simple Workspace automation — defer to `google-apps-script` skill |

### Database

| Option | Default? | When to use |
|--------|----------|-------------|
| **PostgreSQL** | Yes | Structured relational data. Also handles unstructured data via JSONB columns — covers 90% of "flexible schema" needs without a second database. |
| **PostgreSQL JSONB** | Yes (for unstructured) | Document-style data that coexists with relational data. Indexable, queryable, transactional. Avoids running a second database. |
| **MongoDB** | No | Only when data is deeply nested with unknown structure at massive scale and Postgres JSONB genuinely doesn't fit |

### Object Storage

| Option | Default? | When to use |
|--------|----------|-------------|
| **SeaweedFS** | Yes (self-hosted) | File uploads, documents, ML artifacts. S3-compatible API. |
| **GCS / Cloud Storage** | Yes (GCP) | When deploying to GCP — managed, no operational overhead |
| **S3** | Yes (AWS) | When deploying to AWS |

### Auth

| Option | Default? | When to use |
|--------|----------|-------------|
| **OAuth 2.0 + PKCE** | Yes (user-facing) | Browser SPAs, mobile apps, any public client |
| **Client Credentials** | Yes (service-to-service) | Internal services calling each other, no user identity |
| **API Keys** | Yes (external) | Webhooks, ERP pushes, third-party integrations |
| **Google Identity Platform** | No | When on GCP and want managed auth |
| **Cognito** | No | When on AWS and want managed auth |

### Deployment Target

| Option | Default? | When to use |
|--------|----------|-------------|
| **Docker / Docker Compose** | Yes (local dev) | Always for local development |
| **Kubernetes (self-hosted)** | Yes (production, self-hosted) | When running own infrastructure (CITADEL stack) |
| **Cloud Run (GCP)** | No | Stateless APIs on GCP, auto-scaling, no K8s overhead |
| **ECS / Fargate (AWS)** | No | Container workloads on AWS without managing K8s |
| **GKE / EKS** | No | When you need K8s but want managed control plane |

### Supporting Services

| Concern | Self-hosted default | Cloud alternative | When to switch |
|---------|-------------------|-------------------|----------------|
| Cache | Redis | GCP Memorystore / AWS ElastiCache | Managed infra, less ops |
| Message queue | NATS | GCP Pub/Sub / AWS SQS | Managed infra, event-driven architecture |
| Secrets | Env vars | GCP Secret Manager / AWS Secrets Manager / Vault | Production with rotation requirements |
| Search | PostgreSQL full-text | Elasticsearch / Meilisearch | Search is a core feature with complex queries |

### Self-Hosted vs Cloud Evaluation

For **every infrastructure component** (database, storage, auth, cache, queue, secrets), evaluate self-hosted vs cloud-managed using these criteria:

| Criterion | Questions to ask |
|-----------|-----------------|
| **Operational complexity** | Do we have the expertise and bandwidth to run this ourselves? What's the maintenance burden (upgrades, backups, monitoring, failover)? |
| **Cost — money** | What's the managed service pricing at our expected scale? Is it predictable or usage-based with surprise potential? How does it compare to the infrastructure cost of self-hosting? |
| **Cost — engineering time** | How many hours/month does self-hosting consume? Could that time be spent on product work instead? |
| **Vendor lock-in** | Does the managed service use a standard protocol (SQL, S3 API) or a proprietary API? How hard is migration if we need to leave? |
| **Scale fit** | Is our scale too small to justify managed service costs? Too large to self-host reliably? |
| **Data sovereignty** | Does the data need to stay in a specific region or on-premise? Does the cloud service support that? |

**Decision rule:** Default to self-hosted when using standard protocols (PostgreSQL, S3-compatible storage) because migration is easy in both directions. Consider cloud-managed when operational complexity outweighs the cost, or when the cloud service offers capabilities hard to replicate (global distribution, serverless scaling, real-time sync).

**Cloud-native databases** (DynamoDB, Firestore, Cosmos DB) are a different category — they offer unique capabilities (serverless scale, real-time sync, global distribution) but come with proprietary APIs and hard lock-in. Only recommend when a specific capability justifies the lock-in and the project is committed to that cloud provider.

### Portability Rule

Code against **interfaces and standard protocols** so cloud vs self-hosted is a deployment decision, not a code change:
- PostgreSQL is PostgreSQL whether on K8s or Cloud SQL/RDS
- SeaweedFS, GCS, and S3 all speak the S3 protocol
- JWT validation works the same regardless of who issued the token
- Only config (endpoints, credentials) should change between environments
- When a cloud-native proprietary service is chosen, isolate it behind an interface so the rest of the codebase doesn't depend on it directly

## Step 2: Project Structure

Scaffold from the OpenAPI spec. The structure follows standard Go project layout:

```
project-name/
├── cmd/
│   └── server/
│       └── main.go              # Entry point, server setup, graceful shutdown
├── internal/
│   ├── api/
│   │   ├── router.go            # Route definitions (from OpenAPI paths)
│   │   ├── middleware.go         # Auth, logging, CORS, request ID
│   │   └── handlers/
│   │       ├── health.go        # GET /health, GET /ready
│   │       └── {resource}.go    # One file per API resource group
│   ├── models/
│   │   └── {resource}.go        # Structs from OpenAPI schemas
│   ├── store/
│   │   ├── store.go             # Store interface
│   │   └── postgres.go          # PostgreSQL implementation (incl. JSONB)
│   ├── service/
│   │   └── {resource}.go        # Business logic between handlers and store
│   └── config/
│       └── config.go            # Env-based configuration
├── migrations/
│   └── 001_initial.sql          # Database migrations (PostgreSQL)
├── Dockerfile
├── docker-compose.yml
├── go.mod
├── go.sum
├── .env.example
└── README.md
```

### Key Principles

- **`internal/`** — all application code lives here. Not importable by other projects. This is intentional.
- **Handler → Service → Store** — handlers parse HTTP, services contain business logic, store handles data access. No skipping layers.
- **One file per resource group** — `handlers/tasks.go` contains all task-related handlers. Don't split into one file per endpoint.
- **Interfaces at boundaries** — `store.Store` is an interface. Implementations are swappable.

## Step 3: Server Setup

Entry point pattern with graceful shutdown:

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "project/internal/api"
    "project/internal/config"
    "project/internal/store"
)

func main() {
    cfg := config.Load()

    db, err := store.NewPostgres(cfg.DatabaseURL)
    if err != nil {
        slog.Error("failed to connect to database", "error", err)
        os.Exit(1)
    }
    defer db.Close()

    router := api.NewRouter(db, cfg)

    srv := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      router,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    go func() {
        slog.Info("server starting", "port", cfg.Port)
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            slog.Error("server error", "error", err)
            os.Exit(1)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
    slog.Info("server stopped")
}
```

## Step 4: Routing

Use Go 1.22+ stdlib routing with method patterns:

```go
func NewRouter(store store.Store, cfg *config.Config) http.Handler {
    mux := http.NewServeMux()

    // Health
    mux.HandleFunc("GET /health", handleHealth)
    mux.HandleFunc("GET /ready", handleReady(store))

    // API routes — derived from OpenAPI paths
    mux.HandleFunc("GET /api/v1/tasks", handleListTasks(store))
    mux.HandleFunc("POST /api/v1/tasks", handleCreateTask(store))
    mux.HandleFunc("GET /api/v1/tasks/{id}", handleGetTask(store))
    mux.HandleFunc("PUT /api/v1/tasks/{id}", handleUpdateTask(store))
    mux.HandleFunc("DELETE /api/v1/tasks/{id}", handleDeleteTask(store))

    // Apply middleware
    var handler http.Handler = mux
    handler = withRequestID(handler)
    handler = withLogging(handler)
    handler = withCORS(handler, cfg.AllowedOrigins)
    handler = withAuth(handler, cfg.JWKSUrl)

    return handler
}
```

### Handler Pattern

Handlers are closures that receive dependencies:

```go
func handleGetTask(s store.Store) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")

        task, err := s.GetTask(r.Context(), id)
        if err != nil {
            writeError(w, http.StatusNotFound, "task not found")
            return
        }

        writeJSON(w, http.StatusOK, task)
    }
}
```

### JSON Helpers

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, message string) {
    writeJSON(w, status, map[string]string{"error": message})
}

func readJSON(r *http.Request, dst any) error {
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()
    return dec.Decode(dst)
}
```

## Step 5: Database

### PostgreSQL (default — structured/relational data)

Use `pgx` as the driver (pure Go, best performance, connection pooling built in):

```go
import "github.com/jackc/pgx/v5/pgxpool"

func NewPostgres(databaseURL string) (*pgxpool.Pool, error) {
    pool, err := pgxpool.New(context.Background(), databaseURL)
    if err != nil {
        return nil, err
    }
    if err := pool.Ping(context.Background()); err != nil {
        return nil, err
    }
    return pool, nil
}
```

Migrations: plain SQL files in `migrations/`, applied in order. Use `golang-migrate/migrate` or apply manually. No ORM — write SQL directly.

### PostgreSQL JSONB (unstructured data in the same database)

Use when: flexible schemas, nested documents, mixed structured + unstructured data. No second database needed.

```sql
-- Migration: add a JSONB column for flexible metadata
ALTER TABLE resources ADD COLUMN metadata JSONB DEFAULT '{}';

-- Create a GIN index for fast JSONB queries
CREATE INDEX idx_resources_metadata ON resources USING GIN (metadata);

-- Query JSONB fields
SELECT * FROM resources WHERE metadata->>'category' = 'electronics';
SELECT * FROM resources WHERE metadata @> '{"tags": ["urgent"]}';
```

```go
// In Go, use map[string]any or a typed struct for JSONB columns
type Resource struct {
    ID       string         `json:"id"`
    Name     string         `json:"name"`
    Metadata map[string]any `json:"metadata"`
}

// pgx handles JSONB serialization automatically
row := pool.QueryRow(ctx,
    "INSERT INTO resources (name, metadata) VALUES ($1, $2) RETURNING id",
    resource.Name, resource.Metadata,
)
```

### SeaweedFS (object/file storage)

Use for: file uploads, document storage, ML artifacts, any binary blob storage. SeaweedFS exposes an S3-compatible API.

```go
import "github.com/aws/aws-sdk-go-v2/service/s3"

// Connect via S3-compatible client pointing at SeaweedFS endpoint
func NewSeaweedFS(endpoint, accessKey, secretKey string) (*s3.Client, error) {
    cfg, err := config.LoadDefaultConfig(context.Background(),
        config.WithRegion("us-east-1"),
        config.WithCredentialsProvider(
            credentials.NewStaticCredentialsProvider(accessKey, secretKey, ""),
        ),
    )
    if err != nil {
        return nil, err
    }
    client := s3.NewFromConfig(cfg, func(o *s3.Options) {
        o.BaseEndpoint = aws.String(endpoint)
        o.UsePathStyle = true
    })
    return client, nil
}
```

### Choosing

PostgreSQL handles both structured (tables) and unstructured (JSONB) data. Default to a single PostgreSQL instance unless there's a clear reason to add another database.

Add SeaweedFS (or cloud object storage) alongside PostgreSQL when the app handles file uploads or binary data.

## Step 6: Authentication & Authorization

### OAuth 2.0 + PKCE (user-facing apps)

The frontend handles the OAuth dance. The backend validates tokens:

```go
func withAuth(next http.Handler, jwksURL string) http.Handler {
    // Fetch JWKS from identity provider (Google, Auth0, etc.)
    keySet, _ := jwk.Fetch(context.Background(), jwksURL)

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Skip auth for public endpoints
        if isPublicPath(r.URL.Path) {
            next.ServeHTTP(w, r)
            return
        }

        token := extractBearerToken(r)
        if token == "" {
            writeError(w, http.StatusUnauthorized, "missing token")
            return
        }

        claims, err := validateJWT(token, keySet)
        if err != nil {
            writeError(w, http.StatusUnauthorized, "invalid token")
            return
        }

        ctx := context.WithValue(r.Context(), ctxUserKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Client Credentials (service-to-service)

For internal services calling each other. Backend validates the token the same way — the difference is the token has no user identity, just a service identity and scopes.

### API Keys (external integrations)

For webhooks, ERP pushes, third-party integrations:

```go
func withAPIKey(next http.Handler, validKeys map[string]string) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := r.Header.Get("X-API-Key")
        if key == "" {
            writeError(w, http.StatusUnauthorized, "missing API key")
            return
        }
        clientID, ok := validKeys[key]
        if !ok {
            writeError(w, http.StatusUnauthorized, "invalid API key")
            return
        }
        ctx := context.WithValue(r.Context(), ctxClientKey, clientID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Auth Decision Table

| Client | Auth method |
|--------|-------------|
| Browser SPA / mobile app | OAuth 2.0 Authorization Code + PKCE |
| Another internal service | OAuth 2.0 Client Credentials |
| External system (SAP, webhooks) | API Key |
| CLI tool (user-operated) | OAuth 2.0 Device Code flow |

## Step 7: Middleware Stack

Apply in this order (outermost first):

1. **Recovery** — catch panics, return 500
2. **Request ID** — generate UUID, add to context and response header
3. **CORS** — allow configured origins
4. **Logging** — structured logging with `slog` (method, path, status, duration)
5. **Auth** — validate token, inject user/client into context
6. **Rate limiting** — if needed, per-client token bucket

```go
// Middleware signature
type Middleware func(http.Handler) http.Handler

// Chain applies middlewares in order
func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}
```

## Step 8: Configuration

Environment variables only. No config files, no flags for production settings:

```go
type Config struct {
    Port            string
    DatabaseURL     string
    MongoURL        string   // if using MongoDB
    SeaweedEndpoint string   // if using SeaweedFS
    JWKSUrl         string
    AllowedOrigins  []string
    LogLevel        string
}

func Load() *Config {
    return &Config{
        Port:           getEnv("PORT", "8080"),
        DatabaseURL:    requireEnv("DATABASE_URL"),
        JWKSUrl:        requireEnv("JWKS_URL"),
        AllowedOrigins: strings.Split(getEnv("ALLOWED_ORIGINS", "*"), ","),
        LogLevel:       getEnv("LOG_LEVEL", "info"),
    }
}
```

## Step 9: Docker & Deployment

### Dockerfile (multi-stage)

```dockerfile
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

FROM scratch
COPY --from=build /app /app
COPY --from=build /src/migrations /migrations
EXPOSE 8080
ENTRYPOINT ["/app"]
```

### docker-compose.yml (local dev)

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 2s
      timeout: 5s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Add MongoDB or SeaweedFS services as needed.

## Step 10: Deliver

Present deliverables in this order:
1. **Stack evaluation** — brief justification of choices
2. **Scaffolded Go project** — all files written to disk, compiles and runs
3. **Database migrations** — initial schema
4. **Docker setup** — Dockerfile + docker-compose for local dev
5. **README** — how to run locally, environment variables, API overview

List all files created at the end.

## Rules

**Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

| Do | Don't |
|----|-------|
| Use Go stdlib `net/http` for routing (1.22+) | Add Chi, Echo, Gin, or other routers |
| Write SQL directly with `pgx` | Use an ORM (GORM, ent) |
| Use `slog` for structured logging | Use `log`, `logrus`, `zap` |
| Use `internal/` for all app code | Put packages in the project root |
| Return JSON errors with consistent shape | Return plain text errors |
| Use environment variables for config | Use config files or CLI flags |
| Validate at the HTTP boundary (handlers) | Validate inside the store layer |
| Use `context.Context` for cancellation and values | Use global state |
| Keep handlers thin — delegate to services | Put business logic in handlers |
| Use `scratch` base image in production | Use full OS images |

## Handoff

### Input
- OpenAPI spec and design document from `software-architecture` skill
- Database choice, auth pattern, deployment target decisions

### Output
- Scaffolded Go project (compiles and runs)
- Database migrations
- Dockerfile + docker-compose.yml
- README with setup instructions

### Next skills

| Next | What it receives |
|------|-----------------|
| `frontend` | The API contract to code the frontend against. Reads `design-language.md` for styling. |
| `testing` | The Go project to add unit, integration, and API contract tests to |
| `deployment` | The Dockerized app ready for dev or prod deployment |

## Continuous Improvement

### Skill Health Check

At the end of every use, evaluate and report if any of the following apply:

- **Skill gap:** "This project needs [X] and no current skill covers it. Consider creating a `[name]` skill."
- **Skill split:** "This skill's [section] is growing complex enough to be its own skill." — e.g., if Python/FastAPI becomes a frequent alternative, it may warrant a `backend-python` skill.
- **Skill overlap:** "This skill and `[other skill]` both cover [topic]. Consider consolidating."

Only report when genuinely applicable. Don't force observations.

### Learnings

Corrections and refinements discovered during use. When the user overrides a recommendation or a default doesn't fit, record it here so future uses benefit.

_(Empty — will accumulate over time)_
