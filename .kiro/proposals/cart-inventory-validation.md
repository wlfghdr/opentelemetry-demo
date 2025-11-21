# Proposal: Add Real-Time Inventory Validation to Cart Service

## Overview

Currently, the cart service allows users to add any quantity of items without checking product availability. This proposal adds real-time inventory validation when items are added to the cart.

## Proposed Changes

### File: `src/cart/src/services/CartService.cs`

**Enhancement:** Add inventory validation in `AddItem` method

```csharp
public override async Task<Empty> AddItem(AddItemRequest request, ServerCallContext context)
{
    var activity = Activity.Current;
    activity?.SetTag("app.user.id", request.UserId);
    activity?.SetTag("app.product.id", request.Item.ProductId);
    activity?.SetTag("app.product.quantity", request.Item.Quantity);

    try
    {
        // NEW: Validate inventory before adding to cart
        var inventoryAvailable = await ValidateInventoryAsync(
            request.Item.ProductId, 
            request.Item.Quantity
        );
        
        if (!inventoryAvailable)
        {
            throw new RpcException(new Status(
                StatusCode.FailedPrecondition, 
                $"Insufficient inventory for product {request.Item.ProductId}"
            ));
        }

        await _cartStore.AddItemAsync(request.UserId, request.Item.ProductId, request.Item.Quantity);

        return Empty;
    }
    catch (RpcException ex)
    {
        activity?.AddException(ex);
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
}

// NEW METHOD: Call product catalog service to check inventory
private async Task<bool> ValidateInventoryAsync(string productId, int requestedQuantity)
{
    using var activity = Activity.Current?.Source.StartActivity("ValidateInventory");
    activity?.SetTag("app.product.id", productId);
    activity?.SetTag("app.requested.quantity", requestedQuantity);
    
    try
    {
        // Call product catalog service via gRPC
        var channel = GrpcChannel.ForAddress(_productCatalogAddress);
        var client = new ProductCatalogService.ProductCatalogServiceClient(channel);
        
        var product = await client.GetProductAsync(new GetProductRequest 
        { 
            Id = productId 
        });
        
        // Parse inventory from product metadata
        var availableInventory = ParseInventoryFromProduct(product);
        
        activity?.SetTag("app.available.inventory", availableInventory);
        
        return availableInventory >= requestedQuantity;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        _logger.LogError(ex, "Failed to validate inventory for product {ProductId}", productId);
        
        // Fail open: allow cart addition if inventory check fails
        return true;
    }
}

private int ParseInventoryFromProduct(Product product)
{
    // Parse inventory from product description or metadata
    // This would need to be implemented based on product catalog schema
    return 100; // Placeholder
}
```

## Benefits

1. **Better User Experience:** Users know immediately if items are available
2. **Reduced Checkout Failures:** Prevents adding out-of-stock items
3. **Inventory Accuracy:** Real-time validation against product catalog
4. **Business Logic:** Enforces inventory constraints at cart level

## Technical Impact

### New Dependencies
- gRPC client for Product Catalog service
- Additional configuration for product catalog endpoint

### Performance Considerations
- **Additional Network Call:** Each `AddItem` operation now calls Product Catalog
- **Latency Impact:** +10-50ms per cart addition (network + processing)
- **Increased Load:** Product Catalog will receive more requests
- **Error Handling:** Need to handle product catalog unavailability

### Resource Impact
- **Cart Service CPU:** Additional processing for gRPC calls and parsing
- **Cart Service Memory:** gRPC channel management and response buffering
- **Network:** Additional inter-service communication
- **Product Catalog:** Increased request volume

## Implementation Plan

1. Add gRPC client dependency to `cart.csproj`
2. Add configuration for product catalog endpoint
3. Implement `ValidateInventoryAsync` method
4. Add unit tests for validation logic
5. Add integration tests for failure scenarios
6. Update observability (metrics, traces, logs)
7. Deploy and monitor

## Rollout Strategy

- Feature flag: `cartInventoryValidation` (default: false)
- Gradual rollout: 10% → 50% → 100%
- Monitor error rates and latency
- Rollback plan if issues detected

---

## Question for Review

**Is this change safe to implement given current system load?**

Should we proceed with this enhancement, or do we need to consider resource constraints first?
