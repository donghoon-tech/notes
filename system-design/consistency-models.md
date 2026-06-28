# Consistency Models

"Consistency" is overloaded. The C in ACID and the C in CAP are different
things. "Strong consistency," "eventual consistency," "read-your-writes" —
these get used as if they sit on one dial, but they belong to different
axes. This note fixes the coordinate system the rest of the wiki assumes:
when a note says "this is fine if consistency is loose," *which* consistency,
and how loose.

A consistency model is a **contract between a data store and its clients**:
given some history of reads and writes, which return values are legal? A
stronger model permits fewer surprising outcomes — and costs more in
latency, availability, or both. The engineering question is never "is this
consistent?" but "which anomalies can this workload tolerate, and what does
ruling them out cost?"

---

## Two Axes People Conflate

There are two different hierarchies wearing the same word.

| Axis | Question it answers | Examples (strong → weak) |
| --- | --- | --- |
| **Data / transaction consistency** | Within one logical operation, do invariants hold? | Serializable → Snapshot → Read Committed |
| **Distributed / replication consistency** | Across replicas, what can a reader observe? | Linearizable → Sequential → Causal → Eventual |

The first axis is the isolation levels — it lives inside a single store and
is about *concurrent transactions*. See [transaction isolation](../data/transaction-isolation.md)
(planned). The second axis is about *where your read landed* in a replicated
system.

They interact (a distributed database has both), but conflating them
produces nonsense like "we use eventual consistency, so we set isolation to
Read Committed" — two unrelated dials. This note is mostly about the second
axis; the [isolation note](../data/transaction-isolation.md) (planned) owns
the first.

---

## The Replication Spectrum

From strongest to weakest. Each level *permits* everything the weaker ones
permit, plus rules out one more class of anomaly.

| Model | Guarantee | Rules out | Cost |
| --- | --- | --- | --- |
| **Linearizable** | Every operation appears to take effect atomically at a single point between its call and return; reads see the latest completed write | Stale reads, reordering | Coordination on every op; unavailable under partition (CAP) |
| **Sequential** | All clients see operations in *some* single global order, consistent with each client's own program order | Per-client reordering | Global agreement (≈ CP); not tied to real time |
| **Causal** | Operations that are causally related are seen in order by everyone; concurrent ops may be seen in any order | Effect-before-cause | No global clock needed; available under partition |
| **Eventual** | If writes stop, replicas converge — eventually | Permanent divergence | Almost nothing — and almost no guarantee |

The jump that matters most in practice is **linearizable vs everything
below it.** Linearizability is the only model where "read the latest value"
is meaningful, because it's the only one with a single real-time order.
Everything weaker means a read can legally return a value that is *already
stale by the time you read it.*

Note this is the spectrum's own ranking, not CAP's. **The C in CAP refers to
linearizability — the top row alone.** Sequential and causal are weaker
models that CAP does not call "consistent"; a causally-consistent store is,
in CAP's vocabulary, an AP system. Keep the two rankings separate: this
table grades how strong a model is; CAP only asks whether you hit the top of
it.

> The dividing line, stated as a question: **does a stale read here cause a
> wrong decision?** If yes, you need linearizable reads for that path — and
> you pay for them. If no (a feed, a count, a recommendation), you're on the
> weak side and you should stop paying.
>
> This is the same dividing line as
> [cache-stampede](./cache-stampede.md)'s "does the user act on this number?"
> A cache *is* a weakly-consistent replica. SWR is eventual consistency with
> a staleness bound, chosen on purpose.

---

## Client-Centric Guarantees (the useful middle)

"Eventual" is too weak to build on directly — "converges someday" says
nothing about what *one user* sees. The practical guarantees are
**session guarantees**: promises scoped to a single client's session, far
cheaper than linearizability but enough to kill the anomalies users
actually notice.

| Guarantee | Promise | The anomaly it kills |
| --- | --- | --- |
| **Read-your-writes** | You see your own writes immediately | Post a comment, refresh, it's gone |
| **Monotonic reads** | Once you've seen a value, you never see an older one | Refresh and watch the count go *backwards* |
| **Monotonic writes** | Your writes apply in the order you issued them | Set name, then set avatar — avatar lands first, name overwrites |
| **Writes-follow-reads** | A write that reacts to a read is ordered after it | Reply appears before the message it replies to |

These four are **orthogonal** — each rules out a different anomaly and none
implies another. Monotonic reads doesn't give you read-your-writes (you can
stop seeing values go backwards yet still not see your *own* latest write);
read-your-writes doesn't give you monotonic reads (your write is visible to
you, but two later reads against different replicas can still go backwards
on *other* data). You buy them à la carte — pick exactly the ones whose
anomaly your users would notice, which is the whole point of the closer
below.

These are the everyday bugs of read-replica architectures. A primary +
async replicas system is eventually consistent globally, but you can layer
**read-your-writes** on top. Two common tricks, neither free:

- **Route the user's reads to the primary for a few seconds after a write.**
  Simple, but it concentrates that user's read load on the primary — fine
  for one user, a problem if a large fraction of traffic is recently-written.
- **Pin reads to a replica that has caught up to the write's position.** No
  primary load, but it assumes the client carries that position around (an
  LSN / GTID token) and that some replica is actually caught up — under lag,
  there may be none, and the read blocks or falls back to the primary anyway.

That's the realistic target for most apps: globally eventual, per-session
"feels consistent." The cost lives in *how* you hide the lag — pursued in the
[replication lag](../data/replication-lag.md) note (planned).

> "Eventual consistency" as a product decision almost always means
> "eventual + read-your-writes + monotonic reads." Naked eventual
> consistency — where a user's own refresh can show their data vanishing
> and reappearing — is rarely acceptable and rarely what people mean. Say
> which session guarantees you actually need; that's the real spec.

This connects to [replication lag](../data/replication-lag.md) (planned): the lag is
the window in which session guarantees break, and the routing tricks above
are how you hide it.

---

## CAP, Stated Honestly

CAP is widely misquoted as "pick two of three." The honest statement:

**When a network partition occurs, a system must choose between remaining
consistent (linearizable) and remaining available. With no partition, you
can have both.**

So CAP is a choice that only bites *during a partition*:

- **CP** — refuse requests rather than serve possibly-stale/conflicting
  data. Consensus systems (etcd, ZooKeeper, the metadata layer of most
  databases) are here. They'd rather be down than wrong.
- **AP** — keep serving, accept divergence, reconcile later. DNS, shopping
  carts, CRDTs, most caches. They'd rather be wrong-for-now than down.

Two refinements that matter more than CAP itself:

- **It's per-operation, not per-system.** The same system can serve a
  linearizable read for "account balance at withdrawal" and an eventual
  read for "profile photo." Don't label a whole system CP or AP; label the
  operation.
- **PACELC** completes it: **if Partition, choose A or C; Else (normal
  operation), choose Latency or Consistency.** This is the one that bites
  daily. Even with zero partitions, linearizable reads cost a round-trip to
  the leader (or a quorum). Most systems are "PC/EL": consistent under
  partition, but trading consistency for latency in normal operation —
  because partitions are rare and latency is every request.

> CAP describes the rare day. PACELC's "EL vs EC" describes every other
> day, and that's where the bill actually comes due — in the p99 of every
> read. See [tail latency](./tail-latency.md) (planned).

---

## How This Maps Onto the Rest of the Wiki

This note is the coordinate system; here's where existing notes plot:

- **[cache-stampede](./cache-stampede.md) / SWR** — deliberately eventual,
  with a staleness bound (soft TTL). The whole "is this data the basis for a
  decision?" table is a consistency-model decision dressed as a caching
  decision.
- **[inventory concurrency](./inventory-concurrency-control.md)** — the
  Redis DECR path is eventual (weak, needs correction); the DB atomic UPDATE
  path is linearizable on that row. The table's "consistency" column *is*
  this axis.
- **[idempotency](./api-idempotency.md) / [event idempotency](./event-idempotency.md)**
  — at-least-once delivery is a weak-consistency artifact; idempotency is how
  you recover correctness on top of a weak guarantee. "Exactly-once effect"
  is achievable precisely because you *don't* demand exactly-once delivery.
- **[reliable messaging](./reliable-messaging.md)** — at-least-once again;
  the in-flight list trades stronger delivery semantics you can't have for a
  recovery mechanism you can.

The pattern across all of them: **pick the weakest model the workload
tolerates, then buy back exactly the guarantee the user notices** — a
staleness bound, an idempotency key, a read-your-writes route. Not more.

---

## Choosing — The Short Version

1. **Default to the weakest model that's correct.** Strong consistency is a
   tax on latency and availability; pay it only where a stale or reordered
   read causes a wrong decision.
2. **Identify the few linearizable operations.** Money movement, inventory
   at order time, uniqueness/allocation, anything audited. These get the
   leader round-trip. Everything else doesn't.
3. **For the weak majority, name the session guarantees.** "Eventual" is
   never the real answer; "eventual + read-your-writes + monotonic reads"
   usually is.
4. **Decide per operation, not per system.** A single service legitimately
   spans linearizable writes and eventual reads. CAP/PACELC is a property of
   the call, not the box.

The senior move is not knowing the definitions — it's refusing to pay for
linearizability on the 95% of reads that don't need it, while never letting
the 5% that do slip onto a stale replica.
