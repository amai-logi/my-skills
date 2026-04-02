---
name: testing
description: "Test applications across all runtimes and deployment targets. Covers Go backend tests (unit, integration, API contract), React frontend tests (component, E2E), Apps Script validation, and test environment setup for self-hosted and cloud deployments. Use this skill when the user asks to add tests, set up testing, validate an API contract, or create a CI test pipeline."
---

# Testing Skill

Test applications at the right level for the right runtime. One principle: **test behavior, not implementation.** Tests should validate what the system does, not how it does it internally.

## Step 0: Understand What We're Testing

Before writing tests, identify:

1. **What runtime?** Go backend, React frontend, Apps Script, or full stack?
2. **What deployment target?** Local, self-hosted (Docker/K8s), or cloud-managed?
3. **Is there an OpenAPI spec?** If the `software-architecture` skill produced one, use it as the contract to test against.
4. **What's the risk profile?** What breaks if this code is wrong? Auth, data integrity, and billing logic need more coverage than a settings page.

## Step 1: Choose Test Levels

Not every project needs every level. Evaluate what's worth testing:

| Level | What it validates | Cost | When to use |
|-------|------------------|------|-------------|
| **Unit** | Single function/method in isolation | Low | Business logic, data transformations, validation rules |
| **Integration** | Component + real dependency (database, external API) | Medium | Data access layer, auth middleware, API handlers with real DB |
| **API contract** | API matches its OpenAPI/GraphQL spec | Medium | Any project with an API spec — catches drift between spec and implementation |
| **Component (frontend)** | UI component renders and behaves correctly | Medium | Interactive components, forms, conditional rendering |
| **E2E** | Full user workflow through the real system | High | Critical paths only — login, core business flow, payment |

**Decision rule:** Start with integration tests for the backend and contract tests against the spec. Add unit tests for complex business logic. Add E2E only for critical paths. Don't test getters, setters, or simple CRUD wrappers.

---

## Go Backend Tests

### Unit Tests

Use Go's `testing` stdlib. Table-driven tests are the standard pattern:

```go
func TestCalculateDiscount(t *testing.T) {
    tests := []struct {
        name     string
        price    float64
        quantity int
        want     float64
    }{
        {"no discount under 10", 100, 5, 500},
        {"10% discount at 10+", 100, 10, 900},
        {"20% discount at 50+", 100, 50, 4000},
        {"zero quantity", 100, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := CalculateDiscount(tt.price, tt.quantity)
            if got != tt.want {
                t.Errorf("CalculateDiscount(%v, %v) = %v, want %v",
                    tt.price, tt.quantity, got, tt.want)
            }
        })
    }
}
```

**When to unit test:**
- Pure functions with logic (calculations, transformations, validation)
- Error handling paths
- Edge cases (zero values, empty strings, boundary conditions)

**When NOT to unit test:**
- Simple struct constructors
- Direct database calls (use integration tests instead)
- Code that just delegates to another function

### Integration Tests

Use `testcontainers-go` to spin up real dependencies. No mocks for databases.

```go
import (
    "context"
    "testing"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestTaskStore(t *testing.T) {
    ctx := context.Background()

    // Start real PostgreSQL container
    pg, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        postgres.BasicWaitStrategies(),
    )
    if err != nil {
        t.Fatal(err)
    }
    defer pg.Terminate(ctx)

    connStr, _ := pg.ConnectionString(ctx, "sslmode=disable")
    store, _ := NewPostgresStore(connStr)

    // Run migrations
    store.Migrate(ctx)

    // Test actual database operations
    t.Run("create and retrieve task", func(t *testing.T) {
        task := &Task{Title: "Test task", Status: "pending"}
        created, err := store.CreateTask(ctx, task)
        if err != nil {
            t.Fatal(err)
        }

        got, err := store.GetTask(ctx, created.ID)
        if err != nil {
            t.Fatal(err)
        }
        if got.Title != "Test task" {
            t.Errorf("got title %q, want %q", got.Title, "Test task")
        }
    })

    t.Run("list tasks filters by status", func(t *testing.T) {
        tasks, err := store.ListTasks(ctx, "pending")
        if err != nil {
            t.Fatal(err)
        }
        for _, task := range tasks {
            if task.Status != "pending" {
                t.Errorf("got status %q, want pending", task.Status)
            }
        }
    })
}
```

**When to integration test:**
- All database operations (CRUD, queries, migrations)
- Auth middleware with a real JWKS endpoint
- API handlers end-to-end (HTTP request → handler → DB → response)
- External API integrations (use testcontainers or a local mock server)

### API Handler Tests

Test handlers as HTTP endpoints using `httptest`:

```go
func TestCreateTaskHandler(t *testing.T) {
    // Setup: real DB via testcontainers (or test helper)
    store := setupTestStore(t)
    router := api.NewRouter(store, testConfig())

    body := `{"title": "New task", "status": "pending"}`
    req := httptest.NewRequest("POST", "/api/v1/tasks", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+testJWT(t))

    rec := httptest.NewRecorder()
    router.ServeHTTP(rec, req)

    if rec.Code != http.StatusCreated {
        t.Errorf("got status %d, want %d", rec.Code, http.StatusCreated)
    }

    var resp Task
    json.NewDecoder(rec.Body).Decode(&resp)
    if resp.Title != "New task" {
        t.Errorf("got title %q, want %q", resp.Title, "New task")
    }
}
```

### API Contract Tests

Validate that the running API matches its OpenAPI spec. Use `schemathesis` or a custom validator:

```bash
# Install schemathesis
uv tool install schemathesis

# Run against the OpenAPI spec and live server
st run openapi.yaml --base-url http://localhost:8080 --checks all
```

Or in Go, validate response schemas against the spec:

```go
import "github.com/getkin/kin-openapi/openapi3"

func TestAPIMatchesSpec(t *testing.T) {
    // Load OpenAPI spec
    loader := openapi3.NewLoader()
    doc, err := loader.LoadFromFile("openapi.yaml")
    if err != nil {
        t.Fatal(err)
    }

    // For each path in the spec, hit the endpoint and validate
    // the response matches the declared schema
    for path, pathItem := range doc.Paths.Map() {
        if pathItem.Get != nil {
            t.Run("GET "+path, func(t *testing.T) {
                // Make request, validate response against spec
                validateEndpoint(t, doc, "GET", path)
            })
        }
    }
}
```

### Test Helpers

Create a `testutil` package for shared test setup:

```go
// internal/testutil/testutil.go
package testutil

func SetupTestDB(t *testing.T) *pgxpool.Pool {
    t.Helper()
    // Spin up testcontainer, run migrations, return pool
    // t.Cleanup handles teardown
}

func TestJWT(t *testing.T, userID string, roles ...string) string {
    t.Helper()
    // Generate a valid JWT signed with a test key
}

func TestConfig() *config.Config {
    // Return config suitable for tests
}
```

---

## React Frontend Tests

### Component Tests

Use Vitest + React Testing Library:

```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { describe, it, expect } from 'vitest'
import { TaskForm } from './TaskForm'

describe('TaskForm', () => {
  it('submits with valid data', async () => {
    const onSubmit = vi.fn()
    render(<TaskForm onSubmit={onSubmit} />)

    await fireEvent.change(screen.getByLabelText('Title'), {
      target: { value: 'New task' },
    })
    await fireEvent.click(screen.getByRole('button', { name: /create/i }))

    expect(onSubmit).toHaveBeenCalledWith(
      expect.objectContaining({ title: 'New task' })
    )
  })

  it('shows validation error for empty title', async () => {
    render(<TaskForm onSubmit={vi.fn()} />)

    await fireEvent.click(screen.getByRole('button', { name: /create/i }))

    expect(screen.getByText(/title is required/i)).toBeInTheDocument()
  })
})
```

**When to component test:**
- Forms with validation
- Conditional rendering (auth states, loading, errors)
- Interactive components (modals, dropdowns, filters)

**When NOT to component test:**
- Static layout components
- Components that just pass props through
- Styling

### E2E Tests

Use Playwright for critical user flows only:

```typescript
import { test, expect } from '@playwright/test'

test('user can create and complete a task', async ({ page }) => {
  await page.goto('/login')
  // Auth handled by test setup (mock OAuth or test credentials)

  await page.click('[data-testid="new-task"]')
  await page.fill('[name="title"]', 'E2E test task')
  await page.click('button:has-text("Create")')

  await expect(page.locator('.task-list')).toContainText('E2E test task')

  await page.click('.task-list >> text=E2E test task')
  await page.click('button:has-text("Complete")')

  await expect(page.locator('.badge-approved')).toBeVisible()
})
```

**Limit E2E to critical paths:**
- Authentication flow
- The primary business workflow (create → view → update → delete)
- Payment or other high-risk operations

---

## Apps Script Testing

Apps Script has no built-in test framework. Testing strategy:

### 1. Extract Logic into Testable Functions

Separate business logic from Apps Script APIs:

```javascript
// TESTABLE — pure logic, no Google API calls
function calculateScore(responses) {
  return responses.reduce((sum, r) => sum + r.score, 0) / responses.length;
}

// NOT TESTABLE — depends on SpreadsheetApp
function processSheet() {
  const data = SpreadsheetApp.getActiveSheet().getDataRange().getValues();
  const score = calculateScore(data);
  // ...
}
```

### 2. Manual Test Checklist

For each Apps Script project, produce a test checklist:

```markdown
## Test Checklist — [Project Name]

### First Run
- [ ] Script installs via Extensions > Apps Script
- [ ] OAuth consent screen appears and completes
- [ ] Custom menu appears after page reload

### Core Functions
- [ ] [Function A] produces expected output
- [ ] [Function B] handles empty data gracefully
- [ ] [Function C] sends email correctly

### Edge Cases
- [ ] Empty spreadsheet — no errors
- [ ] Missing columns — clear error message
- [ ] Duplicate data — handled correctly

### Triggers
- [ ] onEdit fires on cell change
- [ ] Time-driven trigger runs on schedule
```

### 3. Logger-Based Validation

Use `Logger.log()` and the Apps Script execution log for debugging:

```javascript
function testCalculateScore() {
  const testData = [
    { score: 80 }, { score: 90 }, { score: 70 }
  ];
  const result = calculateScore(testData);
  Logger.log('Expected: 80, Got: ' + result);
  if (result !== 80) {
    throw new Error('calculateScore failed: expected 80, got ' + result);
  }
  Logger.log('PASS');
}
```

---

## Test Environment Setup

The same tests run everywhere — only the infrastructure setup changes.

### Local Development

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"

  seaweedfs:
    image: chrislusf/seaweedfs
    command: "server -s3"
    ports:
      - "8333:8333"
```

Or use testcontainers (preferred for Go — containers are managed per test).

### Self-Hosted CI (GitHub Actions / GitLab CI)

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./... -v -count=1

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test
      - run: npx playwright install --with-deps
      - run: npx playwright test

  contract:
    runs-on: ubuntu-latest
    needs: [backend]
    steps:
      - uses: actions/checkout@v4
      - run: uv tool install schemathesis
      - run: docker compose up -d
      - run: st run openapi.yaml --base-url http://localhost:8080 --checks all
      - run: docker compose down
```

### Cloud CI

When testing against cloud-managed services, additional setup is needed:

| Concern | What changes |
|---------|-------------|
| **Database** | Use testcontainers locally. In CI, either testcontainers or a dedicated test instance of Cloud SQL/RDS. Never test against production. |
| **Auth** | Create a test service account or test user with limited permissions. Store credentials in CI secrets. |
| **Object storage** | Use local SeaweedFS container. In CI against cloud, use a dedicated test bucket with auto-cleanup. |
| **Secrets** | CI needs access to Secret Manager. Use a separate test project/account with minimal permissions. |
| **Staging environment** | For E2E tests against cloud, deploy to a staging environment first. Run E2E there, tear down after. |

**Cost control:** Cloud test resources should auto-clean. Use ephemeral databases, short-lived test buckets, and tear down staging after E2E passes. Set budget alerts on test projects.

---

## Test Organization

### Go Backend

```
internal/
  api/
    handlers/
      tasks.go
      tasks_test.go        # Handler tests (httptest + real DB)
  service/
    task_service.go
    task_service_test.go   # Unit tests for business logic
  store/
    postgres.go
    postgres_test.go       # Integration tests (testcontainers)
  testutil/
    testutil.go            # Shared test helpers
```

### React Frontend

```
src/
  components/
    TaskForm.tsx
    TaskForm.test.tsx      # Component tests
  pages/
    Dashboard.tsx
    Dashboard.test.tsx     # Page-level component tests
tests/
  e2e/
    task-flow.spec.ts      # Playwright E2E tests
```

---

## Rules

**Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

| Do | Don't |
|----|-------|
| Test behavior (inputs → outputs) | Test implementation details |
| Use real databases in integration tests | Mock the database |
| Test against the OpenAPI spec | Assume spec and code stay in sync |
| Keep E2E tests to critical paths only | E2E test every feature |
| Use table-driven tests in Go | Write one test function per case |
| Co-locate test files with source | Put all tests in a separate `tests/` directory (Go) |
| Extract testable logic from Apps Script | Try to unit test Google API calls |
| Auto-clean cloud test resources | Leave test databases/buckets running |
| Run contract tests in CI | Only run unit tests in CI |
| Use testcontainers over docker-compose for Go | Manage test containers manually |

## Handoff

### Input
- Go project from `backend` skill (handlers, services, store, OpenAPI spec)
- React/HTML project from `frontend` skill (components, pages)
- Apps Script project from `google-apps-script` skill (functions to validate)

### Output
- Test files (Go `_test.go`, Vitest `.test.tsx`, Playwright `.spec.ts`)
- Test helpers and utilities
- CI workflow (`.github/workflows/test.yml`)
- Apps Script test checklist

### Previous skills
- `backend` — Go service to test
- `frontend` — UI components to test
- `google-apps-script` — Apps Script to create test checklist for

### Next skill

| Next | What it receives |
|------|-----------------|
| `deployment` | Tested application with CI pipeline. Deployment skill adds deploy steps to the GitHub Actions workflow. |

## Continuous Improvement

### Skill Health Check

At the end of every use, evaluate and report if any of the following apply:

- **Skill gap:** "This project needs [X] and no current skill covers it. Consider creating a `[name]` skill."
- **Skill split:** "This skill's [section] is growing complex enough to be its own skill." — e.g., if E2E testing patterns become extensive enough to warrant a dedicated skill.
- **Skill overlap:** "This skill and `[other skill]` both cover [topic]. Consider consolidating."

Only report when genuinely applicable. Don't force observations.

### Learnings

Corrections and refinements discovered during use. When the user overrides a recommendation or a default doesn't fit, record it here so future uses benefit.

_(Empty — will accumulate over time)_
