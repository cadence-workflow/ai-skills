# Track: Replication & multi-cluster

slug: replication-multicluster
days: 15
repos: server=cadence-workflow/cadence
status: curated

15 days, 10-15 minutes each. Mental model: a durable, per-shard replication queue carrying three internal task forms — history ranges, activity state, and graceful-failover markers. The remote side fetches by per-shard cursor, hydrates history from mutable state and raw branch storage, and applies it through NDC, where version histories and branch tokens are the source of truth and a deterministic winner is chosen between divergent workflows.

Assumes task-processing week 1 (task categories, queue basics) or teaches it in passing on day 1.

## The arc

### Week 1 (days 1-5): fundamentals

1. The replication queue contract: replication as an immediate task category (ID 3), read strictly after cursor through exclusive max (`common/persistence/data_manager_interfaces.go`, `common/persistence/tasks.go`, `service/history/replication/task_reader.go`). Exercise: model an immediate category read over the half-open `(readLevel, maxReadLevel]`.
2. Generating history and activity tasks: one history replication task per committed event batch; sync-activity tasks; replication IDs allocated last under the shard lock (`service/history/execution/mutable_state_builder.go`, `service/history/shard/context.go`, `common/persistence/tasks.go`). Exercise: `eventsToTask` returning firstEventID, lastEventID+1, version.
3. Serialization and hydration boundaries: compact persistence blob vs hydrated wire task with raw history blobs (`common/persistence/serialization/task_serializer.go`, `service/history/replication/task_hydrator.go`, `service/history/engine/engineimpl/get_replication_messages.go`). Exercise: JSON round trip of a task descriptor.
4. Fetch, read, ack: host-local fetch loops per source cluster, aggregated shard requests, GetReplicationMessages, per-source-cluster cursor (`service/history/replication/task_fetcher.go`, `service/history/replication/task_ack_manager.go`, `service/history/replication/task_reader.go`). Exercise: cursor loop fetching batches until hasMore is false, advancing only to last returned.
5. Processor ordering and backpressure: per-shard consumer loop, rate limiting (markers exempt), retries, DLQ conversion on failure (`service/history/replication/task_processor.go`, `service/history/replication/task_executor.go`, `service/history/replication/task_ack_manager.go`). Exercise: sequential processor that stops before ack advancement on error.

### Week 2 (days 6-10): depth

6. NDC task construction: validate the V2 RPC, derive source cluster from the first event version (`service/history/ndc/replication_task.go`, `service/history/replication/task_hydrator.go`, `service/history/ndc/history_replicator.go`). Exercise: validator rejecting an empty batch plus a version-to-cluster map lookup.
7. Version history items and LCA (`common/persistence/versionHistory.go`, `common/types/shared.go`, `service/history/ndc/branch_manager.go`). Exercise: LCA scan over two sorted (eventID, version) slices.
8. Branch forking and out-of-order delivery: duplicate vs retry vs append decision, fork plus register in VersionHistories (`service/history/ndc/branch_manager.go`, `service/history/ndc/replication_task.go`, `service/history/ndc/history_replicator.go`). Exercise: append decision — behind → duplicate, ahead → retry, else append.
9. Conflict resolution and rebuild: current/stale/newer branch, rebuild mutable state from a newer non-current branch, workflowHappensAfter (`service/history/ndc/conflict_resolver.go`, `service/history/execution/workflow.go`, `service/history/ndc/existing_workflow_transaction_manager.go`). Exercise: comparator ordering by last-write version → task ID → timestamp/runID tiebreakers.
10. NDC persistence policies: create-current, update-current, update-zombie, conflict-resolve (`service/history/ndc/transaction_manager.go`, `service/history/ndc/existing_workflow_transaction_manager.go`, `service/history/ndc/history_replicator.go`). Exercise: state table over existence plus isRebuilt.

### Week 3 (days 11-15): edge cases

11. Failover version arithmetic: cluster identity in the increment remainder, aligned advancement (`common/cluster/metadata.go`, `common/domain/handler.go`, `common/types/replicator.go`). Exercise: nextVersion via `current/increment*increment + initial`, plus increment when not greater.
12. Graceful failover markers: pending markers in shard info, marker tasks emitted on domain update, validation against domain failover version (`service/frontend/api/domain_handlers.go`, `common/domain/handler.go`, `service/history/engine/engineimpl/register_domain_failover_callback.go`, `service/history/shard/context.go`). Exercise: marker filter — accept only pending markers for non-active domains not yet passed.
13. Active-active cluster attributes: per-scope active clusters, fencing version bumps even when the active name is unchanged (`common/domain/handler.go`, `service/history/engine/engineimpl/register_domain_failover_callback.go`, `service/history/shard/context.go`, `service/history/ndc/transaction_manager.go`). Exercise: emit (scope, name) pairs whose ActiveClusterName equals the local cluster.
14. Replication DLQ, merge, and purge: hydrate-or-fetch-from-source, merge with forceApply, purge as delete-only (`service/history/replication/task_processor.go`, `service/history/replication/dlq_handler.go`, `common/persistence/history_task_dlq_manager.go`, `service/history/engine/engineimpl/dlq_operations.go`). Exercise: retry loop appending failed descriptors to a DLQ, force-applying in order, removing only processed IDs.
15. Resend, reapply, and recovery drills: paged history resend from retry hints, signal reapplication with dedup keys; wrap-up plus cumulative quiz (`common/ndc/history_resender.go`, `service/history/replication/task_executor.go`, `service/history/engine/engineimpl/reapply_events.go`, `service/history/ndc/events_reapplier.go`). Exercise: range-resend planner deduplicating (runID, eventID, version) keys.

## Traps worth teaching (weave into quizzes)

1. Replication task IDs are NOT workflow event IDs — they are shard-allocated immediate queue sequence IDs, allocated last.
2. A history replication task does not carry the full payload — hydration loads raw event blobs from history storage.
3. A higher incoming version does not always win — branch currency, policy mismatch, then deterministic tiebreakers decide.
4. Failover versions are not `current + 1` — they encode cluster identity in the increment remainder; active-active changes bump the fencing version even when the active cluster name is unchanged.
5. DLQ merge is not a delete — it hydrates missing payloads from the source cluster, executes with forceApply, and only then deletes the range.

## syllabus.md format

```markdown
# <track title> — <N>-day course

## Day NN: <topic title>
- **Goal**: <one sentence>
- **Anchors**: <2-4 repo-relative file paths, verified to exist>
- **Exercise idea**: <one line, a 5-10 line simplified implementation>
- **Builds on**: day(s) <...>
```

Anchor paths MUST be re-verified with `ls`/`grep` against the repos before the syllabus is finalized — the repo evolves.
