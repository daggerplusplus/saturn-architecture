# Sample change-validation record — cap per-request LLM cost + submission budgets

> **What you're looking at.** One real record from this project, **lightly redacted**: the
> business domain is genericized, host names / registry / exact commit IDs are
> anonymized, and file-line references are dropped. The *structure and discipline* are
> exactly as they run. It's a single instance of the practice described in
> [`process.md`](./process.md) — here, all four things in one place: **the ask**,
> **the decisions I made** (where the agent diverged and why), **the tests** (including a
> test proven to fail before the fix), and **what I deliberately did *not* do**.
>
> The original folder was `a1c9f0e-assistant-llm-cost-caps/` (folder ID = the code-commit
> hash, so the record is `git show`-able). It held three files: the change request, this
> validation record, and the scoped `.patch`.

---

## Part 1 — The ask (the change request, condensed)

**Origin:** security audit — the "amplifier" half of an LLM cost/DoS chain. The rate
limiter was already keyed on the validated identity (a companion change, already shipped),
but *each allowed request* could still spend an unbounded number of tokens, and the fleet
had no ceiling.

**Symptom (two attacker-influenced cost amplifiers):**
- **Per-request.** A valid question like *"write a 30-page essay on X"* hit the LLM with
  **no output-length cap** — one request could burn an arbitrarily large share of the
  model's capacity. The 120s HTTP timeout bounds wall-clock, not tokens.
- **Fleet-wide.** The only limit was `10/60s` **per identity**. N users → up to `10·N`
  submissions/min with **no global ceiling** and **no per-user daily ceiling**, so a slow,
  distributed burn is unbounded.

**The requested change (three additive caps, each with a safe default):**
1. **`max_tokens` on every LLM request** (the must-land part) — a new config value, default
   2048.
2. **Per-user daily budget** — a second, much larger fixed-window counter; exceed → `429`.
3. **Global fleet ceiling** — a single shared counter; exceed → `429`.

**Must NOT change:** the existing per-window limiter, the poll-path exemption, the RAG
fail-open behavior, and the job lifecycle/queue.

**Validation plan (written before the code):** add a pure unit test that the serialized
request carries `max_tokens`; an integration test that the (N+1)th submission over the
daily budget is `429` (a 202 where a 429 is expected — **fails before the fix**); a
global-cap-across-users test; a config-default test. All existing tests must still pass.

---

## Part 2 — The change + the decisions

### TL;DR
Three additive caps on the LLM cost path; `max_tokens` is the must-land part, the two
budgets are defense-in-depth. Every new knob is optional and defaulted, so existing
deploys are unaffected on first deploy.

### Where I diverged from the request (and why)
These are the "owner decision" lines — the record exists so they're legible later.

- **Budget helper lives in `rate_limit.rs`, not `redis_client.rs`.** The request suggested
  adding it to the low-level store. I put it in the *policy* module — where the
  handler-facing limit helpers already live and where it can **reuse** the existing
  fixed-window Lua rather than duplicate the script. The store stays low-level; the policy
  layer stays the policy layer. Net effect identical.
- **The two distinct `429` messages** required two new error variants in the **shared**
  crate (`DailyBudgetExceeded`, `GlobalBudgetExceeded`). That's a small additive change to
  a second crate, so the code commit touches both — called out explicitly rather than
  surprising a reviewer.
- **`max_tokens` field name** — I used `max_tokens` (the OpenAI-compatible default). The
  request flagged confirming `max_tokens` vs `max_completion_tokens` against the live
  provider; I recorded that as a **deploy-time verification item**, not a code blocker,
  because the code is wire-correct for OpenAI-compatible endpoints either way.
- **Extracted `build_llm_request`** as a pure, I/O-free function shared by both the
  streaming and non-streaming call sites. That both de-duplicated the identical
  message-building block **and** made the `max_tokens`-carrying request shape unit-testable
  without a network call (which is what the pure unit test in the plan needed).

### Files changed (10)
| Area | Change |
|---|---|
| `llm.rs` | `LlmRequest` gains `max_tokens`; new pure `build_llm_request`; both call sites use it; 2 unit tests |
| `config.rs` | new fields + env reads (all defaulted) |
| `rate_limit.rs` | `check_budget` (fail-open) + daily + global budget checks |
| `handlers.rs` | both checks wired after the per-window limit; test fixtures updated |
| shared `error.rs` | two new variants + 2 mapping arms + 2 tests |
| `embed.rs`, `qdrant.rs` | config-literal field additions only (no behavior) |
| integration tests | fixtures + 3 new tests |
| `Cargo.toml` / `Cargo.lock` | a `temp-env` dev-dep (for the env-isolated default test) |

### Key diff (the must-land part)
```rust
#[derive(Serialize)]
struct LlmRequest {
    model: String,
    messages: Vec<ChatMessage>,
    stream: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    max_tokens: Option<u32>,   // always bounds completion length
}
```
Wiring (in the submit handler, **after** the per-window limit, **before** enqueue):
```rust
rate_limit::check_rate_limit(&headers, &state, user_id.as_deref()).await?;
rate_limit::check_daily_budget(&headers, &state, user_id.as_deref()).await?;  // 429 daily
rate_limit::check_global_budget(&state).await?;                               // 429 global
```

### Must-NOT-change compliance
Per-window limiter and its identity: untouched, still runs first. Poll-path exemption:
untouched. RAG fail-open: untouched (the `embed`/`qdrant` changes are config-literal
field additions only). Job lifecycle/queue: untouched — both new checks run *before*
save/enqueue.

---

## Part 3 — The tests (including the fail-pre-fix proof)

**Added:** `llm_request_includes_max_tokens`, `llm_request_omits_max_tokens_when_none`
(pure); `submission_blocked_over_daily_budget`, `global_cap_enforced_across_users`,
`max_tokens_default_when_unset` (integration, against the real handler + in-memory limiter,
each cap tripped in isolation with the other two set high so they never interfere); and
two shared-error tests. The two budget tests additionally assert the **distinct 429
message** in the body.

**Fail-pre-fix proof** — this is the load-bearing one. I temporarily disabled the two
budget checks and re-ran the two budget tests; they **fail** with the exact expected
assertion, proving the checks (not something else) produce the cap:

```
running 2 tests
test submission_blocked_over_daily_budget ... FAILED
test global_cap_enforced_across_users ... FAILED

---- submission_blocked_over_daily_budget stdout ----
panicked: the (N+1)th submission must be throttled by the daily budget
  left: 202
  right: 429

---- global_cap_enforced_across_users stdout ----
panicked: the (N+1)th fleet-wide submission must be throttled by the global cap
  left: 202
  right: 429

test result: FAILED. 0 passed; 2 failed
```

The two lines were then restored verbatim (confirmed via diff — both calls present, no
temp markers) and re-run green.

**Full workspace suite:** `184 passed / 0 failed / 5 ignored` (serial) — the prior baseline
plus the 9 new executions, no regressions. Build clean: only **pre-existing** warnings,
none from the new code.

---

## Part 4 — What I deliberately did *not* do

This is the part that's easy to skip and important to state:

- **Did NOT bundle an unrelated flaky-test fix.** During verification I hit a pre-existing
  flaky middleware network test (a real-TCP connection failing under parallel load because
  the test server gives the listener a fixed 10 ms readiness delay). It is **not this
  change** — I never touched that file, and the panic is a connection error, not a status
  assertion; it passes on retry and passes serially. Rather than slip a fix into a
  security change, I **called it out and deferred it to its own change** (commit-first: no
  unrelated changes in this code commit). It got its own record later.
- **Did NOT run the live budget checks at deploy time** — the low-`DAILY_MAX` /
  `GLOBAL_MAX` live checks were **skipped as disproportionate**: the integration tests
  already drive both through the real handler with the distinct-429 assertions, and a live
  check would need a temporary config override + re-rollout on a dev cluster. I recorded
  the skip and the reasoning rather than performing theater.
- **Did NOT change the wire field name speculatively.** `max_tokens` vs
  `max_completion_tokens` was left as a deploy-time confirmation (see below) instead of a
  guess baked in as "certain."

---

## Commit
- **Code:** `a1c9f0e` — "Add LLM cost caps: max_tokens, per-user daily budget, global
  fleet ceiling" (10 files). This code-commit hash is the folder ID.
- **Record:** this folder (change request + validation record + scoped patch) is committed
  separately as the documentation commit. **I commit; the agent does not.**

---

## Outcome + deploy-time verification

**Local outcome: PASS.** `max_tokens` bounds every completion's output length; the
per-user daily budget and global fleet ceiling cap the rate of accepted questions; both
return a distinct 429. No regressions, no new warnings. The per-window limiter, poll
exemption, RAG fail-open, and job lifecycle are untouched.

**Deploy-time (closed the deferred items), on the cluster:**
1. **`max_tokens` field name — CONFIRMED.** A direct in-cluster probe of the self-hosted
   LLM with `max_tokens: 16` returned `200`, `finish_reason: "length"`, and
   `usage.completion_tokens: 16` — the cap honored to the exact token. The provider is
   **vLLM**, which uses the OpenAI `max_tokens` key; **no switch was required.**
2. **A reasoning-model finding (affects interpretation, not the cap).** The model is a
   *reasoning* model: completions carry a separate `reasoning` field, and `max_tokens`
   bounds the **total** new tokens (reasoning + visible content). So a low budget can
   return a short or empty *visible* answer while the cost is still bounded — the
   intended cost-cap behavior, not a defect.
3. **Worst case — bounded.** A maximally-verbose prompt at the default cap consumed the
   entire budget on reasoning and completed with a bounded (empty) visible answer at a
   bounded cost — exactly the desired DoS behavior for an adversarial prompt.
4. **Normal usage — the default is appropriate.** An ordinary question completed in ~14s
   with a correct answer, so the 2048 default is fine for real usage.

**Verdict: PASS, SHIPPED.** Cap works end-to-end, the field name was confirmed against the
live provider, the default is appropriate, and no config/manifest change was required.
*The full scoped diff ships as `<date>-<slug>.patch` in the folder.*
