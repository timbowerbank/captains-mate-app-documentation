---
id: cb9a4d7e-458e-4fd5-982e-6952b5ea3132
blueprint: captains_mate_app_43
title: 'Set Location Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535248
---
# SetLocationScreen

**File:** `lib/src/screens/create_edit_location/set_location_screen.dart`
**Route:** `/create_location/set_location`

---

## Purpose

The map-based pin-placement step in the create/edit location wizard. Users position a location by dragging the map (a red pin marker is fixed at the centre of the viewport) or by manually entering DMS coordinates via a bottom-sheet form. Before confirming, the screen checks whether any existing locations are within a configurable proximity radius and warns the user with a dialog listing nearby locations. On confirmation the selected position is written to `createEditLocationProvider` and the screen either pushes to `ContentScreen` (new location) or pops back (editing an existing location's pin).

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    ├── Title: "CREATE LOCATION" or "EDIT LOCATION" (based on shouldPop)
 │    ├── automaticallyImplyLeading: false
 │    └── Action: IconButton (close ✕)
 │         └── Resets createEditLocationProvider if creating, then pop
 │
 ├── extendBodyBehindAppBar: true
 ├── resizeToAvoidBottomInset: false
 │
 └── body: Stack
      │
      ├── Positioned (top:0 → bottom: sheetHeight - marginLarge)
      │    └── Stack
      │         ├── TcaAsyncDataWrapper<List<Location>>
      │         │    └── TcaMap
      │         │         ├── controller: _mapController
      │         │         ├── currentPosition (user's GPS, if permission granted)
      │         │         ├── center: args.position (if editing)
      │         │         └── markers: List<TcaLocationMarker> (all existing locations)
      │         │
      │         └── Align (center)
      │              └── SvgPicture (red pin marker, fixed at map centre)
      │
      ├── Align (bottomCenter)
      │    └── _buildBottomSheet() → TcaBottomSheet
      │         ├── Text ("Set Pin Location")
      │         ├── Container (green info box)
      │         │    └── "Drag the map or edit coordinates"
      │         ├── Row
      │         │    ├── TcaFormField ("Latitude", read-only DMS, tappable)
      │         │    └── TcaFormField ("Longitude", read-only DMS, tappable)
      │         │         └── onTap → _openLocationSelector() (coordinate editor)
      │         └── TcaButton ("SET LOCATION")
      │              └── _setLocation() → nearby check → confirm → write provider
      │
      └── TcaBackgroundGradient (top fade for app bar readability)

─── Coordinate Editor Bottom Sheet ───

showModalBottomSheet (isScrollControlled)
 └── TcaBottomSheet > StatefulBuilder > Form > Column
      ├── "Latitude" header
      │    └── Row: Degrees (number) │ Mins Dec (decimal) │ N/S (select)
      ├── "Longitude" header
      │    └── Row: Degrees (number) │ Mins Dec (decimal) │ E/W (select)
      ├── TcaButton ("SAVE COORDINATES")
      │    └── Validates → converts DMS → moves map → updates display → pop
      └── TcaButton ("CANCEL", outlined) → pop

─── Nearby Locations Dialog ───

TcaDialog
 ├── RichText ("Your chosen point is very close to {location names}")
 │    └── Location names are tappable → navigate to ViewLocationScreen
 ├── TcaButton ("CANCEL") → dismiss
 └── TcaButton ("CONTINUE") → proceed with set location
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `createEditLocationProvider` | `lib/src/data/providers/create_edit_location.dart` | `StateProvider<CreateEditLocationRequest>` — the shared wizard state. This screen writes the confirmed `position` (LatLng) via `copyWith`. On close (new location), it is reset to empty |
| `locationsProvider` | `lib/src/data/providers/location.dart` | `FutureProvider<List<Location>>` — loaded via `TcaAsyncDataWrapper` to render existing location markers on the map, and read directly in `_setLocation()` to check for nearby locations within `Constants.nearbyCheckDistance` |

### Models

| Model | File | Role |
|---|---|---|
| `SetLocationScreenArguments` | (defined in-file, line 34) | Route args with `shouldPop` (bool — if true, pops instead of pushing to ContentScreen) and `position` (LatLng? — initial map centre for editing) |
| `CreateEditLocationRequest` | `lib/src/features/locations/models/create_edit_location_request.dart` | Freezed wizard state model. This screen writes `position` field |
| `Location` | `lib/src/features/locations/models/location.dart` | Used for rendering existing markers and checking proximity (`l.position.value`) |
| `SelectedLocationRoute` | `lib/src/helpers/route_argumnets/selected_location_route.dart` | Route args passed when navigating to `ViewLocationScreen` from the nearby-locations dialog |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaMap` | `lib/src/widgets/tca_map.dart` | flutter_map wrapper with tile layers and current-position marker. Controlled via `MapController` |
| `TcaLocationMarker` | `lib/src/widgets/tca_location_marker.dart` | Custom marker widget per existing location, placed on the map |
| `TcaAsyncDataWrapper` | `lib/src/widgets/tca_async_data_wrapper.dart` | Handles loading/error/data states for `locationsProvider` |
| `TcaBottomSheet` | `lib/src/widgets/tca_bottom_sheet.dart` | Styled bottom sheet container used for both the fixed coordinate display and the pop-up coordinate editor |
| `TcaBackgroundGradient` | `lib/src/widgets/tca_background_gradient.dart` | Semi-transparent gradient overlay at the top, ensuring the app bar text is readable over the map |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Used in multiple modes: read-only (lat/lng display), number (degrees/minutes input), and select (N/S/E/W direction picker) |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used for "SET LOCATION", "SAVE COORDINATES", and "CANCEL" actions |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Confirmation dialog for nearby-location warnings |

### Helpers

| Helper | File | Role |
|---|---|---|
| `PositionHelper` | `lib/src/helpers/position.dart` | `checkPermissions()` — checks location permission; `getCurrentPosition()` → user's LatLng; `fallbackPosition` → default LatLng; `latitudeToDms()` / `longitudeToDms()` → format to DMS string; `dmsToDegree()` → parse DMS back to decimal |
| `Distance` | `lib/src/helpers/distance.dart` | `distanceInNM()` — calculates nautical-mile distance between two LatLng points, used for the nearby-location proximity check |

### Constants

| Constant | File | Role |
|---|---|---|
| `Constants.defaultMapZoom` | `lib/src/values/constants.dart` | Initial zoom level when centering the map on user position or edited coordinates |
| `Constants.nearbyCheckDistance` | `lib/src/values/constants.dart` | Nautical-mile radius threshold for the nearby-location warning |

---

## API / Backend Dependencies

None — this screen operates entirely on local state and locally-cached location data. No network calls are made. The actual API submission happens later in `ProgressScreen`.

---

## Data Flow

### Initial Map Positioning

```
initState
  │
  ├─ Listen to _mapController.mapEventStream
  │    └─ On every map move (except programmatic):
  │         update _selectedLatitude / _selectedLongitude DMS text
  │
  └─ Post-frame callback:
       ├─ Measure bottom sheet height → _sheetHeight (for map sizing)
       │
       ├─ PositionHelper.checkPermissions()
       │    ├─ Has permission → getCurrentPosition() → user's LatLng
       │    └─ No permission → fallbackPosition
       │
       └─ Move map to args.position (if editing) or user position
            └─ Set _selectedLatitude / _selectedLongitude DMS text
```

### Manual Coordinate Entry

```
User taps Latitude or Longitude field
  │
  └─ _openLocationSelector() → showModalBottomSheet
       │
       ├─ Parse current DMS into Degrees / Mins / Direction fields
       │    └─ Regex: "^(.*)° (.*)' (.*)$"
       │
       ├─ User edits fields (validated: lat ±90, lng ±180, mins 0-60)
       │
       └─ User taps "SAVE COORDINATES"
            ├─ Validate form
            ├─ Format DMS strings
            ├─ PositionHelper.dmsToDegree() → decimal lat/lng
            ├─ _mapController.move(LatLng, defaultZoom)
            ├─ Update display text controllers
            └─ Navigator.pop() → closes bottom sheet
```

### Set Location (with Nearby Check)

```
User taps "SET LOCATION"
  │
  ├─ Read all locations from locationsProvider
  ├─ Filter to those within nearbyCheckDistance NM of map centre
  │
  ├─ If nearby locations found:
  │    └─ Show TcaDialog listing nearby names (tappable → ViewLocationScreen)
  │         ├─ User taps "CANCEL" → abort
  │         └─ User taps "CONTINUE" → proceed ↓
  │
  ├─ Write map centre to createEditLocationProvider.position
  │
  ├─ If shouldPop (editing pin):
  │    └─ Navigator.pop() → back to ContentScreen
  │
  └─ If not shouldPop (new location):
       └─ pushReplacementNamed → ContentScreen
```

### Close Button

```
User taps ✕
  │
  ├─ If not shouldPop (creating): reset createEditLocationProvider to empty
  └─ Navigator.pop()
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| HomeScreen / TcaDrawer | Via "Create Location" action | `SetLocationScreenArguments()` (defaults: `shouldPop: false`, `position: null`) | First step when creating a new location |
| ContentScreen | `lib/src/screens/create_edit_location/content_screen.dart` | `SetLocationScreenArguments(shouldPop: true, position: currentPosition)` | "Set Pin Location" action — edit an already-set pin and pop back |

---

## Known Caveats / Tech-Debt Notes

1. **Route arguments read in `build()`** (lines 133-138): `_shouldPop` and `_position` are reassigned from route arguments on every rebuild. Since route args don't change, this should be done once in `initState` or `didChangeDependencies`.

2. **`_position` is never updated after init** (line 59): The `_position` field is set from route arguments but is only used as the initial map centre. Once the user drags the map, `_position` becomes stale. The actual confirmed position is read from `_mapController.camera.center`, which is correct — but having `_position` as a field is misleading.

3. **`ModalRoute.of(context)` in `initState`** (lines 43-44): `_hlrId` is set by accessing route arguments via `ModalRoute.of(context)!` inside `initState`. This works because `WidgetsBinding.instance.addPostFrameCallback` is used elsewhere, but the direct access in `initState` before the frame is built may be fragile. (Note: this is actually correct for `initState` in this widget since the route is already available, but `didChangeDependencies` is the conventional place.)

4. **`dialogContext` captured via late variable** (line 275): The nearby-locations dialog captures its `BuildContext` into a `late` variable (`dialogContext`) inside the builder, which is then used by `TapGestureRecognizer` callbacks to pop the dialog before navigating. This works but is fragile — if the dialog were to be dismissed by another mechanism, the captured context would be stale.

5. **`TapGestureRecognizer` instances created but never disposed** (lines 278-291): A `Map` of `TapGestureRecognizer` objects is created for tappable location names in the nearby dialog. These are never explicitly disposed, leaking until garbage collected.

6. **Recursive `_setLocation` call** (line 289): When a user taps a nearby location name in the dialog, the code navigates to `ViewLocationScreen` and then calls `_setLocation()` again after returning. This creates a recursive flow where the user could bounce between the dialog and location details indefinitely. While intentional (lets the user check a nearby location and then re-confirm), the recursion could theoretically grow the call stack on repeated use.

7. **No validation that position is within valid bounds**: The map allows free panning to any coordinate. The manual coordinate editor validates ranges (lat ±90, lng ±180), but the drag-to-set path has no equivalent validation. Extreme map positions (e.g. Antarctica) are accepted without warning.

8. **`PanelController()` passed but unused** (lines 191, 416): Fresh `PanelController` instances are passed to both `TcaBottomSheet` usages but are never used for programmatic panel control.

9. **Listener lifecycle in coordinate editor** (lines 420-425, 629-630): The `StatefulBuilder` adds listeners to `_modalLatDirection` and `_modalLngDirection` with a `hasSetListeners` guard to avoid duplicates. The listeners are removed after the bottom sheet closes. This pattern is correct but complex — a simpler approach would be to use `StatefulBuilder`'s `setState` directly in the `onChanged` callbacks.

10. **Eight `TextEditingController` instances** (lines 60-67): The screen maintains eight controllers for the two DMS displays and six coordinate-editor fields. While `_selectedLatitude` and `_selectedLongitude` are properly disposed, all six modal controllers (`_modalLat*`, `_modalLng*`) are also disposed in `dispose()` — this is correct but creates a large controller surface area that could be simplified by scoping the modal controllers to the bottom sheet's lifecycle.