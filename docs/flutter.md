# Savers App SDK (Flutter)

Flutter SDK that acts as the communication layer between your Host App and Savers App — it hosts the Savers merchant web app inside your app via `HostedAppComponent`, pulls member/reward data for display in your own screens, and relays native device actions (maps, dialer, browser) between the hosted web experience and the device.

Package name: `savers_app_sdk`

## Contents

- [Getting Access — Partner Registration & Credentials](#getting-access--partner-registration--credentials)
- [Security: Where Credentials Must Live](#security-where-credentials-must-live)
- [Installation](#installation)
- [Platform Setup](#platform-setup)
- [Quick Start](#quick-start)
- [Information Request Functions](#information-request-functions)
- [Example App](#example-app)
- [Notes](#notes)

---

## Getting Access — Partner Registration & Credentials

This SDK is published as a public package, but it is only functional for registered Savers App partners. Access is gated at the credential level, not at the package level — anyone can install `savers_app_sdk`, but every API call requires credentials issued through partner registration.

**Procedure to obtain your security credentials:**

1. **Register as a partner** at <https://www.saversapp.com>. Registration includes business verification and a payment step handled by the Savers App business team — self-signup alone does not grant access.
2. Once verification and payment are complete, your organization is issued:
   - **API Key** — identifies your organization to Savers App
   - **Encryption Key** — base64-encoded 256-bit AES key used to encrypt the `qP` payload
   - **Program Referral Code (`pRefCode`)** — identifies your specific program
3. Credentials are managed from the partner portal — this is also where you rotate or revoke a compromised key, and where you'll find sandbox vs. production credential pairs.
4. Do not share credentials outside your organization's backend team. Each partner's credentials are unique to that partner and tied to the agreement signed during registration.

## Security: Where Credentials Must Live

**Your API Key and Encryption Key must be held on your own backend server — never hardcoded in your app's source, and never committed to any repository, public or private.**

The current SDK API (`SaversAppSDK.initialize`) requires your app process to hold these values in memory to call the SDK — that part is unavoidable given how the SDK is built today. What you control, and what matters for security, is *where the values come from*:

- ✅ **Correct:** Your host backend stores the API Key and Encryption Key (in a secrets manager or environment variable, not in code). Your app requests them from your own backend at runtime/login, holds them only in memory, and passes them into `initialize()`. They are never written into your app's source code or build config.
- ❌ **Incorrect:** The API Key or Encryption Key appears as a literal string constant anywhere in your app's source, `.env` files committed to git, CI config, or build scripts.

Additional practices:

- Treat `authSessionId` the same way you'd treat any session credential — don't log it, and don't pass it through mechanisms that could leak it (e.g. URL query strings that end up in logs or browser history). The SDK already stores the underlying `sessionId` via `flutter_secure_storage` (iOS Keychain / Android Keystore-backed), not plain local storage — it persists until the user logs out (`clearUserSession`).
- Generated URLs from `generateUrl` are single-use (nonce-based) by design — don't cache or reuse a previously generated URL; call `generateUrl` again for a fresh one.
- If you suspect a credential has leaked, rotate it immediately from the partner portal rather than waiting for a scheduled rotation.
- A rotated/revoked/invalid API Key surfaces as a `401` thrown from `initializeUserSession` (it gates the authorize call). A rotated/revoked/invalid Encryption Key, or any other malformed payload issue, surfaces as a `400` thrown from `generateUrl`. Catch both so your app can react to a credential problem (e.g. re-fetch from your backend, alert your team) instead of failing silently or retrying with a dead credential.

## Installation

```yaml
dependencies:
  savers_app_sdk:
    path: ../SaversFlutterLibrary   # or your published package name
```

The SDK depends on `webview_flutter`. After adding the package, run `flutter pub get`.

## Platform Setup

Android `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

iOS `Info.plist`:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>Location access is used to show nearby offers.</string>
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>tel</string>
  <string>telprompt</string>
</array>
```

## Quick Start

### 1. Initialize the SDK

Fetch your credentials from your own backend at runtime (see [Security](#security-where-credentials-must-live) above) before calling `initialize`. Omitted `environment` defaults to production (`https://m.saversapp.com/`).

```dart
import 'package:savers_app_sdk/savers_app_sdk.dart';

await SaversAppSDK.initialize(
  apiKey: apiKeyFromYourBackend,
  encryptionKey: encryptionKeyFromYourBackend, // base64 of 32 bytes
  pRefCode: 'PROGRAM_REF_CODE',
  authMode: 'EMAIL', // 'EMAIL' | 'PHONE'
  environment: SaversSdkHostedEnvironment.sandbox | SaversSdkHostedEnvironment.prod, // optional: sandbox | prod
);
```

| `environment` | `generateUrl` host |
|---|---|
| `SaversSdkHostedEnvironment.sandbox` | `https://testm.saversapp.com/` |
| `SaversSdkHostedEnvironment.prod` (or omitted) | `https://m.saversapp.com/` |

`initialized` remains as a deprecated alias of `initialize`.

### 2. User session and device

Call after host login. Behind the scenes, `initializeUserSession` now calls a backend authorization endpoint on your behalf — secured with the API key and HMAC-SHA256 payload signing you already supplied to `initialize()` — to establish the session and store the `sessionId` it returns; `generateUrl` and the Information Request Functions use that stored `sessionId` afterward. You still only call `initializeUserSession` itself — there's no separate authorize step for you to trigger from the SDK.

Every call to `initializeUserSession` sends the current profile to the backend, along with the `sessionId` already in secure storage if one exists (for example, after an app restart). When that `sessionId` is present and still valid, the backend updates the profile on the existing session and returns the same `sessionId` with a fresh access token — this is what makes repeat calls idempotent on the session itself, not just on the profile. If the `sessionId` is missing, expired, or revoked, the backend creates a new session and returns a new `sessionId` and access token instead. Either way, the profile update always takes effect.

If the API Key is rejected (rotated, revoked, or otherwise invalid), `initializeUserSession` throws a `401` — catch this so your app can handle it (e.g. re-fetch credentials from your backend) rather than proceeding with a broken session.

`registerDevice` requires a `deviceId`; `location` is optional. Call it again on each visit/session, not just once after login — refreshing the location on every visit keeps it current for features that depend on the device's location (e.g. nearby offers), rather than relying on a stale coordinate from a previous session.

```dart
await SaversAppSDK.initializeUserSession(
  userId: 'USER_ID',
  firstname: 'FIRST_NAME',
  lastname: 'LAST_NAME',
  email: 'EMAIL_ADDRESS',
  phone: 'PHONE_NUMBER', // required when authMode is PHONE
  city: 'CITY',
  zipcode: 'ZIP_CODE',
  dob: 'DATE_OF_BIRTH', // optional
  pv: '1',
  ev: '1',
);

await SaversAppSDK.registerDevice(
  deviceId: await getDeviceId(),
  location: (lat: latitude, lng: longitude), // from your own location APIs
);

// On logout:
await SaversAppSDK.clearUserSession();
```

Profile rules:

- `userId` and `email` are mandatory
- `phone` is mandatory when `authMode` is `PHONE`
- `dob` is optional
- `pv` / `ev` are optional (`'0'` or `'1'`)
- `referrerUserId` is optional (`referrer_user_id` in the encrypted payload)

### 3. Generate URL

Hosted URLs are single-use (nonce) and expire **60 seconds** after generation — once consumed or expired, the backend rejects the URL. Call `generateUrl` again to mint a fresh URL; user/device come from SDK context.

`generateUrl` throws a `400` if the Encryption Key is rejected (rotated, revoked, or otherwise invalid) or the payload otherwise fails validation — catch this the same way you'd catch a `401` from `initializeUserSession`.

```dart
final url = await generateUrl(screen: const Screen(name: 'Explore'));
// https://testm.saversapp.com/?pRefCode=...&qP=...  (sandbox)
```

When `screen.name` is `OfrDetails`, `screen.attributes` must be non-empty. Omitting `screen` defaults to `Explore`.

Nonce is fetched automatically from the stored `userId` + `pRefCode`. Coordinates from `registerDevice` / `setLocationCoordinates` are included in `deviceInfo.location` when set.

Because `initializeUserSession` now establishes the session upfront (see [above](#2-user-session-and-device)), `generateUrl` encrypts the stored `sessionId` as `authSessionId` from the first call — the hosted app no longer needs a prior visit to restore login. `authSessionId` is only omitted if `initializeUserSession` hasn't been called yet, or the SDK hasn't captured a value for it.

`qP` encryption matches React Native `AesCbcCrypto` / hosted web decrypt:

- AES-256-CBC
- inner `hex(iv):base64(content)`
- outer standard base64 (so the page can `atob(qP)`)

### 4. HostedAppComponent

Pass the generated URL into `HostedAppComponent`. To refresh a consumed URL, call `generateUrl` again and pass the new string.

```dart
HostedAppComponent(
  saversAppUrl: generatedUrl,
  onSaversSdkMessage: (raw, postBack) {
    // Required by HostedAppComponent. The SDK handles action dispatch
    // internally (including ending the session), so no logic is needed
    // here for a standard integration.
  },
);
```

`HostedAppComponent` maintains two WebViews — at any point in time only one is active/visible, the other stays hidden (not destroyed).

Flow:

1. Load: WebView 1 opens `saversAppUrl`. Header and WebView 2 stay hidden.
2. Hub → **Travel** tile: hosted app posts an envelope with the travel URL.
3. SDK processes `open_travel`, hides WebView 1, shows header + WebView 2 with that URL.
4. Header **Back** (native, no close Post Message required): hide WebView 2, show WebView 1 (still loaded), inject Hub navigation `{ type: 'NAVIGATE', route: 'Hub' }`.

Travel Post Message (from Hub):

```json
{
  "target": "SDK",
  "from": "savers",
  "action": "open_travel",
  "payload": { "url": "<TRAVEL_PORTAL_URL>" }
}
```

Optional: `prepare_travel` (show header early), `close_travel` (same as native Back), `relay` between surfaces. Payload may include branding (`logo`, `backIconColor`, `loaderColor`).

`HostedAppController.closeSession()` asks the Savers WebView to run Cognito `closeSession`.

## Session & Login

`initializeUserSession` calls a backend authorization endpoint internally, using the API key and encryption key you already passed to `initialize()`, together with the current profile and the `sessionId` already in secure storage if one exists. The endpoint is gated by a mandatory `x-api-key` plus payload signing, so only requests carrying valid partner credentials are accepted. The SDK persists only the `sessionId` it gets back — via `flutter_secure_storage` (see [Security](#security-where-credentials-must-live) above), not plain local storage — and holds the access token **in the SDK's in-memory context only — never written to local storage or disk.** The SDK never holds a refresh token at all. The backend is the source of truth on both token and session validity: the SDK does not track expiry itself. When it needs an access token and doesn't have a valid one in memory, it asks the backend for one using the stored `sessionId`:

- If the session is still valid, the backend issues a new access token (the refresh token, if the backend keeps one, stays unchanged server-side — the SDK never sees or manages it).
- If the session itself is gone (expired/revoked), the backend returns `401` and the SDK silently re-runs the authorize flow using the profile it already holds in memory from the last `initializeUserSession` call — no action or re-login prompt is required from the host app. This mid-session recovery only covers the app's current run, since it relies on the profile still being in memory. A real app restart is covered a different way: the host app calls `initializeUserSession` again on login/startup per the [Quick Start](#2-user-session-and-device) flow, which now also sends the `sessionId` already in secure storage — so a restart typically reuses the existing session (with the profile refreshed) rather than creating a new one, and only falls back to a brand-new session if that stored `sessionId` turns out to be invalid. (Recommended, not yet confirmed as implemented: the silent mid-session retry should be capped rather than unbounded, so a backend that keeps rejecting doesn't loop indefinitely.)

This means the only client-side credential that survives an app restart is the opaque, revocable `sessionId` — not a bearer token. Information Request Function calls authenticate with the in-memory access token via a `Bearer` header rather than a client-supplied `userId` (see [Information Request Functions](#information-request-functions) below) — this also closes off a client from being able to request another user's data just by passing a different `userId`.

The `sessionId` itself is just an opaque reference your backend resolves — if it's missing, stale, or revoked, the web portal re-authenticates the user automatically and a new `sessionId` is issued. Treat `authSessionId` / `sessionId` as sensitive like any other session credential — don't log it or pass it through anything that could leak it (e.g. URL query strings). Holding the access token only in memory limits exposure to disk-based extraction (backups, other apps on a rooted/jailbroken device, storage inspection) — it doesn't defend against a fully compromised device reading process memory directly, but that's a much higher bar than reading local storage.

Re-calling the authorize endpoint with a `sessionId` that's still valid updates the profile on that existing session and returns the same `sessionId` rather than minting a new one — a new `sessionId` only appears once the previous session is no longer valid (or none was supplied yet).

The access token is valid for **30 minutes**. The SDK doesn't need to track this itself (see the reactive, backend-authoritative refresh above) — it's noted here for context on how often an active session triggers a token refresh in the background.

The Information Request Functions use the in-memory access token to authenticate on the SDK's behalf; those requests are protected with payload signing to prevent tampering in transit. Note this is a different property than app-instance attestation (Play Integrity / App Attest) — payload signing verifies a request wasn't altered, not that it came from a genuine, unmodified copy of the app. Attestation is not currently implemented.

## Information Request Functions

| Function | Purpose | Response shape |
|---|---|---|
| `getTotalEarnings()` | Fetch member earnings to date | `{ totalEarnings: <float>, totalPayouts: <float> }` |
| `getTransactions()` | Fetch member's latest reward transactions (30 days) | `{ txnId, purchaseAmount, status, ... }` |
| `getEarnings()` | Member earnings/payouts by month (90 days) | `{ earnings: [{month, value}], payouts: [{month, value}] }` |
| `getRecommendations()` | Offer recommendations (10–20 offers) | `[{ offId, brandName, logo, offType, ... }]` |
| `getFavouriteOffers()` | Fetch member's favorite offers | `[{ offId, brandName, logo, offType, ... }]` |

None of these take a `userId` argument — the member is identified by the in-memory access token sent as a `Bearer` header (see [Session & Login](#session--login) above), not by a client-supplied ID. This also means a caller can't request another member's data by passing a different `userId`.

From the Flutter app, these SDK functions handle the request on the Host App's behalf — you don't need to call Savers App APIs directly to get this data.

## Example App

```bash
cd example
flutter pub get
flutter run
```

Demo notes:

- Init uses sandbox credentials and `SaversSdkHostedEnvironment.sandbox`, then `initializeUserSession` + `registerDevice`.
- **Open in Browser** / **Open in WebView** sit on the Generated URL card. WebView opens `HostedAppComponent`.
- After encryption or init changes, use **hot restart** (not only hot reload).

## Notes

- Location permissions are required on Android and iOS for coordinate enrichment.
- `encryptionKey` must be base64 of exactly 32 bytes (AES-256).
- `pRefCode` and `initializeUserSession` are required before `generateUrl`.
- This package is public on pub.dev; functionality requires valid partner credentials obtained per [Getting Access](#getting-access--partner-registration--credentials) — installing the package alone does not grant access to any Savers App data.
