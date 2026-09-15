# Force update instead of schema migration

Gym Notes stores a user's training record in Cloud Firestore, cached on the
device by Firestore's offline persistence. Every schema change in an app that
keeps years of data normally needs a migration story. This one does not have
one: **the stored shape only ever gains optional fields, no schema version is
stored in user data at all, and a change that cannot be made additively is
shipped as a forced update.**

Two supporting rules make that hold: **one device signed in per account at a
time**, and **attaching a recorder requires both apps at the same schema
version**.

## Considered options

**Rewrite migrations** — ship version N+1, walk the record, rewrite every
document — were the starting assumption and are ruled out on cost. Firestore's
free quotas are per project, per day, across every user: 50,000 reads and
20,000 writes ([ADR-0002](0002-flutter-and-firebase.md)). The same ADR requires
every `Set` to be its own document, so five years at five sessions a week is
roughly 26,000 set documents for *one* user. Migrating a single heavy user's
record therefore exceeds the whole project's daily write quota before anyone
else opens the app. There is no cheaper local path either — Firestore's offline
persistence is a cache of the cloud documents, not a separate store, so
migrating on the device and writing to the cloud are the same act. Cloud
Functions require the Blaze plan just to provision, so a server-side migration
is not available in v1 at all. The machinery is kept as a declared exception
that has to justify itself, because promising never to need it is a promise
broken under duress and then handled badly.

**A schema version stored on the user-data root**, compared against a constant
in the app, was designed in full before being discarded. The reasoning was that
the app knows what it understands while only the record can say what has been
done to it. It collapsed when the routes by which the data could actually get
ahead of the app were enumerated, because every one of them is closed: a
recorder writing a newer shape is prevented by the attachment check; a newer
export file is refused outright by
[the import format](https://github.com/kingdragonfly43/gym-notes/issues/15); a
restore onto a device too old to run the current build never yields a running
app; and a second device on one account is not allowed. With those closed, the
comparison can never come out true, and a field whose only job is to detect an
unreachable state is dead defensive code — the kind most likely to be wrong on
the day it finally runs.

**A per-record version gate rather than a global floor** was argued for on
precision: it would block only the devices that have genuinely met data they
cannot read, leaving a single-device user who has not updated in six months to
carry on untroubled. The precision turns out to buy nothing once one device per
account is the rule, and it costs the stored version field that the previous
paragraph removes.

**Blocking writes while leaving history readable** was the proposed humane
version of the gate — nobody should lose sight of two years of lifts in a
basement gym with no signal. It was rejected because it buys a partially
capable app that has to be designed, tested and reasoned about at every read
path, in order to soften a state that only arises after a user has ignored
updates long enough for the floor to move. If an old device cannot update, it
cannot be used.

**Reserving room for known growth** was considered and declined. Completion
ratio is already a number, so markings beyond half reps need no schema change,
and the post-v1 backlog mostly adds collections rather than changing existing
ones. Measurement type is the one genuinely breaking case — a closed enum
defined in code whose *values* travel in user data on each exercise document,
so a future type with a different step shape, such as cardio's time and
distance, is unrenderable by an older build. Both proposed insurances — a
measurement-type read-only fallback, and a general rule skipping an
uninterpretable document with a placeholder row — were rejected as second
mechanisms built to avoid using the first. A new measurement type is a forced
update, and that is an acceptable price.

## Consequences

- **`SCHEMA_VERSION` is a code constant and appears nowhere in user data.** It
  is the single version line shared with the export file, satisfying the
  one-version-line commitment made when the import format was decided. An
  export carrying a higher number than the reading app still refuses outright.
- **Apps write changed fields only, never a whole-document `set()`.** Without
  this rule, expand-only is unsafe: a build editing a set silently strips the
  fields it does not know about.
- **Force update is driven by a minimum supported app version held in a
  Firestore document**, read at launch and cached. A device offline with a
  stale cached value proceeds normally, which is correct — it cannot be meeting
  newer data either.
- **The minimum must not be raised until both stores are serving the new
  build.** Apple review and Play's staged rollout do not land together, and
  raising the floor early locks out a platform with nothing to update to. This
  is a release-process rule, not app behaviour.
- **Downgrade is unsupported and presents as the update gate**, never as
  corruption. A reinstall of an older build, a TestFlight step back, or a
  device restore all resolve to "update Gym Notes to continue".
- **Attaching a recorder is refused unless both apps are at the same schema
  version**, in either direction, and the refusal lands at attach time rather
  than mid-session. The asymmetric rule — that the recorder need only be able
  to *read* the owner's version — is wrong: a newer recorder creating an
  exercise with a measurement type the owner's app cannot interpret would lock
  the owner out of their own record.
- **The minimum OS floor now evicts devices rather than degrading them**, since
  there is no read-only fallback. This is a direct input to
  [Minimum supported OS versions](https://github.com/kingdragonfly43/gym-notes/issues/17).
- **One device per account has to be actively enforced.** Firebase Auth will
  sign one credential in on any number of devices, so this needs an active
  device recorded on the account and a new sign-in deposing the old one — along
  with an answer for the deposed device's local cache and its queued offline
  writes. That is an identity decision rather than a schema one and is
  ticketed separately.
