---
id: 712aebff-3a2b-4485-9fc6-6a26ca64a548
blueprint: captains_mate_app_43
title: Providers
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1780581325
---
# TCA Mobile App Providers

## Summary

**Total Providers: 52**

| Type | Count |
|------|-------|
| Provider | 9 |
| StateProvider | 13 |
| FutureProvider | 16 |
| StreamProvider | 1 |
| NotifierProvider (manual) | 1 |
| NotifierProvider (code-generated) | 6 |
| StateNotifierProvider | 1 |

---

## Core Providers

| Provider | Type | Purpose |
|----------|------|---------|
| `kiwiProvider` | `Provider<KiwiConfig>` | Provides Kiwi dependency injection configuration |
| `repositoryManagerProvider` | `Provider<RepositoryManager>` | Provides access to domain repositories |
| `hiveDatabaseProvider` | `Provider<HiveDatabase>` | Provides the Hive database instance |

---

## Data Providers

### Attributes

| Provider | Type | Purpose |
|----------|------|---------|
| `attributeProvider` | `FutureProvider<List<AttributeGroup>>` | Fetches all attribute groups from repository |
| `filterAttributesProvider` | `FutureProvider<List<Attribute>>` | Provides filtered attributes with internal HLR and Discount attributes removed |

### Location

| Provider | Type | Purpose |
|----------|------|---------|
| `locationsProvider` | `FutureProvider<List<Location>>` | Fetches all locations from repository |
| `locationProvider` | `FutureProvider.autoDispose.family<Location, String>` | Fetches a specific location by ID |
| `locationMapSelectedProvider` | `StateProvider<String?>` | Tracks currently selected location on map |
| `locationSearchProvider` | `StateProvider<String>` | Tracks location search text |
| `locationSortProvider` | `StateProvider<LocationSort>` | Tracks location sort order (default: nearest) |
| `locationTypeFiltersProvider` | `StateProvider<List<String>>` | Tracks selected location type filters |
| `locationAttributeFiltersProvider` | `StateProvider<List<String>>` | Tracks selected location attribute filters |
| `locationFiltersSelectedProvider` | `Provider<bool>` | Derived value — true if any location filters are currently active |
| `mapPositionProvider` | `StateProvider<LatLng>` | Tracks map centre position; defaults to CA House fallback |
| `searchedAndFilteredLocationsProvider` | `FutureProvider<List<Location>>` | Provides filtered/sorted locations based on search and filters |
| `createEditLocationProvider` | `StateProvider<CreateEditLocationRequest>` | Tracks location creation/editing form state |
| `locationAttributeProvider` | `FutureProvider<LocationAttributeResponse>` | Fetches location attributes from API |

### Position & Maps

| Provider | Type | Purpose |
|----------|------|---------|
| `currentPositionProvider` | `StateProvider<LatLng?>` | Tracks current user GPS position |
| `sortPositionProvider` | `StateProvider<LatLng?>` | Position used for sorting the location list by distance; updated when sort is triggered |
| `positionProvider` | `StreamProvider<LatLng>` | Streams position updates from device location services |
| `positionBroadcastingProvider` | `Provider<PositionBroadcastingService>` | Service that manages starting/stopping position broadcasting to the friends API |
| `maptileProvider` | `Provider<MaptileHelper>` | Provides map tile operations helper |
| `savedMapSectionProvider` | `FutureProvider<List<SavedMapSection>>` | Fetches all saved map sections |

### Members & Boats

| Provider | Type | Purpose |
|----------|------|---------|
| `memberListProvider` | `FutureProvider.autoDispose.family<List, SearchFilterOptions>` | Fetches and filters members or boats based on search options |
| `selectedMemberProvider` | `FutureProvider.autoDispose.family<Member?, String>` | Fetches a specific member by ID |
| `selectedBoatProvider` | `FutureProvider.autoDispose.family<Boat?, String>` | Fetches a specific boat by ID |

### Friends

| Provider | Type | Purpose |
|----------|------|---------|
| `friendsProvider` | `StateProvider.autoDispose<List<Friend>>` | Tracks list of friends |
| `friendSettingProvider` | `StateNotifierProvider<_FriendSettingNotifier, _FriendSettingState>` | Manages friend settings (position sharing, email prefs, background updates) |

### Notifications

| Provider | Type | Purpose |
|----------|------|---------|
| `pendingNotificationProvider` | `NotifierProvider<PendingNotificationNotifier, NotificationData?>` | Holds a pending notification that should be acted on when the app returns to foreground; cleared after handling |
| `hasSeenNotificationModal` | `StateProvider<bool>` | Tracks if the notification permission modal has been seen this session |

### Sync

| Provider | Type | Purpose |
|----------|------|---------|
| `syncLocationCacheProvider` | `StateProvider<List<Location>?>` | Caches the full location list during a sync operation to avoid redundant fetches |

### Permissions

| Provider | Type | Purpose |
|----------|------|---------|
| `permissionsProvider` | `FutureProvider<PermissionStatus>` | Checks current location permission status |

### UI State

| Provider | Type | Purpose |
|----------|------|---------|
| `showFilterProvider` | `StateProvider<bool>` | Tracks whether the filter panel is visible |
| `homeDrawerProvider` | `StateProvider<bool>` | Tracks home drawer open/close state |
| `homeScreenLoading` | `StateProvider<bool>` | Tracks home screen loading state |
| `homeSelectedTab` | `StateProvider<HomeScreenTabOption>` | Tracks currently selected home screen tab |

### Storage & Preferences

| Provider | Type | Purpose |
|----------|------|---------|
| `sharedPreferencesProvider` | `Provider<SharedPreferencesHelper>` | Provides shared preferences access |

### Other Data

| Provider | Type | Purpose |
|----------|------|---------|
| `typeOptionProvider` | `FutureProvider<List<TypeOption>>` | Fetches all type options |
| `filterTypeOptionProvider` | `FutureProvider<List<TypeOption>>` | Provides filtered type options for filter UI |
| `hlrProvider` | `FutureProvider<List<Hlr>>` | Fetches HLRs with centroid or primaryLocationId |
| `overviewSectionProvider` | `FutureProvider<List<OverviewSection>>` | Fetches all overview sections |
| `draftLocationProvider` | `FutureProvider<List<CreateEditLocationRequest>>` | Fetches all draft locations |
| `draftReportProvider` | `FutureProvider.autoDispose<List<CreateReportRequest>>` | Fetches all draft reports |
| `refreshProvider` | `Provider<_RefreshInternal>` | Refreshes multiple providers at once |

---

## Feature Providers (Code-Generated)

These providers use the `@Riverpod` / `@riverpod` annotation and are code-generated via `build_runner`.

| Provider | Annotation | Purpose |
|----------|------------|---------|
| `authenticationProvider` | `@Riverpod(keepAlive: true)` | Manages authentication state (login, logout, token refresh, OAuth flow) |
| `notificationProvider` | `@Riverpod(keepAlive: true)` | Manages notifications (fetch, mark as read) |
| `locationListProvider` | `@Riverpod(keepAlive: true)` | Manages locations (create, update, mark visited) |
| `reportListProvider` | `@Riverpod(keepAlive: true)` | Manages reports (create, update, delete) |
| `syncNotifierProvider` | `@Riverpod(keepAlive: true)` | Manages sync operations (run, cancel, check connection) |
| `createEditReportProvider` | `@riverpod` | Manages create/edit report form state and submission |

---

## File Locations

| Directory | Description |
|-----------|-------------|
| `lib/core/providers/` | Core infrastructure providers |
| `lib/src/data/providers/` | Data layer providers |
| `lib/src/features/*/providers/` | Feature-specific providers |
| `lib/generated/*/providers/` | Generated provider files (gitignored) |