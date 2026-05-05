# Inventory Concurrency Control

## Problem

High-traffic commerce requires inventory to be checked and decremented atomically.
Two separate stores (Redis + RDB) cannot be updated atomically — design must accept this and work around it.

---

## Three Approaches

| Approach | Throughput | Scale | Consistency |
|---|---|---|---|
| Distributed Lock (Redisson) | Low (serialized) | ~tens of thousands | Strong |
| DB Atomic UPDATE | Medium | ~hundreds of thousands | Strong |
| Redis DECR | High | ~millions | Weak (requires correction) |

---

## DB Atomic UPDATE

```sql
UPDATE inventory SET stock = stock - 1
WHERE product_id = ? AND stock > 0
```

- Use affected rows to determine success/failure — no explicit lock needed beyond row-level.
- **Do not bundle with the order transaction.** The inventory row lock is held for the entire transaction duration.
- Run inventory decrement in a separate transaction, commit first, then proceed with order.
- On order failure → compensate with `stock + 1`. If compensation fails → push to `compensation_queue` + scheduler retry.

---

## Redis DECR

```java
Long remaining = redisTemplate.opsForValue().decrement("stock:" + productId);
if (remaining < 0) {
    redisTemplate.opsForValue().increment("stock:" + productId);
    throw new SoldOutException();
}
```

- `DECR` is atomic within Redis — no lock needed.
- Two options for safe decrement:

| Option | Mechanism | Note |
|---|---|---|
| Decrement-then-check | `DECR` → if < 0 → `INCR` revert | Simple; brief negative state between commands |
| Atomic check-and-decrement | Lua script | No negative exposure; single key = no cluster issue |

### Why Redis is faster than RDB

- **In-memory**: no disk I/O
- **Single-threaded event loop**: multi-threading exists to handle blocking I/O — Redis has none, so single-thread is optimal. No context switching, no lock contention between threads.
- **Connection scaling**: RDB assigns a thread per connection (MySQL) — 100 connections = 100 threads = memory + context switching overhead. Redis is single-threaded, so 100 connections are still handled by 1 thread — resource cost per connection is far lower.

---

## Redis + RDB Consistency

### The fundamental constraint

Atomically updating two separate stores is impossible. Design around it — don't fight it.

- DECR succeeds → server dies → no record in RDB → scheduler can't detect
- RDB written first → server dies before DECR → stock appears to increase
- Real-time orders make it impossible to compare Redis vs RDB snapshots — there's no "frozen moment"

### Possible states

| State | Direction | Cause | Acceptable? |
|---|---|---|---|
| Redis < RDB | Redis runs out first | Failed `INCR` revert after order failure | ✅ Yes — false sold-out, correctable |
| RDB < Redis | RDB runs out first | Back-door writes (admin, batch, direct SQL) | ⚠️ Rare, handle in order flow |

### RDB < Redis handling

- Detected when RDB decrement hits 0
- Immediately `SET stock:{productId} 0` in Redis within the order flow — no extra mechanism needed
- Affected users fail slightly later in the flow instead of at the Redis gate — acceptable

### Redis < RDB (false sold-out) handling

- Redis hits 0 → users see sold-out → order traffic naturally stops → **this is the "frozen moment"**
- Operator checks RDB actual stock → manually corrects Redis if stock remains, then reopen
- Or run a batch sync at this point

**Why not automate fully:** Edge cases (DECR then server crash) occur maybe a few times a month even at millions of orders/day. Complex automation for this is over-engineering.

---

## Inventory Table: Separate from Product

```sql
-- Preferred
product   (id, name, description, price, ...)
inventory (id, product_id, stock, updated_at)

-- Avoid
product   (id, name, description, price, stock, ...)
```

- Inventory is decremented on every order — holds a row lock until transaction commits
- Product info updates are unrelated but would contend on the same row
- Separation eliminates cross-concern lock contention

**Lock timing (InnoDB):** Lock is acquired when the query executes, not when the transaction starts.
Place inventory decrement as late as possible in the transaction to minimize lock hold time.

---

## When to Use What

| Scenario | Approach |
|---|---|
| Millions of requests, weak consistency OK (ticketing, limited flash sale) | Redis DECR + manual correction |
| Hundreds of thousands, strong consistency (general commerce) | DB atomic UPDATE + compensation |
| Tens of thousands, complex business logic (loan limit deduction) | Distributed lock + transaction |

## Finance Domain Note

In financial systems, eventual consistency is often unacceptable:
- Every discrepancy must be explainable at every point in time (audit trail)
- "Sync later" leaves an unexplainable gap
- Prefer pessimistic locking to serialize access — lower throughput is an acceptable tradeoff for auditability and correctness