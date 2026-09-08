# Track: History & mutable state

slug: history-mutable-state
days: 15
repos: server=cadence-workflow/cadence
status: curated

15 days, 10-15 minutes each. Mental model: mutable state is an in-memory projection of a workflow's history plus transaction deltas (pending entity maps, update/delete sets, buffered events, task inserts). The history builder collects events; closing a transaction turns the projection into a persistence mutation/snapshot plus an event batch appended to a history V2 branch tree. Everything runs under a per-execution cached, locked context.

## The arc

### Week 1 (days 1-5): fundamentals

1. The mutable-state projection: the MutableState interface, the builder's in-memory collections, replay via StateBuilder (`service/history/execution/mutable_state.go`, `service/history/execution/mutable_state_builder.go`, `service/history/execution/state_builder.go`). Exercise: projection with nextEventID, pendingActivities, ApplyEvent(kind) for start/complete.
2. Event IDs, versions, and buffered events: BufferedEventID sentinel, NextEventID advances only for committed events, FlushBufferedEvents (`service/history/execution/mutable_state_builder.go`, `service/history/execution/history_builder.go`, `service/history/execution/mutable_state_builder_methods_decision.go`). Exercise: event creator using a sentinel ID for buffered events.
3. Normal and transient history: transient decision retry events never hit the committed history (`service/history/execution/history_builder.go`, `service/history/execution/mutable_state_decision_task_manager.go`, `service/history/execution/mutable_state_builder.go`). Exercise: builder with normal/transient slices; Commit() returns only normal.
4. Activity lifecycle state: pending activity map, scheduled events through the events cache, terminal transitions (`service/history/execution/mutable_state_builder_methods_activity.go`, `service/history/execution/history_builder.go`, `service/history/execution/mutable_state.go`). Exercise: activity map rejecting duplicates, scheduled → started → completed.
5. Timer lifecycle state: TimerInfo map, fire/cancel with buffered-event special cases (`service/history/execution/mutable_state_builder_methods_timer.go`, `service/history/execution/history_builder.go`, `service/history/execution/mutable_state_builder.go`). Exercise: timer map with start/fire/cancel and unknown-timer errors.

### Week 2 (days 6-10): depth

6. Decision scheduling and retries: attempts, transient schedule/start on retry, DecisionInfo projection (`service/history/execution/mutable_state_decision_task_manager.go`, `service/history/execution/mutable_state_builder_methods_decision.go`, `service/history/decision/handler.go`). Exercise: decision record incrementing attempts, emitting events only on first attempt.
7. Completing decisions and applying commands: token validation, the per-decision dispatch loop, activity local dispatch (`service/history/decision/handler.go`, `service/history/decision/task_handler.go`, `service/history/execution/mutable_state_builder_methods_activity.go`, `service/history/execution/mutable_state_builder_methods_timer.go`). Exercise: command loop switching on activity/timer commands.
8. Closing a mutable-state transaction: mutation vs snapshot, event batch, delta clearing; the standard locked update loop (`service/history/execution/mutable_state_builder.go`, `service/history/execution/context.go`, `service/history/workflow/util.go`). Exercise: Close() returning snapshot plus events, then clearing deltas.
9. History V2 branch persistence: the branch tree, AppendHistoryNodes validation, ancestor-range reads (`common/persistence/data_manager_interfaces.go`, `common/persistence/data_store_interfaces.go`, `common/persistence/history_manager.go`, `common/persistence/persistence-utils/history_manager_util.go`). Exercise: in-memory branch store with append, fork-at-event, and ancestor+current reads.
10. Rebuilding state from history: paginate → replay → validate boundary (`service/history/execution/state_rebuilder.go`, `service/history/execution/state_builder.go`, `common/persistence/persistence-utils/history_manager_util.go`). Exercise: replayer applying ordered event batches to a fresh projection.

### Week 3 (days 11-15): edge cases

11. Execution cache, locks, and release semantics: pinned cache, lock before return, once-only release that clears only on error (`service/history/execution/cache.go`, `service/history/execution/context.go`, `service/history/workflow/util.go`). Exercise: cache with per-key mutex and once-only release that evicts on error.
12. Event-cache fallback and corrupted history: single-event cache, branch read fallback, missing-event errors (`service/history/events/cache.go`, `common/persistence/history_manager.go`, `service/history/execution/mutable_state_builder_methods_activity.go`). Exercise: single-event cache falling back to a branch reader.
13. Query consistency and registry races: buffered vs direct dispatch, strong consistency waiting on the first decision, atomic termination (`service/history/engine/engineimpl/query_workflow.go`, `service/history/query/registry.go`, `service/history/query/query.go`, `service/history/execution/mutable_state.go`). Exercise: query registry with buffered/completed/unblocked outcomes via channel plus mutex.
14. Reset, branch fork, and event reapplication (`service/history/reset/resetter.go`, `service/history/ndc/workflow_resetter.go`, `service/history/execution/state_rebuilder.go`, `common/persistence/history_manager.go`). Exercise: reset routine — replay prefix, fork branch, reapply signals, schedule decision.
15. Failover, stale state, and conflict retry: stale-read and conditional-write conflict retries in the update loop; wrap-up plus cumulative quiz (`service/history/workflow/util.go`, `service/history/decision/handler.go`, `service/history/execution/context.go`, `service/history/execution/mutable_state_decision_task_manager.go`). Exercise: update loop retrying a stale read by clearing and replaying.

## Traps worth teaching (weave into quizzes)

1. A history event does not necessarily consume the next event ID — buffered events get a sentinel ID; NextEventID advances only for committed events.
2. Mutable state is far more than the persisted WorkflowExecutionInfo: pending maps, deltas, buffered events, task inserts, queries, version histories, and the history builder.
3. Decision retries emit TRANSIENT schedule/start events; the failed/timeout event is written only on the first attempt.
4. Adding an event is not persisting it — close builds a mutation/snapshot and the context appends the batch via AppendHistoryNodes.
5. A cache hit is not permission to mutate — the execution cache locks the context before returning it, and release clears only on error.

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
