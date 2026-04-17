# Savers App SDK (Flutter)

Savers App SDK is a Flutter package that bridges web and native actions, provides device and session utilities, generates Savers URLs, and exposes navigation helpers for maps, dialer, and browser flows.

## Features
- Web message handling for native actions
- Open maps by coordinates with browser fallback
- Dial pad launch and direct call support
- External browser navigation
- Device ID retrieval and storage
- Session key management
- Device location access
- Savers URL generator
- HTTP request helper

## Getting Started

Add the package to your `pubspec.yaml`:

```yaml
dependencies:
  savers_app_sdk:
    path: ../savers_app_sdk
```

## Platform Setup

Android `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
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

## Usage

Initialize the SDK:

```dart
import 'package:savers_app_sdk/savers_app_sdk.dart';

void main() {
  SaversAppSDK.initialized(
    apiKey: 'YOUR_API_KEY',
    encryptionKey: 'YOUR_ENCRYPTION_KEY',
  );
  runApp(const App());
}
```

Enable navigation helpers:

```dart
import 'package:flutter/material.dart';
import 'package:savers_app_sdk/savers_app_sdk.dart';

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      navigatorKey: navigationKey,
      home: const HomeScreen(),
    );
  }
}
```

Open native features:

```dart
await openMap(37.7749, -122.4194, 'San Francisco');
await openDialPad('+1234567890');
await openBrowser('https://www.example.com');
```

Generate Savers URLs:

```dart
final input = UrlInput(
  pRefId: 'PROGRAM_CODE',
  authType: AuthType.phone,
  profile: const Profile(
    firstname: 'Jane',
    lastname: 'Doe',
    phone: '+15555550100',
    pv: '1',
    ev: '1',
  ),
  screen: const Screen(
    name: 'Explore',
    attributes: [Attribute(key: 'KEY', value: 'VALUE')],
  ),
);

final url = await generateUrl(input);
```

Web message handling:

```dart
import 'dart:convert';
import 'package:savers_app_sdk/savers_app_sdk.dart';

void onWebMessage(String raw) {
  handleWebMessage(
    raw,
    postBack: (data) {
      final payload = jsonEncode(data);
    },
  );
}
```

Supported actions:
- OPEN_MAP { lat, lng, label }
- SHOW_DIAL_PAD { number? }
- MERCHANT_PORTAL_REDIRECT { url }
- SESSION_ID { sessionId }
- CLOSE_CURRENT_SCREEN

API request helper:

```dart
final response = await apiRequest<Map<String, dynamic>>(
  options: ApiRequestOptions(
    url: 'https://api.example.com/v1/status',
    method: 'GET',
    apiKey: 'YOUR_API_KEY',
  ),
);
```

## Example App

```bash
cd example
flutter pub get
flutter run
```

## Notes
- Some features require location permissions on Android and iOS.
- Add your own storage namespace with `createStorageNamespace` if you need scoped keys.
