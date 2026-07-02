---
id: 84360cdf-9c48-4481-895b-8bae0068a4be
blueprint: captains_mate_app_43
title: Geolocation
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780582541
---
# flutter_background_geolocation

**Package:** `flutter_background_geolocation: ^4.18.3`
**Alias:** Imported as `bg` throughout the codebase

---

## Purpose

The `flutter_background_geolocation` package provides the core GPS/location engine for the app. It is used for three distinct features:

1. **Foreground position tracking** — showing the user's blue dot on the map
2. **Background friend-position broadcasting** — sharing the user's location with friends even when the app is backgrounded or terminated
3. **Geofence monitoring** — triggering local notifications when the user enters a radius around a saved location

---

## Architecture Overview

```
main.dart
 └── bg.BackgroundGeolocation.registerHeadlessTask(headlessTask)

AuthenticationScreen (after login)
 ├── PositionHelper.init()              ← bg.BackgroundGeolocation.ready() + start()
 ├── LocationGeofenceService.initialise() ← addGeofences + onGeofence listener
 └── PositionBroadcastHelper.init()     ← sets up isolate + location callback

HomeScreen
 ├── _registerListeners()               ← bg.onLocation for blue-dot updates
 ├── didChangeAppLifecycleState         ← toggles config between foreground/background
 └── dispose()                          ← bg.removeListener

SettingsScreen
 ├── "Share my location" toggle          ← start/stopBroadcasting
 ├── "Background updates" toggle         ← start/stopBroadcasting
 └── "Geographic notifications" toggle   ← LocationGeofenceService.start/stop
```

---

## Files Involved

| File | Role |
|---|---|
| `lib/main.dart` | Registers the headless task handler. Handles `LOCATION`, `TERMINATE`, and `GEOFENCE` headless events |
| `lib/src/helpers/position.dart` | **PositionHelper** — core wrapper around BackgroundGeolocation. Handles `ready()`, `start()`, `stop()`, `setConfig()`, `setSharingConfig()`, `getCurrentPosition()`, `watchCurrentPosition()` |
| `lib/src/helpers/position_broadcast.dart` | **PositionBroadcastHelper** — manages friend-position broadcasting. Uses Dart Isolates to bridge between bg location callbacks and API calls |
| `lib/src/features/locations/services/location_geofence_service.dart` | **LocationGeofenceService** — singleton that manages geofence registration, syncing, and event handling |
| `lib/src/helpers/enable_localation_sharing_helper.dart` | **EnableLocationSharingHelper** — UI-facing helper that orchestrates the "switch to friends tab" flow, including permission checks and broadcasting setup |
| `lib/src/helpers/headless_user_helper.dart` | **HeadlessUserHelper** — builds a `HeadlessUser` object from SharedPreferences/SecureStorage for use in headless (no-Riverpod) contexts |
| `lib/src/helpers/config.dart` | **Config** — distance thresholds and timeout constants |
| `lib/src/screens/home_screen.dart` | **HomeScreen** — foreground listener registration, lifecycle management |
| `lib/src/screens/authentication_screen.dart` | Initialization entry point after login |
| `lib/src/screens/settings/settings_screen.dart` | User-facing toggles for sharing, background updates, and geofence notifications |

---

## Configuration

### Initial Config (`PositionHelper.init()`)

Called once during the auth flow. Sets up BackgroundGeolocation with default foreground-only settings:

| Parameter | Value | Notes |
|---|---|---|
| `desiredAccuracy` | `DESIRED_ACCURACY_HIGH` | |
| `distanceFilter` | `300.0` (foreground) | Updates when user moves > 300m — increased from 20m to reduce battery usage |
| `stationaryRadius` | `100.0` | Increased from 25m — a yacht at anchor can swing/drift beyond 25m |
| `locationAuthorizationRequest` | `'WhenInUse'` | Only foreground permission by default |
| `showsBackgroundLocationIndicator` | `false` | No iOS status bar indicator |
| `elasticityMultiplier` | `0` | Disables distance-filter elasticity |
| `stopOnTerminate` | `true` | Stops tracking when app killed |
| `startOnBoot` | `true` | |
| `preventSuspend` | `false` | |
| `enableHeadless` | `false` | No headless events by default |

### Sharing Config (`PositionHelper.setSharingConfig()`)

Applied when user enables background sharing or geofence notifications. Upgrades to background-capable mode:

| Parameter | Value | Notes |
|---|---|---|
| `locationAuthorizationRequest` | `'Always'` | Requests "Always" permission |
| `showsBackgroundLocationIndicator` | `true` | iOS status bar indicator shown |
| `distanceFilter` | `300.0` (background and foreground) | Same value used in both contexts |
| `enableHeadless` | `true` | Headless events enabled |
| `stopOnTerminate` | `false` | Continues after app killed |

### Config Constants (`Config`)

| Constant | Value | Usage |
|---|---|---|
| `foregroundDistance` | `300.0` m | Distance filter (foreground and background) — increased from 20m to reduce battery usage |
| `backgroundDistance` | `300.0` m | Distance threshold for broadcasting API calls |
| `stationaryRadius` | `100.0` m | Stationary detection radius — increased from 25m to avoid spurious movement detection from tidal swing |
| `locationTimeout` | `5` s | Timeout for `getCurrentPosition()` |

---

## Feature 1: Foreground Position Tracking

### Flow

```
HomeScreen.initState
 └── _registerListeners()
      ├── bg.BackgroundGeolocation.onLocation(callback)
      │    └── callback: setState(_currentPosition = LatLng from event)
      └── PositionHelper.getCurrentPosition()  ← triggers first callback

HomeScreen.didChangeAppLifecycleState(paused)
 ├── bg.BackgroundGeolocation.removeListener(callback)
 ├── If sharing/geofence: PositionHelper.setSharingConfig()
 └── Else: PositionHelper.setConfig() + stop()

HomeScreen.didChangeAppLifecycleState(resumed)
 ├── PositionHelper.start()
 ├── _registerListeners()  ← re-registers onLocation
 └── If sharing/geofence: setSharingConfig(foregroundDistance)

HomeScreen.dispose
 └── bg.BackgroundGeolocation.removeListener(callback)
```

### Position Caching

`PositionHelper.getCurrentPosition()` has a 2-minute cache. If called within 2 minutes of the last update, it returns the cached `_currentPosition` instead of querying the GPS.

---

## Feature 2: Background Friend Broadcasting

### Foreground Broadcasting

```
User enables sharing (Settings or Friends tab switch)
 │
 ├── PositionBroadcastHelper.startBroadcasting()
 │    ├── PositionHelper.setSharingConfig(foregroundDistance)
 │    ├── broadcastOnce()  ← immediate API call
 │    ├── IsolateNameServer.registerPortWithName(port, 'BackgroundPositionIsolate')
 │    └── PositionHelper.watchCurrentPosition(_onLocation)
 │
 └── _onLocation callback (fires on each location event):
      ├── Check: has user moved > 300m from last broadcast?
      ├── Check: isBackground OR hasMoved?
      └── If yes: _isolateCallback(location)
           └── SendPort → ReceivePort → broadcastOnce()
                └── ApiClient.friends.publicisePosition()
```

### Headless Broadcasting (app terminated)

```
headlessTask(LOCATION event)
 │
 ├── bg.BackgroundGeolocation.startBackgroundTask()  ← keep-alive
 │
 ├── PositionBroadcastHelper.headlessTask(location)
 │    ├── SharedPreferencesHelper (fresh init — no Riverpod)
 │    ├── Check: backgroundUpdatesEnabled && isPositionSharingEnabled?
 │    ├── Check: moved > 300m from lastLocation (stored in prefs)?
 │    └── If yes: broadcastOnce() via API
 │
 └── bg.BackgroundGeolocation.stopBackgroundTask(taskId)

headlessTask(TERMINATE event)
 │
 ├── Check: isSharing || isGeofenceSharing?
 └── If yes: PositionHelper.setSharingConfig()  ← ensures bg distance filter
```

### Isolate Architecture

Broadcasting uses Dart Isolates to decouple the bg location callback from the API call:

```
bg.onLocation callback
  └── _isolateCallback(location)
       └── IsolateNameServer.lookupPortByName('BackgroundPositionIsolate')
            └── SendPort.send({latitude, longitude, headlessUser})

ReceivePort listener (top-level _port)
  └── PositionBroadcastHelper.broadcastOnce(position, headlessUser)
       └── ApiClient.friends.publicisePosition()
```

---

## Feature 3: Geofence Monitoring

### Setup

```
AuthenticationScreen._loadProviders
 └── LocationGeofenceService().initialise()
      ├── LocationRepository().getAll()  ← get locations with non-triggered notifications
      ├── syncGeofences(locations)
      │    ├── bg.BackgroundGeolocation.removeGeofences()  ← clear all
      │    └── bg.BackgroundGeolocation.addGeofences(newGeofences)
      │         └── Each geofence: bg.Geofence(
      │              identifier: location.id,
      │              radius: notification.radius,
      │              lat/lng from location.position,
      │              notifyOnEntry: true,
      │              extras: {location_id, type: 'location'}
      │           )
      │
      ├── If geofence disabled in prefs: stop() + return
      └── start()
           ├── PositionHelper.setSharingConfig()  ← needs 'Always' permission
           └── bg.BackgroundGeolocation.onGeofence(onGeofence)
```

### Geofence Event Handling

```
onGeofence(GeofenceEvent)  ← both foreground and headless
 │
 ├── Check: geofenceNotificationEnabled?
 ├── Check: event.extras has 'type' == 'location'?
 ├── LocationRepository().get(locationId)
 ├── Check: notification exists and not already triggered?
 │
 ├── FcmService().handleNotificationViaHeadless(RemoteMessage)
 │    └── Shows a local notification to the user
 │
 └── LocationRepository().save(location with triggered: true)
      └── Prevents re-triggering
```

### Sync During Data Refresh

Geofences are re-synced whenever the app performs a full data sync:

```
SyncDataService (step in sync pipeline)
 └── LocationGeofenceService().syncGeofences(locations)
      └── removeGeofences() → addGeofences(filtered)
```

---

## Lifecycle Management (HomeScreen)

| App State | Actions |
|---|---|
| **Active (foreground)** | `onLocation` listener registered, `foregroundDistance` (20m) filter, `WhenInUse` or `Always` depending on sharing |
| **Paused/Detached** | Listener removed. If sharing/geofence: `setSharingConfig()` (300m bg filter, headless enabled). Else: `setConfig()` + `stop()` |
| **Resumed** | `start()`, re-register listener, if sharing/geofence: `setSharingConfig(foregroundDistance)` to get fine-grained updates in foreground |
| **Terminated** | Headless task handler: if sharing, ensures bg config. Broadcasts position on LOCATION events. Handles GEOFENCE events |

---

## Permission Model

| Feature | Permission Required | Requested Via |
|---|---|---|
| Foreground tracking | `WhenInUse` | `PositionHelper.init()` → `checkPermissions()` via `geolocator` |
| Background sharing | `Always` | `setSharingConfig()` → `locationAuthorizationRequest: 'Always'` (triggers bg permission prompt via the plugin) |
| Geofence monitoring | `Always` | Same as above — `LocationGeofenceService.start()` calls `setSharingConfig()` |

Permission checks use the **`geolocator`** package (`Geolocator.checkPermission()`, `Geolocator.requestPermission()`), not the bg plugin's own permission API. The bg plugin's `locationAuthorizationRequest` config simply tells the plugin what level to operate at.

---

## Known Technical Debt & Issues

### 1. Two location packages used for permissions

**Files:** `position.dart`, `enable_localation_sharing_helper.dart`

Permission checking uses `geolocator` (`Geolocator.checkPermission()`, `Geolocator.requestPermission()`) while location tracking uses `flutter_background_geolocation`. This creates a dual-dependency where two separate packages manage the same system resource. The bg plugin has its own permission management (`locationAuthorizationRequest` config), but it's partially bypassed by using `geolocator` for the actual permission requests. This can cause inconsistent states — e.g., `geolocator` might report `WhenInUse` permission while the bg plugin is configured to expect `Always`.

### 2. `dispose()` calls `super.dispose()` before removing the observer and listener

**File:** [home_screen.dart:247-260](lib/src/screens/home_screen.dart#L247-L260)

```dart
void dispose() {
  super.dispose();                          // ← called first
  WidgetsBinding.instance.removeObserver(this);
  bg.BackgroundGeolocation.removeListener(...)
}
```

`super.dispose()` is called before `removeObserver` and `removeListener`. If a lifecycle event fires during disposal, `didChangeAppLifecycleState` could be called on a disposed widget, potentially calling `setState`. The conventional order is cleanup-first, then `super.dispose()`.

### 3. `checkPermissions()` called but result not awaited in `startBroadcasting`

**File:** [position_broadcast.dart:127](lib/src/helpers/position_broadcast.dart#L127)

```dart
static Future<void> startBroadcasting({...}) async {
  PositionHelper.checkPermissions();  // ← not awaited
  await PositionHelper.setSharingConfig(...);
```

The permission check is fire-and-forget. If permissions are not granted, the code proceeds to `setSharingConfig()` and registers location listeners anyway. The bg plugin will handle the permission mismatch internally, but the app won't know the check failed.

### 4. Isolate architecture adds unnecessary indirection in foreground

**File:** [position_broadcast.dart:15-32](lib/src/helpers/position_broadcast.dart#L15-L32)

The `ReceivePort`/`SendPort` isolate mechanism is used for all location broadcasts, including foreground. In the foreground, the `_onLocation` callback calls `_isolateCallback` which sends through `IsolateNameServer` back to the same main isolate's `ReceivePort`. This is a roundabout way to call `broadcastOnce()` — it could call it directly when in the foreground. The isolate bridge is only genuinely needed for the headless path.

### 5. `_onLocation` uses `late` initialisation without guard

**File:** [position_broadcast.dart:37](lib/src/helpers/position_broadcast.dart#L37)

```dart
static late Function(bg.Location) _onLocation;
```

`_onLocation` is declared `late` but only initialised inside `init()`. If `startBroadcasting()` or `stopBroadcasting()` is called before `init()`, accessing `_onLocation` (e.g., in `removeListener`) throws a `LateInitializationError`. There is no guard against this sequence.

### 6. `headlessTask` TERMINATE event doesn't `await` the config change

**File:** [main.dart:51](lib/main.dart#L51)

```dart
if (isSharing || isGeofenceSharing) PositionHelper.setSharingConfig();
```

`setSharingConfig()` is async but not awaited. The headless task may complete before the config is applied, meaning the bg plugin could terminate before the background distance filter is set.

### 7. `isBackground` flag is a mutable static with no synchronisation

**File:** [position_broadcast.dart:40](lib/src/helpers/position_broadcast.dart#L40)

```dart
static bool isBackground = false;
```

This public mutable static is toggled by `HomeScreen.didChangeAppLifecycleState` and read by `_onLocation`. In the foreground, `shouldPost = isBackground || (!isBackground && hasMoved)` uses this flag to decide whether to always post (background) or only post on movement (foreground). Since location callbacks can fire on background threads, there is a theoretical race between the lifecycle toggle and the callback read.

### 8. Distance check logic differs between foreground and headless

**Files:** [position_broadcast.dart:53](lib/src/helpers/position_broadcast.dart#L53) vs [position_broadcast.dart:112](lib/src/helpers/position_broadcast.dart#L112)

**Foreground (`_onLocation`):**
```dart
bool shouldPost = isBackground || (!isBackground && hasMoved);
```
In foreground mode, posts only if moved > 300m. In background mode, posts unconditionally.

**Headless (`headlessTask`):**
```dart
if (isSharing && hasMoved) { ... }
```
Always requires movement > 300m, regardless of foreground/background. The inconsistency means the headless path is more conservative — it won't post if the user hasn't moved, even though the foreground-background path would.

### 9. `syncGeofences` removes ALL geofences then re-adds

**File:** [location_geofence_service.dart:142-168](lib/src/features/locations/services/location_geofence_service.dart#L142-L168)

```dart
await bg.BackgroundGeolocation.removeGeofences();
// ... filter and re-add
await bg.BackgroundGeolocation.addGeofences(newGeofences);
```

Every sync wipes all geofences and re-registers them. During the window between `removeGeofences()` and `addGeofences()`, no geofences are active. If the user crosses a geofence boundary during a data sync, it will be missed. A diff-based approach (only adding/removing changed geofences) would be more robust.

### 10. `LocationGeofenceService.start()` unconditionally calls `setSharingConfig()`

**File:** [location_geofence_service.dart:101-107](lib/src/features/locations/services/location_geofence_service.dart#L101-L107)

```dart
Future<void> start() async {
  await PositionHelper.setSharingConfig();
  bg.BackgroundGeolocation.onGeofence(onGeofence);
}
```

`start()` calls `setSharingConfig()` which requests `Always` permission, enables headless mode, and sets `stopOnTerminate: false`. This is a side effect — enabling geofence notifications permanently upgrades the location config to background mode, even if the user hasn't opted into friend position sharing. The `stop()` method only reverts the config if `isPositionSharingEnabled` is false, so the two features (sharing and geofencing) have interleaved config management.

### 11. `onGeofence` is a top-level function used as both foreground and headless handler

**File:** [location_geofence_service.dart:11](lib/src/features/locations/services/location_geofence_service.dart#L11)

The `onGeofence` function is a top-level function (not a class method) that creates a new `SharedPreferencesHelper()` and `LocationRepository()` on every invocation. In the headless context this is necessary (no Riverpod), but in the foreground it bypasses the provider system entirely, reading directly from Hive. This means foreground geofence handling doesn't benefit from any caching or state management that the provider layer provides.

### 12. `print` statement left in headless task

**File:** [main.dart:42](lib/main.dart#L42)

```dart
print('Position broadcast complete');
```

A `print` statement was left in the LOCATION headless event handler instead of using `TcaLog`.

### 13. Filename typo: `enable_localation_sharing_helper.dart`

**File:** `lib/src/helpers/enable_localation_sharing_helper.dart`

The filename contains a typo — "localation" instead of "location".

### 14. No position staleness check in broadcasting

**File:** [position_broadcast.dart:80-93](lib/src/helpers/position_broadcast.dart#L80-L93)

`broadcastOnce()` can be called with `position: null`, in which case the API is called with no position parameter (relying on the server to handle it). There is no check for whether the position is stale or how old the GPS fix is. The bg plugin provides `location.coords.accuracy` and `location.age`, but these are never inspected before broadcasting.

### 15. `_isHandlingLifecycle` guard may mask legitimate lifecycle events

**File:** [home_screen.dart:64-137](lib/src/screens/home_screen.dart#L64-L137)

The `_isHandlingLifecycle` flag prevents re-entry during lifecycle handling. However, if the handler takes a long time (e.g., awaiting permission checks or sync operations), legitimate lifecycle transitions during that window are silently dropped. For example, if the user backgrounds the app while a resumed handler is still running, the paused event would be ignored.