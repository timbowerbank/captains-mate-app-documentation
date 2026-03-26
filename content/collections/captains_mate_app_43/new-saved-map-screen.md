---
id: 3c65987a-2b48-4d21-b16b-aca812f74c3c
blueprint: captains_mate_app_43
title: 'New Saved Map Screen'
use_synced_content: false
parent: faa4011a-a306-467e-ac40-635e775f6e76
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774537275
---
# NewSavedMapScreen

**File:** `lib/src/screens/settings/new_saved_map_screen.dart`
**Route:** `/settings/new_saved_map`

---

## Purpose

An interactive map screen for selecting a new area to save for offline use. The user pans and zooms the map to frame the desired area, taps "DONE", enters a name in a dialog, and the visible map bounds are saved as a `SavedMapSection` to Hive. A background tile download is then kicked off via `maptileProvider`. Once the download starts, the screen pops back to `ManageSavedMapsScreen` where the new section appears with a progress indicator. This doesn't appear to be working as intended, see 11. in tech debt list.

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    └── Text ("SELECT MAP AREA")
 │
 ├── extendBodyBehindAppBar: true
 │
 └── body: Stack
      │
      ├── TcaMap
      │    ├── controller: _mapController
      │    └── currentPosition: _currentPosition (user GPS, if permitted)
      │
      ├── TcaBackgroundGradient (top fade for app bar readability)
      │
      └── Align (bottomCenter)
           └── TcaButton ("DONE")
                └── _promptForName()

─── Name Dialog ───

showDialog > Form
 └── TcaDialog
      ├── TcaFormField ("Area Name", with RequiredValidator)
      ├── TcaButton ("SAVE")
      │    └── _saveArea(dialogContext)
      └── TcaButton ("CANCEL", outlined) → pop
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Accesses `SavedMapSectionRepository` to persist the new `SavedMapSection` to Hive |
| `savedMapSectionProvider` | `lib/src/data/providers/saved_map_section.dart` | Invalidated after saving, so `ManageSavedMapsScreen` picks up the new section when the user navigates back |
| `maptileProvider` | `lib/src/data/providers/maptile.dart` | `Provider<MaptileHelper>` — `downloadSection(section)` is called to begin background tile downloading for the selected area |

### Models

| Model | File | Role |
|---|---|---|
| `SavedMapSection` | `lib/src/features/maptiles/models/saved_map_section.dart` | Created via `SavedMapSection.create()` factory with name, topLeft/bottomRight bounds, min/max zoom, and `downloading: true`. Persisted to Hive before tile download begins |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaMap` | `lib/src/widgets/tca_map.dart` | flutter_map wrapper with tile layers. Controlled via `MapController` for initial positioning |
| `TcaBackgroundGradient` | `lib/src/widgets/tca_background_gradient.dart` | Semi-transparent gradient at the top for app bar readability over the map |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used for "DONE", "SAVE", and "CANCEL" actions |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Styled `AlertDialog` wrapper for the name input dialog |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Text input for the area name, with `RequiredValidator` |

### Helpers

| Helper | File | Role |
|---|---|---|
| `PositionHelper` | `lib/src/helpers/position.dart` | `checkPermissions()` — checks location permission; `getCurrentPosition()` → user's LatLng; `fallbackPosition` → default LatLng. Used to centre the map on the user's position on init |

### Constants

| Constant | File | Role |
|---|---|---|
| `Constants.defaultMapZoom` | `lib/src/values/constants.dart` | Initial zoom level when centering the map |
| `Constants.maxDownloadZoom` | `lib/src/values/constants.dart` | Maximum zoom level for tile downloads. If the user has zoomed beyond this, the selected zoom is used instead |

### Services

| Service | File | Role |
|---|---|---|
| `SavedMapSectionRepository` | `lib/src/features/maptiles/services/saved_map_section_repository.dart` | Hive CRUD repository — `save(section, section.id)` persists the new section |
| `MaptileHelper` (via `maptileProvider`) | `lib/src/helpers/maptile.dart` | `downloadSection(section)` — begins background downloading of map tiles for the bounded area at zoom levels from `minZoom` to `maxZoom` |

---

## API / Backend Dependencies

None directly — no REST API calls. However, `maptileProvider.downloadSection()` triggers background HTTP requests to tile servers (OpenStreetMap / OpenSeaMap) to download map tiles for the selected area. These are tile-layer fetches, not application API calls.

---

## Data Flow

### Initial Map Positioning

```
initState
  │
  └─ Post-frame callback:
       │
       ├─ PositionHelper.checkPermissions()
       │    ├─ Has permission → getCurrentPosition() → user's LatLng
       │    └─ No permission → fallbackPosition
       │
       ├─ if hasPermission: setState(_currentPosition = position)
       │    └─ TcaMap shows blue dot at user's location
       │
       └─ _mapController.move(position, Constants.defaultMapZoom)
```

### Save Flow

```
User frames desired area → taps "DONE"
  │
  └─ _promptForName() → showDialog
       │
       ├─ User enters area name
       │
       └─ User taps "SAVE" → _saveArea(dialogContext)
            │
            ├─ Validate form (name required)
            │
            ├─ Read map state:
            │    ├─ selectedZoom = _mapController.camera.zoom.floor()
            │    ├─ maxZoom = max(selectedZoom, Constants.maxDownloadZoom)
            │    ├─ topLeft = camera.visibleBounds.northWest
            │    └─ bottomRight = camera.visibleBounds.southEast
            │
            ├─ SavedMapSection.create(
            │    name, topLeft, bottomRight,
            │    minZoom: selectedZoom, maxZoom, downloading: true
            │  )
            │
            ├─ SavedMapSectionRepository.save(section, section.id)
            │
            ├─ Invalidate savedMapSectionProvider
            │
            ├─ maptileProvider.downloadSection(section)
            │    └─ Begins background tile download
            │
            ├─ Navigator.pop(dialogContext)  ← close name dialog
            │
            └─ Navigator.popUntil(ManageSavedMapsScreen.routeName)
                 └─ Returns to manage screen, which rebuilds with new section
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| ManageSavedMapsScreen | `lib/src/screens/settings/manage_saved_maps_screen.dart` | None | "SAVE NEW MAP" button at the bottom of the saved maps list |

---

## Known Caveats / Tech-Debt Notes

1. **`_areaName` controller not cleared between uses**: The `TextEditingController` is an instance field that persists across the screen's lifetime. If the user opens the name dialog, cancels, and re-opens it, the previous text remains. This is minor since the screen is typically used once and popped.

2. **Download starts before dialog is dismissed** (lines 158-160): `maptileProvider.downloadSection(section)` is `await`ed before `Navigator.pop(dialogContext)`. This means the dialog stays open (with no loading indicator) until the download setup completes. If `downloadSection` takes time to initialise, the UI appears frozen. The actual tile fetching happens in the background, but the initial setup is synchronous from the user's perspective.

3. **`maxZoom` logic may be inverted** (lines 139-141): The code sets `maxZoom = selectedZoom > Constants.maxDownloadZoom ? selectedZoom : Constants.maxDownloadZoom`. This means if the user is zoomed in beyond `maxDownloadZoom`, the user's zoom is used as max. If zoomed out, `maxDownloadZoom` is used. The intent appears to be "always download up to at least `maxDownloadZoom`", which is correct — but the variable name `maxDownloadZoom` suggests it should be a cap, not a floor. The semantics may be confusing for future maintainers.

4. **No area size warning or tile count estimate**: The user can frame an arbitrarily large area at a high zoom level, potentially triggering a download of millions of tiles. There is no estimate shown of how many tiles will be downloaded or how much disk space will be used.

5. **No error handling on save or download**: Neither `SavedMapSectionRepository.save()` nor `maptileProvider.downloadSection()` is wrapped in try-catch. If Hive throws or the download setup fails, the error propagates unhandled.

6. **Position permission result partially ignored** (lines 57-61): When `hasPermission` is false, `_currentPosition` is not set (so `TcaMap` won't show a blue dot), but the map is still moved to `fallbackPosition`. The `Future.wait` pattern with `Future.value(PositionHelper.fallbackPosition)` and `Future.value(hasPermission)` is unnecessarily complex — a simple if/else with two separate branches would be clearer.

7. **`_formKey` validation not checked before `_saveArea`** (line 135): The `_saveArea` method correctly calls `_formKey.currentState!.validate()` and returns early if invalid. However, the `onTap` for the "SAVE" button is `() async => await _saveArea(dialogContext)` — the async wrapper is fine, but if the form state were null (e.g. disposed), the force-unwrap would crash.

8. **Screen does not show the selected bounds visually**: Unlike `ViewSavedMapScreen` which displays the saved area, this screen has no overlay rectangle or boundary indicator showing what area will be downloaded. The user must infer it from the visible map viewport.

9. **Lack of dialog on Android following saving a map**: No obvious indication that a map had been saved meant that the same map was saved/downloaded multiple times as I'd assumed it hadn't worked. When it has downloaded it returns to the saved maps list but this can take several seconds, according to the AI analysis this should happen as soon as the download starts.