---
id: 3b295089-9daa-419e-ab84-04264b92c455
blueprint: captains_mate_app_43
title: 'Settings Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780413531
parent: faa4011a-a306-467e-ac40-635e775f6e76
---
# Settings Screen

**File:** `lib/src/screens/settings/settings_screen.dart`
**Route:** `/settings`

---

## Purpose

The central settings hub of the application. It provides grouped controls for managing **Friends** (location sharing, email permissions, boat name, background updates), **Preferences** (reports display/sort, data refresh frequency), **Admin** (boat show mode — only visible to demonstrator users), **Notification Preferences** (background and geofence notifications), **Offline Storage** (saved maps, sync, clear and redownload, clear cached maps), and **Account** (logged-in user, staging API toggle for admins, developer mode via hidden 5-tap, view logs, report issue, logout). Settings are persisted via `SharedPreferencesHelper` and `FriendSettingProvider`.

Switch availability is gated on device-level permissions, read on every build via `permissionsProvider`. The relevant gates are:

| Permission | Controls |
|---|---|
| Notification | Background notifications toggle enabled/disabled |
| Location (When In Use) | Location sharing toggle enabled/disabled |
| Location (Always) | Background updates toggle, Geographic notifications toggle |
| Motion/Activity | Warning shown in subtitle when Always location is granted but motion is denied |

---

## Permission and Preference Model

The screen enforces a two-layer model for every togglable setting:

- **Layer 1 — Device permission** (read from `permissionsProvider` on every `build`): Controls whether the toggle is interactive (`isDisabled`) and whether the displayed value can show `true`.
- **Layer 2 — Stored preference** (`SharedPreferences` or `FriendSettingProvider`): The user's expressed intent.

The displayed toggle value is always computed as `storedPreference && hasPermission`. If a user enables a feature and then revokes the relevant device permission, the toggle shows as off — but the stored preference is preserved. When the user re-grants permission, the toggle automatically reflects the saved preference with no re-tapping required.

**Exception — geofence preference is actively reset on init**: During `initState`, if `(!hasAlwaysLocation || !hasNotification) && geofenceEnabled == true`, the geofence preference is explicitly written to `false` in SharedPreferences. The code comment explains the deliberate policy: granting the missing permission later should not silently auto-enable geofence notifications without the user choosing to do so.

Background notifications intentionally does not follow this pattern — granting notification permission is treated as equivalent to wanting to receive notifications, so auto-enabling that toggle on permission grant is considered expected behaviour.

### Toggle State Rules

| Setting | Disabled when | Displayed value | Preference storage |
|---|---|---|---|
| Share my location | No location permission, OR no friend name AND no boat name | `isPositionSharingEnabled && hasLocation` | `FriendSettingProvider` |
| Allow friends to email | — | `allowEmailFromFriends` | `FriendSettingProvider` |
| Background updates | No Always location permission, OR location sharing is off | `enabledBackgroundUpdates && hasAlwaysLocation && sharingOn` | `FriendSettingProvider` |
| Background notifications | No notification permission | `_isNotificationEnabled && hasNotification` | `notification_enabled` |
| Geographic notifications | No notification permission, OR no Always location permission | `_isGeofenceNotificationEnabled && hasNotification && hasAlwaysLocation` | `geofence_notification_enabled` |

### Subtitle Warnings

When a toggle is disabled by a missing permission, the subtitle changes from the normal description to an instruction. When multiple permissions are missing the most specific combined message is shown.

| Setting | Condition | Subtitle shown |
|---|---|---|
| Location sharing | No location permission | "Enable location in device settings to use this feature" |
| Location sharing | No friend name AND no boat name | "You must set a name before sharing your location" |
| Location sharing | Enabled | "Your updated location will continue to be shared periodically" |
| Background updates | No Always location permission | "Enable location (Always) in device settings to use this feature" |
| Background updates | Enabled | "The App will send position updates even if not being used" |
| Background notifications | No notification permission | "Enable notifications in device settings to use this feature" |
| Background notifications | Enabled | "Receive notifications while the app is in the background" |
| Geographic notifications | No notification AND no Always location | "Enable notifications and location (Always) in device settings to use this feature" |
| Geographic notifications | No notification permission only | "Enable notifications in device settings to use this feature" |
| Geographic notifications | No Always location only | "Enable location (Always) in device settings to use this feature" |
| Geographic notifications | Enabled | "Enable the app to notify you when close to points of interest (uses background geolocation)" |

**Motion / Activity permission warning** — appended to the normal subtitle (not shown when the toggle is already disabled by a missing permission):

- Applies to: Background updates and Geographic notifications
- Condition: Always location is granted AND `permissions.hasMotionPermission == false`
- iOS text: "Enable Motion & Fitness in device settings for better battery life"
- Android text: "Enable Physical Activity in device settings for better battery life"
- Only appended when the feature is actually active: for background updates, only when `enabledBackgroundUpdates && isPositionSharingEnabled`; for geographic notifications, only when `_isGeofenceNotificationEnabled`

### Boat Name Subtitle

The "Boat name shown to other members" item has no toggle — its subtitle shows the current name resolution in priority order:

| Condition | Subtitle |
|---|---|
| Custom friend name set | "Using custom: {friendName}" |
| No custom name, membership boat name exists | "Using default: {boatname}" |
| Neither set | "None set" |

---

## UI Simplified Structure

```
TcaSafeScaffold
 ├── AppBar
 │    └── Text ("SETTINGS")
 │
 └── body: Consumer (watches friendSettingProvider + authenticationProvider)
      │
      └── ListView
           │
           ├── ── Friends ──────────────────────────
           ├── TcaSettingSwitch ("Share my location with friends")
           │    ├── Disabled if no location permission, or no friend/boat name set
           │    └── Toggles position broadcasting on/off
           ├── TcaSettingSwitch ("Allow friends to email me")
           ├── TcaSettingItem ("Boat name shown to other members")
           │    └── onTap → _showFriendNameDialog()
           ├── TcaSettingSwitch ("Show my own name to other members")
           ├── TcaSettingSwitch ("Background updates")
           │    └── Disabled if no Always location permission or sharing is off
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
           │    └── Disabled if no notification permission
           ├── TcaSettingSwitch ("Geographic notifications")
           │    ├── Disabled if no notification permission or no Always location
           │    └── Starts/stops LocationGeofenceService
           │
           ├── ── Offline Storage ──────────────────
           ├── TcaSettingItem ("Saved Maps") → ManageSavedMapsScreen
           ├── Consumer > TcaSettingItem ("Update your App with the latest data now")
           │    └── onTap → _showSyncLocationsDialog()
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
| `permissionsProvider` | `lib/src/data/providers/permissions_provider.dart` | `FutureProvider<PermissionStatus>` — watched on every build. Provides `hasNotification`, `hasLocation`, `hasAlwaysLocation`, `hasMotionPermission`. Gates which switches are enabled |
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
| `TcaSettingSwitch` | `lib/src/widgets/settings/setting_switch.dart` | Toggle switch setting with title, subtitle, loading, and disabled states. Adaptive: uses Cupertino on iOS and Material on Android — changes must be tested on both platforms |
| `TcaSettingItem` | `lib/src/widgets/settings/setting_item.dart` | Tappable setting row with title, subtitle, and optional loading state |
| `TcaSettingSlider` | `lib/src/widgets/settings/setting_slider.dart` | Slider setting with label display for the refresh frequency |
| `TcaMultiTapInkwell` | `lib/src/widgets/tca_multitap_inkwell.dart` | Detects 5 consecutive taps to toggle hidden developer mode |
| `TcaRadio` | `lib/src/widgets/tca_radio.dart` | Radio button used in the reports display/sort dialogs |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used in all dialogs for confirm/cancel actions |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Styled `AlertDialog` wrapper for all setting dialogs |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Text input in the friend name dialog |
| `SyncLocationsDialog` | `lib/src/features/sync/widgets/` | Dialog showing sync progress with optional background-sync dismiss |

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
| "Update your App" | `syncNotifierProvider.runSync(force: true, startAsBackground: false)` | Forced delta sync (uses `lastSyncTfrom` as `tfrom`, or full sync if never synced). Always runs in foreground so the progress modal is shown |
| "Clear and redownload" | `syncNotifierProvider.runSync(force: true, startAsBackground: false)` | Always a full sync — `lastSyncTfrom`, `lastSuccessfulSync`, and `syncQueueState` are all reset to null before this call, so `tfrom` is null and all 5 batches run from scratch |
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
  ├─ setState(_isLocationSharingLoading = true)
  │
  ├─ Check if friendName and boatName are both empty
  │    └─ If so: show missing-details dialog → (finally) loading = false → return
  │
  ├─ PositionHelper.checkPermissions()
  │    └─ If denied: show permission dialog → (finally) loading = false → return
  │
  ├─ friendSettingProvider.setIsPositionSharingEnabled(true)
  │
  ├─ If hasAlwaysLocationPermission:
  │    └─ Also sets enabledBackgroundUpdates = true automatically
  │
  ├─ HeadlessUserHelper.getUser()
  │
  ├─ if backgroundUpdates enabled (or Always permission granted):
  │    └─ PositionBroadcastHelper.startBroadcasting()
  └─ else: PositionBroadcastHelper.broadcastOnce()

User toggles "Share my location" OFF
  │
  ├─ friendSettingProvider.setIsPositionSharingEnabled(false)
  └─ PositionBroadcastHelper.stopBroadcasting(unpublicise: true)

on DioException (either direction):
  ├─ TcaLog.error('Friend Sharing: failed to update: ...')
  └─ Reset isPositionSharingEnabled to previous value

finally: setState(_isLocationSharingLoading = false)
```

### Sync

Both sync actions route through `_showSyncLocationsDialog()`, which shows the `SyncLocationsDialog` (non-dismissible) and calls `runSync(force: true, startAsBackground: false)`. The difference is whether `lastSyncTfrom` has been cleared beforehand:

```
User taps "Update your App"
  │
  └─ _showSyncLocationsDialog()
       ├─ If already syncing: guard returns early (dialog already visible)
       ├─ Shows SyncLocationsDialog (barrierDismissible: false)
       │    ├─ "Sync in background" button visible immediately (delta sync)
       │    │    └─ Tap → continueInBackground() + Navigator.pop(dialog)
       │    └─ Countdown shown during retry delays
       └─ syncNotifierProvider.runSync(force: true, startAsBackground: false)
            └─ Delta sync — lastSyncTfrom is intact, so only changes since
               last sync are fetched. Runs as full sync if never synced before.
```

```
User taps "Clear and redownload"
  │
  ├─ If already syncing: show sync dialog → return
  │
  ├─ Show confirmation dialog → "CLEAR & DOWNLOAD" or "CANCEL"
  │
  ├─ setLastSuccessfulSync(null)   ← reset before delete to ensure full
  ├─ setLastSyncTfrom(null)        ← cleared so next sync has no tfrom
  ├─ setSyncQueueState(null)       ← ensures next sync starts from batch 0
  │
  ├─ repositoryManagerProvider.deleteSyncData()
  │    └─ Wipes all synced Hive data
  │
  └─ _showSyncLocationsDialog()
       ├─ Shows SyncLocationsDialog (barrierDismissible: false)
       │    ├─ "Sync in background" button hidden until first batch failure (full sync)
       │    │    └─ Tap → continueInBackground() + Navigator.pop(dialog)
       │    └─ Countdown shown during retry delays
       └─ syncNotifierProvider.runSync(force: true, startAsBackground: false)
            └─ Full sync — lastSyncTfrom is null so tfrom is null and all
               5 batches run from scratch
```

#### Retry behaviour

Batches are retried by `SyncQueueRunner` on transient failures. Each retry waits `Config.retryDelay` before re-attempting the same batch; the countdown is shown in the dialog. The runner retries up to `Config.syncMaxRetries` consecutive times before giving up and rethrowing. Non-retryable errors (e.g. auth failures, classified by `SyncErrorCategory.shouldFailImmediately`) skip retries entirely and fail immediately.

#### Error routing

When `runSync` catches a terminal error it emits `SyncState.error(isBackground: _isBackgroundSync)`. Which code surfaces it to the user depends on `_isBackgroundSync`:

| `_isBackgroundSync` | Who handles the error |
|---|---|
| `false` (foreground) | `SyncLocationsDialog` — pops itself and shows `SyncFailedModal` via `OneContext` |
| `true` (background) | HomeScreen's `ref.listen(syncNotifierProvider)` — shows `SyncFailedModal` |

`_isBackgroundSync` starts as `false` (because `startAsBackground: false` is passed). If the user taps "Sync in background", `continueInBackground()` flips it to `true` and the dialog is dismissed — subsequent errors are then handled by HomeScreen.

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

### API Switch (Admin only)

```
User toggles "Use Staging API"
  │
  ├─ _showConfirmSwitchApiDialog(value) → confirm or cancel
  │
  ├─ authenticationProvider.logout()
  ├─ sharedPreferencesProvider.setUseStagingApi(value)
  │
  └─ Navigator.pushNamedAndRemoveUntil(
       AuthenticationScreen.routeName,
       (route) => false,
       args: AuthenticationScreenArguments(checkAuthStatus: false, skipFirst: true)
     )
```

---

## Navigation Callers

| Source | File | Context |
|---|---|---|
| TcaDrawer | `lib/src/widgets/tca_drawer.dart` | "Settings" menu item in the app drawer |

---

## Known Caveats / Tech-Debt Notes

1. **All preferences loaded via async `SharedPreferences` in `initState`** (lines 90–114): Eight preference values are loaded asynchronously via `Future.wait` in a post-frame callback. Until the future completes, the UI shows default values (e.g. `_boatShowMode = false`, `_refreshFrequency = twoDays`). There is no loading indicator — toggles are interactive before their true values are loaded, so a user could flip a switch that is immediately overwritten by the async result.

2. **`_isShowNameLoading` set to `false` twice** (lines 285, 289): In the "Show my own name" toggle handler, `setState(() => _isShowNameLoading = false)` is called both before and after the provider update. The first call should be `true` to show a loading state — both being `false` means the loading indicator never appears.

3. **Switch between slider integers and enum values** (lines 361–384): The refresh frequency slider maps integer values (0–6) to enum values via a manual `switch` statement. If new enum values are added, the switch, `max`, and `divisions` must all be updated in sync.

4. **Developer mode toggle always resets log timestamp** (lines 499–510): Tapping the logged-in username 5 times toggles developer mode and unconditionally calls `setLogEnabledAt(DateTime.now())`. This fires on both enable and disable, meaning the log timestamp is reset even when developer mode is being turned off.

5. **`_showSyncLocationsDialog` race condition** (lines 667–689): The sync dialog is shown with an unawaited `showDialog`, then the sync-already-running guard is checked. If a sync starts between the dialog appearing and the guard check, the dialog is already visible but the guard correctly prevents a second `runSync` call. In practice this window is negligible, but the ordering is fragile.

6. **No confirmation on "Clear cached maps"** (lines 483–494): Tapping "Clear cached maps" immediately calls `MaptileHelper.resetCaches()` with no confirmation dialog, unlike "Clear and redownload" which requires confirmation.

7. **`_showConfirmSwitchApiDialog` parameter naming** (lines 898–931): The parameter is named `isProduction` but receives the new toggle value for `useStagingApi`. When staging is being enabled `isProduction = true` → dialog shows "Switch to Staging API", which is functionally correct but semantically backwards. The parameter name implies the current state, not the new value being applied.

8. **Massive, structurally flat file**: At ~946 lines, this is the largest and most complex screen in the app. There is an acknowledged `TODO` at the top of the file (`///TODD: Move some of the logic into separate functions/files to clean up the widget tree`) and a second one inline in the location sharing handler (`//TODO: Move this out to a separate function once we have tidied and tested as this is very hard to read here`). Specific problems and what is needed to fix them:

   - **Async workflows inline in `build`**: The `onChanged` callbacks for location sharing (~80 lines) and background updates (~35 lines) contain full async business logic directly inside the widget tree. These should be extracted to named methods on the state class (e.g. `_onLocationSharingChanged`, `_onBackgroundUpdatesChanged`).

   - **All section state in one class**: Every loading flag, preference value, and local state variable for all five sections (Friends, Preferences, Notifications, Offline Storage, Account) lives in `_SettingsScreenState`. Extracting each section to its own `ConsumerStatefulWidget` would give each section its own isolated state, making changes to one section impossible to accidentally affect another.

   - **8 dialog methods in one file**: `_showFriendsMissingDetailsDialog`, `_showClearLocationsDialog`, `_showSyncLocationsDialog`, `_showLocationPermissionDialog`, `_showReportsDefaultDisplayDialog`, `_showReportsDefaultSortDialog`, `_showFriendNameDialog`, `_showConfirmSwitchApiDialog` are all defined inline. These could move to dedicated dialog widget files or at minimum be grouped and extracted to a separate dialogs helper/mixin.

   - **`build()` returns a single flat `ListView` with ~80 direct children**: All setting rows, spacers, headers, and `Consumer` wrappers are siblings in one list. Grouping them into section builder methods (e.g. `_buildFriendsSection()`, `_buildPreferencesSection()`) would make the structure scannable and reduce the chance of misplacing a widget during edits.

9. **`startAsBackground: false` is an intentional workaround for a stuck-dialog bug**: Without the explicit flag, `_isBackgroundSync` is derived automatically as `lastSuccessfulSync != null`. For returning users this evaluates to `true`, treating the settings-triggered sync as a background sync. `SyncLocationsDialog` only dismisses itself when `isError && !isBackgroundError` — so with `_isBackgroundSync = true`, a terminal error produces `isBackgroundError = true`, the dialog condition fails, and the dialog stays open. HomeScreen's listener then shows `SyncFailedModal` via `OneContext`, but `SyncLocationsDialog` is still visible behind it, leaving the user stuck.

   Passing `startAsBackground: false` forces `_isBackgroundSync = false`, so on terminal error `isBackgroundError = false`, `SyncLocationsDialog` pops itself cleanly before showing `SyncFailedModal`. This is correct for the settings context where the dialog is always visible.

10. **Logout does not end the OAuth session**: Tapping Logout calls `authenticationProvider.logout()`, which clears local tokens, wipes Hive data, revokes the FCM token, and stops position broadcasting. The local app state is fully reset. However, the device browser retains the OAuth server's session cookie. When the user taps LOGIN again, the system browser opens the OAuth flow and auto-authenticates using the cached cookie — the user is never prompted for credentials and is silently logged back into the same account.

    There is a `//TODO` in `auth_provider.dart` acknowledging this: *"Need an API endpoint that causes logout globally as at the moment this doesn't work as browser app caches cookies"*. The fix requires a server-side logout endpoint that invalidates the OAuth session, which the app would call during `logout()` before navigating to the auth screen. Until then, logout only clears local state.

11. **`friendSettingProvider` read-then-write pattern** (e.g. lines 166, 306): Several toggle handlers read the current friend settings state before writing back. Between the read and write, another event could modify the state, though this is unlikely in practice given user-driven interactions.