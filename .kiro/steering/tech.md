## Technology Stack

### Languages & Frameworks

The demo showcases OpenTelemetry across multiple languages:

- **Go**: checkout, product-catalog, flagd (gRPC services)
- **.NET/C#**: accounting, cart (ASP.NET Core, Entity Framework)
- **Java**: ad, fraud-detection (Gradle, gRPC)
- **Node.js/TypeScript**: frontend (Next.js), payment (Express)
- **Python**: recommendation, product-reviews, llm, load-generator (Flask, gRPC)
- **Ruby**: email (Sinatra)
- **Rust**: shipping (Actix-web, Tonic)
- **PHP**: quote (Slim framework)
- **C++**: currency (gRPC)
- **Elixir**: flagd-ui (Phoenix)

### Infrastructure

- **Container orchestration**: Docker Compose, Kubernetes (Helm charts)
- **Message broker**: Kafka
- **Databases**: PostgreSQL, Valkey (Redis fork)
- **Observability**: OpenTelemetry Collector, Jaeger, Grafana, Prometheus, OpenSearch
- **Feature flags**: flagd (OpenFeature)
- **Proxy**: Envoy (frontend-proxy)
- **Load testing**: Locust

### Build System

- **Primary**: Docker Compose with multi-stage builds
- **Makefile** for common tasks
- **Language-specific**: Gradle (Java), npm (Node.js), Go modules, Cargo (Rust), dotnet CLI, pip (Python)

### Common Commands

```bash
# Start the full demo
make start

# Start minimal version (fewer services)
make start-minimal

# Stop all services
make stop

# Build all images
make build

# Run tests
make run-tests

# Rebuild and restart a single service
make redeploy service=frontend

# Generate protobuf files
make docker-generate-protobuf

# Code quality checks
make check  # runs misspell, markdownlint, checklicense, checklinks
```

### Development

```bash
# Build specific service
docker compose build <service-name>

# View logs
docker logs <container-name>

# Restart single service
make restart service=<service-name>
```

## Observability-Driven Development (ODD)

ODD is a non-negotiable part of change implementation in this repo: behavior changes are not “done” unless they can be **validated via telemetry**.

### Guardrails (apply to code changes)

- **Instrument + verify:** for non-trivial changes, ensure traces/metrics/logs exist to evaluate impact (latency, error rate, resource usage).
- **Tier-based rigor:**
	- Tier 0 (docs/refactor): minimal validation.
	- Tier 1 (non-hot-path): targeted tests + basic telemetry.
	- Tier 2 (hot-path / sync downstream call / retry/payload changes): baseline comparison required (Dynatrace preferred), strict timeouts/budgets, and explicit rollback triggers.
- **High-cardinality safety:** do not add unbounded labels/attributes (raw user input, emails, full IDs, unique URLs) to metrics or span attributes.
- **Baseline required (Tier 2):** provide before/after evidence via Dynatrace DQL or local load + p95/error rate comparison.

### References (within Kiro)

- `.kiro/steering/dynatrace-mcp.md` (Dynatrace MCP usage + DQL templates)

### Protobuf

Services use shared protobuf definitions from `/pb/demo.proto`. Generate code with `make docker-generate-protobuf` or `make generate-protobuf` (local).

### Environment Configuration

- `.env`: Default environment variables
- `.env.override`: Local overrides (not committed)
- `.env.arm64`: macOS ARM64 specific settings
