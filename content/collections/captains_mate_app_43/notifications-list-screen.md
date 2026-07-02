---
id: 2c803d17-c35b-44c7-bd17-9195adf49cb3
blueprint: captains_mate_app_43
title: 'Notifications List Screen'
use_synced_content: false
parent: faa4011a-a306-467e-ac40-635e775f6e76
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780407483
---
# NotificationsListScreen
**File:** `lib/src/features/notifications/screens/notifications_list_screen.dart`
**Route:** `/notifications`

## Purpose

Displays the user's received notifications in two tabs — unread and read. The screen polls local Hive storage every 10 seconds for new notifications, and also supports pull-to-refresh and text search. Tapping a notification marks it as read and navigates to the relevant content based on the notification's `action` and `path` fields.

The screen can be opened with a specific `notificationId` in the route arguments, in which case that notification is visually highlighted in the unread tab.

Route name: `/notifications`

---

## UI Structure (Simplified Tree)

```
TcaSafeScaffold
├── AppBar
│    └── Text ("UNREAD NOTIFICATIONS" / "READ NOTIFICATIONS")
│
└── body: Column
     ├── TcaSearchBar
     │    └── onSearch → filters notification list
     │
     ├── TcaSegmentedControl (UNREAD | READ)
     │    └── onValueChanged → clears search, switches tab, reloads
     │
     └── Expanded
          └── NotificationList
               ├── if _isLoading → CircularProgressIndicator
               │
               ├── if empty → RefreshIndicator
               │                └── ListView with empty state message
               │
               └── else → RefreshIndicator
                           └── ListView.builder
                                └── NotificationItem (per notification)
                                     ├── title, body, createdAt date
                                     └── onTap → mark as read + reload + navigate
```

---

## Associated Providers

| Provider | Type | Purpose |
|---|---|---|
| `notificationProvider` | `StateNotifierProvider` | Wraps `NotificationRepository`. Called for `getNotifications()` (filtered, sorted) and `markAsRead()` |

---

## Associated Models

| Model | Key Fields | Source |
|---|---|---|
| `NotificationData` | `id`, `title`, `body`, `action`, `path`, `resourceId`, `readAt`, `createdAt` | `lib/src/features/notifications/models/notification_data.dart` |
| `NotificationsListScreenRoute` | `notificationId?` | Defined in-file. Carries an optional notification ID to highlight on arrival |
| `NotificationFilterOptions` | `isUnreadType`, `searchValue?` | Defined in `notification_list_provider.dart`. Passed to `getNotifications()` on every load |

---

## Associated Widgets (Bespoke)

| Widget | File | Role |
|---|---|---|
| `TcaSearchBar` | `lib/src/widgets/tca_search_bar.dart` | Text search input. Triggers reload on value change |
| `TcaSegmentedControl` | `lib/src/widgets/tca_segmented_control.dart` | Unread/Read tab switcher |
| `NotificationList` | `lib/src/features/notifications/widgets/notification_list.dart` | Renders the loading, empty, and populated states; wraps list in `RefreshIndicator` |
| `NotificationItem` | `lib/src/features/notifications/widgets/notification_item.dart` | Individual notification row — title, body, date, focus highlight. Owns tap handling: marks as read, triggers reload callback, then navigates |
| `TcaListItem` | `lib/src/widgets/tca_list_item.dart` | Card wrapper used by `NotificationItem` |

---

## API / Backend Dependencies

None. All data is read from local Hive storage via `NotificationRepository`. Navigation targets (locations, boats, members) are also sourced locally.

External URLs are opened via `LauncherHelper.launch()`.

---

## Data Flow

### Startup

On first render, the screen reads the `notificationId` to highlight from route arguments, falling back to `pendingNotificationProvider` if no argument is set. It then calls `_loadNotifications()`, clears `pendingNotificationProvider` (preserving it if the screen unmounts mid-load), and starts the 10-second polling timer.

### Loading Notifications

`_loadNotifications()` is called on startup, on every timer tick, on pull-to-refresh, on search input change, and on tab switch. It:

1. Sets `_isLoading = true`
2. Calls `notificationProvider.getNotifications(NotificationFilterOptions(isUnreadType, searchValue))`
3. Results are sorted newest-first by `createdAt` inside the provider
4. Updates local `_notifications` list and sets `_isLoading = false`

### Tab Switching

Switching tabs clears the search text and search controller, flips `_isUnreadType`, and calls `_loadNotifications()`. The focused `notificationId` is only applied on the unread tab — switching to read clears the highlight.

### Notification Tap

Tap handling lives in `NotificationItem`, not the screen. When tapped:

1. If unread, `notificationProvider.markAsRead(notification)` is called (awaited), setting `readAt` to now
2. `onNotificationTapped` callback fires — this is `_loadNotifications` passed down from the screen, refreshing the list
3. `_navigate()` runs regardless of read state, based on `action` and `path`:

| Action | Path | Destination |
|---|---|---|
| `path` | `location` | `ViewLocationScreen` (tab 1) |
| `path` | `report` | `ViewLocationScreen` (tab 2) |
| `path` | `boat` | `ViewBoatScreen` |
| `path` | `member` | `ViewMemberScreen` |
| `externalUrl` | URL | Opens external browser via `LauncherHelper` |

Navigation only occurs if both `path` and `resourceId` are present for `path` actions.

### Cleanup

On `dispose`, the polling timer is cancelled and the search controller is disposed.

---

## Known Caveats / Tech Debt

1. **Data held in local state rather than `ref.watch`** — Notifications are loaded into `_notifications` via `_loadNotifications()` and stored as local widget state. This is the same anti-pattern noted in `MyDraftsScreen` — reactive updates via `ref.watch` would remove the need for manual polling.

2. **10-second polling of local Hive storage** — The timer polls local storage rather than a remote API, so the overhead is low, but polling is unnecessary if the screen used a reactive provider. The poll exists to pick up notifications delivered while the screen is open. Polling ticks pass `showLoading: false` so no spinner is shown.

3. **`dispose()` does not check `_refreshTimer` for null before cancel** — `_refreshTimer?.cancel()` uses null-safe access correctly, but the timer is always started in `initState` so this is safe in practice.

4. **`notificationProvider` holds a repository as state, not notification data** — The `Notification` notifier's `build()` returns a `NotificationRepository` instance, making the repository itself the Riverpod state:

   ```dart
   @Riverpod(keepAlive: true)
   class Notification extends _$Notification {
     @override
     NotificationRepository build() => NotificationRepository(); // state = repository
   ```

   This misuses Riverpod in two ways. First, state should represent UI-observable data (e.g. `List<NotificationData>`) so that widgets can `ref.watch` it and rebuild automatically when notifications change. With a repository as state, nothing ever notifies Riverpod that the underlying data has changed, so reactive rebuilds never happen. Second, `NotificationRepository` is already a singleton — storing it as state adds no value; it can be called directly from anywhere.

   The downstream consequence is that `NotificationsListScreen` cannot use `ref.watch` and instead resorts to manual `_loadNotifications()` calls and a 10-second polling timer to stay up to date. If the state held `List<NotificationData>` and the notifier updated it on save/markAsRead, the screen could watch the provider and remove the polling entirely.

5. **No empty state distinction between "no results" and "no notifications"** — If the search returns zero results, the same "No unread notifications" / "No read notifications" message is shown as when there are genuinely no notifications, giving the user no indication that their search filtered everything out.