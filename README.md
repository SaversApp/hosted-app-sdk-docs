# Savers App SDK Documentation

SDK that bridges host app features like maps, dialer, browser, device info, session storage, URL generation, navigation, and WebView event handling. It also includes a robust networking layer.

---

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
- Configurable networking layer with interceptors (logging, network connectivity, auth, retry, error 

---

## Platforms

| Platform | Version | Distribution |
|---|---|---|
| Android | 1.0.1 | Maven Central |
| iOS | 1.0.2 | Github |
| React Native | 1.2.7 | npm |
| Flutter | 1.0.0 | pav.dev |

---

## Requirements

### Android
- Min SDK: 24
- Kotlin: 1.8+

### iOS
- iOS 13+
- Xcode 15+

### React Native
- React Native 0.81.5+
- React 19.1.0+
- Node.js 20+

### Flutter
- Flutter 1.17.0+
- Dart SDK 2.8+

---

## Documentation

- [Android Integration](docs/android.md)
- [iOS Integration](docs/ios.md)
- [React Native Integration](docs/react-native.md)
- [Flutter Integration](docs/flutter.md)

---

## Changelog
| Version | Date | Notes |
|---|---|---|
| 1.0.0 | Apr 2026 | Initial release |

---

## Support
- 📧 Email: support@saversapp.com
- 🐛 Issues: raise via your partner contact
