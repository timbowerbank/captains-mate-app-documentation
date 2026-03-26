---
id: ef24be5e-e847-4547-a5cd-508e49184362
blueprint: captains_mate_app_43
title: 'Progress Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535174
---
# ProgressScreen

**File:** `lib/src/screens/create_edit_location/progress_screen.dart`
**Route:** `/create_location/progress`

---

## Purpose

The checklist hub of the create/edit location wizard. It displays the location name and three actionable steps — Overview, Facilities, and Report — each showing completion status. From here the user can navigate into each step to add or edit content, then return to this screen to see updated progress. Once ready, the user can **publish** a new location (API create), **save** edits to an existing location (API update), or **save as draft** (local Hive storage). After a successful publish, the screen navigates to the newly created location's detail view and cleans up any associated draft.

---

## UI Simplified Structure

```
Consumer (watches createEditLocationProvider)
 └── Scaffold
      ├── AppBar
      │    ├── Title: "CREATE LOCATION" or "EDIT LOCATION" (based on location.id)
      │    ├── automaticallyImplyLeading: false
      │    └── Action: IconButton (close ✕)
      │         └── Resets createEditLocationProvider → Navigator.pop()
      │
      └── body: Container > Column
           │
           ├── Text (location name)
           │
           ├── TcaItemCard ("Overview")
           │    ├── suffix: green tick SVG (always shown — Overview is already filled)
           │    └── onTap → pushNamed(ContentScreen, shouldPop: true)
           │
           ├── TcaItemCard ("Facilities")
           │    ├── suffix: CircleAvatar with count of selected attributes
           │    └── onTap → pushNamed(FacilitiesScreen)
           │
           ├── TcaItemCard ("Report")
           │    ├── suffix: green tick SVG (if report exists) or grey tick
           │    └── onTap → pushNamed(CreateEditReportScreen, isCreatingLocation: true)
           │
           ├── Spacer
           │
           ├── if location.id == null:
           │    TcaButton ("PUBLISH LOCATION")
           │         └── _publishLocation()
           │
           ├── if location.id != null:
           │    TcaButton ("SAVE LOCATION")
           │         └── _saveLocation()
           │
           └── TcaButton ("SAVE AS DRAFT", outlined)
                └── _saveAsDraft()
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `createEditLocationProvider` | `lib/src/data/providers/create_edit_location.dart` | `StateProvider<CreateEditLocationRequest>` — the shared wizard state. Watched via `Consumer` so the checklist updates reactively. Reset to empty on close/publish/save |
| `locationListProvider` | `lib/src/features/locations/providers/location_list_provider.dart` | `Riverpod(keepAlive: true)` notifier wrapping `LocationRepository`. Called for `createLocation()` (publish) and `updateLocation()` (save edits). Both methods POST/PATCH via `ApiClient.location`, save to Hive, and invalidate `locationsProvider` |
| `locationProvider` | `lib/src/data/providers/location.dart` | `FutureProvider.family<Location, String>` — used during save-edit flow to fetch the original `Location` object by ID, which is required by `updateLocation()` |
| `authenticationProvider` | `lib/src/features/auth/providers/auth_provider.dart` | Reads `currentUser` to pass to `createLocation()` and `updateLocation()` (the API needs the user for author attribution) |
| `draftLocationProvider` | `lib/src/data/providers/draft_location.dart` | `FutureProvider<List<CreateEditLocationRequest>>` — invalidated after saving or deleting a draft to refresh the drafts list |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Accesses `DraftLocationRepository` for saving and deleting drafts in Hive |

### Models

| Model | File | Role |
|---|---|---|
| `CreateEditLocationRequest` | `lib/src/features/locations/models/create_edit_location_request.dart` | Freezed model holding the full wizard state. Key fields checked here: `position` (guard — if null, renders empty Container), `name` (displayed as title), `id` (null = new location, non-null = editing), `attributes` (count shown on Facilities badge), `report` (presence determines Report tick colour), `draftId` (used to delete draft after publish) |
| `Location` | `lib/src/features/locations/models/location.dart` | Returned by `createLocation()` — its `id` is used to navigate to `ViewLocationScreen`. Also fetched by `locationProvider` during the save-edit flow |
| `CreateReportRequest` | `lib/src/features/reports/api/requests/create_report_request.dart` | Constructed inline when navigating to the Report step — pre-populated with `_location.report?.title` and `_location.report?.text` |
| `CreateEditReportScreenArguments` | `lib/src/screens/create_edit_report_screen.dart` | Route args for the report sub-screen: `isCreatingLocation: true`, `draftLocation`, `draftReport` |
| `ContentScreenArguments` | `lib/src/screens/create_edit_location/content_screen.dart` | Route args for the content sub-screen: `shouldPop: true` (so it pops back here) |
| `SelectedLocationRoute` | `lib/src/helpers/route_argumnets/selected_location_route.dart` | Route args for navigating to `ViewLocationScreen` after publish |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaItemCard` | `lib/src/widgets/tca_item_card.dart` | Card widget for each checklist step (Overview, Facilities, Report) |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used for Publish, Save, and Save as Draft actions. Supports `isLoading` and `isDisabled` states |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Error dialog shown when publish or save fails (displays `DioException.message`) |

---

## API / Backend Dependencies

| Action | Via | Endpoint | Description |
|---|---|---|---|
| Publish new location | `locationListProvider.createLocation()` → `ApiClient.location.createLocation()` | `POST /mLocation` | Sends the full `CreateEditLocationRequest` with current user. Returns the created `Location` with server-assigned ID |
| Save edited location | `locationListProvider.updateLocation()` → `ApiClient.location.updateLocation()` | `PATCH /mLocation/{id}` | Sends the diff between original location and updated request. Returns the updated `Location` |

Both methods also save the returned `Location` to the local Hive `LocationRepository` and invalidate `locationsProvider` to refresh cached data.

Draft operations (save/delete) are purely local via `DraftLocationRepository` (Hive).

---

## Data Flow

### Publish Flow (New Location)

```
User taps "PUBLISH LOCATION"
  │
  ├─ setState(_isPublishing = true)
  │
  ├─ locationListProvider.createLocation()
  │    ├─ POST request to API
  │    ├─ Save returned Location to Hive
  │    └─ Invalidate locationsProvider
  │
  ├─ Reset createEditLocationProvider to empty
  │
  ├─ Show snackbar "New location published"
  │
  ├─ If _location.draftId != null (was a draft):
  │    ├─ Delete draft from DraftLocationRepository
  │    └─ Invalidate draftLocationProvider
  │
  └─ Navigator.pushNamedAndRemoveUntil(
       ViewLocationScreen,
       ModalRoute.withName(HomeScreen),
       args: SelectedLocationRoute(id: newLocation.id)
     )

  on DioException:
    ├─ Show TcaDialog with error message
    └─ setState(_isPublishing = false)
```

### Save Flow (Edit Existing Location)

```
User taps "SAVE LOCATION"
  │
  ├─ setState(_isSaving = true)
  │
  ├─ Read original Location via locationProvider(_location.id)
  │
  ├─ locationListProvider.updateLocation()
  │    ├─ PATCH request to API
  │    ├─ Save returned Location to Hive
  │    └─ Invalidate locationsProvider
  │
  ├─ Reset createEditLocationProvider to empty
  │
  ├─ Show snackbar "Location updated"
  │
  └─ Navigator.pop()

  on error:
    ├─ Show TcaDialog with error message
    └─ setState(_isSaving = false)
```

### Save as Draft Flow

```
User taps "SAVE AS DRAFT"
  │
  ├─ Assign draftId if null (UUID v4)
  ├─ Set lastEdited to now
  │
  ├─ Save to DraftLocationRepository (Hive)
  │
  ├─ Invalidate draftLocationProvider
  │
  ├─ Reset createEditLocationProvider to empty
  │
  ├─ Navigator.pop()
  │
  └─ Show snackbar "Location saved as draft"
```

### Sub-Screen Navigation

```
User taps "Overview"  → pushNamed(ContentScreen, shouldPop: true)
                         ← pops back; provider already updated

User taps "Facilities" → pushNamed(FacilitiesScreen)
                          ← pops back; provider already updated

User taps "Report"     → pushNamed(CreateEditReportScreen,
                           isCreatingLocation: true,
                           draftLocation: _location,
                           draftReport: CreateReportRequest(title, text))
                          ← pops back; provider already updated
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| ContentScreen | `lib/src/screens/create_edit_location/content_screen.dart` | None | `pushReplacementNamed` after user completes the content form for a new location |
| MyDraftsScreen | `lib/src/screens/my_drafts_screen.dart` | None | `pushNamed` after setting `createEditLocationProvider` to the selected draft location |
| ViewLocationScreen | `lib/src/screens/view_location_screen.dart` | None | `pushNamed` after setting `createEditLocationProvider` from an existing location (edit flow via "EDIT LOCATION" button) |

---

## Known Caveats / Tech-Debt Notes

1. **Guard renders empty `Container` when position is null** (line 49): If `_location.position == null`, the screen returns an empty `Container()` instead of the `Scaffold`. This can happen briefly if the provider is reset while the screen is still mounted, producing a blank screen with no way to navigate away.

2. **`_isPublishing` not reset on draft cleanup failure**: In `_publishLocation()` (lines 220-273), if the API call succeeds but the subsequent draft deletion fails, `_isPublishing` remains `true` (the button stays in loading state). However, since navigation happens immediately after, this is unlikely to be visible.

3. **`_isSaving` not reset on success** (lines 177-217): In `_saveLocation()`, `setState(_isSaving = true)` is called at the start but never set back to `false` on the success path — the screen navigates away via `Navigator.pop()`. If the pop were to fail or be intercepted, the save button would remain in a loading state. The error path correctly resets it.

4. **Report step always constructs a new `CreateReportRequest`** (lines 113-123): When navigating to the Report step, a fresh `CreateReportRequest` is constructed from `_location.report?.title` and `_location.report?.text`. Any additional report fields (e.g. images) stored in the provider's report are not forwarded, though this is mitigated by the `CreateEditReportScreen` writing directly back to the provider's report field.

5. **Close button resets state without confirmation** (lines 298-306): Tapping the ✕ button immediately resets `createEditLocationProvider` to an empty state and pops the screen. There is no "discard changes?" confirmation dialog, so a user who accidentally taps close loses all unsaved wizard progress.

6. **No offline handling**: The publish and save operations call the API directly. If the device is offline, a `DioException` is caught and shown in a dialog, but there is no specific offline message or suggestion to save as draft instead.

7. **`_location` reassigned on every build** (line 47): The `_location` instance field is reassigned from `ref.watch(createEditLocationProvider)` inside the `Consumer` builder on every rebuild. While this is necessary for reactivity, it means `_location` is effectively a local variable masquerading as an instance field — the field declaration with default value on line 39 is never used.