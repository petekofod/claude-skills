---
name: dev-standards
description: "Development standards and conventions for all Go, Rust, and Next.js projects. ALWAYS use this skill when writing code for all projects. Triggers on: any Go code generation, any Rust code generation, any Next.js frontend work, any backend API development, any project scaffolding, any test creation, any documentation generation, versioning decisions, or when Claude Code is asked to build, modify, or extend a codebase. This skill defines mandatory patterns for dependency injection, testing, documentation, versioning, and project structure. Use it even for small changes — consistency matters."
---

# Development Standards

## Who This Is For

Pete is a system architect building prototypes that will be handed to seasoned developers for production implementation. Code must be clean enough to hand off without a walkthrough. Every design decision must be documented so the receiving developer understands not just *what* but *why*.

## Language Defaults

| Context | Language | Framework |
|---------|----------|-----------|
| Backend services, APIs, plugins | **Go** | Standard library + minimal dependencies |
| Performance-critical or blockchain/crypto | **Rust** | Tokio for async, Serde for serialization |
| Frontend / UI | **Next.js** (TypeScript) | App Router, Tailwind CSS |
| API communication | HTTP | REST preferred, GraphQL when appropriate |

If Pete doesn't specify a language, default to Go for backend and Next.js for frontend. Ask before using anything else.

---

## Mandatory Code Patterns

### 1. Dependency Injection via Interfaces

Every service, repository, and external dependency MUST be accessed through an interface. No concrete types in function signatures except at the composition root (main.go or wire setup).

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
```

**Why:** Testability (mock any dependency), swappability (Postgres today, CockroachDB tomorrow), and clear contracts for the receiving developer.

### 2. Constructor Functions Return Interfaces

```go
// ✅ CORRECT — returns interface type
func NewOrderService(repo OrderRepository, mailer Notifier) *OrderService {
    return &OrderService{repo: repo, mailer: mailer}
}
```

### 3. Context Propagation

Every function that performs I/O (database, network, Vault, etc.) MUST accept `context.Context` as its first parameter. No exceptions.

```go
// ✅ CORRECT
func (s *OrderService) GetOrder(ctx context.Context, id string) (*Order, error)

// ❌ WRONG — no context
func (s *OrderService) GetOrder(id string) (*Order, error)
```

### 4. Error Handling

Wrap errors with context using `fmt.Errorf("...: %w", err)`. Never discard errors silently. Never use `panic` for recoverable errors.

```go
// ✅ CORRECT
if err := s.repo.Save(ctx, order); err != nil {
    return fmt.Errorf("failed to save order %s: %w", order.ID, err)
}

// ❌ WRONG — swallowed error
s.repo.Save(ctx, order)

// ❌ WRONG — no context
return err
```

---

## Testing Requirements

### Unit Tests

Every package MUST have unit tests. Tests use interfaces and mocks — never real databases or external services.

```go
// Mock implementation for testing
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

**Test file naming:** `*_test.go` in the same package.

**Table-driven tests** are preferred for multiple cases:

```go
func TestScoreCalculation(t *testing.T) {
    tests := []struct {
        name     string
        input    []IndexMatch
        expected int
    }{
        {"no matches returns zero", nil, 0},
        {"exact match scores high", []IndexMatch{{MatchType: "exact"}}, 80},
        // ... more cases
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

### Integration Tests

Integration tests use build tags to separate from unit tests:

```go
//go:build integration
// +build integration

package integration
```

Run with: `go test -tags=integration ./test/integration/...`

Integration tests may use real databases (via Docker/testcontainers) and real Vault dev servers. They MUST clean up after themselves.

### Test Coverage Target

Aim for 80%+ coverage on business logic packages. Don't chase 100% — focus on the paths that matter.

---

## Documentation Requirements

### Inline Documentation

Every exported function, type, and constant MUST have a doc comment. Comments should explain *why*, not just *what*.

```go
// ✅ CORRECT — explains the design decision
// Score computes the confidence score for a match candidate.
// The scoring formula uses three components (index weight, match quality,
// corroboration) to produce a 1-100 score. Components are independently
// capped to prevent any single factor from dominating.
func Score(matches []IndexMatch, defs map[int]IndexDefinition) ScoreBreakdown {

// ❌ WRONG — restates the function signature
// Score scores the matches.
func Score(matches []IndexMatch, defs map[int]IndexDefinition) ScoreBreakdown {
```

**Design decision comments:** When a non-obvious choice was made, document it with a `DESIGN DECISION:` prefix:

```go
// DESIGN DECISION: We use Dice coefficient over Jaccard because it gives
// higher scores for partial matches, which aligns better with the identity
// matching use case where a 6-of-8 bigram overlap should score strongly.
```

### Project Documentation Structure

Every project MUST have the following docs in a `docs/` directory, written in Markdown:

```
docs/
├── index.md                    # Table of contents linking all docs
├── developer-guide.md          # Architecture, setup, code patterns, API reference
├── quickstart.md               # Get running in <10 minutes
├── admin-guide.md              # Deployment, configuration, operations (when needed)
└── decisions/                  # Architecture Decision Records (optional)
    ├── 001-database-choice.md
    └── 002-vault-plugin-design.md
```

#### index.md (Required)

Acts as both human-readable table of contents and machine-parseable index:

```markdown
# Project Name — Documentation Index

## Guides
- [Quick Start](quickstart.md) — Get running in under 10 minutes
- [Developer Guide](developer-guide.md) — Architecture, patterns, and API reference
- [Admin Guide](admin-guide.md) — Deployment, configuration, and operations

## Architecture Decisions
- [ADR-001: Database Choice](decisions/001-database-choice.md)

## API Reference
- [REST API](developer-guide.md#rest-api)
```

#### quickstart.md (Required)

Prerequisites, install steps, first run, verify it works. A developer should go from zero to running in under 10 minutes. No explanations of architecture — just the steps.

#### developer-guide.md (Required)

Architecture overview, project structure, code patterns, how to add features, API reference, testing guide. This is the document the receiving developer reads first.

#### admin-guide.md (When Needed)

Create this when the project has operational concerns: Vault configuration, database migrations, backup procedures, monitoring, scaling. Not every prototype needs this — ask Pete.

### Markdown Formatting for Export

Docs should render cleanly as PDF (via pandoc or similar) and be parseable by LLMs:

- Use ATX headers (`#`, `##`, `###`) consistently
- Use fenced code blocks with language identifiers
- Use tables for structured data
- Keep lines under 100 characters for readability
- Use relative links between docs (not absolute paths)
- Include a YAML front matter block for PDF metadata:

```markdown
---
title: "PersonaLink Developer Guide"
version: "0.1.0"
date: 2026-02-28
---
```

---

## Versioning

All projects use **Semantic Versioning** starting at `v0.0.1`.

### Rules

| Version Component | When to Bump | Permission Required |
|-------------------|--------------|---------------------|
| Patch (0.0.x) | Bug fixes, documentation, minor improvements | **No — bump automatically** |
| Minor (0.x.0) | New features, non-breaking API changes | **Ask Pete first** |
| Major (x.0.0) | Breaking API changes, major architecture shifts | **Ask Pete first** |

### Version Location

- `go.mod` or `Cargo.toml` — the source of truth
- `VERSION` file in project root (plain text, just the version string)
- Tag in git: `git tag v0.0.1`

When making changes, check the current version and bump the patch automatically. If the change warrants a minor or major bump, ask before doing it.

---

## Project Structure Conventions

### Go Projects

```
project-name/
├── cmd/
│   ├── server/main.go          # API server entry point
│   └── cli/main.go             # CLI tool entry point (if needed)
├── internal/                   # Private packages (not importable by other projects)
│   ├── model/                  # Domain types
│   ├── service/                # Business logic (depends on interfaces)
│   ├── repository/             # Data access (implements interfaces)
│   └── {feature}/              # Feature-specific packages
├── pkg/                        # Public packages (importable by other projects)
│   └── api/                    # HTTP handlers, middleware, request/response types
├── db/
│   ├── schema.sql              # Database schema
│   └── migrations/             # Versioned migrations
├── docs/                       # Documentation (see above)
├── test/
│   └── integration/            # Integration tests (build-tagged)
├── go.mod
├── go.sum
├── VERSION
├── Dockerfile
└── README.md
```

### Next.js Frontend

```
project-name-ui/
├── src/
│   ├── app/                    # Next.js App Router pages
│   ├── components/             # Reusable React components
│   ├── lib/                    # Utility functions, API client
│   ├── hooks/                  # Custom React hooks
│   └── types/                  # TypeScript type definitions
├── public/                     # Static assets
├── docs/                       # Documentation
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── VERSION
└── README.md
```

### API Communication

Frontend communicates with backend via HTTP. Use a typed API client:

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

## Checklist Before Handing Off Code

Before presenting code to Pete, verify:

- [ ] All exported functions have doc comments explaining *why*
- [ ] Design decisions are documented with `DESIGN DECISION:` prefix
- [ ] All dependencies are injected via interfaces
- [ ] All I/O functions accept `context.Context`
- [ ] Errors are wrapped with context (`fmt.Errorf("...: %w", err)`)
- [ ] Unit tests exist for business logic
- [ ] Integration test stubs exist (even if not fully implemented)
- [ ] `docs/index.md` and `docs/quickstart.md` exist
- [ ] `VERSION` file exists with current version
- [ ] `README.md` explains what the project is and how to run it