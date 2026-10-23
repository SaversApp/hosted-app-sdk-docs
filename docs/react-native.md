# Savers App SDK (React Native)

React Native SDK that embeds the Savers App experience inside your Host App. It keeps the hosted Savers web app loaded once in the background, shows it on your own routes (ready-made or custom screens), signs your members in, and gives you data functions (banners, categories, hub tiles, earnings, …) to build your own native screens.

Package name: `@savers_app/react-native-sdk` · Version: **1.5.0**

> **Upgrading from 1.3.x?** `generateUrl`, `HostedAppComponent` and the old `initialize({ pRefCode, … })` options are deprecated. See [Deprecation Notes & Migration](#deprecation-notes--migration).

## Contents

- [How It Works](#how-it-works)
- [Step 0 — Getting Access — Partner Registration & Credentials](#step-0--getting-access--partner-registration--credentials)
- [Security: Where Credentials Must Live](#security-where-credentials-must-live)
- [Step 1 — Install](#step-1--install)
- [Step 2 — Platform Setup](#step-2--platform-setup)
- [Step 3 — Initialize the SDK](#step-3--initialize-the-sdk)
  - [Certificate pinning (`sslPinning`)](#certificate-pinning-sslpinning)
- [Step 4 — Add SaversAppProvider (load the WebView once)](#step-4--add-saversappprovider-load-the-webview-once)
- [Step 5 — Sign In / Sign Out](#step-5--sign-in--sign-out)
- [Profile Verification](#profile-verification)
- [Session & Login](#session--login)
- [Step 6 — Show Savers Screens](#step-6--show-savers-screens)
  - [Full app: SaversAppScreen](#full-app-saversappscreen)
  - [Embedded app](#embedded-app) — [Option A: Ready-made screens](#option-a-ready-made-screens), [Option B: Custom screens](#option-b-custom-screens-hostedappscreen)
  - [Screen names](#screen-names)
- [Guest Mode](#guest-mode)
- [Data Functions (for your own screens)](#data-functions-for-your-own-screens)
- [Host ↔ Hosted App Communication](#host--hosted-app-communication)
- [Error Handling](#error-handling)
- [Deprecation Notes & Migration](#deprecation-notes--migration)
- [Example App](#example-app)
- [Notes](#notes)

---

## How It Works

```
Your App
 └─ SaversAppProvider            ← wraps your navigator, loads the Savers WebView ONCE
     ├─ your NavigationContainer
     │   ├─ your native screens   ← use SaversAppSDK.getBanners(), getCategories(), …
     │   ├─ CashbackScreen        ← ready-made Savers screen (native back arrow)
     │   └─ HostedAppScreen       ← your custom Savers screen
     └─ Savers WebView (hidden until a Savers screen is focused)
```

1. You call `SaversAppSDK.initialize(...)` once at startup.
2. `SaversAppProvider` loads the Savers web app in a WebView **once** and keeps it alive.
3. When the user goes to a Savers route, the SDK shows the WebView and switches it to that screen — **no reload**. When they leave, it hides again.
4. When a member signs in through your app, the SDK passes the session to the hosted app automatically.

---

## Step 0 — Getting Access — Partner Registration & Credentials

This SDK is a public package, but it only works for registered Savers App partners. Access is controlled by credentials, not by the package: anyone can install `@savers_app/react-native-sdk`, but every call needs credentials issued when you register.

**How to get your credentials:**

1. **Register as a partner** at [https://saversapp.com](https://saversapp.com). Registration includes business verification and a payment step, both handled by the Savers App business team.
2. **Register your app** by contacting **Savers App support**. *(Soon you'll be able to do this yourself at [https://saversapp.com](https://saversapp.com).)* Give us:
   - Your **iOS bundle ID** and **Android package name**. The SDK checks that the running app matches the registered one. This catches configuration mistakes (wrong App ID, wrong build). It is **not** a security control: it runs on the device and can be faked, so your keys are still the only real protection.
   - Whether **guest mode** should be on (guests can browse Savers screens without signing in). See [Guest Mode](#guest-mode).
3. Once verification and payment are done, you'll receive:

   | Credential | What it is |
   |---|---|
   | **API Key** | Identifies your organization to Savers App |
   | **Encryption Key** | Base64-encoded 256-bit AES key used to sign requests |
   | **App ID** | Identifies this app (tied to your bundle ID / package name) |

   Separate credential sets are issued for **sandbox** and **production**.
4. **To rotate or revoke a compromised key**, contact Savers App support (later, the partner portal).
5. **Don't share credentials** outside your organization's backend team. Each partner's credentials are unique to that partner and tied to the agreement signed during registration. Keep them on your backend — see [Security](#security-where-credentials-must-live).

> You no longer need a `pRefCode`. The SDK gets your program code from your App ID automatically.

---

## Security: Where Credentials Must Live

**Keep your API Key and Encryption Key on your own backend.** Never hardcode them in app source, committed `.env` files, CI config or build scripts.

- ✅ Your app fetches them from your backend at runtime, keeps them in memory, and passes them to `initialize`.
- ❌ Keys as string literals anywhere in the app or repository.

How the SDK protects the session:

- Only an opaque, revocable **session ID** is stored on the device, in iOS Keychain / Android Keystore (`react-native-keychain`).
- The **access token lives in memory only** (short-lived, refreshed automatically). It is never written to disk.
- Requests are signed (HMAC-SHA256) to prevent tampering.
- Don't log the session ID or put it in URLs.

If a key leaks, contact Savers App support to rotate it right away.

---

## Step 1 — Install

```bash
npm install @savers_app/react-native-sdk

npm install @react-native-async-storage/async-storage \
  react-native-keychain \
  @react-native-community/netinfo \
  @react-navigation/native \
  react-native-device-info \
  react-native-get-random-values \
  react-native-safe-area-context \
  react-native-svg \
  react-native-webview \
  react-native-ssl-public-key-pinning
```

(or the same with `yarn add`). Then `cd ios && pod install`.

- `react-native-keychain` stores the member's session ID securely (iOS Keychain / Android Keystore). `async-storage` holds non-sensitive state only. Both are required.
- `react-native-safe-area-context` is required — wrap your app in `<SafeAreaProvider>`.
- **Expo:** works with a [development build](https://docs.expo.dev/develop/development-builds/introduction/), not Expo Go (`react-native-keychain` and `react-native-device-info` are native modules).
- `react-native-ssl-public-key-pinning` turns on **certificate pinning** for the SDK's API calls (`api.saversapp.com`, `devapi.saversapp.com`): they only connect when the server's certificate chains to Savers App's pinned root certificates, which stops interception by a fake certificate. Install it — without it the SDK logs a warning and its API calls go out **unpinned**. The SDK sets it up in `initialize` (no code needed). It doesn't apply to the Savers WebViews. **If your app already pins its own domains**, see [Certificate pinning](#certificate-pinning-sslpinning).

There is one package for both environments; you pick sandbox or production in `initialize`.

---

## Step 2 — Platform Setup

Android `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

iOS `Info.plist` (lets Savers open the phone dialer):

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>tel</string>
  <string>telprompt</string>
</array>
```

**Location (optional).** The SDK never reads location or asks for permission itself. It only uses the coordinates you pass to `registerDevice` (used for nearby offers). If your app collects location, you add the location permissions (`ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION`, `NSLocationWhenInUseUsageDescription`) and a location library such as `@react-native-community/geolocation` as part of your own app.

---

## Step 3 — Initialize the SDK

Call `initialize` **once, before your app renders `SaversAppProvider`**.

```ts
import DeviceInfo from 'react-native-device-info';
import { SaversAppSDK } from '@savers_app/react-native-sdk';

async function setUpSavers() {
  const { apiKey, encryptionKey } = await fetchKeysFromYourBackend();

  SaversAppSDK.initialize({
    environment: 'sandbox', // 'sandbox' | 'production' (required)
    apiKey,
    encryptionKey,
    appId: 'YOUR_APP_ID',
  });

  // Recommended: register the device. Call again on each visit to refresh location.
  SaversAppSDK.registerDevice({
    deviceId: await DeviceInfo.getUniqueId(),
    location: { lat, lng }, // optional — only if your app already has it
  });
}
```

Neither call needs `await`: both return immediately.

| `environment` | Hosted app |
|---|---|
| `'sandbox'` | `https://testm.saversapp.com/` |
| `'production'` | `https://m.saversapp.com/` |

What's different from 1.3.x:

- **It never throws and returns nothing (no `await`).** It stores the keys in memory and checks your App ID in the background.
- If the check fails (unknown App ID, wrong bundle ID, revoked key) or a key is missing, you're told through `SaversAppProvider`'s `onConfigError`, and SDK calls reject with that error. The rest of your app keeps working. See [Error Handling](#error-handling).
- Network errors during the check are retried automatically (and immediately when the device comes back online).
- Calling it again with the same keys does nothing. Different keys restart the check.

### Certificate pinning (`sslPinning`)

With `react-native-ssl-public-key-pinning` installed (see [Step 1](#step-1--install)), the SDK's API calls only connect to a server whose certificate chains to Savers App's pinned root certificates. Choose who sets it up with the `sslPinning` option of `initialize`:

| `sslPinning` | Who sets up pinning | Use when |
|---|---|---|
| `'sdk'` (default — omit it) | **The SDK**, inside `initialize`. Every SDK API call waits until it's in place. | Your app doesn't use `react-native-ssl-public-key-pinning` itself. |
| `'app'` | **Your app** — the SDK skips its own setup and only uses the pins you set. | Your app already pins its own domains with this library. |

**Why `sslPinning: 'app'` exists:** the library keeps **one** pinning setup per app, and each `initializeSslPinning` call **replaces** the previous one. If your app and the SDK both called it, whichever ran last would wipe out the other's domains. With `'app'`, there's exactly one call — yours — containing both.

**How to merge (`sslPinning: 'app'`):**

1. Get the SDK's entries with `SaversAppSDK.getSslPinningConfig()`. It returns one entry per SDK API domain, in the library's format:

   ```ts
   {
     'api.saversapp.com':    { includeSubdomains: false, publicKeyHashes: ['++MBgDH5…', 'f0KW/Ftq…', 'NqvDJlas…', '9+ze1cZg…'] },
     'devapi.saversapp.com': { includeSubdomains: false, publicKeyHashes: [ /* same */ ] },
   }
   ```

   It works before `initialize` (no network call). Always read it from the SDK — don't copy the hashes — so an SDK update with new certificates reaches your app.
2. Spread it into **your** `initializeSslPinning` call, next to your own domains.
3. **`await` that call, then** call `initialize` with `sslPinning: 'app'` — so the SDK's first API call (the App ID check, which starts inside `initialize`) is already pinned.

```ts
import { initializeSslPinning } from 'react-native-ssl-public-key-pinning';
import { SaversAppSDK } from '@savers_app/react-native-sdk';

async function setUpSavers() {
  const { apiKey, encryptionKey } = await fetchKeysFromYourBackend();

  // 1–2. ONE pinning setup for the whole app: your domains + the SDK's.
  await initializeSslPinning({
    ...SaversAppSDK.getSslPinningConfig(),
    'api.yourapp.com': {
      includeSubdomains: true,
      publicKeyHashes: ['YOUR_PIN_1', 'YOUR_PIN_2'], // at least two per domain (iOS)
    },
  });

  // 3. Then the SDK — told not to set pinning up itself.
  SaversAppSDK.initialize({
    environment: 'production',
    apiKey,
    encryptionKey,
    appId: 'YOUR_APP_ID',
    sslPinning: 'app',
  });
}
```

Good to know:

- In `'app'` the SDK never calls `initializeSslPinning`, so it can't overwrite your setup. If you leave the SDK's entries out of your call, its API calls go out **unpinned** — always include them.
- If you change your pinning later (another `initializeSslPinning` call), include `getSslPinningConfig()` again — the new call replaces everything.
- The deprecated `initialize({ apiKey, encryptionKey, pRefCode, … })` options accept `sslPinning` too.
- On an iOS Expo **development** build, `expo-dev-client`'s network inspector interferes with pinning — see the [library's notes](https://github.com/frw/react-native-ssl-public-key-pinning#expo-managed-workflow). Production builds are unaffected.

---

## Step 4 — Add SaversAppProvider (load the WebView once)

Wrap your navigator with `SaversAppProvider`. It loads the Savers web app **one time** and keeps it alive while your users move between screens, so Savers screens open instantly with no reload.

```tsx
import { NavigationContainer, createNavigationContainerRef } from '@react-navigation/native';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { SaversAppProvider, type SaversAppTheme } from '@savers_app/react-native-sdk';

const navigationRef = createNavigationContainerRef();

// Keep at module scope so it's one stable object
const theme: SaversAppTheme = {
  primary: '#333399',      // back arrows, Sign in button
  secondary: '#FF9900',
  textPrimary: '#111111',
  textSecondary: '#555555', // status messages
  // Optional — the SDK's native header (Embedded apps, Travel, sign-in / unavailable states):
  logo: require('./assets/logo.png'),    // or { uri: 'https://…/logo.png' | '…/logo.svg' } or <Logo />; fitted within 160×80
  logoSize: { width: 160, height: 60 },  // optional; max 160×80 — pass it for a remote logo to skip a short loader
  logoPosition: 'right',                 // 'right' (default) or 'center'
  backIcon: require('./assets/back.png'), // optional; default = SDK arrow in `primary`; 24×24
};

export default function App() {
  return (
    <SafeAreaProvider>
      <SaversAppProvider
        theme={theme}
        onSignInRequired={() => navigationRef.navigate('SignIn')}
        onSignOutRequested={signOut} // your logout, then SaversAppSDK.closeSession()
        onNavigateBack={() => navigationRef.goBack()}
        onEndSession={() => navigationRef.navigate('Home')}
        onConfigError={(e) => console.warn('Savers unavailable', e.code, e.message)}
      >
        <NavigationContainer ref={navigationRef}>
          {/* your stack */}
        </NavigationContainer>
      </SaversAppProvider>
    </SafeAreaProvider>
  );
}
```

| Prop | When it's called / what it does |
|---|---|
| `theme` | Your brand colours for the SDK's native UI (back arrows, messages, Sign in button), plus an optional header `logo` — PNG/JPG, remote SVG, or an SVG component (max 160×80; optional `logoSize`, `logoPosition` `'right'` or `'center'`) and custom `backIcon`. Full apps use the Hosted App header, so the logo and back icon apply only where the SDK draws the header. |
| `onSignInRequired(screen?)` | A guest needs to sign in (a members-only screen, or the Hub's **Sign in** tile) — **navigate to your sign-in / sign-up screen**. See [Step 5](#step-5--sign-in--sign-out). |
| `onSignOutRequested()` | The member tapped **Sign out** in the Savers Hub — **sign them out of your app and call `SaversAppSDK.closeSession()`**. The Savers screen stays open, now signed out. See [Step 5](#step-5--sign-in--sign-out). |
| `onNavigateBack()` | The Hosted App header's back arrow was tapped with nothing left to go back to — pop your route. |
| `onEndSession()` | The user left the Savers experience (e.g. "Home" in Savers) — go to your home screen. |
| `onConfigError(error)` | The App ID check failed. Savers screens show an "unavailable" message instead. |

> `SaversAppProvider` throws `NOT_INITIALIZED` if it renders before `SaversAppSDK.initialize(...)`.

---

## Step 5 — Sign In / Sign Out

Your app owns sign-in. Add a **sign-in route** to your navigator and point `onSignInRequired` at it. The SDK calls it when a guest taps "Sign in" inside Savers, or opens a members-only screen.

On that screen, after your own login succeeds, start the Savers session:

```ts
await SaversAppSDK.initializeUserSession({
  userId: 'USER_ID',          // required
  email: 'EMAIL_ADDRESS',     // required
  firstname: 'FIRST_NAME',
  lastname: 'LAST_NAME',
  phone: 'PHONE_NUMBER',      // required when your auth mode is PHONE
  city: 'CITY',               // optional
  zipcode: 'ZIP_CODE',        // optional
  dob: 'YYYY-MM-DD',          // optional
  pv: '1',                    // optional, '0' | '1'
  ev: '1',                    // optional, '0' | '1'
  referrer_user_id: 'ID',     // optional
});
navigation.goBack(); // back to the Savers screen they came from
```

The provider passes the session to the hosted app automatically. Nothing else to wire up.

**Profile verification** (only if Savers App has switched it on for your app): also pass a fresh one-time token from your server — see [Profile Verification](#profile-verification):

```ts
const profileToken = await yourApi.createSaversProfileToken(); // your server, per sign-in
await SaversAppSDK.initializeUserSession(profile, { profileToken });
```

- Call it again on every app start / login — it reuses the stored session when still valid and refreshes the profile.
- Expired tokens are refreshed silently in the background.

Sign out (when the member logs out of your app):

```ts
await SaversAppSDK.closeSession();
```

This ends the session on the backend, clears the stored session, and signs the hosted app out too.

Sign-in and sign-out always belong to your app — the Hosted App's Hub only asks for them:

- **Sign in** tile (guests) → `onSignInRequired` — open your sign-in screen; after your login, call `initializeUserSession`.
- **Sign out** tile (members) → `onSignOutRequested` — log out of your app, then call `closeSession()`:

```tsx
<SaversAppProvider
  onSignOutRequested={async () => {
    await myAuth.logout();              // your app's sign-out
    await SaversAppSDK.closeSession();  // ends the Savers session too
  }}
>
```

---

## Profile Verification

An optional extra check, switched on per app by Savers App (agreed with you): when a member signs in, Savers asks **your server** to confirm the member, so a sign-in can't be faked from the app alone.

**Flow**

1. The member signs in to your app.
2. Your server creates a **one-time `profileToken`** for that member (short expiry, single use).
3. Your app calls `SaversAppSDK.initializeUserSession(profile, { profileToken })`.
4. Savers calls **your verify API**, server to server, with the token.
5. Your API returns that member's profile → Savers compares it with `profile` → the member is signed in, or the sign-in is refused.

**Your verify API** (you build it; give Savers its URL, an API key and a signing secret):

```http
POST https://api.yourapp.com/savers/verify
Content-Type: application/json
x-api-key: <API key you gave Savers>
x-signature: <HMAC-SHA256, hex, of the exact request body — key = your signing secret, base64-decoded>

{ "profileToken": "...", "timestamp": 1761120000000, "nonce": "6f1c2b0e-…" }
```

- Check `x-api-key` and `x-signature` (compute the HMAC over the raw body bytes), and reject a `timestamp` older than a few minutes or a repeated `nonce`.
- **Valid token → `200`** with the member's profile, in the **same format** you pass to `initializeUserSession` (optional fields may be `null` or left out):

  ```json
  { "userId": "SV-10293", "firstname": "Asha", "lastname": "Menon", "email": "asha@example.com", "phone": "+15551234567", "dob": null, "city": "New York", "zipcode": null }
  ```

- **Invalid / expired / used token → `401`.**
- Answer within **5 seconds**.

Savers checks that `userId` matches and that the **email** (case-insensitive) or the **phone** (digits only) matches — which one is agreed per app.

**Good to know**

- Create a **new token for every `initializeUserSession` call** — each one is used once.
- Only **new** sessions are verified. While the member's session is active, the SDK re-signs in silently in the background without a token; after the session ends (or a reinstall), your app calls `initializeUserSession` with a fresh token again.
- The SDK adds your App ID to the request itself — nothing else to pass.

**Errors** (`SDKError.code` from `initializeUserSession`):

| `code` | HTTP | Meaning |
|---|---|---|
| `INVALID_USER_SESSION` | 401 | No `profileToken`, or your API answered 401 |
| `PROFILE_MISMATCH` | 422 | Your API returned a different member / email / phone than `profile` |
| `PARTNER_VERIFICATION_UNAVAILABLE` | 503 | Your API didn't answer (timeout / error) — the sign-in is refused; try again |
| `APP_ID_REQUIRED` | 400 | An old SDK version without the App ID — update the SDK |

---

## Session & Login

What happens behind `initializeUserSession` and `closeSession` (Step 5). Nothing extra to wire up — this is for understanding and security reviews.

### What the SDK keeps, and where

| Item | Where | Lifetime |
|---|---|---|
| Session ID | iOS Keychain / Android Keystore (`react-native-keychain`) | Until sign-out or revoked. Without `react-native-keychain` it isn't kept across app restarts. |
| Access token | Memory only — never written to disk | Short-lived (set by the backend); refreshed automatically |
| Refresh token | Memory only | Gone when the app restarts |
| Member profile (from `initializeUserSession`) | Memory only | Used to re-authorize silently; gone when the app restarts |

### How a session works

1. **Start** — `initializeUserSession` sends a signed request (your `x-api-key` + an HMAC-SHA256 signature made with your Encryption Key) along with the profile and the stored session ID, if any.
   - Stored session ID still valid → the **same** session is reused and its profile updated.
   - No session ID, or it's no longer valid → a **new** session is created and its session ID stored.
2. **Hosted app** — `SaversAppProvider` passes the session to the hosted app (on every page load too), which signs itself in.
3. **Expired token** — when a data call gets `401`, the SDK refreshes the access token; if that fails, it re-authorizes with the profile it holds in memory, then retries the call **once** (no loops). No prompt for the member.
4. **App restart** — memory is cleared, so call `initializeUserSession` again on startup / login (Step 5). With the stored session ID it usually continues the same session.
5. **Sign out** — `closeSession()` ends the session on the backend, clears the tokens, profile and session ID, and tells the hosted app, which ends its own sessions.

`initializeUserSession` rejects with `SDKError` `statusCode: 409`, `code: 'EMAIL_ALREADY_IN_USE'` when the email belongs to another member.

### Security notes

- The only credential that survives a restart is the opaque, revocable session ID — never a bearer token. Treat it like a password: don't log it or put it in URLs.
- Data functions authenticate with the member's access token (`Bearer`), not a `userId` you pass, so one member can't request another member's data.
- Request signing proves a request wasn't altered on the way; it doesn't prove it came from an unmodified copy of your app (app attestation — Play Integrity / App Attest — isn't implemented).
- Memory-only tokens are safe from disk extraction, but not from a fully compromised device reading app memory.

---

## Step 6 — Show Savers Screens

The Savers screens you show come from the **Hosted App** (a Savers App web page inside the SDK). Each screen has one of two headers:

- **Hosted App header** — the Hosted App's own header.
- **Native header** — a header the SDK draws in your app, with a back arrow.

The SDK supports two ways to show the Hosted App — pick the one that fits your app:

- **Full app** — you open **the whole Savers app** from one place (a tab or button) and members move around inside it. Use [`SaversAppScreen`](#full-app-saversappscreen).
- **Embedded app** — Savers screens sit inside your own native UI, one at a time. See [Embedded app](#embedded-app).

### Full app: `SaversAppScreen`

One route opens the complete Savers app: it starts on Explore (the Savers home), and members reach everything else — offers, cashback, cards, Hub — inside it, with the Hosted App header. When there's nothing left to go back to in Savers, its back (and Android back) returns to your app.

```tsx
import { SaversAppScreen } from '@savers_app/react-native-sdk';

<Stack.Screen name="Savers" component={SaversAppScreen} options={{ headerShown: false }} />

// From your tab / button:
navigation.navigate('Savers');
```

To open it on another page — e.g. from a push notification or deep link — pass `startScreen`: a [screen name](#screen-names), or a screen with attributes. As a prop, or as the route's `startScreen` param:

```tsx
navigation.navigate('Savers', { startScreen: 'CASHBACK' });

<SaversAppScreen startScreen={{ name: 'OFR_DETAILS', attributes: [{ key: 'ofrId', value: '12345' }] }} />
```

| Prop | Default |
|---|---|
| `startScreen` | The route's `startScreen` param, else `EXPLORE` |
| `onBack` | `navigation.goBack()` (does nothing if nothing is behind) |

Expo Router: import it from `@savers_app/react-native-sdk/expo-router` (route param `startScreen`, e.g. `router.push({ pathname: '/savers', params: { startScreen: 'CASHBACK' } })`).

Signing out (`SaversAppSDK.closeSession()`) while it's open keeps it open, now signed out — navigate away yourself if your app needs to.

### Embedded app

Show each Savers screen from your own UI: the ready-made screens (Option A), or your own route around any Savers screen (Option B).

#### Option A: Ready-made screens

Drop them into your stack like any other screen. Each one handles Android back and iOS swipe-back, and never exits your app. For the header each one shows, see [Headers](#headers).

```tsx
import {
  OffersScreen, CashbackScreen, CardsScreen, CardEnrollScreen, TravelScreen,
  SupportScreen, ProfileScreen, FavoritesScreen, InboxScreen, AboutUsScreen,
} from '@savers_app/react-native-sdk';

<Stack.Screen name="Offers"   component={OffersScreen}   options={{ headerShown: false }} />
<Stack.Screen name="Cashback" component={CashbackScreen} options={{ headerShown: false }} />
<Stack.Screen name="Cards"    component={CardsScreen}    options={{ headerShown: false }} />
<Stack.Screen name="Travel"   component={TravelScreen}   options={{ headerShown: false }} />
```

Set `headerShown: false` — the SDK (or the Hosted App) draws the header.

##### Headers

| Screen | Header |
|---|---|
| `CashbackScreen`, `CardsScreen`, `CardEnrollScreen`, `TravelScreen`, `SupportScreen`, `ProfileScreen`, `FavoritesScreen`, `InboxScreen` | Native header with a back arrow |
| `OffersScreen` | Hosted App header (location, map toggle, back) |
| `AboutUsScreen` | No header (the Hosted App's own close button goes back) |

Pass `nativeHeader` to a screen to override this (not `AboutUsScreen`, which never has one).

Props (all optional):

| Prop | Default |
|---|---|
| `onBack` | `navigation.goBack()` (does nothing if nothing is behind) |
| `backIconColor` | `theme.primary` |
| `nativeHeader` | As in [Headers](#headers). `true` = native back-arrow header, `false` = the Hosted App header. |
| `category` (`OffersScreen` only) | The route's `category` param, e.g. `navigation.navigate('Offers', { category: 'food' })` |
| `nearMe` (`OffersScreen` only) | The route's `nearMe` param — opens the **Near Me** offers map (`NEAR_ME`) at the device location from `registerDevice`, e.g. `navigation.navigate('Offers', { nearMe: true })` (all categories; `category` is ignored). |
| `search` (`OffersScreen` only) | The route's `search` param — opens offer **search** in all categories (`SEARCH`), as the Explore search bar, e.g. `navigation.navigate('Offers', { search: true })` (`category` is ignored). |

To pass props, render it as a child:

```tsx
<Stack.Screen name="Cashback" options={{ headerShown: false }}>
  {() => <CashbackScreen onBack={() => navigation.navigate('Home')} nativeHeader />}
</Stack.Screen>
```

> Ready-made screens use React Navigation. expo-router works too, in a development build (not Expo Go). On another navigator, use Option B.

#### Option B: Custom screens (`HostedAppScreen`)

Use `HostedAppScreen` to show any Savers screen with a **native back arrow** while you control focus and back.

```tsx
import { useIsFocused, useNavigation } from '@react-navigation/native';
import { HostedAppScreen } from '@savers_app/react-native-sdk';

function OfferDetailsScreen({ route }) {
  const navigation = useNavigation();
  return (
    <HostedAppScreen
      screen={{
        name: 'OFR_DETAILS',
        attributes: [{ key: 'ofrId', value: route.params.offerId }],
        showHeader: false, // hide the Hosted App header — the native one is used
      }}
      focused={useIsFocused()}
      onBack={() => navigation.goBack()}
    />
  );
}
```

| Prop | Required | Description |
|---|---|---|
| `screen` | ✅ | Which Savers screen to show — see [Screen names](#screen-names) |
| `focused` | ✅ | Your navigator's "is this route focused" value. Shown while `true`, hidden when `false`. |
| `onBack` | | What "back to your app" does — see [Back behaviour](#back-behaviour) |
| `nativeHeader` | | Default `true`. `false` = no native header — see [Back behaviour](#back-behaviour) |
| `backIconColor` | | Default `theme.primary` |

`screen` options:

| Field | Description |
|---|---|
| `name` | Screen name (below) |
| `attributes` | `[{ key, value }]` for screens that need them |
| `showHeader` | `false` hides the Hosted App header, `true` shows it, omitted = screen default |
| `back` | Only matters when the Hosted App draws the header — see [Back behaviour](#back-behaviour) |

##### Back behaviour

Pick one setup. The first is what the ready-made screens (except `OffersScreen`) use.

| Setup | Header the user sees | Back arrow | Android back button |
|---|---|---|---|
| **Native header** (default): `nativeHeader` omitted, `showHeader: false` | Your native back arrow | Calls `onBack` | Goes back inside Savers first; when Savers has no page left → `onBack` |
| **Hosted App header, back to your app**: `nativeHeader={false}`, `showHeader: true`, `back: 'native'` (what `OffersScreen` uses) | Hosted App header | Savers goes back inside its pages; when it has none left → `onBack` (or the provider's `onNavigateBack` if no `onBack`) | Same as the back arrow |
| **Hosted App header, stays in Savers**: `nativeHeader={false}`, `showHeader: true`, `back` omitted | Hosted App header | Savers goes back inside its pages; when none are left → Savers Explore | Goes back inside Savers; when none left → `onBack` |

iOS swipe-back always pops the route through your navigator. With no `onBack`, the ready-made screens use `navigation.goBack()`. Back never exits your app.

### Screen names

Used by `SaversAppScreen`'s `startScreen` and `HostedAppScreen`'s `screen`.

| Name | Screen | Attributes |
|---|---|---|
| `EXPLORE` | Offers and categories (default) | — |
| `OFFERS` | Offers for a category | `category` (required) |
| `NEAR_ME` | The Near Me offers map (all categories) at the device location from `registerDevice` | — |
| `SEARCH` | Search offers in all categories (the Explore search bar) | — |
| `OFR_DETAILS` | One offer's details | `ofrId` (required) |
| `TRAVEL` | Members-only travel portal | — |
| `CARD_ENROLL` | Card enrollment | — |
| `CARD_LIST` | Enrolled cards | — |
| `CASHBACK` | Cashback (transactions + payouts) | — |
| `PAYOUT` | Payouts tab of Cashback | — |
| `TRX` | Transactions tab of Cashback | — |
| `CONTACT` | Support ticket | — |
| `PROFILE` | Profile details | — |
| `FAV` | Favorite offers | — |
| `INBOX` | Inbox | — |
| `ABOUT_US` | About us (the program's intro and website link) | — |
| `NOTIF` | Notifications tab of Inbox | — |
| `REC` | Recommendations tab of Inbox | — |

> To test `OFFERS` / `OFR_DETAILS`, use `SaversAppSDK.getCategories()` for valid category slugs, or contact Savers App support for test offer IDs.

---

## Guest Mode

Guest mode is set per app at registration ([Step 0](#step-0--getting-access--partner-registration--credentials)).

| Guest mode | Not signed in |
|---|---|
| **On** | Guests can browse Savers screens. Members-only actions ask them to sign in. |
| **Off** | Savers screens don't open. The SDK asks them to sign in and shows a "Sign in to continue" message with a Sign in button. |

Gate your own native buttons (e.g. "Explore" tiles) the same way:

```tsx
const { guestBlocked, requestSignIn, isSignedIn, appInfo } = useSaversApp();

onPress={() => guestBlocked ? requestSignIn() : navigation.navigate('Offers')}
```

| Value | Meaning |
|---|---|
| `isSignedIn` | A member session is active |
| `guestBlocked` | Guest mode is off **and** nobody is signed in |
| `requestSignIn(screen?)` | Asks the member to sign in |
| `appInfo.guestMode` | The app's guest mode setting |

---

## Data Functions (for your own screens)

Use these to build native screens (home banners, category rails, hub tiles) with your own UI. They wait for the App ID check to finish. Guest-friendly functions also work before sign-in.

| Function | Signed in? | Returns |
|---|---|---|
| `getBanners(memberType?)` | No | Carousel banners |
| `getStrings(scope, lang?)` | No | CMS copy (text) for your own screens |
| `getCategories()` | No | Offer categories |
| `getHubTiles()` | No | Hub tiles (member-specific when signed in) |
| `getTotalEarnings()` | Yes | Earnings to date |
| `getEarnings()` | Yes | Earnings by month (90 days) |
| `getTransactions()` | Yes | Latest transactions (30 days) |
| `getRecommendations({ limit?, offset? })` | Yes | Recommended offers |
| `getFavoriteOffers({ limit?, offset? })` | Yes | The member's favorite offers |

Member functions identify the member from the signed-in session — none take a `userId`. All functions return a Promise and reject with `SDKError` (see [Error Handling](#error-handling)).

### getBanners

**Request**

```ts
const banners = await SaversAppSDK.getBanners();   // or getBanners(1) for a free member
```

Returns your program's **carousel** banners.

| Param | Type | Required | Notes |
|---|---|---|---|
| `memberType` | number | No | `1` = free member, `2` / `3` = paid. Omit it to skip member-specific banners. |

**Response**

```json
[
  {
    "id": <NUMBER>,
    "name": <STRING>,
    "imgUrl": <STRING>,
    "redirectUrl": <STRING>,
    "redirectUrlLink": <STRING>,
    "offerId": <NUMBER>,
    "rank": <NUMBER>
  }
]
```

### getStrings

**Request**

```ts
const strings = await SaversAppSDK.getStrings(['S32', 'S32.2']); // lang defaults to 'en'
```

| Param | Type | Required | Notes |
|---|---|---|---|
| `scope` | string[] | Yes | CMS scopes to load (one per screen / section) |
| `lang` | string | No | Default `'en'` |

**Response** — read a value by `scope` + `name`; fall back to your own copy when it's missing.

```json
[
  { "lang": <STRING>, "scope": <STRING>, "name": <STRING>, "val": <STRING> }
]
```

```ts
const text = (scope: string, name: string) =>
  strings.find(s => s.scope === scope && s.name === name)?.val ?? '';
```

### getCategories

**Request**

```ts
const categories = await SaversAppSDK.getCategories();
```

No parameters. Only active categories are returned; `status: 1` marks a new category (e.g. for a "New" badge).

**Response** — use `slug` with `OffersScreen` (`{ category: slug }`) or the `OFFERS` screen.

```json
[
  {
    "slug": <STRING>,
    "iconUrl": <STRING>,
    "rank": <NUMBER>,
    "status": <NUMBER>,
    "categoryDetail": [
      { "lang": <STRING>, "name": <STRING>, "description": <STRING> }
    ]
  },
  …
]
```

### getHubTiles

**Request**

```ts
const { tiles } = await SaversAppSDK.getHubTiles();
```

No parameters. The SDK sends the member's session when someone is signed in.

**Response** — guest → your program's tiles for guests; signed in → all of that member's tiles.

```json
{
  "tiles": [
    {
      "id": <NUMBER>,
      "name": <STRING>,
      "iconUrl": <STRING>,
      "redirectUrl": <STRING>,
      "rank": <NUMBER>,
      "loginReq": <NUMBER>
    },
    …
  ]
}
```

`loginReq`:

| Value | Meaning | Returned to |
|---|---|---|
| `0` | Everyone | Guests and members |
| `1` | Signed-in members only | Members |
| `2` | Signed-out only (e.g. a Sign in tile) | Guests and members — hide these when signed in |

```tsx
const { isSignedIn } = useSaversApp();
const visibleTiles = tiles.filter(tile => (isSignedIn ? tile.loginReq !== 2 : true));
```

### getTotalEarnings

**Request** — `await SaversAppSDK.getTotalEarnings()` (signed in). No parameters.

**Response**

```json
{ "totalEarned": <STRING> }
```

### getEarnings

**Request** — `await SaversAppSDK.getEarnings()` (signed in). No parameters.

**Response**

```json
[ { "year": <YEAR>, "month": <MONTH>, "totalEarned": <STRING> } ]
```

### getTransactions

**Request** — `await SaversAppSDK.getTransactions()` (signed in). No parameters.

**Response**

```json
{
  "totalNumberOfPages": <NUMBER>,
  "totalNumberOfRecords": <NUMBER>,
  "items": [
    {
      "transactionId": <STRING>,
      "userId": <STRING>,
      "purchaseAmount": <STRING>,
      "rewardAmount": <STRING>,
      "payoutId": <STRING>,
      "status": <STRING>,
      "dateTracked": <DATE_STRING>,
      "dateConfirmed": <DATE_STRING>,
      "datePaid": <DATE_STRING>,
      "dateRejected": <DATE_STRING>
    }
  ]
}
```

### getRecommendations

**Request**

```ts
const page = await SaversAppSDK.getRecommendations({ limit: 10, offset: 0 });
```

| Param | Type | Required | Default |
|---|---|---|---|
| `limit` | number | No | `10` |
| `offset` | number | No | `0` |

**Response**

```json
{
  "totalNumberOfPages": <NUMBER>,
  "totalNumberOfRecords": <NUMBER>,
  "items": [
    {
      "offerId": <STRING>,
      "canonicalBrandId": <STRING>,
      "brandDba": <STRING>,
      "brandLogo": <STRING>,
      "brandLogoSm": <STRING>,
      "reward": <STRING>,
      "title": <STRING>,
      "redemptionType": <STRING>,
      "storeDetails": [
        {
          "id": <STRING>,
          "name": <STRING>,
          "address1": <STRING>,
          "city": <STRING>,
          "state": <STRING>,
          "postCode": <STRING>,
          "countryCode": <STRING>,
          "phone": <STRING>,
          "isOnline": <BOOLEAN>,
          "geoLocation": { "latitude": <FLOAT>, "longitude": <FLOAT> },
          "supportedSchemes": <STRING>[]
        }
      ]
    }
  ]
}
```

### getFavoriteOffers

**Request**

```ts
const page = await SaversAppSDK.getFavoriteOffers({ limit: 10, offset: 0 });
```

| Param | Type | Required | Default |
|---|---|---|---|
| `limit` | number | No | `10` |
| `offset` | number | No | `0` |

**Response** — only offers that haven't ended.

```json
{
  "totalNumberOfPages": <NUMBER>,
  "totalNumberOfRecords": <NUMBER>,
  "items": [
    {
      "offerId": <STRING>,
      "canonicalBrandId": <STRING>,
      "brandDba": <STRING>,
      "brandLogo": <STRING>,
      "brandLogoSm": <STRING>,
      "reward": <STRING>,
      "title": <STRING>,
      "redemptionType": <STRING>,
      "storeDetails": [ … same as getRecommendations … ]
    }
  ]
}
```

---

## Host ↔ Hosted App Communication

Your app and the hosted Savers app talk to each other through **WebView messages**, and the SDK handles them for you: passing the session, switching screens, back navigation, sign-in requests, opening the travel portal, the phone dialer, maps and external links. You don't need to send or listen to these messages yourself. Just provide the `SaversAppProvider` callbacks.

---

## Error Handling

Errors are `SDKError` instances with a `code` (and `statusCode` for API errors):

```ts
import { SDKError } from '@savers_app/react-native-sdk';

try {
  await SaversAppSDK.getTotalEarnings();
} catch (e) {
  if (e instanceof SDKError) console.warn(e.code, e.statusCode, e.message);
}
```

**How errors reach you.** `initialize` itself never throws. Setup errors (bad keys, wrong environment, App ID check failed) reach you two ways:

1. `SaversAppProvider`'s `onConfigError(error)`, called once, and
2. every `SaversAppSDK` call rejects with that same error.

Everything else is a rejected promise from the call you made.

**Setup errors** — via `onConfigError` + every SDK call:

| Code | Cause | Fix |
|---|---|---|
| `MISSING_KEYS` | `apiKey`, `encryptionKey` or `appId` not passed | Pass all three |
| `INVALID_ENVIRONMENT` | `environment` isn't `'sandbox'` / `'production'` | Fix the value |
| `INVALID_APP_ID` (404) | App ID not registered for this partner + bundle ID / package | Check credentials and registered bundle ID ([Step 0](#step-0--getting-access--partner-registration--credentials)) |
| `MISSING_DEVICE_INFO` | `react-native-device-info` not installed (needed for the App ID check) | Install it ([Step 1](#step-1--install)) |
| `statusCode` 401 / 403 | API key or Encryption Key rotated, revoked or wrong | Fetch fresh keys from your backend; if still failing, contact Savers App support |

Network errors during the App ID check are **not** reported through `onConfigError`. The SDK keeps retrying in the background. SDK calls wait for it up to **15 seconds**, then reject with `NETWORK_ERROR`. Once the device is back online, the check finishes and later calls work as normal.

**Call errors** — rejected promise from that call:

| Code | Cause | Fix |
|---|---|---|
| `EMAIL_ALREADY_IN_USE` (409) | `initializeUserSession`: the email belongs to another member | Ask the member to sign in with their existing account, or use another email |
| `NO_ACCESS_TOKEN` | A member-only function (`getTotalEarnings`, `getEarnings`, `getTransactions`, `getRecommendations`, `getFavoriteOffers`) called with nobody signed in | Check `isSignedIn` first, or call `requestSignIn()` |
| `SESSION_EXPIRED` | The session expired and the SDK couldn't renew it silently (e.g. after an app restart without `initializeUserSession`) | Call `initializeUserSession` again |
| `NETWORK_ERROR` | No connection / request failed, or the App ID check is still unanswered after 15 s (e.g. offline at launch) | Retry, or show an offline message |

**Render error:**

| Code | Cause | Fix |
|---|---|---|
| `NOT_INITIALIZED` | `SaversAppProvider` rendered before `initialize` | Call `initialize` first |

---

## Deprecation Notes & Migration

Deprecated in **1.4.0** — these still work, but will be removed in a future major version:

| Deprecated | Use instead |
|---|---|
| `initialize({ apiKey, encryptionKey, pRefCode, signingKey?, environment: 'SANDBOX' \| 'PROD' })` | `initialize({ apiKey, encryptionKey, appId, environment: 'sandbox' \| 'production' })` |
| `initialized(...)` | `initialize(...)` |
| `generateUrl({ screen })` + `HostedAppComponent` | `SaversAppProvider` + ready-made screens / `HostedAppScreen` |

**Breaking changes in 1.4.0** — check these before upgrading:

- **Import from the `SaversAppSDK` bundle only.** 1.3.1 also exported functions by name (e.g. `import { generateUrl, initializeUserSession } from '@savers_app/react-native-sdk'`). These named imports are gone — call them through the bundle instead:

  ```diff
  - import { generateUrl } from '@savers_app/react-native-sdk';
  - await generateUrl({ screen });
  + import { SaversAppSDK } from '@savers_app/react-native-sdk';
  + await SaversAppSDK.generateUrl({ screen });
  ```

  Components (`SaversAppProvider`, the screens, `HostedAppComponent`), `useSaversApp`, `SDKError` and the types are still exported by name.
- **`SaversAppSDK.closeSession()` now fully signs the member out.** In 1.3.1 it only signed the hosted web page out. Now it also ends the backend session and deletes the stored session. Call it only on logout.

**New in 1.4.1 — certificate pinning.** Install `react-native-ssl-public-key-pinning` (then `cd ios && pod install` / a new native build) — see [Step 1](#step-1--install). Nothing else to change: `initialize` sets it up. If your app already uses that library for its own domains, pass `sslPinning: 'app'` and merge `SaversAppSDK.getSslPinningConfig()` into your `initializeSslPinning` call. Without the library the SDK still works, unpinned (with a warning).

Behaviour changes to know when migrating:

- **`initialize` no longer throws or needs `await`.** Config problems arrive via `onConfigError` / rejected SDK calls.
- **No `pRefCode`.** It comes from your App ID.
- **The legacy `initialize` options skip the App ID check.** With legacy options and no `environment`, the default is unchanged from 1.3.1 (production).
- **No more single-use URLs.** The provider loads the hosted app once. There's no 60-second URL expiry or URL regeneration to manage.
- **Session handoff is automatic.** No `qP` / `authSessionId` in URLs.
- **Travel portal is built in.** No `open_travel` handling on your side.

**Corrections to the 1.3.1 README.** It documented a few APIs and behaviours that didn't match the SDK. If you relied on them, here's what's true:

| 1.3.1 README said | Actual API |
|---|---|
| `SdkEnvironment.sandbox` / `.prod` | Legacy: `'SANDBOX'` / `'PROD'` · New: `'sandbox'` / `'production'` |
| `clearUserSession()` | `closeSession()` |
| `getRecommendations(limit, offset)` | `getRecommendations({ limit, offset })` |
| `getFavouriteOffers(limit, offset)` | `getFavoriteOffers({ limit, offset })` (new in 1.4.0) |
| `HostedAppController.closeSession()` | `closeSession()` on the `HostedAppComponent` ref |
| `onSaversSdkMessage` prop | Not needed — the SDK handles messages itself |
| "The SDK never holds a refresh token" | It does: the refresh token is kept **in memory only** (never on disk) and is gone when the app restarts. See [Session & Login](#session--login). |

Minimal migration:

```diff
- await SaversAppSDK.initialize({ apiKey, encryptionKey, pRefCode, environment: 'SANDBOX' });
+ SaversAppSDK.initialize({ apiKey, encryptionKey, appId, environment: 'sandbox' });

- const url = await SaversAppSDK.generateUrl({ screen: { name: 'CASHBACK' } });
- <HostedAppComponent saversAppUrl={url} navigationRef={navigationRef} />
+ <SaversAppProvider onSignInRequired={…}>          {/* once, at the root */}
+   …
+   <Stack.Screen name="Cashback" component={CashbackScreen} options={{ headerShown: false }} />
```

---

## Example App

```bash
# From the repo root: install dependencies (root + example workspace)
yarn install

# Then run the example app from its folder
cd example
yarn start        # Metro bundler (keep it running)
yarn ios          # in a second terminal (react-native run-ios), or
yarn android      # (react-native run-android)
```

The example shows the full flow: keys screen → `initialize` → `SaversAppProvider` → Explore (categories) → Hub (tiles) → ready-made screens → sign-in route → `closeSession`.

Demo notes:

- The keys screen takes your **sandbox** API Key, Encryption Key and App ID, then calls `initialize` + `registerDevice`. The last keys are refilled when you go back to it (Explore's back arrow); a full restart clears them.
- The example's bundle ID / package is `com.saversapp.sdk.example` — register it for the App ID you test with, or the App ID check fails (`INVALID_APP_ID`).
- Sign in from Explore ("Sign in or join") or when a Savers screen asks for it; **Open API Tests** is enabled once signed in. Sign out from the Hub.
- After changing keys or native code, do a full app restart instead of relying on Fast Refresh — native / session state can get out of sync with a hot reload.

## Notes

- `encryptionKey` is the base64 key issued with your credentials — use it exactly as issued.
- Location is optional: pass `location` to `registerDevice` if your app already has it. The SDK doesn't read location or request permission itself — see [Step 2](#step-2--platform-setup).
- The session ID is stored with `react-native-keychain` (Keychain / Keystore). `@react-native-async-storage/async-storage` is a separate, required dependency for the SDK's other, non-sensitive local state — it doesn't hold the session ID.
- This package is public on npm; it only works with valid partner credentials (see [Step 0](#step-0--getting-access--partner-registration--credentials)). Installing it doesn't grant access to any Savers App data.
