# Guardrails

Cross-cutting rules that apply to **every skill** in the pipeline. Each skill must follow these in addition to its own skill-specific Do/Don't table.

Read this file before producing any output. Violations of these guardrails are treated as bugs.

---

## 1. Security

### Authentication & Authorization

| Rule | Rationale |
|------|-----------|
| Never store tokens in localStorage or sessionStorage | XSS can steal them. Use httpOnly, Secure, SameSite cookies for session tokens. |
| Never implement custom crypto or password hashing | Use proven libraries (bcrypt, argon2) or delegate to an identity provider (Google, Auth0). |
| Always validate JWTs server-side — signature, expiry, issuer, audience | Client-side JWT parsing is for display only, never for access control. |
| Never trust client-side authorization checks | Always enforce permissions on the backend. Frontend hides UI elements; backend enforces access. |
| Use PKCE for all OAuth flows from public clients | Implicit flow is deprecated. Authorization Code + PKCE is the standard. |
| API keys are for machine-to-machine only | Never use API keys for end-user authentication. |
| Rotate secrets on a schedule | No credential should be permanent. Use secret managers with rotation policies. |

### Input Handling

| Rule | Rationale |
|------|-----------|
| Always use parameterized queries for SQL | String concatenation causes SQL injection. No exceptions. `pgx` uses `$1` placeholders by default — never bypass this. |
| Sanitize and escape all user input before rendering in HTML | Prevents XSS. In React, JSX escapes by default — but `dangerouslySetInnerHTML` and `href="javascript:"` bypass this. In Apps Script HTML, escape manually. |
| Validate all input at the HTTP boundary | Check types, lengths, ranges, and formats in handlers before passing to services. Reject early. |
| Set request body size limits | Prevent denial-of-service via oversized payloads. Default to a reasonable max (e.g., 1MB for JSON, larger for file uploads with explicit limits). |
| Never pass unsanitized user input to shell commands, system calls, or template engines | Command injection and template injection are as dangerous as SQL injection. |

### Data Exposure

| Rule | Rationale |
|------|-----------|
| Never log PII (names, emails, passwords, tokens, IP addresses) | Logs are often stored unencrypted and broadly accessible. Log user IDs, request IDs, and error codes instead. |
| Never return internal errors, stack traces, or database errors to clients | Expose a generic error message and error code. Log the full error server-side. |
| Never include sensitive fields in API responses unless explicitly needed | Filter out password hashes, internal IDs, auth tokens, and metadata before serialization. |
| Strip EXIF data from uploaded images | EXIF can contain GPS coordinates, device info, and timestamps. |

### Network & Transport

| Rule | Rationale |
|------|-----------|
| HTTPS everywhere — no exceptions | TLS in transit for all API calls, webhooks, and inter-service communication. |
| Set restrictive CORS defaults | Allow only specific origins, not `*`. Whitelist the frontend domain(s) explicitly. In dev, `localhost` origins are fine. |
| Use security headers | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Content-Security-Policy` on frontend responses. |
| Databases on private networks only | Never expose database ports to the internet. Use VPC, private subnets, or firewall rules. |

### Dependency Management

| Rule | Rationale |
|------|-----------|
| Vet dependencies before adding | Check maintenance status, known vulnerabilities, license compatibility. Prefer well-maintained, widely-used packages. |
| Pin dependency versions | Use exact versions in `go.mod`, `package.json`, and `requirements.txt`. Avoid `latest` or floating ranges in production. |
| Run dependency audit in CI | `go vuln`, `npm audit`, or equivalent. Fail the build on critical vulnerabilities. |
| Minimize dependencies | Every dependency is an attack surface and maintenance burden. Prefer stdlib when it covers the use case. |

---

## 2. Cost

| Rule | Rationale |
|------|-----------|
| Estimate cost before provisioning cloud resources | Use cloud pricing calculators. Present the estimated monthly cost to the user before generating Terraform. |
| Set budget alerts on every cloud project | GCP Budget Alerts or AWS Budgets. Alert at 50%, 80%, and 100% of expected spend. |
| Tag all cloud resources | Every resource must have: `project`, `environment` (dev/staging/prod), `owner`, `managed-by` (terraform/manual). Untagged resources are invisible to cost tracking. |
| Use auto-scaling with explicit maximums | Set `max_instances`, `max_nodes`, or equivalent. Auto-scaling without caps leads to surprise bills. |
| Default to the smallest viable instance size | Start with `db.t4g.micro` or `db-f1-micro`, not production-sized instances. Scale up based on actual load. |
| Use committed use discounts for stable workloads | For production resources that run 24/7, committed use (GCP) or reserved instances (AWS) save 30-60%. Flag this to the user. |
| Clean up dev/test resources | Dev databases, test buckets, and sandbox deployments should auto-expire or be cleaned up weekly. |
| Never leave idle resources running | Unused load balancers, unattached disks, stopped-but-not-terminated instances all cost money. |

---

## 3. Data Integrity

### Database

| Rule | Rationale |
|------|-----------|
| Always use database migrations — never modify schemas manually | Migrations are versioned, repeatable, and auditable. Use `golang-migrate` or plain SQL files in order. |
| Every migration must be reversible | Include both `up` and `down` steps. If a migration can't be reversed, document why and get explicit approval. |
| Test migrations against a copy of production data | A migration that works on an empty database may fail on real data (constraints, data volumes, locks). |
| Enable point-in-time recovery in production | PostgreSQL WAL archiving, Cloud SQL PITR, or RDS automated backups. Recovery should be tested periodically. |
| Use transactions for multi-step operations | If step 2 fails, step 1 should be rolled back. Don't leave the database in a partial state. |
| Never delete data in production — soft-delete first | Add a `deleted_at` timestamp. Hard delete only after a retention period or legal requirement. |
| Validate data at the database level too | Use constraints (`NOT NULL`, `CHECK`, `UNIQUE`, foreign keys) in addition to application-level validation. The database is the last line of defense. |

### Object Storage

| Rule | Rationale |
|------|-----------|
| Enable versioning on production buckets | Protects against accidental overwrite or deletion. |
| Set lifecycle policies | Auto-transition old objects to cheaper storage tiers. Auto-delete temporary uploads after a retention period. |
| Validate file types and sizes before accepting uploads | Don't trust `Content-Type` headers — inspect file magic bytes. Reject unexpected file types. |

### Backups

| Rule | Rationale |
|------|-----------|
| Automated backups for all production databases | Daily minimum. Retention period based on compliance requirements (default: 30 days). |
| Test backup restoration quarterly | A backup that can't be restored is not a backup. Schedule and document restoration tests. |
| Store backups in a different region or account | Protects against regional outages and account compromise. |

---

## 4. Process

### Evaluation

| Rule | Rationale |
|------|-----------|
| Never skip the evaluation step — even when defaults seem obvious | Every project has context that might invalidate a default. Spend 2 minutes evaluating, save hours of rework. |
| Match output depth to project complexity | A 10-person task app doesn't need a 50-page architecture doc. A warehouse system with ERP integration does. Scale the deliverables. |
| Ask about budget and cost constraints early | Architecture decisions are constrained by budget. A serverless-first approach might be cheaper than K8s for low traffic. |
| Ask about team skills and experience | Don't recommend Go to a Python-only team without flagging the learning curve. Don't recommend K8s if nobody has operated it. |
| Ask about compliance and data residency | GDPR, HIPAA, SOC2, data sovereignty requirements affect every layer — database location, logging, auth, encryption. Surface these early. |

### Pipeline Discipline

| Rule | Rationale |
|------|-----------|
| Don't skip skills in the pipeline | Architecture → Backend → Frontend → Testing → Deployment. Each skill expects artifacts from the previous one. |
| Don't produce artifacts that orphan | Every file you generate should be referenced by the README, Docker Compose, CI pipeline, or another artifact. Dead files confuse future developers. |
| Document every non-obvious decision | If you chose PostgreSQL over MongoDB, say why in the architecture doc. If you skipped a BFF layer, say why. Decisions without rationale are invisible tech debt. |
| Version the API contract | The OpenAPI spec is a living document. When it changes, bump the version. Breaking changes get a new major version (`/api/v2/`). |

### Deployment Discipline

| Rule | Rationale |
|------|-----------|
| Never deploy to production on Friday | Unless it's an emergency fix. Reduced staffing on weekends makes incident response harder. |
| Never deploy without a rollback plan | Know the previous working image tag. Know how to roll back the database migration. Test the rollback before deploying. |
| Never use `latest` tag for container images in production | Tags must be immutable and traceable — use git SHA or semantic version. `latest` is ambiguous and non-reproducible. |
| Never share credentials between environments | Dev, staging, and prod must have separate databases, separate service accounts, separate API keys. Compromise in dev shouldn't affect prod. |
| Require manual approval for production deployments | Use GitHub Environment protection rules. No auto-deploy to prod — a human reviews the plan and approves. |
| Run the production checklist before every first deployment | TLS, health checks, backups, monitoring, budget alerts, DNS. Don't go live without verifying each item. |

---

## 5. Resilience

### Error Handling

| Rule | Rationale |
|------|-----------|
| Handle errors at every layer — don't swallow them silently | Log the error, return an appropriate status code, and ensure the caller knows something went wrong. |
| Use structured error responses with consistent shape | `{"error": "message", "code": "ERROR_CODE"}` — never plain text, never inconsistent formats. |
| Set timeouts on all external calls | HTTP clients, database queries, and inter-service calls must have explicit timeouts. Default: 10s for HTTP, 30s for database. |
| Implement health check and readiness endpoints | `/health` (is the process running?) and `/ready` (are dependencies connected?). K8s, Cloud Run, and load balancers need these. |
| Graceful shutdown — drain connections before stopping | Handle SIGTERM, stop accepting new requests, finish in-flight requests, close database connections, then exit. |

### Testing Resilience

| Rule | Rationale |
|------|-----------|
| Tests must be deterministic — no flaky tests allowed | If a test fails intermittently, fix it or delete it. Flaky tests erode trust in the test suite. |
| Tests must be isolated — no shared state between test cases | Each test sets up its own data and cleans up after itself. Use subtests, transactions, or separate databases. |
| Test error paths, not just happy paths | What happens when the database is down? When the API returns 500? When the input is empty? These are the paths that break in production. |
| Set test timeouts in CI | No test should run longer than 5 minutes. A hanging test blocks the entire pipeline. |
| Contract tests are mandatory when an API spec exists | If you have an OpenAPI spec, validate the running API against it in CI. Spec drift is a silent bug. |

---

## 6. Maintenance

| Rule | Rationale |
|------|-----------|
| Every project must have a README with setup instructions | A new developer should be able to run the project locally within 15 minutes of cloning. |
| Use structured logging (`slog` in Go, structured JSON elsewhere) | Structured logs are searchable, filterable, and machine-parseable. Free-text logs are not. |
| Log with context — request ID, user ID, operation | Every log line should be traceable to a specific request and user. This makes debugging production issues possible. |
| Don't log at DEBUG level in production | Use INFO as the default. DEBUG generates noise that obscures real issues and increases storage costs. |
| Monitor what matters — latency, error rate, saturation | The four golden signals: latency, traffic, errors, saturation. Start with these before adding custom metrics. |
| Set alerts on error rate spikes, not on individual errors | Individual errors are noise. A sudden increase in error rate is a signal. Alert on the rate, investigate individual errors. |
| Keep the dependency tree shallow | Deep dependency chains are fragile and hard to update. Prefer direct dependencies over transitive ones. |
| Update dependencies monthly | Stale dependencies accumulate vulnerabilities. Schedule a monthly update, run tests, merge if green. |

---

## How Skills Reference This File

Every skill should include this line in its Rules section:

> **Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

Skill-specific Do/Don't tables remain in each skill for rules that only apply to that skill's domain (e.g., "use `pgx` not GORM" in the backend skill).
