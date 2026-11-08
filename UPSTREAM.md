# Upstream report (draft, not filed)

The text of the issue and the pull request for matrix-org/gomatrixserverlib. The pull request is
commit `cb8cd45` of this repository alone.

## Issue

**Title:** State resolution v2.1 drops conflicted events and panics for room version 12

`ResolveConflictsNew` (and `ResolveStateConflictsV2New` with `StateResV2_1`) does not resolve the
conflicted state of a room version 12 room correctly, at `main` (`4fe595d`):

1. `authAndApplyEvents` falls back to an event's `auth_events` for a key the partial state lacks,
   but `addFromAuthEventsIfNotRejected` adds `event`, the event under check, to the auth provider in
   place of the matching `authEv`. State resolution v2.1 starts the iterative auth checks from an
   empty state map, so every conflicted event is checked before the partial state holds its sender's
   membership or the power levels, and fails auth: every conflicted non-control event is dropped.
   (Under v2 the unconflicted state is applied first, which hides the defect most of the time.)
2. In room version 12 the create event stands in no event's `auth_events`, and v2.1 applies no
   unconflicted state before the conflicted control events, so `resolvedCreate` is never set:
   `getPowerLevelFromAuthEvents` panics with "missing resolved create event, cannot calculate PL of
   sender!" on the first conflicted control event, and every auth check runs without the create
   event.
3. `calculateFullAuthChainAndConflictedSubgraph` prints `found conflicted subgraph …` to stdout for
   every conflicted event, and inserts a nil PDU into the subgraph for a conflicted event that is
   not among the auth events passed in.

To reproduce: a room version 12 room (create, the creator's join, power levels, join rules, a second
member's join), then two state sets that differ in one `m.room.topic` each. `ResolveConflictsNew`
returns no topic. With the sets differing in the power levels and the join rules instead, it panics.

## Pull request

**Title:** Fix state resolution v2.1 for room version 12

- `addFromAuthEventsIfNotRejected` adds the matching auth event, not the event under check.
- `ResolveStateConflictsV2New` holds the room's create event from the start (taken from the auth
  events, else from the state sets), as `ReverseTopologicalOrdering` does since #459: it takes part
  in every auth check and in the power level of a privileged creator, and under v2.1 it is otherwise
  absent until the unconflicted state is reapplied at the end. The create event is unique to the
  room, so this changes nothing under v2.
- The conflicted subgraph walk no longer prints, and skips an event missing from the auth events
  (such an origin is a conflicted event, which the full conflicted set holds already).

Tests (`stateresolutionv2_1_test.go`) build signed room version 12 events and check that a
conflicted topic resolves by mainline order, that an event present in one state set alone survives,
that conflicted power levels and join rules resolve without a panic, that a power level change its
sender may not make is dropped, and that nothing is written to stdout.
