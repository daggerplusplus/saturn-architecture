# 0002 — Schema-per-service (one Postgres, several schemas)

**Status:** Accepted

## Context

The services each need their own durable state, but the deployment is a **single
self-hosted node** — running one Postgres *instance per service* (the textbook
"database per service") would mean several Postgres processes, several replication
topologies, several connection pools, on one box. That's a lot of moving parts for a
single node. At the same time, the services are real microservices: a compromised or
buggy service must not be able to read or write another service's tables.

## Decision

**One Postgres instance, one schema per service, and a least-priv role per service.**
Each service connects **as its own role**, granted only on its own schema (DML + sequence
usage; `CREATE` revoked). The migration Job is the single component that holds the
superuser, because DDL must cross schemas.

- A service's role can **only** see its own schema — cross-schema reads/writes and
  `CREATE` are denied (verified on the cluster: own-schema read OK, cross-schema denied).
- Schemas are the isolation boundary; the **service boundary** (separate process, separate
  deploy, separate scale) is what makes them microservices, not just "namespaces."

## Consequences

- **Isolation without N databases.** Least-priv blast-radius control (a compromised
  service touches only its own tables) without the ops cost of many Postgres instances on
  one node.
- **Cheap atomic cross-service *writes within the DB* — and that's a constraint too.**
  Because it's one database, a shared table *could* sit in one transaction across schemas
  (this is what makes the outbox in [0006](./0006-transactional-outbox.md) cheap). We
  *chose* not to lean on that for service-to-service coupling — services still talk via the
  event bus — because a shared transaction is a shared failure domain and a coupling the
  codebase otherwise avoids.
- **Cost: it's not "database per service."** Two services sharing a Postgres process share
  its I/O and its fate; a Postgres OOM or long lock can affect all schemas. On a single
  node that was an acceptable trade for the ops simplicity, and it's a known, stated
  limitation rather than an oversight.
- **Cost: migration ordering.** The migrate Job runs all schemas' DDL; a schema's migration
  must not depend on un-applied migrations in another. Kept schemas self-contained by
  convention.
