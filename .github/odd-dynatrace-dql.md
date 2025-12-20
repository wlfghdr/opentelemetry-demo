# ODD Dynatrace DQL templates

This doc contains DQL snippets referenced by `.github/copilot-instructions.md`.
Use short windows for triage and longer windows for trend verification.

## Step 0: Discover the right filters (don’t guess)

Learn real values for `k8s.namespace.name`, `k8s.deployment.name`, and/or `dt.entity.service`.

### Discover active Kubernetes namespaces

```dql
fetch logs
| filter timestamp > now() - 1h
| summarize logCount = count(), by:{k8s.namespace.name}
| sort logCount desc
```

### Discover active deployments within a namespace

```dql
fetch logs
| filter timestamp > now() - 1h
   and k8s.namespace.name == "<YOUR_NAMESPACE>"
| summarize logCount = count(), by:{k8s.deployment.name}
| sort logCount desc
```

### If service entities are used instead of k8s fields

```dql
fetch dt.entity.service
| fields id, entity.name
| sort entity.name asc
```

## Kubernetes CPU/Memory by deployment

```dql
timeseries avg(dt.kubernetes.container.cpu_usage),
   avg(dt.kubernetes.container.memory_working_set),
   from: now() - 1h, interval: 1m,
   filter: k8s.namespace.name == "<YOUR_NAMESPACE>",
   by: {k8s.deployment.name}
```

## Errors in logs by deployment

```dql
fetch logs
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
   and loglevel == "ERROR"
   and timestamp > now() - 1h
| summarize errorCount = count(), by:{k8s.deployment.name}
| sort errorCount desc
```

## Service response time + failures (service metrics)

```dql
timeseries avg(dt.service.request.response_time),
   percentile(dt.service.request.response_time, 95),
   sum(dt.service.request.failure_count),
   from: now() - 1h, interval: 1m,
   // Optional: add a filter for the specific service/entity naming scheme in your environment
   by: {dt.entity.service}
```

## Slow spans (find hot operations)

```dql
fetch spans
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
   and timestamp > now() - 30m
   and duration > 1000000000
| summarize p95 = percentile(duration, 95), cnt = count(), by:{dt.entity.service, span.name}
| sort p95 desc
```
