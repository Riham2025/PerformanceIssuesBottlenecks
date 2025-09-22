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

