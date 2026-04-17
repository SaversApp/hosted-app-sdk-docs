# Savers App Android Native Library

SaversSDK is an Android library that bridges WebView messages to native features such as Maps, Dial Pad, browser navigation, session management, device ID, location, onboarding API, and encrypted URL generation for Savers platform payloads.

## Features
- Web-to-native message handling with WebView
- Map navigation with geo intents and browser fallback
- Dial pad launch and direct call validation
- External browser navigation
- Device ID retrieval
- Session management and local caching
- Location provider with runtime permission handling
- Encrypted URL generator (AES‑GCM) for Savers platform payloads
- Structured logging via Logger

## Project Layout
- Library module: `saversapp-sdk` (folder: `app`)
- Sample app: `sample`

Key modules (mirroring RN-style src layout):
- WebView bridge: `sdk/services/webview/messageHandler.kt`
- Navigation: `sdk/services/navigation/openMap.kt`, `dialPad.kt`, `openBrowser.kt`, `goBackNavigation.kt`
- Device: `sdk/services/device/location.kt`
- URL generator: `sdk/services/url/urlGenerator.kt`
- Storage: `sdk/data/storage/KeysManager.kt`, `SessionManager.kt`, `DeviceIdManager.kt`, `StorageManager.kt`, `StorageKeys.kt`
- Network: `sdk/data/network/apiClient.kt`, `apiService.kt`, `apiEndpoints.kt`, `apiExceptionHandler.kt`, and `interceptors/*`
- Core: `sdk/core/runtime.kt`
- Utils: `sdk/utils/logger.kt`, `sdk/utils/encryption.kt`, `sdk/utils/validator.kt`, `sdk/utils/errors.kt`

## Getting Started

### Integrate the Library
- Open the project in Android Studio.
- Include the library module in your app.

```kotlin
dependencies {
  implementation(project(":saversapp-sdk"))
}
```

### Initialize the SDK
```kotlin
import com.saversapp.sdk.core.SaversAppSDK

SaversAppSDK.initialize(
  context = applicationContext,
  apiKey = "YOUR_API_KEY",
  encryptionKey = "BASE64_32_BYTE_KEY", // base64 of 32 bytes (256-bit)
  pRefCode = "YOUR_P_REF_CODE",
  authMode = "PHONE" // or "EMAIL" | "USERNAME"
)
```

Notes:
- encryptionKey must be a base64-encoded 32-byte key (AES‑256).
- pRefCode is required for URL generation to target the correct program.

### Use with WebView
Attach a JavaScript interface named `WebView` and forward messages to `messageHandler.handleWebMessage` (mirrors the RN bridge naming for shared HTML).

```kotlin
import android.webkit.JavascriptInterface
import android.webkit.WebView
import android.webkit.WebSettings
import com.saversapp.sdk.services.webview.messageHandler
import org.json.JSONObject

fun setupWebViewBridge(webView: WebView) {
  webView.settings.javaScriptEnabled = true
  webView.settings.domStorageEnabled = true
  webView.settings.cacheMode = WebSettings.LOAD_DEFAULT
  webView.addJavascriptInterface(object {
    @JavascriptInterface
    fun postMessage(raw: String) {
      messageHandler.handleWebMessage(webView.context, raw) { data ->
        val payload = JSONObject(data as Map<*, *>).toString()
        webView.post {
          webView.evaluateJavascript("document.dispatchEvent(new MessageEvent('message', { data: $payload }));", null)
        }
      }
    }
  }, "WebView")
}
```

HTML example (inside the WebView):

```html
<!doctype html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    body{font-family:sans-serif;margin:0;padding:8px 0}
    .scroll{overflow-x:auto;white-space:nowrap;padding:8px 16px;box-sizing:border-box}
    .btn{display:inline-block;padding:12px 16px;font-size:16px;margin-right:8px;border-radius:8px;border:1px solid #ccc;background:#f5f5f5}
    .btn:active{background:#e9e9e9}
  </style>
</head>
<body>
  <div class="scroll">
    <button class="btn" onclick="window.WebView.postMessage(JSON.stringify({action:'OPEN_MAP',payload:{lat:37.7749,lng:-122.4194,label:'SaverAppsPoint'}}))">Open Map</button>
    <button class="btn" onclick="window.WebView.postMessage(JSON.stringify({action:'SHOW_DIAL_PAD',payload:{number:'+1234567890'}}))">Open Dialpad</button>
    <button class="btn" onclick="window.WebView.postMessage(JSON.stringify({action:'MERCHANT_PORTAL_REDIRECT',payload:{url:'https://www.google.com'}}))">Open Browser</button>
    <button class="btn" onclick="window.WebView.postMessage(JSON.stringify({action:'SESSION_ID',payload:{sessionId:'123456789'}}))">Set Session ID</button>
    <button class="btn" onclick="window.WebView.postMessage(JSON.stringify({action:'END_SESSION'}))">Close Screen</button>
  </div>
  <script>
    document.addEventListener('message', function(e) {
      try { const data = JSON.parse(e.data); console.log('postBack', data); } catch(_) {}
    });
  </script>
</body>
</html>
```

### Supported Actions (messageHandler)
- `OPEN_MAP`: `{ lat: number, lng: number, label?: string }`
- `SHOW_DIAL_PAD`: `{ number?: string }`
- `MERCHANT_PORTAL_REDIRECT`: `{ url: string }`
- `SESSION_ID`: `{ sessionId?: string }`
- `END_SESSION`: no payload

Each action returns a postBack JSON:
```json
{ "ok": true, "action": "OPEN_MAP" }
```
or
```json
{ "ok": false, "action": "OPEN_MAP", "error": "..." }
```

### URL Generator
Generate a signed and encrypted Savers URL. Defaults mirror the RN SDK.

```kotlin
import com.saversapp.sdk.data.storage.SessionManager
import com.saversapp.sdk.services.url.urlGenerator
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.launch

val sessionManager = SessionManager(applicationContext)
sessionManager.setSessionId("YOUR_SESSION_ID")

// call inside a coroutine (suspend function)
lifecycleScope.launch {
  val url = urlGenerator.generate(
    context = applicationContext,
    profile = mapOf(
      "userId" to "USER-12345",
      "firstname" to "Jane",
      "lastname" to "Doe",
      "phone" to "+1234567890",
      "email" to "jane.doe@example.com",
      "pv" to "1",
      "ev" to "1"
    ),
    authType = "PHONE", // or "EMAIL" | "USERNAME"
    screen = mapOf(
      "name" to "Default Explore",
      "attributes" to listOf(mapOf("key" to "KEY", "value" to "VALUE"))
    ),
    sessionManager = sessionManager
  )
  // use url: https://m.saversapp.com/?pRefCode=...&qP=...
}
```

Profile requirements:
- `userId` and `email` are mandatory
- `pv` and `ev` are optional (`'0'` or `'1'`)
- `phone` is mandatory when `authType` is `'PHONE'`
- `username` is mandatory when `authType` is `'USERNAME'`

Screen requirements:
- `screen` is optional; defaults to `{ name: 'Explore' }`
- When `screen.name === 'OfrDetails'`, `screen.attributes` must be provided and non-empty

Location / coordinates:
- If you set coordinates via the SDK’s location manager, they are included in the encrypted payload as `deviceInfo.location`
- If not set, the URL is still generated and `location` is omitted

Notes:
- `authType` defaults to `'PHONE'` (also accepts `'EMAIL' | 'USERNAME'`)
- `sessionId` is taken from `SessionManager` when not provided
- The SDK requires `encryptionKey` and `pRefCode` initialized via `SaversAppSDK.initialize`; generation throws if either is missing

### Onboarding API (optional)
```kotlin
import com.saversapp.sdk.data.network.ApiService
import com.saversapp.sdk.data.models.OnboardingModel
import kotlinx.coroutines.flow.collect
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.launch

val api = ApiService(applicationContext)
GlobalScope.launch {
  api.getOnBoarding().collect { resp ->
    if (resp.status) {
      val count = OnboardingModel.imageCount(resp.data)
      // use onboarding data...
    }
  }
}
```

## Permissions
Add these to your app manifest if you use location or WebView features:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

## Requirements
- Android minSdkVersion 24+ (library build config)
- AES‑GCM encryption requires a base64‑encoded 32‑byte (256‑bit) key

## Sample App
Run the sample module:

```bash
./gradlew :sample:installDebug
```

## Publishing
Publish the library to Maven Central via Sonatype:

```bash
./gradlew publishAndReleaseToCentral
```

Prerequisites:
- Central User Token configured as sonatypeUsername/sonatypePassword (or ossrhUsername/ossrhPassword)
- GPG signing keys configured: signing.keyId, signing.password, signing.key

## Notes
- Ensure WebView JavaScript and your JS bridge are enabled.
- Avoid logging sensitive values in production.
- Some features (Maps, Dial Pad, Location) rely on device capabilities and permissions.
