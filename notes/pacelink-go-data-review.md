# PaceLink Go data-handling review

Reviewed September 4, 2026 against the local `gapline` checkout. This is a source/configuration review, not an inspection of deployed AWS tables, logs, backups, App Store disclosures, provider contracts, or actual deletion timings. No PaceLink Go app/backend code was changed.

## Findings reflected in the website

- **Personal data is processed.** Trip participants supply a display name and transmit latitude, longitude, time, accuracy, speed, and course. The backend retains temporary trip/participant IDs, membership timestamps, credentials, push tokens, and cached Live Activity state. No account or email is needed. See `backend/src/lib/dynamo.ts` (`ParticipantItem`, `TripMetaItem`, `putLocations`).
- **Trip members can access location data.** The normal app polls computed gaps, but authenticated `/locations` and `/tracks` endpoints expose other active members' current/recent coordinates. `requireLiveTrip` rejects ended/expired/unactivated trips; authorization rejects departed participants. See `backend/src/handlers/locations.ts` and `backend/src/handlers/trips.ts:427`.
- **GPS expiry is not physical deletion.** `putLocations` sets TTL to the earlier of sample time plus 45 minutes and the scheduled trip expiry. AWS says automatic physical deletion normally happens within a few days after expiry. Explicit leave/end operations request earlier GPS deletion. See `backend/src/lib/dynamo.ts:944` and [AWS TTL documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html).
- **Trip metadata outlives an early End.** Ending sets `endedAt`, deletes invitation lookups, and attempts GPS cleanup. It does not immediately remove participant names, token hashes, push tokens, cached activity state, or create-response credentials. Most temporary records have TTL at the scheduled trip expiry plus five minutes. Extensions update relevant TTLs. See `backend/src/lib/dynamo.ts:228`, `:361`, `:746`, and `:769`.
- **Purchase records persist indefinitely in the current implementation.** The consumed-purchase record includes transaction ID, product ID, purchase time, consumption time, and trip ID. It has no TTL and is not deleted with the trip. It prevents transaction reuse; it contains no coordinates. See `backend/src/lib/dynamo.ts:195` and `backend/src/lib/storekit.ts`.
- **First-party usage measurement exists.** Daily integer counters cover hosting, repeat hosting, invitations, joins, duration milestones, ends, paywall views, and conversion. The counters have no per-person identifiers, coordinates, event timestamps finer than a day, or TTL. This is not advertising/cross-app tracking, but “no analytics” would be inaccurate. See `backend/src/lib/stats.ts`, `backend/src/handlers/stats.ts`, and `ios/Gapline/Stats/StatEvent.swift`.
- **Diagnostics are separate from anonymous counters.** Rejected-fix logs include participant IDs and rejection reasons. Other logs include error/route details. Source configuration sets Lambda log retention to 14 days in AWS eu-central-1. No intentional raw-coordinate logging or third-party ad/analytics SDK was found in the reviewed source/dependencies. No claim is made about unseen provider logs. See `backend/src/handlers/locations.ts:65`, `backend/src/lib/http.ts:141`, `backend/serverless.yml`, and `backend/package.json`.
- **Local summaries are richer than “numbers only.”** `SavedTrip.summary.companions` contains names and actual participant IDs, plus closest/farthest/average/final gaps and together time. Summaries also include dates, distance, duration, speeds, and group size. No coordinates or route archive is serialized. History is stored in UserDefaults with no expiry or delete UI. Ordinary device backups are not explicitly excluded in the reviewed code. See `ios/Gapline/Stats/LocalTripHistory.swift`, `ios/Gapline/Trip/TripTally.swift:249`, and `ios/Gapline/UI/Home/TripHistoryView.swift`.
- **Host recovery uses Keychain.** The current host trip ID, participant credential and invitation details are stored with `AfterFirstUnlockThisDeviceOnly`. Confirmed completion clears it; a failed end may retain it for recovery. Do not promise uninstall clears all Keychain data. See `ios/Gapline/Trip/HostSessionStore.swift` and `TripManager.swift:580` / `:824`.
- **Apple receives activity and purchase data.** APNs payloads include participant/activity information and relative measurements, not raw coordinate tracks. StoreKit signed transactions are sent to the backend for verification. See `backend/src/lib/activityState.ts`, `apns.ts`, `storekit.ts`, and `ios/Gapline/LiveActivity/ActivityController.swift`.

## Cleanup issues to resolve before promising immediate erasure

1. `deleteLocations` queries one DynamoDB page only and ignores `LastEvaluatedKey`. It also ignores `BatchWriteCommand.UnprocessedItems`; batches can be partially processed without throwing. Add pagination and bounded retry/backoff, plus failure handling. See `backend/src/lib/dynamo.ts:1039`.
2. `putLocations` authorizes before writing. An already-authorized write can race with leave/end cleanup; enforce live membership at write time or run reliable follow-up cleanup. See `backend/src/handlers/locations.ts:57` and `backend/src/lib/dynamo.ts:944`.
3. `TripManager.endTrip` / `leaveTrip` catch server failures after local teardown without a durable retry queue. Server-side membership/access can remain valid until a later successful request or trip expiry. Do not equate local completion with server deletion. See `ios/Gapline/Trip/TripManager.swift:584`.
4. `getLatestLocation` does not explicitly filter an expired record's TTL; older fixes can remain queryable during a still-active trip pending DynamoDB deletion (the gap logic treats old samples as stale). A 45-minute TTL is not a hard retention/access guarantee. See `backend/src/lib/dynamo.ts:1011`.

## Publication and app-disclosure follow-ups

The outdated privacy gallery slide is no longer displayed on the app page; its image file is preserved. “No shared personal info” and “then discarded” are too broad without qualification: display names and live positions are shared, cleanup can lag, and the records above remain. The page provides the full retention explanation through its feature copy and footer privacy link. Revise the same claims in app/store artwork and in-app copy before treating them as an unconditional promise.

The privacy manifest now declares device ID and product interaction collection alongside UserDefaults API use. Complete/verify the separate App Store Connect App Privacy disclosure against actual data flows, including precise location, identifiers, purchases, and usage/diagnostic data; a required-reason API entry is not a data-collection inventory.

Confirm deployed TTL/log settings, any additional backup/logging configuration, international-transfer arrangements, support correspondence retention, and the appropriate legal bases before publication. Source code alone cannot establish those operational/legal facts.

The App Store link uses ID `6807346681` from `ios/APP_STORE_CONNECT.md`; public listing availability was not confirmed. The page therefore says “View on the App Store” without a release-date claim. This site remains in its existing static GitHub Pages structure; publishing/commit/push was not part of this edit.


## September 4 device-allowance update

The three PaceLink Go pages now describe the new device-verification release. Publish
with the corresponding production backend/client rollout; configuring dev alone does
not make this behavior live for existing production clients. No publish or push was performed.

Verified against `backend/src/lib/devices.ts`, `welcome.ts`, `dynamo.ts`,
`backend/serverless.yml`, and `ios/Gapline/Monetization/DeviceAllowanceClient.swift`:

- Random app identity and recovery secret are transmitted over HTTPS; the server
  persists HMAC identity and secret hash, not their raw values. It stores public key,
  key ID, attestation fingerprint, replay counter, allocation status and granted/used credits.
- App Attest keys have permanent ownership records. Allowance/authentication records
  have no TTL. Issuance recovery may retain a ledger reference until resolved.
- DeviceCheck tokens go to Apple to query/set the developer-scoped allocated marker;
  raw tokens and full proofs are not persisted in the device database.
- Device usage counts (verified free/paid creates and first-guest activations) are
  separate from daily aggregates and expire 730 days after the last usage write.
  Temporary trip metadata links to the device record, so no anonymity claim is made.
- Challenges expire after five minutes. Database deletion is asynchronous. Device
  table configuration enables point-in-time recovery; backup erasure is not immediate.
- App preferences contain the installation marker and legacy telemetry counter;
  ThisDeviceOnly Keychain contains persistent credentials and pending-create recovery.
  Credential loss can break statistical continuity and recovery of unused credits.
- Creating an invitation spends a credit. A failed verification is not zero balance.
  Reinstall does not reset Apple’s allocation marker or the retained allowance.

Before publication, review the proportionality of indefinite allowance/authentication
retention and purchase retention, the legitimate-interest basis for per-device analytics,
any applicable device-storage consent requirements, international-transfer safeguards,
and a workable identity-verification/deletion/used-device support procedure. The website
states the actual current lack of automatic expiry and self-service deletion; it does
not claim these code choices alone establish legal compliance. Transparency requirements:
[GDPR, particularly Articles 5, 6, 11, 13, 17 and 21](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng).
The removed artwork should also be revised wherever it appears in App Store materials.
