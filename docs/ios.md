# SaversAppSDK iOS Native Library

SaversAppSDK provides a bridge between WKWebView and native features (Maps, Dial Pad, Browser), plus utilities for device ID, session management, location, and URL generation.

Repository: `https://github.com/SaversApp/hosted-app-ios-sdk-distribution.git`

Package name: `SaversAppSDK`

## Features
- Web-to-native message handling via WKWebView
- Apple Maps navigation
- Dial pad launch
- In-app browser (SFSafariViewController) or system browser
- Device ID retrieval
- Session management
- Location provider with permission handling and fallbacks
- URL generation for Savers platform payloads
- Simple logging via `Logger` (prints to console)

## Project Layout
- Library: `SaversAppSDK`
- Demo app: `SaversAppSDKDemo`

Key modules:
- Webview bridge: messageHandler.swift
- Navigation: openMap.swift, dialPad.swift, openBrowser.swift, goBackNavigation.swift
- Core/runtime: runtime.swift
- Device/location: location.swift
- Permissions: permissionManager.swift
- URL generation: urlGenerator.swift
- Storage/keys/session: DeviceIdManager.swift, KeysManager.swift, SessionManager.swift
- Networking (data layer): apiClient.swift, interceptors, repository, service
- Utilities: Encryption.swift, Logger.swift, Validator.swift

## Installation

### Swift Package Manager (Recommended)
- Xcode: File → Add Packages… → enter your repository URL → select a version tag (e.g., 1.0.0) → Add Package → add SaversAppSDK to your app target.
- Package.swift example:

```swift
dependencies: [
  .package(url: "https://github.com/SaversApp/hosted-app-ios-sdk-distribution.git", from: "1.0.0")
],
targets: [
  .target(
    name: "YourApp",
    dependencies: ["SaversAppSDK"]
  )
]
```

### CocoaPods
- Podfile (published to trunk):

```ruby
platform :ios, '13.0'
use_frameworks!
target 'YourApp' do
  pod 'SaversAppSDK', '~> 1.0'
end
```

- Podfile (direct from Git tag):

```ruby
platform :ios, '13.0'
use_frameworks!
target 'YourApp' do
  pod 'SaversAppSDK', :git => 'https://github.com/SaversApp/hosted-app-ios-sdk-distribution.git', :tag => '1.0.0'
end
```

Notes:
- iOS 13+ minimum due to CryptoKit.
- After installation, import with `import SaversAppSDK`.

## Getting Started

### Integrate the Library
- Open `SaversAppSDK.xcodeproj` and add it to your workspace or project.
- Add target dependency on `SaversAppSDK` to your app target.
- Embed `SaversAppSDK.framework` into your app target (Embed & Sign).

### iOS Setup
- Permissions (add keys and descriptions in Info.plist):
  - `NSLocationWhenInUseUsageDescription`
  - `NSLocationAlwaysAndWhenInUseUsageDescription` (optional; add if you request Always)
- Dialer schemes (allow canOpenURL for tel/telprompt):

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>tel</string>
  <string>telprompt</string>
</array>
```

Example:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>We need your location to provide personalized services</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>We need your location to provide personalized services</string>
```

### Initialize the SDK
```swift
import SaversAppSDK

try SaversAppSDK.initialize(
  apiKey: "YOUR_API_KEY",
  encryptionKey: "YOUR_ENCRYPTION_KEY_BASE64", // base64 of 32 bytes (256-bit)
  pRefCode: "YOUR_P_REF_CODE",
  authMode: "PHONE" // or "EMAIL" | "USERNAME"
)
```

### WKWebView Bridge
Configure a `WKWebView` with the `savers` message handler and forward messages to `MessageHandler.handle`.

```swift
import WebKit
import SaversAppSDK

final class ExampleViewController: UIViewController, WKScriptMessageHandler {
  private var webView: WKWebView!

  override func viewDidLoad() {
    super.viewDidLoad()

    let controller = WKUserContentController()
    controller.add(self, name: "savers")

    let config = WKWebViewConfiguration()
    config.userContentController = controller
    webView = WKWebView(frame: .zero, configuration: config)
    view.addSubview(webView)

    webView.loadHTMLString(html(), baseURL: nil)
  }

  func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
    let raw = message.body as? String
    MessageHandler.handle(raw: raw, presenter: self) { [weak self] data in
      guard let json = try? JSONSerialization.data(withJSONObject: data),
            let payload = String(data: json, encoding: .utf8) else { return }
      self?.webView.evaluateJavaScript("document.dispatchEvent(new MessageEvent('message', { data: \(payload) }));")
    }
  }
}
```

Supported actions:
- `OPEN_MAP`: `{ lat: number, lng: number, label?: string }`
- `SHOW_DIAL_PAD`: `{ number?: string }`
- `MERCHANT_PORTAL_REDIRECT`: `{ url: string }`
- `SESSION_ID`: `{ sessionId?: string }`
- `END_SESSION`: no payload

### HTML Showcase
The demo uses this simple HTML to trigger native actions via the iOS WKWebView bridge:

```html
<!doctype html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    body{font-family:-apple-system,sans-serif;margin:0;padding:8px 0}
    .scroll{overflow-x:auto;white-space:nowrap;padding:8px 16px;box-sizing:border-box}
    .btn{display:inline-block;padding:12px 20px;font-size:16px;margin-right:8px;border-radius:8px;border:none;background:#2563EB;color:white;font-weight:500}
    .btn:active{background:#1d4ed8}
  </style>
  <script>
    function sendNative(action, payload) {
      const msg = JSON.stringify({ action, payload });
      window.webkit?.messageHandlers?.savers?.postMessage(msg);
    }
  </script>
  </head>
<body>
  <div class="scroll">
    <button class="btn" onclick="sendNative('OPEN_MAP',{lat:37.7749,lng:-122.4194,label:'SaverAppsPoint'})">Open Map</button>
    <button class="btn" onclick="sendNative('SHOW_DIAL_PAD',{number:'+971504945159'})">Open Dialpad</button>
    <button class="btn" onclick="sendNative('MERCHANT_PORTAL_REDIRECT',{url:'https://www.google.com'})">Open Browser</button>
    <button class="btn" onclick="sendNative('SESSION_ID',{sessionId:'123456789'})">Set Session ID</button>
    <button class="btn" onclick="sendNative('END_SESSION')">Close Screen</button>
  </div>
  <script>
    document.addEventListener('message', function(e) {
      try {
        const data = JSON.parse(e.data);
        console.log('postBack', data);
      } catch(err) {}
    });
  </script>
</body>
</html>
```

### URL Generation
Generate a Savers URL that encodes profile, screen, device info, and session payload. If coordinates are set via `LocationManager`, they are included automatically.

```swift
let session = SessionManager()
session.setSessionId("YOUR_SESSION_ID")

let screen: [String: Any] = [
  "name": "Default Explore",
  "attributes": [["key": "KEY", "value": "VALUE"]]
]

let profile: [String: Any] = [
  "userId": "USER-12345",
  "firstname": "Jane",
  "lastname": "Doe",
  "phone": "+1234567890",
  "email": "jane.doe@example.com",
  "pv": "1",
  "ev": "1"
]

// Optional: include coordinates in deviceInfo for the encrypted payload
LocationManager.setLocationCoordinates(37.7749, -122.4194)

let url = try UrlService.generate(
  profile: profile,
  authType: "PHONE", // or "EMAIL" | "USERNAME"
  screen: screen,
  sessionId: session.getSessionId()
)
```

Profile requirements:
- userId and email are mandatory
- pv and ev are optional ('0' or '1')
- phone is mandatory when authType is 'PHONE'
- username is mandatory when authType is 'USERNAME'

Screen requirements:
- screen is optional; defaults to { name: 'Explore' } when omitted
- when screen.name is 'OfrDetails', screen.attributes must be provided and non-empty

Location / coordinates:
- if LocationManager coordinates are set, they are included as deviceInfo.location
- if no coordinates are set, the URL is still generated and location is omitted

Notes:
- authType is optional; defaults to 'PHONE' | also 'EMAIL' | 'USERNAME'
- sessionId is optional; when nil, the SDK uses its stored session id
- requires encryptionKey (base64 32-byte) and pRefCode initialized via SaversAppSDK.initialize
- the SDK automatically fetches a nonce during URL generation using profile.userId and pRefCode

## Device & Session Management
- Device ID: `try DeviceIdManager.getDeviceId()`
- Session ID: 

```swift
let sm = SessionManager()
sm.setSessionId("123456789")
let current = sm.getSessionId()
```

## Location & Coordinates
- Ask for permission when needed; the SDK uses `PermissionManager` internally.
- Include coordinates for URL encryption:

```swift
LocationManager.setLocationCoordinates(37.7749, -122.4194)
```

To resolve device location at runtime (uses CoreLocation and permission prompts):

```swift
Location.getDeviceLocation { result in
  switch result {
  case .success(let coord):
    if let c = coord { print("lat=\(c.latitude), lng=\(c.longitude)") }
  case .failure(let error):
    print("location error: \(error)")
  }
}
```

## Maps, Dialer & Browser Helpers
```swift
// Maps
try? OpenMap.open(lat: 37.7749, lng: -122.4194, label: "San Francisco")

// Dial pad / call
DialPad.open("+1234567890")
try? DialPad.call("+1234567890")

// Browser (SFSafariViewController if presenter provided, else UIApplication)
OpenBrowser.open("https://www.example.com", presenter: self)
```

## Networking (ApiService)
Use the built-in data layer with interceptors and caching strategies.

```swift
let api = SaversData.ApiService() // defaults to production environment
api.getOnBoarding { res in
  if res.status, let data = res.data {
    // data.day?.images / data.night?.images
    print("fromCache=\(res.fromCache)")
  } else {
    print("error: \(res.message)")
  }
}
```

Cache strategies:
- `.normal`: returns cache immediately (if present) and fetches network afterward
- `.cache`: prefer cache; falls back to network
- `.cacheOnly`: return only cached data
- `.network`: always request from network

Auth and request enrichment:
- x-api-key header is injected automatically
- oauthSubId is appended to query and JSON body for POST/PUT/PATCH when available

## Logging
Logs are printed via `Logger` and appear in the device/simulator console. Examples:
- `[SDK] Device ID available`
- `[SDK] Permission OK, requesting location + starting updates`
- `[Demo] Loading HTML into WKWebView`

## Demo App
Open `SaversAppSDK.xcodeproj` and run the `SaversAppSDKDemo` scheme.

CLI build example:
```bash
xcodebuild -project SaversAppSDK.xcodeproj -scheme SaversAppSDKDemo -configuration Debug -destination "platform=iOS Simulator,name=iPhone 15" CODE_SIGNING_ALLOWED=NO build
```

## Notes
- Ensure WKWebView includes the `savers` message handler.
- Provide valid keys in `SaversAppSDK.initialize` for URL generation (apiKey, encryptionKey base64, pRefCode, authMode).
- Device location depends on permissions and platform availability (simulator may require a simulated location).
- Requirements: iOS 13+; encryption uses AES‑256‑GCM with a base64‑encoded 32‑byte key.

## Troubleshooting
- If `aesEncrypt` throws, verify `encryptionKey` is base64 of 32 bytes (256‑bit).
- If dialer does not open, ensure `LSApplicationQueriesSchemes` includes `tel` and `telprompt`.
- If location never resolves on Simulator, set a simulated location in Xcode.
- If `OPEN_MAP` fails, validate coordinates are within valid ranges.
- If the web bridge is not receiving postBack, ensure you dispatch the `message` event in `WKWebView` as shown above.
