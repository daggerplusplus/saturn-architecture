# 0004 — Async AI endpoint (submit → poll)

**Status:** Accepted

## Context

The assistant endpoint is the only path with a per-request LLM call — tens of seconds of
model time. The question is how the client faces that latency: a long-lived **sync**
response (the connection holds the LLM call open, optionally streamed) or an **async**
job (submit, get a job id, poll for the result).

## Decision

**Async: `POST /api/questions` returns `202 Accepted` + a `job_id` immediately; the client
polls `GET /api/questions/{job_id}/status`.** A KEDA-scaled worker pool claims jobs from a
Redis Stream queue and does the real work (embed → retrieve → LLM → persist → emit event).

## Why not a long-lived sync/streamed response

- **Scale on work, not connections.** A sync path ties one request handler (and one
  connection) to an LLM call; scaling it means scaling for *concurrent open connections*.
  A job queue decouples submission from processing: the API layer is cheap, and the
  *worker pool* scales on **queue depth** (see [0005](./0005-queue-depth-autoscaling.md)).
- **Resilience.** If a worker pod dies mid-job, the queue re-delivers it (bounded retries,
  then DLQ). A sync request that dies mid-flight is just a failed/timed-out request with
  the work lost.
- **Backpressure + cost caps have a place to live.** Admission (503 under a bounded
  in-flight queue), per-user/global cost caps, and retry budgets all attach naturally to a
  *queued* job. On a sync path the only lever is "drop the connection."

## Consequences

- **The client is slightly worse off.** Polling means the client manages a job lifecycle
  and can't get a streaming token-by-token answer. That's an accepted product trade for a
  backend that must stay up and scale under load. (The internal worker *does* stream from
  the LLM; the client-facing contract is the poll.)
- **Redis is on this critical path.** The job queue lives in Redis, so the same
  single-stateful-dependency tradeoff as the event bus applies (accepted, one node).
- **Polling must stay cheap.** `GET .../status` is deliberately exempt from the
  per-submission rate limit, and identity resolution on the hot poll path uses the
  gateway-stamped header (no per-poll call to the auth service) — so polling a running job
  doesn't itself become a chokepoint.
