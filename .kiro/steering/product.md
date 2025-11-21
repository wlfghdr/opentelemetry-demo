## OpenTelemetry Astronomy Shop Demo

A microservice-based distributed system demonstrating OpenTelemetry instrumentation and observability in a near real-world e-commerce environment.

### Purpose

- Provide a realistic example of distributed system observability using OpenTelemetry
- Serve as a base for vendors and tooling authors to demonstrate OpenTelemetry integrations
- Act as a living example for testing new OpenTelemetry API, SDK, and component versions

### Key Features

- Multi-language microservices architecture (15+ services)
- Full OpenTelemetry instrumentation (traces, metrics, logs)
- Feature flag management via flagd
- Load generation and testing capabilities
- Observability stack (Jaeger, Grafana, Prometheus, OpenSearch)
- Kafka-based event streaming
- AI/LLM integration for product reviews

### Services

Core services include: frontend (Next.js), product-catalog, cart, checkout, payment, shipping, recommendation, ad, email, accounting, fraud-detection, product-reviews, quote, currency, and supporting infrastructure (Kafka, Valkey, PostgreSQL).
