# How I work with an agentic coder

Saturn was built with **Claude Code** as the implementer and me as the owner/reviewer.
This page describes the loop and the *paper trail* it produces, because the part
employers worry about with agentic coding — *"who checks the AI?"* — is answered by a
convention, not a claim.

**The short version:** every code change ships with a **validation record** that says what
was asked, what I decided, what the agent did, what tests ran (including a test that was
proven to *fail before* the fix), what was explicitly *not* done, and whether it was
verified live and shipped. One folder per change, for every change. **~70 of them** exist
for this project; [`sample-validation-record.md`](./sample-validation-record.md) is one,
sanitized.

## The loop

1. **I specify; the agent implements.** I write or steer the change request. The agent
   does the code, the tests, and the first pass of verification.
2. **Decisions are mine, and I record the non-obvious ones.** Where the agent's approach
   diverged from the request — and especially where it was *right to* diverge — the record
   says so with the reasoning. The record is where "owner decision" is legible, not buried
   in a commit message.
3. **Fail-pre-fix tests.** A new behavior comes with a test that is **proven to fail
   before the change and pass after.** The record shows the failure (the exact assertion,
   a 202 where a 429 is expected, etc.) and then the green re-run. This is the difference
   between "I added a test" and "this test is what makes the behavior true."
4. **Verify live, not just in the sandbox.** Unit/integration tests aren't the finish
   line. Changes that touch the running system get **end-to-end verification on the
   cluster** (deploy, drive the real path, assert the real outcome), and the record notes
   when a piece is *deferred* to deploy-time and then closes it out.
5. **Status is tracked to the end.** Records carry an explicit lifecycle — local green →
   verified on a QA/staging cluster → **SHIPPED to prod**. "Works on my machine" and
   "shipped and verified in production" are different statuses and are labeled as such.
6. **Commit-first, scoped commits.** The code commit contains *only* that change. An
   unrelated fix found along the way is *not* bundled into it — it's called out and
   deferred to its own record. There are records in the index that are *only* a one-line
   deploy fix, because it was its own change.
7. **Commit only when asked.** The agent does not commit on its own; I commit. The
   validation record is typically a separate, later documentation commit.

## What I review vs. what the agent does

| | Agent | Me (owner) |
|---|---|---|
| Implements the code + tests | ✓ | |
| Runs the suites, reports output | ✓ | |
| Proposes a design when the spec is ambiguous | ✓ | |
| **Decides the design when it's a real choice** | | ✓ |
| **Decides scope** (what's in, what's deferred/its own change) | | ✓ |
| **Confirms the fail-pre-fix proof is real** | | ✓ |
| **Runs / signs off the live E2E** | runs it | confirms the outcome |
| **Commits and ships** | | ✓ |

The agent does a large share of the *work*; I own the *judgment* — design, scope,
verification sign-off, and the commit. The record is the artifact that makes that split
visible after the fact, which is exactly the part that's hard to trust when you only see
the final code.

## How I steer the agent

The **change request is the steering instrument.** Before any code, it fixes the scope:
the symptom and root cause, the proposed fix, an explicit **must-NOT-change** list, and
the test plan. The agent works to that spec rather than to my vague intent — that's most
of how it stays in a lane.

A small set of **standing conventions** apply to every change without being re-stated
(always ship a test with a behavior change, keep commits scoped to one change, fail-pre-fix
new behavior), so the request can stay short. And planning is kept separate from
execution: on an ambiguous spec the agent **proposes** a design and I **decide** — it
doesn't silently pick between real alternatives (that split is the table above).

What it **can't** do is a hard boundary, not an oversight:

- **It doesn't commit** — I always do.
- **It doesn't reach prod, and its cluster access is non-prod only.** Live E2E runs
  against a dev/QA/staging target; the prod deployment and its sign-off are mine.
- **It doesn't touch live secrets.** Credentials live outside the repo, and the build
  **refuses to boot on a committed placeholder secret** — so the code it edits never
  carries one, and it has no reason to reach for one.

## When the agent is wrong

The loop above is only meaningful if the agent can be wrong and I catch it. It can —
these are real cases from the records (genericized, the same way the sample is):

- **An approach I rejected.** The agent put a new helper in the low-level storage layer,
  which would have *duplicated* a fixed-window script that already existed. I moved it to
  the policy layer, next to the other limit helpers, so it could *reuse* that script.
  Same diff, wrong layering — and the record's divergence note says who made the call and
  why.
- **A "done" the test killed.** The agent reported a cost cap as implemented, but the
  fail-pre-fix run came back **202 where a 429 was expected** — the guard had been wired
  after the request was already accepted, so it wasn't on the path at all. Moving it before
  enqueue made the test green. The red run is in the record; that's what proves the cap,
  and not something else, is what enforces it.
- **A scope item I punted.** Mid-verification I hit a pre-existing flaky test (a real-TCP
  listener that drops under parallel load behind a fixed readiness delay). Unrelated to
  the change, so I didn't slip it in — I called it out and deferred it to its own record.
  Commit-first is what keeps that clean: the change stayed scoped.

## Why this is worth showing

A diff proves what changed. It does *not* show: what was asked, what was considered and
rejected, which tests were proven load-bearing, what was deliberately left out, or whether
it was verified in the environment that matters. For agentic work, the record is how I keep
a human decision trail: what was asked, what I decided, and what evidence backed the change.
The sanitized sample in this folder is a single instance of that; the full practice (the ID
scheme, the lifecycle statuses, the fail-pre-fix proofs, the deferred-then-closed items)
is the point.

## Reference: the folder layout and ID scheme

### One folder per change

```
change-validations/<id>-<slug>/
    CHANGE-REQUEST-*.md      ← the originating ask (the spec, before code)
    <date>-<slug>.md         ← the validation record (the narrative + test evidence)
    <date>-<slug>.patch      ← the scoped diff for that change
```

Three separate documents, on purpose:

- **The change request** is written *before* implementation — the symptom, the root cause,
  the proposed remediation, **what must NOT change**, and the validation plan (which tests
  to add, which must still pass, and what to verify live). Writing it first is what keeps
  a change scoped: "must not change" and the test plan exist before the code does.
- **The validation record** is the after-the-fact narrative: the design decisions (and
  where I *diverged from the request*, and why), the files touched, the test output, the
  issues hit, and the outcome. It's a record of reasoning, not a changelog.
- **The scoped patch** is the diff, so the change is reviewable in isolation and
  reproducible.

### The ID scheme (how a record maps to a change)

The folder ID is picked by priority:

1. **The code-commit hash** — `<hash>-<slug>`. A hash ID pins the record to exactly one
   code change. (Separate doc/validation commits don't make a change "multi-commit.")
2. **The PR number** — `PR<nnnn>-<slug>`, when a change spans multiple code commits.
3. **A `Cxxxxxx` sequence** — the fallback when there's no PR, or the code is bundled into
   a commit that also carries other changes (so no single hash identifies it). C-numbers
   stay chronological.

Using the commit hash as the folder ID means every record maps to exactly one change in
the private repo. Given an ID, I can pull up the exact diff behind any claim in its
record. The code is closed, so this repo can't offer that check to readers; it documents
the convention so you can judge how the work is tracked.

> **Note on the sample:** [`sample-validation-record.md`](./sample-validation-record.md) is
> a real record from this project, lightly redacted — the business domain is genericized,
> host names / registry / exact commit IDs are anonymized, and file-line references are
> dropped. The structure and the discipline are exactly as they run.
