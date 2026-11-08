# Siegelkern fork

This repository is matrix-org/gomatrixserverlib at upstream commit
[`4fe595d`](https://github.com/matrix-org/gomatrixserverlib/commit/4fe595dc7f58) with one patch on
top. siegelkern-operator's Matrix interop part (`services/matrix`) builds on it through a `replace`
in its `go.mod`.

**The defect.** State resolution v2.1, which room version 12 requires (MSC4297), fails in
`ResolveConflictsNew` / `ResolveStateConflictsV2New`:

- the iterative auth checks' fallback to an event's auth events adds the event itself to the auth
  provider in place of the matching auth event, so a conflicted event checked before the partial
  state holds its key fails auth; v2.1 starts from an empty state map, so every conflicted
  non-control event is dropped;
- the create event stands in no room version 12 event's `auth_events`, and v2.1 applies no
  unconflicted state first, so the resolver never holds it: the first conflicted control event
  panics in `getPowerLevelFromAuthEvents`, and every auth check lacks the create event;
- the conflicted subgraph walk prints each subgraph to stdout.

**The patch.** Commit
[`cb8cd45`](https://github.com/siegelkern/gomatrixserverlib/commit/cb8cd4552254e23a00b74e983dfd8a669242f0bb)
on branch `fix-state-res-v2.1`: the fallback adds the auth event, the resolver holds the room's create
event from the start, the print goes; tests resolve a room version 12 room's conflicted state. The
module path stays `github.com/matrix-org/gomatrixserverlib`.

**The upstream report.** Issue [matrix-org/gomatrixserverlib#471](https://github.com/matrix-org/gomatrixserverlib/issues/471),
pull request [#472](https://github.com/matrix-org/gomatrixserverlib/pull/472) (branch `fix-state-res-v2.1`).

**Removal.** Drop this fork and the `replace` once upstream carries an equivalent fix.

gomatrixserverlib
=================

[![GoDoc](https://godoc.org/github.com/matrix-org/gomatrixserverlib?status.svg)](https://godoc.org/github.com/matrix-org/gomatrixserverlib)

Go library for common functions needed by matrix servers. This library assumes Go 1.22+.
