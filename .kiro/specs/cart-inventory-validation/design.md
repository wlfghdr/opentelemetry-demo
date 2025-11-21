# Design Document: Cart Inventory Validation

## Overview

This design adds real-time inventory validation to the Cart Service's `AddItem` operation. When a user attempts to add a product to their cart, the service will synchronously call the Product Catalog Service to verify sufficient inventory exists before completing the operation. The feature is controlled by a feature flag and implements graceful degradation patterns to ensure availability.

## Architecture

### Current Architecture
```
User → Frontend → Cart Service → Valkey (Redis)
```

### Proposed Architecture
```
User → Frontend → Cart Service → Product Catalog Service (new)
                              ↓
                           Valkey (Redis)
```

### Component Interaction Flow

1. **AddItem Request Received**
   - Cart Service receives gRPC AddItem request
   - Extract userId, productId, quantity from request
   - Create distributed trace span

2. **Feature Flag Evaluation**
   - Query OpenFeature for "cartInventoryValidation" flag
   - If disabled: skip to step 5 (legacy behavior)
   - If enabled: proceed to step 3

3. **Inventory Validation** (NEW)
   - Create child span "ValidateInventory"
   - Call Product Catalog Service via gRPC
   - Request: GetProduct(productId)
   - Parse inventory from product response
   - Compare available vs requested quantity
   - Record validation metrics

4. **Validation Decision**
   - If sufficient: proceed to step 5
   - If insufficient: return error to user
   - If validation fails: log error, proceed to step 5 (fail open)

5. **Cart Update**
   - Call Valkey to add/update cart item
   - Set 60-minute expiration
   - Return success response

## Components and Interfaces

### Modified Component: CartService

**File:** `src/cart/src/services/CartService.cs`

**New Dependencies:**
```csharp
using Grpc.Net.Client;
using Oteldemo; // For ProductCatalogService client
using Polly; // For circuit breaker and retry policies
```

**New Fields:**
```csharp
private readonly string _productCatalogAddress;
private readonly GrpcChannel _productCatalogChannel;
private readonly ProductCatalogService.ProductCatalogServiceClient _productCatalogClient;
private readonly IAsyncPolicy<bool> _validationPolicy;
```

**New Method Signature:**
```csharp
private async Task<InventoryValidationResult> ValidateInventoryAsync(
    string productId, 
    int requestedQuantity, 
    CancellationToken cancellationToken
)
```

**Modified Method:**
```csharp
public override async Task<Empty> AddItem(
    AddItemRequest request, 
    ServerCallContext context
)
```

### New Component: InventoryValidator

**File:** `src/cart/src/services/InventoryValidator.cs`

**Purpose:** Encapsulate inventory validation logic and Product Catalog communication

**Interface:**
```csharp
public interface IInventoryValidator
{
    Task<InventoryValidationResult> ValidateAsync(
        string productId, 
        int requestedQuantity, 
        CancellationToken cancellationToken
    );
}

public class InventoryValidationResult
{
    public bool IsValid { get; set; }
    public int AvailableQuantity { get; set; }
    public string ErrorMessage { get; set; }
    public bool ValidationPerformed { get; set; }
}
```

### Configuration

**File:** `src/cart/src/appsettings.json`

**New Settings:**
```json
{
  "ProductCatalog": {
    "Address": "http://productcatalogservice:3550",
    "TimeoutMs": 500,
    "CircuitBreaker": {
      "FailureThreshold": 5,
      "SamplingDurationSeconds": 30,
      "MinimumThroughput": 10,
      "DurationOfBreakSeconds": 60
    }
  },
  "InventoryValidation": {
    "Enabled": false,
    "FailOpen": true,
    "MaxRetries": 1
  }
}
```

## Data Models

### Existing Models (No Changes)
- `AddItemRequest`: Contains userId, productId, quantity
- `Cart`: Contains userId and list of CartItems
- `CartItem`: Contains productId and quantity

### New Internal Models

**InventoryValidationResult:**
```csharp
public class InventoryValidationResult
{
    public bool IsValid { get; init; }
    public int AvailableQuantity { get; init; }
    public int RequestedQuantity { get; init; }
    public string ProductId { get; init; }
    public string ErrorMessage { get; init; }
    public bool ValidationPerformed { get; init; }
    public TimeSpan ValidationDuration { get; init; }
}
```

### Product Catalog Service Contract (Existing)

**Request:**
```protobuf
message GetProductRequest {
  string id = 1;
}
```

**Response:**
```protobuf
message Product {
  string id = 1;
  string name = 2;
  string description = 3;
  string picture = 4;
  Money price_usd = 5;
  repeated string categories = 6;
}
```

**Note:** Inventory information must be parsed from product description or added to Product message

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Validation occurs before cart modification

*For any* AddItem request with inventory validation enabled, the Product Catalog Service SHALL be called before any modifications are made to the cart in Valkey.

**Validates: Requirements 1.1**

**Test Strategy:** Verify through distributed tracing that the "ValidateInventory" span completes before the Valkey write operation span begins.

### Property 2: Insufficient inventory prevents cart addition

*For any* product with available inventory less than requested quantity, the AddItem operation SHALL fail and the cart SHALL remain unchanged.

**Validates: Requirements 1.3**

**Test Strategy:** Property-based test with random product IDs, random available inventory (0-100), and random requested quantities (1-200). Verify cart state unchanged when requested > available.

### Property 3: Sufficient inventory allows cart addition

*For any* product with available inventory greater than or equal to requested quantity, the AddItem operation SHALL succeed and the cart SHALL contain the requested item.

**Validates: Requirements 1.2**

**Test Strategy:** Property-based test with random product IDs, random available inventory (1-1000), and random requested quantities (1-available). Verify cart contains item after successful addition.

### Property 4: Feature flag controls validation behavior

*For any* AddItem request, when the feature flag "cartInventoryValidation" is disabled, the Cart Service SHALL NOT call the Product Catalog Service.

**Validates: Requirements 3.1, 3.2**

**Test Strategy:** Property-based test with random AddItem requests and feature flag states. Verify Product Catalog Service call count is zero when flag is disabled.

### Property 5: Validation failure does not prevent cart addition (fail open)

*For any* AddItem request where Product Catalog Service returns an error or times out, the Cart Service SHALL complete the cart addition successfully.

**Validates: Requirements 4.1, 4.2, 4.3**

**Test Strategy:** Property-based test with random AddItem requests and simulated Product Catalog failures (timeout, error, network failure). Verify cart addition succeeds in all failure scenarios.

### Property 6: Telemetry is emitted for all validation attempts

*For any* inventory validation attempt, the Cart Service SHALL emit a distributed trace span with validation duration and result.

**Validates: Requirements 2.1, 2.2**

**Test Strategy:** Property-based test with random AddItem requests. Verify trace span exists with required attributes (productId, requestedQuantity, availableQuantity, validationResult, duration).

### Property 7: Error messages include product context

*For any* AddItem request that fails due to insufficient inventory, the error message SHALL include the product ID.

**Validates: Requirements 6.1**

**Test Strategy:** Property-based test with random product IDs and insufficient inventory scenarios. Verify error message contains the product ID from the request.

### Property 8: gRPC connections are reused

*For any* sequence of AddItem requests, the Cart Service SHALL reuse the same gRPC channel to the Product Catalog Service rather than creating new connections.

**Validates: Requirements 5.4**

**Test Strategy:** Monitor gRPC channel creation count during load test. Verify channel count remains constant (1) regardless of request volume.

### Property 9: Circuit breaker prevents cascading failures

*For any* sequence of Product Catalog Service failures exceeding the threshold, the Cart Service SHALL open the circuit breaker and bypass validation for subsequent requests.

**Validates: Requirements 4.4**

**Test Strategy:** Generate sequence of requests with simulated Product Catalog failures. Verify circuit opens after threshold and subsequent requests bypass validation without calling Product Catalog.

### Property 10: Validation timeout is enforced

*For any* Product Catalog Service call that exceeds the configured timeout, the Cart Service SHALL cancel the request and proceed with cart addition.

**Validates: Requirements 4.2**

**Test Strategy:** Property-based test with random timeout values and simulated slow Product Catalog responses. Verify requests are cancelled at timeout boundary and cart addition proceeds.

## Error Handling

### Error Categories

1. **Insufficient Inventory (User Error)**
   - HTTP Status: 400 Bad Request
   - gRPC Status: FailedPrecondition
   - Message: "Insufficient inventory for product {productId}. Requested: {requested}, Available: {available}"
   - Action: Return error to user, do not add to cart

2. **Product Catalog Unavailable (System Error)**
   - HTTP Status: N/A (internal)
   - gRPC Status: N/A (internal)
   - Message: Logged internally
   - Action: Fail open, add to cart, emit degraded mode metric

3. **Product Catalog Timeout (System Error)**
   - HTTP Status: N/A (internal)
   - gRPC Status: DeadlineExceeded
   - Message: Logged internally
   - Action: Fail open, add to cart, emit timeout metric

4. **Invalid Product Response (System Error)**
   - HTTP Status: N/A (internal)
   - gRPC Status: N/A (internal)
   - Message: Logged with product ID and response details
   - Action: Fail open, add to cart, emit parsing error metric

5. **Circuit Breaker Open (System State)**
   - HTTP Status: N/A (internal)
   - Message: Logged with circuit state
   - Action: Bypass validation, add to cart, emit circuit open metric

### Error Handling Flow

```
AddItem Request
    ↓
Feature Flag Check
    ↓
[If Enabled] → Validate Inventory
    ↓
    ├─ Success → Add to Cart → Return Success
    ├─ Insufficient → Return Error (FailedPrecondition)
    └─ System Error → Log Error → Add to Cart → Return Success
```

### Resilience Patterns

1. **Circuit Breaker**
   - Failure Threshold: 5 failures in 30 seconds
   - Break Duration: 60 seconds
   - Half-Open: Allow 1 request to test recovery

2. **Timeout**
   - Product Catalog Call: 500ms
   - Cancellation Token: Propagated to all async operations

3. **Retry**
   - Max Retries: 1
   - Retry On: Transient network errors only
   - No Retry On: Timeout, insufficient inventory

4. **Fail Open**
   - Default Behavior: Allow cart addition on validation failure
   - Configurable: Can be changed to fail closed via configuration

## Testing Strategy

### Unit Tests

1. **InventoryValidator Tests**
   - Test successful validation with sufficient inventory
   - Test rejection with insufficient inventory
   - Test timeout handling
   - Test error response handling
   - Test circuit breaker behavior

2. **CartService Tests**
   - Test AddItem with validation enabled
   - Test AddItem with validation disabled
   - Test AddItem with validation failure (fail open)
   - Test feature flag evaluation
   - Test telemetry emission

### Property-Based Tests

**Framework:** FsCheck for C# (or similar)

**Test Configuration:**
- Minimum 100 iterations per property
- Random seed for reproducibility
- Shrinking enabled for failure cases

**Property Test 1: Validation Before Modification**
```csharp
// Feature: cart-inventory-validation, Property 1: Validation occurs before cart modification
[Property(Arbitrary = new[] { typeof(CartGenerators) })]
public async Task Property_ValidationOccursBeforeCartModification(
    string userId, 
    string productId, 
    int quantity)
{
    // Arrange: Enable feature flag, setup tracing
    // Act: Call AddItem
    // Assert: Verify ValidateInventory span completes before Valkey write span
}
```

**Property Test 2: Insufficient Inventory Rejection**
```csharp
// Feature: cart-inventory-validation, Property 2: Insufficient inventory prevents cart addition
[Property(Arbitrary = new[] { typeof(CartGenerators) })]
public async Task Property_InsufficientInventoryPreventsAddition(
    string userId,
    string productId,
    PositiveInt availableInventory,
    PositiveInt requestedQuantity)
{
    // Arrange: Setup product with availableInventory
    // Act: Request requestedQuantity where requestedQuantity > availableInventory
    // Assert: Verify cart unchanged and error returned
}
```

**Property Test 3: Sufficient Inventory Success**
```csharp
// Feature: cart-inventory-validation, Property 3: Sufficient inventory allows cart addition
[Property(Arbitrary = new[] { typeof(CartGenerators) })]
public async Task Property_SufficientInventoryAllowsAddition(
    string userId,
    string productId,
    PositiveInt availableInventory)
{
    // Arrange: Setup product with availableInventory
    // Act: Request quantity <= availableInventory
    // Assert: Verify cart contains item
}
```

**Property Test 4: Feature Flag Control**
```csharp
// Feature: cart-inventory-validation, Property 4: Feature flag controls validation behavior
[Property(Arbitrary = new[] { typeof(CartGenerators) })]
public async Task Property_FeatureFlagControlsValidation(
    string userId,
    string productId,
    int quantity,
    bool featureFlagEnabled)
{
    // Arrange: Set feature flag state
    // Act: Call AddItem
    // Assert: Verify Product Catalog called only if flag enabled
}
```

**Property Test 5: Fail Open Behavior**
```csharp
// Feature: cart-inventory-validation, Property 5: Validation failure does not prevent cart addition
[Property(Arbitrary = new[] { typeof(CartGenerators) })]
public async Task Property_ValidationFailureDoesNotPreventAddition(
    string userId,
    string productId,
    int quantity,
    ProductCatalogError errorType)
{
    // Arrange: Setup Product Catalog to return error
    // Act: Call AddItem
    // Assert: Verify cart addition succeeds despite validation failure
}
```

### Integration Tests

1. **End-to-End Flow**
   - Test complete flow from frontend through cart to product catalog
   - Verify distributed tracing spans
   - Verify metrics emission

2. **Failure Scenarios**
   - Test with Product Catalog service down
   - Test with Product Catalog service slow
   - Test with network partition
   - Test with malformed responses

3. **Load Testing**
   - Measure latency impact under normal load
   - Measure latency impact under high load
   - Verify circuit breaker behavior under sustained failures
   - Measure resource usage (CPU, memory, network)

### Performance Testing

**Baseline Metrics (Current):**
- AddItem P50: ~5ms
- AddItem P95: ~25ms
- AddItem P99: ~50ms
- CPU Usage: 18-21%
- Memory Usage: 168MB

**Target Metrics (With Validation):**
- AddItem P50: <15ms (+10ms)
- AddItem P95: <50ms (+25ms)
- AddItem P99: <100ms (+50ms)
- CPU Usage: <25% (+4%)
- Memory Usage: <200MB (+32MB)

**Load Test Scenarios:**
1. Steady state: 100 req/s for 10 minutes
2. Spike: 0 → 500 req/s → 0 over 5 minutes
3. Sustained high: 300 req/s for 30 minutes
4. Product Catalog failures: 50% error rate for 5 minutes

## Deployment Strategy

### Phase 1: Development (Week 1)
- Implement InventoryValidator component
- Add unit tests
- Add property-based tests
- Local testing with docker-compose

### Phase 2: Staging (Week 2)
- Deploy to staging environment
- Run integration tests
- Run load tests
- Verify observability (traces, metrics, logs)

### Phase 3: Production Rollout (Week 3-4)
- Deploy with feature flag disabled
- Enable for 1% of traffic
- Monitor for 24 hours
- Increase to 10% if metrics acceptable
- Increase to 50% if metrics acceptable
- Increase to 100% if metrics acceptable

### Rollback Plan
- Disable feature flag immediately if issues detected
- Redeploy previous version if feature flag doesn't resolve
- Maximum rollback time: 5 minutes

### Success Criteria
- No increase in cart service error rate
- P95 latency increase <30ms
- CPU usage increase <5%
- Memory usage increase <50MB
- No circuit breaker activations under normal load

## Observability

### Distributed Tracing

**Span: ValidateInventory**
- Parent: AddItem span
- Attributes:
  - `app.product.id`: Product ID being validated
  - `app.requested.quantity`: Quantity requested
  - `app.available.inventory`: Inventory available (if successful)
  - `app.validation.result`: "success", "insufficient", "error", "bypassed"
  - `app.validation.duration.ms`: Validation duration
  - `app.circuit.breaker.state`: "closed", "open", "half-open"

### Metrics

**Histogram: `app.cart.inventory_validation.duration`**
- Unit: seconds
- Buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
- Labels: result (success/insufficient/error/bypassed)

**Counter: `app.cart.inventory_validation.total`**
- Labels: result (success/insufficient/error/bypassed)

**Counter: `app.cart.inventory_validation.circuit_breaker.state_changes`**
- Labels: from_state, to_state

**Gauge: `app.cart.inventory_validation.circuit_breaker.state`**
- Values: 0 (closed), 1 (open), 2 (half-open)

### Logging

**Log: Validation Success**
```
Level: Debug
Message: "Inventory validation succeeded for product {ProductId}"
Fields: userId, productId, requestedQuantity, availableInventory, duration
```

**Log: Validation Insufficient**
```
Level: Info
Message: "Insufficient inventory for product {ProductId}"
Fields: userId, productId, requestedQuantity, availableInventory
```

**Log: Validation Error**
```
Level: Warning
Message: "Inventory validation failed for product {ProductId}, failing open"
Fields: userId, productId, errorType, errorMessage, duration
```

**Log: Circuit Breaker State Change**
```
Level: Warning
Message: "Circuit breaker state changed from {FromState} to {ToState}"
Fields: fromState, toState, failureCount, timestamp
```

## Security Considerations

1. **Input Validation**
   - Validate productId format before calling Product Catalog
   - Validate quantity is positive integer
   - Sanitize error messages to prevent information disclosure

2. **Rate Limiting**
   - Existing cart service rate limits apply
   - No additional rate limiting needed for Product Catalog calls

3. **Authentication**
   - Use existing service-to-service authentication
   - No user credentials passed to Product Catalog

4. **Data Privacy**
   - No PII in validation requests
   - Product IDs and quantities are not sensitive

## Open Questions

1. **Inventory Data Location**
   - Where is inventory data stored in Product Catalog?
   - Is it in the product description or a separate field?
   - Do we need to modify the Product proto definition?

2. **Inventory Consistency**
   - How do we handle race conditions between validation and checkout?
   - Should we implement inventory reservation?
   - What is the acceptable window for inventory inconsistency?

3. **Performance Impact**
   - What is the current Product Catalog service capacity?
   - Can it handle the additional load from cart validations?
   - Do we need to scale Product Catalog before enabling this feature?

4. **Monitoring Thresholds**
   - What latency increase is acceptable?
   - What error rate triggers rollback?
   - What CPU/memory increase is acceptable?
