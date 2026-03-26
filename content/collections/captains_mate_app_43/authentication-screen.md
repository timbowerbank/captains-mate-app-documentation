---
id: 512e14a9-9276-49d1-aa2c-a6b16e2fd813
blueprint: captains_mate_app_43
title: 'Authentication Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774533593
---
# AuthenticationScreen

**File:** `lib/src/screens/authentication_screen.dart`
**Route:** `/authentication` (the app's `initialRoute`)

---

## Purpose

The app's entry point and splash screen. It orchestrates the entire startup sequence: initialising core services and providers, evaluating authentication state, performing token refresh if needed, checking connectivity, running the full data sync, and ultimately routing the user to the home screen. If the user is logged out or their refresh token has expired, it presents a LOGIN button. If the device is offline, and the user is logged in, it offers an "Open in Offline Mode" option. Other wise it redirects to the device's browser and show's the brower's 'no connection' screen.

This screen is also re-navigated to from `HomeScreen` (when a sync is due) and from `SettingsScreen` (on account switch or logout).

---

## UI Simplified Structure

```
PopScope (canPop: false - prevents back-navigation)
 └── Scaffold
      ├── AppBar (empty - only sets dark status bar icons)
      │
      └── body: Stack
           │
           ├── Padding > Column (spaceEvenly)
           │    │
           │    ├── Image.asset (app logo)
           │    │
           │    ├── AnimatedOpacity (visible when _isLoading)
           │    │    └── OfflineDataSync widget
           │    │         ├── CircularProgressIndicator
           │    │         ├── Text (sync step description / override text)
           │    │         └── TcaButton ("Sync in background")
           │    │              ← only shown once sync is past initial steps
           │    │
           │    └── AnimatedOpacity (visible when NOT _isLoading)
           │         └── TcaButton ("LOGIN")
           │
           └── SafeArea > Align (bottomCenter)
                └── TcaVersionNumber (inline)
```

The two `AnimatedOpacity` blocks crossfade between the sync progress indicator and the login button depending on `_isLoading` state.

---

## Associated

### Providers (initialised/read in this screen)

| Provider | File | Usage |
|---|---|---|
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Core auth state (Riverpod Notifier, keepAlive). Called for `initialize()`, `login()`, `refreshToken()`, and reading `accessToken`/`refreshTokenExpiration`/`currentUser` |
| `syncNotifierProvider` | `lib/src/features/sync/providers/sync_provider.dart` | Manages sync lifecycle (Riverpod Notifier, keepAlive). Called for `checkForConnection()` and `runSync()`. Its `isBackgroundSync` flag is toggled when user opts to sync in background |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Initialised first via `.initialize()` to open all Hive boxes |
| `sharedPreferencesProvider` | `lib/src/data/providers/shared_preferences.dart` | Initialised via `.init()`. Used for reading log level settings |
| `friendSettingProvider` | `lib/src/data/providers/friend_setting.dart` | Initialised via `.init()` - loads position sharing / friend name settings from SharedPreferences |
| `notificationProvider` | `lib/src/features/notifications/providers/notification_list_provider.dart` | Initialised via `.initialize()`. Later queried for unread notifications before navigating to home |
| `hasSeenNotificationModal` | `lib/src/features/notifications/providers/notification_list_provider.dart` | `StateProvider<bool>` - checked to decide whether to query unread notifications |
| `maptileProvider` | `lib/src/data/providers/maptile.dart` | Initialised via `.init()`. Also used to resume downloading any saved map sections |
| `savedMapSectionProvider` | `lib/src/data/providers/saved_map_section.dart` | `FutureProvider` that loads saved offline map sections from Hive. Sections marked as `downloading` are resumed |

### Models / Route Arguments

| Model | File | Role |
|---|---|---|
| `AuthenticationScreenArguments` | (defined in-file, line 31) | `checkAuthStatus` (skip auth checks, used on logout) and `skipFirst` (skip first-run storage wipe, used on account switch) |
| `HomeScreenRoute` | `lib/src/helpers/route_argumnets/home_screen_route.dart` | Carries `showUnreadNotificationModal` flag to the home screen |
| `SyncState` | `lib/src/features/sync/providers/sync_state.dart` | Freezed union: `idle`, `loading(step)`, `success`, `cancelled`, `error(message, stack)` |

### Widgets

| Widget | File | Role |
|---|---|---|
| `OfflineDataSync` | `lib/src/features/sync/widgets/offline_data_sync.dart` | Watches `syncNotifierProvider` and displays sync progress text + spinner. Provides `onComplete`, `onError`, and `onBackgroundSync` callbacks. Shows a "Sync in background" button once sync is past initial steps |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Styled button used for the LOGIN action |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Used for the offline mode modal and missing-scope warning modal |
| `TcaVersionNumber` | `lib/src/widgets/tca_version_number.dart` | Displays app version and build number at the bottom of the screen |

### Helpers / Services (initialised here)

| Service | File | Role |
|---|---|---|
| `PositionHelper` | `lib/src/helpers/position.dart` | Static `.init()` - calls `BackgroundGeolocation.ready()` which must precede all geolocation operations |
| `LocationGeofenceService` | `lib/src/features/locations/services/location_geofence_service.dart` | `.initialise()` - sets up geofencing after position helper is ready |
| `PositionBroadcastHelper` | `lib/src/helpers/position_broadcast.dart` | `.init()` - starts position broadcasting for friends feature |
| `TcaLog` | `lib/src/helpers/log.dart` | `.init()` - configures logging with current user and log level |

### Exceptions

| Exception | File | Triggers |
|---|---|---|
| `InvalidTokenException` | `lib/src/data/exceptions/invalid_token_exception.dart` | Thrown during token refresh if token is revoked. Causes fallback to login button |
| `MissingScopeException` | `lib/src/data/exceptions/MissingScopeException.dart` | Thrown during login if OAuth scopes don't match. Shows warning dialog then continues |
| `SkipSyncException` | `lib/src/data/exceptions/SkipSyncException.dart` | Thrown by sync when no sync is needed. Treated as success and proceeds to home |

---

## API / Backend Dependencies

Direct API calls from this screen are minimal - most are delegated through providers:

| Action | Via | Endpoint | Description |
|---|---|---|---|
| Connectivity check | `syncNotifierProvider.checkForConnection()` → `ConnectivityApiClient` | `GET /ping` (or similar) | Quick ping to CA website (hardcoded in config file) to verify server reachability |
| OAuth login | `authenticationProvider.login()` → `FlutterAppAuth` | OAuth2 authorize + exchange code | Opens system browser for OAuth flow against configured auth/token endpoints |
| Token refresh | `authenticationProvider.refreshToken()` → `FlutterAppAuth` | OAuth2 token refresh | Silent token refresh using stored refresh token |
| Full data sync | `syncNotifierProvider.runSync()` → `SyncDataService.download()` | Multiple endpoints sequentially | Downloads meta types, attributes, overview sections, locations, localities, overviews, location attributes, reports, discounts, HLRs, boats, members - then saves all to Hive |
| Map tile download | `maptileProvider.downloadSection()` | Tile server | Resumes any in-progress offline map section downloads |

The sync service (`SyncDataService.download`) calls the following API clients in sequence:
1. `ApiClient.metaTypes.getMetaLocationTypes()`
2. `ApiClient.metaAttributes.getMetaAttributes()`
3. `ApiClient.overviews.getOverviewSections()`
4. `ApiClient.location.getLocations(tfrom:)`
5. `ApiClient.localities.getLocalities(tfrom:)`
6. `ApiClient.overviews.getLocationOverviews(tfrom:)`
7. `ApiClient.locationAttribute.getLocationAttributes(tfrom:)`
8. `ApiClient.reports.getReports(tfrom:)`
9. `ApiClient.discounts.getDiscounts(tfrom:)`
10. `ApiClient.hlrs.getHlrs(tfrom:)`
11. `ApiClient.boat.getBoats(tfrom:)`
12. `ApiClient.members.getMembers(tfrom:)`

---

## Data Flow

### Startup Sequence (Happy Path - Returning User, Online)

```
App launches → initialRoute: /authentication
  │
  ├─ initState: _isLoading = true
  │
  └─ _loadProviders() (post-frame callback)
       │
       ├─ 1. Initialize core services (sequential)
            ├─ hiveDatabaseProvider.initDatabase()   ← initialises IsolatedHive and registers HiveAdapters
       │    ├─ repositoryManagerProvider.initialize()   ← opens Hive boxes for each repository
       │    ├─ sharedPreferencesProvider.init()
       │    ├─ friendSettingProvider.init()
       │    ├─ notificationProvider.initialize()
       │    ├─ maptileProvider.init()
       │    ├─ authenticationProvider.initialize()      ← loads tokens from secure storage
       │    ├─ PositionHelper.init()                    ← BackgroundGeolocation.ready()
       │    ├─ LocationGeofenceService.initialise()
       │    ├─ PositionBroadcastHelper.init()
       │    └─ TcaLog.init()
       │
       ├─ 2. Artificial delay (1400ms) to prevent flicker
       │
       ├─ 3. Check refresh token expiry
       │    └─ refreshToken valid → continue
       │
       ├─ 4. Connectivity check (syncNotifier.checkForConnection)
       │    └─ online → continue
       │
       ├─ 5. Check access token validity
       │    ├─ valid → _loadLocations() directly
       │    └─ expired → refreshToken() then _loadLocations()
       │
       ├─ 6. _loadLocations()
       │    ├─ Resume any downloading map sections
       │    └─ syncNotifier.runSync() → SyncDataService.download()
       │         └─ OfflineDataSync widget shows progress
       │
       └─ 7. handleHomeScreenRoute()
            ├─ Check for unread notifications
            └─ pushReplacementNamed → HomeScreen
```

### Startup - Offline User

```
_loadProviders()
  │
  ├─ Steps 1-3 same as above
  │
  ├─ 4. Connectivity check fails
  │    ├─ If check took >1 second (slow timeout):
  │    │    ├─ setNextOnlineDue() → shows offline countdown text
  │    │    └─ handleHomeScreenRoute() → HomeScreen (offline mode)
  │    │
  │    └─ If check was fast (<1 second, immediate failure):
  │         └─ _showOfflineModal()
  │              └─ "Open in Offline Mode" button
  │                   ├─ setNextOnlineDue()
  │                   └─ handleHomeScreenRoute() → HomeScreen
  │
  └─ If no refresh token → show LOGIN button → Clicking opens browser with SSO url but 'no connection' error page.
```

### Startup - Logged Out / Expired Token

```
_loadProviders()
  │
  ├─ Steps 1-2 same as above
  │
  ├─ 3. refreshToken expired or checkAuthStatus == false
  │    └─ setState(_isLoading = false)
  │         └─ UI crossfades to LOGIN button
  │
  └─ User taps LOGIN
       ├─ _handleLogin()
       │    ├─ authenticationProvider.login() → OAuth browser flow
       │    ├─ On MissingScopeException → _showScopeErrorModal() (warning, continues)
       │    └─ _loadLocations(force: true) → full sync
       │
       └─ handleHomeScreenRoute() → HomeScreen
```

### Background Sync Option

```
During sync, OfflineDataSync shows "Sync in background" button
  │
  └─ User taps "Sync in background"
       ├─ syncNotifier.isBackgroundSync = true
       └─ handleHomeScreenRoute() → HomeScreen
            └─ Sync continues in background via syncNotifierProvider
                 └─ On error → SyncFailedModal shown via OneContext
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| App launch | `lib/src/app.dart` | None (defaults) | `initialRoute` |
| HomeScreen sync check | `lib/src/screens/home_screen.dart` | None | `pushReplacementNamed` when a periodic sync is due |
| Settings - Switch Account | `lib/src/screens/settings/settings_screen.dart` | `checkAuthStatus: false, skipFirst: true` | `pushNamedAndRemoveUntil` after clearing storage |
| Settings - Logout | `lib/src/screens/settings/settings_screen.dart` | `checkAuthStatus: false` | `pushNamedAndRemoveUntil` after `authProvider.logout()` |

---

## Known Caveats / Tech-Debt Notes

1. **Hardcoded 1400ms delays** (lines 121, 148): Two `Future.delayed(Duration(milliseconds: 1400))` calls are used to prevent UI flicker. The first delays before checking auth status; the second adds delay before entering offline mode. These create a perceptible wait on every app launch regardless of how fast initialisation actually is.

2. **`accessTokenExpiration` force-unwrap** (line 162): `ref.read(authenticationProvider).accessTokenExpiration!` is force-unwrapped. If the auth state has a valid refresh token but a null access token expiration (edge case), this will crash.

3. **`_loadProviders` error handling relies on `mounted` but is not comprehensive**: The method is `async` and runs through many sequential awaits. If the widget is disposed mid-sequence (e.g. hot reload), any `setState` call will throw. The `handleHomeScreenRoute` method checks `mounted`, but earlier steps like `setState(() => _isLoading = false)` at line 136 do not.

4. **Connectivity heuristic based on timing** (lines 148-157): The offline detection logic uses `stopwatch.elapsed.inSeconds > 1` to distinguish between "device is offline" (fast failure) and "server is slow/unreachable" (slow timeout). This is fragile - a 1.1-second DNS timeout could silently skip the offline modal.

5. **`setNextOnlineDue` duration calculation is inverted** (line 151): The call at line 151 passes `DateTime.now().difference(refreshTime)` which produces a *negative* duration (now minus a future date). The `setNextOnlineDue` method then reads `difference.inDays` and `difference.inHours` which will be negative, producing messages like "due within -5 days (I've checked this but it is reflecting the positive duration)". The modal path at line 326 correctly uses `refreshTokenExpiration.difference(DateTime.now())` (positive).

6. **OfflineDataSync side-effects in `build`** (widget lines 26-30): The `OfflineDataSync` widget calls `onError()` and `onComplete()` directly inside its `build` method based on `syncState`. This means navigation side-effects fire during widget builds, which is an anti-pattern and can cause "setState during build" errors.

7. **Sequential provider initialisation**: All providers are initialised sequentially in `_loadProviders`. Independent initialisations (e.g. `friendSettingProvider.init()`, `notificationProvider.initialize()`, `maptileProvider.init()`) could be parallelised with `Future.wait` to reduce startup time.

8. **Map section downloads fire-and-forget**: Downloaded map sections (line 64) use `.onError` to capture to Sentry but the futures are not awaited or tracked. If downloads fail silently, the user has no indication.

9. **No retry mechanism**: If the connectivity check or token refresh fails, the only recovery is the offline modal or showing the login button. There is no "Retry" option to re-attempt the connection without restarting the app.

10. **`_isDeleting` / `_isLoading` pattern**: The `_isLoading` boolean drives the crossfade between sync indicator and login button but is never reset to `true` after the login button is shown and tapped if `_handleLogin` throws - the catch block at line 249 sets `_isLoading = false` which is already false, meaning a second tap works correctly, but there's a brief re-render.