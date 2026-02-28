# Development Standards — Code Examples and Templates

This reference contains all code examples, templates, and detailed patterns
referenced by SKILL.md. Claude Code should read the relevant section when
implementing a specific pattern.

## Table of Contents

- [Go Dependency Injection Examples](#go-dependency-injection-examples)
- [Go Error Handling Examples](#go-error-handling-examples)
- [Go Unit Test Patterns](#go-unit-test-patterns)
- [Go Error Path Testing](#go-error-path-testing)
- [Go Test Helpers and Fixtures](#go-test-helpers-and-fixtures)
- [Go Interface Compliance Tests](#go-interface-compliance-tests)
- [Frontend Testing (Next.js)](#frontend-testing-nextjs)
- [API Documentation Template](#api-documentation-template)
- [CHANGELOG Template](#changelog-template)
- [README Template](#readme-template)
- [Documentation Index Template](#documentation-index-template)
- [Structured Logging Examples](#structured-logging-examples)
- [Configuration Management Pattern](#configuration-management-pattern)
- [Graceful Shutdown Pattern](#graceful-shutdown-pattern)
- [Next.js API Client Pattern](#nextjs-api-client-pattern)
- [Go HTTP Handler Testing](#go-http-handler-testing)
- [Rust Testing](#rust-testing)
- [ADR Template](#adr-template)

---

## Go Dependency Injection Examples

```go
// ✅ CORRECT — service depends on interface
type OrderService struct {
    repo   OrderRepository  // interface
    mailer Notifier         // interface
}

// ✅ CORRECT — interface defined by the consumer, not the provider
type OrderRepository interface {
    FindByID(ctx context.Context, id string) (*Order, error)
    Save(ctx context.Context, order *Order) error
}

// ❌ WRONG — concrete dependency
type OrderService struct {
    repo *PostgresOrderRepo  // concrete type
}

// ✅ CORRECT — constructor accepts interfaces
func NewOrderService(repo OrderRepository, mailer Notifier) *OrderService {
    return &OrderService{repo: repo, mailer: mailer}
}
```

---

## Go Error Handling Examples

```go
// ✅ CORRECT — wrapped with context
if err := s.repo.Save(ctx, order); err != nil {
    return fmt.Errorf("failed to save order %s: %w", order.ID, err)
}

// ❌ WRONG — swallowed error
s.repo.Save(ctx, order)

// ❌ WRONG — no context
return err
```

---

## Go Unit Test Patterns

### Mock Implementation

```go
type mockOrderRepo struct {
    orders map[string]*Order
    err    error
}

func (m *mockOrderRepo) FindByID(ctx context.Context, id string) (*Order, error) {
    if m.err != nil {
        return nil, m.err
    }
    order, ok := m.orders[id]
    if !ok {
        return nil, ErrNotFound
    }
    return order, nil
}
```

### Table-Driven Tests

```go
func TestScoreCalculation(t *testing.T) {
    tests := []struct {
        name     string
        input    []IndexMatch
        expected int
    }{
        {"no matches returns zero", nil, 0},
        {"exact match scores high", []IndexMatch{{MatchType: "exact"}}, 80},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Score(tt.input, testIndexDefs)
            if result.Total != tt.expected {
                t.Errorf("got %d, want %d", result.Total, tt.expected)
            }
        })
    }
}
```

---

## Go Error Path Testing

```go
func TestGetOrder_RepoFailure(t *testing.T) {
    repo := &mockOrderRepo{err: fmt.Errorf("connection refused")}
    svc := NewOrderService(repo, &mockNotifier{})

    _, err := svc.GetOrder(context.Background(), "order-123")

    if err == nil {
        t.Fatal("expected error, got nil")
    }
    if !strings.Contains(err.Error(), "connection refused") {
        t.Errorf("expected wrapped error, got: %v", err)
    }
}
```

Test these error scenarios for every service:
- Repository/database failures
- External service timeouts (use `context.WithTimeout` in tests)
- Invalid input (nil, empty strings, malformed data)
- Not-found cases (distinct from errors)
- Partial failures (one dependency fails, others succeed)

---

## Go Test Helpers and Fixtures

```go
// internal/testutil/fixtures.go
package testutil

// NewTestPerson creates a Person with sensible defaults for testing.
func NewTestPerson(overrides ...func(*model.Person)) *model.Person {
    p := &model.Person{
        PersonToken: "test-token-" + uuid.NewString()[:8],
        Status:      "active",
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    for _, fn := range overrides {
        fn(p)
    }
    return p
}
```

---

## Go Interface Compliance Tests

```go
// Compile-time verification that PostgresRepo implements Repository
var _ discovery.Repository = (*PostgresRepo)(nil)
```

---

## Frontend Testing (Next.js)

Use: **Vitest** (unit), **React Testing Library** (components), **MSW** (API mocking).

```typescript
// src/lib/__tests__/api-client.test.ts
import { ApiClient, ApiError } from '../api-client';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer();
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('lookup returns candidates on success', async () => {
  server.use(
    http.post('/api/v1/lookup', () => {
      return HttpResponse.json({ candidates: [{ person_token: 'abc' }] });
    })
  );
  const client = new ApiClient('');
  const result = await client.lookup({ last_name: 'Hansen' });
  expect(result.candidates).toHaveLength(1);
});

test('lookup throws ApiError on 400', async () => {
  server.use(
    http.post('/api/v1/lookup', () => {
      return new HttpResponse('invalid request', { status: 400 });
    })
  );
  const client = new ApiClient('');
  await expect(client.lookup({})).rejects.toThrow(ApiError);
});
```

---

## API Documentation Template

Document every REST endpoint in developer-guide.md:

```markdown
### POST /api/v1/lookup

Resolve a patient identity from demographic fields.

**Request:**
\`\`\`json
{
  "first_name": "Bob",
  "last_name": "Hansen",
  "dob": "1965-03-15",
  "requestor_system": "memorial_ehr",
  "requestor_user": "dr.smith@memorial.org",
  "purpose": "clinical"
}
\`\`\`

**Response (200):**
\`\`\`json
{
  "candidates": [
    {
      "person_token": "550e8400-...",
      "confidence_score": 87,
      "personas": [...]
    }
  ]
}
\`\`\`

**Errors:**
- 400: Invalid request (missing required fields)
- 401: Authentication required
- 500: Internal server error
```

---

## CHANGELOG Template

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [0.0.3] - 2026-02-28
### Added
- Bloom filter comparison via Vault /compare endpoint

### Changed
- DisplayLabel replaced with DataCategory enum

### Fixed
- Phonetic HMAC not matching when DOB format varies
```

Update with every version bump. Categories: Added, Changed, Deprecated, Removed, Fixed, Security.

---

## README Template

```markdown
# Project Name

One-sentence description of what this project does.

## Quick Start

Link to docs/quickstart.md or inline 3-5 steps.

## Architecture

One paragraph + diagram (ASCII or Mermaid).

## Development

\`\`\`bash
# Prerequisites
# Build
# Test
# Run
\`\`\`

## Documentation

Link to docs/index.md.

## Version

Current version: see VERSION file. Uses semantic versioning.
```

---

## Documentation Index Template

```markdown
# Project Name — Documentation Index

## Guides
- [Quick Start](quickstart.md) — Get running in under 10 minutes
- [Developer Guide](developer-guide.md) — Architecture, patterns, API reference
- [Admin Guide](admin-guide.md) — Deployment, configuration, operations

## Architecture Decisions
- [ADR-001: Database Choice](decisions/001-database-choice.md)

## API Reference
- [REST API](developer-guide.md#rest-api)
```

---

## Structured Logging Examples

```go
// ✅ CORRECT — structured, contextual
slog.Info("lookup completed",
    "person_token", candidate.PersonToken,
    "confidence", score.Total,
    "match_count", len(candidates),
    "duration_ms", elapsed.Milliseconds(),
)

// ❌ WRONG — unstructured
log.Printf("Found %d matches for %s", len(candidates), token)

// Logger injection via constructor
type Service struct {
    repo   Repository
    logger *slog.Logger
}

func NewService(repo Repository, logger *slog.Logger) *Service {
    return &Service{
        repo:   repo,
        logger: logger.With("component", "discovery"),
    }
}
```

---

## Configuration Management Pattern

```go
// internal/config/config.go
type Config struct {
    Port         int           `env:"PORT" default:"8080"`
    Environment  string        `env:"ENVIRONMENT" default:"development"`
    DBMaxConns   int           `env:"DB_MAX_CONNS" default:"25"`
    DBTimeout    time.Duration `env:"DB_TIMEOUT" default:"5s"`
    VaultAddr    string        `env:"VAULT_ADDR" required:"true"`
    VaultMount   string        `env:"VAULT_MOUNT" default:"personalink"`
}
```

Non-secret config: environment variables → typed struct. Secrets: always from Vault, never env vars.

---

## Graceful Shutdown Pattern

```go
func main() {
    // ... setup ...

    srv := &http.Server{Addr: fmt.Sprintf(":%d", cfg.Port), Handler: router}

    go func() {
        slog.Info("server starting", "port", cfg.Port)
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            slog.Error("server failed", "error", err)
            os.Exit(1)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    slog.Info("shutting down gracefully")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        slog.Error("forced shutdown", "error", err)
    }
    slog.Info("server stopped")
}
```

---

## Next.js API Client Pattern

```typescript
// src/lib/api-client.ts
export class ApiClient {
  private baseUrl: string;

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async lookup(req: LookupRequest): Promise<LookupResponse> {
    const res = await fetch(`${this.baseUrl}/api/v1/lookup`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(req),
    });
    if (!res.ok) throw new ApiError(res.status, await res.text());
    return res.json();
  }
}
```

---

## Go HTTP Handler Testing

Test every HTTP handler with `httptest`. Cover: correct status codes, response body structure, error responses, and auth/validation middleware.

```go
func TestLookupHandler_Success(t *testing.T) {
    // Arrange: mock service returns a known result
    mockSvc := &mockDiscoveryService{
        result: &discovery.LookupResponse{
            Candidates: []model.MatchCandidate{
                {PersonToken: "abc-123", ConfidenceScore: 87},
            },
        },
    }
    handler := NewLookupHandler(mockSvc, slog.Default())

    reqBody := `{"last_name":"Hansen","dob":"1965-03-15","requestor_system":"test","requestor_user":"test","purpose":"clinical"}`
    req := httptest.NewRequest(http.MethodPost, "/api/v1/lookup", strings.NewReader(reqBody))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    // Act
    handler.ServeHTTP(rec, req)

    // Assert
    if rec.Code != http.StatusOK {
        t.Fatalf("expected 200, got %d: %s", rec.Code, rec.Body.String())
    }
    var resp LookupResponse
    if err := json.NewDecoder(rec.Body).Decode(&resp); err != nil {
        t.Fatalf("failed to decode response: %v", err)
    }
    if len(resp.Candidates) != 1 {
        t.Errorf("expected 1 candidate, got %d", len(resp.Candidates))
    }
}

func TestLookupHandler_InvalidJSON_Returns400(t *testing.T) {
    handler := NewLookupHandler(&mockDiscoveryService{}, slog.Default())

    req := httptest.NewRequest(http.MethodPost, "/api/v1/lookup", strings.NewReader("{invalid"))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    handler.ServeHTTP(rec, req)

    if rec.Code != http.StatusBadRequest {
        t.Fatalf("expected 400, got %d", rec.Code)
    }
}

func TestLookupHandler_ServiceError_Returns500(t *testing.T) {
    mockSvc := &mockDiscoveryService{err: fmt.Errorf("vault unavailable")}
    handler := NewLookupHandler(mockSvc, slog.Default())

    reqBody := `{"last_name":"Hansen","requestor_system":"test","requestor_user":"test","purpose":"clinical"}`
    req := httptest.NewRequest(http.MethodPost, "/api/v1/lookup", strings.NewReader(reqBody))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    handler.ServeHTTP(rec, req)

    if rec.Code != http.StatusInternalServerError {
        t.Fatalf("expected 500, got %d", rec.Code)
    }
}
```

---

## Rust Testing

### Unit Tests (same file)

```rust
pub fn dice_coefficient(a: &[u8], b: &[u8]) -> f64 {
    if a.len() != b.len() { return 0.0; }
    let intersection: u32 = a.iter().zip(b).map(|(x, y)| (x & y).count_ones()).sum();
    let count_a: u32 = a.iter().map(|x| x.count_ones()).sum();
    let count_b: u32 = b.iter().map(|x| x.count_ones()).sum();
    let total = count_a + count_b;
    if total == 0 { return 0.0; }
    2.0 * intersection as f64 / total as f64
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_dice_coefficient_identical_filters_returns_one() {
        let filter = vec![0xFF; 32];
        assert!((dice_coefficient(&filter, &filter) - 1.0).abs() < f64::EPSILON);
    }

    #[test]
    fn test_dice_coefficient_disjoint_filters_returns_zero() {
        let a = vec![0xF0; 32];
        let b = vec![0x0F; 32];
        assert!((dice_coefficient(&a, &b)).abs() < f64::EPSILON);
    }

    #[test]
    fn test_dice_coefficient_empty_filters_returns_zero() {
        let filter = vec![0x00; 32];
        assert!((dice_coefficient(&filter, &filter)).abs() < f64::EPSILON);
    }

    #[test]
    fn test_dice_coefficient_mismatched_length_returns_zero() {
        let a = vec![0xFF; 32];
        let b = vec![0xFF; 16];
        assert!((dice_coefficient(&a, &b)).abs() < f64::EPSILON);
    }
}
```

### Async Integration Tests (tests/ directory)

```rust
// tests/integration_test.rs
use tokio;

#[tokio::test]
async fn test_encode_roundtrip_produces_consistent_hashes() {
    let client = TestVaultClient::new().await;
    let resp1 = client.encode("Bob", "Hansen", "1965-03-15").await.unwrap();
    let resp2 = client.encode("Bob", "Hansen", "1965-03-15").await.unwrap();
    assert_eq!(resp1.exact_hmac, resp2.exact_hmac);
}

#[tokio::test]
async fn test_encode_timeout_returns_error() {
    let client = TestVaultClient::with_timeout(Duration::from_millis(1)).await;
    let result = client.encode("Bob", "Hansen", "1965-03-15").await;
    assert!(result.is_err());
}
```

### Trait Mocking with mockall

```rust
use mockall::automock;

#[automock]
pub trait Repository: Send + Sync {
    async fn find_by_exact_hmac(&self, hmac: &[u8]) -> Result<Vec<IndexInstance>, Error>;
}

#[tokio::test]
async fn test_lookup_repo_failure_propagates_error() {
    let mut mock_repo = MockRepository::new();
    mock_repo
        .expect_find_by_exact_hmac()
        .returning(|_| Err(Error::ConnectionRefused));

    let svc = DiscoveryService::new(mock_repo);
    let result = svc.lookup(test_request()).await;
    assert!(matches!(result, Err(Error::ConnectionRefused)));
}
```

---

## ADR Template

Store in `docs/decisions/NNN-title.md`. Use sequential numbering.

```markdown
# ADR-NNN: Title of Decision

## Status

Accepted | Proposed | Deprecated | Superseded by ADR-XXX

## Date

YYYY-MM-DD

## Context

What is the technical or business problem? What constraints exist?
What options were considered?

## Decision

What was decided and why. Be specific about the choice made.

## Alternatives Considered

### Option A: [Name]
- Pros: ...
- Cons: ...

### Option B: [Name]
- Pros: ...
- Cons: ...

## Consequences

### Positive
- What becomes easier or better?

### Negative
- What becomes harder? What trade-offs were accepted?

### Risks
- What could go wrong? What mitigations exist?
```

Example: `docs/decisions/001-vault-custom-plugin.md`

```markdown
# ADR-001: Custom Vault Plugin vs Transit Engine

## Status
Accepted

## Date
2026-02-28

## Context
PersonaLink requires HMAC-SHA256 hashing of demographic data. Vault's
transit engine supports HMAC, but using it requires the Go application to
normalize and phonetically encode demographics before sending to Vault,
exposing intermediate fragments in application memory and on the network.

## Decision
Build a custom Vault secrets engine plugin that accepts raw demographics
and returns only opaque hashes. The entire pipeline (normalize → phonetic
encode → n-gram → HMAC → Bloom filter) runs inside Vault's process boundary.

## Alternatives Considered

### Transit Engine
- Pros: No custom plugin code, faster to implement
- Cons: Application sees normalized text and phonetic codes, weaker security claim

### Application-Side HMAC with Vault-Managed Keys
- Pros: Simpler than custom plugin
- Cons: Key must be exported to application, defeats purpose of Vault

## Consequences
### Positive
- HMAC key never leaves Vault
- Strongest possible security and patent claim
### Negative
- 2-3 days additional development for custom plugin
- Plugin must be maintained alongside Vault upgrades
```
