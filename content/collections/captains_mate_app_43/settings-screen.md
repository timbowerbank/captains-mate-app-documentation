---
id: 3b295089-9daa-419e-ab84-04264b92c455
blueprint: captains_mate_app_43
title: 'Settings Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535667
---
# SettingsScreen

**File:** `lib/src/screens/settings/settings_screen.dart`
**Route:** `/settings`

---

## Purpose

The central settings hub of the application. It provides grouped controls for managing **Friends** (location sharing, email permissions, boat name, background updates), **Preferences** (reports display/sort, data refresh frequency), **Admin** (boat show mode — only visible to demonstrator users), **Notification Preferences** (background and geofence notifications), **Offline Storage** (saved maps, sync, clear and redownload, clear cached maps), and **Account** (logged-in user, staging API toggle for admins, developer mode via hidden 5-tap, view logs, report issue, logout). Settings are persisted via `SharedPreferencesHelper` and `FriendSettingProvider`.

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    └── Text ("SETTINGS")
 │
 └── body: Consumer (watches friendSettingProvider + authenticationProvider)
      │
      └── ListView
           │
           ├── ── Friends ──────────────────────────
           ├── TcaSettingSwitch ("Share my location with friends")
           │    └── Toggles position broadcasting on/off
           ├── TcaSettingSwitch ("Allow friends to email me")
           ├── TcaSettingItem ("Boat name shown to other members")
           │    └── onTap → _showFriendNameDialog()
           ├── TcaSettingSwitch ("Show my own name to other members")
           ├── TcaSettingSwitch ("Background updates")
           │    └── Disabled when location sharing is off
           │
           ├── ── Preferences ──────────────────────
           ├── TcaSettingItem ("Reports default display")
           │    └── onTap → _showReportsDefaultDisplayDialog()
           ├── TcaSettingItem ("Reports default sort")
           │    └── onTap → _showReportsDefaultSortDialog()
           ├── TcaSettingSlider ("Data refresh frequency")
           │    └── 7 stops: Auto → 2h → 6h → 12h → 1d → 2d → Manual
           │
           ├── ── Admin (if isDemonstrator) ────────
           ├── TcaSettingSwitch ("Boat Show Mode")
           │
           ├── ── Notification Preferences ─────────
           ├── TcaSettingSwitch ("Background notifications")
           ├── TcaSettingSwitch ("Geographic notifications")
           │    └── Starts/stops LocationGeofenceService
           │
           ├── ── Offline Storage ──────────────────
           ├── TcaSettingItem ("Saved Maps") → ManageSavedMapsScreen
           ├── Consumer > TcaSettingItem ("Update your App with the latest data now")
           │    └── onTap → _showSyncLocationsDialog(canClose: true)
           ├── Consumer > TcaSettingItem ("Clear and redownload all locations")
           │    └── onTap → _showClearLocationsDialog()
           ├── TcaSettingItem ("Clear cached maps")
           │    └── MaptileHelper.resetCaches() + update size display
           │
           ├── ── Account ──────────────────────────
           ├── TcaMultiTapInkwell (5 taps → toggle developer mode)
           │    └── TcaSettingItem ("Logged in as {name}")
           ├── if isAdmin: TcaSettingSwitch ("Use Staging API")
           │    └── Confirms → logout → switch API → AuthenticationScreen
           ├── if developerMode: TcaSettingItem ("View Logs")
           │    └── → ViewStoredLogScreen
           ├── TcaSettingItem ("Report an issue") → ReportIssueScreen
           └── TcaSettingItem ("Logout")
                └── logout → AuthenticationScreen
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `friendSettingProvider` | `lib/src/data/providers/friend_setting.dart` | `StateNotifierProvider` — watched for current friend settings (position sharing, email, name, background updates, show name). Written to via notifier methods for each toggle/field |
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Watched for `currentUser` (name, membership, isDemonstrator, isAdmin). Notifier called for `logout()` |
| `sharedPreferencesProvider` | `lib/src/data/providers/shared_preferences.dart` | Reads initial values on init (reportsDefaultState, reportsDefaultSort, refreshFrequency, boatShowMode, developerModeEnabled, useStagingApi, notificationEnabled, geofenceNotificationEnabled). Writes updates for each preference change |
| `syncNotifierProvider` | `lib/src/features/sync/providers/sync_provider.dart` | Watched (via nested `Consumer`) for sync loading state. Notifier called for `runSync(force: true)` |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Called during "Clear and redownload" to `deleteSyncData()` — wipes all synced Hive data |

### Models / Enums

| Model / Enum | File | Role |
|---|---|---|
| `CurrentUser` | `lib/src/features/user/models/current_user.dart` | Provides `name`, `email`, `membership` (with `boatname`, `memberNumber`), `isDemonstrator`, `isAdmin` |
| `TcaReportsDisplayOption` | `lib/src/enums/reports_display_option.dart` | Enum: `collapsed`, `expanded`. Used for the reports default display dialog |
| `TcaReportsSortOption` | `lib/src/enums/reports_sort_option.dart` | Enum: `newest`, `oldest`. Used for the reports default sort dialog |
| `TcaRefreshFrequency` | `lib/src/enums/refresh_frequency.dart` | Enum with 7 values: `auto`, `twoHours`, `sixHours`, `twelveHours`, `oneDay`, `twoDays`, `manual`. Used for the slider |
| `AuthenticationScreenArguments` | `lib/src/screens/authentication_screen.dart` | Route args for logout/API-switch navigation: `checkAuthStatus`, `skipFirst` |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaSettingHeader` | `lib/src/widgets/settings/setting_header.dart` | Section header text (e.g. "Friends", "Preferences") |
| `TcaSettingSwitch` | `lib/src/widgets/settings/setting_switch.dart` | Toggle switch setting with title, subtitle, loading, and disabled states |
| `TcaSettingItem` | `lib/src/widgets/settings/setting_item.dart` | Tappable setting row with title, subtitle, and optional loading state |
| `TcaSettingSlider` | `lib/src/widgets/settings/setting_slider.dart` | Slider setting with label display for the refresh frequency |
| `TcaMultiTapInkwell` | `lib/src/widgets/tca_multitap_inkwell.dart` | Detects 5 consecutive taps to toggle hidden developer mode |
| `TcaRadio` | `lib/src/widgets/tca_radio.dart` | Radio button used in the reports display/sort dialogs |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used in all dialogs for confirm/cancel actions |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Styled `AlertDialog` wrapper for all setting dialogs |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Text input in the friend name dialog |
| `SyncLocationsDialog` | `lib/src/features/sync/widgets/` (via `sync/index.dart`) | Dialog showing sync progress with optional "background sync" dismiss |

### Helpers / Services

| Helper | File | Role |
|---|---|---|
| `PositionBroadcastHelper` | `lib/src/helpers/position_broadcast.dart` | `startBroadcasting()` / `stopBroadcasting()` / `broadcastOnce()` — manages friend position sharing lifecycle |
| `PositionHelper` | `lib/src/helpers/position.dart` | `checkPermissions()` — validates location permissions before enabling sharing; `openSettings()` — opens system settings |
| `HeadlessUserHelper` | `lib/src/helpers/headless_user_helper.dart` | `getUser()` — builds a headless user object needed by the broadcast helper |
| `LocationGeofenceService` | `lib/src/features/locations/services/location_geofence_service.dart` | `start()` / `stop()` — enables/disables geofence-based notifications |
| `MaptileHelper` | `lib/src/helpers/maptile.dart` | `resetCaches()` — clears all cached map tiles from FMTC stores |
| `SnackbarHelper` | `lib/src/helpers/snackbar.dart` | Success messages for sync, cache clearing |
| `TcaLog` | `lib/src/helpers/log.dart` | Error logging for failed friend sharing operations |

---

## API / Backend Dependencies

No direct API calls are made from this screen. However, several actions trigger API interactions indirectly:

| Action | Indirect API Call | Description |
|---|---|---|
| Toggle location sharing ON | `PositionBroadcastHelper.broadcastOnce()` or `startBroadcasting()` | POSTs the user's position to the friends API |
| Toggle location sharing OFF | `PositionBroadcastHelper.stopBroadcasting(unpublicise: true)` | DELETEs the user's published position from the friends API |
| "Update your App" / "Clear and redownload" | `syncNotifierProvider.runSync(force: true)` | Triggers the full 12-step sync pipeline (sequential API calls for locations, members, boats, HLRs, etc.) |
| Logout | `authenticationProvider.logout()` | Revokes the OAuth2 token |

---

## Data Flow

### Initialisation

```
initState
  │
  ├─ Initialise _friendNameController from friendSettingProvider.friendName
  │
  └─ Post-frame callback:
       ├─ updateCachedMapsSize()
       │    └─ Read FMTC store sizes (basemap + satellite + seamap) → _cachedMapsSize
       │
       └─ Future.wait (8 SharedPreferences reads):
            ├─ reportsDefaultState → _reportsDefaultState
            ├─ reportsDefaultSort → _reportsDefaultSort
            ├─ refreshFrequency → _refreshFrequency
            ├─ boatShowMode → _boatShowMode
            ├─ developerModeEnabled → _developerModeEnabled
            ├─ useStagingApi → _isUsingStaging
            ├─ notificationEnabled → _isNotificationEnabled
            └─ geofenceNotificationEnabled → _isGeofenceNotificationEnabled
```

### Location Sharing Toggle

```
User toggles "Share my location" ON
  │
  ├─ Check if friendName or boatName exists
  │    └─ If neither: show missing-details dialog → return
  │
  ├─ friendSettingProvider.setIsPositionSharingEnabled(true)
  │
  ├─ PositionHelper.checkPermissions()
  │    └─ If denied: show permission dialog → reset to false → return
  │
  ├─ HeadlessUserHelper.getUser()
  │
  ├─ if backgroundUpdates enabled:
  │    └─ PositionBroadcastHelper.startBroadcasting()
  │
  └─ else: PositionBroadcastHelper.broadcastOnce()

  on DioException:
    └─ Reset isPositionSharingEnabled to previous value
```

### Sync / Clear and Redownload

```
User taps "Clear and redownload"
  │
  ├─ if already syncing: show sync dialog → return
  │
  ├─ Show confirmation dialog → "CLEAR & DOWNLOAD" or "CANCEL"
  │
  ├─ repositoryManagerProvider.deleteSyncData()
  │    └─ Wipes all synced Hive data
  │
  ├─ sharedPreferencesProvider.setLastLocationFetchTime(null)
  │
  └─ _showSyncLocationsDialog(canCloseDialog: false)
       ├─ Shows SyncLocationsDialog (non-dismissible)
       │
       └─ syncNotifierProvider.runSync(force: true)
            └─ Full 12-step sync pipeline
```

### Logout

```
User taps "Logout"
  │
  ├─ authenticationProvider.logout()
  │
  └─ Navigator.pushNamedAndRemoveUntil(
       AuthenticationScreen.routeName,
       (route) => false,
       args: AuthenticationScreenArguments(checkAuthStatus: false)
     )
```

---

## Navigation Callers

| Source | File | Context |
|---|---|---|
| TcaDrawer | `lib/src/widgets/tca_drawer.dart` | "Settings" menu item in the app drawer |

---

## Known Caveats / Tech-Debt Notes

1. **All preferences loaded via async `SharedPreferences` in `initState`** (lines 84-107): Eight preference values are loaded asynchronously and set via `setState`. Until the future completes, the UI shows default values (e.g. `_boatShowMode = false`, `_refreshFrequency = twoDays`). There is no loading indicator — toggles are interactive before their true values are loaded, so a user could flip a switch that immediately gets overwritten by the async result.

2. **`_isShowNameLoading` set to `false` twice** (lines 252, 256): In the "Show my own name" toggle handler, `setState(() => _isShowNameLoading = false)` is called before the provider update and again after. The first call should likely be `true` to show a loading state, but both are `false`, meaning the loading indicator never appears.

3. **`DioException` error message in sharing log is misleading** (lines 199-203, 288-293): Both the "share location" and "background updates" error handlers log `'Friend Sharing: failed to disable: '` even when the failure is on the enable path. The message doesn't reflect the actual operation.

4. **Switch between slider integers and enum values** (lines 321-346): The refresh frequency slider maps integer values (0-6) to enum values via a manual `switch` statement. This is fragile — if new enum values are added, the switch and slider `max`/`divisions` must all be updated in sync.

5. **Developer mode toggle has side effect** (lines 460-463): Tapping the logged-in username 5 times toggles developer mode AND unconditionally calls `setLogEnabledAt(DateTime.now())`. This means even if developer mode is being toggled OFF, the log-enabled timestamp is reset. The intent seems to be enabling logging when developer mode is activated, but the toggle-off path should arguably not set this.

6. **`_showSyncLocationsDialog` guard may be insufficient** (lines 636-638): The method checks `syncNotifierProvider.isLoading` after showing the dialog, but there is a race condition — the dialog is shown via `showDialog` (which doesn't await), then the sync check happens. If another sync starts between the dialog show and the check, the guard catches it, but the dialog is already visible.

7. **No confirmation on "Clear cached maps"** (lines 436-445): Tapping "Clear cached maps" immediately calls `MaptileHelper.resetCaches()` with no confirmation dialog. This could delete significant amounts of cached data on an accidental tap, unlike the saved maps deletion which requires confirmation.

8. **`_showConfirmSwitchApiDialog` label is inverted** (lines 837-844): The dialog title says "Switch to Staging API" when `isProduction` is `true` and "Switch to Production API" when `false`. However, the parameter is named `isProduction` but actually receives the new toggle value (i.e. `true` means "use staging"), making the naming confusing.

9. **Massive single widget**: At ~870 lines, this screen handles 15+ distinct settings, 8+ dialogs, and multiple async workflows in a single file. The complexity makes it difficult to reason about state interactions and would benefit from being broken into smaller setting-group widgets.

10. **`friendSettingProvider` read-then-write pattern** (e.g. lines 140-141, 268): Several toggle handlers read the current friend settings state, then write back to the notifier. Between the read and write, another event could modify the state (though this is unlikely in practice since it's driven by user interaction).

11. **`TcaSettingSwitch` is  an adaptive Flutter widget: uses Cupertino on iOS and Material on Android. Any changes need to be tested on both devices as properties are interpreted differently.