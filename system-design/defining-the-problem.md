# Defining the Problem: Turning an Ambiguous Ask Into a Defensible Spec

Every design starts as a sentence someone said in a meeting: "we need a
notification system," "make the checkout exactly-once," "it has to be
real-time." That sentence is not a spec. It is a compressed pointer to a spec
that lives in the stakeholder's head, half of it never articulated, some of it
wrong. The pre-design job — the one that decides whether the whole build is
aimed at the right target — is decompressing that sentence into the two or
three numbers and one reframing that actually flip the architecture, and
*refusing to decompress the rest*.

This is the judgment a coding agent structurally will not do. Hand a model
"build an exactly-once payment flow" and it will build one — correctly,
fast, and against a spec nobody validated. It optimizes toward the objective
you give it; it does not push back that the objective is underspecified, that
"exactly-once" is the wrong framing, or that the peak-QPS number you assumed
was never measured. Naming which requirement is load-bearing, and reframing
the ask before a single box is drawn, is the work that has to stay on the
human side of [the dividing line](../philosophy.md).

---

## The failure this note prevents

You build to an assumed scale and an assumed consistency class that were never
validated. Every test passes. The demo works. Then the real peak arrives at
8× the number you guessed, or the "eventually consistent is fine" you assumed
turns out to double-charge on a stale read — and the system is *correct for a
spec nobody had*. The fix is not a patch; it is a redesign, now under
incident pressure, with the wrong architecture already in production and
consumers depending on its contract.

> The most expensive bug is not in the code. It's in the problem statement —
> a wrong assumption about scale or consistency that the code faithfully
> implements. No test catches it, because the code does exactly what the spec
> said. The spec was the defect.

This is the same asymmetry as the [ADR](./adr-rfc.md): a wrong problem
statement, like a lost decision context, surfaces weeks later borne by
everyone downstream at once. The difference is that the ADR records a decision
*after* the framing is settled. This note is about the framing itself — the
step before there is anything to record.

---

## Load-bearing vs cosmetic ambiguity

An ambiguous ask has a dozen open questions. The staff skill is triage: which
ones, once answered, *change the architecture*, and which ones resolve to the
same design either way. Spend your clarifying budget only on the first kind.

| | Load-bearing ambiguity | Cosmetic ambiguity |
|---|---|---|
| **Test** | Two plausible answers imply two different architectures | Every plausible answer implies the same architecture |
| **Cost of guessing wrong** | Redesign after launch | A config change, or nothing |
| **Examples** | Peak QPS (10× flips you from one box to sharded + queue); consistency class of the money path; cost of a duplicate vs a drop | Exact retention (90 vs 180 days — same store); admin-panel colors; which JSON field naming |
| **Where the answer comes from** | Data or an explicit stakeholder decision — never a guess | Pick a sane default and move |

The two failure directions are symmetric and both fatal:

- **Under-clarify** the load-bearing ones → you build correctly to a spec
  nobody had. This is the redesign-after-launch failure above.
- **Over-clarify** the cosmetic ones → analysis paralysis. You burn the
  stakeholder's patience — and your own week — resolving questions whose answers
  don't move a single box, while the load-bearing one hides in the noise.

> The dividing line: **does resolving this ambiguity change the architecture,
> or just a parameter?** If the design is identical under both answers, stop
> asking — pick a default and note it. If the two answers are two different
> systems, this is the only kind of question worth a stakeholder's time.
> Treating every open question as equally urgent is not diligence; it's the
> inability to triage, and it reads as junior in the room.

The tell that you found a load-bearing one: you cannot draw the first box
until it's answered. If you *can* draw the box and would just tune it later,
it's cosmetic — assume, document the assumption, move on.

---

## The load-bearing numbers — the short list that flips designs

Most systems are pinned by a handful of numbers. Drive these out first,
because each has a threshold where the architecture changes shape. The point
is not the number itself but the *design decision it gates*.

| Number to pin | What it flips | The redesign you avoid |
|---|---|---|
| **Peak QPS** (not average — peak, and the spike ratio) | Single node vs sharded; sync vs queued; cache or not | Sizing to average, dying at the flash-sale peak that is the whole point of the system |
| **Read:write ratio** | Read replicas / cache-heavy vs write-path optimization / partitioning | Optimizing the wrong path; a cache layer on a write-bound system |
| **Consistency class, per data type** | Linearizable leader path vs eventual replica read — *per slice* | A stale read making a wrong money/inventory decision, or paying for strong consistency on a receipt email |
| **Cost of a duplicate vs cost of a drop** | At-least-once + idempotent receiver vs at-most-once | Building dedup for a telemetry pipeline that never needed it, or dropping a payment that needed at-least-once |
| **Latency target** (p99, and who acts on it) | Sync request path vs async + notify; how much you cache | A synchronous design that can't meet p99, or an async one the user experience didn't need |
| **Data volume + growth** | Single store vs sharded/partitioned; retention vs archival | Designing for today's 10 GB, hitting the single-node ceiling in two quarters |

The discipline underneath all of these is **set it from data, don't guess.**
A model will cheerfully invent "let's assume 1000 QPS" and design around it;
only you know that number has to be *defended* — from a traffic estimate, a
current-system metric, a business projection someone will sign. An assumed
number is a load-bearing beam you never checked was there.

> Not every number is load-bearing. The art is finding the *two or three* that
> flip the design and pinning those hard, while defaulting the rest. A spec
> that pins twenty numbers is as unusable as one that pins none — it hides the
> three that mattered under seventeen that didn't.

Two subtleties that separate the pinned-number from a vanity metric:

- **Peak, not average.** A system sized to average throughput and hit by a
  10× peak is the canonical redesign-after-launch. The
  [payment walkthrough](../design-walkthrough/payment-system.md) pins
  "~500 orders/sec, flash-sale spikes to ~3k/sec" precisely because the spike
  ratio — not the mean — is what decides whether inventory can ride a hot row
  or must shed load. Ask for the peak and the spike shape; the average tells
  you almost nothing about where the system breaks.
- **Consistency is per-data-type, not per-system.** "Is it consistent?" is the
  wrong question. The right one, from
  [consistency-models](./consistency-models.md) and demonstrated in the
  payment walkthrough's per-class table: default *everything* to the weakest
  correct model, then buy back linearizability only for the slice where a
  stale read causes a wrong decision. A system is rarely all-strong or
  all-weak; it's a strong 5% carrying a weak 95%, and the spec's job is to
  draw that line.

---

## The spine: reframe the stated ask

The highest-leverage pre-design move is not answering the stakeholder's
question — it's *correcting the question*. A stated ask often names a
mechanism when it means an effect. Build the literal mechanism and you get the
wrong system, correctly.

The archetype, owned in full by [exactly-once](./exactly-once.md): someone
asks for "exactly-once delivery." Delivery cannot be exactly-once — it's the
Two Generals result, not an engineering gap. What they actually need is an
exactly-once *effect*, delivered by at-least-once + an idempotent receiver.
Build the literal ask and you go hunting for a broker feature flag that
doesn't deliver the guarantee across your database boundary. Reframe it and
the design falls out.

| Stated ask (the mechanism) | What they mean (the effect) | Why the literal build is wrong |
|---|---|---|
| "Exactly-once delivery" | Exactly-once *effect* | Delivery can't be once; you'd chase a guarantee that evaporates at the first external side effect ([exactly-once](./exactly-once.md)) |
| "Make it real-time" | p99 under N ms / user sees their own write | "Real-time" buys a websocket layer the SLA never required, or hides that only read-your-writes was needed |
| "It must never lose data" | Which data — and durable to fsync, or replicated? | Uniform max-durability everywhere is unaffordable; the ask is really "the money path is durable, the analytics aren't" |
| "Strongly consistent" | Which slice reads must be fresh | Whole-system linearizability throttles the 95% that tolerated lag |
| "Scale to a million users" | Peak concurrent QPS on the hot path | Total registered users is a vanity number; concurrency on one path is the constraint |

> The reframe is the staff move a model won't make. The agent takes
> "exactly-once delivery" at face value and builds toward it, because it
> optimizes the objective as stated. Reframing "delivery" to "effect,"
> "real-time" to "p99," "never lose data" to "which data, how durable" is
> pre-design judgment the prompt gives the model no signal to perform. You have
> to supply the spine to say "that's the wrong framing — here's what you
> actually need."

The mechanism of the reframe is always the same: the stated ask names a
*how*; you translate it to a *what* and a measurable threshold. "Real-time"
→ a number and a reader ("who acts on this, and within how long?"). "Never
lose data" → a data class and a durability level. Once it's a threshold, it's
testable, defensible, and no longer a mechanism you're married to before you've
judged whether it's right.

---

## What a defensible spec contains — and what it deliberately omits

The output of this phase is a spec a stranger can attack. Its power is not
completeness; it's that every load-bearing claim is *defended* and every
omission is *deliberate*.

- **The two or three pinned numbers, with their source.** Not "assume 1000
  QPS" but "1000 QPS peak, from current-system p99 × 2.5 projected growth."
  A number without a source is a guess wearing a suit.
- **The consistency class per data type.** The strong slice named explicitly,
  everything else defaulted to eventual with a one-line why.
- **The failure-cost priority.** Is a duplicate worse than a drop, or the
  reverse? This single ordering picks your delivery semantics and cannot be
  inferred from the happy path.
- **The reframes, stated as reframes.** "Requested exactly-once delivery;
  scoping to exactly-once effect via idempotent receiver, because delivery
  cannot be once." This is where you show your work so the stakeholder can
  disagree *before* you build.
- **Explicit non-goals and assumptions.** Every cosmetic ambiguity you chose
  not to resolve, recorded as an assumption a reviewer can challenge. The
  payment walkthrough's "what I deliberately defer" is this discipline in
  practice — naming multi-currency, fraud, and PSP-failover as *out* is what
  keeps the exactly-once spine legible.

What it omits: everything the model does well. No implementation, no schema,
no box diagram yet — those are downstream of the framing and cheap to
regenerate once the framing is right. A spec that dives into table columns has
skipped the only step that was hard.

This is where the note hands off. Once the framing is settled, the
*reversible* decisions inside it get decided fast and the *irreversible* ones
(the data format, the public contract, the consistency boundary you're
committing to) get an [ADR](./adr-rfc.md); that note owns the decision record.
The framing is never final either: load moves, the spec you defended today
stops fitting, and that's the trigger
[evolving-architecture](./evolving-architecture.md) owns. This note owns the
*first* framing; that one owns re-framing under a named force.

---

## The trap: over-fitting the spec to the first number you hear

The symmetric danger to under-specifying is *over-committing* to a number
someone said offhand. A stakeholder says "millions of users" and you shard on
day one for a system that will see 200 concurrent requests. You've spent the
irreversible-decision budget — a sharding key is a one-way door
([evolving-architecture](./evolving-architecture.md)) — on an unvalidated
number.

The move is to pin the number *and* its confidence. A load-bearing number you
trust gets designed around. A load-bearing number you *don't* trust gets an
abstraction seam so the design can absorb being wrong: a repository interface
that could become sharded, a queue you could add, a consistency boundary you
drew but didn't yet enforce strongly. This is "engineer reversibility in"
from [evolving-architecture](./evolving-architecture.md), applied at the spec
layer — you convert "we guessed the scale" from a one-way door into a two-way
one before you walk through it.

> Pin the number, but pin your confidence in it too. Design hard around the
> numbers you can defend; leave a seam around the ones you can't. Committing an
> irreversible decision to an unvalidated number is the same failure as not
> pinning the number at all — you've just made the wrong guess load-bearing.

---

## The staff move

The staff move is to refuse to start designing until you've separated the two
or three load-bearing ambiguities from the dozen cosmetic ones, and to spend
your entire clarifying budget on the former. Drive out peak QPS (not average),
the read:write ratio, the consistency class per data type, the cost of a
duplicate versus a drop, and the latency target — each defended from data, not
guessed. Then have the spine to reframe the stakeholder's mechanism into an
effect: "exactly-once delivery" into "exactly-once effect," "real-time" into a
p99 and a reader, "never lose data" into a data class and a durability level.
Then write it down as a spec a stranger can attack, with every pinned number
carrying its source and every omission recorded as a deliberate assumption. A
coding agent will build precisely the system you specify — which is exactly
why the specification, not the code, is the load-bearing artifact, and why
getting the problem statement right is the one part of the work that was never
in the prompt.

Related: [exactly-once](./exactly-once.md),
[consistency-models](./consistency-models.md), [adr-rfc](./adr-rfc.md),
[evolving-architecture](./evolving-architecture.md),
[payment-system](../design-walkthrough/payment-system.md),
[philosophy](../philosophy.md).
