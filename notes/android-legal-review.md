# Android legal-page review — September 9, 2026

Updated the shared Glance for PV and PaceLink Go privacy pages and terms to cover Apple and Android with labelled platform differences. Added Glance terms and linked them from its product/privacy pages. Both product pages identify Android as awaiting Google Play approval, per the owner. Existing policy URLs remain valid for store and in-app links.

## Source evidence

Read local sibling repositories without modifying app/backend code:

- `solis-ios/android/app/src/main/AndroidManifest.xml` and `build.gradle.kts`: location/notification permissions, disabled backup, Android API 26 minimum.
- Glance Android `GlanceStore.kt`, `SolisApi.kt`, `PurchaseVerifier.kt`, `PlayBilling.kt`: Keystore-protected credentials, private local caches, direct SolisCloud requests, local signed-receipt validation and Google Play billing.
- Glance Android `PVForecast.kt`, `ForecastUI.kt`: site coordinates/panel configuration sent directly to Open-Meteo or Forecast.Solar; optional one-shot device location; local energy notifications.
- Glance Apple `PVForecastService.swift`, `iCloudSnapshotStore.swift`, and Game Center participation/reporting code: forecasts, iCloud snapshot relay, opt-in scores/achievements and system context. Corrected the previous blanket “everything stays on device” claim.
- PaceLink Android `SecureStore.kt`, `AndroidIntegrity.kt`, `PlayBilling.kt`, manifest/backup rules, `PaceLinkScreen.kt`: encrypted no-backup journal, hashed Android ID, Play Integrity, foreground location service, local notifications, history deletion controls.
- PaceLink backend `androidDevices.ts`, `playIntegrity.ts`, `googlePlay.ts`: further HMAC device identity, persistent allowance, Google verification, hashed purchase-token transaction identifier.
- PaceLink backend `securityStore.ts`: short-lived IP-derived rate-limit hashes and cleanup-job references retained until completion.

## Scope and remaining release checks

This was a source review, not verification of the submitted AABs, deployed infrastructure, Play Console answers, provider contracts, or support deletion operations. Policies must describe the build and service actually shipped. The earlier PaceLink review is historical; some of its cleanup implementation findings have since changed.

Before release, reconcile Google Play Data safety with all actual transfers, including Glance forecast coordinates and PaceLink location, device identifiers, purchases, usage, and diagnostics. Check provider SDK disclosures as well. Verify that location disclosure and affirmative consent occur in the app before collection/permission requests where required; a website policy does not replace that flow. Public policy links must be present in the app and Play Console. No account-creation flow was found; do not invent one to satisfy deletion wording. The policies provide a direct support deletion request route and describe remaining local/provider/server records.

Primary guidance reviewed: [Google Play User Data policy](https://support.google.com/googleplay/android-developer/answer/10144311?hl=en) and [Play Integrity overview](https://developer.android.com/google/play/integrity/overview).

Validation: all six edited/created HTML pages have balanced tags, unique IDs, and resolving local file links; `git diff --check` passes. No commit, push, or deployment performed.
