# Cross-platform stack comparison

Research for [#4](https://github.com/kingdragonfly43/gym-notes/issues/4). This
gathers evidence; the decision is [#8](https://github.com/kingdragonfly43/gym-notes/issues/8).

Language familiarity is deliberately excluded as a criterion, per the map's
standing preference. Every argument below is about problem fit.

Backend choice is a separate research ticket
([#5](https://github.com/kingdragonfly43/gym-notes/issues/5)), so each stack is
assessed on *which backends it can reach and how well*, rather than assuming one.

Date of research: 2026-08-15. Version numbers and tier pricing move; re-check
before the decision hardens.

---

## The candidates

1. **Flutter** (Dart, Skia/Impeller-rendered UI)
2. **React Native / Expo** (TypeScript, native views, Expo tooling + EAS cloud builds)
3. **Kotlin Multiplatform + Compose Multiplatform** (Kotlin, shared logic + shared UI)

No fourth candidate is added. The two obvious extras were considered and
rejected before evaluation:

- **.NET MAUI** — fails the Windows constraint in a specific and worsening way.
  Hot Restart let you deploy a *debug* build to a locally attached iOS device
  from Visual Studio 2022 without a Mac build host, but it is limited to debug
  configuration and cannot handle static libraries, frameworks, XCFrameworks or
  binding resource packages — i.e. exactly the things AdMob and Billing SDKs
  ship as. It is also **not supported in Visual Studio 2026**, where Microsoft
  directs you back to Pair to Mac. Publishing always required a network-reachable
  Mac.
  ([Hot restart](https://learn.microsoft.com/en-us/dotnet/maui/ios/hot-restart?view=net-maui-10.0),
  [Publish for iOS](https://learn.microsoft.com/en-us/dotnet/maui/ios/deployment/?view=net-maui-9.0))
- **Capacitor / Ionic** — a WebView UI is the wrong bet for the single most
  performance-critical surface in this app (the set-entry hot path, ticket #10),
  and it inherits the same iOS-toolchain problem as Flutter without the
  compensating cloud-build ecosystem.

---

## Scorecard

Read this as a map of where the risk is, not as a sum to be totalled.

| Criterion | Flutter | React Native / Expo | KMP + Compose MP |
|---|---|---|---|
| Real-time backend SDKs w/ offline + reconnect | **Strong** — Firebase is first-party | **Strong** — Firebase via officially-recommended community SDK; best JS-backend reach | **Weak-to-medium** — no first-party Firebase; Supabase is community |
| Local-first database | **Strong** — Drift/sqflite, PowerSync stable | **Strong** — expo-sqlite (first-party) + Drizzle, PowerSync stable | **Strong** — Room 3.0 KMP, SQLDelight, PowerSync stable |
| One-time non-consumable IAP | **Strong** — first-party `in_app_purchase` | **Good** — mature community (`expo-iap` / `react-native-iap`) or RevenueCat | **Good** — RevenueCat's own KMP SDK; otherwise hand-rolled expect/actual |
| AdMob banner + UMP + ATT | **Strong** — Google-maintained, Google-documented | **Good** — Invertase, Expo config plugin, UMP + ATT covered | **Weak** — no Google SDK; expect/actual or small third-party wrappers |
| Camera QR scanning | **Strong** — `mobile_scanner` (MLKit/Vision) | **Strong** — built into first-party `expo-camera` | **Medium** — several small community libs, none dominant |
| Ecosystem for a solo dev + coding agents | **Strong** — official MCP server, agent skills, llms.txt | **Strong** — official agent skills, MCP, llms.txt, and an explicit "LLMs get Expo wrong" corrective | **Medium** — good language docs, thin third-party surface for agents to imitate |
| **iOS build/release from Windows** | **Medium** — cloud build works, zero local iOS iteration | **Strong** — designed for it; cloud build *and* a real device loop | **Blocked locally** — macOS+Xcode required by design; CI-only |

---

## Criterion 1 — Real-time backend clients with offline persistence and reconnection

The app needs a recorder's set to land on the owner's phone within seconds
(map: "Live sync, not merge-at-end"), and it must degrade to offline without
losing writes.

**Firebase / Firestore.** Firebase's own docs list first-party SDKs for Apple,
Android, **Web, Flutter, C++, Unity and server** — React Native and Kotlin
Multiplatform are not on that list
([Firebase docs](https://firebase.google.com/docs)).

- *Flutter* gets a genuine first-party SDK. Firestore offline persistence is
  **enabled by default on Android and Apple platforms**, and on reconnect the
  client syncs local changes automatically with last-write-wins on conflicts
  ([Enable offline data](https://firebase.google.com/docs/firestore/manage-data/enable-offline)).
  This is precisely the "usable offline, converges on reconnect" behaviour the
  app needs, for free.
- *React Native* uses **React Native Firebase**, which its own docs describe as
  "the officially recommended collection of packages that brings React Native
  support for all Firebase services" ([rnfirebase.io](https://rnfirebase.io/)).
  It wraps the same native Android/iOS SDKs, so it inherits the same
  default-on offline persistence. It is Invertase-maintained, not Google-maintained
  — a real but well-mitigated dependency, given how long it has been the de facto
  standard.
- *KMP* has **no official Firebase SDK**. The practical option is GitLive's
  `firebase-kotlin-sdk`, an unofficial community wrapper. Its own README states
  "all the project maintainers are volunteers, they are not paid to maintain
  this project", and API coverage varies wildly by service — as high as 90% for
  Installations, as low as 1% for Cloud Messaging and Performance
  ([GitLiveApp/firebase-kotlin-sdk](https://github.com/GitLiveApp/firebase-kotlin-sdk)).
  You can drop through to the platform SDKs via `.android` / `.ios` extension
  properties for anything unwrapped — which means writing expect/actual code
  twice for the parts that matter. **This is the single biggest ecosystem risk
  in the KMP option.**

**Supabase.** Supabase publishes an official Flutter/Dart client
([supabase-flutter reference](https://supabase.com/docs/reference/dart/introduction))
and an official JavaScript client. The **Kotlin client is explicitly
community-maintained**: "The Kotlin client library is created and maintained by
the Supabase community, and is not an official library"
([Kotlin reference](https://supabase.com/docs/reference/kotlin/introduction)).
`supabase-kt` is a well-regarded project with a single principal maintainer
(jan-tennert), but it is one person deep.

Note that Supabase Realtime alone does not give you offline persistence — that
needs a local cache or a sync engine on top.

**PowerSync** (the sync-engine route, backend-agnostic over Postgres) is the one
place KMP is *not* disadvantaged. Its SDK list marks Flutter/Dart, React Native
& Expo, Web, Swift and Kotlin all as fully supported, with .NET, Capacitor and
Node in beta
([client SDKs](https://docs.powersync.com/client-sdk-references/introduction)).
The Kotlin SDK is genuinely multiplatform — Android, JVM and Apple targets
(iOS, macOS, tvOS, watchOS) — and production-ready everywhere except web, which
is experimental
([Kotlin SDK](https://docs.powersync.com/client-sdk-references/kotlin-multiplatform)).
**If #5 lands on PowerSync + Postgres, KMP's biggest weakness largely evaporates.**
That coupling between #4 and #5 is worth flagging to the decision ticket.

## Criterion 2 — Local-first database

All three are fine here. This criterion does not discriminate.

- *Flutter*: `sqflite` and Drift (typed, reactive queries over SQLite) are both
  mature; PowerSync ships a stable Flutter SDK. Note that the previous
  "premium" option, Realm/Atlas Device SDK, was deprecated by MongoDB and is off
  the table for every stack.
- *React Native*: `expo-sqlite` is **first-party Expo**, has a full async API,
  documents Drizzle ORM integration, and supports SQLCipher encryption and
  libSQL sync
  ([expo-sqlite](https://docs.expo.dev/versions/latest/sdk/sqlite/)).
  Heavier alternatives (WatermelonDB, op-sqlite) exist if the set list needs
  them, though at this data scale — one user's lifting history — they are
  unlikely to be necessary.
- *KMP*: SQLDelight has been the mature multiplatform answer for years, and
  **Room now supports KMP first-class** — KMP support shipped in Room 2.7,
  and Room 3.0 (March 2026) makes Android/iOS/JVM/native co-equal targets with
  KSP-only, Kotlin-only code generation
  ([Room releases](https://developer.android.com/jetpack/androidx/releases/room),
  [Room for KMP](https://developer.android.com/kotlin/multiplatform/room)).

## Criterion 3 — One-time non-consumable unlock

The requirement is narrow: a single non-consumable "remove ads" SKU, restorable
after reinstall, on both stores. No subscriptions (ruled out in the map).

- *Flutter*: `in_app_purchase` is published under the **flutter.dev verified
  publisher** — first-party. It covers consumables, permanent upgrades and
  subscriptions, uses **StoreKit 2** on Apple platforms and Google Play Billing
  on Android, and targets iOS 13+ / Android SDK 24+
  ([pub.dev](https://pub.dev/packages/in_app_purchase)). A non-consumable is the
  easiest case it handles: `restorePurchases()` is well-trodden.
- *React Native*: no first-party option. `expo-iap` (Expo Modules) and
  `react-native-iap` (Nitro Modules) are maintained **in parallel by the same
  author**, targeting the Expo and bare-RN ecosystems respectively; both are
  backed by StoreKit 2 and Play Billing and both support non-consumables
  ([expo-iap vs react-native-iap](https://github.com/hyochan/react-native-iap/discussions/3105)).
  For an Expo project, `expo-iap` is the documented fit. RevenueCat's
  `react-native-purchases` is the managed alternative.
- *KMP*: no first-party multiplatform billing API exists; the DIY route is
  expect/actual over Play Billing and StoreKit — two native integrations, i.e.
  the thing you adopted a cross-platform stack to avoid. The strong mitigation
  is that **RevenueCat ships its own official KMP SDK** (`purchases-kmp`),
  wrapping BillingClient and StoreKit behind one Kotlin API, with paywall UI
  available through Compose Multiplatform
  ([purchases-kmp](https://github.com/RevenueCat/purchases-kmp),
  [RevenueCat KMP docs](https://www.revenuecat.com/docs/getting-started/installation/kotlin-multiplatform)).
  That turns a hard gap into a vendor dependency — acceptable, but it means
  routing a one-time $3 unlock through a third party's backend, which is more
  machinery than a single non-consumable warrants.

## Criterion 4 — Banner ads: AdMob + GDPR/UMP + ATT

This is where the three stacks separate most sharply, and it matters because
monetization is half the v1 product (map: banner + one-time unlock).

- *Flutter*: **Google itself maintains and documents `google_mobile_ads`**,
  with the guide hosted on developers.google.com. It supports Android and iOS
  only (no web/desktop, which is irrelevant here), and Google's own Flutter
  privacy guide walks through the full UMP flow —
  `requestConsentInfoUpdate()`, `loadAndShowConsentFormIfRequired()`,
  `getPrivacyOptionsRequirementStatus()`, `canRequestAds()` — plus a dedicated
  "Present IDFA message" section for the ATT/IDFA path
  ([quick start](https://developers.google.com/admob/flutter/quick-start),
  [privacy](https://developers.google.com/admob/flutter/privacy)).
  This is the strongest position of any stack on any criterion in this document.
- *React Native*: `react-native-google-mobile-ads` by **Invertase**. It exposes
  an `AdsConsent` helper for the EEA consent flow and documents the iOS ATT
  authorization request, and it ships an **Expo config plugin**, so it works in
  a managed Expo project without ejecting
  ([docs](https://docs.page/invertase/react-native-google-mobile-ads)).
  Community-maintained, but by the same organisation behind React Native
  Firebase, and it is the unambiguous default.
- *KMP*: **no Google SDK, official or otherwise.** The community pattern is an
  `expect @Composable fun AdMobBanner()` with `actual` implementations calling
  the Android and iOS AdMob SDKs directly, and there are a few small third-party
  wrappers of varying maturity. You are writing and maintaining the AdMob
  integration — including the UMP consent form lifecycle and the ATT prompt —
  twice, natively, against SDKs that update on Google's schedule. For an app
  whose entire free-tier revenue depends on that banner rendering correctly and
  consent being legally sound in the EEA, this is real, recurring, solo-owned work.

## Criterion 5 — Camera QR scanning

Used for one flow: scanning a friend's QR code to add them as a contact
(map: friendship via QR scan).

- *Flutter*: `mobile_scanner`, from a verified publisher, using **CameraX +
  MLKit on Android** and **AVFoundation + Apple Vision on iOS**; ~2.3k likes,
  160/160 pub points, 1.27M downloads
  ([pub.dev](https://pub.dev/packages/mobile_scanner)). Clear winner by
  concentration of usage.
- *React Native*: scanning is **built into first-party `expo-camera`** —
  `onBarcodeScanned` plus a `launchScanner()` that hands off to Google's code
  scanner on Android and Apple's `DataScannerViewController` on iOS 16+. QR is
  supported on both platforms, and the feature can be compiled out via
  `barcodeScannerEnabled: false` to save binary size
  ([expo-camera](https://docs.expo.dev/versions/latest/sdk/camera/)). Zero
  third-party dependency for this requirement.
- *KMP*: works, but is a choose-your-own-adventure among small libraries —
  QRKit, KScan (MLKit on Android, AVFoundation on iOS), Kamera, Camposer,
  EasyQRScan — with no dominant option and no first-party answer. Fallback is
  expect/actual over CameraX and AVFoundation.

## Criterion 6 — Ecosystem maturity for a solo dev working with coding agents

Two things matter: how much correct material an agent has seen, and whether the
platform gives agents live, authoritative context rather than relying on
training data.

**Flutter and Expo have both invested directly in this; KMP has not.**

- *Flutter*: Google's Dart and Flutter teams ship an **official MCP server**
  and **official agent skills/plugins** for Claude Code, Codex, Cursor and
  others, with docs published in llms.txt format
  ([Flutter & AI](https://docs.flutter.dev/ai),
  [MCP server](https://docs.flutter.dev/ai/mcp-server),
  [agent skills](https://docs.flutter.dev/ai/agent-skills)).
  An agent can query the analyzer and the running app, not just guess.
- *Expo*: publishes `llms.txt`, Expo Skills for AI agents, MCP integration and
  agent toolkits — and notably opens its AI guidance with a warning that "AI
  models and LLMs frequently provide outdated information about Expo", followed
  by corrections to the common misconceptions
  ([docs.expo.dev/llms.txt](https://docs.expo.dev/llms.txt)). That warning is a
  double-edged finding: Expo shipped the corrective *because* the problem is
  real. React Native's rapid churn (Expo Go vs dev builds, the New Architecture,
  the removal of `expo-ads-admob` and `expo-in-app-purchases`, the
  `expo-barcode-scanner` → `expo-camera` merge) means agents confidently emit
  code for APIs that no longer exist. The mitigation is good, but the underlying
  volatility is a standing tax.
- *KMP*: Kotlin's own language and JetBrains docs are excellent, and the
  first-party library situation improved substantially through 2025-26 (Room,
  DataStore, ViewModel, Lifecycle all now KMP-capable). But agent reliability
  tracks the volume of *correct public examples for the exact task*, and for
  "AdMob banner in Compose Multiplatform" or "restore a non-consumable in KMP"
  that volume is thin and largely blog-post-shaped. Expect agents to hallucinate
  plausible-looking expect/actual code more often here, and expect to be the one
  who verifies it — with no Mac to verify the iOS half on (see below).

Flutter's edge here is compounding: it has a large stable third-party ecosystem
*and* first-party agent tooling *and* less API churn than React Native.

## Criterion 7 — iOS build and release from a Windows machine

**This is the hard constraint, and it is the criterion that most changes the
ranking.** All three require macOS to *compile* iOS — that is Apple's rule, not
a framework limitation. The difference is entirely in how much of the loop each
ecosystem has moved off the local machine.

- *React Native / Expo* — **best by a wide margin.** EAS Build runs iOS builds
  on "macOS runners hosted in Expo's macOS cloud", automatically provisions and
  manages signing credentials, and auto-submits to the stores via EAS Submit
  with `--auto-submit`
  ([EAS Build](https://docs.expo.dev/build/introduction/)). Critically, this is
  not just a release pipeline: you can produce an iOS **development build**
  (`.ipa`) in the cloud from Windows and install it on a physical iPhone, then
  run the normal Metro dev loop against it — so day-to-day iteration on real
  iOS hardware is available. Caveats: you need paid Apple Developer Program
  enrolment (cloud signing requires it — there is no free-provisioning path
  without local Xcode), and a few device-side steps (enabling Developer Mode)
  are documented via Xcode and need working around. Free tier is **15 iOS
  builds/month**, 1 concurrency, low priority, 45-minute timeout; Starter is
  $19/mo ([Expo pricing](https://expo.dev/pricing)).
- *Flutter* — **workable for release, poor for iteration.** Flutter's install
  docs do not offer iOS as a target from Windows; iOS setup is documented only
  under macOS ([Flutter install](https://docs.flutter.dev/get-started/install)).
  Codemagic is the standard answer and handles automatic code signing —
  generating the certificate and provisioning profile on your behalf and
  shipping to the App Store as part of the build — but Codemagic's own writeup
  states the limitation plainly: "you will still need a Mac if you need to do any
  debugging of your app on an iOS Simulator or real device"
  ([Codemagic](https://blog.codemagic.io/how-to-build-and-distribute-ios-apps-without-mac-with-flutter-codemagic/)).
  So: you can *ship* to iOS from Windows, but every iOS-only bug becomes a
  push-wait-TestFlight cycle at ~10 minutes per iteration. Free tier is 500
  macOS M2 minutes/month, then $0.095/min
  ([Codemagic pricing](https://codemagic.io/pricing/)). GitHub Actions macOS
  runners are the DIY alternative.
- *KMP + Compose Multiplatform* — **the worst position.** Kotlin's own
  quickstart is unambiguous: "To create iOS applications, you need a macOS host
  with Xcode installed. Your IDE will run Xcode under the hood to build iOS
  frameworks", and "You will need to launch Xcode manually every time it is
  updated"
  ([KMP quickstart](https://kotlinlang.org/docs/multiplatform/quickstart.html)).
  The docs offer no non-Mac path at all. You *can* build the XCFramework on a
  macOS CI runner, but there is no equivalent of EAS's turnkey credential
  management and device-install loop, no first-party cloud build product, and
  the KMP build is Gradle+Xcode-shaped, which is fiddlier to run headless than
  a Flutter or Expo build. Worse, this compounds with Criterion 4: the AdMob
  and billing integrations you must hand-write for iOS are exactly the code
  you'd most want to debug locally, and you cannot.

  Compose Multiplatform for iOS itself is not the problem — it went **Stable and
  production-ready in 1.8.0 (May 2025)**, with finalized APIs, VoiceOver
  accessibility and type-safe navigation
  ([JetBrains blog](https://blog.jetbrains.com/kotlin/2025/05/compose-multiplatform-1-8-0-released-compose-multiplatform-for-ios-is-stable-and-production-ready/)).
  The framework is ready; the Windows workflow is not.

---

## Trade-offs, stated plainly

**Flutter's case.** It is the only stack where *every single feature this app
needs* is served by a first-party or Google-maintained package: Firebase,
`in_app_purchase`, `google_mobile_ads` with UMP and ATT documented by Google
itself, and an official MCP server plus agent skills. For a solo developer, that
means the fewest owners to depend on and the fewest integrations to maintain.
Its cost is the iOS dev loop: from Windows you get no simulator, no device
debugging, only cloud builds. Every iOS-specific defect is expensive.

**React Native / Expo's case.** It is the only stack that treats "no Mac" as a
supported configuration rather than an obstacle to route around. EAS gives you
cloud builds, managed credentials, auto-submit, *and* an installable dev build
on real iOS hardware — the difference between a 10-minute and a 10-second
iteration on iOS-only bugs. Every feature requirement is met, mostly by
first-party Expo modules (camera/QR, SQLite) with strong Invertase packages for
Firebase and AdMob. Its costs are ecosystem churn — deprecations and migrations
that agents reliably get wrong, which Expo acknowledges in its own AI docs — and
a soft dependency on Expo's hosted service and its free-tier limits.

**KMP + Compose Multiplatform's case.** Best-in-class local persistence (Room
3.0, SQLDelight), a genuinely stable iOS UI story since mid-2025, and the
cleanest escape hatch when you *do* need platform code. But it loses on three of
the seven criteria and one of them is the hard constraint. There is no
first-party Firebase, no Google AdMob SDK, no dominant QR library, no
first-party billing — so for a monetized, ad-supported, camera-using,
realtime-syncing app you are the integration layer, and you are writing that
integration for a platform you cannot build or run locally. **Unless #5 selects
PowerSync-over-Postgres (which neutralises the backend gap) and the ad
integration is deliberately scoped as accepted native work, the Windows
constraint alone makes this the hardest path.**

**On the #4/#5 coupling.** The backend decision materially changes this
comparison for exactly one candidate. Firebase widens Flutter's lead and hurts
KMP most; PowerSync levels the backend criterion across all three; Supabase sits
in between (official for Flutter and JS, community for Kotlin). Ticket #8 should
take #4 and #5 together rather than in sequence.

---

## Recommendation (advisory — the decision is #8)

**React Native / Expo, with Flutter as a close and defensible second.**

The tie-breaker is the constraint the ticket calls out as real. Flutter wins on
SDK quality — meaningfully, especially on AdMob, where Google writes both the
plugin and the docs. But Flutter's answer to "you develop on Windows" is "ship
it to a build farm and read the crash logs", while Expo's is a working device
loop. Over a solo-developer v1 that must ship a banner ad, a StoreKit
non-consumable, a camera permission flow and a live-sync session — four of the
most platform-divergent, most iOS-fiddly things a mobile app can contain — the
ability to actually run and debug on an iPhone from a Windows desk is likely
worth more than the marginal SDK quality it costs.

Flutter becomes the better answer if a Mac (even a used Mac mini, or a rented
cloud Mac) is acceptable — at which point its first-party coverage wins outright
and the ad and IAP integrations become the least risky of the three.

KMP should be ruled out for this app unless the Windows constraint is lifted.

---

## Sources

**Framework and build tooling**
- [Flutter install docs](https://docs.flutter.dev/get-started/install)
- [Flutter & AI](https://docs.flutter.dev/ai) · [MCP server](https://docs.flutter.dev/ai/mcp-server) · [Agent skills](https://docs.flutter.dev/ai/agent-skills)
- [EAS Build introduction](https://docs.expo.dev/build/introduction/) · [Expo pricing](https://expo.dev/pricing) · [Expo llms.txt](https://docs.expo.dev/llms.txt)
- [Kotlin Multiplatform quickstart](https://kotlinlang.org/docs/multiplatform/quickstart.html)
- [Compose Multiplatform 1.8.0 — iOS stable](https://blog.jetbrains.com/kotlin/2025/05/compose-multiplatform-1-8-0-released-compose-multiplatform-for-ios-is-stable-and-production-ready/)
- [Codemagic: iOS without a Mac](https://blog.codemagic.io/how-to-build-and-distribute-ios-apps-without-mac-with-flutter-codemagic/) · [Codemagic pricing](https://codemagic.io/pricing/)
- [.NET MAUI hot restart](https://learn.microsoft.com/en-us/dotnet/maui/ios/hot-restart?view=net-maui-10.0) · [Publish .NET MAUI for iOS](https://learn.microsoft.com/en-us/dotnet/maui/ios/deployment/?view=net-maui-9.0)

**Backend / sync**
- [Firebase docs (supported platforms)](https://firebase.google.com/docs) · [Firestore offline persistence](https://firebase.google.com/docs/firestore/manage-data/enable-offline)
- [React Native Firebase](https://rnfirebase.io/)
- [GitLive firebase-kotlin-sdk](https://github.com/GitLiveApp/firebase-kotlin-sdk)
- [Supabase Flutter reference](https://supabase.com/docs/reference/dart/introduction) · [Supabase Kotlin reference](https://supabase.com/docs/reference/kotlin/introduction)
- [PowerSync client SDKs](https://docs.powersync.com/client-sdk-references/introduction) · [PowerSync Kotlin Multiplatform SDK](https://docs.powersync.com/client-sdk-references/kotlin-multiplatform)

**Local database**
- [expo-sqlite](https://docs.expo.dev/versions/latest/sdk/sqlite/)
- [Room releases](https://developer.android.com/jetpack/androidx/releases/room) · [Room for KMP](https://developer.android.com/kotlin/multiplatform/room)

**Purchases**
- [in_app_purchase (flutter.dev)](https://pub.dev/packages/in_app_purchase)
- [expo-iap vs react-native-iap](https://github.com/hyochan/react-native-iap/discussions/3105) · [expo-iap](https://github.com/hyochan/expo-iap)
- [RevenueCat purchases-kmp](https://github.com/RevenueCat/purchases-kmp) · [RevenueCat KMP install docs](https://www.revenuecat.com/docs/getting-started/installation/kotlin-multiplatform)

**Ads and consent**
- [AdMob Flutter quick start](https://developers.google.com/admob/flutter/quick-start) · [AdMob Flutter privacy/UMP](https://developers.google.com/admob/flutter/privacy)
- [react-native-google-mobile-ads](https://docs.page/invertase/react-native-google-mobile-ads)

**QR scanning**
- [mobile_scanner](https://pub.dev/packages/mobile_scanner)
- [expo-camera](https://docs.expo.dev/versions/latest/sdk/camera/)
- [KScan](https://github.com/ismai117/KScan) · [QRKit](https://github.com/Chaintech-Network/QRKitComposeMultiplatform) · [EasyQRScan](https://github.com/kalinjul/EasyQRScan)
