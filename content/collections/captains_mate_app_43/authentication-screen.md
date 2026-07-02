---
id: 512e14a9-9276-49d1-aa2c-a6b16e2fd813
blueprint: captains_mate_app_43
title: 'Authentication Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780399934
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# AuthenticationScreen

**File:** `lib/src/screens/authentication_screen.dart`
**Route:** `/authentication` (the app's `initialRoute`)

---

## Purpose

The app's entry point and splash screen. It orchestrates the entire startup sequence: delegating core service initialisation to `AppInitialisationService`, evaluating authentication state, performing token refresh if needed, checking connectivity, running the data sync, and ultimately routing the user to the home screen. If the user is logged out or their refresh token has expired, it presents a LOGIN button. If the device is offline and the user is logged in, it offers an "Open in Offline Mode" option. Otherwise it redirects to the device's browser and shows the browser's 'no connection' screen.

This screen is also re-navigated to from `HomeScreen` (when a sync is due) and from `SettingsScreen` (on account switch or logout).

---

## UI Simplified Structure

```
PopScope (canPop: false - prevents back-navigation)
 └── TcaSafeScaffold
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
           │    │         └── Text (sync step description / override text)
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
| `appInitialisationProvider` | `lib/src/data/providers/app_initialisation_provider.dart` | Orchestrates all core service initialisation. Called once via `.initialize(skipFirst:)` |
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Core auth state (Riverpod Notifier, keepAlive). Called for `login()`, `refreshToken()`, and reading `accessToken` / `accessTokenExpiration` / `refreshTokenExpiration` |
| `syncNotifierProvider` | `lib/src/features/sync/providers/sync_provider.dart` | Manages sync lifecycle (Riverpod Notifier, keepAlive). Called for `checkForConnection()` and `runSync()` |
| `sharedPreferencesProvider` | `lib/src/data/providers/shared_preferences.dart` | Read for `lastSuccessfulSync` to distinguish first run from subsequent runs |
| `maptileProvider` | `lib/src/data/providers/maptile.dart` | Used to resume downloading any in-progress saved map sections |
| `savedMapSectionProvider` | `lib/src/data/providers/saved_map_section.dart` | `FutureProvider` that loads saved offline map sections from Hive. Sections marked as `downloading` are resumed |

### Models / Route Arguments

| Model | File | Role |
|---|---|---|
| `AuthenticationScreenArguments` | (defined in-file) | `checkAuthStatus` (skip auth checks, used on logout) and `skipFirst` (skip first-run storage wipe, used on account switch) |
| `SyncState` | `lib/src/features/sync/providers/sync_state.dart` | Freezed union: `idle`, `loading(step)`, `success`, `cancelled`, `error(message, stack)` |

### Widgets

| Widget | File | Role |
|---|---|---|
| `OfflineDataSync` | `lib/src/features/sync/widgets/offline_data_sync.dart` | Watches `syncNotifierProvider` and displays sync progress text and spinner. Provides `onComplete` and `onBackgroundSync` callbacks. Handles sync errors inline — the "Continue to app" button triggers `onBackgroundSync`, which calls `continueInBackground()` then navigates home |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Styled button used for the LOGIN action |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Used for the offline mode modal and missing-scope warning modal |
| `TcaVersionNumber` | `lib/src/widgets/tca_version_number.dart` | Displays app version and build number at the bottom of the screen |

### Services (initialised via AppInitialisationService)

All core service initialisation is delegated to `AppInitialisationService` (`lib/src/data/providers/app_initialisation_provider.dart`). The following are initialised in order:

| Service | File | Role |
|---|---|---|
| `hiveDatabaseProvider` | `lib/core/providers/hive_database_provider.dart` | Initialises IsolatedHive and registers Hive adapters |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Opens all Hive boxes |
| `sharedPreferencesProvider` | `lib/src/data/providers/shared_preferences.dart` | Loads persisted user preferences |
| `friendSettingProvider` | `lib/src/data/providers/friend_setting.dart` | Loads position sharing / friend name settings |
| `notificationProvider` | `lib/src/features/notifications/providers/notification_list_provider.dart` | Initialises notification repository |
| `FMTCObjectBoxBackend` | (flutter_map_tile_caching) | Initialises the map tile cache backend |
| `maptileProvider` | `lib/src/data/providers/maptile.dart` | Registers map tile caches |
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Loads tokens from secure storage |
| `FcmService.checkPermissions()` | `lib/src/features/notifications/services/fcm_service.dart` | Requests notification permission from OS |
| `PositionHelper.checkAndRequestPermissions()` | `lib/src/helpers/position.dart` | Requests location permissions from OS |
| `PositionHelper.init()` | `lib/src/helpers/position.dart` | Calls `BackgroundGeolocation.ready()` — must precede all geolocation operations |
| `LocationGeofenceService.initialise()` | `lib/src/features/locations/services/location_geofence_service.dart` | Sets up geofencing |
| `PositionBroadcastHelper.init()` | `lib/src/helpers/position_broadcast.dart` | Starts position broadcasting for friends feature |
| `TcaLog.init()` | `lib/src/helpers/log.dart` | Configures logging with current user and log level |
| `FcmService.initialiseFirebaseMessaging()` | `lib/src/features/notifications/services/fcm_service.dart` | Registers FCM listeners |

### Exceptions

| Exception | File | Triggers |
|---|---|---|
| `InvalidTokenException` | `lib/src/data/exceptions/invalid_token_exception.dart` | Thrown during token refresh if token is revoked. Causes fallback to login button |
| `MissingScopeException` | `lib/src/data/exceptions/MissingScopeException.dart` | Thrown during login if OAuth scopes don't match. Shows warning dialog then continues |
| `SkipSyncException` | `lib/src/data/exceptions/SkipSyncException.dart` | Thrown by sync when no sync is needed. Treated as success and proceeds to home |

---

## API / Backend Dependencies

| Action | Via | Description |
|---|---|---|
| Connectivity check | `syncNotifierProvider.checkForConnection()` | Ping to CA server to verify reachability |
| OAuth login | `authenticationProvider.login()` → `FlutterAppAuth` | Opens system browser for OAuth flow |
| Token refresh | `authenticationProvider.refreshToken()` → `FlutterAppAuth` | Silent token refresh using stored refresh token |
| Full data sync | `syncNotifierProvider.runSync()` → `SyncDataService.runBatch()` | Dispatches five named batches sequentially; saves to Hive |
| Map tile download | `maptileProvider.downloadSection()` | Resumes any in-progress offline map section downloads |

The sync service (`SyncDataService.runBatch()`) dispatches five named batches in sequence. Batch progress is persisted to `SyncQueueState` in SharedPreferences, so a sync that was interrupted mid-way can resume from the failed batch rather than starting over.

| # | Batch enum | Class | Steps | Contents |
|---|---|---|---|---|
| 0 | `meta` | `MetaBatch` | 0–3 | Update current user (if stale); configure log level; meta location types; meta attributes; overview sections |
| 1 | `locations` | `LocationsBatch` | 4–5 | Locations, localities |
| 2 | `locationContent` | `LocationContentBatch` | 6–7 | Location attributes, location overviews |
| 3 | `locationRelated` | `LocationRelatedBatch` | 8–10 | Reports, discounts, HLRs |
| 4 | `members` | `MembersBatch` | 11–12 + notifications | Boats, members; notifications (delta syncs only — skipped on full sync) |

Batches 1–3 share a `syncLocationCacheProvider` so each batch can read the already-loaded `Location` list from memory rather than re-reading from Hive.

---

## Data Flow

### Startup Sequence (Happy Path - Returning User, Online)

```
App launches → initialRoute: /authentication
  │
  ├─ initState: _isLoading = true
  │
  └─ _initApp() (post-frame callback)
       │
       ├─ 1. AppInitialisationService.initialize()
       │    └─ Sequential service init (see Services table above)
       │
       └─ 2. _attemptLogin()
            │
            ├─ Check refresh token expiry → valid, continue
            │
            ├─ Online check (syncNotifier.checkForConnection)
            │    └─ online → continue
            │
            ├─ Check access token validity
            │    ├─ valid → _handleApiCall(_syncDataAndDownloadMaps())
            │    └─ expired → _handleApiCall(refreshToken(), callback: _syncDataAndDownloadMaps)
            │
            ├─ _syncDataAndDownloadMaps()
            │    ├─ Resume any in-progress map section downloads (fire and forget)
            │    │
            │    ├─ First run (no lastSuccessfulSync):
            │    │    └─ await runSync()
            │    │         └─ OfflineDataSync widget shows progress with live countdown on retries
            │    │              ├─ on complete → handleHomeScreenRoute()
            │    │              ├─ after first retry fails → "Sync in background" button appears
            │    │              │    └─ onBackgroundSync → continueInBackground() → handleHomeScreenRoute()
            │    │              └─ after all retries exhausted → inline error + "Continue to app" button
            │    │                   └─ onBackgroundSync → continueInBackground() → handleHomeScreenRoute()
            │    │
            │    └─ Subsequent runs:
            │         └─ runSync() fires in background (not awaited)
            │              └─ App navigates to home immediately
            │                   └─ On sync failure → SyncFailedModal via OneContext
            │
            └─ handleHomeScreenRoute()
                 ├─ Guard: skip if not mounted or no longer on /authentication route
                 └─ pushNamedAndRemoveUntil → HomeScreen (clears back stack)
```

### Startup - Offline User

```
_attemptLogin()
  │
  ├─ Refresh token valid → continue
  │
  ├─ Online check fails
  │    └─ await Future.delayed(1400ms)
  │         │
  │         ├─ stopwatch.elapsed > 1s (slow timeout — server unreachable):
  │         │    ├─ setNextOnlineDue() → shows offline countdown text
  │         │    ├─ await Future.delayed(2s)
  │         │    └─ handleHomeScreenRoute() → HomeScreen (offline mode)
  │         │
  │         └─ stopwatch.elapsed ≤ 1s (fast failure — device offline):
  │              └─ _showOfflineModal()
  │                   └─ "Open in Offline Mode" button
  │                        ├─ setNextOnlineDue()
  │                        ├─ await Future.delayed(2s)
  │                        └─ handleHomeScreenRoute() → HomeScreen
  │
  └─ No refresh token → show LOGIN button
       └─ Tapping opens browser OAuth flow, which shows 'no connection' error
```

### Startup - Logged Out / Expired Token

```
_attemptLogin()
  │
  ├─ refreshToken expired or checkAuthStatus == false
  │    └─ setState(_isLoading = false)
  │         └─ UI crossfades to LOGIN button
  │
  └─ User taps LOGIN
       └─ _handleLogin()
            ├─ authenticationProvider.login() → OAuth browser flow
            ├─ On MissingScopeException → _showScopeErrorModal() (warning, continues)
            └─ _syncDataAndDownloadMaps(force: true) → full sync
       └─ handleHomeScreenRoute() → HomeScreen
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| App launch | `lib/src/app.dart` | None (defaults) | `initialRoute` |
| HomeScreen sync check | `lib/src/screens/home_screen.dart` | None (defaults) | `pushReplacementNamed` when a periodic sync is due |
| Settings - Switch Account | `lib/src/screens/settings/settings_screen.dart` | `checkAuthStatus: false, skipFirst: true` | `pushNamedAndRemoveUntil` after clearing storage |
| Settings - Logout | `lib/src/screens/settings/settings_screen.dart` | `checkAuthStatus: false` | `pushNamedAndRemoveUntil` after `authProvider.logout()` |
| Token refresh failure | `lib/core/api/interceptors/refresh_interceptor.dart` | None (defaults) | `OneContext().pushNamedAndRemoveUntil` when a mid-session token refresh fails (e.g. refresh token revoked). Uses `OneContext` because it fires outside the widget tree. Guarded: skips redirect if already on the authentication screen |

---

## Known Caveats / Tech-Debt Notes

1. **Hardcoded 1400ms delay** (line 156): A `Future.delayed(Duration(milliseconds: 1400))` is used in the offline path before checking how long the connectivity check took. This creates a perceptible wait on every offline launch regardless of how quickly the failure was detected.

2. **`accessTokenExpiration` force-unwrap** (line 170): `ref.read(authenticationProvider).accessTokenExpiration!` is force-unwrapped. If the auth state has a valid refresh token but a null access token expiration (edge case), this will crash.

3. **Connectivity heuristic based on timing** (lines 157): The offline detection logic uses `stopwatch.elapsed.inSeconds > 1` to distinguish between "device is offline" (fast failure) and "server unreachable" (slow timeout). This is fragile — a 1.1-second DNS timeout could silently skip the offline modal.

4. **OfflineDataSync side-effects in `build`** (widget lines 23–29): The `OfflineDataSync` widget calls `onBackgroundSync()` and `onComplete()` inside `addPostFrameCallback` based on `syncState`. Navigation side-effects therefore fire after a widget build, which can cause "setState during build" errors in edge cases.

5. **Sequential service initialisation**: All services are initialised sequentially in `AppInitialisationService.initialize()`. Independent initialisations (e.g. `friendSettingProvider`, `notificationProvider`, `maptileProvider`) could be parallelised with `Future.wait` to reduce startup time.

6. **Map section downloads are fire-and-forget**: Resumed map section downloads use `.onError` to capture to Sentry but the futures are not awaited or tracked. If downloads fail silently, the user has no indication.

7. **No retry mechanism**: If the connectivity check or token refresh fails, the only recovery is the offline modal or showing the login button. There is no "Retry" option to re-attempt without restarting the app.