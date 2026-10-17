# Blaze from v1, so account deletion runs on the server

Account deletion is a store submission gate: Apple requires it inside the app, and
Play also requires a way to ask for it after uninstalling. It collides with
Firestore's free plan. [ADR-0002](0002-flutter-and-firebase.md) requires every
`Set` to be its own document, so a five-year user's cloud copy is roughly 26,000
documents, against **20,000 free deletes a day for the whole project**. On the
Spark plan, exhausting a daily quota makes every further operation fail with
`RESOURCE_EXHAUSTED` **for every user** until the reset. Deleting a document does
not delete its subcollections, and client SDKs cannot list them.

We chose to **move the Firebase project to the Blaze plan for v1** and to delete an
account through **a "delete my account" function on the server**, which the app
calls and which replies only once the deletion is complete.

## Considered options

**Client-driven deletion across days, on Spark** was ruled out on three counts. The
phone has to survive for days to finish. Every day it runs, it spends the delete
quota every other user needs. And the local copy *is* Firestore's offline cache of
the cloud documents, so deleting the cloud copy through the phone's own Firestore
instance deletes the phone's copy too — which breaks the "delete my account and
keep this phone's copy" choice outright.

**Delete the sign-in at once and sweep the data from the developer's Mac, on
Spark** is free and would work, paced under the quota. It was ruled out as
permanent manual work for a solo developer: a sweep at least monthly to meet
Nevada's 30-day deadline, and a hand-written email for every completion
confirmation Apple requires.

**A function triggered when the Firebase Auth user is deleted**, including the
official *Delete User Data* extension, was ruled out. The 2nd-gen trigger is in
Preview, with no SLA. The extension is pinned to a Node runtime retired on
2026-10-30. And a trigger runs *after* the account is already gone, so the app
cannot tell the user the deletion has finished.

**Storing a session's sets in one document** would make deletion cheap and is
ruled out by ADR-0002's one-document-per-set rule, which keeps fast set entry
under Firestore's per-document write ceiling.

## Consequences

- **Blaze keeps Spark's free daily allowance and bills beyond it**: in
  `us-central1`, $0.01 per 100,000 deletes, $0.03 per 100,000 reads and $0.09 per
  100,000 writes. A heavy user's whole deletion costs about $0.0006.
- **The budget alert ADR-0002 requires is attached in the same act** as the plan
  change. Budget alerts only notify; nothing caps Firestore spending. A spend-cap
  budget that pauses Cloud Functions must not be used, because it would switch off
  account deletion, which is a store requirement.
- **Exhausting a quota becomes a small bill instead of an outage for everyone.**
  This matters beyond deletion: on Spark, a heavy user restoring onto a new phone
  reads about 26,000 documents, half the daily read allowance, so two in one day
  would stop sync for every user.
- **The function checks for a fresh sign-in**, which the app's mandatory
  sign-in-again step provides. It deletes the user's cloud data and everything
  about them stored outside it (friendships on both sides, friend requests sent
  and received, split grants given and received, the entitlement record), then the
  Firebase Auth user, and only then replies. It is idempotent, so a retry after a
  dropped connection finishes the job.
- **Anything stored about a user outside their own data tree must be listed in the
  deletion function.** The server deletes the user's own tree recursively, so its
  subcollections take care of themselves. A new collection elsewhere that names an
  account is not covered until it is added. That is a standing schema rule.
- **A daily cleanup repeats the sweep for accounts deleted the previous day.** An
  ID token stays valid for up to an hour after the user is deleted, so a replaced
  installation's queued writes can recreate documents inside a deleted account.
- **Sign in with Apple revocation happens on the phone**, using the authorization
  code from the sign-in-again step, before the function is called. Firebase stores
  no Apple tokens, so a deletion requested by email cannot revoke, and the
  confirmation reply tells the user how to stop using Sign in with Apple
  themselves.
- **A deletion requested by email runs the same deletion code from the developer's
  Mac** through the Admin SDK.
- **Firestore keeps no backups**: point-in-time recovery is off and no scheduled
  backup exists. Turning either on must revisit deletion and the retention
  disclosure.
- **[ADR-0004](0004-force-update-over-schema-migration.md)'s cost argument against
  rewrite migrations no longer holds**, and it has been amended to rest on its
  remaining grounds.
