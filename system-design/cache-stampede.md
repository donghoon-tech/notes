# Cache Stampede

When a cache entry expires, multiple concurrent requests miss simultaneously
and stampede the origin (DB). At scale, simultaneous expiry of a popular key
can take the DB down for the seconds it takes to recompute.

---

## Defense Patterns

| Pattern | Mechanism | UX | Cost |
|---------|-----------|-----|------|
| **Mutex / Lock** | Only one request hits the DB on miss; others wait | Always fresh | Waiting requests blocked |
| **Probabilistic Early Expiration (PER)** | Refresh probabilistically before TTL expires; probability rises as expiry approaches | Always fresh | Triggered request blocks on DB |
| **stale-while-revalidate (SWR)** | Return stale entry immediately, refresh in background | Briefly stale after refresh | None (all requests served instantly) |

These are not mutually exclusive. PER + background refresh is effectively
SWR. In practice, **SWR + lockKey covers most cases.**

---

## What to Serve to the 99 Waiting Requests

When a miss happens with concurrent requests in flight:

- **Stale cache (SWR):** slightly outdated, fast response. Feeds,
  recommendations — anywhere consistency is loose.
- **Wait for lock:** correct data, latency cost. Orders, payments —
  anywhere consistency is tight. Timeout handling is mandatory.

---

## When SWR Applies — Is the Data the Basis for a Decision?

| Data | SWR? | Reason |
|------|------|--------|
| Main banner / promotion | ✅ | 1–2s stale invisible to UX |
| Liked products list | ✅ | Slight staleness fine |
| Product detail | ✅ | Few seconds stale on price/name acceptable |
| Ticket inventory | ❌ | Data drives the decision — stale = trusting a phantom balance |
| Stock count (at order time) | ❌ | Selling sold-out stock is an incident |
| Loan limit | ❌ | Consistency is a legal artifact; audited |

> The dividing line: does the user act on this number? Banners and feeds
> aren't the basis for "balance is X, so I'll do Y." Inventory and balances
> are. That distinction decides whether SWR is safe.

---

## SWR Implementation — Dual TTL

SWR's soft TTL is **not a Redis feature.** Redis holds a key with a hard
TTL (e.g. 6 minutes); the soft TTL (e.g. 5 minutes) lives as a timestamp
inside the cached value. The application code reads the value, checks
"more than 5 minutes old?", and decides whether to trigger a background
refresh.

```
CacheEntry {
    value: ...,
    softExpireAt: <epoch ms>
}
```

Hard TTL = how long Redis keeps the key. Soft TTL = when refresh kicks in.

---

## Choosing TTLs

TTL is not a magic number. It's a function of three things:

| Input | Question | Effect on TTL |
|-------|----------|---------------|
| Data change frequency | How often does the underlying data actually change? | Lower bound — TTL ≤ acceptable lag |
| Stale tolerance | What's the worst stale value users can see? | Sets the ceiling on soft TTL |
| Origin (DB) load budget | What miss rate can the DB sustain? | Pushes TTL up — longer TTL = fewer misses |

For SWR specifically: **soft TTL** controls staleness window;
**hard TTL** is just a long safety net (typically soft + small buffer).
The buffer should fit one refresh round-trip.

If TTL choice cannot satisfy all three, the cache strategy is wrong for
the workload — pre-warming or a different pattern is needed, not a
different number.

---

## When the Lock Holder Dies

The request that won the lock crashed mid-refresh. What now?

- **TTL on the lock itself.** Lock keys must have their own short TTL
  (a few seconds, slightly longer than expected refresh time). Without
  it, a dead holder blocks all refresh forever.
- **Other waiters during the gap:** between holder death and lock TTL
  expiry, no one is refreshing. With SWR, this is fine — stale entries
  keep serving. Without SWR, requests pile up waiting.
- **Lock TTL too short** is its own problem: a slow-but-alive holder gets
  its lock yanked, two refreshes run, both write — last write wins, but
  duplicate DB load defeats the purpose.

The right TTL depends on the refresh's p99 latency. Measure, don't guess.

---

## Distributed Lock Caveats

Single Redis node: `SET key value NX EX 5` is sufficient. Redis Cluster
and Sentinel introduce edge cases:

- **Sentinel failover:** lock written to old primary, replication hasn't
  caught up, new primary doesn't see it. Two requests hold "the lock."
- **Redlock:** Antirez's algorithm using N independent Redis nodes
  (majority quorum). The Kleppmann critique argues it's not safe under
  arbitrary clock skew or pauses.

For cache-stampede protection, **near-perfect** lock semantics are
acceptable — the worst case is two refreshes running, not data
corruption. Don't over-engineer with Redlock just for this. Reserve it
(or proper consensus systems) for cases where lock violation has real
consequences (e.g., financial deduction).

---

## Hard-Expiry Cases — Don't Over-Engineer

True hard-TTL expiry (the entire window passes with zero traffic, then a
spike hits) is rare in practice. Soft 5min / hard 6min means six full
minutes of zero traffic followed by a stampede — unusual outside cold
deploys.

- **Cold deploy / restart:** handle with **cache warming** at boot.
- **Heavy aggregation queries (multi-second):** schedule periodic
  warming. Never let hard TTL expire on these.
- **Everything else:** sync load + lockKey is fine. Don't build elaborate
  defenses for events that don't happen.

---

## Negative Caching

Cache misses for non-existent data are a stampede source too. If a key
never exists, every lookup goes to the DB, and bots/attacks targeting
random IDs amplify this.

Cache the absence — short TTL (seconds to a minute), explicit "not found"
marker, distinct from "no cache entry." Short TTL because eventually the
data may exist (delayed write, race with creation).

This is closely related to cache penetration — covered in a separate
note.

---

## Related: Thundering Herd

Stampede is miss-driven (cache fails to absorb concurrent requests).
**Thundering herd** is wakeup-driven (many waiting clients released
simultaneously — server recovery, lock release, retry timer expiry).
Same symptom shape, different cause and remedy. See the [thundering herd note](./thundering-herd.md).

---

## Monitoring

Cache problems are invisible until the DB falls over. Useful signals:

- **Miss rate spike** — sudden jump from baseline = something de-cached
  in bulk (eviction, mass expiry, code bug).
- **Origin (DB) RPS spike correlated with cache miss spike** — the
  signature of stampede.
- **Lock acquisition contention** — high wait count or lock TTL
  near-misses indicates the lock is the bottleneck.
- **Refresh duration** — if p99 approaches lock TTL, lock yank events
  start happening; raise the lock TTL or fix the slow query.
- **Stale-serve duration (SWR)** — how long entries serve stale before
  refresh. Long durations mean refresh is failing silently.

The point isn't to prevent every stampede — it's to detect when defenses
are degrading before they fail.

---

## Common Misconceptions

**"PER is a strict upgrade over SWR."**
PER avoids stale data but the request that triggers the refresh blocks
on the DB. SWR has zero blocking but exposes briefly stale values.
Trade-off, not upgrade.

**"Redis TTL expiry can drive SWR."**
Redis TTL deletes the key. SWR's soft TTL is a timestamp inside the
value, evaluated by application code. Distinct mechanisms.

**"SWR" alone is unambiguous in interviews.**
"SWR" also names Vercel's React data-fetching library. Use
"stale-while-revalidate strategy" to avoid confusion.
