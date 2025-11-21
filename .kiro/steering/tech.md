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

### Protobuf

Services use shared protobuf definitions from `/pb/demo.proto`. Generate code with `make docker-generate-protobuf` or `make generate-protobuf` (local).

### Environment Configuration

- `.env`: Default environment variables
- `.env.override`: Local overrides (not committed)
- `.env.arm64`: macOS ARM64 specific settings
