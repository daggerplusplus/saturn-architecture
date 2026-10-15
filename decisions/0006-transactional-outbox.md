# 0006 — Durable transactional outbox

**Status:** Deferred (designed; build staged, event-producing services first)

## Context

The [event bus](../architecture/event-bus.md) is **at-least-once with no distributed
transaction**, and producers today use the **best-effort emit** shape:

```
1. write the state change to Postgres  →  COMMIT
2. XADD the event to the stream         (best-effort)
```

The sharp edge: each event is emitted **only on its fresh write**, and the idempotent
guards that make the *write* safe to retry are exactly what make a **re-run a no-op that
does not re-emit**. So if the `XADD` is dropped (step 2 fails, or the process dies between
the commit and the `XADD`), the downstream effect is **permanently missed** — healable only
by a manual stream replay. This was a real finding: the code comments claimed a dropped
emit was recoverable, and it was **not**. The comments were corrected (the honest, small
fix); this ADR is the plan for the *real* fix.

## Decision

Introduce a **durable transactional outbox** so the *durable* halves of steps 1 and 2
commit atomically:

```
BEGIN → write state → INSERT outbox row (same transaction) → COMMIT
(a separate relay reads committed rows and XADDs them, at-least-once)
```

- **No distributed transaction is created** — the invariant is preserved. The *Postgres*
  side (state + intent-to-emit) becomes atomic; the *Postgres→Redis* leg stays the same
  at-least-once, idempotent-consumer contract.
- **One outbox table per producing schema** (least-priv: each service writes only its own
  schema, in its own transaction — no new cross-schema grants), with an identical shape:
  monotonic `id` (ordering), `event`, `idempotency_key` (UNIQUE — belt-and-braces dedup),
  `payload JSONB`, `status` (`pending|sent|dead`), `attempts`.
- **A shared outbox crate** writes the relay/reaper/claim/`insert_in_tx`/DDL **once**,
  parameterized by the table name, and each service instantiates it against its own table.
  This absorbs the only real cost of per-service tables (duplicated relay code) — "shared
  code, per-service data," the same split the rest of the system uses.
- **The relay** claims with `… WHERE status='pending' ORDER BY id LIMIT n FOR UPDATE SKIP
  LOCKED` (same claim pattern the ingestion job queue already uses), `XADD`s, and marks a row
  `sent` only after a successful `XADD` — so a crash re-queues and re-emits, and the
  consumer's existing `idempotency_key` dedup absorbs the double. Poison rows go `dead` →
  DLQ; `sent` rows are purged on a retention schedule.

### Why per-service table and not one shared table

It's **one database with several schemas**, so a shared table *could* sit in the same
transaction as any service's write (the "can't write two databases atomically" objection
doesn't apply). That makes it a real tradeoff:

- **Per-service (chosen):** preserves least-priv, isolates failure/backlog domains per
  service, rolls out incrementally, and matches the existing per-service queue pattern.
- **Single shared table:** one relay/gauge, one global ordering — but breaks least-priv
  (N services need `INSERT` on a shared schema) and creates a shared failure domain.

The topology removes per-service's *structural* objection but not the shared table's
*lasting* cost (least-priv + shared failure domain), so per-service wins.

## Consequences

- **Closes the missed-emit gap honestly.** A dropped emit is now a *backlogged* outbox row
  the relay drains, not a lost effect.
- **Cost: a new long-running task per service** (the relay) — reused from the ingestion
  worker's stop-flag/shutdown hygiene so it exits cleanly.
- **Cost: a table that grows while the relay is down** — mitigated by the retention purge
  (the `pending` backlog is the point; `sent` rows are just history).
- **The cutover is cleanest as a full hand-off per service:** when a service's emits are
  wired through the outbox, its direct `XADD` is *removed* (not dual-emitted), so the
  outbox fully owns emission for that service.
- **Deferred, deliberately.** The design and the tradeoffs are the artifact; the build is
  staged for when it's wanted (event-producing services first). Recording the decision now
  means it isn't made on the fly, and the corrected comments point at a concrete plan.
