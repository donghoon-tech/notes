# API Idempotency

Same operation, any number of times, same result. In distributed systems, network timeouts and retries are not edge cases — they are the default. Idempotency is not optional for state-mutating APIs.

This page covers **synchronous client-server API calls** — the Stripe-style `Idempotency-Key` header pattern. Asynchronous consumer idempotency in event-driven systems is a related but distinct problem; see [Event Idempotency](./event-idempotency.md).

---

## Why Not Just a Status Lookup?

A reasonable first reaction: if the client doesn't know whether the request succeeded, why not just expose a status endpoint? `POST /payments` times out, client follows up with `GET /payments?order_id=...`, and the answer is there. With a unique constraint on the business key, duplicate inserts fail at the DB layer anyway.

This works — in some cases. The pattern is real and used. But it falls short in four places where idempotency keys are necessary:

- **Server-generated identifiers.** When the server assigns the resource ID (`payment_id`, `transaction_id`), the client has nothing to look up by after a timeout. The combination `(amount, card_token)` doesn't identify *this* request — the same card paying the same amount twice is a legitimate scenario.
- **No natural business key.** Send-this-notification, run-this-inference, refund-this-transaction-partially — operations where two identical-looking requests can both be valid. There is no business key to deduplicate on.
- **Race between lookup and retry.** The status lookup races the original request: the lookup returns "not found" because the original is still committing, the client retries, and now two operations execute. A unique constraint catches the second insert at commit time, but the violation surfaces as a low-level error the server still has to translate into a meaningful response — essentially reinventing the idempotency-key flow ad hoc.
- **Where the guarantee lives.** Status-lookup pushes correctness logic onto every client (lookup-then-decide-then-retry). Idempotency keys keep that logic on the server: one implementation, every caller benefits. For public APIs with diverse clients, the server-side guarantee is the only realistic option.

The two patterns aren't competitors. **Idempotency keys and business-key uniqueness are usually layered together** — the unique constraint is the durable correctness floor, the idempotency key is the user-facing retry contract. The rest of this page is about the second layer.

---

## HTTP Methods

Spec-level idempotency and real-world behavior are not the same thing.

| Method | Spec | Notes |
| --- | --- | --- |
| GET | Idempotent | No side effects. A GET that increments a counter has broken the contract. |
| PUT | Idempotent | Full replacement. Same payload → same result. |
| DELETE | Idempotent | Already-deleted resource should return 200/204, not 404. |
| POST | Not idempotent | Assign idempotency via `Idempotency-Key` header. |
| PATCH | Depends | `{ stock: 10 }` is idempotent. `{ stock: { increment: 1 } }` is not. |

Idempotency is a property of the implementation, not the method name. The spec expresses intent, not a guarantee.

---

## Idempotency Key Pattern

Client sends a unique key (UUID) per logical request. Server uses it to decide whether to process or replay.

```
First time  → process, store result under key
Seen before → return stored result, skip processing
In-flight   → return 409 or "processing" status
```

The pattern is most associated with Stripe's payment API and is now common in any system that takes payments, creates orders, or otherwise mutates expensive state in response to client requests.

**The trap:** receiving the key is not enough. If the server stores the result *after* the business logic, a crash in between leaves the key without a stored result and the next retry reprocesses. Result storage and business logic must be in the same transaction.

---

## Storage: Redis First, RDB Second

Single-store is fragile in opposite directions. Redis alone breaks idempotency on failure (no durability). RDB alone is too slow to block double-submits at millisecond latency.

| Layer | Role | Characteristic |
| --- | --- | --- |
| Redis | First gate | TTL-based, blocks double-submits in ms |
| RDB | Durable record | Permanent result storage, fallback if Redis is down, audit trail |

The deeper reason for this split: the idempotency record needs to commit in the same transaction as the business operation (durability), but the gate that blocks duplicate submits needs to respond in single-digit ms (latency). One store rarely satisfies both. Redis as a fast gate, RDB as the source of truth.

**Failure modes to handle explicitly:**

- Redis down at gate-check time → fall through to RDB lookup, accept the latency.
- RDB durable record absent but Redis says "in flight" → wait briefly, then re-check; the first writer's transaction may not have committed yet.
- Redis says "done" with cached result → safe to return; the result was written *after* the RDB commit.

The order of writes matters: business operation + RDB record commit first, then Redis. Reverse order risks Redis saying "done" before RDB has the proof.

---

## TTL: The Domain Decides

TTL should cover the maximum window in which a retry with the same intent could arrive.

| Domain | Retry window | TTL |
| --- | --- | --- |
| Commerce (gift/purchase) | Seconds to minutes (network retry) | Minutes to hours |
| Finance (loan application) | Hours to days (user retries manually) | Days to weeks |
| Background job re-submission | Job scheduler's max retry interval | ≥ scheduler retention |

After TTL expires, a request with the same key is treated as a new intent. For business-level deduplication beyond the retry window, use a business key (order number, application ID) with a unique constraint — that's a different mechanism with different semantics.

---

## Domain Weight

The same "idempotency failure" means very different things depending on context.

- **Commerce:** duplicate charge = bad UX, recoverable via refund. Idempotency is a quality-of-service feature.
- **Finance (credit):** duplicate execution = possible compliance violation. The act itself is a legal event. "Just refund it" is not always an option — the audit trail already shows two executions.

In financial systems, idempotency is the first line of defense, not a recovery mechanism. The system must not produce the duplicate in the first place.

---

## Timeout vs. Double-Submit

These are different problems. Conflating them produces broken designs.

| Problem | Solution |
| --- | --- |
| Accidental double-submit (button tapped twice) | Idempotency Key on the same request |
| Timeout — outcome unknown | `GET /orders/{key}` to check status |
| Timeout — confirmed failure, want to retry | Query first → retry with same key |

**Correct timeout flow:**

1. Do not retry immediately on timeout — the original request may have succeeded.
2. `GET /orders/{idempotency-key}` to learn the actual outcome.
3. If the original failed → retry with the same key. Server deduplicates.
4. If the original succeeded → no retry needed, return success to user.

This flow requires the server to expose a status-by-key endpoint, which is itself a design decision: the key becomes a first-class identifier, not just a deduplication token.

---

## Retry Strategy: Exponential Backoff + Jitter

Immediate retry after timeout starves a recovering server (thundering herd).

- **Exponential backoff:** double the interval each attempt — 1s → 2s → 4s → 8s.
- **Jitter:** add random noise to the interval. Without it, clients that failed together retry together, reproducing the exact spike that caused the failure.
- Backoff without jitter is half a solution. AWS's "Exponential Backoff and Jitter" article is the standard reference; full jitter (`sleep = random(0, base * 2^n)`) is the recommended default.

---

## Concurrent Duplicates

Two requests with the same key arriving simultaneously — before either has stored a result. A naive implementation processes both.

The gate must be atomic. Options:

- **Redis SETNX:** set the key only if absent, atomically. First writer wins; second sees the key exists and waits or returns "processing."
- **DB unique constraint:** insert the key as a row. The second concurrent insert fails with a constraint violation — catch it and return the stored result once the first transaction commits.

Both work. Redis SETNX is faster; the DB unique constraint gives durability for free if you're already writing to RDB. Many systems use both: Redis to fast-fail the obvious cases, DB unique constraint as the actual correctness guarantee.

---

## The Failure Mode This Pattern Targets

A narrow window exists where every other approach fails: the business operation has committed, but the response never reaches the client. Server died after commit, network dropped on the response, load balancer reset the connection — the cause doesn't matter. The client sees a timeout and treats it as failure.

The asymmetry: **the server's view says "success," the client's view says "unknown."** Logs show the operation completed; the client retries because it has no other choice.

Why this case specifically:

- **Pre-commit failures are easy.** Anything that fails before COMMIT rolls back. The retry processes from scratch. No special handling needed.
- **Failed business operations are easy.** The error response itself tells the client what happened.
- **Concurrent double-submits are easy *with a key*** — the second request sees the key in flight and waits or returns the in-progress status. (Without a key they're not easy at all; that's part of why the pattern exists.)
- **Post-commit-pre-response failures are uniquely hard without this pattern.** The effect is real, the client is blind to it, and a naive retry doubles the effect.

This is the case `Idempotency-Key` is engineered around. With the pattern in place, a retry carrying the same key finds the stored result and returns it. Without it, the same retry triggers a duplicate operation.

### Where the implementation goes wrong

Knowing the case isn't enough — the pattern only works when result storage is atomic with the business operation. The common mistake:

```
1. BEGIN transaction
2. Execute business logic
3. COMMIT transaction
4. Store idempotency result  ← crash here leaves the key without a record
```

After step 3, the operation is real. After a crash before step 4, the key has no stored result, the next retry sees "new request," and the operation reprocesses. The idempotency-key header is present, the table exists, the deduplication code runs — and a duplicate still goes through.

The fix is structural: step 4 must be inside the same transaction. The result row commits with the business write, atomically. There is no in-between state where the operation succeeded but the dedup record is missing.

---

## Common Misconceptions

**"GET is always idempotent."**

By spec, yes. In practice, GETs with counter increments or cache-warming side effects break idempotency. The method name is not a guarantee.

**"PATCH is idempotent."**

Only when the operation is a replacement. Increment/decrement operations make it non-idempotent.

**"An idempotency key is enough."**

Only when result storage is atomic with the business operation. A crash between processing and storage leaves the key without a stored result — the next retry reprocesses.

**"Won't Redis TTL expiry make it unsafe?"**

TTL defines the retry window, not the safety guarantee. Requests outside that window are new intents. Permanent deduplication belongs to business keys with unique constraints, not idempotency keys.

**"Idempotency key and event_id are the same thing."**

Different layers. Idempotency keys are client-generated for sync API calls; event IDs are producer-generated for async messages. The storage policy, TTL, and consumer model all differ. See the event idempotency note.

---

## The Full Picture

Storing the key in Redis is the starting point. The harder questions: when exactly is the result stored, and is it in the same transaction as the business operation? How is the TTL derived — and what happens after it expires? What does the second concurrent request see before the first has finished? How does the timeout flow differ from the double-submit flow?

These questions come from having operated the thing, not just built it.