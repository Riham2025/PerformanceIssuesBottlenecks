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

1- N+1 pattern: You call the DB for each item twice (validation & processing).

2- No transaction: if an exception occurs mid-loop, some updates are persisted, others are not.

3- Stale reads: products are fetched in validation phase, but stock/prices could change before the processing loop.

4- Using Name instead of ID: collisions and case/culture issues; hard to guarantee uniqueness without a strict constraint.

5- Update per item with immediate UpdateProduct(...) → multiple SaveChanges paths if services call the DB eagerly (depends on service implementation).

## What “Correct” Should Guarantee

1- Atomicity: order + all its lines + stock deductions succeed or nothing persists.

2- No oversell: concurrent orders never drive stock negative.

3- Performance: bounded number of DB calls (ideally 2–3 queries total per order).

4- Deterministic totals: price/qty snapshot is consistent within the transaction.

## Recommended Data & Mapping Tweaks

1- Prefer IDs in DTOs: add a ProductId to OrderItemDTO. Names can be user-facing, but persistence should use IDs.

2- Optional: add RowVersion to Product for optimistic concurrency (EF Core IsRowVersion).

3- Unique constraint on Product.Name only if you must accept names.

## Sample Entities (extract) 

```
public class Product
{
public int PID { get; set; }
public string Name { get; set; } = default!;
public decimal Price { get; set; }
public int Stock { get; set; }


// Optional for optimistic concurrency:
public byte[]? RowVersion { get; set; }
}


public class Order
{
public int OID { get; set; }
public int UID { get; set; }
public DateTime OrderDate { get; set; }
public decimal TotalAmount { get; set; }
public List<OrderProducts> Items { get; set; } = new();
}


public class OrderProducts
{
public int OID { get; set; } // FK → Order
public int PID { get; set; } // FK → Product
public int Quantity { get; set; }
}

```

## Order Item DTO

Prefer this structure (ID-based):