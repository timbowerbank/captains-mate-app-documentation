---
id: c37f594f-ec7c-42c0-b8aa-9fb252227535
blueprint: captains_mate_app_43
title: 'Home Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780405250
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# HomeScreen
**File:** `lib/src/screens/home_screen.dart`
**Route:** `/home`

## Purpose

The HomeScreen is the primary screen of the app, presented after authentication and data sync. It serves as the central hub for two core features:

1. **Location browsing** — an interactive map with a sliding-up panel listing searchable, filterable, sortable locations.
2. **Friends tracking** — real-time position sharing and viewing of friends on the map.

The screen also bootstraps several background concerns on entry: foreground GPS listening, position broadcasting, and unread-notification prompts.

---

## UI Structure (Simplified Tree)

```
Scaffold
├── appBar: TcaHomeAppBar
│   └── Segmented control (Locations | Friends tab switcher)
│
├── drawer: TcaDrawer
│   └── Navigation items (Locations, Friends, Membership, Notifications, Settings, …)
│
└── body: Stack
    ├── SlidingUpPanel                       ← sliding_up_panel package
    │   ├── body: TcaHomeScreenMap
    │   │   ├── TcaMap (flutter_map core)
    │   │   │   ├── TcaLocationMarker (per location)
    │   │   │   ├── TcaFriendMarker (per friend)
    │   │   │   ├── TcaHlrMarker (harbour-limit polygons)
    │   │   │   └── Popup layer (flutter_map_marker_popup)
    │   │   ├── TcaMapSwitcher (map ↔ satellite toggle)
    │   │   ├── My-Location FAB
    │   │   ├── Create-Location FAB
    │   │   └── Loading indicator
    │   │
    │   └── panel: TcaLocationSheet
    │       ├── TcaSearchBar (text search + filter toggle)
    │       ├── TcaDropdownButton (sort: nearest / name / date)
    │       └── TcaListItem (per location)
    │
    └── TcaBackgroundGradient (decorative top fade)
```

---

## Layout Calculations

Panel dimensions and FAB (Floating Action Button) positioning are computed by `HomeScreenLayoutCalculations` (`lib/src/helpers/home_screen_layout_calculations.dart`) using `MediaQueryData` on every build.

### `bottomPadding(data)`
Used as additional height added to the panel's `minHeight`.

- **Android:** `max(padding.bottom - viewInsets.bottom, 0)` — accounts for gesture navigation bars and avoids negative values when the keyboard is visible
- **iOS:** `Dimens.marginDefault` — fixed margin, as iOS safe area is handled differently

### `panelHeight(data)`
The maximum height the panel can expand to:

```
availableHeight = screen height − top padding − toolbar height − bottom padding
panelHeight     = max(availableHeight − keyboard inset, Dimens.minPanelHeight)
```

Clamped to `Dimens.minPanelHeight` as a floor so the panel is never rendered with a negative or zero height.

### `mapButtonBottomPadding(positionSharingEnabled, selectedTab, data)`
Controls the vertical position of the map FABs (My Location, Create Location):

- **Friends tab with position sharing on:** `Dimens.marginDefault + padding.bottom` — FABs sit just above the bottom safe area since the panel is hidden
- **All other states:** `Dimens.minPanelHeight + (Dimens.marginSmall / 2) + padding.bottom` — FABs sit above the collapsed panel

---

## SlidingUpPanel Configuration

The sliding panel (`sliding_up_panel` package) is configured as follows:

| Property | Value | Effect |
|---|---|---|
| `minHeight` | `Dimens.minPanelHeight + bottomPadding` | Collapsed height, adjusted per platform |
| `maxHeight` | `panelHeight(data)` | Expanded height, fills available screen space |
| `snapPoint` | `0.65` | Panel snaps to 65% of max height on release |
| `parallaxEnabled` | `true` | Map scrolls as panel moves |
| `parallaxOffset` | `0.5` | Map moves at half the speed of the panel |
| `renderPanelSheet` | `false` | Disables the default white background sheet |

The panel is controlled via `_panelController` (`PanelController`). It is hidden when the Friends tab is active with position sharing enabled, and shown otherwise.

---

## Associated Providers

| Provider | Type | Purpose |
|---|---|---|
| `positionBroadcastingProvider` | `Provider` | Orchestrates position-sharing setup on init. Checks prefs and permissions, starts/stops broadcasting accordingly |
| `positionProvider` | `StreamProvider<LatLng>` | Streams GPS position updates. Kept alive with a no-op `ref.listen` in `build` to seed `sortPositionProvider` with the real GPS position |
| `currentPositionProvider` | `StateProvider<LatLng?>` | Latest raw GPS position. Updated on every fix |
| `sortPositionProvider` | `StateProvider<LatLng?>` | Position used for "nearest" location sorting. Only updates when the device has moved ≥1 nautical mile to avoid resorting 9,000 locations on every GPS tick |
| `mapPositionProvider` | `StateProvider<LatLng?>` | Debounced map centre position. Updated as the user pans the map |
| `permissionsProvider` | `FutureProvider<PermissionStatus>` | Device permission state. Invalidated on app resume so the UI reflects any permission changes made in device settings |
| `friendSettingProvider` | `StateNotifierProvider` | Position-sharing prefs (enabled, background updates, friend name, email, show-name) |
| `friendsProvider` | `StateProvider<List<Friend>>` | Current list of friends sharing positions |
| `homeDrawerProvider` | `StateProvider<bool>` | Whether the drawer is open |
| `homeSelectedTabProvider` | `StateProvider<HomeScreenTabOption>` | Currently active tab (locations or friends) |
| `locationsProvider` | `FutureProvider<List<Location>>` | All locations fetched from the repository |
| `searchedAndFilteredLocationsProvider` | `FutureProvider<List<Location>>` | Locations after applying search, type/attribute filters, and sort |
| `locationMapSelectedProvider` | `StateProvider<String?>` | ID of the currently selected location on the map |
| `locationSearchProvider` | `StateProvider<String>` | Current search-bar text |
| `locationSortProvider` | `StateProvider<LocationSort>` | Active sort mode (nearest, nearestCenter, name, date) |
| `locationTypeFiltersProvider` | `StateProvider<List<String>>` | Active location-type filters |
| `locationAttributeFiltersProvider` | `StateProvider<List<String>>` | Active location-attribute filters |
| `savedMapSectionProvider` | `FutureProvider<List<SavedMapSection>>` | Downloaded offline map sections |
| `sharedPreferencesProvider` | `Provider` | App-wide persistent settings |
| `syncNotifierProvider` | `StateNotifierProvider` | Data-sync lifecycle |
| `notificationProvider` | `StateNotifierProvider` | Notification repository access — used to fetch unread notifications and look up a notification by ID |
| `pendingNotificationProvider` | `StateProvider<NotificationData?>` | Holds a tapped notification waiting for navigation. Set by the FCM tap handler or `_checkForPendingNotification` at startup; cleared after navigation |
| `hasSeenNotificationModal` | `StateProvider<bool>` | Guards the unread-notification modal from re-showing more than once per session |

---

## Associated Models

| Model | Key Fields | Source |
|---|---|---|
| `Location` | `id`, `name`, `position` (LatLng), `types`, `attributes`, `reports`, `hlrs`, `images`, `isEditable`, `distance` (computed) | `features/locations/` |
| `Friend` | `id`, `boatname`, `name`, `position` (LatLng), `updated`, `emailable`, `isOurs`, `memberId` | `features/friends/` |
| `SavedMapSection` | `id`, `name`, `topLeft`/`bottomRight` (LatLng), `minZoom`, `maxZoom`, `downloadProgress` | `features/maptiles/` |
| `HomeScreenRoute` | `locationId?`, `friendId?`, `sectionId?` | `helpers/route_argumnets/` |

---

## Associated Widgets (Bespoke)

| Widget | File | Role |
|---|---|---|
| `TcaHomeAppBar` | `widgets/tca_home_app_bar.dart` | App bar with Locations/Friends segmented control |
| `TcaDrawer` | `widgets/tca_drawer.dart` | Side-navigation drawer |
| `TcaHomeScreenMap` | `widgets/home_screen/tca_home_screen_map.dart` | Map layer: tiles, markers, popups, FABs |
| `TcaMap` | `widgets/tca_map.dart` | Core flutter_map wrapper (tile layers, clustering, offline sections) |
| `TcaMapSwitcher` | `widgets/tca_map_switcher.dart` | Map/satellite style toggle |
| `TcaLocationMarker` | `widgets/tca_location_marker.dart` | Custom location pin on map |
| `TcaFriendMarker` | `widgets/tca_friend_marker.dart` | Custom friend pin on map |
| `TcaHlrMarker` | (inside map widgets) | Harbour-limit polygon overlay |
| `TcaLocationSheet` | `widgets/tca_location_sheet.dart` | Sliding panel with search, filters, sort, and location list |
| `TcaBackgroundGradient` | `widgets/tca_background_gradient.dart` | Decorative gradient behind the app bar |
| `TcaHomeScreenHiddenLocationModal` | `widgets/home_screen/tca_home_screen_hidden_location_modal.dart` | Dialog shown when a deep-linked location is hidden by filters |
| `TcaHomeScreenFriendModal` | `widgets/home_screen/tca_home_screen_friend_modal.dart` | Bottom sheet when a friend marker is tapped |

---

## API / Backend Dependencies

| Endpoint | Method | Trigger |
|---|---|---|
| `GET /friends` | `FriendsApiClient.getFriends()` | Switching to Friends tab via drawer (`EnableLocationSharingHelper`) |
| `POST /friends` | `FriendsApiClient.publicisePosition()` | `PositionBroadcastHelper.broadcastOnce()` / `startBroadcasting()` on init |
| `DELETE /friends` | `FriendsApiClient.unpublicisePosition()` | `PositionBroadcastHelper.stopBroadcasting(unpublicise: true)` when sharing is revoked |
| Location data | `LocationRepository.getAll()` (local DB) | `locationsProvider` read during `_goToLocation` and panel rendering |
| Saved map sections | `SavedMapSectionRepository.getAll()` (local DB) | `savedMapSectionProvider` read during `_goToLocation` |
| Data sync check | `SyncDataService` / `ApiClient.connectivity` | `syncNotifierProvider.checkForSync()` on app resume |
| Current user | `ApiClient.user.getCurrentUser()` | `HeadlessUserHelper.getUser()` fallback if not cached |

Most data is accessed through a local repository layer populated by a periodic background sync, not by direct API calls from this screen.

---

## Data Flow

### Startup (`initState`)

```
initState
  ├─ FcmService().sendToken()  (re-registers FCM token on every home screen mount)
  │
  ├─ Register WidgetsBindingObserver (app lifecycle)
  │
  ├─ _registerPositionListeners()
  │   └─ PositionHelper.watchCurrentPosition
  │       → callback → setState(_currentPosition)
  │       → PositionHelper.getCurrentPosition() triggers first callback
  │
  ├─ positionBroadcastingProvider.initialize()
  │   ├─ Read: isPositionSharingEnabled, backgroundUpdatesEnabled,
  │   │         geofenceNotificationEnabled, locationAlways permission
  │   ├─ If geofence enabled → setSharingConfig (foreground distance)
  │   ├─ If sharing disabled or no permission, but currently broadcasting
  │   │   → stopBroadcasting(unpublicise: true), clear pref
  │   ├─ If sharing disabled → return
  │   └─ If sharing enabled
  │       ├─ backgroundUpdates on → startBroadcasting()
  │       └─ backgroundUpdates off → broadcastOnce()
  │
  └─ Post-first-render callback
      ├─ _goToLocation()
      ├─ _handleNotifications()  (pending tap → navigate; unread → modal)
      ├─ _startupReady = true    (gates pendingNotificationProvider listener in build)
      └─ setState(_postFirstRender = true)
```

### App Lifecycle (resume / pause)

```
AppLifecycleState.paused / detached
  ├─ _startupReady = false  (prevents notification listener firing on stale state)
  ├─ _notificationHandled = false  (allows re-check on next resume)
  ├─ PositionBroadcastHelper.setBackground(true)
  ├─ Remove foreground GPS listener
  ├─ syncNotifier.reset()  (if not currently processing — clears state for fresh resume check)
  ├─ If position sharing OR geofence notifications enabled
  │   → PositionHelper.setSharingConfig() (background config)
  └─ Otherwise
      → PositionHelper.setConfig() + PositionHelper.stop()

AppLifecycleState.resumed
  ├─ ref.invalidate(permissionsProvider)
  ├─ syncNotifier.checkForSync(isStartUp: false)  (skipped if sync already processing)
  │   └─ If sync due → pushReplacementNamed → AuthenticationScreen
  ├─ PositionBroadcastHelper.setBackground(false)
  ├─ PositionHelper.start()
  ├─ _registerPositionListeners() (re-register foreground GPS listener)
  ├─ If sharing OR geofence enabled
  │   → PositionHelper.setSharingConfig(foregroundDistance)
  │  Otherwise
  │   → PositionHelper.setConfig()
  ├─ _handleNotifications()  (re-check for pending taps or unread notifications)
  └─ _startupReady = true
```

> Both `paused` and `detached` are handled identically. `inactive` is deliberately ignored to avoid reacting to permission dialogs. Re-entry is guarded by `_isHandlingLifecycle`.

### Sync Feedback

A `ref.listen(syncNotifierProvider)` in `build` handles two transitions:

- **`isSuccess`** — shows a SnackBar: _"All set! Your data is up to date."_
- **`isBackgroundError`** — shows `SyncFailedModal` via `OneContext()`, with a message tailored to whether it was a full or partial sync failure

### Notification Handling

Notification handling runs from three entry points and is coordinated by two instance flags.

#### Entry Points

| When | How |
|---|---|
| Startup | `_handleNotifications()` called in the post-frame callback after `initState` |
| App resumed | `_handleNotifications()` called at the end of the `resumed` lifecycle handler |
| Notification arrives while screen is active | `ref.listen(pendingNotificationProvider)` in `build` calls `_handlePendingNotification()` directly |

#### Instance Flags

- **`_notificationHandled`** — set to `true` on the first call to `_handleNotifications()` within a lifecycle cycle; prevents double-handling if both the post-frame callback and a provider event fire in quick succession. Reset to `false` on `paused`/`detached`.
- **`_startupReady`** — set to `true` after the post-frame startup check completes, and again after the `resumed` handler finishes. Gates the `pendingNotificationProvider` `ref.listen` in `build` so it does not fire during the initial startup sequence before `_handleNotifications()` has already run.

#### `_handleNotifications()` Flow

```
_handleNotifications()
  │
  ├─ Guard: return if _notificationHandled already true
  ├─ _notificationHandled = true
  │
  ├─ 1. Check pendingNotificationProvider
  │       Set during cold start (from getNotificationAppLaunchDetails)
  │       or by the FCM tap handler (foreground / iOS background)
  │       └─ If set → _handlePendingNotification() → return
  │
  ├─ 2. Check SharedPreferences (pendingTappedNotificationId)
  │       Set by background isolate on Android when container unavailable
  │       └─ If found → look up NotificationData by ID from Hive
  │                    → _handlePendingNotification() → return
  │
  └─ 3. Fall through to unread check
          → _handleUnreadNotifications()
```

#### `_handlePendingNotification()`

1. Clears SharedPreferences pending ID (prevents re-fire if both paths fired)
2. Sets `hasSeenNotificationModal = true` (suppresses unread modal — user is already going to the notification screen)
3. Guards against navigating if `NotificationsListScreen` is already the active route
4. Pushes `NotificationsListScreen` with `notificationId` argument so the matching notification is focused and highlighted
5. Clears `pendingNotificationProvider` after navigation returns

#### `_handleUnreadNotifications()`

1. Guards via `hasSeenNotificationModal` — returns immediately if already shown this session
2. Sets `hasSeenNotificationModal = true` before the async fetch to prevent a second call winning the race
3. Fetches unread notifications via `notificationProvider`
4. If any unread exist, shows `UnreadNotificationModal` via `OneContext`

### Map Position Updates

```
User pans map
  → onMapCenterChanged(LatLng)
  → DebounceHelper
  → mapPositionProvider.state = center
  → searchedAndFilteredLocationsProvider re-evaluates "nearest to center" sort
```

### Navigating to a Location / Friend / Section

```
_goToLocation()
  ├─ Read HomeScreenRoute from route arguments
  ├─ If no route args → _gotoMyLocation()
  │
  ├─ handleLocation(locationId)
  │   ├─ Find location in locationsProvider
  │   └─ If location filtered out → show TcaHomeScreenHiddenLocationModal
  │
  ├─ handleFriend(friendId)
  │   └─ Find friend in friendsProvider
  │
  └─ handleSection(sectionId)
      ├─ Find section in savedMapSectionProvider
      └─ Set zoom to section.minZoom
  │
  ├─ If position resolved → mapController.move(position, zoom)
  │   └─ Also calls _gotoMyLocation(shouldMove: false) to show
  │      the user's position dot without moving the map
  └─ If position unresolved → _gotoMyLocation()
```

### Tab Switching (Locations ↔ Friends)

**From the app bar** — directly shows or hides the panel:
```
positionSharingEnabled && switching to Friends tab
  → _panelController.hide()
Otherwise
  → _panelController.show()
```

**From the drawer** — routes through `EnableLocationSharingHelper.switchFriendTab()`, which handles permission checks, friend-name validation, and API calls before updating the tab.

---

## Known Caveats / Tech Debt

1. **Typo in paths** — `helpers/route_argumnets/` is misspelled (`arguments`). The helper filename `enable_localation_sharing_helper.dart` also has a typo (`location`).

2. **Silent error swallowing** — The `_registerListeners` GPS callback catches all exceptions with only a log message; errors during map-position updates could go unnoticed in production.

3. **Lifecycle re-entry guard** — `_isHandlingLifecycle` is a manual boolean flag to prevent re-entry from permission dialogs triggering `inactive` → `resumed` cycles. This is fragile and could mask legitimate lifecycle transitions.

4. **`_postFirstRender` flag** — Used to skip lifecycle handling before the first frame, but couples widget lifecycle to rendering state in a way that could break if build order changes.

5. **Mixed async patterns** — The file mixes `.then().catchError()` chains with `async/await`. Consistency would improve readability and error handling.

6. **Tight coupling to `PositionHelper` statics** — Most position logic lives in static methods on `PositionHelper` and `PositionBroadcastHelper`, making unit testing difficult without a dependency-injection seam.

7. **Null position handling** — `_goToLocation` falls back to `_gotoMyLocation()` when position is null, but if both the route handler and GPS fail, the user lands on the fallback position (CA House, London) with no visible feedback.

8. **Panel state tracked outside widget state** — `_panelOpen` is toggled in callbacks without `setState`, which is intentional (no re-render needed) but relies on the variable only being read inside other callbacks, not in `build`.

9. **`positionProvider` kept alive with a no-op listen** — `ref.listen(positionProvider, (_, __) {})` in `build` is required to prevent `sortPositionProvider` from staying null and falling back to the London default. This is a non-obvious side-effect dependency that could break silently if the listen is removed.

10. **Friends tab requires second tap after first-time setup** — When switching to the Friends tab without a boat name set, a dialog directs the user to Settings to add one. After returning, the user must tap the Friends tab a second time for friend pins to appear — `getFriends()` is not retried automatically after the dialog completes. This affects both iOS and Android.