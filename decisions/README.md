# Architecture Decisions

Short ADRs for the load-bearing choices. Each is deliberately brief: **context →
decision → consequences** (including the costs, not just the wins). These record *why*
the system has the shape it does, so a future change has the original tradeoff in front
of it.

| #                                         | Decision                           | One line                                                                                                               |
| ----------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [0001](./0001-strangler-fig-migration.md) | **Strangler-fig migration**        | Move the monolith to services one capability at a time, behind the gateway, with the old app as fallback at each step. |
| [0002](./0002-schema-per-service.md)      | **Schema-per-service**             | One Postgres schema (and least-priv role) per service in a single instance — isolation without N databases.            |
| [0003](./0003-redis-streams-event-bus.md) | **Redis Streams event bus**        | At-least-once delivery + idempotent consumers, over the Redis we already run — not a new broker.                       |
| [0004](./0004-async-ai-endpoint.md)       | **Async AI endpoint (202 + poll)** | Submit → poll for the LLM path so it scales on queue depth, not open connections.                                      |
| [0005](./0005-queue-depth-autoscaling.md) | **Queue-depth autoscaling (KEDA)** | Scale the worker pool on *work in the queue*, not CPU.                                                                 |
| [0006](./0006-transactional-outbox.md)    | **Durable transactional outbox**   | Make state + intent-to-emit atomic in Postgres; relay at-least-once to Redis. (Designed; staged.)                      |

Conventions: numbers are chronological; **Status** is one of *Accepted / Superseded /
Deferred*. ADRs that are **Deferred** are kept because the *decision and its tradeoffs*
are the artifact — the build can happen later.
