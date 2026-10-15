# 0001 — Strangler-fig migration

**Status:** Accepted (complete — the monolith is retired)

## Context

The system started as a single FastAPI application (~1,400 lines at the start of the
migration, grown since). As it grew, one feature — the AI assistant — became the compute
hot spot (an LLM call per question), and the monolith became both the bottleneck and the
risk: any change to a hot path meant touching and redeploying the whole app, with no way
to scale one feature independently.

The alternatives were: (a) keep the monolith and optimize, (b) a **big-bang** rewrite
(build the whole new system, cut over in one go), or (c) **strangler-fig** — grow the new
system around the old one, strangling traffic over incrementally.

## Decision

**Strangler-fig.** Migrate capability to a new Rust service, put it behind the API
gateway, cut that traffic over, and leave the monolith serving everything else — repeat
until the monolith has no traffic left.

The gateway is what makes this possible: because routing is by path prefix, a newly built
service can own `/api/<area>` while the old app keeps owning the rest, and the old app
stays the fallback for anything not yet cut over. Each service went live **independently**,
one at a time.

## Consequences

- **Every step was shippable and reversible.** If a new service misbehaved, its route
  flipped back to the monolith. There was never a moment where "the rewrite" was a single
  all-or-nothing bet.
- **Independent rollout + scaling from day one.** The assistant could become async and
  autoscale without the rest of the app knowing or caring.
- **Cost: a long coexistence window.** For a while the gateway, the monolith, and N new
  services all ran, and some domain logic was temporarily reachable in two places. That
  duplication was deliberate and time-boxed — the exit criteria were explicit (monolith has
  no traffic) rather than "it feels done."
- **Cost: the gateway is now a hard dependency.** It did the token validation, rate
  limiting, and identity stamping for everything. That's now a first-class component with
  its own tests and probes (see `../architecture/auth-and-security.md`), but it is a real
  concentration of cross-cutting logic in one place.
