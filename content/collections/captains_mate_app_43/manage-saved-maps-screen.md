---
id: 4b261b6b-7406-4232-adf1-48ee228db62e
blueprint: captains_mate_app_43
title: 'Manage Saved Maps Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535487
---
# ManageSavedMapsScreen

**File:** `lib/src/screens/settings/manage_saved_maps_screen.dart`
**Route:** `/settings/manage_saved_maps`

---

## Purpose

A management screen for offline saved map areas. It lists all saved map sections from the local Hive repository, showing each section's name, download date, and download progress (if still downloading). Users can rename a section via a dialog, delete individual sections or all sections at once, navigate to the home map centred on a section, or create a new saved map via `NewSavedMapScreen`. Deletion clears both the Hive record and the cached map tiles on disk.

---

## UI Simplified Structure

```
TcaAsyncDataWrapper<List<SavedMapSection>>
 └── provider: savedMapSectionProvider
      └── Scaffold
           ├── AppBar
           │    ├── Text ("MANAGE SAVED MAPS")
           │    └── Action: IconButton (delete icon)
           │         └── _showDeleteAllDialog()
           │
           └── body: Stack
                │
                ├── Consumer
                │    ├── if empty: Text ("No saved maps", centred)
                │    └── else: ListView.builder
                │         └── per section: _buildSection()
                │              └── Material > InkWell > ListTile
                │                   ├── title: section.name
                │                   ├── subtitle:
                │                   │    ├── if downloading: TcaLinearProgressIndicator
                │                   │    └── else: "Downloaded {date}"
                │                   ├── trailing: Wrap
                │                   │    ├── IconButton (edit) → _promptForName()
                │                   │    └── IconButton (delete) → _showDeleteDialog()
                │                   └── onTap (if not downloading):
                │                        ├── Set locationMapSelectedProvider = section.id
                │                        └── pushNamedAndRemoveUntil(HomeScreen)
                │                             with HomeScreenRoute(sectionId)
                │
                └── Align (bottomCenter)
                     └── TcaButton ("SAVE NEW MAP")
                          └── pushNamed(NewSavedMapScreen)

─── Rename Dialog ───

TcaDialog > Form
 ├── TcaFormField ("Update Area Name", with RequiredValidator)
 ├── TcaButton ("SAVE")
 │    └── Save to SavedMapSectionRepository → invalidate provider → pop
 └── TcaButton ("CANCEL", outlined) → pop

─── Delete Single Dialog ───

TcaDialog
 ├── Text ("Are you sure you want to delete this saved map?")
 ├── TcaButton ("DELETE")
 │    └── Delete from repo → delete tiles via maptileProvider → invalidate → pop
 └── TcaButton ("CANCEL", outlined) → pop

─── Delete All Dialog ───

TcaDialog
 ├── Text ("Are you sure you want to delete all saved maps?")
 ├── TcaButton ("DELETE")
 │    └── Clear repo → delete all tiles via maptileProvider → invalidate → pop
 └── TcaButton ("CANCEL", outlined) → pop
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `savedMapSectionProvider` | `lib/src/data/providers/saved_map_section.dart` | `FutureProvider<List<SavedMapSection>>` — loads all saved map sections from Hive `SavedMapSectionRepository`. Wrapped by `TcaAsyncDataWrapper` for loading state. Invalidated after every mutation (rename, delete, delete-all) |
| `repositoryManagerProvider` | `lib/core/providers/repository_provider.dart` | Accesses `SavedMapSectionRepository` for save, delete, and clear operations |
| `maptileProvider` | `lib/src/data/providers/maptile.dart` | `Provider<MaptileHelper>` — used to cancel ongoing tile downloads and delete cached tiles for individual sections (`deleteSection`) or all sections (`deleteSections`) |
| `locationMapSelectedProvider` | `lib/src/data/providers/location.dart` | Written to when a section is tapped — sets the selected map ID so the home screen can centre on the saved area |

### Models

| Model | File | Role |
|---|---|---|
| `SavedMapSection` | `lib/src/features/maptiles/models/saved_map_section.dart` | Hive-stored model representing a saved map area. Fields used: `id`, `name`, `downloading` (bool), `downloadProgress` (double), `createdAt` (date). Has `copyWith` for renaming |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaAsyncDataWrapper` | `lib/src/widgets/tca_async_data_wrapper.dart` | Handles loading/error/data states for `savedMapSectionProvider` |
| `TcaLinearProgressIndicator` | `lib/src/widgets/tca_linear_progress_indicator.dart` | Shows download progress bar for sections still being downloaded |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used for "SAVE NEW MAP", dialog confirm/cancel actions |
| `TcaDialog` | `lib/src/widgets/tca_dialog.dart` | Styled `AlertDialog` wrapper for rename, delete, and delete-all dialogs |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Text input in the rename dialog, with `RequiredValidator` |

### Helpers / Services

| Helper | File | Role |
|---|---|---|
| `SavedMapSectionRepository` | `lib/src/features/maptiles/services/saved_map_section_repository.dart` | Hive CRUD repository for `SavedMapSection` records |
| `MaptileHelper` (via `maptileProvider`) | `lib/src/helpers/maptile.dart` | Manages on-disk tile cache. `deleteSection()` removes tiles for one section; `deleteSections()` removes tiles for a list of sections |

### Constants

| Constant | File | Role |
|---|---|---|
| `AppDateFormats.date` | `lib/src/values/date_formats.dart` | Date formatter for the "Downloaded {date}" subtitle |

---

## API / Backend Dependencies

None — all data is local. Saved map sections are stored in Hive and map tiles are cached on disk. No network calls are made from this screen (tile downloads are initiated from `NewSavedMapScreen`).

---

## Data Flow

### Viewing Saved Maps

```
Screen opens
  │
  └─ TcaAsyncDataWrapper loads savedMapSectionProvider
       └─ SavedMapSectionRepository.getAll() (Hive)
            └─ Renders ListView of sections
                 ├─ Downloading sections: show progress bar, tap disabled
                 └─ Completed sections: show date, tap navigates to HomeScreen
```

### Tapping a Section (Navigate to Map)

```
User taps a completed section
  │
  ├─ if section.downloading: return (no-op)
  │
  ├─ Set locationMapSelectedProvider = section.id
  │
  └─ Navigator.pushNamedAndRemoveUntil(
       HomeScreen.routeName,
       (route) => false,
       args: HomeScreenRoute(sectionId: section.id)
     )
```

### Renaming a Section

```
User taps edit icon → _promptForName()
  │
  ├─ Pre-fill areaName controller with section.name
  │
  └─ showDialog (Form + TcaFormField)
       │
       └─ User taps "SAVE"
            ├─ section = section.copyWith(name: areaName.text)
            ├─ SavedMapSectionRepository.save(section, section.id)
            ├─ Invalidate savedMapSectionProvider
            └─ Navigator.pop(dialogContext)
```

### Deleting a Single Section

```
User taps delete icon → _showDeleteDialog()
  │
  └─ showDialog (confirmation)
       │
       └─ User taps "DELETE"
            ├─ SavedMapSectionRepository.delete(section.id)
            ├─ maptileProvider.deleteSection(section)
            │    └─ Cancels any ongoing download + clears tile cache
            ├─ Invalidate savedMapSectionProvider
            └─ Navigator.pop(dialogContext)
```

### Deleting All Sections

```
User taps delete icon in AppBar → _showDeleteAllDialog()
  │
  └─ showDialog (confirmation)
       │
       └─ User taps "DELETE"
            ├─ Read all sections from savedMapSectionProvider.future
            ├─ SavedMapSectionRepository.clear()
            ├─ maptileProvider.deleteSections(items)
            │    └─ Cancels all ongoing downloads + clears all tile caches
            ├─ Invalidate savedMapSectionProvider
            └─ Navigator.pop(dialogContext)
```

### Creating a New Map

```
User taps "SAVE NEW MAP"
  │
  └─ Navigator.pushNamed(NewSavedMapScreen.routeName)
       └─ On return, savedMapSectionProvider has been invalidated by NewSavedMapScreen
            └─ TcaAsyncDataWrapper rebuilds with updated list
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| SettingsScreen | `lib/src/screens/settings/settings_screen.dart` | None | "Saved Maps" setting item under the "Offline Storage" section |

---

## Known Caveats / Tech-Debt Notes

1. **Shared `TextEditingController` and `FormKey` across rename dialogs** (lines 32-33): `areaName` and `formKey` are instance fields reused for every rename dialog. Since only one dialog can be open at a time this works, but if the dialog were dismissed mid-edit, the controller retains the previous text. The controller is pre-filled on each open (line 132), so this is mostly cosmetic.

2. **`_promptForName` does not validate before saving** (lines 148-159): The form has a `RequiredValidator` on the `TcaFormField`, but the "SAVE" button's `onTap` never calls `formKey.currentState!.validate()`. An empty name will bypass validation and be saved.

3. **Delete-all reads from the provider future** (line 188-189): `_showDeleteAllDialog` reads `savedMapSectionProvider.future` to get the list of sections to delete tiles for, but immediately after calls `clear()` on the repository and `invalidate()` on the provider. If the future hasn't resolved yet, `items` could be stale. In practice, the data is already loaded (since the list is visible), so this is unlikely to be an issue.

4. **No confirmation before navigating away**: Tapping a section immediately clears the entire navigation stack and navigates to HomeScreen. There is no confirmation, which could be surprising if the user tapped accidentally.

5. **`Consumer` wrapper in body is unnecessary** (lines 51-65): The `Consumer` inside the `Stack` body rebuilds on any watched provider change, but it doesn't read any providers — `savedMapSections` is passed down from `TcaAsyncDataWrapper`. The `Consumer` adds a redundant rebuild scope.

6. **Download progress is display-only**: The `TcaLinearProgressIndicator` shows `section.downloadProgress`, but this value comes from the Hive record. If the download is happening in the background (via `maptileProvider`), the progress updates only when the Hive record is updated. There is no stream or polling mechanism on this screen to refresh the progress in real time — the user would need to pull-to-refresh or re-open the screen.

7. **No error handling on rename or delete operations**: The `save`, `delete`, and `clear` calls on the repository are not wrapped in try-catch. If Hive throws (e.g. storage full), the error would propagate unhandled.