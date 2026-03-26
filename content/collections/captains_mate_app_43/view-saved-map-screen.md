---
id: 432b7440-c4f3-4eb6-8bb3-3c5db5ebf948
blueprint: captains_mate_app_43
title: 'View Saved Map Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535722
---
# ViewSavedMapScreen

**File:** `lib/src/screens/settings/view_saved_map_screen.dart`
**Route:** `/settings/view_saved_map`

---

## Purpose

A read-only map preview of a previously saved offline map area. The map is centred on the saved section's bounds and displayed at the section's minimum zoom level. Map interaction is disabled (`isDisabled: true`) — the user can only view the area, then tap "DONE" to return to the `ManageSavedMapsScreen`. The `SavedMapSection` object is passed directly as the route argument (not looked up by ID).

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    └── Text (section.name or "View Saved Map")
 │
 ├── extendBodyBehindAppBar: true
 │
 └── body: Stack
      │
      ├── TcaMap
      │    ├── controller: _mapController
      │    └── isDisabled: true (no pan/zoom interaction)
      │
      ├── TcaBackgroundGradient (top fade for app bar readability)
      │
      └── Align (bottomCenter)
           └── TcaButton ("DONE")
                └── Navigator.popUntil(ManageSavedMapsScreen.routeName)
```

---

## Associated

### Providers

None — this is a purely presentational screen with no Riverpod providers.

### Models

| Model | File | Role |
|---|---|---|
| `SavedMapSection` | `lib/src/features/maptiles/models/saved_map_section.dart` | Passed directly as the route argument. Fields used: `name` (app bar title), `topLeft` / `bottomRight` (LatLng bounds for centroid calculation), `minZoom` (initial zoom level) |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaMap` | `lib/src/widgets/tca_map.dart` | flutter_map wrapper, rendered in disabled mode (no gesture interaction) |
| `TcaBackgroundGradient` | `lib/src/widgets/tca_background_gradient.dart` | Semi-transparent gradient at the top for app bar readability over the map |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | "DONE" button that pops back to the manage screen |

### Helpers

| Helper | File | Role |
|---|---|---|
| `Distance` | `lib/src/helpers/distance.dart` | `computeCentroid()` — calculates the geographic centre point between `topLeft` and `bottomRight` bounds, used to position the map |

---

## API / Backend Dependencies

None — this screen is entirely local and presentational. No network calls are made.

---

## Data Flow

### Initialisation

```
initState
  │
  └─ Post-frame callback:
       │
       ├─ Read route args → SavedMapSection
       │
       └─ setState(_section = args)
            │
            └─ Nested post-frame callback:
                 └─ _mapController.move(
                      Distance.computeCentroid([topLeft, bottomRight]),
                      section.minZoom.toDouble()
                    )
```

### Dismiss

```
User taps "DONE"
  │
  └─ Navigator.popUntil(
       ModalRoute.withName(ManageSavedMapsScreen.routeName)
     )
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| ManageSavedMapsScreen | `lib/src/screens/settings/manage_saved_maps_screen.dart` | `SavedMapSection` (passed directly as route argument) | *(Note: this route is defined but there is no visible navigation call to it in `ManageSavedMapsScreen` — tapping a section navigates to HomeScreen instead. This screen may be unused or reached from elsewhere.)* |

---

## Known Caveats / Tech-Debt Notes

1. **Nested `addPostFrameCallback`** (lines 27-42): The initialisation uses a post-frame callback that calls `setState`, which triggers a rebuild, and then schedules *another* post-frame callback to move the map. This double-deferred pattern is used because the `MapController.move()` requires the map widget to be laid out first. While functional, it adds two frames of delay before the map is correctly positioned — the map briefly shows its default position before snapping to the saved area.

2. **Route argument is a full model, not an ID** (line 29): The `SavedMapSection` object is passed directly as the route argument rather than an ID that would be looked up from a provider. This means the screen shows the data as it was at the time of navigation, not the latest from Hive. If the section were renamed or updated concurrently, this screen would show stale data.

3. **`_section` is nullable with conditional rendering** (line 52): The app bar title uses `_section?.name ?? 'View Saved Map'` as a fallback, meaning the title briefly shows "View Saved Map" during the first frame before the post-frame callback sets `_section`.

4. **Possibly unused screen**: `ManageSavedMapsScreen` navigates to `HomeScreen` when tapping a saved map section, not to this screen. No other navigation call to `ViewSavedMapScreen.routeName` was found in the codebase. This screen may be a remnant from an earlier design, or may be intended for future use.

5. **`MapController` not disposed** (line 21): The `MapController` is created as an instance field but is never disposed. While `MapController` from flutter_map does not currently require disposal, this is inconsistent with the pattern used in other map screens.

6. **`popUntil` assumes `ManageSavedMapsScreen` is in the stack** (line 70-73): The "DONE" button pops until `ManageSavedMapsScreen.routeName` is found. If this screen were ever navigated to from a different route (bypassing the manage screen), the `popUntil` would pop the entire stack, potentially reaching the root.