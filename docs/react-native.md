# Savers App React Native SDK

React Native SDK that bridges host app features like maps, dialer, browser, device info, session storage, URL generation, navigation, and WebView event handling. It also includes a robust networking layer.

## Features
- Open native maps by coordinates with graceful browser fallback
- Dial pad invocation with optional number and direct call helper
- External browser navigation from the host app or from web content
- Close current screen via global `navigationRef` (React Navigation) using **END_SESSION**
- Device ID retrieval (via `react-native-device-info`)
- Optional coordinates enrichment for URL payload
- Session and keys management (API key, encryption key, program ref code, auth mode)
- URL generator to compose signed and encrypted Savers mobile URLs
- AES-GCM encryption for URL payload (AES-256-GCM)
- WebView message handler to trigger native actions from web (maps, dialer, browser, session id, end session)
- Configurable networking layer with interceptors (logging, network connectivity, auth, retry, error formatting)

## Installation
- Install the library:

```bash
npm install @savers_app/react-native-sdk
# or
yarn add @savers_app/react-native-sdk
```

- Install required peer dependencies:

```bash
npm install @react-native-async-storage/async-storage \
  @react-native-community/netinfo \
  @react-navigation/native \
  react-native-device-info \
  react-native-aes-gcm-crypto
# or
yarn add @react-native-async-storage/async-storage \
  @react-native-community/netinfo \
  @react-navigation/native \
  react-native-device-info \
  react-native-aes-gcm-crypto
```

- Optional (for WebView-based messaging):

```bash
npm install react-native-webview
# or
yarn add react-native-webview
```

## iOS Setup
- Add dialer schemes to Info.plist:

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>tel</string>
  <string>telprompt</string>
</array>
```

Ensure native pods are installed and linked for the peer dependencies.

## Android Setup
- Add permissions in AndroidManifest.xml:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

Link and configure the peer dependencies according to their documentation.
 
Requirements for AES-GCM:
- Android minSdkVersion must be 26 or higher (example app is configured to 26).
- iOS 13 or higher.

## Quick Start

### 1. Initialize the SDK

Before using any other APIs, initialize the SDK with keys you receive from Savers:

```ts
import { SaversAppSDK } from '@savers_app/react-native-sdk';

SaversAppSDK.initialized({
  apiKey: 'YOUR_API_KEY',
  encryptionKey: 'BASE64_256_BIT_ENCRYPTION_KEY',
  pRefCode: 'PROGRAM_REF_CODE',
  authMode: 'CUSTOMER_SIGN_IN_UP_MODE' // 'EMAIL' | 'PHONE'
});
```

The SDK stores these values in memory (not in AsyncStorage), via an internal `keysManager`. You can later read them using:

```ts
import {
  getApiKey,
  getEncryptionKey,
  getPRefCode,
  getAuthMode,
} from '@savers_app/react-native-sdk';
```

### 2. Generate URL

```ts
import { generateUrl } from '@savers_app/react-native-sdk';

const url = await generateUrl({
  // Top-level UrlInput fields
  authType: 'PHONE', // optional, defaults to 'PHONE' | also 'EMAIL' | 'USERNAME'
  profile: {
    // Mandatory fields
    userId: 'USER_ID',
    email: 'jane@example.com',
    firstname: 'Jane',
    lastname: 'Doe',

    // Conditionally mandatory
    phone: '+15555550100', // required when authType === 'PHONE'
    username: 'jane.doe', // required when authType === 'USERNAME'

    // Optional profile flags
    pv: '1', // optional: '0' | '1'
    ev: '1', // optional: '0' | '1'

    // Other optional profile fields
    dob: '1990-01-01',
    city: 'San Francisco',
    zipcode: '94103',
    referrer_user_id: 'REFERRER_ID',
  },
  screen: {
    // Optional screen; defaults to { name: 'Explore' } when omitted
    name: 'Explore',

    // attributes are optional in general, but mandatory when name === 'OfrDetails'
    attributes: [{ key: 'offerId', value: '123' }],
  },
  // sessionId is optional; when omitted, the SDK uses its stored session id
  // deviceInfo is optional; the SDK builds it internally using device id (and optional coordinates)
});
// returns https://m.saversapp.com/?pRefCode=...&qP=...
```

Profile requirements:
- `userId` and `email` are mandatory
- `pv` and `ev` are optional (`'0'` or `'1'`)
- `phone` is mandatory when `authType` is `'PHONE'`
- `username` is mandatory when `authType` is `'USERNAME'`
Nonce requirement:
- The SDK resolves a nonce automatically using `profile.userId` and your `pRefCode`; no manual action is required.

Screen requirements:
- `screen.name` is optional, defaults to `'Explore'`
- When `screen.name === 'OfrDetails'`, `screen.attributes` **must** be provided and non-empty

Location / coordinates:
- If you use the location manager (see **Location & Coordinates** below) and set coordinates,
  they are automatically included in the encrypted payload as `deviceInfo.location`.
- If no coordinates are set, the URL is still generated, and `location` is simply omitted.

Notes:
- The SDK requires `encryptionKey` and `pRefCode` to be initialized via `SaversAppSDK.initialized`; `generateUrl` will throw if either is missing.
- The `qP` payload is encrypted using AES-256-GCM. The encryption key must be a base64-encoded 32-byte (256-bit) value.
- The encrypted payload combines `iv`, `content`, and `tag` as `iv:content:tag` internally before being base64-encoded.

## Navigation (End Session)

The SDK can close the current screen without knowing the host app's navigation structure. This is useful for flows initiated from web content or deep links that need to programmatically exit back to the host app.

One-time host app setup using a global `navigationRef`:

```tsx
import { NavigationContainer, createNavigationContainerRef } from '@react-navigation/native';
import { SaversAppSDK } from '@savers_app/react-native-sdk';

export const navigationRef = createNavigationContainerRef();

export function App() {
  // Initialize the SDK once (e.g., in a root effect or before rendering)
  SaversAppSDK.initialized({
    apiKey: 'YOUR_API_KEY',
    encryptionKey: 'BASE64_256_BIT_ENCRYPTION_KEY',
    pRefCode: 'PROGRAM_REF_CODE',
    authMode: 'EMAIL', // or 'PHONE'
    navigationRef,     // pass the ref to the SDK
  });

  return <NavigationContainer ref={navigationRef}>{/* your navigators */}</NavigationContainer>;
}
```

SDK API:

```ts
import { closeCurrentScreen, closeCurrentScreenSafe } from '@savers_app/react-native-sdk';

closeCurrentScreen(); // returns boolean
closeCurrentScreenSafe(); // Android-only fallback to exitApp if it can't goBack()
```

Notes:
- The same `navigationRef` is passed to the `NavigationContainer` and to `SaversAppSDK.initialized`.
- The SDK stores this ref internally to implement `closeCurrentScreen()` reliably.

## WebView Integration

Trigger native actions from web & bind `onMessage` to a `WebView`:

```ts
import { handleWebMessage } from '@savers_app/react-native-sdk';
import { WebView } from 'react-native-webview';
...

const onMessage = (e: any) => {
  const raw = e?.nativeEvent?.data;
  const postBack = (data: any) => {
    // send data back to the webview as needed
  };
  handleWebMessage(raw, postBack);
};

<WebView
  originWhitelist={['*']}
  source={{ html }}
  onMessage={onMessage}
/>;
```



Supported actions:
- `OPEN_MAP` `{ lat, lng, label }`
- `END_SESSION`
- `SHOW_DIAL_PAD` `{ number? }`
- `MERCHANT_PORTAL_REDIRECT` `{ url }`
- `SESSION_ID` `{ sessionId }`

Note: unified WebView helpers were removed. Use `handleWebMessage` directly with `react-native-webview`.

### Web Messaging (for Web Developers)

Your web page loaded in react-native-webview can trigger native features by posting a JSON message. Use:

```html
<script>
  function sendNative(action, payload) {
    const msg = JSON.stringify({ action, payload });
    window.ReactNativeWebView?.postMessage(msg);
  }

  // Examples:
  // Open map
  sendNative('OPEN_MAP', { lat: 37.7749, lng: -122.4194, label: 'San Francisco' });

  // Dial pad
  sendNative('SHOW_DIAL_PAD', { number: '+1234567890' });

  // Open external browser
  sendNative('MERCHANT_PORTAL_REDIRECT', { url: 'https://www.example.com' });

  // Set session ID
  sendNative('SESSION_ID', { sessionId: '123456789' });

  // End session / close current screen (goBack if possible)
  sendNative('END_SESSION');

  // Receive postBack from native (optional)
  document.addEventListener('message', function (e) {
    try {
      const data = JSON.parse(e.data);
      console.log('Native postBack:', data);
      // { ok: boolean, action: string, ...additional fields }
    } catch (_) {}
  });
</script>
```

Message schema:
- `action`: one of `OPEN_MAP`, `END_SESSION`, `SHOW_DIAL_PAD`, `MERCHANT_PORTAL_REDIRECT`, `SESSION_ID`
- `payload`: object with action-specific fields:
  - `OPEN_MAP`: `{ lat: number, lng: number, label?: string }`
  - `SHOW_DIAL_PAD`: `{ number?: string }`
  - `MERCHANT_PORTAL_REDIRECT`: `{ url: string }`
  - `SESSION_ID`: `{ sessionId: string }`
  - `END_SESSION`: no payload required

## Device & Session Management

The SDK provides helpers for device id, session id, and keys:

```ts
import {
  getDeviceId,
  setSessionId,
  getSessionId,
  getApiKey,
  getEncryptionKey,
  getPRefCode,
  getAuthMode,
} from '@savers_app/react-native-sdk';
```

- `getDeviceId()`: uses `react-native-device-info` to resolve a unique device id and caches it in memory and AsyncStorage.
- `setSessionId(sessionId)` / `getSessionId()`: store and retrieve a session identifier using AsyncStorage.
- `getApiKey()`, `getEncryptionKey()`, `getPRefCode()`, `getAuthMode()`: read values initialized via `SaversAppSDK.initialized`.

## Location & Coordinates

Manual coordinates for URL generation:

   ```ts
   import { setLocationCoordinates, getLocationCoordinates } from '@savers_app/react-native-sdk';

   await setLocationCoordinates(37.7749, -122.4194);
   const coords = await getLocationCoordinates(); // { lat, lng } | null
   ```

   If coordinates are available, `generateUrl` automatically includes them in the encrypted payload as `deviceInfo.location`. If not, the URL is still generated without `location`.

## Maps, Dialer & Browser Helpers

```ts
import {
  openMap,
  openDialPad,
  callPhone,
  openBrowser,
} from '@savers_app/react-native-sdk';
```

- `openMap(lat, lng, label?)`: opens the native maps app if available, otherwise falls back to browser (Apple Maps / Google Maps).
- `openDialPad(number?)`: opens the dial pad; if a number is provided, it pre-fills the dialer.
- `callPhone(number)`: directly attempts to call the number; validates format and throws errors like `INVALID_PHONE_NUMBER` or `CALL_NOT_SUPPORTED`.
- `openBrowser(url)`: opens an external browser with the given URL.

## Networking (ApiService)

Use the `ApiService` to make network requests with unified error handling, caching, and interceptors.

### ApiService Overview

```ts
import { ApiService } from '@savers_app/react-native-sdk';

const apiService = new ApiService(); // defaults to production base URL
```

- `ApiService` uses:
  - `ApiClient` (wraps `fetch` and applies interceptors)
  - `ApiRepository` (adds caching strategies)
  - `ApiEndpoints` (centralized endpoint paths)
  - `ApiResponse<T>` model for response shape

### Onboarding Example

```ts
import { ApiService } from '@savers_app/react-native-sdk';

const apiService = new ApiService();

// Example: fetch onboarding data as an async generator
const fetchOnboarding = async () => {
  try {
    const generator = apiService.getOnBoarding();
    for await (const response of generator) {
      if (response.status) {
        console.log('Data:', response.data);
      } else {
        console.error('Error:', response.message);
      }
    }
  } catch (error) {
    console.error('Request failed:', error);
  }
};
```

- Responses use the shape: `ApiResponse<T> = { data?: T | null; status?: boolean; message?: string; fromCache: boolean }`.
- `CacheStrategy` controls whether data comes from cache, network, or both.

Interceptors:
- `LoggingInterceptor`: logs requests and responses to the console.
- `NetworkInterceptor`: uses `@react-native-community/netinfo` to detect no-internet conditions and throws a unified error.
- `RetryInterceptor`: retries failed requests a configurable number of times.
- `ErrorInterceptor`: maps errors to user-friendly messages.
- `AuthInterceptor`: placeholder for attaching auth / user information to requests.
## Example App

This repository includes an example React Native app that demonstrates most of the SDK features end‑to‑end:

- SDK initialization (`SaversAppSDK.initialized`)
- Device id, keys, session id, and coordinates usage
- URL generation (`generateUrl`) and opening the resulting URL in a browser
- Navigation and end‑session flow via `navigationRef` and `END_SESSION`
- WebView actions (`OPEN_MAP`, `SHOW_DIAL_PAD`, `MERCHANT_PORTAL_REDIRECT`, `SESSION_ID`, `END_SESSION`)
- Networking with `ApiService.getOnBoarding()`

To run the example:

```bash
# Install dependencies (root + example workspace)
npm install
# or
yarn install

# Start the Metro bundler for the example app
yarn workspace @saversapp/react-native-sdk-example start

# Run the example app on a device / simulator (from the repo root)
yarn run:ios     # uses scripts.run:ios from package.json
yarn run:android # uses scripts.run:android from package.json
```


## Troubleshooting
- Ensure peer dependencies are installed and linked (AsyncStorage, NetInfo, DeviceInfo, WebView, React Navigation).
- If `SaversAppSDK.initialized` logs missing modules, install the indicated packages.
- If AES‑GCM encryption fails, verify that `encryptionKey` is a base64‑encoded 32‑byte (256‑bit) value and that your Android/iOS versions meet the minimum requirements.

## License
MIT
