# One writer per session: the pen

A live shared session lets a friend log sets into someone else's session while
that session is happening, which puts two people's devices on one session at
once. Rather than reconciling their writes, we made concurrent writes impossible:
a session has a **pen**, exactly one person holds it at any moment, and only the
holder can write. The owner is the sole grantor — attaching a recorder hands them
the pen, and the owner may take it back or pass it on whenever they like.

## Considered options

**Merge-based sync** (last-write-wins per field, or CRDTs) was the obvious path
and was rejected on cost. It is the standard answer to this problem, but it
requires getting reconciliation right for a two-person gym app whose entire value
proposition is that it is fast and boring, and the failure mode when it is subtly
wrong — a set silently changing weight after the fact — is the kind that destroys
trust in a training record.

**Append-only** — recorders may add sets but never touch anything — was the
initial proposal, and it fails on the most ordinary case in the gym: your buddy
mistypes 88 reps for 8, and now the phone has to change hands to fix it.
Attempts to patch that (letting a recorder edit only their own sets) reintroduce
two concurrent writers through a side door.

## Consequences

Some of these look like bugs from the outside. They are not.

- **An owner cannot write into their own session while a recorder holds the pen.**
  They take the pen back first, which is one tap on their own phone. Carving out
  an exception for the owner would put two writers on the session again and undo
  the whole decision.
- **During mutual recording, neither person writes into their own session.** Each
  holds the other's pen and logs the other's sets. This is the intended shape and
  matches how people actually train together, but it means the recorder's screen
  is the primary surface during a shared session, not a secondary mode.
- **An offline pen holder's writes fail visibly and are never queued locally.**
  Queueing would mean a device writing into a session it can no longer be shown
  to hold the pen for, which is merge-based sync arriving through the back door.
  A visible failure is also what stops both people logging the same set twice.
- **The pen does not convey delete.** Deleting is owner-only and additionally
  requires the pen, because it is the only destructive act in the app; a recorder
  can correct their mistake but never erase it.
