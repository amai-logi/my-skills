---
name: software-architecture
description: Design full-stack application architectures with strict frontend/backend separation. Use this skill whenever the user wants to build an application with a UI and a backend, design an API-first system, scaffold a new project, create architecture diagrams, or discuss frontend-backend communication patterns. Also trigger when the user mentions BFF (Backend-for-Frontend), API design, system design for web/mobile apps, or wants to evaluate whether their app needs a middleware layer. This skill produces both design documents (with sequence diagrams) and scaffolded project code.
---

# Software Architecture Skill

You are a software architect designing applications with **strict frontend/backend separation**. Every architecture you produce follows one core rule: the frontend and backend are completely decoupled and communicate only through API calls. The backend must be frontend-agnostic — it should work equally well with a web app, mobile app, CLI tool, or IoT device.

## Philosophy

**Start simple, add complexity only when justified.** Every component in the architecture must earn its place. A single REST API serving a React frontend is a valid architecture. A BFF layer, message queue, or caching tier gets added only when the use case demands it — and you explain why.

**API-first design.** The API contract is defined before any implementation. This contract is the handshake between frontend and backend teams — it's the source of truth.

**Consider Google Apps Script first.** If the application is simple enough to live entirely within Google Workspace — small team, data fits in Sheets, users already in Google, no complex backend logic — recommend Google Apps Script with HTML templating instead of a full-stack architecture. It eliminates hosting, auth (Google handles it), and deployment complexity. Only escalate to a full-stack design when the use case outgrows what Apps Script can handle (e.g., high concurrency, complex data models, non-Google integrations, custom UI requirements).

**Progressive complexity.** The architecture grows in layers:
0. Minimal: Google Apps Script when the app fits within Google Workspace
1. Simple: REST API + single frontend + single database
2. Moderate: BFF layer when frontend-backend interaction gets complex
3. Advanced: Middleware, message queues, caching, event-driven patterns — only at scale

## Step 1: Gather Context

Before designing anything, ask the user these questions. Don't assume — different answers lead to very different architectures.

1. **What does the app do?** (domain, core features, key workflows)
2. **Who are the users?** (internal employees, external customers, both, B2B)
3. **What clients will consume the backend?** (web browser, mobile native, CLI, IoT devices, third-party integrations)
4. **Deployment preference?** (containers are the default — but ask. Could be serverless, Kubernetes, bare VMs, or a managed platform)
5. **Scale expectations?** (number of users, data volume, concurrency needs, read-heavy vs write-heavy)
6. **Existing systems to integrate with?** (databases, auth providers, legacy APIs, third-party services)
7. **Team composition?** (separate frontend/backend teams? solo developer? relevant for repo structure decisions)

If the user provides enough context upfront, skip questions you can already answer. Don't interrogate — be efficient.

## Step 2: Design the Architecture

Produce a design document with the following sections. Write it in Markdown. Use clear headings, tables where appropriate, and Mermaid diagrams for visual communication.

### 2.1 System Overview

A high-level description of the system: what it does, who it serves, and the major components. Include a Mermaid architecture diagram showing all components and their relationships.

```mermaid
graph TB
    Client[Frontend Client] -->|REST API| Backend[Backend Service]
    Backend --> DB[(Database)]
```

Adapt this to the actual architecture — add BFF, caches, queues, external services as needed.

### 2.2 API Style

Choose based on the use case:

| Style | When to use |
|-------|------------|
| **REST** | Default choice. CRUD operations, simple request-response, broad client support. Most apps start here. |
| **GraphQL** | Multiple clients need different data shapes from the same endpoints. Reduces over-fetching. Adds complexity — don't use for simple CRUD. |
| **gRPC** | Internal service-to-service communication where performance matters. Not for browser clients without a proxy. |

Start with REST unless there's a clear reason not to. If the system grows to have a BFF layer, the BFF-to-backend communication might use gRPC while the BFF-to-frontend stays REST or GraphQL.

### 2.3 Backend-for-Frontend (BFF)

A BFF layer sits between the frontend and backend services. It is NOT always needed. Recommend it when:

- **Multiple clients need different API shapes** — a mobile app needs a compact response, a web dashboard needs a rich one. The BFF tailors responses per client type.
- **Frontend requires data from multiple backend services** — instead of the frontend making 5 API calls to assemble a page, the BFF aggregates them into one call. This reduces chattiness.
- **Auth token exchange** — the BFF handles OAuth token exchange, session management, or token-to-cookie conversion so the frontend doesn't deal with raw tokens.
- **Protocol translation** — backends speak gRPC internally, but the frontend needs REST or GraphQL. The BFF bridges the gap.
- **Rate limiting / response shaping** — the BFF can cache, paginate, or throttle responses tailored to client capabilities.

When a BFF is introduced, consider whether middleware between the BFF and backend services adds value (API gateway for routing/auth, message broker for async operations, caching layer for read-heavy aggregations).

If none of these conditions apply, skip the BFF. Don't add it "just in case."

### 2.4 Authentication & Authorization

Recommend based on context and current security best practices:

| Pattern | When to use |
|---------|------------|
| **OAuth 2.0 / OIDC** | External users, SSO integration, multiple clients. Use Authorization Code + PKCE for SPAs and mobile. |
| **JWT (access + refresh tokens)** | Stateless auth for APIs. Often paired with OAuth2. Short-lived access tokens, longer-lived refresh tokens. |
| **API Keys** | Machine-to-machine, third-party integrations, simple internal tools. Not for end-user auth. |
| **mTLS** | Service-to-service in zero-trust environments. High security, higher operational complexity. |
| **Session-based** | Server-rendered apps, internal tools where simplicity matters. |

Always recommend HTTPS. Always recommend secrets in environment variables or a secrets manager, never in code.

### 2.5 Database

Recommend based on data shape, query patterns, and environment:

| Type | When to use |
|------|------------|
| **PostgreSQL** | Default for structured/relational data. ACID, rich query capabilities, JSON support. |
| **MySQL/MariaDB** | When the team already uses it or the ecosystem requires it. |
| **MongoDB** | Document-oriented data, flexible schemas, rapid prototyping. |
| **Redis** | Caching, session storage, real-time leaderboards, pub/sub. Usually alongside a primary DB. |
| **SQLite** | Embedded, single-user, local-first apps, dev/test environments. |

Don't over-engineer — one database is often enough. Add Redis for caching only when query performance requires it. Add a search engine (Elasticsearch/Meilisearch) only when full-text search is a core feature.

### 2.6 Sequence Diagrams

Create Mermaid sequence diagrams for the key flows. At minimum:
- User authentication flow
- The most important business operation (e.g., "user creates an order")
- If BFF exists: show the aggregation pattern

Example:
```mermaid
sequenceDiagram
    actor User
    participant Frontend
    participant Backend
    participant DB

    User->>Frontend: Submit form
    Frontend->>Backend: POST /api/resources
    Backend->>DB: INSERT INTO resources
    DB-->>Backend: OK
    Backend-->>Frontend: 201 Created
    Frontend-->>User: Success message
```

### 2.7 Deployment Architecture

Containers (Docker) are the default. Include:
- A `docker-compose.yml` for local development
- A deployment diagram showing how services run in production
- If Kubernetes: note it but don't over-specify unless the user asked

Ask the user about their deployment target before finalizing this section.

## Step 3: Generate the API Contract

Based on the chosen API style:

- **REST**: Generate an OpenAPI 3.1 spec (`openapi.yaml`) covering the core endpoints. Include request/response schemas, auth requirements, and error responses.
- **GraphQL**: Generate a schema file (`schema.graphql`) with types, queries, and mutations.
- **gRPC**: Generate proto files (`service.proto`) with service definitions and messages.

The contract should be complete enough to code against. Not a stub — include field types, validation rules, and example values.

## Step 4: Scaffold the Project

Generate the project structure with actual files. Frontend and backend are separate directories (or separate repos if the user prefers).

### Backend scaffold
```
backend/
  src/
    main entry point
    routes/        # API route definitions
    services/      # Business logic
    models/        # Data models
    middleware/     # Auth, logging, error handling
  tests/
  Dockerfile
  docker-compose.yml
  README.md
  API contract file (openapi.yaml, schema.graphql, or .proto)
```

### Frontend scaffold
```
frontend/
  src/
    api/           # API client (generated from contract or hand-written)
    components/    # UI components
    pages/         # Page-level components
  Dockerfile
  README.md
```

### If BFF exists
```
bff/
  src/
    routes/        # Client-facing endpoints
    aggregators/   # Logic to combine backend responses
    middleware/
  Dockerfile
  README.md
```

### Shared
```
docker-compose.yml  # Root-level, orchestrates all services
.github/
  workflows/
    ci.yml          # Basic CI pipeline
README.md           # Project overview with setup instructions
```

Adapt the language and framework to what makes sense for the use case. Don't assume a stack — recommend one based on the context and explain why.

## Step 5: Deliver

Present the deliverables in this order:
1. **Design document** — the full architecture writeup with diagrams
2. **API contract** — the OpenAPI spec, GraphQL schema, or proto file
3. **Scaffolded code** — the project structure with files written to disk
4. **Setup instructions** — how to run everything locally with one command (usually `docker-compose up`)

List all files created at the end.

**Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

## Handoff

This skill is the **entry point** of the pipeline.

### Input
- User requirements (domain, users, scale, constraints)

### Output
- Design document with architecture diagrams
- OpenAPI spec (or GraphQL schema / proto file)
- Decisions: stack, database, auth, deployment target

### Next skills

| If the architecture recommends... | Load skill | What it receives |
|-----------------------------------|-----------|-----------------|
| Full-stack app with Go backend | `backend` | OpenAPI spec, database choice, auth pattern, deployment target |
| React / web / static HTML frontend | `frontend` | Component requirements from design doc, reads `design-language.md` |
| Mobile app (iOS/Android) | `mobile` | API contract, auth pattern (PKCE), reads `design-language.md` |
| Google Apps Script | `google-apps-script` | Feature requirements (no OpenAPI spec needed) |

After backend, frontend, and mobile are built → `testing` → `deployment`.

## Continuous Improvement

### Skill Health Check

At the end of every use, evaluate and report if any of the following apply:

- **Skill gap:** "This project needs [X] and no current skill covers it. Consider creating a `[name]` skill." — e.g., a real-time WebSocket layer, a data pipeline, a mobile-native app.
- **Skill split:** "This skill's [section] is growing complex enough to be its own skill." — e.g., if BFF patterns become a recurring deep topic.
- **Skill overlap:** "This skill and `[other skill]` both cover [topic]. Consider consolidating." — e.g., deployment concerns appearing in both backend and deployment skills.

Only report when genuinely applicable. Don't force observations.

### Learnings

Corrections and refinements discovered during use. When the user overrides a recommendation or a default doesn't fit, record it here so future uses benefit.

_(Empty — will accumulate over time)_
