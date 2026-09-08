# Track: Matching service internals

slug: matching-internals
days: 15
repos: server=cadence-workflow/cadence, client=cadence-workflow/cadence-go-client
status: curated

15 days, 10-15 minutes each. Matching owns task lists and worker polls: one in-memory task-list manager per partition, an unbuffered-channel matcher for sync match, a persistence-backed backlog for async match, an N-ary forwarding tree between partitions, and poller/isolation-group bookkeeping.

## The arc

### Week 1 (days 1-5): fundamentals

1. Request-to-poller map: service wiring → handler → engine → getOrCreateTaskListManager (`server:service/matching/service.go`, `server:service/matching/handler/handler.go`, `server:service/matching/handler/engine.go`, `server:service/matching/tasklist/task_list_manager.go`). Exercise: route an Add to a manager map, report whether a waiting poller consumed it.
2. One task-list manager lifecycle: NewManager, Start (lease first), Stop (`server:service/matching/tasklist/task_list_manager.go`, `server:service/matching/tasklist/task_list_registry.go`, `server:service/matching/tasklist/task_writer.go`). Exercise: Manager with Start/Stop and an idempotent stop channel.
3. InternalTask, the union type: local activity/decision, query, started remote tasks; sync response channels, forwardedFrom, completion callbacks (`server:service/matching/tasklist/task.go`, `server:service/matching/tasklist/interfaces.go`, `server:service/matching/handler/engine.go`). Exercise: union-like task struct with source, forwardedFrom, and a finish-once callback.
4. Sync match: matcher Offer/Poll rendezvous on unbuffered channels (`server:service/matching/tasklist/task_list_manager.go`, `server:service/matching/tasklist/matcher.go`, `server:service/matching/tasklist/matcher_test.go`). Exercise: unbuffered channel plus bounded context implementing producer Offer and consumer Poll.
5. Async backlog match: task_writer persists what didn't sync-match; the task_reader pump reads ranges, dispatches with retry, GC deletes below ack (`server:service/matching/tasklist/task_reader.go`, `server:service/matching/tasklist/task_writer.go`, `server:service/matching/tasklist/task_gc.go`, `server:service/matching/tasklist/task_reader_test.go`). Exercise: drain a slice-backed backlog; retain/retry on consumer error.

### Week 2 (days 6-10): depth

6. Persistence lease and task IDs: task-ID blocks from the range ID, lease renewal, fatal lease loss (`server:service/matching/tasklist/db.go`, `server:service/matching/tasklist/task_writer.go`, `server:service/matching/tasklist/task_list_manager.go`). Exercise: monotonic IDs from a fixed range, renew when exhausted.
7. Ack cursor and garbage collection: contiguous-prefix ack over gappy IDs (`server:common/messaging/ackManager.go`, `server:common/messaging/interface.go`, `server:service/matching/tasklist/task_reader.go`, `server:service/matching/tasklist/task_gc.go`). Exercise: accept 1,2,3, complete 2,1,3, report the ack cursor after each.
8. Partition names and read/write config: reserved-name N-ary tree, root = partition 0, read count >= write count (`server:service/matching/tasklist/identifier.go`, `server:common/types/matching.go`, `server:service/matching/tasklist/task_list_manager.go`, `server:service/matching/handler/engine.go`). Exercise: parse root and `/__cadence_sys/root/3` names into root plus partition, compute parent.
9. Forwarded polls and tasks: forwarder token channels, ForwardedFrom, ServiceBusy → ErrForwarderSlowDown; forwarded tasks are sync-only at the parent (`server:service/matching/tasklist/forwarder.go`, `server:service/matching/tasklist/matcher.go`, `server:service/matching/tasklist/forwarder_test.go`, `server:service/matching/tasklist/matcher_test.go`). Exercise: parent forwarder with an outstanding-request semaphore and ForwardedFrom field.
10. Dispatch throttling layers: task-list limiter learned from poller limits, divided across partitions; clock.ErrCannotWait → ErrTasklistThrottled (`server:service/matching/tasklist/task_list_limiter.go`, `server:service/matching/tasklist/matcher.go`, `server:service/matching/tasklist/task_reader.go`, `client:internal/internal_task_pollers.go`). Exercise: token bucket dividing total RPS across N partitions, refusing when the context cannot wait.

### Week 3 (days 11-15): edge cases

11. Poller history and cancellation: five-minute identity history, outstanding poll cancellation, per-group counts (`server:service/matching/poller/manager.go`, `server:service/matching/tasklist/task_list_manager.go`, `server:service/matching/handler/engine.go`, `client:internal/internal_worker_base.go`). Exercise: track poller IDs, cancel one outstanding poll, retain last identity plus end time.
12. Isolation groups and leakage guards: per-group matcher channels; drained/unknown/stale groups fall back to the default buffer (`server:service/matching/tasklist/task_list_manager.go`, `server:service/matching/tasklist/matcher.go`, `server:service/matching/poller/manager.go`, `server:service/matching/tasklist/isolation_balancer.go`). Exercise: route group-tagged tasks to a matching channel with default fallback.
13. Sticky task lists: client sticky-vs-regular poll balancing (backlog hint first), server recent-sticky-poller check, forwarder rejects sticky (`client:internal/internal_task_pollers.go`, `client:internal/internal_worker.go`, `client:isolationgroup/wrapper.go`, `server:service/matching/handler/engine.go`). Exercise: choose sticky vs regular poll by backlog, then smaller pending-poll count, sticky on ties.
14. Adaptive scaling and partition drain: aggregate QPS → adjust read/write partition counts → persist and broadcast via root; drain read-only partitions when empty (`server:service/matching/tasklist/adaptive_scaler.go`, `server:service/matching/tasklist/isolation_balancer.go`, `server:service/matching/tasklist/task_list_manager.go`, `server:service/matching/tasklist/task_reader.go`). Exercise: scale up on sustained QPS, remove a read-only partition only when backlog is empty.
15. Failure semantics and shutdown: fatal writer lease loss, membership eviction plus drain interval; wrap-up plus cumulative quiz (`server:service/matching/service.go`, `server:service/matching/handler/engine.go`, `server:service/matching/tasklist/task_writer.go`, `server:service/matching/tasklist/task_reader.go`). Exercise: coordinate writer/reader shutdown via a fatal-error channel; no task silently dropped.

## Traps worth teaching (weave into quizzes)

1. A task that misses sync match is not immediately persisted — a child partition may first forward the offer to its parent; forwarded tasks are sync-only at the receiver.
2. The ack level is the highest CONTIGUOUS completed prefix, not the highest completed ID.
3. Read and write partition maps are separate; a drained read-only partition stays readable until empty.
4. Isolation groups are not just labels — separate channels, buffers, dispatch durations, recency checks, and fallback metrics.
5. Sticky task lists do not participate in the forwarding tree; the client balances sticky vs normal polls itself using backlog hints.

## syllabus.md format

```markdown
# <track title> — <N>-day course

## Day NN: <topic title>
- **Goal**: <one sentence>
- **Anchors**: <2-4 repo-relative file paths, verified to exist>
- **Exercise idea**: <one line, a 5-10 line simplified implementation>
- **Builds on**: day(s) <...>
```

Anchor paths MUST be re-verified with `ls`/`grep` against the repos before the syllabus is finalized — the repos evolve.
