# PerformanceIssuesBottlenecks

## Fixing Loop-Related Database Issues in PlaceOrder

Audience: C# / EF Core developers (ASP.NET Core).
Goal: Eliminate logic + database anti-patterns (N+1 calls, race conditions, partial writes), and ship a safe, fast, and correct order-placement routine.

### TL;DR 

Your original implementation:

Performs repeated DB lookups inside loops (GetProductByName) → latency & load.

Does validation in one pass and then re-queries in a second pass → stale reads under concurrency.


