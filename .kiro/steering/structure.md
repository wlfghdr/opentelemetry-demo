## Project Structure

### Root Level

- `src/` - All microservice source code
- `pb/` - Shared protobuf definitions
- `test/` - Integration and trace-based tests
- `kubernetes/` - Kubernetes deployment manifests
- `internal/tools/` - Build and development tools
- `docker-compose.yml` - Main service orchestration
- `docker-compose.minimal.yml` - Minimal deployment
- `docker-compose-tests.yml` - Test environment
- `Makefile` - Build automation and common tasks

### Service Structure

Each service in `src/` follows a consistent pattern:

```
src/<service-name>/
├── Dockerfile           # Container build definition
├── README.md           # Service-specific documentation
├── <language-files>    # Source code in service's language
└── genproto/           # Generated protobuf code (if applicable)
```

### Service Categories

**Core Business Services**:
- `frontend/` - Next.js web application and API gateway
- `product-catalog/` - Product inventory (Go)
- `cart/` - Shopping cart management (C#)
- `checkout/` - Order processing (Go)
- `payment/` - Payment processing (Node.js)
- `shipping/` - Shipping calculations (Rust)
- `recommendation/` - Product recommendations (Python)
- `ad/` - Advertisement service (Java)
- `email/` - Email notifications (Ruby)
- `quote/` - Shipping quotes (PHP)
- `currency/` - Currency conversion (C++)
- `product-reviews/` - Product reviews with LLM (Python)

**Supporting Services**:
- `accounting/` - Order accounting (C#)
- `fraud-detection/` - Fraud detection (Kotlin)
- `llm/` - Mock LLM service (Python)
- `load-generator/` - Traffic generation (Python/Locust)

**Infrastructure Services**:
- `frontend-proxy/` - Envoy proxy configuration
- `image-provider/` - Static image serving (nginx)
- `flagd/` - Feature flag configuration
- `flagd-ui/` - Feature flag management UI (Elixir)
- `kafka/` - Message broker configuration
- `postgres/` - Database initialization
- `otel-collector/` - OpenTelemetry Collector config
- `grafana/` - Grafana dashboards and datasources
- `jaeger/` - Jaeger configuration
- `prometheus/` - Prometheus configuration
- `opensearch/` - OpenSearch configuration

### Testing

```
test/tracetesting/
├── <service>/          # Service-specific trace tests
├── cli-config.yml      # Tracetest CLI configuration
└── run.bash           # Test execution script
```

### Conventions

- Each service has its own Dockerfile in its directory
- Services communicate via gRPC using shared protobuf definitions
- All services export telemetry to OpenTelemetry Collector
- Feature flags are managed through flagd
- Environment variables are prefixed by service name (e.g., `CART_PORT`, `FRONTEND_ADDR`)
- Services use health checks for container orchestration
