---
id: c37f594f-ec7c-42c0-b8aa-9fb252227535
blueprint: captains_mate_app_43
title: 'Home Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774533829
---
# HomeScreen (`lib/src/screens/home_screen.dart`)

## Purpose

The HomeScreen is the primary screen of the app, presented after authentication and data sync. It serves as the central hub for two core features:

1. **Location browsing** - an interactive map with a sliding-up panel listing searchable, filterable, sortable locations.
2. **Friends tracking** - real-time position sharing and viewing of friends on the map.

The screen also bootstraps several background concerns on entry: position broadcasting, geofence configuration, foreground GPS listening, and unread-notification prompts.

Route name: `/home`

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
    ├── SlidingUpPanel
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

## Associated Providers

| Provider | Type | Purpose |
|---|---|---|
| `friendSettingProvider` | `StateNotifierProvider` | Position-sharing prefs (enabled, background updates, friend name, email, show-name). |
| `friendsProvider` | `StateProvider<List<Friend>>` | Current list of friends sharing positions. |
| `homeDrawerProvider` | `StateProvider<bool>` | Whether the drawer is open. |
| `locationsProvider` | `FutureProvider<List<Location>>` | All locations fetched from the repository. |
| `searchedAndFilteredLocationsProvider` | `FutureProvider<List<Location>>` | Locations after applying search, type/attribute filters, and sort. |
| `locationMapSelectedProvider` | `StateProvider<String?>` | ID of the currently selected location on the map. |
| `locationSearchProvider` | `StateProvider<String>` | Current search-bar text. |
| `locationSortProvider` | `StateProvider<LocationSort>` | Active sort mode (nearest, nearestCenter, name, date). |
| `locationTypeFiltersProvider` | `StateProvider<List<String>>` | Active location-type filters. |
| `locationAttributeFiltersProvider` | `StateProvider<List<String>>` | Active location-attribute filters. |
| `mapPositionProvider` | `StateProvider<LatLng?>` | Current map centre (debounced). |
| `savedMapSectionProvider` | `FutureProvider<List<SavedMapSection>>` | Downloaded offline map sections. |
| `sharedPreferencesProvider` | `Provider<SharedPreferencesHelper>` | App-wide persistent settings (position sharing, background updates, geofence, etc.). |
| `syncNotifierProvider` | `StateNotifierProvider` (annotation-based) | Data-sync lifecycle (checkForSync, runSync). |
| `hasSeenNotificationModal` | `StateProvider<bool>` | Guards the unread-notification modal from re-showing. |
| `homeSelectedTabProvider` | `StateProvider<HomeScreenTabOption>` | Currently active tab (locations or friends). |

---

## Associated Models

| Model | Key Fields | Source |
|---|---|---|
| `Location` | `id`, `name`, `position` (LatLng), `types`, `attributes`, `reports`, `hlrs`, `images`, `isEditable`, `distance` (computed) | `features/locations/` |
| `Friend` | `id`, `boatname`, `name`, `position` (LatLng), `updated`, `emailable`, `isOurs`, `memberId` | `features/friends/` |
| `SavedMapSection` | `id`, `name`, `topLeft`/`bottomRight` (LatLng), `minZoom`, `maxZoom`, `downloadProgress` | `features/maptiles/` |
| `HomeScreenRoute` | `locationId?`, `friendId?`, `sectionId?`, `showUnreadNotificationModal?` | `helpers/route_argumnets/` |
| `HeadlessUser` | `friendName`, `allowEmail`, `showName` | `helpers/headless_user_helper.dart` |

---

## Associated Widgets (Bespoke)

| Widget | File | Role |
|---|---|---|
| `TcaHomeAppBar` | `widgets/tca_home_app_bar.dart` | App bar with Locations/Friends segmented control. |
| `TcaDrawer` | `widgets/tca_drawer.dart` | Side-navigation drawer. |
| `TcaHomeScreenMap` | `widgets/home_screen/tca_home_screen_map.dart` | Map layer: tiles, markers, popups, FABs. |
| `TcaMap` | `widgets/tca_map.dart` | Core flutter_map wrapper (tile layers, clustering, offline sections). |
| `TcaMapSwitcher` | `widgets/tca_map_switcher.dart` | Map/satellite style toggle. |
| `TcaLocationMarker` | `widgets/tca_location_marker.dart` | Custom location pin on map. |
| `TcaFriendMarker` | `widgets/tca_friend_marker.dart` | Custom friend pin on map. |
| `TcaHlrMarker` | (inside map widgets) | Harbour-limit polygon overlay. |
| `TcaLocationSheet` | `widgets/tca_location_sheet.dart` | Sliding panel with search, filters, sort, and location list. |
| `TcaSearchBar` | (child of location sheet) | Text search + filter-toggle bar. |
| `TcaBackgroundGradient` | `widgets/tca_background_gradient.dart` | Decorative gradient behind the app bar. |
| `TcaHomeScreenHiddenLocationModal` | `widgets/home_screen/tca_home_screen_hidden_location_modal.dart` | Dialog shown when a deep-linked location is hidden by filters. |
| `TcaHomeScreenFriendModal` | `widgets/home_screen/tca_home_screen_friend_modal.dart` | Bottom sheet when a friend marker is tapped. |

---

## API / Backend Dependencies

| Endpoint | Method | Trigger |
|---|---|---|
| `GET /friends` | `FriendsApiClient.getFriends()` | Switching to Friends tab (via `EnableLocationSharingHelper`). |
| `POST /friends` | `FriendsApiClient.publicisePosition()` | `PositionBroadcastHelper.broadcastOnce()` / `startBroadcasting()` on init and resume. |
| `DELETE /friends` | `FriendsApiClient.unpublicisePosition()` | `PositionBroadcastHelper.stopBroadcasting(unpublicise: true)` when sharing is revoked. |
| Location sync | `LocationRepository.getAll()` (local DB, populated by sync) | `locationsProvider` read during `_goToLocation` and panel rendering. |
| Saved map sections | `SavedMapSectionRepository.getAll()` (local DB) | `savedMapSectionProvider` read during `_goToLocation`. |
| Data sync check | `SyncDataService` / `ApiClient.connectivity` | `syncNotifierProvider.checkForSync()` on app resume. |
| Current user | `ApiClient.user.getCurrentUser()` | `HeadlessUserHelper.getUser()` fallback if not cached. |

Most data (locations, map sections, notifications) is accessed through a local repository layer populated by a periodic background sync, not by direct API calls from this screen.

---

## Data Flow

### Startup (`initState`)

```
initState
  ├─ Register GPS listener (_registerListeners)
  │   └─ bg.BackgroundGeolocation → onLocation → setState(_currentPosition)
  │
  ├─ Check position-sharing state (SharedPreferences + Permission)
  │   ├─ If sharing enabled & has permission → start/broadcast position
  │   └─ If no permission or disabled but still broadcasting → stop & unpublicise
  │
  └─ Post-first-render callback
      ├─ _goToLocation() → read route args → move map to location/friend/section
      └─ _handleUnreadNotificationModal() → show modal if flagged in route args
```

### App Lifecycle (resume / pause)

```
AppLifecycleState.paused
  ├─ Stop foreground GPS listener
  ├─ If sharing or geofencing → switch to background config
  └─ Otherwise → stop GPS entirely

AppLifecycleState.resumed
  ├─ syncNotifier.checkForSync() → if refresh needed → navigate to AuthScreen
  ├─ Re-register foreground GPS listener
  └─ Restore sharing/foreground config
```

### Map Position Updates

```
User pans map
  → onMapCenterChanged(LatLng)
  → DebounceHelper (1 s)
  → mapPositionProvider.state = center
  → searchedAndFilteredLocationsProvider re-evaluates "nearest to center" sort
```

### Tab Switching (Locations ↔ Friends)

```
User taps tab / drawer item
  → EnableLocationSharingHelper.switchFriendTab()
      ├─ Check: position sharing enabled? → else show dialog
      ├─ Check: friend name set? → else show dialog
      ├─ Request permissions if needed
      ├─ Fetch friends list from API
      └─ Update homeSelectedTabProvider
  → Panel visibility toggled (hide for Friends, show for Locations)
```

---

## Known Caveats / Tech Debt

1. **Typo in path** - `helpers/route_argumnets/` is misspelled (`arguments`). Similarly the helper filename `enable_localation_sharing_helper.dart` has a typo (`location`).

2. **Silent error swallowing** - The `_registerListeners` GPS callback catches all exceptions and silently fails with only a log message; errors during map-position updates could go unnoticed.

3. **Lifecycle re-entry guard** - `_isHandlingLifecycle` is a manual flag to prevent re-entry from permission dialogs triggering `inactive` → `resumed` cycles. This is fragile and could mask legitimate lifecycle transitions.

4. **`_postFirstRender` flag** - Used to skip lifecycle handling before the first frame, but couples widget lifecycle to rendering state in a way that could break if the build order changes.

5. **Mixed async patterns** - The file mixes `.then().catchError()` chains with `async/await`. Consistency would improve readability and error handling.

6. **Large `initState`** - Position-broadcasting setup, permission checks, route handling, and notification logic are all inlined in `initState`, making it hard to test or reuse.

7. **`friendsProvider` is `autoDispose`** - The friends list is disposed when all listeners are removed, meaning navigating away and back may trigger unnecessary re-fetches.

8. **Tight coupling to `PositionHelper` statics** - Most position logic lives in static methods on `PositionHelper` and `PositionBroadcastHelper`, which makes unit testing difficult without a dependency-injection seam.

9. **Null position handling** - `_goToLocation` falls back to `_gotoMyLocation` when position is null after handling route args, but if both the route handler and GPS fail, the user lands on the fallback position (CA House, London) with no visible feedback.

10. **Panel state tracked outside widget state** - `_panelOpen` is toggled in callbacks without `setState`, which is intentional (no re-render needed) but relies on the variable only being read inside other callbacks, not in `build`.

11. **Possible bug** - Android: when switching to friends without a boat name set and sharing on, no friend pins appear. Dialogs ask to set boat name and sharing on but friend pins don't appear until next time the app is launched.