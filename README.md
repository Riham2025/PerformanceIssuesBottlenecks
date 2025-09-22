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
```
public class OrderItemDTO
{
public int ProductId { get; set; }
public int Quantity { get; set; }
}

```

## AutoMapper Profile (for order-lines)

Use context Items to avoid manual FK assignment:
```
public class OrderMappingProfile : Profile
{
public OrderMappingProfile()
{
CreateMap<OrderItemDTO, OrderProducts>()
.ForMember(d => d.PID, opt => opt.MapFrom(s => s.ProductId))
.ForMember(d => d.OID, opt => opt.Ignore())
.AfterMap((src, dest, ctx) =>
{
var order = (Order)ctx.Items["Order"];
dest.OID = order.OID; // set FK via mapping context
});
}
}
```

## The Fixed Implementation (EF Core) : 

Key ideas: Load all products in one query, validate & compute totals in-memory, wrap everything in a transaction, save once.
```
using System.Data;
if (items is null || items.Count == 0)
throw new ArgumentException("Order must contain at least one item.");


// 1) Normalize: merge duplicate lines by ProductId
var mergedItems = items
.GroupBy(i => i.ProductId)
.Select(g => new OrderItemDTO { ProductId = g.Key, Quantity = g.Sum(x => x.Quantity) })
.ToList();


// 2) Load all products in ONE query
var productIds = mergedItems.Select(i => i.ProductId).ToList();
var products = await db.Products
.Where(p => productIds.Contains(p.PID))
.ToDictionaryAsync(p => p.PID, ct);


// 3) Validate existence & stock, compute total
decimal total = 0m;
foreach (var item in mergedItems)
{
if (!products.TryGetValue(item.ProductId, out var p))
throw new InvalidOperationException($"Product {item.ProductId} not found.");


if (item.Quantity <= 0)
throw new InvalidOperationException($"Quantity must be positive for product {p.Name}.");


if (p.Stock < item.Quantity)
throw new InvalidOperationException($"Insufficient stock for '{p.Name}'. Requested {item.Quantity}, available {p.Stock}.");


total += item.Quantity * p.Price;
}


// 4) Transaction to make the operation atomic and concurrency-safe
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Serializable, ct);


// 5) Create order first to get OID
var order = new Order
{
UID = uid,
OrderDate = DateTime.UtcNow,
TotalAmount = 0m
};
db.Orders.Add(order);
await db.SaveChangesAsync(ct); // generates OID


// 6) Map order-items via AutoMapper (passing the order through context)
var orderLines = mapper.Map<List<OrderProducts>>(mergedItems, opt => opt.Items["Order"] = order);
db.OrderProducts.AddRange(orderLines);


// 7) Deduct stock in-memory; EF tracks changes
foreach (var item in mergedItems)
{
var p = products[item.ProductId];
p.Stock -= item.Quantity; // safe due to earlier validation & transaction isolation
}


// 8) Set total & save once
order.TotalAmount = total;
await db.SaveChangesAsync(ct);


// 9) Commit
await tx.CommitAsync(ct);


return order.OID;
}
```

## Notes on Concurrency

Serializable isolation prevents concurrent transactions from interleaving in a way that causes oversell.

Alternatively (or additionally), add RowVersion to Product and turn on optimistic concurrency:

In the Product entity: public byte[] RowVersion { get; set; }

In OnModelCreating: modelBuilder.Entity<Product>().Property(p => p.RowVersion).IsRowVersion();

Catch DbUpdateConcurrencyException and return 409 Conflict to clients.