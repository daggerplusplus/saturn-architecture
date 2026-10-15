# 0005 — Queue-depth autoscaling (KEDA)

**Status:** Accepted

## Context

The assistant worker pool and the ingestion worker each have a queue of pending work. When
should the pool scale out? The default instinct is to scale on **CPU** — but for an async
worker that is the wrong signal: a worker sitting on a queue is often *not* CPU-bound
(it's waiting on an LLM call or an I/O hop), so CPU can be low while a large backlog sits
unprocessed.

## Decision

**Scale on the amount of work in the queue, not CPU.** KEDA drives the pools from the
queue's pending depth: the assistant pool scales **1→5** on the assistant job-queue lag,
and the ingestion worker **1→8** on its pending-queue gauge.

- The **assistant** uses a Redis-Streams scaler (queue length is the metric).
- The **ingestion** worker uses a Prometheus gauge of its durable Postgres queue's pending
  rows (it claims with `FOR UPDATE SKIP LOCKED`, so the queue is the Postgres table).

## Why not CPU

- **CPU mis-scales an I/O-bound worker.** A pool can be 20% CPU and still have 200 jobs
  waiting — CPU autoscaling would see "healthy" and not scale, so latency balloons. Queue
  depth is the *actual* user-facing signal: it's the waiting work.
- **It scales down honestly too.** When the queue drains, depth → 0 and the pool scales
  back to its floor, so an idle system holds the minimum (1) rather than parking hot pods.

## Consequences

- **The right knob for an async pool.** Latency tracks backlog, and the expensive path
  (LLM) gets more workers exactly when there's more queued work to feed it.
- **Cost: queue-depth is only as good as the queue.** The scaler reads *the queue* — so
  the queue must be the true source of pending work (which is why the ingestion queue is the
  durable Postgres table, not an in-memory map). A queue that lies (e.g. jobs stuck in a
  non-claimable state) mis-drives scaling; the stuck-job cleanup and the requeue/reclaim
  logic exist partly to keep the queue honest.
- **Cost: a scaler dependency.** The Redis scaler needs Redis reachable; the Prometheus
  scaler needs the gauge exported and scraped. Both are already in the architecture, so
  this added no new stateful dependency — a reason it fit.
- **Bounded, not unbounded.** Combined with the admission 503 and cost caps
  (`../architecture/auth-and-security.md`), scaling *up* on a flood spawns more workers to
  drain it rather than letting an unbounded queue OOM a small pool — the scaling is part
  of the backpressure story, not a substitute for it.
