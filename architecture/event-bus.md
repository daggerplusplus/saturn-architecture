# Event Bus

How state changes in one service produce effects in others — and the durability gap at
the edge of that, plus the design that closes it.

## The contract: at-least-once, no distributed transaction

The event bus is a set of **Redis Streams** with **consumer groups**. The non-negotiable
invariant, stated up front so it can be argued about instead of rediscovered:

> **Delivery is at-least-once and there is no distributed transaction.** A state change
> and the event that describes it are not atomically durable *together*. Correctness
> therefore does not come from the bus — it comes from **idempotent consumers**.

The streams in use:

- **`…events`** — the domain event bus. Producers `XADD`; consumers read with
  `XREADGROUP` in a named group and `XACK` after their effect is durable.
- **`…dlq`** — dead-letter: an event that has failed past its retry budget lands here
  rather than blocking the group, and is surfaced to an alert.
- **`assistant:jobs`** (and its `:dlq`) — the *job queue* for the async worker; same
  stream mechanics, different purpose (work items, not domain facts).

Example event names (the domain is genericized): `content.ingested`,
`conversation.message.created`, `item.completed`, `resource.subscribed`.

**Every event carries an `idempotency_key`** — `"{event}:{business_id}"` — so a consumer
can tell a redelivery from a new occurrence and apply each logical event exactly once
*effectually*, even though the *delivery* may repeat.

## The producer pattern (today)

Every producer uses the **best-effort emit** shape:

```
1. write the state change to Postgres  →  COMMIT
2. XADD the event to the stream         (best-effort: a failure is logged, not propagated)
```

This is fine for the *common* path and keeps the write path fast and simple. It has one
sharp edge, which is the reason this document exists.

## The consumer pattern (idempotent, self-healing)

A consumer is shaped so that *re-applying* an event is a no-op:

- Read fresh entries (`XREADGROUP group … ">"`).
- Apply the effect **inside one Postgres transaction** using idempotent guards
  (`ON CONFLICT DO NOTHING`, `WHERE completed_at IS NULL`, result flags that make a
  re-run a no-op).
- `XACK` only **after** the Postgres commit.

**Self-heal for a dead consumer.** A subtle durability hole: an entry read into a pod's
**pending-entries list (PEL)** but not yet `XACK`ed — because that pod died in the
window — is *stranded*: a fresh `>` read never returns it, so its effect never happens.
The fix is a **lazy, off-hot-path reclaim**: when a consumer finds no fresh work, it
checks the oldest pending entry; if it's been idle well past a live consumer's ack time
(30s here — ~300× the ~1s a live ack takes), the consumer `XCLAIM`s it to itself and
re-drives it through the normal (idempotent) handler. Because re-application is a no-op,
reclaim is safe. This recovers a dead pod's orphan at a single replica (a restarted pod
reclaims it) and across replicas (any surviving group member can).

## The gap: a dropped emit is not recoverable (ACID item 3)

The best-effort emit has a real failure mode that was initially *documented wrong* and
then found:

- Each event is emitted **only on its fresh write**.
- The very guards that make the *write* safe to retry (`ON CONFLICT DO NOTHING`,
  `WHERE completed_at IS NULL`, the result flags) are exactly what make a **re-run a
  no-op that does not re-emit**.

So if the `XADD` is dropped — because step 2 failed, or the process died *between* the
Postgres commit and the `XADD` — the downstream effect is **permanently missed**, healable
only by a manual stream replay. (A lost `item.completed` event, for instance, means whatever downstream update it was
supposed to trigger simply never happens.) This was corrected in the code comments and closed with a
design, below.

## The durable outbox (the design that closes it)

The durable **transactional outbox** collapses the *durable* halves of steps 1 and 2 into
a single atomic Postgres write. **No distributed transaction is created** — the invariant
is preserved — the Postgres side simply becomes atomic and the Postgres→Redis leg stays
the same at-least-once, idempotent-consumer contract.

```
1. BEGIN
2.   write the state change
3.   INSERT the event row into the outbox        (same transaction)
4. COMMIT          ← state + intent-to-emit are now atomically durable together
5. (separately) a relay reads committed outbox rows and XADDs them, at-least-once
```

Two tiers:

- **Tier 1 — inside Postgres (atomic):** the outbox row is written in the *same
  transaction* as the state change, so "state changed + event recorded" can no longer
  half-apply.
- **Tier 2 — Postgres → Redis (eventually consistent):** a relay reads committed rows and
  `XADD`s them. It marks a row `sent` only *after* a successful `XADD`, so a crash in
  between re-queues and re-emits the row — and the consumer's existing `idempotency_key`
  dedup absorbs the double. Same semantics as today's bus, now backed by a durable
  backlog instead of a fire-and-forget `XADD`.

**Shape** (one outbox table *per producing schema*): a monotonic identity `id` (ordering +
batch window), the `event`, the `idempotency_key` (UNIQUE — belt-and-braces dedup),
`producer`, `occurred_at`, `payload JSONB`, and `status` (`pending | sent | dead`) +
`attempts`.

**The relay** claims rows with `SELECT … WHERE status='pending' ORDER BY id LIMIT n
FOR UPDATE SKIP LOCKED` — the same claim pattern the ingestion job queue already uses — so
multiple relay replicas are safe (disjoint rows), order is preserved within a batch, and a
poison row (past its attempt budget) is set `dead` and forwarded to the DLQ. `sent` rows
are purged on a retention schedule so the table doesn't grow unbounded while the backlog
(the *point*) is retained.

### Per-service table vs. one shared table

Because it's **one database with several schemas**, a *shared* outbox table *could* sit in
the same transaction as any service's state change — the usual "you can't write two
databases atomically" objection doesn't apply. So it's a real tradeoff, not a no-brainer.

- **Per-service table (recommended):** preserves the least-priv model (each service writes
  only its own schema, in its own transaction — no new cross-schema grants), isolates
  failure/backlog domains per service, and rolls out incrementally (one service at a time).
  The one real cost — duplicated relay/reaper/claim code — is **fully absorbed by a small
  shared crate**: write the generic relay/claim/insert/DDL once, parameterized by the
  table name, and instantiate it per service. This is the same "shared code, per-service
  data" split the rest of the system already uses.
- **Single shared table:** one relay, one gauge, one global ordering — but it breaks the
  least-priv schema model (N services need `INSERT` on a shared schema) and creates a
  shared failure domain across services.

**Decision: per-service table + a shared outbox crate.** The build is staged (the
event-producing services first) — see `../decisions/` (transactional outbox).

## Testing the bus

- **Idempotency (the core):** apply the same logical event twice → the effect lands once.
- **Reclaim:** an unacked (crashed) orphan is re-delivered; a recently-claimed (live) event
  is *not* stolen. (Redis-gated, self-skips without a live Redis.)
- **Outbox atomicity:** state + outbox row commit together (commit → both present;
  rollback → neither).
- **Outbox at-least-once:** leave a row claimable across a simulated crash, re-run the
  relay, assert it re-emits and the consumer still applies the effect exactly once.
- **Redis down:** the relay leaves rows `pending`; when Redis returns, it drains in order.
