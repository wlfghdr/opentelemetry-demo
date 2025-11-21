# Cart Service Monitoring Hook

## Overview

This hook provides **continuous, automated monitoring** of Cart Service resource usage during development of the inventory validation feature. It runs automatically whenever you save a C# file in the cart service, querying Dynatrace for current metrics and alerting if resources are approaching saturation.

## Purpose

**Problem:** Developers might not realize they're pushing a service toward resource limits until it's too late.

**Solution:** Automated, real-time feedback using Dynatrace observability data during development.

## How It Works

### Trigger
- **Event:** File save
- **Pattern:** `src/cart/**/*.cs`
- **Frequency:** Every time you save a cart service C# file

### Action
1. Queries Dynatrace for Cart Service metrics (last 15 minutes)
2. Checks CPU usage, memory usage, and error rates
3. Compares against warning thresholds
4. Alerts if approaching resource limits

### Thresholds

| Metric | Warning Threshold | Rationale |
|--------|------------------|-----------|
| CPU | > 25% | Current baseline is 20%, leaving only 5% headroom |
| Memory | > 200MB | Current baseline is 168MB, 200MB indicates growth |
| Error Rate | > 0.1% | Any errors during development indicate issues |

## Hook Configuration

**Location:** `.kiro/hooks/cart-service-monitor.json`

```json
{
  "name": "Cart Service Resource Monitor",
  "trigger": {
    "type": "onFileSave",
    "filePattern": "src/cart/**/*.cs"
  },
  "action": {
    "type": "sendMessage",
    "message": "Check Cart Service resource usage..."
  }
}
```

## Example Alerts

### ✅ Healthy State
```
Cart Service Metrics (Last 15 min):
- CPU: 19.2% ✓ (Below 25% threshold)
- Memory: 171MB ✓ (Below 200MB threshold)
- Errors: 0 ✓ (No errors)

Status: HEALTHY - Safe to continue development
```

### ⚠️ Warning State
```
⚠️ RESOURCE WARNING

Cart Service Metrics (Last 15 min):
- CPU: 26.8% ⚠️ (Above 25% threshold)
- Memory: 189MB ✓ (Below 200MB threshold)
- Errors: 0 ✓ (No errors)

Recommendation:
- CPU usage is elevated during development
- Consider pausing to investigate CPU increase
- May need to increase resource allocation before proceeding
- Review recent code changes for inefficiencies
```

### 🚨 Critical State
```
🚨 RESOURCE CRITICAL

Cart Service Metrics (Last 15 min):
- CPU: 28.4% 🚨 (Well above 25% threshold)
- Memory: 215MB 🚨 (Above 200MB threshold)
- Errors: 3 🚨 (Errors detected)

STOP DEVELOPMENT:
- Service is approaching saturation
- Memory growth detected
- Errors appearing during development
- MUST increase resources before continuing
- Review and rollback recent changes if needed
```

## Dynatrace Queries Used

### CPU Usage
```dql
timeseries avg(dt.kubernetes.container.cpu_usage),
  from: now() - 15m, interval: 1m,
  filter: contains(k8s.deployment.name, "cart")
```

### Memory Usage
```dql
timeseries avg(dt.kubernetes.container.memory_working_set),
  from: now() - 15m, interval: 1m,
  filter: contains(k8s.deployment.name, "cart")
```

### Error Count
```dql
fetch logs
| filter contains(k8s.deployment.name, "cart")
  and loglevel == "ERROR"
  and timestamp > now() - 15m
| summarize cnt = count()
```

## Benefits

### 1. **Proactive Problem Detection**
- Catch resource issues during development, not in production
- Immediate feedback on code impact
- Prevent "works on my machine" scenarios

### 2. **Data-Driven Development**
- Objective metrics guide decisions
- No guessing about performance impact
- Clear thresholds for action

### 3. **Continuous Validation**
- Every save triggers validation
- No manual monitoring needed
- Automatic safety net

### 4. **Learning Tool**
- Developers see real-time impact of their code
- Builds awareness of resource consumption
- Encourages performance-conscious coding

## Usage During Development

### Step 1: Enable the Hook
The hook is automatically enabled when you start working on the cart service.

### Step 2: Develop Normally
Write code, save files as usual. The hook runs automatically.

### Step 3: Review Alerts
When you save a file, check the AI response for resource metrics.

### Step 4: Take Action
- **Green:** Continue development
- **Yellow:** Investigate and optimize
- **Red:** Stop and increase resources

## Integration with Tasks

This hook complements the explicit Dynatrace validation tasks:

| Task | Type | When | Purpose |
|------|------|------|---------|
| Task 0.1-0.3 | Manual | Before starting | Pre-flight check |
| **Hook** | **Automatic** | **During development** | **Continuous monitoring** |
| Task 7.1-7.2 | Manual | Mid-implementation | Impact assessment |
| Task 11.1-11.4 | Manual | Before deployment | Final validation |

## Customization

### Adjust Thresholds
Edit `.kiro/hooks/cart-service-monitor.json`:

```json
"thresholds": {
  "cpu_warning": "30%",      // Increase if too sensitive
  "memory_warning": "250MB",  // Increase if too sensitive
  "error_rate_warning": "0.5%" // Increase if too sensitive
}
```

### Change Time Window
Modify the Dynatrace queries to use different time ranges:

```json
"from: now() - 30m"  // 30 minutes instead of 15
```

### Disable Hook
Set `"enabled": false` in the hook configuration.

## Troubleshooting

### Hook Not Triggering
1. Verify hook is enabled: Check `"enabled": true`
2. Verify file pattern matches: Save a file in `src/cart/src/services/`
3. Check Kiro hook UI for status

### False Positives
1. Increase thresholds if too sensitive
2. Increase time window to smooth out spikes
3. Add filters to exclude test environments

### Dynatrace Connection Issues
1. Verify Dynatrace MCP is configured
2. Check MCP connection in Kiro settings
3. Test queries manually in Dynatrace

## Best Practices

### 1. **Don't Ignore Warnings**
If the hook alerts, investigate before continuing. Warnings indicate real issues.

### 2. **Track Trends**
Watch for gradual increases over time, not just absolute values.

### 3. **Correlate with Changes**
When metrics spike, review what code you just changed.

### 4. **Share Insights**
If you find a pattern (e.g., "parsing JSON increases CPU by X%"), document it.

### 5. **Adjust as Needed**
If thresholds are too sensitive or not sensitive enough, adjust them.

## Demo Value

This hook demonstrates:

1. **Shift-Left Observability:** Monitoring during development, not just production
2. **Automated Guardrails:** AI prevents risky changes automatically
3. **Continuous Feedback:** Real-time impact assessment
4. **Proactive Culture:** Catch issues before they become problems

## Example Development Session

```
[Save CartService.cs]
✓ Cart Service: CPU 19%, Memory 172MB - Healthy

[Save InventoryValidator.cs - adds gRPC call]
⚠️ Cart Service: CPU 24%, Memory 185MB - Approaching limits

[Save CartService.cs - adds circuit breaker]
⚠️ Cart Service: CPU 26%, Memory 198MB - Warning threshold exceeded
Recommendation: Investigate CPU increase before continuing

[Optimize gRPC connection pooling]
✓ Cart Service: CPU 22%, Memory 190MB - Improved but still elevated

[Decision: Increase resource allocation]
✓ Resources increased: CPU limit 50% → 75%
✓ Cart Service: CPU 22% (of new limit) - Healthy headroom restored
```

---

**Remember:** This hook is a safety net, not a replacement for good engineering practices. Use it as a tool to inform decisions, not as the sole decision maker.
