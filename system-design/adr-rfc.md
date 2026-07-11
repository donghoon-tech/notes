# Documenting Technical Decisions (ADR / RFC)

The code says *what* the system does. It never says *why* this and not the
three alternatives you rejected. Six months later — after a reorg, after the
author left — someone hits the constraint you designed around, sees no reason
for it, and "fixes" it. The decision didn't survive because it was never
written down. A decision document is the durable artifact of judgment; the code
is just its current compilation.

This note is about the *vehicle*, not the prose. A decision's reach is bounded
less by how well it's built than by whether a room agrees to it and a future
team honors it. The written trade-off is what carries it past the people in the
meeting — verbal consensus dies when they leave the room.

---

## RFC vs ADR — Different Jobs, Different Tense

They are not two formats for the same thing. The split is **before vs after**,
and it maps onto the reversibility of the decision.

| | RFC (Request for Comments) | ADR (Architecture Decision Record) |
|---|---|---|
| **Tense** | Future — "we propose to" | Past — "we decided to" |
| **Job** | Build consensus *before* committing | Record context *after* deciding |
| **Audience** | Reviewers who can still say no | A future engineer asking "why is it like this?" |
| **Length** | Long: alternatives, trade-offs, open questions | Short: one decision, its context, its consequences |
| **Lifecycle** | Discussed, amended, then closed | Immutable once accepted; superseded, never edited |
| **Mutability** | Living until accepted | Append-only log |

An RFC is the *negotiation*. An ADR is the *minute* — the single fact "we chose
Postgres over DynamoDB on 2026-03, here is the context that made that right."
A large RFC often *spawns several ADRs*: the discussion settles many questions,
and each settled question becomes its own immutable record. Conflating them is
the common mistake — people write a 12-page ADR that nobody reads, or a
two-line RFC that decides nothing.

> The dividing line: are you asking permission or recording a verdict? Asking →
> RFC. Recording → ADR. If you're doing both in one document, you'll do both
> badly.

---

## The Real Axis: One-Way vs Two-Way Doors

Bezos's framing is the single most useful filter for how much process a
decision deserves. A **two-way door** is reversible — walk through, and if it's
wrong, walk back at low cost. A **one-way door** is hard or impossible to
reverse: the cost is sunk into schemas, public contracts, data you can't
recall.

The failure mode in most orgs is **applying the wrong weight to each.**

| Decision | Door | Right amount of process |
|---|---|---|
| Choice of logging library | Two-way | None. Just pick one and move. |
| Internal service's retry policy | Two-way | A PR comment, maybe |
| Public API contract / event schema | One-way | RFC. Once consumers depend on it, you own it forever |
| Primary datastore for a domain | One-way (ish) | RFC + ADR. Migration is a project, not a patch |
| Auth model / tenancy boundary | One-way | RFC, threat-modeled, signed off |
| Sync→async refactor of a flow | Mostly reversible | Lightweight; the *contract* change inside it may not be |

The judgment that matters is **calibration**: spend the RFC budget on one-way
doors, and refuse to spend it on two-way doors. Demanding a design doc for a reversible
choice is not rigor — it's tax. The most expensive thing about a two-way door is
treating it like a one-way door and stalling. Conversely, the most expensive
incident is a one-way door someone walked through in a Slack thread.

A subtlety that ages teams badly: doors are **one-way in the direction of
adoption, not in the moment of choosing.** Choosing the schema is cheap;
choosing it *after* 40 services emit it is the one-way door. The reversibility
of a decision *decays over time* as dependents accrue — this is exactly the
mechanism in [evolving-architecture.md](./evolving-architecture.md): cheap
decisions calcify. The RFC's job on a one-way door is partly to keep a future
"wrong" cheap to fix: version the schema (ship `v2` alongside `v1` instead of
editing the field 40 consumers read), or hide it behind an adapter so a change
touches one seam, not 40 call sites. Either turns "rewrite everyone" back into
"change one place." See [api-contract-design.md](./api-contract-design.md) for
the contract-level mechanics.

---

## What Actually Belongs In the Document

The format matters far less than whether these are present. A good RFC is
*adversarial against itself* — it makes the case for the options it rejects
strongly enough that the reader trusts the conclusion.

- **Context / forces.** What constraints made this a decision at all? (Load,
  deadline, team skill, existing systems, compliance.) Strip these and the
  decision looks arbitrary later — which is exactly how it gets reverted.
- **The alternatives, steelmanned.** Not a strawman list. The reader must see
  that the road not taken was genuinely considered. An RFC with one option
  isn't a proposal, it's an announcement.
- **The trade-off you're accepting.** Every real decision *costs* something.
  "We chose X, and we are knowingly paying Y for it." A document that lists only
  upsides is hiding the ball, and reviewers can smell it.
- **Consequences — including the bad ones.** What becomes harder now? What did
  we just make a one-way door?
- **Explicit non-goals.** Half of scope-creep in review comes from readers
  importing problems the author never meant to solve.

What does **not** belong: implementation minutiae that the code expresses
better, and anything that re-derives from first principles things the team
already agrees on. The document should be the *minimum* that lets a stranger
reconstruct your reasoning — no more.

> A reviewer's trust is a function of how well you argued *against* yourself. If
> the strongest objection in the room is one you already raised and answered in
> the doc, you've won before the meeting.

---

## Persuasion: The Document Is the Argument

A written trade-off persuades in ways a meeting cannot, and this is the
mechanism by which a decision outlasts the meeting that made it:

- **It scales past the room.** The people who weren't in the meeting — the
  on-call who inherits it, the team that forks the service — are reached only by
  the artifact. Verbal consensus has a blast radius of one room and a half-life
  of a week.
- **It moves disagreement off personalities.** A review comment argues with the
  *document*, not with you. This is what lets a junior engineer overturn a
  principal's proposal without it being a status fight — the text is the
  arena.
- **It forces your own thinking.** Writing the trade-off down exposes the
  blank you can't fill — and sends you to go fill it. Most ideas come back
  stronger for it; the ones that can't be filled even after you look were the
  weak ones, and they die before anyone else reads them. A trade-off you can't
  write down is usually one you haven't actually made yet.

The anti-pattern is the **decision-laundering RFC**: written after the choice is
locked, alternatives included only as theater, circulated for rubber-stamp.
Reviewers learn this fast and stop reading. An RFC is only worth writing if the
answer can still change — otherwise write the ADR and skip the ceremony.

---

## When Not To — and How This Dies Quietly

Everything above describes the discipline working. The two failures below are
how it never starts, and how it stops — and they cost more teams their decision
history than any badly-written document ever has.

**The system itself is a door — don't walk through it too early.** The whole
note argues for calibrating process to the decision. Apply that one level up:
adopting an ADR/RFC practice is *itself* a two-way door for a small, co-located
team, and forcing it there is the same tax the note warns about, just wider. In
a room of three where every decision is still reversible and the context lives
in shared memory, an `docs/adr/` folder has negative ROI — it's ceremony with no
reader. The discipline earns its keep only when one of two things becomes true:
**the context starts outliving the people** (a reorg, a departure, a second team
forking the service), or **you begin walking through actual one-way doors** (a
schema consumers will depend on, a datastore you can't un-choose). Before either,
the honest move is a one-line commit message and nothing more. The dividing line:
*is there a future stranger who will need the reason, and is the reason about to
become unrecoverable?* If no on either, you're documenting for an audience that
doesn't exist.

**The common death is not a bad ADR — it's the empty folder.** The failure the
"decision-laundering" section catches is a *written* document gone wrong. The far
more common one is the document that was never written. A team writes ADR-0001
through 0003 with enthusiasm, then a busy sprint arrives and 0004 never happens.
Six months later the folder holds three records and the fifty decisions made
since live nowhere — and now the folder's *existence lies*: a newcomer reads it
and believes this is a team that records its reasoning, when it stopped after
week two. The cause is never laziness; it is **friction** — the work between
"we decided" and "it's recorded." Every extra step there (book a meeting, open
the template, fill five sections) is one more place a busy person drops it.

The fixes are all friction-reduction, and they are the actual best practice hiding
behind the ideal:

- **One paragraph, not a template to fill.** The four-section form is a ceiling,
  not a floor. A durable ADR is often five sentences: what we decided, the forces,
  the alternative we killed, the trade-off we're buying. If writing it feels like
  homework, it won't get written next time.
- **Ride an existing ritual; don't add one.** Commit the ADR file *in the same
  PR* that implements the decision — same branch, same review — so it ships when
  the code ships and no one has to remember a separate step. A standalone "ADR
  meeting" is a second habit to sustain, and the second habit is the one that
  dies when the sprint gets busy.
- **Record one-way doors, not everything.** A team that resolves to document
  *every* decision documents none within a quarter. Aim the practice only at the
  irreversible calls — the coverage target that's actually sustainable is "the
  decisions a future stranger can't reverse cheaply," not 100%.

> An append-only log that no one appends to is worse than no log: it doesn't just
> lose the decisions, it advertises a rigor the team doesn't have. The measure of
> the practice is not the quality of your best ADR — it's whether ADR-0040 exists
> at all.

---

## Surviving the Reorg — Why Decisions Decay

A decision dies when the *context* behind it is lost and only the *consequence*
remains. After a reorg, the new team sees the weird constraint, not the load
test that justified it, and removes it. The whole point of an ADR is to keep
context and consequence welded together so they travel as one unit.

What makes a decision durable:

- **Co-located with the code, version-controlled.** ADRs live in the repo
  (`docs/adr/NNNN-title.md`), not a wiki that rots when the team that owned the
  wiki dissolves. The decision and the thing it decided about move together,
  diff together, get found by grep.
- **Append-only.** You never *edit* an accepted ADR to reflect a new decision —
  you write a new one that **supersedes** it and link back. The superseded one
  stays. The audit trail ("we used to do X, then learned Y, now do Z") is the
  asset; erasing it destroys exactly the context the next person needs.
- **Immutable identity.** Stable number/title so a code comment can say
  `// see ADR-0017` and that pointer survives forever.
- **Decision, not narrative.** A doc *organized by date* — a running log of what
  was discussed each meeting — ages into noise: to learn the current rule you
  must read the whole timeline and guess which entry still holds. The date still
  belongs *inside* an ADR as context; it just isn't the organizing axis. An ADR
  states the *current standing decision* plus the context that produced it, so
  one file is the answer — and it stays timeless until explicitly superseded.

> A decision survives a reorg only if a stranger can find *both* the rule and
> the reason for it without asking anyone. If the reason lives only in someone's
> head, the rule has the lifespan of that person's tenure.

This is the same discipline as a changelog or a WAL: an append-only record of
*what changed and why* is what lets a system — or an org — recover state after
the people who held it in memory are gone.

---

## The AI-Era Twist

When [agents and LLMs write the code](./ai-era-review.md), the *why* migrates
out of the implementation faster than ever — generated code rarely carries the
rejected alternatives, and the model will happily "fix" a constraint it has no
record of. The decision document becomes the **load-bearing artifact of human
judgment**: the place where a human asserts what is correct, safe, and
appropriate, in a form the next human (or the next agent's context window) can
read. Code is increasingly cheap and regenerable; the *decision* — which
trade-off you accepted and why — is the part that was never in the prompt.
That is the understanding that can't be outsourced.

---

## Match the Document to the Door

Match the document to the door. Two-way door: decide in a comment and move,
because process on reversible work is just latency you're adding to your own
team. One-way door: write the RFC, steelman the alternative you're about to
kill, name the trade-off you're buying, and once it's settled, leave behind an
ADR so terse and so co-located that the engineer who inherits it after the next
reorg can reconstruct your reasoning without ever finding you. The artifact, not
the meeting, is what carries a decision past the room that made it — and the
unit of judgment that survives.

Related: [evolving-architecture.md](./evolving-architecture.md),
[api-contract-design.md](./api-contract-design.md),
[threat-modeling.md](./threat-modeling.md),
[ai-era-review.md](./ai-era-review.md), [philosophy.md](../philosophy.md).
