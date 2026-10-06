# Privacy policy and store data disclosures: research

Research for [#19](https://github.com/kingdragonfly43/gym-notes/issues/19). This
document gathers findings and the decisions they force. It is not a privacy policy and
makes no product decisions. Where a finding implies a decision, the implication is
stated and the decision is flagged for the human under **Decisions this forces**.

Every finding is graded:

- **Confirmed**: primary source (store policy, vendor documentation, statute,
  regulator guidance, court judgment), URL cited.
- **Community**: credible secondary source (press, practitioner write-up), URL cited.
- **Inferred**: reasoning from the sources above applied to this app, no URL of its own.

Research date: **2026-10-05**. Where a page shows an update date, it is given.

### Facts this research relies on (from earlier tickets, not re-litigated)

- Flutter client; Cloud Firestore in `us-central1` on the **Spark** plan; Cloud
  Functions need Blaze and are not in v1 ([ADR-0002](../adr/0002-flutter-and-firebase.md),
  [#18](https://github.com/kingdragonfly43/gym-notes/issues/18)).
- An **anonymous account** exists from first launch. A user who never signs in has
  **no cloud copy**: their user data lives only on the device
  ([ADR-0004](../adr/0004-force-update-over-schema-migration.md)). Signing in with Google
  or Apple links a credential and uploads the local user data
  ([#6](https://github.com/kingdragonfly43/gym-notes/issues/6)).
- **User data** = sessions, splits, exercise library and the global unit preference
  ([CONTEXT.md](../../CONTEXT.md)). Account, friends, friend requests, display name,
  aliases and split grants are outside it.
- Every set is its own Firestore document; a heavy user has roughly **26,000 set
  documents** after five years (ADR-0004).
- Ads: one AdMob interstitial at the start of rest, after a usage threshold; UMP consent
  in EEA/UK/CH; ATT prompt after the announcement card; max ad content rating PG,
  weight-loss and supplement categories blocked; one non-consumable **Remove Ads** IAP
  whose entitlement attaches to the account when signed in
  ([#11](https://github.com/kingdragonfly43/gym-notes/issues/11)).
- Apple **Individual** developer account; Play **Personal** account with a payments
  profile ([#18](https://github.com/kingdragonfly43/gym-notes/issues/18)).
- The export file is a **plaintext JSON** file of user data, delivered through the OS
  share sheet, containing nothing about friends
  ([#15](https://github.com/kingdragonfly43/gym-notes/issues/15)).
- **Assumed, not established:** the developer is based in the US and has no EU or UK
  establishment. Several GDPR findings below depend on this.

A note on the data model that matters for health-data status: under **bodyweight and
reps**, the weight field is the weight *added* to the body, not the user's body mass
(CONTEXT.md, Measurement type). **The app stores no body-mass measurement anywhere.**
Issue #15's description of the export ("bodyweight and added weight over years")
overstates what it contains. — **Inferred** from CONTEXT.md.

---

## Bottom line

1. **Both stores require a public privacy policy URL**, in the store metadata *and*
   inside the app. Play adds: active, public, non-geofenced, not a PDF, not editable, and
   it must name the app or the developer entity. — **Confirmed**.
2. **The data declarations are driven mostly by AdMob, not by Firebase.** AdMob
   collects and *shares* approximate location (IP), app interactions, diagnostics and
   device identifiers for advertising, analytics and fraud prevention. Firebase's
   footprint is small and is processor-side. The app's own contribution is the training
   record, which both stores classify as **Fitness**, plus name, email, user ID,
   purchase history and the friend graph. — **Confirmed** (vendor data) /
   **Inferred** (mapping).
3. **Account deletion on Play must also be reachable outside the app**: a web resource
   where someone who has uninstalled can request deletion without reinstalling. Apple
   requires deletion to be *initiated* in the app; a web hand-off is allowed only as a
   direct link. — **Confirmed**.
4. **Deleting a heavy user's cloud copy exceeds the project's whole daily Firestore
   delete quota.** About 26,000 set documents against 20,000 free deletes a day, and
   Firestore does not cascade deletes to subcollections. On Spark, with no Cloud
   Functions, deletion must be client-driven and may have to span days. Apple allows
   this only if the user is told how long it takes and gets a confirmation when it is
   done. — **Confirmed** (quota, cascade, Apple rule) / **Inferred** (collision).
5. **The training record is Fitness data for both stores, and a grey area for GDPR.**
   It is not inherently health data. But the Article 29 Working Party said in 2015 that
   "several years' worth" of quantified-self records including exercise habits is health
   data, and the CJEU now treats data that can reveal health "by an intellectual
   operation involving collation or deduction" as health data. A conservative reading
   treats the cloud copy as Article 9 data needing **explicit consent**. — **Confirmed**
   (sources) / **Inferred** (application).
6. **GDPR almost certainly needs an EU representative (Art. 27), and UK GDPR a UK
   one**, if the app is offered on EEA/UK storefronts and the developer has no
   establishment there. The "occasional processing" exemption cannot apply to an
   app's core processing, and health-data status would block it a second time.
   — **Confirmed** (law, EDPB) / **Inferred** (application).
7. **"No age gate" survives, but "no age signal" does not, in the US.** Texas SB 2420
   has been in force since **4 June 2026**, Utah since 6 May 2026 and Louisiana since
   1 July 2026. Each requires developers to use the *store-supplied* age category and
   parental-consent signals (Apple Declared Age Range API, Significant Change API,
   consent-revocation notifications; Play Age Signals API). These are store APIs, not an
   onboarding questionnaire. — **Confirmed**.
8. **AdMob replaced the child-directed and under-age-of-consent tags with one Tag for
   Age Treatment (`CHILD` / `TEEN` / `UNSPECIFIED`)**, and the Flutter plugin exposes it.
   `TEEN` turns off personalised ads and applies Google's teen ad protections, so the
   store's age signal can drive ad treatment without the app asking anyone's age.
   — **Confirmed**.
9. **The plaintext export survives, with three caveats.** No store rule says anything
   about export format or protection. GDPR's portability guidance names JSON as a
   suitable format, says security measures "must not be obstructive", and puts the
   storage of a delivered file on the data subject, who "should be made aware". Health
   status does not change this. The caveats: tell the user the file is unencrypted;
   hand it over from app-private storage only; and accept that it is not, on its own,
   a complete GDPR access or portability answer for a signed-in user. See
   **The plaintext export: verdict**.
10. **Publishing obligations reach the developer's own identity.** EU traders on the
    App Store have their address, phone and email shown publicly, and monetising
    apps are likely traders. Play merchant accounts (apps with IAP) "must show their full
    address". — **Confirmed**.

---

## 1. What each store demands: data declarations

### 1.1 Apple App Privacy ("nutrition labels")

- Privacy details are "required to submit new apps and app updates", and the developer
  must "identify all of the data you or your third-party partners collect", where
  third-party partners include "advertising networks, third-party SDKs". —
  **Confirmed**, [App Privacy Details](https://developer.apple.com/app-store/app-privacy-details/)
- "Collect" means "transmitting data off the device in a way that allows you and/or your
  third-party partners to access it for a period longer than what is necessary to
  service the transmitted request in real time". "Data that is processed only on device
  is not 'collected'". — **Confirmed**, same page.
  - *Implication:* a never-signed-in user's training record is **not collected**,
    because it never leaves the device. It becomes collected at sign-in, when it uploads
    to Firestore. The label must declare it all the same, because the label describes the
    app, not one user. — **Inferred**.
- Relevant data types, as Apple defines them (all **Confirmed**, same page):
  - **Fitness**: "Fitness and exercise data, including but not limited to the Motion and
    Fitness API".
  - **Health**: "Health and medical data, including … any other user provided health
    or medical data".
  - **Contacts**: "Such as a list of contacts in the user's phone, address book, or
    **social graph**".
  - **User ID**: "screen name, handle, account ID, assigned user ID…"; **Device ID**:
    "the device's advertising identifier, or other device-level ID".
  - **Purchase History**, **Coarse Location**, **Product Interaction**, **Advertising
    Data**, **Crash Data**, **Performance Data**, **Other User Content**, **Photos or
    Videos**, **Name**, **Email Address**.
- "Tracking" means linking app data with third-party data for targeted advertising or
  measurement, or sharing it with a data broker. "Personal Data … as defined under
  relevant privacy laws, are considered linked to the user". — **Confirmed**, same page.

**SDK disclosures that feed the Apple label**

| SDK | What the vendor says it collects | Grade / source |
|---|---|---|
| Google Mobile Ads (AdMob) | IP address ("may be used to estimate the general location"), device ID ("advertising identifier or other app- or developer-bounded device identifiers"), advertising data, product interactions, crash logs ("non-user related"), performance data; purposes third-party advertising, analytics, product improvement. Google does not break down linked-to-user or tracking per data type, and leaves verifying the SDK's privacy manifest and the App Store answers to the developer. Updated 2026-10-02. | **Confirmed**, [AdMob iOS data disclosure](https://developers.google.com/admob/ios/privacy/data-disclosure) |
| Firebase Authentication | "Generates and stores identifiers for user authentication purposes"; usage-dependent: display names, email addresses, "third-party provider contact info". | **Confirmed**, [Firebase App Store data collection](https://firebase.google.com/docs/ios/app-store-data-collection) (updated 1 Oct 2026) |
| Cloud Firestore | Firebase user agent only; the developer is responsible for what it stores. | **Confirmed**, same page |
| Firebase Installations | Firebase user agent. | **Confirmed**, same page |
| Crashlytics / Analytics / Remote Config | Only if adopted. Crashlytics always collects stack traces and device state; Analytics has its own article. Not decided for v1. | **Confirmed**, same page |

Firebase's own warning: its manifests cover data "always collected and collected by
default. **It is your responsibility to ensure your privacy nutrition labels are
accurate based on your app's actual usage of Firebase.**" — **Confirmed**, same page.

**Draft mapping for Gym Notes (all Inferred, to be re-checked against the shipped
SDK versions):**

| Apple data type | Why | Linked | Tracking | Purpose |
|---|---|---|---|---|
| Fitness | Sessions, sets and steps in Firestore for signed-in users | Yes | No | App Functionality |
| Other User Content | Free-text exercise notes and agendas | Yes | No | App Functionality |
| Name, Email Address | Display name and provider email held by Firebase Auth | Yes | No | App Functionality |
| User ID | Firebase UID | Yes | No | App Functionality |
| Contacts | The friend graph is a "social graph" under Apple's definition | Yes | No | App Functionality |
| Photos or Videos | Only if an avatar is uploaded rather than taken from the provider | Yes | No | App Functionality |
| Purchase History | Remove Ads entitlement attached to the account | Yes | No | App Functionality |
| Device ID | AdMob (IDFA when ATT is granted; app-bounded IDs otherwise) | Yes | **Yes, if ATT granted** | Third-Party Advertising, Analytics |
| Coarse Location | AdMob IP-derived location | per AdMob | per ATT | Third-Party Advertising, Analytics |
| Product Interaction, Advertising Data | AdMob | per AdMob | per ATT | Third-Party Advertising, Analytics |
| Crash Data, Performance Data | AdMob (and Crashlytics if adopted) | per vendor | No | Analytics, App Functionality |
| Health | **Not declared** unless the app solicits health data; see §3 | n/a | n/a | n/a |

**Privacy manifests.** Apple requires a privacy manifest (and a signature, for binary
dependencies) for each SDK on its list "when you submit new apps". The list includes
FirebaseAuth, FirebaseCore, FirebaseFirestore, FirebaseInstallations,
FirebaseCrashlytics, GoogleSignIn, GoogleUtilities and **Flutter**. Google-Mobile-Ads-SDK
is not on the list, but Google says versions 11.2.0 and later ship a privacy manifest.
— **Confirmed**, [Apple third-party SDK requirements](https://developer.apple.com/support/third-party-SDK-requirements/),
[AdMob iOS data disclosure](https://developers.google.com/admob/ios/privacy/data-disclosure)

**Regulated medical device status (new, 2026).** Apps distributed in the EEA, UK or US
whose "primary or secondary category is Health & Fitness or Medical" must declare whether
they are a regulated medical device. New apps must do so from 26 March 2026; "If your app
is not a regulated medical device, you can select No." — **Confirmed**,
[Apple news, 26 Mar 2026](https://developer.apple.com/news/?id=nyqbfz1y)

**Age rating questionnaire (new, September 2026).** Responses to new questions on
"social media capabilities" are required on submission from September 2026. A social
media capability is "the ability to redistribute, amplify, or interact with user-generated
content through a social feed or similar discovery method". — **Confirmed**,
[Apple news, 9 Jul 2026](https://developer.apple.com/news/?id=tlur8uvi). Gym Notes has
friends but no feed and no discovery, so the answer is likely *no*. — **Inferred**.

### 1.2 Google Play Data safety

- "All developers that have an app published on Google Play must complete the Data
  safety form, including apps on closed, open, or production testing tracks." Only
  internal-testing-only apps are exempt. — **Confirmed**,
  [Data safety help](https://support.google.com/googleplay/android-developer/answer/10787469)
- Developers must disclose data collected and shared by SDKs. Collection means sending
  data off the device; on-device-only processing is not disclosed. Exemptions: ephemeral
  processing, **service providers** ("entities processing it on your behalf"),
  **user-initiated sharing** (with prominent disclosure), legal requests, and fully
  anonymised data. — **Confirmed**, same page.
- Data types: **Fitness info** is "Information about a user's fitness, such as exercise
  or other physical activity"; **Health info** is "medical records or symptoms". The form
  asks whether data is encrypted in transit and whether users can request deletion. A
  privacy policy link is required to publish it. — **Confirmed**, same page.

| SDK | Play disclosure (vendor) | Grade / source |
|---|---|---|
| Google Mobile Ads 25.5.0 | IP address, user product interactions, diagnostic information, device and account identifiers (AAID, app set ID): each **collected and shared**, for "advertising, analytics, and fraud prevention"; encrypted in transit with TLS; ad ID collection can be prevented in the manifest; "Limited Ads … may also disable transmission of the ad ID". Updated 2026-10-02. | **Confirmed**, [AdMob Play data disclosure](https://developers.google.com/admob/android/privacy/play-data-disclosure) |
| Firebase Authentication | Firebase user agent, "IP addresses to provide added security and prevent abuse during sign-up", user agent strings, app ID. | **Confirmed**, [Firebase Play data disclosure](https://firebase.google.com/docs/android/play-data-disclosure) (updated 2026-10-01) |
| Cloud Firestore | Firebase user agent; "If an end-user is signed-in, then every request automatically includes the User ID". | **Confirmed**, same page |
| Firebase Installations | FID ("does not uniquely identify a user"), user agent. | **Confirmed**, same page |
| All Firebase | "Firebase encrypts the data in transit using HTTPS" and transfers to third parties only to subprocessors. | **Confirmed**, same page |

**Draft mapping for Gym Notes (Inferred):** Personal info (name, email address, user
IDs): collected, not shared, for app functionality and account management. Health and
fitness (**Fitness info**): collected, not shared, for app functionality. Financial info
(purchase history): collected, for app functionality. App activity (app interactions;
other user-generated content for notes): collected; AdMob's portion is **shared**.
Location (approximate, from AdMob IP): collected and shared. App info and performance
(crash logs, diagnostics): collected and shared (AdMob). Device or other IDs: collected
and shared (AdMob). Encrypted in transit: **yes**. Deletion request: **yes**, with the
deletion URL (§4).

- Firebase counts as a **service provider**, so Firebase's processing is *collection*,
  not *sharing*. — **Inferred** from the service-provider exemption and Firebase's
  processor role (§5).
- What friends see (display name, avatar, a granted split) falls under the
  **user-initiated sharing** exemption, not "shared". — **Inferred**.
- Play has no "social graph" category. Its Contacts type is the device's contact list,
  which the app never reads. — **Inferred**.

**Health apps declaration.** "All developers that have an app published on Google Play
must complete the Health apps declaration", and "after August 31, 2024, all apps will be
required to have completed an accurate Health apps declaration". Categories include
**Activity and Fitness**. — **Confirmed**,
[Health apps declaration](https://support.google.com/googleplay/android-developer/answer/14738291).
Gym Notes would declare Activity and Fitness. — **Inferred**.

**Health Content and Services policy.** It requires a privacy policy in Play Console and
in the app, and says health and medical apps not regulated as devices "must include a
clear disclaimer in their app description" that the app is "not a medical device and
does not diagnose, treat, cure, or prevent any medical condition". — **Confirmed**,
[Health Content and Services](https://support.google.com/googleplay/android-developer/answer/16679511).
Whether this binds a pure Activity and Fitness app with no medical function is not clear
from the page. Adding the sentence costs nothing. — **Inferred**.

**Prominent disclosure** is needed only "in cases where your app's access, collection,
use, or sharing of personal and sensitive user data may not be within the reasonable
expectation of the user". Health data is on Play's list of personal and sensitive data.
— **Confirmed**, [User Data policy](https://support.google.com/googleplay/android-developer/answer/10144311).
Uploading the training record at sign-in is expected, because the sign-in screen sells
free backup. The prominent-disclosure requirement is therefore probably met if that
screen *says* the record goes to the cloud. — **Inferred**.

### 1.3 Sign-in providers

- Apple 4.8: an app using a third-party login (Google Sign-In is named) for the primary
  account must also offer a login that limits collection to name and email, lets the user
  keep the email private, and does not collect interactions for advertising without
  consent. Sign in with Apple satisfies this, and the app already plans both. —
  **Confirmed**, [App Review Guidelines 4.8](https://developer.apple.com/app-store/review/guidelines/)
- Firebase Auth stores what the provider returns: email (for Apple, possibly a private
  relay address), display name, provider identifiers. Hence Name, Email and User ID in
  both forms. — **Confirmed** (Firebase usage-dependent list above) / **Inferred** (mapping).
- Sign in with Apple relay addresses are moving from `privaterelay.appleid.com` to
  `private.icloud.com`, and developers "must accept both domains". — **Confirmed**,
  [Apple news, 24 Aug 2026](https://developer.apple.com/news/?id=1ptvdtcm). This is not a
  disclosure item, but it touches any email-handling code.

---

## 2. The privacy policy: mandatory, what it must say, where it lives

### 2.1 Mandatory on both stores

- Apple 5.1.1(i): "All apps must include a link to their privacy policy in the App Store
  Connect metadata field **and within the app in an easily accessible manner**."
  App Store Connect lists "Privacy Policy (Required): The URL to your publicly accessible
  privacy policy." — **Confirmed**,
  [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/),
  [App Privacy Details](https://developer.apple.com/app-store/app-privacy-details/)
- Play: "link in the designated field within Play Console, and a privacy policy link or
  text within the app itself", on "an active, publicly accessible and non-geofenced URL
  (no PDFs) and is non-editable". — **Confirmed**,
  [User Data policy](https://support.google.com/googleplay/android-developer/answer/10144311)
- Apple 5.1.4: apps that collect personal information "from a minor must include a privacy
  policy and must comply with all applicable children's privacy statutes". —
  **Confirmed**, App Review Guidelines.

### 2.2 What it must contain

| Source | Required content | Grade |
|---|---|---|
| Apple 5.1.1(i) | What data is collected, how, and all uses; confirmation that third parties (analytics, ad networks, SDKs) "will provide the same or equal protection"; retention and deletion policies, and how a user can revoke consent or request deletion. | **Confirmed**, App Review Guidelines |
| Play User Data | Developer information and a contact mechanism; types of personal and sensitive data accessed, collected, used and shared; parties it is shared with; secure handling; retention and deletion; "clear labeling as a privacy policy"; the entity named on the store listing (or the app) must appear in it. | **Confirmed**, User Data policy |
| Play account deletion | Retention practices for data kept after deletion ("for example, within your privacy policy"). | **Confirmed**, [account deletion help](https://support.google.com/googleplay/android-developer/answer/13327111) |
| GDPR Art. 13 | Controller identity and contact details **and the representative's**; purposes and legal basis for each; legitimate interests if relied on; recipients; third-country transfers and safeguards; retention period or criteria; rights of access, rectification, erasure, restriction, objection and **portability**; right to withdraw consent; right to complain to a supervisory authority; whether data is required; automated decision-making. | **Confirmed**, [GDPR Art. 13](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679) |
| AdMob / EU User Consent Policy | Publishers must provide disclosures, link to Google's page on "how Google manages data in its ads products", gather consent and identify the parties. Google and the publisher are **independent controllers** for ads personalisation. | **Confirmed**, [AdMob help 7666366](https://support.google.com/admob/answer/7666366) |
| Firebase Auth retention | Auth data "removed from live and backup systems within 180 days". This belongs in the retention section. | **Confirmed**, [Firebase privacy](https://firebase.google.com/support/privacy) (updated 15 Sep 2026) |
| Texas SB 2420 §121.053 | The developer "shall provide notice to each app store … before making any significant change to the terms of service or privacy policy". | **Confirmed**, [SB 2420 enrolled](https://capitol.texas.gov/tlodocs/89R/billtext/html/SB02420F.htm) |

- *Implication:* policy changes become a release-process item in the US, not just an
  edit to a web page. — **Inferred**.
- *Implication:* the policy has to describe two populations. Anonymous users' user data
  never leaves the device. Signed-in users' user data is stored in Firestore
  (`us-central1`, Google as processor). — **Inferred**.

### 2.3 Where a solo developer hosts it

- The repo `kingdragonfly43/gym-notes` is **public**, so GitHub Pages is available on a
  free plan. GitHub's own limits forbid using Pages as hosting for an online business,
  e-commerce site or SaaS, but a static policy page is none of these. — **Confirmed**
  (repo visibility via `gh repo view`; [GitHub Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits))
  / **Inferred** (fit).
- **Firebase Hosting runs on Spark** at no cost, up to 10 GB of storage and 10 GB of
  transfer a month. If the transfer limit is exceeded, "your sites will be disabled"
  until the next month. — **Confirmed**,
  [Firebase Hosting quotas](https://firebase.google.com/docs/hosting/usage-quotas-pricing)
  (updated 2026-10-01). That failure mode would take down the policy URL and the
  deletion page together, though a text page will not come near 10 GB. — **Inferred**.
- Play's "non-editable" rule excludes a publicly editable document such as an open
  Google Doc. "No PDFs" excludes a PDF link. — **Inferred** from the User Data policy.
- The same host should carry the **account-deletion web resource** (§4.2), since Play
  requires one URL for it. — **Inferred**.

### 2.4 What else gets published: the developer's own details

- **Apple, EU trader status.** Every developer must declare trader status. Traders
  distributing in the EU provide, as individuals, an "Address or P.O. Box", phone number
  and email address, and "Apple will publish this information on your App Store product
  page". Apple lists revenue "if your app includes In-App Purchases, or if it's a paid or
  ad-sponsored app" as an indicator of trader status. — **Confirmed**,
  [Apple DSA trader requirements](https://developer.apple.com/help/app-store-connect/manage-compliance-information/manage-european-union-digital-services-act-trader-requirements/)
- **Play, merchant address.** "Merchant accounts (developer accounts with apps that
  monetize via paid apps or in-app purchases) must show their full address on Google
  Play". Otherwise personal-account contact details are "only used by Google to contact
  you". — **Confirmed**,
  [Play account information](https://support.google.com/googleplay/android-developer/answer/13634081)
- *Implication:* the Remove Ads IAP puts the developer's address on Play. Selling in
  the EU on the App Store probably does the same there. A P.O. box or virtual address is
  the usual answer. — **Inferred**.

---

## 3. Is any of this health data?

### 3.1 Under the stores' rules

- **Apple** separates *Fitness* ("exercise data") from *Health* ("health and medical
  data … or any other user provided health or medical data"). The training record is
  Fitness. — **Confirmed** (definitions) / **Inferred** (classification).
- Apple 5.1.3 still covers fitness: "Health, fitness, and medical data are especially
  sensitive". Under 5.1.3(i), apps "may not use or disclose to third parties data
  gathered in the health, fitness, and medical research context … for advertising,
  marketing, or other use-based data mining purposes", and must "disclose the specific
  health data that you are collecting". Under 5.1.3(ii), apps "may not store personal
  health information in iCloud". — **Confirmed**, App Review Guidelines.
  - *Implication:* **no training data may ever reach an ad request.** That means no
    keyword or content targeting built from exercises, categories or agendas, and no
    custom targeting parameters. Ads must remain untargeted by anything the user
    trained. — **Inferred**.
  - The iCloud clause targets the app storing data in iCloud. An export file the *user*
    sends to iCloud Drive through the share sheet is the user's act, and the training
    record is fitness rather than "personal health information". Risk is low. —
    **Inferred**.
- Apple 5.1.1(ix): apps "in highly regulated fields (such as … healthcare …) or that
  require sensitive user information should be submitted by a legal entity … and not by
  an individual developer". Gym Notes provides no healthcare service. The risk to an
  Individual account is low but not zero. — **Confirmed** (text) / **Inferred**
  (application).
- **Play** has *Fitness info* separate from *Health info* ("medical records or
  symptoms"). Gym Notes is Fitness, and Activity and Fitness in the Health apps
  declaration. — **Confirmed** (definitions) / **Inferred** (classification).

### 3.2 Under GDPR

- Art. 4(15): "data concerning health" means "personal data related to the physical or
  mental health of a natural person … which reveal information about his or her health
  status". Recital 35 covers "past, current or future" health status, "disease risk" and
  "the physiological or biomedical state of the data subject independent of its
  source". — **Confirmed**, [GDPR](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679)
- Art. 9(1) prohibits processing health data unless a 9(2) exception applies. The only
  realistic one for an app is 9(2)(a), **explicit consent** "for one or more specified
  purposes". — **Confirmed**, GDPR.
- **CJEU, broad reading.** Data "liable indirectly to reveal sensitive information" are
  not excluded from Art. 9 (C-184/20, para. 127). In *Lindenapotheke* (C-21/23, 4 Oct
  2024), "it is sufficient that they are capable of revealing information about the
  health status of the data subject by means of an intellectual operation involving
  collation or deduction" (para. 83). That holds "even where it is only with a certain
  degree of probability, and not with absolute certainty" (para. 90). — **Confirmed**,
  [C-184/20](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:62020CJ0184),
  [C-21/23](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:62023CJ0021)
- **Article 29 Working Party annex on lifestyle apps (2015).** A single walk's step count
  "will not qualify as health data". But "an app combining several years' worth of
  extensive quantified-self records of an individual (tracking, for example, sleep and
  exercise habits, detailed records of diet, weight, body mass index, blood pressure and
  other vital statistics, as well as a mood diary) will be processing health data", and
  then "not only the conclusions and inferences, but also the raw data will be considered
  health data" (fn. 5). Its test turns on whether conclusions about health status or
  health risk can be drawn, "irrespective of whether these conclusions are accurate". —
  **Confirmed**,
  [WP29 annex, 5 Feb 2015](https://ec.europa.eu/justice/article-29/documentation/other-document/files/2015/20150205_letter_art29wp_ec_health_data_after_plenary_annex_en.pdf).
  It was written under the old Directive, but GDPR's definition is the one the annex
  quotes in draft.
- **ICO** lists "data from medical devices or fitness trackers" among health-data
  examples, and says profiling that infers health status is processing special category
  data. — **Confirmed**,
  [ICO, what is special category data](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/special-category-data/what-is-special-category-data/)
  (updated 9 Apr 2024).
- **Pending law change.** The Commission's Digital Omnibus proposal (19 Nov 2025) adds
  narrow Art. 9 derogations, for residual data in AI training and for biometric
  verification under the user's sole control. The EDPB-EDPS Joint Opinion 2/2026 shows no
  change to the definition of health data, and the text was not adopted as of this
  research. — **Confirmed** (opinion, Feb 2026,
  [EDPB-EDPS 2/2026](https://www.edpb.europa.eu/system/files/documents/2026-02/edpb_edps_jointopinion_202602_digitalomnibus_en.pdf))
  / **Community** (adoption timeline, e.g.
  [White & Case](https://www.whitecase.com/insight-alert/gdpr-under-revision-key-takeaways-from-digital-omnibus-regulation-proposal)).

**Application to Gym Notes (Inferred):** the record is exercise habits over years,
with no body mass, heart rate, sleep, diet or mood. That puts it between WP29's two
poles: closer to the step counter than the quantified-self diary, but long-running and
per-person. Some signals point towards health data. Free-text notes can name an injury
("left knee: rehab"). Half reps and assisted reps (completion ratio) and a sudden halt
in sessions are what a deduction would work from. And *Lindenapotheke* lowers the bar to
"capable of revealing … with a certain degree of probability". **A conservative
controller treats the cloud copy of a signed-in EEA/UK user's user data as Article 9
data.** A defensible but riskier reading treats it as ordinary personal data, provided
the app never draws health conclusions from it.

### 3.3 What changes if it is health data

| Consequence | Grade |
|---|---|
| A lawful basis under Art. 9(2)(a) is needed: **explicit consent** to cloud storage, specific to that purpose, withdrawable, for EEA/UK users, at sign-in (when the record first leaves the device). Contract (Art. 6(1)(b)) alone would not be enough. | **Confirmed** (Art. 9) / **Inferred** (placement) |
| Withdrawing that consent means the cloud copy must go (Art. 17(1)(b)), which the existing "delete the cloud copy" control already does. | **Confirmed** (Art. 17) / **Inferred** |
| The Art. 27 "occasional" exemption from appointing an EU representative fails a second time (see §5.2). | **Confirmed** (Art. 27(2)(a)) |
| A DPIA becomes more likely. Art. 35(3)(b) mandates one for processing "on a large scale" of special categories, and a solo developer's user base is not large scale. Lower risk, but the ICO's Children's Code requires one where children are likely users (§6). | **Confirmed** (Art. 35) / **Inferred** (scale) |
| Apple and Play declarations do **not** change: still Fitness, not Health. | **Inferred** |
| The **export file** does not change (see the verdict). | **Inferred** |
| No change to ads: AdMob never receives training data either way (Apple 5.1.3 already forbids it). | **Inferred** |

### 3.4 US health-privacy laws (outside this ticket's stated scope; flagged)

- **Washington My Health My Data Act.** "Consumer health data" includes "bodily
  functions, vital signs, symptoms, or measurements of the information described in this
  subsection". A regulated entity is any entity that "conducts business in Washington, or
  produces or provides products or services that are targeted to consumers in
  Washington", with **no size threshold**; small businesses only had a later start date.
  Obligations include a consumer health data privacy policy linked from the homepage,
  consent to collect, separate consent to share, and rights of access and deletion. —
  **Confirmed**, [RCW 19.373.010](https://app.leg.wa.gov/RCW/default.aspx?cite=19.373.010),
  [WA AG](https://www.atg.wa.gov/protecting-washingtonians-personal-health-data-and-privacy).
  Whether a strength-training record counts is unresolved. — **Inferred**.
- **FTC Health Breach Notification Rule** (amended 2024). It covers personal health
  record vendors whose record "has the technical capacity to draw information from
  multiple sources". Disclosure "without their consent" counts as a breach, and sharing
  with ad networks is the FTC's own example. — **Confirmed**,
  [FTC HBNR guidance](https://www.ftc.gov/business-guidance/resources/complying-ftcs-health-breach-notification-rule-0).
  Gym Notes takes only manual entry, so it probably fails the multiple-sources test. A
  future HealthKit or Health Connect import would change that. — **Inferred**.

---

## 4. Account deletion

### 4.1 Apple

- 5.1.1(v): "If your app supports account creation, you must also offer account deletion
  **within the app**." — **Confirmed**, App Review Guidelines.
- From Apple's deletion guidance (**Confirmed**,
  [Offering account deletion](https://developer.apple.com/support/offering-account-deletion-in-your-app/)):
  - Make the option "easy to find in your app. Typically, it's included in the app's
    account settings."
  - "If people need to visit a website to finish deleting their account, include a link
    **directly** to the page on your website where they can complete the process."
  - Outside highly regulated industries, apps "should not require people to make a phone
    call, send an email, use a chat support flow, or go through other support flows."
  - "If your process for account deletion is manual or otherwise takes a reasonable
    amount of time to complete, this is acceptable. **Inform the person how long it will
    take** to delete the account and **provide a confirmation when the deletion is
    complete**."
  - "Apps that support Sign in with Apple should use the Sign in with Apple REST API to
    revoke user tokens."
  - Deletion "removes the account from the developer's records, along with any data
    associated with the account that the developer isn't legally required to maintain".
    If anything is retained, "let people know what information will and won't be
    retained".
- Apple imposes **no web-facing deletion obligation**. A website is allowed only as the
  finishing step, through a direct link. — **Confirmed** (by its absence from the
  guidance above).

### 4.2 Google Play

- Apps that let users create an account must offer deletion "from within your app **and
  outside of your app**". "Temporary account deactivation, disabling, or 'freezing' the
  app account does not qualify". — **Confirmed**,
  [User Data policy](https://support.google.com/googleplay/android-developer/answer/10144311)
- The web resource: "You can offer this in many ways, like an additional link that
  initiates account deletion, a **customer service email or a form**". It must be
  functional, prominently feature the deletion pathway, "reference the app or developer
  name", and "give users a way to request that their data be deleted **without sending
  the user back to the app and requiring them to re-download it**". The URL goes in a
  Play Console field. — **Confirmed**,
  [account deletion help](https://support.google.com/googleplay/android-developer/answer/13327111)
- "If your app relies on service providers to process user data, you should delete the
  data from your own servers and request the service provider to do the same." Data may
  be retained for "security, fraud prevention or regulatory compliance" if disclosed. —
  **Confirmed**, same page.
- *Implication:* a **public deletion page with an email address or form** satisfies
  Play. The developer then has to be able to delete a user's cloud data *without that
  user's device*. — **Inferred**.

### 4.3 What a Firestore-backed app on Spark has to be able to do

- "**Deleting a document does not delete its subcollections!**" Deleting a collection
  from mobile or web clients is "not recommended" because it "requires coordinating an
  unbounded number of individual delete requests". The recommended tools are the Admin
  SDK `recursiveDelete()`, the CLI `firebase firestore:delete`, and Cloud Functions. —
  **Confirmed**, [Firestore delete data](https://firebase.google.com/docs/firestore/manage-data/delete-data)
- Spark free quota: **20,000 document deletes per day** and 50,000 reads per day. The
  quotas page does not say what happens at the limit. — **Confirmed**,
  [Firestore quotas](https://firebase.google.com/docs/firestore/quotas) (updated 2026-10-01)
- Cloud Functions deploy only on Blaze: "to deploy functions, your project must be on the
  Blaze pricing plan". — **Confirmed**,
  [Functions get started](https://firebase.google.com/docs/functions/get-started)
- Account deletion in the client needs a recent sign-in. `user.delete()` otherwise fails
  with `requires-recent-login` and the user must re-authenticate. — **Confirmed**,
  [Firebase Auth manage users](https://firebase.google.com/docs/auth/flutter/manage-users)
- Apple token revocation is supported client-side.
  `revokeTokenWithAuthorizationCode()` takes the authorization code that Apple sign-in
  returns. — **Confirmed**,
  [Firebase federated auth (Flutter)](https://firebase.google.com/docs/auth/flutter/federated-auth)

**What follows (all Inferred):**

1. **The delete-quota collision.** A heavy user's cloud copy is at least ~26,000
   documents (ADR-0004's set count alone), which is more than the whole project's
   20,000 free deletes a day. It also needs at least ~26,000 reads to find the IDs,
   which is half the daily read quota. One deletion can therefore (a) fail to finish in a
   day and (b) consume the project's delete budget for every other user that day. This is
   the same arithmetic that ruled out rewrite migrations in ADR-0004, and it applies to
   deletion too. Apple's "reasonable amount of time … inform the person how long"
   provision allows a multi-day deletion, but a client-driven deletion that spans days
   cannot rely on the deleting device staying around.
2. **The client must know the whole tree.** Client SDKs cannot list subcollections, so
   the deletion code has to walk every path the schema defines. A path added later and
   forgotten in deletion leaves orphans. That is a maintenance rule for the schema, not
   a one-off.
3. **The Play web path does not need Blaze.** The developer can delete a user's data from
   their own Mac with the Admin SDK (`recursiveDelete`) or `firebase firestore:delete`,
   plus Firebase Auth user deletion. Neither needs Cloud Functions. Both still draw on
   Firestore quota.
4. **Sign in with Apple revocation** needs an authorization code, which exists only at
   sign-in. Re-running Sign in with Apple at the deletion step obtains a fresh code and
   satisfies `requires-recent-login` at the same time. A web-initiated deletion would
   need Apple's REST revocation with the developer's key, from the same local script.
5. **The granular deletion decided in #6 needs one clarification.** "Deleting the cloud
   side removes the profile and every friendship and returns the app to its anonymous
   state" has to mean, for Apple and Play, that the **Firebase Auth user** (with its
   linked Google or Apple credential) and **all cloud user data** (sessions, splits,
   exercise library, unit preference, entitlement record, friend requests, split
   grants) are deleted, and a *new* anonymous account takes its place. Unlinking the
   credential while keeping the same UID and its cloud documents would not count as
   deletion.
6. **No provenance makes erasure simpler.** Sets a deleted user entered as a
   **recorder** stay in the owner's sessions and carry no trace of who entered them
   ([#15](https://github.com/kingdragonfly43/gym-notes/issues/15),
   [ADR-0001](../adr/0001-one-writer-per-session.md)). There is nothing of the deleted
   user's to scrub from friends' records. Aliases *about* the deleted user live in
   friends' own data. Removing the friendship should remove them.
7. **Retention disclosure.** Firebase Auth data clears from backups "within 180 days"
   (§2.2). The policy should say so.

---

## 5. GDPR obligations beyond the UMP consent form

### 5.1 Does GDPR apply at all?

- Art. 3(2)(a): GDPR applies to a controller not established in the EU that processes
  the data of people in the EU when offering them goods or services, "irrespective of
  whether a payment … is required". Recital 23: "mere accessibility" of a site is not
  enough, but "the use of a language or a currency generally used in one or more Member
  States" can show intent. — **Confirmed**, GDPR.
- *Application:* listing the app on EEA storefronts, with EUR store prices for Remove
  Ads and a consent flow built for EEA users, is very likely "offering". UK GDPR mirrors
  this for the UK. — **Inferred**.
- **On-device-only user data is mostly outside the developer's processing.** WP29: "If
  the data processing only takes place on the device itself, and no personal data are
  transmitted outside the device, the law wouldn't apply to the user" (the domestic
  exception), while a controller running a remote platform stays responsible for that
  platform. — **Confirmed**, WP29 annex. A never-signed-in user's training record is
  therefore not something the developer processes. AdMob's and Firebase Auth's
  transmissions still are. — **Inferred**.

### 5.2 EU and UK representatives

- Art. 27(1): where Art. 3(2) applies, the controller "shall designate in writing a
  representative in the Union". The exemption in 27(2)(a) needs processing that is
  "**occasional**, does not include, on a large scale, processing of special categories
  … and is unlikely to result in a risk". — **Confirmed**, GDPR.
- EDPB: processing "can only be considered as 'occasional' if it is not carried out
  regularly, and occurs outside the regular course of business or activity of the
  controller". — **Confirmed**,
  [EDPB Guidelines 3/2018 on territorial scope, v2.1](https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_3_2018_territorial_scope_after_public_consultation_en_1.pdf)
- UK: organisations with no UK establishment that offer goods or services to people in
  the UK "generally need to appoint someone to act as their representative in the UK",
  with an exemption for "occasional low-risk processing". The representative's details go
  in the privacy notice. — **Confirmed**,
  [ICO, who does the UK GDPR apply to](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/personal-information-what-is-it/who-does-the-uk-gdpr-apply-to/)
- *Implication:* an app's core processing is regular by definition, so the exemption
  fails **whether or not** the data is health data. Offering on EEA and UK storefronts
  means appointing two representatives (paid services exist) or not distributing there.
  Both stores let a developer choose countries. — **Inferred**.

### 5.3 Lawful bases, transparency and transfers

- **Bases (Inferred):** contract (Art. 6(1)(b)) for cloud backup, friends and the
  entitlement. Consent under GDPR and ePrivacy for ads storage and personalisation, which
  UMP already gathers. Plus explicit consent (Art. 9(2)(a)) for the cloud copy if §3's
  conservative reading is taken.
- **Art. 13 information** at the time of collection (§2.2 table). — **Confirmed**.
- **Transfers.** Collecting directly from an EU user by a non-EU controller is **not** a
  Chapter V transfer (EDPB Example 1). But the controller's disclosure to a non-EEA
  **processor** "would amount to a transfer" needing Art. 28 and Chapter V safeguards
  (Example 2). — **Confirmed**,
  [EDPB Guidelines 05/2021 v2.0](https://www.edpb.europa.eu/system/files/documents/2023-02/edpb_guidelines_05-2021_interplay_between_the_application_of_art3-chapter_v_of_the_gdpr_v2_en_0.pdf).
  Google is the processor for Firebase (customers "act as 'data controller'"), under
  Firebase's Data Processing and Security Terms, with EU-U.S. DPF certification and SCCs
  "where applicable". Firebase Authentication "processes exclusively in US data
  centers". — **Confirmed**, [Firebase privacy](https://firebase.google.com/support/privacy).
  *Implication:* the transfer safeguard is Firebase's terms. Accept them in the console
  and name the transfer in the policy. — **Inferred**.
- **Google Analytics for Firebase is under separate terms.** Adopting it changes the
  processor picture. — **Confirmed**, same page.

### 5.4 Data subject rights the app has to be able to serve

| Right | Requirement | What Gym Notes has / needs | Grade |
|---|---|---|---|
| Timing, fees (Art. 12(3), 12(5)) | Act "without undue delay and in any event within one month", extendable by two months; free of charge. A controller "cannot remain silent", even when refusing. | A monitored contact address. | **Confirmed** (GDPR; [WP242](https://ec.europa.eu/newsroom/article29/items/611233)) / **Inferred** |
| Identification (Art. 12(6)) | No prescriptive method. "Providing the relevant login and password might be sufficient". | In app, the signed-in session. By email, ask the requester to confirm from the app, or match the provider email. | **Confirmed** (WP242) / **Inferred** |
| Access (Art. 15) | Confirmation, a **copy** of the personal data, and the Art. 15(1) information. Electronic requests get electronic form. | The export file covers user data. **Account-level data** (provider email, display name, friend list, friend requests, split grants, entitlement) is not in it. | **Confirmed** (Art. 15) / **Inferred** |
| Portability (Art. 20) | Data "provided by" the subject, for processing based on consent or contract, by automated means, "in a structured, commonly used and machine-readable format". Includes observed data; excludes derived data such as personal records. | The export file, for user data. Display name and the friend list are also "provided by" the user, but friends' data is limited by Art. 20(4). | **Confirmed** (Art. 20, WP242) / **Inferred** |
| Erasure (Art. 17) | On withdrawal of consent, objection, and other grounds. "Data portability cannot be used … as a way of delaying or refusing such erasure." | In-app cloud deletion (§4) plus the web/email path. | **Confirmed** / **Inferred** |
| Before closing an account | WP29 recommends "always include information about the right to data portability before data subjects close any account". | Already met: export is offered before account deletion (#6, #15). | **Confirmed** (WP242) / **Inferred** |

### 5.5 Security, design, and the Omnibus

- Art. 32 requires security "appropriate to the risk", including "pseudonymisation and
  encryption" as appropriate. Art. 25 requires data protection by design and default.
  Firebase encrypts in transit (HTTPS) and at rest for Firestore and Auth. —
  **Confirmed**, GDPR, Firebase privacy.
- The Digital Omnibus would change Art. 12(5) (abusive access requests), Art. 13(4) and
  terminal-equipment rules. It is a proposal, not yet law. — **Confirmed** (EDPB-EDPS
  2/2026) / **Community** (timeline).

---

## 6. Ads plus an under-16 audience, with no age gate

### 6.1 Store rules

- **Apple.** Apps in the Kids Category "should not include third-party analytics or
  third-party advertising" (1.3). Apps outside it must not imply children are the main
  audience (2.3.8). Apps that collect personal information "from a minor" need a privacy
  policy and must follow children's statutes (5.1.4). Age ratings must be answered
  honestly (2.3.6). — **Confirmed**, App Review Guidelines.
- **Play.** The developer declares target age groups, and Play may review them. If the
  target includes children: content must suit them, AAID must not be transmitted, "only
  use Google Play Families Self-Certified Ads SDKs", no interest-based ads, and mixed
  audiences use "a neutral age screen". Imagery and terminology that appeal to children
  can override the declaration. — **Confirmed**,
  [Play target audience and Families](https://support.google.com/googleplay/android-developer/answer/9893335)
- *Implication:* **declaring a 13+ target audience on Play, and staying out of the Kids
  Category on Apple, avoids both Families obligations and any age gate.** Nothing in Gym
  Notes appeals to under-13s. — **Inferred**.

### 6.2 GDPR and the UK Children's Code

- GDPR Art. 8: where consent is the basis for an information society service offered
  "directly to a child", it is valid from **16**, or as low as 13 where a Member State
  says so. Below that, a parent must give or authorise it, and the controller "shall make
  reasonable efforts to verify". — **Confirmed**, GDPR.
  - *Implication:* a UMP "Consent" tap from a 14-year-old in a 16-country is not valid
    consent to personalised ads. Without knowing age, the app cannot tell. —
    **Inferred**.
- **ICO Age Appropriate Design Code.** It covers services "likely to be accessed by"
  under-18s, meaning "more probable than not", and "is not restricted to services
  specifically directed at children". It reaches non-UK services that offer to UK
  users. Standard 3: "Take a risk-based approach to recognising the age of individual
  users" or "apply the standards in this code to all your users instead". Standard 12:
  profiling "off by default". Standard 13: no nudges to weaken privacy. Standard 2: a
  DPIA. — **Confirmed**,
  [ICO code: services covered](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/services-covered-by-this-code/),
  [standard 3](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/3-age-appropriate-application/).
  Whether teenagers are "more probable than not" to use a strength tracker is a
  judgement call. The "apply to everyone" route costs little here. — **Inferred**.

### 6.3 AdMob's age treatment (the tool that makes "no age gate" workable)

- TFCD and TFUA are deprecated. "Instead, use the Tag for age treatment (TFAT)." Values
  are `CHILD` (personalised ads and remarketing off, third-party ad vendor requests off,
  child protections on, AAID and IDFA not transmitted) and `TEEN` (personalised ads and
  remarketing off, "ad-serving protections for teens are applied"). — **Confirmed**,
  [AdMob help 6219315](https://support.google.com/admob/answer/6219315),
  [Android targeting](https://developers.google.com/admob/android/targeting)
- The Flutter plugin has `ageRestrictedTreatment`, which "replaces the deprecated
  `RequestConfiguration.tagForChildDirectedTreatment`" and `tagForUnderAgeOfConsent`.
  Updated 2 Oct 2026. — **Confirmed**,
  [AdMob Flutter targeting](https://developers.google.com/admob/flutter/targeting).
  This confirms #18's correction.
- Google's teen protections "disable ads personalization" and restrict sensitive
  categories, including "body modification and weight loss" and "pharmaceuticals and
  supplements". These largely duplicate the console blocks set in #18. — **Confirmed**,
  [Ad-serving protections for teens](https://support.google.com/admob/answer/12171027)
- Since #11 removed revenue as a design criterion, applying `TEEN` treatment
  (non-personalised) to EEA/UK users of unknown age, or to everyone, costs only eCPM. —
  **Inferred**.

### 6.4 US App Store Accountability Acts: in force now

- **Texas SB 2420** (enrolled). The developer "shall create and implement a system to use
  information received [from the app store] to verify … the age category assigned to
  that user" and parental consent (§121.054). It must assign an age rating (§121.052).
  It must give the store notice before a "significant change", which includes changes
  to the types of data collected and new monetisation features (§121.053). Age data may
  be used only to enforce age restrictions, comply with law and implement safety
  features, and must be deleted once verification is done (§121.055). There is a
  good-faith safe harbour for relying on the store's signals (§121.056(c)). Violations
  are deceptive trade practices. — **Confirmed**,
  [SB 2420](https://capitol.texas.gov/tlodocs/89R/billtext/html/SB02420F.htm)
- **Status.** Apple announced the Texas requirements for 1 January 2026, paused them
  after a December 2025 injunction, and on 3 June 2026 said that "due to a recent court
  ruling lifting an injunction", new Texas Apple Accounts are subject from **4 June
  2026**. Developers must "implement the Declared Age Range API", "use the Significant
  Change API under the PermissionKit framework", configure server notifications for
  withdrawn consent, and "determine when there's a significant change". — **Confirmed**,
  [Apple news 8 Oct 2025](https://developer.apple.com/news/?id=btkirlj8),
  [4 Nov 2025](https://developer.apple.com/news/?id=2ezb6jhj),
  [3 Jun 2026](https://developer.apple.com/news/?id=sg176nne). The ruling was a Fifth
  Circuit stay of the injunction. — **Community**,
  [MacRumors](https://www.macrumors.com/2026/06/03/apple-app-store-texas-sb-2420/)
- **Utah** (new accounts from 6 May 2026) and **Louisiana** (from 1 July 2026) use the
  same Apple toolset. — **Confirmed**,
  [Apple news 24 Feb 2026](https://developer.apple.com/news/?id=f5zj08ey)
- **Play Age Signals API** (beta) returns signals for eligible Texas users from 28 May
  2026. Use is limited to "age-appropriate content and experiences in compliance with
  laws", never "advertising, marketing, user profiling, or analytics". — **Confirmed**,
  [Play Age Signals](https://developer.android.com/google/play/age-signals/overview)
  (updated 20 Jul 2026)
- **COPPA.** A general-audience service has no duty "to investigate the ages of
  visitors", but gains actual knowledge when it learns a user's age. — **Confirmed**,
  [FTC COPPA FAQ H.1](https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions).
  *Implication:* once a store signal says "under 13", the developer **has actual
  knowledge** for that user. COPPA then applies to sign-in (email, name), friends and
  ad identifiers. — **Inferred**.

**Answer to the sub-question (Inferred):** no obligation forces an *in-app* age gate.
Play's 13+ target and Apple's general-audience listing avoid Families and Kids rules.
GDPR Art. 8 and the Children's Code can be met by not relying on a minor's consent,
using `TEEN` or non-personalised treatment where age is unknown. But US law now **does**
force the app to *consume* store age signals and parental-consent events in Texas, Utah
and Louisiana. It must also decide what each age category does: at minimum `CHILD`/`TEEN`
ad treatment, and for under-13s, possibly no sign-in or friends. That is store
plumbing, not an onboarding questionnaire, so the zero-friction first run survives.

---

## The plaintext export: verdict

**Survives, with three caveats.** The decision under test, from #15: the export file is
plaintext JSON with no password and no encryption, delivered through the OS share sheet.

### What the sources say about format

- Neither store says anything about data export at all: Apple's App Review Guidelines
  5.1.x, Play's User Data, Data safety and account deletion pages. No store obliges an
  export, so none constrains its format or protection. — **Confirmed** (by absence in
  the pages read).
- GDPR Art. 20(1): "structured, commonly used and machine-readable format". Recital 68
  adds "interoperable". — **Confirmed**.
- WP242: where no industry format exists, "data controllers should provide personal data
  using commonly used open formats (e.g. XML, **JSON**, CSV,…) along with useful
  metadata"; "it is crucial that the individual is in a position to fully understand the
  definition, schema and structure"; hindrances include "deliberate obfuscation of the
  dataset". — **Confirmed**, [WP242 rev.01](https://ec.europa.eu/newsroom/article29/items/611233).
  **JSON is named outright. The format choice is endorsed, not just tolerated.**

### What the sources say about protection

- WP242, "How can portable data be secured?": the controller must secure **transmission**
  ("end-to-end or data encryption") "to the right destination (by the use of strong
  authentication measures)". But "such security measures **must not be obstructive** in
  nature and must not prevent users from exercising their rights". Once the data is
  delivered, "the data subject requesting the data is responsible for identifying the
  right measures in order to secure personal data in his own system. However, **he
  should be made aware of this**". As leading practice, controllers "may also recommend
  appropriate format(s), encryption tools and other security measures". — **Confirmed**,
  WP242.
- WP242: controllers answering portability requests "are not responsible for the
  processing handled by the data subject". — **Confirmed**.
- Play User Data: "Handle all personal and sensitive user data securely, including
  transmitting it using modern cryptography". — **Confirmed**.

### Applying it (Inferred)

1. **There is no controller-run transmission channel to secure.** The file is generated
   on the user's own device, inside an already-authenticated session, and handed to the
   OS. The *user* picks the destination (Files, AirDrop, Drive, mail), whose transport
   that app or service secures. WP242's "right destination … strong authentication"
   concern is about a controller sending data across a network to someone claiming to be
   the user, and that does not arise here. For never-signed-in users the developer is not
   even processing the data (domestic exception, §5.1).
2. **A password would cut against the guidance.** WP242 requires security measures to be
   non-obstructive. A forgotten password on a rescue artifact is the textbook way to
   prevent a user exercising the right. #15's reasoning ("a password on a rescue artifact
   is a way to lose your own data") matches the regulator's own test.
3. **Health status does not change the answer.** Art. 32 raises the bar for the
   *controller's systems* in proportion to risk, and the export leaves those systems
   at the user's request. Health status would make the "be made aware" step more
   important, not make encryption mandatory.

### The three caveats

1. **Make the user aware.** The export flow should say, in one plain line, that the file
   is not encrypted and holds the whole training record, so it should be stored like any
   personal document. That meets WP242's "he should be made aware" and needs no
   confirmation gate, so it fits #15's "the offer is an offer". WP242's leading practice
   of recommending encryption tools stays optional.
2. **Hand over from app-private storage only.** On Android, share through a content URI
   (FileProvider) from the app's private cache, never by writing to public storage such
   as Downloads, where other apps can read it. On iOS, share from the app container or
   temp directory. This is what makes "the OS is the real boundary" true in practice, and
   it is what Play's "handle … securely" asks of the part the app controls.
3. **The export file is not, alone, the whole GDPR access or portability answer** for a
   signed-in EEA/UK user. It deliberately excludes account-level data: provider email,
   display name, the friend list, friend requests, split grants and the entitlement.
   Art. 15 reaches all of it, and Art. 20 reaches what the user provided. #15's statement
   that the export file "is the mechanism already committed to for the data-export
   obligation" holds for **user data** only. Account-level data needs a request path, and
   answering by hand from the Firebase console within a month is enough at this scale.
   This caveat does not argue for putting friends into the file: WP242 itself recommends
   letting users "exclude, where relevant, data of other individuals".

One optional improvement follows from WP242's schema language, though no rule requires
it: document the export schema publicly (field meanings, units, version line), so the
file is interpretable without the app.

---

## Decisions this forces (flagged for the human, not taken here)

1. **Policy host.** GitHub Pages from the public repo, Firebase Hosting on Spark, or
   something else. The same host carries the Play account-deletion page.
2. **Policy contact identity.** Which name, email and postal address appear in the
   policy, the Apple EU trader declaration and Play's merchant address. Consider a P.O.
   box or virtual address before the Remove Ads product goes live on Play.
3. **EEA and UK distribution.** Either appoint an EU representative and a UK
   representative (recurring cost, details in the policy), or exclude EEA/UK storefronts
   from v1. This one decision also decides whether UMP and the GDPR-specific items
   below are needed at launch.
4. **Health-data stance for GDPR.** The conservative reading (explicit Art. 9 consent to
   cloud backup at sign-in, for EEA/UK users) or the ordinary-personal-data reading.
   This only matters if decision 3 keeps the EEA/UK.
5. **Account-deletion mechanics under the delete quota.** Options: (a) client-driven,
   resumable, multi-day deletion with a stated duration and a completion confirmation;
   (b) Blaze, for `recursiveDelete` in a Function, at negligible per-delete cost but with
   a billing account and budget alert per ADR-0002; (c) a schema change that cuts
   documents per user, which runs into ADR-0002's one-set-per-document rule. The decision
   also has to say what "returns the app to its anonymous state" means: delete the
   Firebase Auth user and create a new anonymous one.
6. **Web deletion path.** An email address or a form, and the developer-side procedure
   behind it (an Admin SDK or CLI script run locally, plus Apple token revocation).
7. **Age-signal handling.** Adopt Apple's Declared Age Range, Significant Change and
   consent-revocation APIs and Play Age Signals for Texas, Utah and Louisiana. Decide what
   each age category does to ad treatment (`CHILD`/`TEEN`), and whether under-13s can
   sign in or have friends.
8. **Ad treatment where age is unknown.** Keep UMP consent-based personalisation in the
   EEA/UK, or apply `TEEN` or non-personalised treatment to everyone there, or globally.
   Revenue was ruled out as a criterion in #11.
9. **Store declarations.** Play target audience 13+ (or 18+), and Apple age rating
   answers including the new social-media questions. Choose the store category: Health &
   Fitness triggers Apple's medical-device declaration ("No") and Play's Activity and
   Fitness declaration.
10. **Export file caveats.** Whether to adopt the three caveats above, which do not change
    #15's core decision, and whether to publish the export schema.
11. **Policy-change process.** Texas §121.053 requires notice to the stores before a
    significant privacy-policy change, so policy edits become part of the release
    checklist.
12. **No training data in ad requests, ever.** This is already implied by Apple 5.1.3, but
    worth writing into the spec as a rule.

## New questions that look like their own tickets

- **Age signals and minors.** What the app does for each store-supplied age category,
  how it handles parental-consent revocation, and what COPPA means once "under 13" is
  known. This crosses ads, sign-in and friends. It is large enough that a "no age gate"
  product decision needs re-stating in its terms.
- **EEA/UK launch scope.** Representatives, explicit consent, UMP, Children's Code DPIA,
  or geo-exclusion from v1. A product and cost decision with legal inputs.
- **Account deletion at scale on Spark.** The delete-quota collision (§4.3) shares its
  arithmetic with ADR-0004 and may want an ADR of its own: resumable client-side
  deletion versus Blaze.
- **US state consumer-health-data laws.** Whether Washington's My Health My Data Act and
  similar state laws reach a strength-training record, and what a "consumer health data
  privacy policy" would add. Out of this ticket's stated scope (stores and GDPR), but the
  app's market is the US.
- **Crashlytics and Analytics adoption.** Neither is decided. Each changes both store
  forms, and Google Analytics brings separate (non-processor) terms.

---

## Sources

**Apple**
- App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- Offering account deletion in your app: https://developer.apple.com/support/offering-account-deletion-in-your-app/
- App Privacy Details: https://developer.apple.com/app-store/app-privacy-details/
- Third-party SDK requirements: https://developer.apple.com/support/third-party-SDK-requirements/
- DSA trader requirements: https://developer.apple.com/help/app-store-connect/manage-compliance-information/manage-european-union-digital-services-act-trader-requirements/
- News: Texas requirements (8 Oct 2025) https://developer.apple.com/news/?id=btkirlj8 · Next steps for Texas (4 Nov 2025) https://developer.apple.com/news/?id=2ezb6jhj · Update for Texas (3 Jun 2026) https://developer.apple.com/news/?id=sg176nne · Brazil, Australia, Singapore, Utah, Louisiana (24 Feb 2026) https://developer.apple.com/news/?id=f5zj08ey · Regulated medical device status (26 Mar 2026) https://developer.apple.com/news/?id=nyqbfz1y · Age rating social media questions (9 Jul 2026) https://developer.apple.com/news/?id=tlur8uvi · Sign in with Apple domain (24 Aug 2026) https://developer.apple.com/news/?id=1ptvdtcm · ATT in the EU (16 Sep 2026) https://developer.apple.com/news/?id=idsft9ai

**Google Play**
- User Data policy: https://support.google.com/googleplay/android-developer/answer/10144311
- Data safety: https://support.google.com/googleplay/android-developer/answer/10787469
- Account deletion: https://support.google.com/googleplay/android-developer/answer/13327111
- Target audience and Families: https://support.google.com/googleplay/android-developer/answer/9893335
- Health apps declaration: https://support.google.com/googleplay/android-developer/answer/14738291
- Health Content and Services: https://support.google.com/googleplay/android-developer/answer/16679511
- Developer account information: https://support.google.com/googleplay/android-developer/answer/13634081
- Play Age Signals API: https://developer.android.com/google/play/age-signals/overview

**Firebase**
- Play data disclosure (Android SDKs): https://firebase.google.com/docs/android/play-data-disclosure
- App Store data collection (Apple SDKs): https://firebase.google.com/docs/ios/app-store-data-collection
- Privacy and security: https://firebase.google.com/support/privacy
- Delete data (Firestore): https://firebase.google.com/docs/firestore/manage-data/delete-data
- Firestore quotas: https://firebase.google.com/docs/firestore/quotas
- Cloud Functions get started: https://firebase.google.com/docs/functions/get-started
- Auth, manage users (Flutter): https://firebase.google.com/docs/auth/flutter/manage-users
- Auth, federated (Flutter): https://firebase.google.com/docs/auth/flutter/federated-auth
- Hosting quotas: https://firebase.google.com/docs/hosting/usage-quotas-pricing

**AdMob / UMP**
- Play data disclosure: https://developers.google.com/admob/android/privacy/play-data-disclosure
- iOS data disclosure: https://developers.google.com/admob/ios/privacy/data-disclosure
- Flutter targeting: https://developers.google.com/admob/flutter/targeting
- Android targeting: https://developers.google.com/admob/android/targeting
- Tag for age treatment: https://support.google.com/admob/answer/6219315
- Ad-serving protections for teens: https://support.google.com/admob/answer/12171027
- Google as controller / EU consent: https://support.google.com/admob/answer/7666366
- UMP quick start: https://developers.google.com/admob/ump/android/quick-start

**EU / UK**
- GDPR (Regulation 2016/679): https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679
- WP29 Guidelines on data portability, WP242 rev.01: https://ec.europa.eu/newsroom/article29/items/611233
- WP29 annex, health data in apps and devices (2015): https://ec.europa.eu/justice/article-29/documentation/other-document/files/2015/20150205_letter_art29wp_ec_health_data_after_plenary_annex_en.pdf
- CJEU C-184/20: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:62020CJ0184
- CJEU C-21/23 Lindenapotheke: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:62023CJ0021
- EDPB Guidelines 3/2018 (territorial scope): https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_3_2018_territorial_scope_after_public_consultation_en_1.pdf
- EDPB Guidelines 05/2021 (Art. 3 and Chapter V): https://www.edpb.europa.eu/system/files/documents/2023-02/edpb_guidelines_05-2021_interplay_between_the_application_of_art3-chapter_v_of_the_gdpr_v2_en_0.pdf
- EDPB-EDPS Joint Opinion 2/2026 (Digital Omnibus): https://www.edpb.europa.eu/system/files/documents/2026-02/edpb_edps_jointopinion_202602_digitalomnibus_en.pdf
- ICO, special category data: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/special-category-data/what-is-special-category-data/
- ICO, who the UK GDPR applies to: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/personal-information-what-is-it/who-does-the-uk-gdpr-apply-to/
- ICO Children's Code, services covered: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/services-covered-by-this-code/
- ICO Children's Code, standard 3: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/childrens-information/childrens-code-guidance-and-resources/age-appropriate-design-a-code-of-practice-for-online-services/3-age-appropriate-application/

**US**
- Texas SB 2420 (enrolled): https://capitol.texas.gov/tlodocs/89R/billtext/html/SB02420F.htm
- Washington RCW 19.373.010: https://app.leg.wa.gov/RCW/default.aspx?cite=19.373.010
- Washington AG on My Health My Data: https://www.atg.wa.gov/protecting-washingtonians-personal-health-data-and-privacy
- FTC Health Breach Notification Rule guidance: https://www.ftc.gov/business-guidance/resources/complying-ftcs-health-breach-notification-rule-0
- FTC COPPA FAQ: https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions

**Other**
- GitHub Pages limits: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits
- Community: MacRumors on Texas SB 2420 (3 Jun 2026): https://www.macrumors.com/2026/06/03/apple-app-store-texas-sb-2420/ · White & Case on the Digital Omnibus: https://www.whitecase.com/insight-alert/gdpr-under-revision-key-takeaways-from-digital-omnibus-regulation-proposal
