# Track: Shard management & membership

slug: shard-management
days: 15
repos: server=cadence-workflow/cadence
status: curated

15 days, 10-15 minutes each. The central model is two coupled systems: membership (ringpop hashring) answers which host SHOULD serve a shard, while the persisted shard row, guarded by a monotonically increasing RangeID, decides which host MAY successfully write. A host loads a shard engine lazily, and a host that loses the persistence fence closes that engine.

## The arc

### Week 1 (days 1-5): fundamentals

1. Shard identity and request routing: workflowID → shardID (modulo shard count), shardID → ring key (`service/history/config/config.go`, `service/history/lookup/lookup.go`, `service/history/shard/controller.go`). Exercise: workflowID → shardID → rune lookup key with configurable count.
2. Ringpop membership and hash-ring snapshots: MultiringResolver, ring refresh + subscriber notification (`common/membership/resolver.go`, `common/membership/hashring.go`, `common/membership/singleprovider.go`, `common/membership/hostinfo.go`). Exercise: sorted ring of host hashes, first host at-or-after a key, wrapping.
3. Controller ownership view and lazy engine creation: the shardManagementPump (membership events + periodic acquisition + shutdown), engines created on first need (`service/history/shard/controller.go`, `service/history/handler/handler.go`, `service/history/engine/interface.go`, `service/history/engine/engineimpl/history_engine.go`). Exercise: mutex-protected map whose Get creates and starts an engine on first request.
4. The persisted shard record: ShardInfo, UpdateShardRequest with PreviousRangeID, ShardOwnershipLostError (`common/persistence/data_manager_interfaces.go`, `common/persistence/shard_manager.go`, `common/persistence/sql/sql_shard_store.go`, `common/persistence/errors.go`). Exercise: store Update(expectedRange, next) rejecting mismatched range.
5. Acquisition end-to-end: acquireShard — get/create row, claim owner, renewRangeLocked conditional update, only then a usable context (`service/history/shard/context.go`, `service/history/shard/controller.go`, `common/persistence/shard_manager.go`, `service/history/engine/engineimpl/history_engine.go`). Exercise: acquire-or-create, set owner, bump epoch, start engine only after the epoch update succeeds.

### Week 2 (days 6-10): depth

6. RangeID fencing across persistence backends: SQL locked-row compare vs Cassandra CAS, condition failure → ShardOwnershipLostError → closeShard (`service/history/shard/context.go`, `common/persistence/sql/sql_shard_store.go`, `common/persistence/sql/sqlplugin/mysql/shard.go`, `common/persistence/nosql/nosqlplugin/cassandra/shard.go`). Exercise: two goroutines racing a CAS epoch update, count stale-writer rejections.
7. RangeID as a task-ID epoch: generateTaskIDLocked, next ID starts at `RangeID << RangeSizeBits` (`service/history/shard/context.go`, `service/history/config/config.go`, `common/persistence/data_manager_interfaces.go`). Exercise: IDs from `epoch << bits | seq`, renew on exhaustion, prove monotonicity across epochs.
8. Legacy transfer/timer/replication ack levels in shard info: task-ID vs timestamp cursors, per-cluster variants (`common/persistence/data_manager_interfaces.go`, `service/history/shard/context.go`, `service/history/queue/processor_base.go`, `service/history/queue/timer_queue_active_processor.go`). Exercise: safe minimum ack over immediate IDs and timer timestamps.
9. Queue-v2 read state and exclusive ack semantics (`service/history/queuev2/queue_base.go`, `service/history/queue/processing_queue.go`, `service/history/shard/context.go`). Exercise: advance exclusive ack only across contiguous acknowledged tasks; convert exclusive → legacy inclusive.
10. Per-shard engine and queue lifecycle: Start/Stop ordering, final ack persistence (`service/history/engine/engineimpl/history_engine.go`, `service/history/queue/transfer_queue_processor_base.go`, `service/history/queue/processor_base.go`, `service/history/shard/controller.go`). Exercise: worker goroutine with stop channel, persist one final ack, wait for clean termination.

### Week 3 (days 11-15): edge cases

11. Shard movement and stale-owner races: routing-view change + new RangeID claim; old owner discovers loss via fenced write, closes async (`service/history/shard/controller.go`, `service/history/shard/context.go`, `service/history/handler/handler.go`, `service/history/lookup/lookup.go`). Exercise: old and new owner writing the same record; stale write → ownership-lost, not blind retry.
12. Unknown write outcomes and defensive RangeID renewal: ambiguous errors force an epoch bump before trusting reads (`service/history/shard/context.go`, `common/persistence/sql/sql_shard_store.go`, `common/persistence/nosql/nosql_shard_store.go`, `common/persistence/errors.go`). Exercise: on ambiguous write error, bump epoch before a read decides whether the write committed.
13. Graceful drain, eviction, and forced stop: EvictSelf → gossip propagation → PrepareToStop → ownership-transfer delay → stop engines (`service/history/service.go`, `service/history/handler/handler.go`, `service/history/shard/controller.go`, `service/history/engine/engineimpl/history_engine.go`). Exercise: ordered shutdown phases with timers.
14. Shard-distributor migration and fallback (`common/membership/sharddistributorresolver.go`, `common/membership/hashring.go`, `cmd/server/cadence/server.go`, `cmd/sharddistributor-canary/main.go`). Exercise: resolver using remote owner lookup when enabled, local ring fallback otherwise.
15. Testing ownership loss and lifecycle invariants; wrap-up plus cumulative quiz (`service/history/shard/controller_test.go`, `service/history/shard/context_test.go`, `common/membership/hashring_test.go`, `service/history/queuev2/queue_reader_cached.go`). Exercise: table test for same-host epoch renewal vs moved-shard epoch change (cache retention vs invalidation).

## Traps worth teaching (weave into quizzes)

1. The membership ring is NOT the ownership lock — it is a routing view; the RangeID conditional write is the fence.
2. A membership change does not immediately stop the old engine; loss is normally discovered through a fenced persistence operation.
3. RangeID is also the epoch prefix for generated immediate task IDs.
4. There is no single ack cursor: transfer/replication use task IDs, timers use timestamps, levels can be per-cluster, queue-v2 adds richer state.
5. Graceful shutdown is evict → propagate → staged stop, not stop-then-fail.

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
