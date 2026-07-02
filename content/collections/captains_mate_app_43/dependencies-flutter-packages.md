---
id: 5c7c77d7-9e96-414f-b2df-054d49356a79
blueprint: captains_mate_app_43
title: Dependencies
use_synced_content: false
parent: a1b7cec3-99c2-4a5d-9c8b-9790b8204982
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780583800
---
# Flutter Packages & Dependencies

---

## Package Management

Flutter SDK versions are managed using FVM (Flutter Version Management), ensuring that development and release builds are performed using a consistent Flutter environment. Project dependencies are controlled through a committed `pubspec.lock` file, which locks all package versions to those that have been tested and approved.

Package updates are not applied automatically. Upgrades should be performed intentionally as part of planned maintenance activities, allowing sufficient testing to take place before changes are released to production. VSCode will sometimes prompt to upgrade packages but this should be ignored. For a planned upgrade to the installed packages use `fvm flutter pub upgrade`.

---

## Flutter Packages

### State Management & Architecture

| Package | Version | Purpose |
|---------|---------|---------|
| `flutter_riverpod` | ^2.4.9 | Reactive state management |
| `riverpod_annotation` | ^2.3.3 | Code-gen annotations for Riverpod |
| `rxdart` | ^0.28.0 | Reactive extensions / stream operators |
| `kiwi` | ^5.0.1 | Dependency injection container |

### Data & Persistence

| Package | Version | Purpose |
|---------|---------|---------|
| `hive_ce` | ^2.19.1 | Primary local database — stores all app data (locations, boats, positions, discounts, HLRs, etc.) |
| `hive_ce_flutter` | ^2.3.4 | Flutter adapter for Hive CE |
| `floor` | git (fork) | SQLite ORM used **only** for the in-app debug/error log (`log_models` table) — chosen here because the log viewer uses time-bounded SQL queries (`WHERE created_at > :value`) that are impractical with Hive's key-value model |
| `streaming_shared_preferences` | ^2.0.0 | Reactive shared preferences |
| `flutter_secure_storage` | ^9.2.4 | Encrypted secure key-value storage |
| `path_provider` | ^2.1.0 | Platform-specific file paths |

### Networking

| Package | Version | Purpose |
|---------|---------|---------|
| `dio` | ^5.4.0 | HTTP client |
| `pretty_dio_logger` | ^1.4.0 | Dio request/response logging |

### Authentication

| Package | Version | Purpose |
|---------|---------|---------|
| `flutter_appauth` | ^9.0.0 | OAuth 2.0 / OIDC authentication |

### Maps & Location

| Package | Version | Purpose |
|---------|---------|---------|
| `flutter_map` | ^8.1.1 | OpenStreetMap-based interactive maps |
| `flutter_map_marker_cluster` | ^8.2.2 | Map marker clustering |
| `flutter_map_tile_caching` | ^10.1.1 | Offline map tile download and caching |
| `flutter_background_geolocation` | ^4.18.3 | GPS tracking for map position; powers friends position sharing (headless/background mode) and geofence arrival notifications |
| `geolocator` | ^13.0.2 | Foreground device geolocation |
| `latlong2` | ^0.9.0 | Latitude/longitude data types |
| `latlong_to_osgrid` | ^1.3.2 | Coordinate conversion to OS Grid |

### Code Generation & Serialisation

| Package | Version | Purpose |
|---------|---------|---------|
| `freezed` | ^3.0.6 | Immutable data classes & union types |
| `freezed_annotation` | ^3.0.0 | Freezed annotations |
| `json_annotation` | ^4.9.0 | JSON serialisation annotations |

### UI & Media

| Package | Version | Purpose |
|---------|---------|---------|
| `flutter_svg` | ^2.0.10+1 | SVG asset rendering |
| `cached_network_image` | ^3.3.1 | Network image caching |
| `photo_view` | ^0.14.0 | Zoomable photo viewer |
| `image_picker` | ^1.1.2 | Camera / gallery image selection |
| `flutter_html` | 3.0.0-beta.2 | Render HTML content in Flutter |
| `html_editor_enhanced` | ^2.7.0 | Rich-text HTML editor (WebView-based); requires bundled assets at `assets/html_editor/` |
| `flutter_inappwebview` | ^6.1.5 | Embedded in-app web browser |
| `sliding_up_panel` | ^2.0.0+1 | Sliding bottom sheet panel |
| `dotted_border` | ^2.0.0+2 | Dotted/dashed border decoration |
| `cupertino_icons` | ^1.0.5 | iOS-style icon set |

### Notifications & Background

| Package | Version | Purpose |
|---------|---------|---------|
| `firebase_core` | ^3.9.0 | Firebase SDK core |
| `firebase_messaging` | ^15.1.6 | Firebase Cloud Messaging (push) |
| `flutter_local_notifications` | ^19.3.0 | Local on-device notifications |
| `workmanager` | ^0.9.0 | Background task scheduling |

### Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `intl` | ^0.17.0 | Internationalisation & date formatting |
| `timeago` | ^3.3.0 | Human-readable relative timestamps |
| `uuid` | ^4.5.1 | UUID generation |
| `crypto` | ^3.0.2 | Cryptographic hashing |
| `mime` | ^1.0.2 | MIME type detection |
| `filesize` | ^2.0.1 | Human-readable file sizes |
| `diacritic` | ^0.1.3 | Diacritic / accent stripping |
| `intersperse` | ^2.0.0 | List interspersing utility |
| `form_field_validator` | ^1.1.0 | Form validation helpers |
| `one_context` | ^4.0.0 | Global `BuildContext` access |
| `package_info_plus` | ^8.0.0 | App version / package info |
| `device_info_plus` | ^10.1.2 | Device model and OS info |
| `url_launcher` | ^6.3.1 | Open URLs in browser / external apps |
| `url_launcher_ios` | ^6.3.2 | iOS-specific url_launcher implementation |
| `permission_handler` | ^12.0.0 | Runtime permission requests |
| `sentry_flutter` | 9.0.0 | Crash and error reporting |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `build_runner` | ^2.4.8 | Code generation runner |
| `floor_generator` | git (fork) | Floor entity/DAO code generation |
| `json_serializable` | ^6.5.3 | JSON serialisation code generation |
| `hive_ce_generator` | ^1.9.1 | Hive adapter code generation |
| `riverpod_generator` | ^2.3.9 | Riverpod provider code generation |
| `custom_lint` | ^0.7.5 | Custom lint rules runner |
| `riverpod_lint` | ^2.3.7 | Riverpod-specific lint rules |
| `mockito` | ^5.4.6 | Mock generation for unit tests |

### Fonts

- **Mulish** — Regular, SemiBold (600), Bold (700) — bundled under `assets/fonts/`

---

## External Dependencies & Services

| Service | Purpose |
|---------|---------|
| **CA SSO** (`sso.theca.org.uk`) | OAuth 2.0 / OIDC authentication |
| **CA API** (`api.theca.org.uk`) | Primary backend API |
| **Staging API** (`apicm.myca.org.uk`) | Staging/test environment |
| **Firebase** (project: `captains-mate-push`) | Push notifications (FCM) |
| **Sentry** | Crash reporting and error monitoring |
| **Transistor Software** | `flutter_background_geolocation` SDK (requires a licence key per environment) |

---

## Package Caveats

### `html_editor_enhanced`
Pinned to `^2.7.0`, which is unmaintained upstream. The package uses a WebView internally and may behave inconsistently across platform/OS versions. It requires bundled local HTML/JS assets at `assets/html_editor/` — if this directory is removed or excluded from the build, the editor will fail silently at runtime.

### `flutter_html`
Pinned to `3.0.0-beta.2` (a beta release). It has known rendering limitations with complex HTML and may produce unexpected layout results. There is a stable version now available.

### Dependency Overrides

A significant number of transitive dependencies are version-overridden in `pubspec.yaml` to address Android 16KB page size support and Flutter 3.32+ v1 embedding removal. There are also overrides where the reason is not clear — these should be reviewed and removed if possible.

#### Flutter 3.32+ v1 embedding removal fixes
These were pinned manually because the parent plugins hadn't yet published versions compatible with Flutter 3.32+'s removal of the v1 Android embedding. The upstream packages have since released multiple updates — it is worth testing whether these overrides are still needed.

| Package | Pinned to | Latest available |
|---------|-----------|-----------------|
| `flutter_plugin_android_lifecycle` | 2.0.31 | 2.0.35 |
| `shared_preferences_android` | 2.4.13 | 2.4.25 |
| `path_provider_android` | 2.2.19 | 2.3.1 |
| `url_launcher_android` | 6.3.20 | 6.3.32 |
| `image_picker_android` | 0.8.13+1 | 0.8.13+19 |
| `geolocator_android` | 5.0.1+1 | 5.0.2 |
| `sqflite_android` | 2.4.2 | 2.4.3 |

#### ObjectBox 16KB page size override
`flutter_map_tile_caching` pins ObjectBox to 2.x (4KB-aligned binaries). The override forces ObjectBox 4.x for 16KB page size compliance. ObjectBox has since released 5.x — the override constraint (`^4.0.0`) will not resolve to 5.x automatically, so this will need revisiting when `flutter_map_tile_caching` is upgraded.

#### Unexplained overrides
The following overrides exist in `pubspec.yaml` without a documented reason. They are likely transitive version conflict resolutions but should be reviewed to confirm they are still needed:

| Package | Pinned to |
|---------|-----------|
| `collection` | 1.19.0 |
| `sqflite` | 2.4.1 |
| `sqflite_common` | 2.5.4+6 |
| `sqflite_common_ffi` | 2.3.4+4 |
| `sqflite_common_ffi_web` | 0.4.5+4 |
| `unicode` | 1.1.8 |
| `path` | 1.9.1 |
| `test_api` | 0.7.3 |

---

## Unsupported Packages

These packages have had no releases for several years and show no signs of active maintenance. They currently function but carry a risk of breaking with future Flutter or OS updates.

| Package | Version in use | Last published | Notes |
|---------|---------------|----------------|-------|
| `sliding_up_panel` | 2.0.0+1 | ~5 years ago | Powers the sliding location panel on the map screen. Maintainer has publicly acknowledged difficulty keeping up. Already causing issues with Android 15+ layout changes — workarounds have had to be applied |
| `streaming_shared_preferences` | 2.0.0 | ~5 years ago | Used for locally stored user settings and preferences. No updates since 2.x launch; wraps `shared_preferences` which has since moved on significantly |
| `form_field_validator` | ^1.1.0 | ~5 years ago | Basic form validation helpers; no updates since initial release |
| `filesize` | ^2.0.1 | ~5 years ago | Human-readable file sizes; narrow use, low risk |
| `kiwi` | ^5.0.1 | ~22 months ago | Dependency injection container; no updates in nearly two years |
| `latlong_to_osgrid` | ^1.3.2 | ~2 years ago | OS Grid coordinate conversion; narrow use case, low update risk |

---

## Out of Date Packages

These packages have newer versions available. Entries are grouped where packages must be upgraded together. Minor patch gaps within the `^` constraint range are omitted.

### Riverpod ecosystem — must be upgraded together
| Package | Version in use | Latest stable |
|---------|---------------|---------------|
| `flutter_riverpod` | ^2.4.9 | 3.3.1 |
| `riverpod_annotation` | ^2.3.3 | 4.0.2 |
| `riverpod_generator` | ^2.3.9 | — |
| `riverpod_lint` | ^2.3.7 | — |

### Firebase ecosystem — must be upgraded together
| Package | Version in use | Latest stable |
|---------|---------------|---------------|
| `firebase_core` | ^3.9.0 | 4.10.0 |
| `firebase_messaging` | ^15.1.6 | 16.3.0 |

### Other packages with significant version gaps
| Package | Version in use | Latest stable | Notes |
|---------|---------------|---------------|-------|
| `flutter_background_geolocation` | ^4.18.3 | 5.2.0 | **Breaking** — v4 licence keys do not work with v5; new keys must be generated via the Transistor Customer Dashboard before upgrading |
| `flutter_appauth` | ^9.0.0 | 12.0.1 | Three major versions behind |
| `flutter_local_notifications` | ^19.3.0 | 21.0.0 | Two major versions behind |
| `flutter_secure_storage` | ^9.2.4 | 10.3.1 | v10 changed default encryption to RSA OAEP + AES-GCM; includes automatic migration for existing stored values |
| `package_info_plus` | ^8.0.0 | 10.1.0 | Two major versions behind |
| `device_info_plus` | ^10.1.2 | 13.1.0 | Three major versions behind; latest requires Flutter >=3.38.1 (current pin is 3.35.7) |
| `geolocator` | ^13.0.2 | 14.0.2 | One major version behind |
| `sentry_flutter` | 9.0.0 | 9.21.0 | 21 minor/patch versions behind; upgrade recommended |
| `dotted_border` | ^2.0.0+2 | 3.1.0 | One major version behind |
| `intl` | ^0.17.0 | 0.20.2 | Constraint cannot reach 0.20.x; must be bumped manually |
| `flutter_html` | 3.0.0-beta.2 | 3.0.0 | Using a beta release when stable is available |
| `dio` | ^5.4.0 | 5.9.2 | Several minor versions behind |
| `photo_view` | ^0.14.0 | 0.15.0 | One minor version behind; last published ~2 years ago |
| `timeago` | ^3.3.0 | 3.7.1 | Several minor versions behind |
| `mime` | ^1.0.2 | 2.0.0 | Major version behind |