# API Gateway — Security, Rate-Limit & Cost Consolidation

> An open-source portfolio project exploring a secure API proxy/gateway for consolidating Various Vendors API access, reducing redundant upstream requests, protecting credentials, managing rate limits, and improving operational visibility.

[![Status](https://img.shields.io/badge/status-early%20design-blue)](#project-status)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Project Status

**Early architecture / design phase**

This repository is intentionally being developed as a portfolio engineering project. The goal is not to present a finished production platform on day one, but to demonstrate the process of taking an older API-caching concept and rethinking it with modern infrastructure, security, observability, deployment, and software-engineering practices.

The initial concept came from a small Python/PostgreSQL JSON caching framework. That prototype established the core idea of putting a controlled gateway between developers/applications and an upstream API, with credentials kept on the gateway and repeat requests served from a cache.

The original blueprint proposed Python/FastAPI, PostgreSQL JSONB, and Docker as a modernized implementation path. Those choices are starting hypotheses—not architectural decisions that this project is committed to keeping.

## Why This Project Exists

Modern development teams frequently have many tools, scripts, services, CI jobs, dashboards, and developers consuming the same external APIs.

That can create several problems:

- Multiple clients repeatedly request identical or nearly identical data
- Upstream API rate limits can become a shared operational constraint
- Credentials and vendor tokens can end up distributed across too many clients
- Unnecessary upstream traffic can increase latency and, depending on the service, cost
- Different development tools may implement API access differently
- It can be difficult to understand who is consuming an API, how often, and why?
- A lack of centralized policy makes it harder to introduce caching, throttling, authentication, auditing, and observability consistently

This project explores whether a lightweight API gateway/proxy can provide a useful consolidation layer without becoming an unnecessarily complicated platform.

## Core Goals

### 1. LLM Integration, Cost and Optimization
- Track prompt and completion tokens per client to monitor and control upstream LLM vendor spend
- Passthrough Server-Sent Events (SSE) and token-streaming responses without buffering or extra latency
- Cache exact and semantic prompt results to avoid redundant, expensive model calls
- Unify request and response formats across OpenAI, Anthropic, and other vendors to make swapping models seamless
- Route requests to secondary models or providers automatically when primary endpoints hit rate limits or downtime
- Intercept requests to redact sensitive PII and enforce policy constraints before reaching upstream models

### 2. Security Consolidation

Provide a controlled boundary between internal clients and Vendor APIs.

Potential responsibilities include:

- Keeping Vendor API credentials/tokens on the gateway rather than distributing them to every client
- Injecting upstream authentication only when required
- Preventing credentials from being returned to downstream clients
- Applying authentication and authorization policies at the gateway
- Providing a foundation for auditing and security logging
- Supporting least-privilege access patterns

**Important:** This project should never contain real API credentials, tokens, secrets, or production account information.

### 3. Rate-Limit Management

Explore ways to make upstream API consumption more predictable.

Potential capabilities:

- Request throttling
- Per-client rate limits
- Shared upstream rate-limit awareness
- Cache-assisted request reduction
- Request coalescing for identical simultaneous requests
- Backoff handling
- Visibility into upstream rate-limit consumption

The objective is not to magically eliminate upstream limits. The objective is to use them intelligently and avoid unnecessary consumption.

### 4. Response Caching

Cache safe-to-cache Vendor API responses so repeated requests can potentially be served locally.

The initial design will investigate:

- PostgreSQL JSONB
- Redis or another dedicated cache
- In-memory caching
- HTTP-aware caching
- TTL-based expiration
- ETags / conditional requests where appropriate
- Cache invalidation
- Cache key design
- Cache size and retention
- Serialization/deserialization overhead

PostgreSQL JSONB is an explicit candidate because the original prototype used PostgreSQL to store structured API responses. It is **not yet the selected solution**.

### 5. Cost and Resource Awareness

The gateway should make API consumption measurable.

Potential metrics include:

- Upstream requests avoided through caching
- Cache hit/miss ratio
- Requests by client
- Requests by endpoint
- Upstream response latency
- Gateway response latency
- Rate-limit consumption
- Error rates
- Approximate infrastructure/resource cost

The project will distinguish between **actual provider costs** and **resource/cost proxies** rather than claiming that every avoided request has a direct dollar value.

### 6. Developer-Friendly API Access

The gateway should be useful to developers with different tooling preferences.

A major design question is how to expose the service cleanly enough that it can be consumed from:

- Python
- Go
- JavaScript / TypeScript
- Shell / curl
- PowerShell
- CI/CD systems
- Other internal tooling

The gateway itself should not require every downstream developer to adopt the same programming language.

## Architecture — Initial Hypothesis

The first conceptual architecture is:

```text
+-------------------+
| Developer / Tool  |
| CI / Script / App |
+---------+---------+
          |
          | Internal API request
          v
+-----------------------------+
|      API Gateway / Proxy    |
|                             |
|  Authentication             |
|  Authorization              |
|  Rate limiting              |
|  Request normalization      |
|  Cache lookup               |
|  Observability              |
+-------------+---------------+
              |
        +-----+------+
        |            |
        v            v
+---------------+  +----------------+
| Cache / Store |  | Vendor API     |
| JSON / Redis  |  | Upstream       |
| PostgreSQL?   |  |                |
+---------------+  +----------------+
```

This diagram is deliberately conceptual. The implementation will evolve as architectural decisions are tested.

## Architecture Questions We Need to Answer

Rather than assuming the first proposed architecture is correct, this project will document and test the trade-offs.

### Hosting / Runtime

Investigate:

- Docker container
- Docker Compose
- Standalone VM
- Kubernetes
- Lightweight Kubernetes distributions
- Bare-metal deployment
- Cloud/container hosting
- Hybrid approaches

Questions:

- What is the smallest sensible deployment?
- What operational complexity does each option introduce?
- What demonstrates useful DevOps knowledge without adding infrastructure purely for résumé value?
- How easily can the service be reproduced by another developer?

### Programming Language

The first prototype may use Python because it allows rapid iteration and has strong HTTP/API tooling.

Potential alternatives or future implementations may include:

- Python / FastAPI
- Go
- Rust
- TypeScript / Node.js

The language decision should be driven by:

- Development speed.
- Runtime characteristics.
- Concurrency model.
- Maintainability.
- Ecosystem maturity.
- Developer accessibility.
- Operational simplicity.
- Suitability for a gateway/proxy workload.

A multi-language implementation is **not a goal by itself**. Additional implementations should exist only when they demonstrate a meaningful architectural or developer-experience benefit.

### Cache / Persistence

Initial candidates:

| Option | Questions to investigate |
|---|---|
| PostgreSQL JSONB | Can one database provide sufficient persistence, indexing, TTL management, and operational simplicity? |
| Redis | Does a dedicated cache provide enough performance/feature advantages to justify another service? |
| In-memory | Is it sufficient for a single-node prototype? What happens after restart? |
| HTTP cache semantics | Can upstream HTTP mechanisms reduce unnecessary storage and improve correctness? |
| Hybrid | Should persistent metadata and hot response caching be separate concerns? |

The project should measure rather than assume performance differences.

## Security Model

Security is a first-class design concern.

The gateway should be designed around the principle that downstream clients do **not** need direct access to upstream credentials.

Planned areas of investigation:

- Secret management.
- Environment variables for local development.
- Docker/Kubernetes secret mechanisms where appropriate.
- Token rotation.
- Least privilege.
- Client authentication.
- Authorization.
- Request validation.
- Audit logging.
- Log redaction.
- TLS.
- Dependency security.
- Container security.
- Supply-chain considerations.

No real credentials should ever be committed to this repository.

## Observability

A portfolio-quality implementation should make its behavior visible.

Potential observability stack:

- Structured application logs.
- Request IDs / correlation IDs.
- Metrics.
- Health checks.
- Readiness checks.
- OpenTelemetry.
- Prometheus-compatible metrics.
- Grafana or another visualization layer.

Example metrics:

```text
gateway_requests_total
gateway_cache_hits_total
gateway_cache_misses_total
gateway_upstream_requests_total
gateway_upstream_errors_total
gateway_request_duration_seconds
gateway_upstream_duration_seconds
gateway_rate_limit_remaining
```

The final metric set will be determined during implementation.

## Proposed Development Phases

### Phase 0 — Architecture

- Define the problem precisely.
- Document assumptions.
- Identify security boundaries.
- Compare deployment models.
- Compare language options.
- Compare caching strategies.
- Define initial API surface.

### Phase 1 — Minimal Working Gateway

Build the smallest useful gateway:

- Accept a request.
- Authenticate the downstream client.
- Authenticate with Vendors upstream.
- Forward a supported request.
- Return the upstream response.
- Never expose the upstream credential.
- Provide basic logging.

### Phase 2 — Caching

Add:

- Deterministic cache keys.
- TTL.
- Cache hit/miss tracking.
- JSON response storage.
- Cache invalidation strategy.
- Tests for stale and fresh data.

### Phase 3 — Rate-Limit Controls

Add:

- Client-side limits.
- Upstream limit awareness.
- Backoff behavior.
- Request coalescing where appropriate.
- Rate-limit metrics.

### Phase 4 — Observability

Add:

- Structured logs.
- Metrics.
- Health/readiness endpoints.
- Request correlation.
- Optional tracing.

### Phase 5 — Containerization & Deployment

Create reproducible deployments using the selected hosting model.

Potential deliverables:

- Dockerfile.
- Compose configuration.
- Environment configuration examples.
- Health checks.
- CI pipeline.
- Security scanning.
- Deployment documentation.

### Phase 6 — Engineering Hardening

Investigate:

- Load testing.
- Failure behavior.
- Upstream outages.
- Cache corruption.
- Database failure.
- Credential rotation.
- Concurrent requests.
- Race conditions.
- Dependency vulnerabilities.
- Resource limits.

## Testing Strategy

Testing should be part of the architecture rather than an afterthought.

Potential test layers:

```text
Unit Tests
    |
    +-- cache-key generation
    +-- TTL behavior
    +-- rate-limit calculations
    +-- authentication policy
    +-- request validation

Integration Tests
    |
    +-- gateway + database
    +-- gateway + cache
    +-- gateway + mocked Vendor API

End-to-End Tests
    |
    +-- client -> gateway -> mocked upstream

Operational Tests
    |
    +-- container startup
    +-- health checks
    +-- dependency failure
    +-- upstream failure
    +-- load behavior
```

External Vendor APIs should not be required for ordinary automated tests.

## Repository Structure

The structure will evolve, but an initial target could look like:

```text
.
├── README.md
├── LICENSE
├── SECURITY.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── pyproject.toml
├── Dockerfile
├── compose.yaml
├── .env.example
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── decisions/
│   ├── caching.md
│   ├── security.md
│   └── deployment.md
├── src/
│   └── gateway/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── .github/
    └── workflows/
```

## Architecture Decision Records

Important technical decisions should be documented rather than buried in commit history.

Potential ADRs:

- ADR-001: Initial gateway architecture
- ADR-002: Python vs. Go vs. other implementation languages
- ADR-003: PostgreSQL JSONB vs. Redis
- ADR-004: Docker vs. VM vs. Kubernetes
- ADR-005: Authentication model
- ADR-006: Cache invalidation strategy
- ADR-007: Observability approach

Each decision should record:

1. Context
2. Options considered
3. Decision
4. Trade-offs
5. Consequences
6. What would cause us to reconsider it

## What This Project Demonstrates

This repository is intentionally broader than a simple API wrapper.

It is intended to demonstrate practical engineering across:

- API design
- Reverse proxies
- HTTP fundamentals
- Authentication and authorization
- Secrets management
- Caching
- Rate limiting
- PostgreSQL
- JSON/JSONB data modeling
- Containerization
- Linux/server concepts
- Networking
- Observability
- Automated testing
- CI/CD
- Infrastructure decisions
- Security practices
- Technical documentation
- Architecture decision-making

The goal is to demonstrate **engineering judgment**, not simply the number of technologies used.

## Portfolio Context

This project is being developed as a portfolio piece to demonstrate a return to hands-on technical engineering.

It is particularly intended to showcase skills relevant to roles involving:

- DevOps
- Platform engineering
- Infrastructure
- Systems administration / engineering
- Backend development
- Entry-to-mid-career software development
- Automation
- Advanced IT engineering

The project should therefore favor understandable engineering decisions, reproducibility, testing, documentation, and operational awareness over unnecessary complexity.

## Non-Goals

This project is not initially intended to become:

- A general-purpose enterprise API management platform.
- A replacement for Vendors API.
- A hosted commercial service.
- A system designed around proprietary infrastructure.
- A demonstration of every available DevOps technology.
- A benchmark optimized solely for impressive numbers.

If a simpler architecture solves the problem, simplicity wins.

## Open Source Intent

This project is intended to remain free and openly available.

The author intends to permit others to:

- Study the source.
- Run it themselves.
- Modify it.
- Learn from it.
- Use it in their own projects.
- Contribute improvements.

The repository will use a permissive open-source license. **MIT is currently the leading candidate**, with BSD-2-Clause and BSD-3-Clause also under consideration.

This project is officially licensed under MIT.

## Current Working Name

**API Gateway — Security, Rate-Limit & Cost Consolidation**

Possible shorter repository names:

```text
api-gateway
api-proxy
api-cache-gateway
api-control-plane
```

The final repository name can be selected once the architecture becomes clearer.

## Current Questions

The project should begin by answering these questions:

1. What API workloads actually benefit from caching?
2. Which responses are safe and correct to cache?
3. How should cache keys represent authenticated users, repositories, endpoints, query parameters, and headers?
4. How should stale data be handled?
5. When is PostgreSQL JSONB preferable to Redis?
6. Do we need both persistence and a dedicated cache?
7. What is the smallest useful deployment?
8. Is Docker sufficient, or does another runtime make more sense?
9. Does Python/FastAPI remain the best first implementation?
10. What would justify a Go, Rust, or TypeScript implementation?
11. How should downstream clients authenticate to the gateway?
12. How should upstream credentials be stored and rotated?
13. How should rate limits be represented and enforced?
14. What observability data is actually useful?
15. How do we demonstrate measurable improvement without manufacturing performance claims?

## Guiding Principle

> Build the smallest system that proves the architectural idea, measure it, document the trade-offs, and only add complexity when the problem justifies it.

---

## Disclaimer

This is an educational and portfolio project.

Before deploying the project against real accounts or production workloads, review vendors current API documentation, authentication requirements, rate limits, terms, and security recommendations.
