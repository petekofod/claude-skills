---
name: dev-standards
description: "Development standards and conventions for all Go, Rust, and Next.js projects. ALWAYS use this skill when writing code for all projects. Triggers on: any Go code generation, any Rust code generation, any Next.js frontend work, any backend API development, any project scaffolding, any test creation, any documentation generation, versioning decisions, or when Claude Code is asked to build, modify, or extend a codebase. This skill defines mandatory patterns for dependency injection, testing, documentation, versioning, and project structure. Use it even for small changes — consistency matters."
---

# Development Standards

Read `references/examples.md` in this skill directory for all code examples and templates.

## Who This Is For

Pete is a system architect building prototypes that will be handed to seasoned developers for production implementation. Code must be clean enough to hand off without a walkthrough. Every design decision must be documented so the receiving developer understands not just *what* but *why*.

## Language Defaults

| Context | Language | Framework |
|---------|----------|-----------|
| Backend services, APIs, plugins | **Go** | Standard library + minimal deps |
| Performance-critical / blockchain | **Rust** | Tokio, Serde |
| Frontend / UI | **Next.js** (TypeScript) | App Router, Tailwind CSS |
| API communication | HTTP | REST preferred, GraphQL when appropriate |

Default to Go for backend, Next.js for frontend. Ask before using anything else.

---

## Mandatory Code Patterns

See `references/examples.md#go-dependency-injection-examples` for full code samples.

### 1. Dependency Injection via Interfaces

Every service, repository, and external dependency MUST be accessed through an interface. No concrete types in function signatures except at the composition root (main.go). Interfaces are defined by the consumer, not the provider.

**Why:** Testability (mock any dependency), swappability (Postgres today, CockroachDB tomorrow), and clear contracts for the handoff developer.

### 2. Context Propagation

Every function that performs I/O MUST accept `context.Context` as its first parameter. No exceptions.

### 3. Error Handling

Wrap errors with context: `fmt.Errorf("failed to save order %s: %w", id, err)`. Never discard errors silently. Never use `panic` for recoverable errors.

### 4. Structured Logging

Use `log/slog` (Go 1.21+). Never `fmt.Println` or `log.Printf` for operational output. JSON in production. Every service constructor accepts a `*slog.Logger` for scoped, testable logging.

See `references/examples.md#structured-logging-examples`.

### 5. Configuration Management

Non-secret config: environment variables loaded into a typed config struct at startup. Secrets: always from Vault, never from environment variables.

See `references/examples.md#configuration-management-pattern`.

### 6. Graceful Shutdown

Every Go HTTP server MUST handle graceful shutdown. In-flight requests complete before the process exits. Use `signal.Notify` for SIGINT/SIGTERM and `srv.Shutdown(ctx)` with a 30-second timeout.

See `references/examples.md#graceful-shutdown-pattern`.

---

## Testing Requirements

Testing is non-negotiable. Untested code is not ready for handoff.

### Test Naming Convention

Test names MUST be self-documenting using the pattern: `Test<Function>_<Scenario>_<ExpectedBehavior>`. A receiving developer inheriting 200 tests should understand what each tests from the name alone.

Good: `TestGetOrder_RepoTimeout_ReturnsWrappedError`, `TestScore_ThreeIndexMatch_CorroborationBonus`
Bad: `TestGetOrder2`, `TestScoreHappy`, `Test1`

### Unit Tests

Every package MUST have unit tests. Tests use interfaces and mocks — never real databases or external services. Table-driven tests are preferred for multiple cases.

See `references/examples.md#go-unit-test-patterns`.

### Error Path Testing (Critical)

Every test suite MUST include error path tests. If a function can fail, test that it fails correctly. Test these scenarios for every service:
- Repository/database failures
- External service timeouts (use `context.WithTimeout`)
- Invalid input (nil, empty strings, malformed data)
- Not-found cases (distinct from errors)
- Partial failures (one dependency fails, others succeed)

See `references/examples.md#go-error-path-testing`.

### Test Helpers and Fixtures

Create reusable test helpers in a `testutil` package. Use functional option overrides for test fixture construction. Avoid duplicating setup across test files.

See `references/examples.md#go-test-helpers-and-fixtures`.

### Interface Compliance Tests

Add compile-time interface checks: `var _ Interface = (*Impl)(nil)`. This catches interface drift immediately.

### Integration Tests

Use build tags (`//go:build integration`) to separate from unit tests. Run with `go test -tags=integration`. Integration tests may use real databases (Docker/testcontainers) and Vault dev servers. They MUST clean up after themselves.

### Frontend Testing (Next.js / TypeScript)

Use **Vitest** (unit), **React Testing Library** (components), **MSW** (API mocking). Every API client function MUST have tests covering success and error responses.

See `references/examples.md#frontend-testing-nextjs`.

### HTTP Handler / API Layer Testing (Go)

Every HTTP handler MUST be tested using `net/http/httptest`. Test the full request/response cycle: status codes, response bodies, error responses, and middleware behavior (auth, validation). These tests catch serialization bugs, routing errors, and middleware misconfigurations that unit tests on the service layer miss.

See `references/examples.md#go-http-handler-testing`.

### Rust Testing

For Rust projects, follow these conventions:
- Unit tests in the same file as the code (`#[cfg(test)] mod tests`)
- Integration tests in `tests/` directory
- Use `#[tokio::test]` for async tests
- Use `mockall` crate for trait mocking
- Same error path and naming conventions as Go: `test_<function>_<scenario>_<expected>`
- `cargo test` must pass before any commit

See `references/examples.md#rust-testing`.

### Test Coverage

Aim for 80%+ on business logic. Error paths count and are often more important than happy paths. Don't chase 100%.

---

## Documentation Requirements

Documentation is as important as code. A prototype without docs is not ready for handoff.

### Inline Documentation

Every exported function, type, and constant MUST have a doc comment. Comments explain *why*, not just *what*. Never restate the function signature.

When a non-obvious choice was made, document it with a `DESIGN DECISION:` prefix so the receiving developer understands the reasoning.

### Project Documentation Structure

Every project MUST have:

```
docs/
├── index.md                    # TOC + machine-parseable links (REQUIRED)
├── quickstart.md               # Zero to running in <10 min (REQUIRED)
├── developer-guide.md          # Architecture, patterns, API ref (REQUIRED)
├── admin-guide.md              # Ops: deployment, config, backup (WHEN NEEDED)
└── decisions/                  # Architecture Decision Records (REQUIRED for non-trivial choices)
    └── 001-some-decision.md
```

See `references/examples.md#documentation-index-template` for the index.md format.

**quickstart.md**: Prerequisites, install, first run, verify. No architecture explanations — just the steps. MUST include:
- Every external dependency and how to install it (Vault, database, etc.)
- A `.env.example` file or equivalent documenting every environment variable
- Database seeding or migration commands
- How to start dependent services (e.g., `vault server -dev`, `docker compose up`)
- A "verify it works" step (curl command, test script, or browser URL)

A developer cloning the repo should have zero ambiguity about how to get running.

**developer-guide.md**: Architecture overview, project structure, code patterns, how to add features, API reference, testing guide. The receiving developer reads this first.

**admin-guide.md**: Create when operational concerns exist (Vault config, migrations, backups, monitoring). Not every prototype needs this — ask Pete.

### Architecture Decision Records (Required for non-trivial choices)

Any decision involving technology selection, security boundaries, data model design, or trade-offs MUST be documented as an ADR in `docs/decisions/`. Use sequential numbering and the standard format.

See `references/examples.md#adr-template`.

Examples of decisions that require ADRs:
- Database choice (Oracle vs PostgreSQL)
- Vault custom plugin vs transit engine
- Bloom filter parameters and similarity thresholds
- API authentication approach
- Choice to use phonetic encoding for fuzzy matching

### CHANGELOG.md (Required)

Every project MUST maintain a CHANGELOG using Keep a Changelog format. Update with every version bump. Categories: Added, Changed, Deprecated, Removed, Fixed, Security.

See `references/examples.md#changelog-template`.

### API Documentation (Required for HTTP endpoints)

Document every endpoint with: path, method, request schema + example, response schema + example, error codes, auth requirements. Document in developer-guide.md for prototypes.

See `references/examples.md#api-documentation-template`.

### README.md (Required)

Follow the standard template: one-line description, quick start, architecture diagram, dev commands, doc links, version.

See `references/examples.md#readme-template`.

### Markdown Formatting for Export

Docs must render cleanly as PDF (pandoc) and be parseable by LLMs:
- ATX headers (`#`, `##`, `###`) consistently
- Fenced code blocks with language identifiers
- Tables for structured data
- Lines under 100 characters
- Relative links between docs
- YAML front matter for PDF metadata (`title`, `version`, `date`)

---

## Versioning

Semantic Versioning starting at `v0.0.1`.

| Component | When to Bump | Permission |
|-----------|--------------|------------|
| Patch (0.0.x) | Bug fixes, docs, minor improvements | **Automatic** |
| Minor (0.x.0) | New features, non-breaking API changes | **Ask Pete** |
| Major (x.0.0) | Breaking API changes, architecture shifts | **Ask Pete** |

Version lives in: `VERSION` file (source of truth), `go.mod`/`Cargo.toml`, git tag.

---

## Project Structure

### Go Projects

```
project-name/
├── cmd/
│   ├── server/main.go          # API server entry point
│   └── cli/main.go             # CLI tool (if needed)
├── internal/                   # Private packages
│   ├── config/                 # Typed configuration
│   ├── model/                  # Domain types
│   ├── service/                # Business logic (depends on interfaces)
│   ├── repository/             # Data access (implements interfaces)
│   └── testutil/               # Shared test helpers and fixtures
├── pkg/api/                    # HTTP handlers, middleware, request/response
├── db/
│   ├── schema.sql
│   └── migrations/
├── docs/                       # See documentation structure above
├── test/integration/           # Integration tests (build-tagged)
├── go.mod
├── VERSION
├── CHANGELOG.md
├── Dockerfile
└── README.md
```

### Next.js Frontend

```
project-name-ui/
├── src/
│   ├── app/                    # App Router pages
│   ├── components/             # Reusable components
│   ├── lib/                    # Utilities, API client
│   ├── hooks/                  # Custom hooks
│   └── types/                  # TypeScript types
├── docs/
├── package.json
├── VERSION
├── CHANGELOG.md
└── README.md
```

### API Communication

Frontend talks to backend via HTTP with a typed API client. See `references/examples.md#nextjs-api-client-pattern`.

---

## Checklist Before Handing Off Code

Before presenting code to Pete, verify:

- [ ] All exported functions have doc comments explaining *why*
- [ ] Design decisions documented with `DESIGN DECISION:` prefix
- [ ] All dependencies injected via interfaces
- [ ] All I/O functions accept `context.Context`
- [ ] Errors wrapped with context (`fmt.Errorf("...: %w", err)`)
- [ ] Structured logging via `slog` (no fmt.Println/log.Printf)
- [ ] Graceful shutdown implemented for HTTP servers
- [ ] Unit tests exist for business logic, **including error paths**
- [ ] Interface compliance checks (`var _ Interface = (*Impl)(nil)`)
- [ ] Integration test stubs exist
- [ ] `docs/index.md`, `docs/quickstart.md`, `docs/developer-guide.md` exist
- [ ] API endpoints documented with request/response examples
- [ ] `CHANGELOG.md` updated with current changes
- [ ] `VERSION` file exists with current version
- [ ] `README.md` follows the template structure
