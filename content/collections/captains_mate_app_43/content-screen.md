---
id: 857a0564-a18d-4c7f-99b5-c25b075cbdfb
blueprint: captains_mate_app_43
title: 'Content Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535068
---
# ContentScreen

**File:** `lib/src/screens/create_edit_location/content_screen.dart`
**Route:** `/create_location/content`

---

## Purpose

The main content-editing form within the create/edit location flow. Users enter a location name, set a pin position (in DMS format), select one or more location types, and fill in editable overview sections using rich-text (HTML) editors. The screen reads from and writes to the shared `createEditLocationProvider` state, so all changes persist across the multi-step wizard. When the user taps "CONTINUE", the screen either pops (if editing an existing location) or pushes to the `ProgressScreen` checklist.

This screen is used both for **creating new locations** (entered from `SetLocationScreen`) and for **editing existing locations** (entered from `ProgressScreen` with `shouldPop: true`).

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    ├── Title: "CREATE LOCATION" or "EDIT LOCATION" (based on shouldPop)
 │    ├── automaticallyImplyLeading: false (no back button)
 │    └── Action: IconButton (close ✕)
 │         └── Resets createEditLocationProvider if creating, pops to HomeScreen
 │
 └── body: Stack
      │
      ├── SingleChildScrollView > Form > Column
      │    │
      │    ├── TcaFormField (text)        ← "Name" with RequiredValidator
      │    │
      │    ├── Consumer > TcaFormField    ← "Pin Location" (read-only DMS display)
      │    │    ├── prefix: SvgPicture (red location marker)
      │    │    └── actionText: "Set Pin Location"
      │    │         └── onTap → pushNamed(SetLocationScreen)
      │    │              └── On return, refreshes DMS text from provider
      │    │
      │    ├── TcaFormField (multiselect)  ← "Type" with RequiredValidator
      │    │    └── Values populated from filterTypeOptionProvider
      │    │
      │    └── ...editable OverviewSections (filtered by isEditable)
      │         └── TcaFormField (html) per section
      │              ├── label: section.name (e.g. "Introduction")
      │              ├── HtmlEditorController per section
      │              └── onChanged → _saveChanges() → updates provider
      │
      └── Align (bottomCenter)
           └── TcaButton ("CONTINUE")
                └── Validates form, then pop or pushReplacement(ProgressScreen)
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `createEditLocationProvider` | `lib/src/data/providers/create_edit_location.dart` | `StateProvider<CreateEditLocationRequest>` — the shared mutable state for the entire create/edit location wizard. This screen reads initial values (name, position, typeIds, overviews) on init and writes back on every change |
| `overviewSectionProvider` | `lib/src/data/providers/overview.dart` | `FutureProvider<List<OverviewSection>>` — loads all overview section definitions from the local Hive `OverviewSectionRepository`. Only sections where `isEditable == true` are rendered as form fields |
| `filterTypeOptionProvider` | `lib/src/data/providers/type_option.dart` | `FutureProvider<List<TypeOption>>` — loads location type options from Hive (filtered to `filterButton == true`, plus a synthetic "Omnibus" entry). Used to populate the type multiselect field |

### Models

| Model | File | Role |
|---|---|---|
| `CreateEditLocationRequest` | `lib/src/features/locations/models/create_edit_location_request.dart` | Freezed model with Hive support. Holds `name`, `position` (LatLng), `typeIds`, `overviews` (list of `CreateEditLocationOverview`), `attributes`, `report`, `id`, `draftId`, `lastEdited`. Used as the shared state across the wizard |
| `CreateEditLocationOverview` | `lib/src/features/locations/models/create_edit_location_overview.dart` | Simple Hive model with `id` (maps to overview section ID) and `text` (HTML content). Stored in the request's `overviews` list |
| `OverviewSection` | `lib/src/features/overview/models/overview_section.dart` | Hive model defining a section template: `id`, `name`, `isRequired`, `isEditable`, `isSensitive`. Loaded from local storage (originally synced from API) |
| `TypeOption` | `lib/src/features/meta/models/type_option.dart` | Represents a location type (e.g. Marina, Anchorage). Has `id` and `value` (display name) |
| `ContentScreenArguments` | (defined in-file, line 24) | Simple args class with `shouldPop` boolean — when `true`, the screen pops on continue instead of pushing to ProgressScreen |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Polymorphic form field used in four configurations here: plain text (Name), read-only with action (Pin Location), multiselect (Type), and HTML rich-text editor (Overview sections) |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Styled button used for the "CONTINUE" action at the bottom |

### Helpers

| Helper | File | Role |
|---|---|---|
| `PositionHelper` | `lib/src/helpers/position.dart` | `positionToDms()` converts a `LatLng` to DMS string for the pin location display field |

---

## API / Backend Dependencies

None — this screen operates entirely on local state (`createEditLocationProvider`) and locally-cached data (overview sections and type options from Hive). The actual API calls happen later in `ProgressScreen` when the user publishes or saves.

---

## Data Flow

### Create Flow (New Location)

```
SetLocationScreen
  │  user sets pin position
  │  writes position to createEditLocationProvider
  │
  └─ pushReplacementNamed → ContentScreen (shouldPop: false)
       │
       ├─ initState:
       │    ├─ Reads name, typeIds, position from provider
       │    ├─ Initialises _name TextEditingController with listener
       │    │    └─ Every keystroke → updates provider.name
       │    ├─ Sets _pinLocation.text = PositionHelper.positionToDms(position)
       │    └─ Post-frame: loads overviewSections + filterTypes from Hive
       │
       ├─ User fills in form fields:
       │    ├─ Name → provider.name (via listener)
       │    ├─ Pin Location → taps "Set Pin Location"
       │    │    └─ pushNamed(SetLocationScreen, shouldPop: true)
       │    │         └─ On return, re-reads provider.position → updates DMS text
       │    ├─ Type → provider.typeIds (via onChanged)
       │    └─ Overview sections → _saveChanges(sectionId, html)
       │         └─ Finds/adds CreateEditLocationOverview in provider.overviews
       │
       └─ User taps "CONTINUE"
            ├─ Validates form (name required, type required, introduction required)
            └─ pushReplacementNamed → ProgressScreen
```

### Edit Flow (Existing Location)

```
ProgressScreen
  │  user taps "Overview" checklist item
  │
  └─ pushNamed → ContentScreen (shouldPop: true)
       │
       ├─ Same initialisation as above (reads existing data from provider)
       │
       └─ User taps "CONTINUE"
            ├─ Validates form
            └─ Navigator.pop(context)  ← returns to ProgressScreen
```

### Close Button Behaviour

```
User taps ✕ (close icon)
  │
  ├─ If shouldPop (editing): Navigator.pop()
  │
  └─ If not shouldPop (creating):
       ├─ Resets createEditLocationProvider to empty CreateEditLocationRequest()
       └─ Navigator.popUntil(HomeScreen.routeName)
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| SetLocationScreen | `lib/src/screens/create_edit_location/set_location_screen.dart` | None (defaults, `shouldPop: false`) | `pushReplacementNamed` after user sets pin location for a new location |
| ProgressScreen | `lib/src/screens/create_edit_location/progress_screen.dart` | `ContentScreenArguments(shouldPop: true)` | `pushNamed` when user taps the "Overview" checklist item to edit content |

---

## Known Caveats / Tech-Debt Notes

1. **`print('types: $_types')` left in code** (line 65): A debug `print` statement is left in `initState`, logging the types list on every screen initialisation. Should be removed or replaced with `TcaLog`.

2. **Route arguments reassigned in `build()`** (lines 116-118): `_shouldPop` is set from route arguments inside `build()`, meaning it runs on every rebuild. While harmless (the value doesn't change), reading route args once in `initState` or `didChangeDependencies` would be cleaner.

3. **`HtmlEditorController` instances are never disposed** (line 47): A map of `HtmlEditorController` instances is created per overview section but there is no `dispose()` call for them. The `_name` and `_pinLocation` controllers are properly disposed, but the HTML editor controllers are not.

4. **Overview section `_saveChanges` replaces by index** (lines 97-111): The method uses `indexWhere` + `replaceRange` to update an overview. The `replaceRange` call on line 103 also assigns the new value to the list before replacing (`overviews[index] = ...`), which is redundant — the `replaceRange` on the next line overwrites the same index. This double-write is harmless but confusing.

5. **No loading state for async data**: The screen loads `overviewSections` and `filterTypes` asynchronously in a post-frame callback but shows no loading indicator. If Hive reads are slow, the type multiselect and overview fields simply don't appear until the data loads, which could briefly show an incomplete form.

6. **`_types` is a detached copy**: `_types` is initialised as `List.from(...)` (line 63), creating a separate copy from the provider's `typeIds`. Changes to `_types` are written back to the provider in `onChanged`, but if external code modifies the provider's `typeIds` directly, `_types` would be stale. This is not currently an issue since only this screen writes type IDs.

7. **Introduction section validation is name-based** (line 242-245): The required validation for the Introduction section checks `section.name == 'Introduction'` by string comparison rather than using the `isRequired` field from the `OverviewSection` model. If the section name changes on the backend, validation would silently stop working.