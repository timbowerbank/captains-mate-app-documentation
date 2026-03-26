---
id: 16018cda-a40e-45ec-b30d-d99239a43e75
blueprint: captains_mate_app_43
title: 'View Location Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774534902
---
# ViewLocationScreen

## Purpose

`ViewLocationScreen` is the primary detail view for displaying a single marine location. It presents comprehensive location information through a tabbed interface with three sections: **Overview**, **Info**, and **Reports**. The screen supports:

- Viewing location details including images, coordinates, and metadata
- Editing locations (if `location.isEditable` is true)
- Marking locations as visited
- Navigating to the location on the map
- Real-time updates when location data changes elsewhere in the app
- Handling deleted/updated resources with modal notifications

**Route:** `/location`

**Navigation:** This screen can be accessed from multiple entry points:

| Source | File | Context |
|--------|------|---------|
| Map popup | [tca_map.dart:417](lib/src/widgets/tca_map.dart#L417) | "VIEW LOCATION" button in marker popup |
| Location list | [tca_location_sheet.dart:160](lib/src/widgets/tca_location_sheet.dart#L160) | Tapping a location in the sliding panel list |
| Related locations | [tca_location_overview.dart:320](lib/src/widgets/tca_location_overview.dart#L320) | "Related Marine Locations" section within another location |
| HLR detail | [view_hlr_screen.dart:337](lib/src/screens/view_hlr_screen.dart#L337) | "Related Marine Locations" section in HLR view |
| Notifications | [notifications_list_screen.dart:333](lib/src/features/notifications/screens/notifications_list_screen.dart#L333) | Tapping a location-related notification |
| After create/edit location | [progress_screen.dart:250](lib/src/screens/create_edit_location/progress_screen.dart#L250) | Redirect after successfully creating/editing |
| After create/edit report | [create_edit_report_screen.dart:426](lib/src/screens/create_edit_report_screen.dart#L426) | Redirect after successfully creating/editing a report |
| Set location | [set_location_screen.dart:286](lib/src/screens/create_edit_location/set_location_screen.dart#L286) | After setting location coordinates |

---

## UI Structure

```
ViewLocationScreen (ConsumerStatefulWidget)
├── Scaffold
│   ├── AppBar (PreferredSize)
│   │   ├── BackButton
│   │   ├── Location Name (Text)
│   │   └── Location Types (Text)
│   │
│   └── NestedScrollView
│       ├── SliverAppBar (collapsible header)
│       │   ├── FlexibleSpaceBar
│       │   │   ├── CachedNetworkImage / Placeholder Image
│       │   │   ├── DMS Coordinates (Text)
│       │   │   ├── "EDIT LOCATION" Button (conditional)
│       │   │   ├── VisitedSectionWidget
│       │   │   └── "VIEW ON MAP" Button
│       │   │
│       │   └── TabBar (Overview | Info | Reports)
│       │
│       └── TabBarView
│           ├── TcaLocationOverview (Tab 0)
│           ├── TcaLocationInfo (Tab 1)
│           └── TcaLocationReports (Tab 2)
│
└── Modals (shown conditionally)
    ├── TcaResourceUpdatedModal
    └── TcaResourceDeletedModal
```

---

## Associated Components

### Providers

| Provider | Type | Purpose |
|----------|------|---------|
| `locationsProvider` | `FutureProvider<List<Location>>` | Fetches all locations from repository |
| `locationProvider` | `FutureProvider.family<Location, String>` | Fetches single location by ID |
| `filterTypeOptionProvider` | `FutureProvider<List<TypeOption>>` | Provides location type metadata for display |
| `createEditLocationProvider` | `StateProvider<CreateEditLocationRequest>` | Holds state for location editing flow |
| `locationMapSelectedProvider` | `StateProvider<String?>` | Tracks which location is selected on map |
| `locationListProvider` | `StateNotifierProvider` | Used by `VisitedSectionWidget` for marking visited |

### Models

| Model | Source | Purpose |
|-------|--------|---------|
| `Location` | `features/locations/models/location.dart` | Core location data model |
| `LocationImage` | `features/locations/models/location_image.dart` | Image with center-of-interest alignment |
| `TypeOption` | `features/meta/models/type_option.dart` | Location type metadata |
| `CreateEditLocationRequest` | `features/locations/models/` | Request model for editing |
| `SelectedLocationRoute` | `helpers/route_arguments/` | Route arguments containing location ID and tab index |
| `HomeScreenRoute` | `helpers/route_arguments/` | Route arguments for navigating to home |

### Widgets (Bespoke)

| Widget | Location | Purpose |
|--------|----------|---------|
| `TcaLocationOverview` | `widgets/tca_location_overview.dart` | Overview tab content - sections, gallery, HLRs, discounts, nearby locations |
| `TcaLocationInfo` | `widgets/tca_location_info.dart` | Info tab content - expandable attribute cards grouped by category |
| `TcaLocationReports` | `widgets/tca_location_reports.dart` | Reports tab content - sortable/filterable user reports with comments |
| `VisitedSectionWidget` | `features/locations/widgets/` | Button to mark location as visited with confirmation dialog |
| `TcaResourceUpdatedModal` | `widgets/modals/` | Modal shown when location data has been updated |
| `TcaResourceDeletedModal` | `widgets/modals/` | Modal shown when location has been deleted |
| `TcaButton` | `widgets/tca_button.dart` | Standard app button with variants |

### Helpers

| Helper | Location | Purpose |
|--------|----------|---------|
| `PositionHelper` | `helpers/position.dart` | Converts `LatLng` to DMS (degrees/minutes/seconds) format for display |
| `TcaLog` | `helpers/log.dart` | Centralised error logging (used in image fetch catch block) |

### Enums

| Enum | Location | Purpose |
|------|----------|---------|
| `LocationTab` | `enums/location_tab.dart` | Tab definitions (Overview, Info, Reports) |
| `TcaButtonVariant` | `enums/button_variant.dart` | Button styling variants (white, white_outlined, etc.) |

### Values / Constants

| Value | Location | Purpose |
|-------|----------|---------|
| `AppColors` | `values/colors.dart` | App colour palette |
| `Dimens` | `values/dimens.dart` | Spacing and sizing constants |
| `Styles` | `values/styles.dart` | Text styles (`locationTitle`, `locationSubtitle`, `locationPositionDMS`, etc.) |
| `Assets` | `values/assets.dart` | Asset paths (placeholder images) |

---

## External Dependencies

| Package | Usage |
|---------|-------|
| `flutter_riverpod` | State management - `ConsumerStatefulWidget`, `ref.read()`, `ref.listen()` |
| `cached_network_image` | Efficient image loading with caching for location header image |
| `collection` | `firstWhereOrNull` - returns `null` instead of throwing when location not found |

---

## API / Backend Dependencies

| Endpoint | Method | Usage |
|----------|--------|-------|
| `ApiClient.location.getLocationImages(locationId)` | GET | Fetches high-resolution images separately after initial load |
| `LocationRepository.getAll()` | GET | Fetches all locations (via `locationsProvider`) |
| `locationListProvider.markLocationAsVisited()` | PUT | Marks location as visited by current user |

**Note:** The screen loads location data from the cached `locationsProvider` first, then separately fetches images via API to avoid blocking the initial render.

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INITIALIZATION                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Route Arguments (SelectedLocationRoute)                          │
│        │                                                             │
│        ▼                                                             │
│  2. _loadLocation()                                                  │
│        │                                                             │
│        ├──► locationsProvider ──► Find location by ID                │
│        │         │                                                   │
│        │         ▼                                                   │
│        │    Location not found? ──► TcaResourceDeletedModal ──► Pop  │
│        │         │                                                   │
│        │         ▼                                                   │
│        └──► filterTypeOptionProvider ──► Get type labels             │
│                  │                                                   │
│                  ▼                                                   │
│             setState() ──► originalLocation, location, _filterTypes  │
│                                                                      │
│  3. ApiClient.location.getLocationImages()                           │
│        │                                                             │
│        ▼                                                             │
│     setState() ──► Update location.images                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      REACTIVE UPDATES                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ref.listen(locationsProvider) ──► _loadLocation()                   │
│                                          │                           │
│                                          ▼                           │
│                        location.isUpdated(originalLocation)?         │
│                                          │                           │
│                                    Yes   │   No                      │
│                                    ▼     │                           │
│                     TcaResourceUpdatedModal                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       USER ACTIONS                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  "EDIT LOCATION" ──► createEditLocationProvider.state = request      │
│         │                                                            │
│         └──► Navigator.pushNamed(ProgressScreen.routeName)           │
│                                                                      │
│  "VIEW ON MAP" ──► locationMapSelectedProvider.state = location.id   │
│         │                                                            │
│         └──► Navigator.pushNamedAndRemoveUntil(HomeScreen)           │
│                                                                      │
│  "VISITED" (VisitedSectionWidget)                                    │
│         │                                                            │
│         └──► locationListProvider.markLocationAsVisited()            │
│                    │                                                 │
│                    └──► onLocationUpdated callback ──► setState()    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Known Caveats / Tech Debt

### 1. Image Loading Strategy
Images are fetched separately after the initial load (`getLocationImages`), which can cause a visual "pop" when images appear. The images are merged into the location state via `copyWith`, preserving them across subsequent `_loadLocation()` calls.

```dart
// Refresh location, ensure we keep the images
location = _location.copyWith(images: location?.images ?? []);
```

### 2. Location ID Extracted in `build()`
The `_locationId` is extracted from route arguments inside the `build()` method rather than in `initState()`. This works but is unconventional:

```dart
@override
Widget build(BuildContext context) {
  _locationId = (ModalRoute.of(context)!.settings.arguments as SelectedLocationRoute).id;
  // ...
}
```

### 3. Modal Shown via Build Method Side-Effect
The `TcaResourceUpdatedModal` is triggered during `build()` via a flag check, which is an anti-pattern. A post-frame callback is used to mitigate this:

```dart
if (isLocationUpdated) _showLocationUpdatedModal(context);
```

### 4. originalLocation State Management
The `originalLocation` variable is used to detect changes for showing the update modal. It's only set once (first load) and never reset, which means:
- If the user views the location, sees an update modal, and the location updates again, no second modal will appear
- The `_hasShownUpdateModal` flag prevents duplicate modals but persists for the widget lifetime

### 5. Tab Controller Initial Animation
The tab controller animates to the requested tab index after load, but only if the current index is 0:

```dart
if (_tabController.index == 0)
  _tabController.animateTo(...);
```

### 6. ScrollController Created in `build()`
A new `ScrollController` is created on every build, which is wasteful. Should be moved to state:

```dart
final ScrollController _scrollController = ScrollController(); // In build()
```

### 7. Widget Location (Recommendation)
Consider moving `TcaLocationOverview`, `TcaLocationInfo`, and `TcaLocationReports` from `widgets/` to `features/locations/widgets/`. These widgets are:
- Tightly coupled to the `Location` model
- Only used by `ViewLocationScreen`
- Consistent with existing pattern (`VisitedSectionWidget`, `GalleryInfoWidget` already live in `features/locations/widgets/`)

This would improve feature cohesion and discoverability.

---

## File References

- Screen: [view_location_screen.dart](lib/src/screens/view_location_screen.dart)
- Overview Tab: [tca_location_overview.dart](lib/src/widgets/tca_location_overview.dart)
- Info Tab: [tca_location_info.dart](lib/src/widgets/tca_location_info.dart)
- Reports Tab: [tca_location_reports.dart](lib/src/widgets/tca_location_reports.dart)
- Visited Widget: [visited_section_widget.dart](lib/src/features/locations/widgets/visited_section_widget.dart)
- Location Provider: [location.dart](lib/src/data/providers/location.dart)
- Location Model: [location.dart](lib/src/features/locations/models/location.dart)