# Auth & Security

The through-line is a recurring philosophy: **fail closed, key on verified identity, and gate the build on the
bugs scanners can't find.**

## The token model

- **JWT, HS256, ~24h**, issued by the **Auth** service. The `sub` is the user's username.
- **Validated at the gateway, not per request.** A small **`jwt-validator`** service does a
  *local* HS256 check (no DB, no gRPC, no call back to Auth). Traefik calls it via
  `forwardAuth` and, on success, copies `X-User-Id` / `X-Role` onto the backend request.
  This is deliberate: if the gateway validated by calling Auth on every request, the
  single-replica Auth service would become the platform's chokepoint. A local check keeps
  auth off the hot path.
- **Services trust the gateway-stamped headers** — *because* of the gate below. A service
  also keeps a Bearer→token-check fallback for dev/outage, but the primary path is the
  stamped header.

### The spoof gate (the subtle one)

`X-User-Id` and `X-Role` are, by themselves, **client-forgable** — a client can send
`X-Role: admin`. The defense is ordered, not trusting any single layer:

1. **Strip first.** A gateway middleware **blanks any client-supplied identity header
   before** `forwardAuth` runs, so the only `X-User-Id`/`X-Role` a backend ever sees are
   the ones the validator set from a verified token.
2. **Then stamp.** The validator writes the real values.
3. **Network policy** admits each service's ingress **only from the gateway** — a direct
   dial that skips the gateway can't reach the app surface at all.
4. **Defense in depth:** admin routes re-check `X-Role: admin` in the handler (401 no
   identity, 403 non-admin, *before* any DB access), so even a misrouted request that
   somehow bypassed the gateway is denied.

The same discipline is applied to **`X-Forwarded-For`** for the per-IP rate limiter: the
edge **strips a client-supplied XFF and stamps the real client IP** at the front hop, and
the gateway trusts XFF only from that hop (`trustedIPs` + `trustForwardHeader`). Why this
matters: a rate limiter that keys on the *leftmost* XFF is a no-op if a client can set
XFF — one fresh bucket per request. Stripping at the edge is what makes per-IP limiting
un-spoofable. (This was a real, found-and-fixed gap: the limiter was originally keyed on a
spoofable header.)

## The LLM cost/DoS chain — found, then closed

The highest-leverage risk on the platform was **not** an injection. It was that the *only*
brake on the expensive LLM path was a rate limiter that (a) was keyed on a spoofable
header, (b) failed *open* when Redis hiccuped, and (c) sat behind an auth check that
failed *open* if its token client couldn't connect at boot. Any one of those turned the
"async Q&A" buffer into a **resource/cost DoS** — and, the moment a paid provider is
pointed at, a **money DoS**.

Each link was closed, and each has a test:

- **Re-key the limiter on the validated identity**, not a header (identity only
  unauthenticated as a fallback) → the limit is un-spoofable.
- **Add a `max_tokens` cap** to every LLM request, plus a **per-user daily budget** and a
  **global fleet ceiling** — three additive caps with safe defaults, each with a distinct
  429. A crafted "write a 30-page essay" question now costs a *bounded* number of tokens.
- **Fail closed.** If Redis is down, the rate limiter falls back to a per-pod in-memory
  limiter rather than allowing everything. If the auth client is unreachable, the service
  **refuses to serve** (or 503s the affected path) instead of letting everyone through.
- **Queue backpressure:** an admission 503 under a bounded in-flight queue instead of
  an unbounded one that OOM-crashloops the (KEDA-scaled-up) pods at peak.
- **Requeue, don't dead-letter, transient LLM failures** (429/5xx/timeout) with a bounded
  retry budget that counts *logical* passes, not broker deliveries.

**Prompt injection** is guarded by an **always-on system prompt** with **retrieved RAG content fenced as data**
(so untrusted text can't steer the instructions), and the blocklist demoted to log-only. **RAG poisoning** (a
poisoned chunk skewing answers for everyone) is addressed by authenticating the vector DB
and default-deny **NetworkPolicies** so only the owning services can write the corpus.

Two historically-feared bugs were **checked and are not present**: SQL injection (every
query is a parameterized `sqlx` `.bind()`) and XSS (the SPA renders the LLM answer as a
plain escaped text node). Committed dev/placeholder secrets are **refused at boot**
(crashloop loudly rather than run with a known-weak key).

## The security test strategy (the part to look at)

CI has a **blocking `security` stage**. It is *blocking* — a new finding fails the
pipeline and gates deploy. The posture across the board is a **ratchet**: baseline the
existing findings, then **fail only on what's new**, so the gate is green *and* actually
tightens over time instead of either passing vacuously or blocking forever on legacy
debt.

The layers, in the order they landed (all merged):

1. **SAST / secrets / dependencies** — `gitleaks` (secrets), `cargo-audit` + `cargo-deny`
   (Rust deps), `clippy`, **Semgrep** (Rust), and `eslint` + `eslint-plugin-security`
   (frontend).
2. **SCA + misconfig** — **Trivy**: lockfile vulnerability scan (`Cargo.lock`,
   `package-lock.json`) and Kubernetes **misconfiguration** scan (`infra/`), each under a
   version-pinned, SARIF-emitting setup and a ratchet baseline.
3. **DAST** — a hand-authored **OpenAPI spec** (the machine-readable counterpart to the
   API doc) turns **OWASP ZAP** from a blind crawl into a *targeted, authenticated* API
   scan (a fresh token armed at scan start). Scoped, cost-aware (the expensive LLM
   endpoint is dropped by default — it'd burn the shared LLM), run on-demand against a
   non-prod target.
4. **The gate scanners can't find** — this is the one I'd lead with:
   - **A two-identity IDOR/BOLA regression suite, as a hard blocking gate.** A scanner
     won't find "user A can read user B's record." This suite logs in as two identities,
     drives every owner-scoped endpoint with the *wrong* identity, and asserts **404/403
     with no data leaked**. It's a *regression guard* — the scoping is already in the
     code (`WHERE id = $1 AND user_id = $2`); the suite *locks it in* so a future change
     that drops a scope fails CI.
   - **In-process LLM cost-cap tests** — the rate-limit 429 cliff and the global
     fail-closed cap, run with a **dead LLM endpoint** so they need *zero* real LLM calls
     (a live load test would burn the self-hosted LLM).
   - **An admin-role ZAP pass** — the non-admin baseline is re-run *as admin* to check the
     admin surface with the privileges it actually has, then the promotion is revoked.

### Why this framing

Most "we scan in CI" stories stop at Trivy/Semgrep — which is correct but table-stakes and
finds the same things everyone's scanner finds. The differentiator is **the bugs the
scanners can't see** (authz-by-identity, cost/DoS ceilings), encoded as *regression
gates* that fail the build. That's what turns a security scan into a security *policy*
the code is continuously checked against.

### Availability (the honest part)

This is *currently* a single self-hosted node, so there is **no multi-replica HA** for Postgres,
Qdrant, or Redis, and no PodDisruptionBudget that would mean anything on one node.
Availability is bought with process-level resilience instead: well-designed probes,
self-healing consumer groups, fail-open retrieval paths, fail-closed auth/rate-limit,
and **nightly off-box backups with restore tests** (see `../ops/restore-runbook-sample.md`).
