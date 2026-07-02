---
id: 5e5d1180-62e2-4db5-8230-eb7850026b2d
blueprint: captains_mate_app_43
title: 'Home 4.3'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780584226
---
# TCA Mobile App — Overview 

## App Overview

The TCA Mobile App (**CAptain's Mate**) gives Cruising Association members access to the CA's database of cruising location data and allows them to share information, experiences, and opinions on destinations. It also includes facilities to find other CA members nearby (opt-in), identify member discounts, and look up the CA's Honorary Local Representatives worldwide. Requires a CA membership and CA website account.

### Key Features

- **Authentication** — OAuth 2.0 / OIDC login via the CA's SSO service
- **Maps** — Interactive map with marina/harbour markers which link to further details; map marker clustering; offline tile caching.
- **Locations** - A searchable, filterable list of 1000s of marinas and harbours worldwide with links to detailed information pages, plus the ability for CA members to add and view reports.
- **Position tracking & Geofencing** — GPS position tracking for the map; optional position sharing with CA friends via the API (works in background/headless mode); Automatic sorting of the location list by those closest, based on GPS position
- **Member & Boat Directory** — Browse the CA member and boat directory
- **Friends & Nearby Members** — Find other CA members nearby and share your position with friends (opt-in)
- **Notifications** — Firebase Cloud Messaging (FCM) push notifications, with or without geoareas attached to target specific regions; optional geofence-based arrival notifications for locations
- **Offline Access** — Core data is cached on-device so the app remains usable without an internet connection
- **Membership & HLRs** — CA membership details and CA Honorary Local Representitive data

### Platform Identifiers

| Platform | Bundle ID |
|----------|-----------|
| Android | `com.cruisingassociation.captainsmate` |
| iOS | `uk.org.cruising.captainsmate` |

---

## Flutter & Dart Versions

| Tool | Version |
|------|---------|
| Flutter | **3.35.7** (pinned via FVM — see `.fvmrc`) |
| Dart SDK | **>=3.6.2 <4.0.0** |
| App version | **4.3.4+285** |

Flutter version management is handled by [FVM](https://fvm.app/). Always use `fvm flutter` rather than `flutter` directly to ensure the pinned version is used.


## Dependencies

### Fonts

- **Mulish** — Regular, SemiBold (600), Bold (700) — bundled under `assets/fonts/`

---

### External Dependencies & Services

| Service | Purpose |
|---------|---------|
| **CA SSO** (`sso.theca.org.uk`) | OAuth 2.0 / OIDC authentication |
| **CA API** (`api.theca.org.uk`) | Primary backend API |
| **Staging API** (`apicm.myca.org.uk`) | Staging/test environment |
| **Firebase** (project: `captains-mate-push`) | Push notifications (FCM) |
| **Sentry** | Crash reporting and error monitoring |
| **Transistor Software** | `flutter_background_geolocation` SDK (requires a licence key per environment) |

---

### Flutter Packages

For a full list of Flutter packages, external dependencies, package caveats, unsupported packages, and out-of-date packages see [dependencies](/v4.3/overview/dependencies-flutter-packages).

---

## Test Coverage
Since version 4.3.3, some unit tests now exist for notifications, sync services and related models, providers, API clients, and helpers. Tests `getLocation - get raw JSON` and `getNotifications - get raw JSON` get data directly from the CA API and need environment variables configured. Copy the ```test_env.dart.example``` file, rename to ```test_env.dart```, and enter the correct values.


## Release Process

Previously the app's developers used CI/CD (the relevant files are still present in the codebase). Currently the app is being built and released manually, which allows for direct oversight of the build process. The signing of the iOS app is automatically managed by XCode. Both apps are built locally using the same flags as the previous developers used, although the ```dart-define``` vars have now been moved to a separate file within the app (not committed to source control, see above).

Android:

```

flutter build appbundle \
  --release \
  --shrink \
  --flavor production \
  --dart-define-from-file=dart-defines.prod.json
```
  
  The bundle file is uploaded to Google Play Developer console manually.
  
 iOS:
  
  ```
  fvm flutter build ipa \
  --release \
  --flavor production \
  --dart-define-from-file=dart-defines.prod.json 
  ```
  
The archive file resulting from the build, is verified and uploaded to App Store Connect through XCode. 


## Caveats

### Code Generation is Required
Generated files (`*.g.dart`, `*.freezed.dart`, Floor DAOs) are **not committed to git**. The project will not build without running `build_runner` first. Keep `build_runner watch` running during active development.

### Git-Referenced Dependencies
`floor` and `floor_generator` are pinned to a **private fork** (`jwyrembelski/floor`) on a specific branch (`update-3.7.0-3.29.0`). If this fork is removed, renamed, or the branch is deleted, `pub get` will fail. 

### Firebase on iOS Simulators
Push notifications sent via Firebase **do not work on iOS simulators**. Use an Android emulator or a physical iOS device for notification testing.

### Unit/Integration Test Coverage
Unit test coverage has only just been implemented for some sync and notifications work, but is not comprehensive. A large part of the codebase also remains untestable without significant refactoring. No integration tests have been implemented.

### Background Location Licence
`flutter_background_geolocation` (Transistor Software) requires a valid licence key per app identifier / environment. The `BACKGROUND_LOCATION_API_KEY` dart-define must be set; it's not needed for local development but a real key is required for production builds.

### `one_context`
`one_context` provides global `BuildContext` access outside the widget tree. This is a known anti-pattern; use with care and avoid expanding its usage.

### App Architecture
From a fairly flat file structure, the previous developers attempted to restructure the app in a domain-driven design, however this wasn't completed. Screens are still in a separate folder, some widgets are in the root `widgets` folder and some are in individual `features` folders. There are a number of files in the app, such as the Settings Screen which are many lines and structurally flat. Widgets have not been split out into separate files, which is best practice. As a result they are difficult to read and maintain.

### Flutter Packages Unsupported and Out-of-Date
For package-specific caveats, unsupported packages, out-of-date packages, and dependency override status see [dependencies](/v4.3/overview/dependencies-flutter-packages).