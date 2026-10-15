# 0003 — Redis Streams as the event bus

**Status:** Accepted

## Context

Services produce cross-service effects (content ingested → index it; an item completed → update the user's progress;
a message created → persist the assistant's answer). The
options for carrying those: (a) synchronous HTTP fan-out from the producer, (b) a
dedicated broker (Kafka/NATS), or (c) **Redis Streams** over the Redis instance we already
run for cache/rate-limiting.

The honest framing of the requirement: this is a **single-node, low-to-moderate-volume**
deployment. The bar is *correct* (no lost effects, no double-effects) and *simple to
operate*, not *horizontally-scalable throughput*.

## Decision

**Redis Streams with consumer groups**, and **idempotent consumers** as the correctness
mechanism. The contract is stated explicitly: **at-least-once delivery, no distributed
transaction** — so correctness comes from consumers applying each `idempotency_key`
exactly once *effectually*, not from the broker.

## Why not the alternatives

- **Sync HTTP fan-out** couples producers to every consumer's availability and turns one
  state change into a tree of synchronous calls that must all succeed. It also gives no
  retry/DLQ story for free.
- **Kafka/NATS** would be the "bigger" answer and is the right one at scale, but on one
  node it's another stateful system to run, back up, and scale — for a volume that doesn't
  need it. **The bar is correctness and operability, not throughput; Redis Streams meets it
  with a system already in the architecture.**

## Consequences

- **Simple + durable enough.** Consumer groups give per-consumer cursors; entries persist
  (Redis runs AOF + noeviction); a `…dlq` stream catches poison entries past a retry budget.
- **Idempotency is mandatory, not optional.** Every consumer uses idempotent guards
  (`ON CONFLICT DO NOTHING`, result flags), and every event carries an
  `idempotency_key`. A redelivery is a no-op. This is the whole correctness story and it's
  tested (see `../architecture/event-bus.md`).
- **Self-healing consumers.** A lazy `XCLAIM` reclaim recovers an entry a dead pod left in
  its pending list — needed because a fresh-only `>` read would otherwise strand it.
- **Cost: one big stateful dependency.** Redis now carries cache + rate-limit + the job
  queue + the event bus. That's a concentration; it's mitigated by AOF/noeviction, a
  fail-closed rate limiter, and off-box backup, and it's accepted *because* it's one node
  where adding a broker was worse than consolidating. The remaining durability gap at the
  emit edge is addressed by the [outbox](./0006-transactional-outbox.md).
- **Cost: at-least-once means consumers *must* be idempotent.** This is a standing
  requirement on any new consumer; it's the price of a simple, no-transaction bus.
