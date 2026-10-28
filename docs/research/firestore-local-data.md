# Firestore as the phone's only store: research

Research for [#31](https://github.com/kingdragonfly43/gym-notes/issues/31). This document
checks the Firestore and Firebase Auth behaviour that the choice of where a never-signed-in
user's data lives depends on. It makes no decision. Where a finding bears on a decision,
the implication is stated and the decision is flagged under **Decisions this forces**.

The three candidates it is measured against:

- **(A) Firestore only.** Firestore's offline cache is the only on-device store. The
  network stays disabled (`disableNetwork()`) until the user signs in, possibly with no
  Firebase Auth user at all until then.
- **(B) Local database as the source of truth.** A separate on-device database (for
  example SQLite through drift) holds the data. Firestore is only the cloud copy for
  signed-in users, kept in step by a hand-written sync layer.
- **(C) Local database until sign-in.** A separate local database is used only by users
  who have never signed in. It is copied into Firestore at sign-in, and Firestore's
  cache takes over from then on.

Every finding is graded:

- **Confirmed**: primary source with a URL, meaning firebase.google.com or
  docs.cloud.google.com documentation, SDK source on GitHub, or a statement by a
  Firebase or Firestore team member in a Firebase GitHub repository.
- **Community**: a credible secondary source with a URL.
- **Inferred**: reasoning from the sources above, applied to this app. It has no URL of
  its own, and anything undocumented is labelled as such.

Research date: **2026-10-27**. Source links are permalinks to these commits:
firebase-android-sdk
[`9179112`](https://github.com/firebase/firebase-android-sdk/tree/91791120f81140fcd7b908d8c99b326b0ae72025),
firebase-ios-sdk
[`854b8ae`](https://github.com/firebase/firebase-ios-sdk/tree/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df)
and flutterfire
[`24ff83a`](https://github.com/firebase/flutterfire/tree/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f).
FlutterFire's `cloud_firestore` and `firebase_auth` hand their work to these native SDKs
on iOS and Android, so the native source is where the behaviour actually lives.

### Facts this research relies on (from earlier tickets, not re-litigated)

- Every set is its own Firestore document. A heavy user has about **26,000 set
  documents** ([ADR-0002](../adr/0002-flutter-and-firebase.md),
  [ADR-0004](../adr/0004-force-update-over-schema-migration.md)).
- **One signed-in installation per account at a time.** A replaced installation's
  offline-queued writes may still land once
  ([ADR-0004](../adr/0004-force-update-over-schema-migration.md),
  [#20](https://github.com/kingdragonfly43/gym-notes/issues/20)).
- Signing in onto a **non-empty cloud account** makes the user pick a side and discards
  the other. Sign-out keeps the local data on the phone
  ([v1 spec §6.3–6.4](../v1-spec.md)).
- The project is on **Blaze**
  ([ADR-0005](../adr/0005-blaze-for-server-side-account-deletion.md)).

---

## Bottom line

1. **`signInAnonymously()` is a server call and cannot succeed offline.** A first launch
   with no connection has no Firebase user until the app retries online. — **Confirmed**
   (iOS SDK source). No document says so in words.
2. **Pending writes are queued per UID and are never sent under another user's
   credentials.** When the signed-in UID changes, the old user's queue is parked on disk
   and disappears from the app's view of the data. It is neither uploaded nor deleted. It
   wakes up only if that same UID becomes current again on this installation. Writes made
   with no Auth user at all sit in a queue for the uid `""`. `linkWithCredential()` keeps
   the UID, so no switch happens. — **Confirmed** (Android and iOS source, FlutterFire API
   docs).
3. **Tens of thousands of pending writes are survivable but not what the SDK is built
   for.** Since 2022, "overlays" keep local query time flat as pending writes grow.
   Firestore's own client lead advised against a network-disabled instance as a store,
   because "this queue will grow without bound". On `enableNetwork()` the queue drains
   one batch per write-stream message, with at most **10 batches in flight**. No flush
   rate is documented. A Firebase engineer confirmed in July 2026 that a batch whose
   acknowledgement is lost **can be applied twice**. — **Confirmed**, apart from the
   flush-rate estimate, which is **Inferred**.
4. **The cache is not a guaranteed complete copy.** The default size is 100 MB with LRU
   garbage collection. Pending writes and documents held by active listeners are never
   evicted. `CACHE_SIZE_UNLIMITED` only turns eviction off: the cache still holds only
   what this device has read or written, and the docs call cached results possibly
   "stale or incomplete". — **Confirmed**. Using the cache as the only copy for export, or
   for "keep this phone's copy", is therefore unreliable. — **Inferred**.
5. **Server-side deletes reach the cache only through listeners or queries that cover the
   deleted documents.** The synced-document cache is shared across users. Sign-out does
   not clear it, and a later user on the same install can read it from the cache.
   `clearPersistence()` works only on a stopped instance, and its own documentation says
   it is not a secure wipe. — **Confirmed** (source and API docs). Readability by a later
   user is **Inferred** from the schema.
6. **Two Firestore instances can run side by side with separate on-disk stores**, either
   through a second named `FirebaseApp` or a second database ID. A Firestore engineer
   suggested exactly this for local-only data: a secondary app with `disableNetwork()`
   called first. He also warned that it suits only small, rarely updated data, and that
   moving it to the cloud means reading the documents and writing them again. —
   **Confirmed**.
7. **There is no longer a 500-write batch limit** (removed 2023-03-29). Batches are now
   limited by the 10 MiB request size. Firebase recommends server libraries such as
   BulkWriter for bulk loads, and no client SDK has one. Writes committed through the SDK
   are written to disk locally first, so they survive the app being killed and resume on
   the next start. Work the app had not yet handed to the SDK is lost and must be
   resumable. — **Confirmed**, apart from the resumability point, which is **Inferred**.

---

## 1. Does `signInAnonymously()` need a network connection?

**Yes.** The iOS SDK's `signInAnonymously` returns any existing anonymous user straight
away. Otherwise it builds a `SignUpNewUserRequest` and awaits `backend.call(with: request)`.
The ID token, refresh token and UID come only from the server's response, and any error
from that call fails the sign-in. — **Confirmed**:
[Auth.swift L700–731](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/FirebaseAuth/Sources/Swift/Auth/Auth.swift#L700-L731),
[SignUpNewUserRequest.swift L18](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/FirebaseAuth/Sources/Swift/Backend/RPC/SignUpNewUserRequest.swift#L18)
(endpoint `signupNewUser`).

- The Android Auth SDK is closed source, and neither the
  [Flutter anonymous-auth guide](https://firebase.google.com/docs/auth/flutter/anonymous-auth)
  (updated 2026-10-20) nor the
  [FlutterFire API doc](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/firebase_auth/firebase_auth/lib/src/firebase_auth.dart#L438-L450)
  says anything about offline behaviour. The API doc lists only `operation-not-allowed`.
  Since the UID is issued by the server on every platform, Android must also need the
  network. — **Inferred**. Offline, the call fails with FlutterFire's generic
  `network-request-failed`. — **Community**:
  [firebase-js-sdk#8978](https://github.com/firebase/firebase-js-sdk/issues/8978)
  shows that code for anonymous sign-in.
- **First launch with no connectivity:** `currentUser` stays `null` and the app has no
  UID. **The SDK has no "sign in later" or offline-provisional mode.** — **Inferred** from
  the source above. On later launches, a user who was signed in once is restored from
  local storage without the network, but refreshing the token offline can fail.
  [flutterfire#1104](https://github.com/firebase/flutterfire/issues/1104) reports exactly
  that for `getIdToken()`. — **Community**.
- `signInAnonymously()` **signs out any other signed-in user**: "If there is any other
  existing user signed in, that user will be signed out." — **Confirmed**,
  [FlutterFire API doc L440–442](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/firebase_auth/firebase_auth/lib/src/firebase_auth.dart#L438-L446).
- **Even an anonymous account is a server-side record.** Creating one is itself a network
  call, and Firebase rate-limits anonymous sign-ups per IP address
  ([Flutter guide](https://firebase.google.com/docs/auth/flutter/anonymous-auth)). An
  account that is never linked can be deleted automatically after 30 days only if the
  project is upgraded to Identity Platform and clean-up is turned on
  ([anonymous auth, web](https://firebase.google.com/docs/auth/web/anonymous-auth)). —
  **Confirmed**.

**What this means for (A):** if the plan is "an anonymous UID before the first write",
that cannot hold on an offline first launch. The app either blocks, which breaks the
zero-friction first run, or writes before a UID exists. Writing first runs into §2. —
**Inferred**.

## 2. Are pending writes queued per user?

**Yes, by UID, on both mobile SDKs.** — **Confirmed**.

- **Storage is partitioned by UID.** The Android SQLite schema creates
  `mutation_queues (uid TEXT PRIMARY KEY, …)` and `mutations (uid, batch_id, …)` with the
  comment "All the mutation batches in the system, partitioned by user". The
  `document_overlays` table is keyed by `uid` too
  ([SQLiteSchema.java L249–270](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteSchema.java#L249-L270),
  [L705–715](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteSchema.java#L705-L715)).
  iOS keys its LevelDB mutation rows by `user_id_`
  ([leveldb_mutation_queue.cc L135–144](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/Firestore/core/src/local/leveldb_mutation_queue.cc#L135-L144)).
- **"No user" is a user with uid `""`.** Firestore's `User.UNAUTHENTICATED` has a null
  UID, and the mutation queue stores it as `""`. The Auth credentials provider reports
  `UNAUTHENTICATED` whenever Firebase Auth has no current user
  ([User.java L26](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/auth/User.java#L26),
  [SQLiteMutationQueue.java L62, L98](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteMutationQueue.java#L98),
  [FirebaseAuthCredentialsProvider.java L151–153](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/auth/FirebaseAuthCredentialsProvider.java#L151-L153)).
- **When the user changes, the queue is swapped.** The swap happens only when the UID
  differs. `SyncEngine.handleCredentialChange` computes
  `userChanged = !currentUser.equals(user)`. If it is true, it fails
  `waitForPendingWrites` waiters and calls `localStore.handleUserChange(user)`, which
  "Swap[s] out the mutation queue". It rebuilds the mutation queue, overlay cache and
  index manager for the new user, then re-emits every document either queue touches
  ([SyncEngine.java L784–796](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/core/SyncEngine.java#L784-L796),
  [LocalStore.java L175–185, L211–233](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/LocalStore.java#L211-L233);
  on iOS:
  [sync_engine.cc L301–315](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/Firestore/core/src/core/sync_engine.cc#L301-L315),
  [local_store.cc L148](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/Firestore/core/src/local/local_store.cc#L148)).
  The remote store then restarts its streams "since mutations are per-user"
  ([RemoteStore.java L347–356](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/remote/RemoteStore.java#L347-L356)).
- **The public API says the same.** `waitForPendingWrites` "Waits until all currently
  pending writes for the active user have been acknowledged", and "Any outstanding
  [waitForPendingWrites] calls are rejected during user changes" —
  [FlutterFire firestore.dart L313–322](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/cloud_firestore/cloud_firestore/lib/src/firestore.dart#L313-L322).
  — **Confirmed**.

**So writes queued under user X (or under no user) when the user changes to Y are:**

- **Not sent under Y's credentials.** The write pipeline is filled from the *current*
  user's queue only. — **Confirmed** (above).
- **Not dropped either.** Nothing in `handleUserChange` deletes the old user's rows. They
  stay on disk and hidden. If X becomes the current user again on this installation, with
  the network on, they flush under X's credentials. A queue for uid `""` would flush
  *unauthenticated* the next time nobody is signed in and the network is on. — **Inferred**
  from the source. **Undocumented.**
- **Invisible from then on.** The local view is "computed by applying the mutations in
  the MutationQueue to the RemoteDocumentCache"
  ([LocalDocumentsView.java L57–59](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/LocalDocumentsView.java#L57-L59)),
  meaning the current user's queue. Documents that existed only as X's pending writes
  disappear from every query after the switch. — **Confirmed** as mechanism; the
  consequence is **Inferred**.
- **Still costly to carry.** Old queues keep pinning their documents against garbage
  collection, because the pin check looks at `document_mutations` without filtering on
  UID
  ([SQLiteLruReferenceDelegate.java L132–150](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteLruReferenceDelegate.java#L132-L150)).
  The swap itself loads *all* batches of both queues
  (`getAllMutationBatches()`), which is a one-off cost proportional to tens of thousands
  of batches. — **Confirmed** as code; the cost is **Inferred**.
- **A rejected write is deleted locally.** If X's queue does flush later and security
  rules reject a batch, the SDK removes that batch from the queue and rolls the local view
  back (`handleRejectedWrite` → `localStore.rejectBatch`). The data is lost
  ([SyncEngine.java L500–513](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/core/SyncEngine.java#L500-L513),
  [LocalStore.java L342–352](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/LocalStore.java#L342-L352)).
  FlutterFire users report this as data loss:
  [flutterfire#13138](https://github.com/firebase/flutterfire/issues/13138) (open). The
  rejected write's contents have still been sent to Google's servers before the
  rejection. — **Confirmed**, apart from the point about contents being sent, which is
  **Inferred**.

**Does `linkWithCredential()` avoid the switch?** Yes, while it succeeds. Linking keeps
the UID. The credentials provider reports `new User(uid)` on every ID-token change,
`User.equals` compares only the UID, and `handleCredentialChange` therefore does not swap
queues. It only restarts streams with the new token, and only if the network is enabled
([User.java](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/auth/User.java#L44-L56),
[RemoteStore.java L347–356](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/remote/RemoteStore.java#L347-L356)).
— **Confirmed**.

**Linking can fail and force a UID change.** If the Google or Apple credential already
belongs to a Firebase user, `linkWithCredential` throws `credential-already-in-use`. The
documented recovery is "signing in with `credential` directly via
[signInWithCredential]", and that changes the UID
([FlutterFire user.dart L147–159](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/firebase_auth/firebase_auth/lib/src/user.dart#L147-L159)).
— **Confirmed**. This is exactly the spec's **non-empty cloud account** case (§6.3), and a
returning user on a new phone or a reinstall. In (A), keeping the local side in that case
means reading the anonymous user's data *before* switching and writing all of it again
under the cloud UID. Whatever is still queued under the anonymous UID afterwards stays on
disk and hidden. — **Inferred**.

**Bearing on [#20](https://github.com/kingdragonfly43/gym-notes/issues/20) (the
replaced-installation race):** the "per-account pending-write queue" assumption holds.
A replaced installation's queued writes are tied to its UID. Once it signs itself out,
they stop flushing, but they stay on disk. They will flush if the same account signs in
on that installation again. That happens after an account deletion too, which could
recreate documents a Cloud Function has just deleted, if the rules allow the write. —
**Inferred**.

## 3. Living with a long-disabled network and tens of thousands of pending writes

**Local query performance and the 2022 overlays change.** Before overlays, every local
read replayed the mutation queue over the cached documents. Overlays store a precomputed
per-document result of all pending mutations
([`document_overlays` table](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteSchema.java#L700-L716)).
The Firestore team's own benchmark in
[firebase-android-sdk#3161](https://github.com/firebase/firebase-android-sdk/pull/3161)
(November 2021) says: "with overlay the query performance stays constant as the number of
local mutations grows; while without overlay, the query performance grows linearly". In
their test, 100 queries took 4.5 s at 10,000 mutations and 4.7 s at 100,000 with
overlays, against 3.9 s rising to 15.4 s without. Writing 10,000 mutations took about
15 s. The work shipped in Android 24.0.1 as "Optimized performance for offline usage"
([CHANGELOG L544–547](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/CHANGELOG.md?plain=1#L544-L547)).
— **Confirmed**. The iOS changelog has no matching entry, though iOS keeps the same
uid-keyed overlay data (§2). — **Inferred**.

**Collection scans.** "By default, the Firestore SDK scans all documents in a collection
in its local cache when executing offline queries". Automatic local indexing can be
turned on, but "Your app must enable automatic indexing each time it starts"
([Access data offline](https://firebase.google.com/docs/firestore/manage-data/enable-offline),
updated 2026-10-27). — **Confirmed**. A collection-group query over all 26,000 `sets`
documents, for example exercise history, is the case this hits. — **Inferred**.

**Memory.** Pending writes live on disk (SQLite or LevelDB), not in memory. Only up to 10
batches are held in the in-memory write pipeline
([RemoteStore.java L144–157](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/remote/RemoteStore.java#L144-L157)).
— **Confirmed**. On the Flutter side, every write's `Future` stays unresolved until the
server acknowledges it: "Once disabled, any writes will only resolve once connection has
been restored"
([firestore.dart L196–200](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/cloud_firestore/cloud_firestore/lib/src/firestore.dart#L196-L200)).
— **Confirmed**. The app must therefore never `await` a write in (A). Pending futures
accumulate within one process lifetime, but nothing is documented about their cost. —
**Inferred**.

**Disk.** Each pending write costs a mutation row, a `document_mutations` index row, an
overlay row and the cached document. No sizing figure is documented. — **Inferred**.

**The flush on `enableNetwork()`:**

- The write pipeline holds at most `MAX_PENDING_WRITES = 10` batches ("TODO: Negotiate
  this value with the backend"). It refills from the local queue as acknowledgements
  arrive in order
  ([RemoteStore.java L64–65, L614–653](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/remote/RemoteStore.java#L614-L653);
  iOS `kMaxPendingWrites = 10`,
  [remote_store.cc L60](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/Firestore/core/src/remote/remote_store.cc#L60)).
  Each batch is one `WriteRequest` on the write stream
  ([WriteStream.java L149–160](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/remote/WriteStream.java#L149-L160)).
  The SDK does not merge separate batches. — **Confirmed**.
- **The rate is undocumented.** About 10 batches per round trip suggests 26,000
  single-document writes would take several minutes on a good connection, and far fewer
  if writes are grouped into batches. — **Inferred** estimate. It is unmeasured.
- **Writes can be applied twice.** In July 2026 a Firebase engineer confirmed on
  [firebase-android-sdk#8440](https://github.com/firebase/firebase-android-sdk/issues/8440)
  that "The Firestore backend … currently does not support write stream resumption".
  "If a network drop or `disableNetwork()` occurs *after* the server commits a write but
  *before* the ACK reaches the client, the SDK re-transmits the unacknowledged batch …
  it executes the batch a second time." Replays of `set` are safe, but
  `FieldValue.increment` "applies the delta twice". The issue is open and tracked
  internally. — **Confirmed**. ADR-0002 names `increment` as an "offline-safe primitive",
  which needs revisiting. — **Inferred**.

**Firebase guidance against Firestore as an offline-only or primary local database:**

- The docs describe the cache as holding "data that your app is actively using"
  ([Access data offline](https://firebase.google.com/docs/firestore/manage-data/enable-offline)).
  — **Confirmed**.
- Gil Wilhuff of the Firestore client team was asked for local-only documents. He
  described a network-disabled secondary instance and added: "I'd consider this viable
  only in limited circumstances … The reason for this reservation is that the SDK will
  still track all the pending writes for this secondary instance and this queue will grow
  without bound. For a small few documents that are updated rarely this would work
  though."
  ([firebase-ios-sdk#6104](https://github.com/firebase/firebase-ios-sdk/issues/6104),
  2020-07-22, still open). — **Confirmed** (Firebase staff, GitHub).
- A Firebase DevRel Medium post, "Why is my Cloud Firestore query slow?", is commonly
  quoted as saying Firestore "was not designed as an 'offline first' database". It could
  not be fetched to verify the quote. — **Community**, unverified:
  [link](https://medium.com/firebase-developers/why-is-my-cloud-firestore-query-slow-e081fb8e55dd).
- A garbage-collection infinite loop occurred when "the mutation queue contains so many
  documents that all documents in cache also happen to be inside the mutation queue".
  It was confirmed by the Firestore team and fixed in Android 24.10.1
  ([firebase-android-sdk#5417](https://github.com/firebase/firebase-android-sdk/issues/5417)).
  — **Confirmed**. It shows how rarely this regime is exercised. — **Inferred**.

## 4. Cache eviction and whether the cache can be the only copy

- **Default size: 100 MB.** "After exceeding the default, Cloud Firestore periodically
  attempts to clean up older, unused documents". `CACHE_SIZE_UNLIMITED` is used "to
  disable clean-up"
  ([Access data offline](https://firebase.google.com/docs/firestore/manage-data/enable-offline);
  [FirebaseFirestoreSettings.java L30–36](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/FirebaseFirestoreSettings.java#L30-L36)).
  Garbage collection first runs 1 minute after start and then every 5 minutes
  ([LruGarbageCollector.java L31–34](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/LruGarbageCollector.java#L31-L34)).
  — **Confirmed**.
- **What is never evicted:** documents in an active target (a live listener), and any
  document with a pending mutation in *any* user's queue (`isPinned` →
  `mutationQueuesContainKey`,
  [SQLiteLruReferenceDelegate.java L132–150](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteLruReferenceDelegate.java#L132-L150)).
  Gil Wilhuff describes the policy as: "documents are deleted based on the least recently
  used query or write that has touched them"
  ([#6104](https://github.com/firebase/firebase-ios-sdk/issues/6104)). — **Confirmed**.
  **Synced, acknowledged documents not held by a listener can be evicted** once the cache
  passes the threshold. — **Confirmed**.
- **`CACHE_SIZE_UNLIMITED` does not guarantee a complete copy.** It stops eviction, but
  the cache only ever contains documents this installation has read or written. A
  restored account on a new phone holds only what its queries have fetched. Firestore
  queries are shallow, so subcollections must each be queried; Gil Wilhuff: "If there are
  subcollections within a user that you want to sync, sync them specifically"
  ([firebase-android-sdk#5](https://github.com/firebase/firebase-android-sdk/issues/5),
  the open request for a `keepSynced`-style API). The docs warn that cached results
  "might be stale or incomplete"
  ([Access data offline](https://firebase.google.com/docs/firestore/manage-data/enable-offline)).
  No API reports whether the cache is complete. — **Confirmed**.
- **Cost of keeping it full:** a listener disconnected for more than 30 minutes is billed
  "as if you had issued a brand-new query"
  ([pricing](https://firebase.google.com/docs/firestore/pricing)). A listener held on a
  26,000-document tree would be expensive to keep. — **Confirmed** rule; the cost is
  **Inferred**.

**Can the app rely on the cache as the only on-device copy of a signed-in user's data?**
Not safely for an offline export, or to keep the data after sign-out. Completeness is not
guaranteed, eviction is on by default, and after a sign-out the pending part of the data
moves into a hidden queue (§2). It could be made *mostly* complete with unlimited size plus
an online full read of every subcollection, but that is undocumented as a guarantee. —
**Inferred**.

## 5. Server-side deletes, sign-out, and wiping the cache

- **Active listeners receive deletes.** The listen stream reports removed documents, and
  when a resumed target's count disagrees (an "existence filter mismatch"), the SDK resets
  the target and re-runs the query
  ([WatchChangeAggregator.java L75–79, L195–230](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/remote/WatchChangeAggregator.java#L195-L230)).
  Documents a client still shows locally but cannot confirm are resolved through "limbo"
  lookups
  ([SyncEngine.java L89–103](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/core/SyncEngine.java#L89-L103)).
  — **Confirmed** as mechanism.
- **Cached documents outside any listener stay stale.** They are corrected only when a
  query or `get` that covers them goes to the server, or when garbage collection removes
  them. No documented call purges arbitrary cached documents, and Gil Wilhuff says
  "This isn't possible today"
  ([#6104](https://github.com/firebase/firebase-ios-sdk/issues/6104)). — **Confirmed**.
  If a Cloud Function deletes the user's tree, an installation that never re-runs those
  queries online keeps showing the deleted data. Its queued writes can also recreate
  documents. — **Inferred**.
- **Sign-out does not clear the cache.** "Cloud Firestore's cache isn't automatically
  cleared between sessions"
  ([Access data offline](https://firebase.google.com/docs/firestore/manage-data/enable-offline)).
  On a user change only the user-scoped parts are swapped (§2). The synced-document table
  is `remote_documents (path TEXT PRIMARY KEY, contents BLOB)`, with **no uid column**
  ([SQLiteSchema.java L363](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLiteSchema.java#L363)).
  — **Confirmed**.
- **A later user can read it.** Security rules are evaluated by the server. A cache read
  by user B of user A's paths is answered from that shared table. — **Inferred** from the
  schema; the docs do not state it in these words.
- **`clearPersistence()`** "Clears the persistent storage, including pending writes and
  cached documents. Must be called while the … instance is not started (after the app is
  shutdown or when the app is first initialized)". On startup it must come before every
  other method except settings, and it fails with `FAILED_PRECONDITION` if the instance is
  running. It is "primarily intended to help write reliable tests … does not attempt to
  securely overwrite … For applications that are sensitive to the disclosure of cached
  data in between user sessions, we strongly recommend not enabling persistence at all"
  ([FirebaseFirestore.java L656–675](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/FirebaseFirestore.java#L656-L675),
  same text in
  [FlutterFire L95–104](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/cloud_firestore/cloud_firestore/lib/src/firestore.dart#L95-L104)).
  — **Confirmed**.
- **`terminate()`**: "After calling terminate() only the clearPersistence() method may be
  used". It "does not cancel any pending writes … The next time you start this instance,
  it will resume attempting to send these writes". To restart, obtain a new instance
  ([FirebaseFirestore.java L580–601](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/FirebaseFirestore.java#L580-L601),
  [FlutterFire L295–308](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/cloud_firestore/cloud_firestore/lib/src/firestore.dart#L295-L308)).
  — **Confirmed**. So `terminate()` followed by `clearPersistence()` wipes *every* user's
  queue and cache in that instance. — **Inferred**.

## 6. Two Firestore instances side by side

- **Separate persistence is built in.** On Android the database file is named
  `firestore.{persistence-key}.{project-id}.{database-id}`, where the persistence key is
  the `FirebaseApp` name
  ([SQLitePersistence.java L66–84](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/local/SQLitePersistence.java#L66-L84),
  [FirebaseFirestore.java L205](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/FirebaseFirestore.java#L205)).
  On iOS the directory is `persistence_key / project_id[.database_id] / main`
  ([leveldb_opener.cc L162–179](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/Firestore/core/src/local/leveldb_opener.cc#L162-L179)).
  FlutterFire caches instances per `app.name|databaseId`, through
  `FirebaseFirestore.instanceFor(app:, databaseId:)`
  ([firestore.dart L34–50](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/cloud_firestore/cloud_firestore/lib/src/firestore.dart#L34-L50)).
  — **Confirmed**.
- **The Firestore team has suggested this pattern.** Gil Wilhuff: "Create a secondary
  Firebase App with a different name than the default; Create a secondary Firestore
  instance associated with that app; Call `disableNetwork` on that Firestore instance as
  the first thing you do with it"
  ([#6104](https://github.com/firebase/firebase-ios-sdk/issues/6104)). He has also
  suggested two instances with different persistence settings
  ([android#5](https://github.com/firebase/firebase-android-sdk/issues/5)). —
  **Confirmed**.
- **Caveats:**
  - **The queue grows without bound.** Gil Wilhuff's own warning, quoted in §3, applies.
    A heavy user's whole record is not "a small few documents that are updated rarely". —
    **Confirmed**.
  - **Moving the data to the cloud is a manual copy.** "If we did nothing to explicitly
    support this, the result is safe but somewhat clunky (to upload you'd have to read the
    local documents you want to upload and then write them)"
    ([#6104](https://github.com/firebase/firebase-ios-sdk/issues/6104)). — **Confirmed**.
  - **Auth is per app.** The secondary app's Firestore follows the secondary app's Auth,
    which has no user, so its writes queue under uid `""`. The iOS provider ignores Auth
    changes from other apps: "ensure we're only notifying for the current app"
    ([firebase_auth_credentials_provider_apple.mm L50–55](https://github.com/firebase/firebase-ios-sdk/blob/854b8aeab8e49b6ab9f61a03dd5bf9da6d9989df/Firestore/core/src/credentials/firebase_auth_credentials_provider_apple.mm#L50-L55)).
    — **Confirmed**. A user who signs in on the *default* app never affects the local
    instance's queue. — **Inferred**.
  - **Network order and settings.** Settings must be set "before invoking any other
    methods"
    ([firestore.dart L271–273](https://github.com/firebase/flutterfire/blob/24ff83a3a9ca4e4e62130cb5e5f89edbc4351c7f/packages/cloud_firestore/cloud_firestore/lib/src/firestore.dart#L271-L273)),
    and `disableNetwork` must be the first call. — **Confirmed**. As a further guard, the
    local instance could be pointed at an unreachable host, so that an accidental
    `enableNetwork` cannot upload anything. This is an idea, not documented practice. —
    **Inferred**.
  - **A named database instead of a second app** needs a database that exists on the
    server. Whether an instance with the network permanently off ever checks that is
    undocumented. The second-app route avoids the question. — **Inferred**.
  - **FlutterFire has had multi-instance bugs.** On Android, a setup with two database IDs
    failed with `ConcurrentModificationException` on the second launch until a fix in
    January 2026
    ([flutterfire#17954](https://github.com/firebase/flutterfire/issues/17954)). —
    **Confirmed**.

## 7. Bulk upload of about 26,000 documents at first sign-in

- **Batch size.** "Firestore no longer limits the number of writes that can be passed to
  a Commit operation or performed in a transaction" (release note, **2023-03-29**; the
  old limit was 500; request-size and transaction-time limits still apply)
  ([Firestore release notes](https://docs.cloud.google.com/firestore/docs/release-notes)).
  The [quotas page](https://firebase.google.com/docs/firestore/quotas) (updated
  2026-10-27) lists "Maximum API request size: 10 MiB" and no per-batch write count. The
  Android `WriteBatch` source has no client-side 500 check
  ([WriteBatch.java](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/WriteBatch.java)).
  Firebase still warns: "A batched write with hundreds of documents might require many
  index updates and might exceed the limit on transaction size"
  ([Transactions and batched writes](https://firebase.google.com/docs/firestore/manage-data/transactions)).
  — **Confirmed**. The common "500 operations" figure is out of date.
- **Throughput.** "We recommend starting with a maximum of 500 operations per second to a
  new collection and then increasing traffic by 50% every 5 minutes" (the 500/50/5 rule).
  If a sequentially increasing field such as a timestamp is indexed, "the maximum write
  rate to the collection is 500 writes per second" unless that field is exempted
  ([Best practices](https://firebase.google.com/docs/firestore/best-practices)).
  — **Confirmed**. A single client capped at 10 batches in flight is unlikely to reach
  those rates with small batches. — **Inferred**.
- **Recommended pattern.** "For bulk data entry, use a server client library with
  parallelized individual writes", and "You should use a server client library for bulk
  data operations and not a mobile/web SDK"
  ([Transactions and batched writes](https://firebase.google.com/docs/firestore/manage-data/transactions)).
  BulkWriter ("can be used to perform a large number of writes in parallel", retrying
  `UNAVAILABLE` and `ABORTED` up to 10 times) exists in the server libraries
  ([Node BulkWriter](https://docs.cloud.google.com/nodejs/docs/reference/firestore/latest/firestore/bulkwriter)).
  The FlutterFire, Android and iOS client APIs have no BulkWriter. — **Confirmed**. For
  this app, the realistic client options are batched writes, or individual `set`s with
  deterministic document IDs. — **Inferred**.
- **Billing.** Every document is one write regardless of how they are batched. 26,000
  writes is more than Spark's 20,000 per day
  ([quotas](https://firebase.google.com/docs/firestore/quotas)), which does not matter on
  Blaze. — **Confirmed**.
- **If the app is killed mid-upload:**
  - **Writes already committed through the SDK are safe.** They are written to the local
    mutation queue on disk at `commit()`, persist offline ("Unlike transactions, write
    batches are persisted offline",
    [WriteBatch.java L39](https://github.com/firebase/firebase-android-sdk/blob/91791120f81140fcd7b908d8c99b326b0ae72025/firebase-firestore/src/main/java/com/google/firebase/firestore/WriteBatch.java#L39)),
    and resume on the next start (`terminate()` text, §5). — **Confirmed**.
  - **Batches whose acknowledgement was lost may be re-applied** (§3, #8440). This is
    safe for plain `set` of absolute values. — **Confirmed**.
  - **Writes not yet handed to the SDK are lost with the process.** These are the rows the
    app had not yet read from its local database and written to Firestore. The copy must
    therefore be resumable, using deterministic document IDs and idempotent `set`s, with
    progress tracked in the local database. A 26,000-document copy also costs roughly
    tens of seconds of local SDK work before any upload: the overlay benchmark's 10,000
    mutations took about 15 s. — **Inferred**.
  - **A copy that runs while the UID is switching** (§2) must finish handing writes to the
    SDK *after* the switch, so that they land in the new user's queue. — **Inferred**.

---

## Decisions this forces

For the human, in the grilling for #31. None of these is decided here.

1. **(A) with no Auth user until sign-in** puts every pre-sign-in write in the uid `""`
   queue. At sign-in those writes vanish from view and never upload under the new UID.
   (A) therefore cannot work as "flip the network on at sign-in". It needs either a UID
   before the first write, which needs a connection on first launch (§1), or a read-out
   and rewrite at sign-in, which is (C) in disguise.
2. **(A) with an anonymous UID and the network disabled** holds only while
   `linkWithCredential` succeeds. The non-empty-cloud-account path (§6.3) and every
   sign-out force a UID change and strand the queue. Sign-out and "keep this phone's
   copy" (§6.4, §6.6) cannot rely on the cache either (§4–5).
3. **(A) also runs into Firebase's own warning** about network-disabled queues that
   "grow without bound", with up to 26,000 pending writes and an undocumented flush.
4. **(C) as two Firestore instances** is the pattern Firebase staff described, with their
   caveat that it is meant for small data. **(C) as drift or SQLite** avoids the unbounded
   queue. Both need the same resumable read-out and rewrite upload at sign-in (§7).
5. **(B)** avoids every UID-queue issue but owns the sync layer, including how deletions
   made in the cloud reach the phone (§5).
6. **Whatever is chosen**, ADR-0002's "`increment` is offline-safe" needs revisiting in
   light of duplicate replay (#8440). Account deletion should also expect a replaced
   installation's dormant queue to recreate documents if the same account signs in there
   again.
