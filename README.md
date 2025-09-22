# PerformanceIssuesBottlenecks

## Fixing Loop-Related Database Issues in PlaceOrder

Audience: C# / EF Core developers (ASP.NET Core).
Goal: Eliminate logic + database anti-patterns (N+1 calls, race conditions, partial writes), and ship a safe, fast, and correct order-placement routine.

# TL;DR 

## Your original implementation:

Performs repeated DB lookups inside loops (GetProductByName) → latency & load.

Does validation in one pass and then re-queries in a second pass → stale reads under concurrency.

No transaction: failures mid-way leave partial writes (e.g., order created, but some rows/stock not updated).

Uses product name as the key (fragile; not guaranteed unique).

## Fixed approach:

Single DB round-trip to load all needed products (by IDs; fall back to names only if they are uniquely constrained).

Merge duplicate lines per product and validate once.

Wrap all changes in a database transaction (preferably Serializable) to prevent overselling.

Update stock, create order + order-items, save once, then commit.

(Optional but recommended) Use optimistic concurrency via a RowVersion column.

## Symptoms You May Be Seeing : 

Slow requests and high DB CPU due to many small queries.

Race conditions: two users buy the last item concurrently → overselling.

Partial updates if an exception happens after the order is created.

Inconsistent totals if prices change between the two loops (validate vs. process).

## Root Causes in the Original Code

```
// (abridged) Key problems in the old version
for (int i = 0; i < items.Count; i++) {
existingProduct = _productService.GetProductByName(items[i].ProductName); // DB call in a loop
// ...
}


foreach (var item in items) {
existingProduct = _productService.GetProductByName(item.ProductName); // DB call again in a loop
// deduct stock, add OrderProducts, update product
}
```

N+1 pattern: You call the DB for each item twice (validation & processing).

No transaction: if an exception occurs mid-loop, some updates are persisted, others are not.

Stale reads: products are fetched in validation phase, but stock/prices could change before the processing loop.

Using Name instead of ID: collisions and case/culture issues; hard to guarantee uniqueness without a strict constraint.

Update per item with immediate UpdateProduct(...) → multiple SaveChanges paths if services call the DB eagerly (depends on service implementation).