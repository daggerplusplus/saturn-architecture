# Restore runbook (representative sample)

A representative backup-validation and restore runbook for the data tier, lightly
redacted (node names, off-box paths, and the sync target are genericized). The point of
the document is the **philosophy**: a backup you haven't *restored and validated* is a
hypothesis, not a backup — so the runbook leads with a **non-destructive restore test**
and only then the real DR procedures.

## Context / what a backup is for

A single self-hosted node is the deployment, so **node death = total loss of local
state** unless the data has a copy somewhere else. The backup exists to close exactly that
risk. It streams the durable state **off the box** (the only off-box channel available to
a home cluster — a cloud object store or a second machine is the stated future step, not
the current reality).

> **PII note:** the Postgres dump contains user records including **password hashes**. The
> backup data is treated as sensitive, kept off-box, and excluded from version control.

## What's backed up

| Store | Method | Why this shape |
|---|---|---|
| **Postgres** | `pg_dump -Fc` (custom format) — the **source of truth** | One instance, several schemas → a single dump restores everything. Custom format allows per-table restore if ever needed. |
| **Qdrant (vectors)** | **Per-collection** snapshot via the REST API | The corpus and chat-history collections are the RAG state; snapshots are the supported export shape. |
| **Redis** | `RDB` snapshot | Captures the cache/queue/event state. (The *durable* record is Postgres; Redis here is the working state — restore it last, and expect the event bus to rebuild from the outbox/streams semantics rather than relying on a perfect Redis snapshot.) |

A **freshness manifest** is written on each successful run, and a daily backstop alerts if
the manifest is older than ~28h (i.e. the nightly never ran). **Retention** keeps a rolling
window (e.g. 14). On failure, an alert fires (the automation is UI-only — it notifies, it
does not self-heal).

## Step 0 — The non-destructive restore test (do this *first*, always)

Before trusting any backup, prove it restores — **into scratch space, never over the
live data**:

1. **Postgres:** restore the dump into a **throwaway database** on a scratch instance,
   then **compare row counts** (per table) against the live database. Counts matching
   (within the window of new writes) is the signal the dump is complete and restorable.
2. **Qdrant:** restore a collection snapshot into a **scratch collection**, and check the
   **point count** matches the source collection.
3. **Redis:** load the RDB into a scratch Redis and spot-check a known key/namespace.

If the counts don't line up, **the backup is bad — stop, don't restore over anything.**
This test is what turns "the backup ran" into "the backup is restorable." It's the reason
a restore has been *demonstrated* rather than assumed.

## Real restore procedures (DR)

> These mutate live state. Confirm the scope, have the validated backup from Step 0 in
> hand, and note the time window (expect data loss back to the last successful backup).

### Postgres (source of truth)
1. Stop writes (scale the app to zero, or `pg_terminate_backend` the app connections).
2. Restore the validated dump:
   - **Whole cluster:** drop + recreate the target database, `pg_restore -d <db> <dump>`.
   - **Single schema/table** (surgical): `pg_restore --schema=<s>` or `--table=<t>` from
     the custom-format dump.
3. Confirm the per-schema least-priv roles' grants are intact (restoring data does not
   re-create roles — the migrate Job owns DDL/roles; run it if roles are missing).
4. Bring the app back; smoke-test the golden path.

### Qdrant
1. Restore the validated collection snapshot into a **new** collection name first.
2. Verify point count, then swap: the owning service points at the restored collection
   (delete the stale one only after the swap is confirmed).

### Redis
1. Restore the RDB (restart the instance from it, into scratch first if in doubt).
2. Expect the event bus / job queue to reconcile: the outbox + consumer-group semantics
   re-drive durable effects; a lost in-flight Redis entry is recovered by the reclaim
   logic, not by the snapshot. Treat a Redis restore as *rebuilding working state*, not
   restoring the source of truth.

## After a restore
- Re-run the **Step 0 counts** against the *restored* data to confirm the restore itself
  landed correctly (not just that the backup was good).
- Update the freshness manifest; clear any staleness alert.
- Record the incident (what was lost, the window, what the restore validated) so the next
  restore has more context.

## What this runbook deliberately does *not* cover
- **Multi-replica HA** — this is a single node; there is no failover peer to promote.
  Recovery is *restore-from-backup*, not *fail over to a standby*. That's stated, not an
  oversight.
- **Automatic self-healing** — the backup/alerting pipeline notifies; a human (or the
  operator) executes the restore. The automation's job is to make the backup *happen* and
  *get noticed when it doesn't*, not to make irreversible decisions on its own.
