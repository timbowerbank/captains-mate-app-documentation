---
id: da9d836b-6e4b-43d3-a180-1104ecb10bd2
blueprint: captains_mate_app_43
title: 'Facilities Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535123
---
# FacilitiesScreen

**File:** `lib/src/screens/create_edit_location/facilities_screen.dart`
**Route:** `/create_location/facilities`

---

## Purpose

The facilities-editing step within the create/edit location wizard. It displays all available location attributes (e.g. "Fuel", "Water", "Wi-Fi") loaded from local storage, sorted alphabetically. Each attribute can be tapped to open a bottom sheet where the user sets a Yes/No value and optionally adds an HTML comment. Selections are written back to the shared `createEditLocationProvider` state. The screen is pushed from `ProgressScreen` and pops back to it when the user taps "CONTINUE".

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    └── Text ("ADD FACILITIES")
 │
 └── body: Stack
      │
      ├── SingleChildScrollView > Consumer > Column
      │    │
      │    └── TcaAsyncDataWrapper<List<AttributeGroup>>
      │         └── Column of _buildAttribute() cards
      │              │
      │              └── per attribute (sorted A-Z across all groups):
      │                   TcaItemCard
      │                    ├── prefix: SVG icon (from attribute.iconFile)
      │                    ├── title: attribute.name
      │                    ├── backgroundColor: tertiary if selected, default otherwise
      │                    ├── suffix: Row
      │                    │    ├── chat icon (if comment exists)
      │                    │    └── ✓ or ✕ icon (if Yes/No selected)
      │                    └── onTap → _openAttributeDetails() bottom sheet
      │
      └── Align (bottomCenter)
           └── TcaButton ("CONTINUE")
                └── Navigator.pop(context)

─── Bottom Sheet (per attribute) ───

SingleChildScrollView > TcaBottomSheet > StatefulBuilder > Column
 ├── SVG icon
 ├── Text (question: "Does this location have {attribute}?" or "{attribute} information")
 │
 ├── if attribute.hasBool:
 │    Row of TcaRadio<YesNo> (Yes / No)
 │
 ├── if attribute.hasText:
 │    TcaFormField (html) with HtmlEditorController
 │
 ├── TcaButton ("ADD TO INFORMATION")
 │    └── Reads HTML text, updates createEditLocationProvider.attributes
 │
 └── TcaButton ("REMOVE INFORMATION", outlined)
      └── Removes this attribute from createEditLocationProvider.attributes
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `createEditLocationProvider` | `lib/src/data/providers/create_edit_location.dart` | `StateProvider<CreateEditLocationRequest>` — the shared wizard state. This screen reads the current `attributes` list to show selection state, and writes back updated attributes when the user adds or removes facility info |
| `attributeProvider` | `lib/src/data/providers/attribute.dart` | `FutureProvider<List<AttributeGroup>>` — loads all attribute groups (with nested attributes) from the local Hive `AttributeGroupRepository`. The screen flattens all groups into a single sorted list |

### Models

| Model | File | Role |
|---|---|---|
| `CreateEditLocationRequest` | `lib/src/features/locations/models/create_edit_location_request.dart` | Freezed model holding the wizard state. This screen reads/writes its `attributes` field (`List<CreateEditLocationAttribute>`) |
| `CreateEditLocationAttribute` | `lib/src/features/locations/models/create_edit_location_attribute.dart` | Hive model with `id` (attribute ID), `selected` (`YesNo?`), and `text` (HTML comment). Represents a single facility selection |
| `AttributeGroup` | `lib/src/features/meta/models/attribute_group.dart` | Hive model with `id`, `name`, and `attributes` (list of `Attribute`). Groups are flattened — the grouping is not displayed in the UI |
| `Attribute` | `lib/src/features/meta/models/attribute.dart` | Hive model defining a facility type: `id`, `name`, `iconUrl`, `hasBool` (supports Yes/No), `hasText` (supports comment), `filterBtn`. Has an `iconFile` getter that returns a cached SVG `File` |
| `YesNo` | `lib/src/enums/yes_no.dart` | Enum with `yes` and `no` values, used for the boolean selection on each attribute |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaAsyncDataWrapper` | `lib/src/widgets/tca_async_data_wrapper.dart` | Handles loading/error/data states for the `attributeProvider` FutureProvider |
| `TcaItemCard` | `lib/src/widgets/tca_item_card.dart` | Card widget for each attribute row. Shows icon prefix, name, and selection status suffix. Background colour changes when selected |
| `TcaBottomSheet` | `lib/src/widgets/tca_bottom_sheet.dart` | Styled bottom sheet container used for the attribute detail panel |
| `TcaRadio` | `lib/src/widgets/tca_radio.dart` | Styled radio button used for Yes/No selection in the bottom sheet |
| `TcaFormField` | `lib/src/widgets/tca_form_field.dart` | Used in `html` mode for the optional rich-text comment per attribute |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used for "CONTINUE", "ADD TO INFORMATION", and "REMOVE INFORMATION" actions |

---

## API / Backend Dependencies

None — this screen operates entirely on local state (`createEditLocationProvider`) and locally-cached attribute definitions from Hive. The actual API submission happens later in `ProgressScreen`.

---

## Data Flow

### Viewing Attributes

```
ProgressScreen
  │  user taps "Facilities" checklist item
  │
  └─ pushNamed → FacilitiesScreen
       │
       ├─ build():
       │    ├─ Consumer reads createEditLocationProvider.attributes
       │    └─ TcaAsyncDataWrapper loads attributeProvider (Hive)
       │         └─ Flattens all AttributeGroups → sorted list of Attributes
       │              └─ For each attribute:
       │                   ├─ Check if attribute.id exists in provider.attributes
       │                   ├─ Show selection state (✓/✕) and comment icon
       │                   └─ Colour card tertiary if selected
       │
       └─ User taps "CONTINUE"
            └─ Navigator.pop() → back to ProgressScreen
```

### Adding/Editing a Facility

```
User taps an attribute card
  │
  └─ _openAttributeDetails() → showModalBottomSheet
       │
       ├─ Displays attribute icon + question text
       ├─ If hasBool: shows Yes/No radio buttons (StatefulBuilder)
       ├─ If hasText: shows HTML editor with existing comment
       │
       └─ User taps "ADD TO INFORMATION"
            │
            ├─ Reads HTML text from _html controller
            ├─ Gets current attributes from createEditLocationProvider
            ├─ Removes existing entry for this attribute.id (if any)
            ├─ Adds new CreateEditLocationAttribute:
            │    ├─ id: attribute.id
            │    ├─ selected: _result (YesNo)
            │    └─ text: HTML content (or null if empty)
            ├─ Writes updated attributes list back to provider
            └─ Navigator.pop() → closes bottom sheet
```

### Removing a Facility

```
User taps "REMOVE INFORMATION" in bottom sheet
  │
  ├─ Gets current attributes from provider
  ├─ Removes entry matching this attribute.id
  ├─ Writes updated list back to provider
  └─ Navigator.pop() → closes bottom sheet
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| ProgressScreen | `lib/src/screens/create_edit_location/progress_screen.dart` | None | `pushNamed` when user taps the "Facilities" checklist item |

---

## Known Caveats / Tech-Debt Notes

1. **Shared `_result` and `_html` state across bottom sheets** (lines 35-38): The `_result` (YesNo) and `_html` (HtmlEditorController) are instance fields on the state class rather than scoped to each bottom sheet invocation. While `_result` is reset to the current selection on open (line 137) and cleared to null on close (line 265), and `_html` is cleared via `setText('')` (line 266), this pattern is fragile — if the bottom sheet is dismissed without tapping a button (e.g. swipe-down), the state from one attribute could leak to the next.

2. **`PanelController()` passed but unused** (line 148): A fresh `PanelController()` is created each time the bottom sheet opens and passed to `TcaBottomSheet`, but it is never used to programmatically control the panel. This appears to be a required parameter of `TcaBottomSheet` even when not needed.

3. **Attribute group structure is discarded** (line 63-64): The screen flattens all groups with `.expand((e) => e.attributes)` and sorts alphabetically. The group names are never displayed. If the UI later needs grouped display (e.g. collapsible sections per group), this flattening would need to be reverted.

4. **`attributes` list mutation** (lines 215-232, 241-253): Both "ADD TO INFORMATION" and "REMOVE INFORMATION" handlers call `List.from(location.attributes)` or use `location.attributes` directly. The "ADD" path creates a new list via `List.from()`, but the "REMOVE" path uses `location.attributes` directly and calls `removeAt()` on it. Since `CreateEditLocationRequest` is Freezed (immutable), the `attributes` getter returns the list by reference — mutating it directly could cause subtle issues if the list is referenced elsewhere. However, `copyWith` is called immediately after, so the new state is assigned correctly.

5. **SVG icon loading is async via `FutureBuilder`** (lines 269-284): Each attribute's icon is loaded via `attribute.iconFile` (a `Future<File?>`) and rendered through a `FutureBuilder`. On first render, all icons flash from empty to loaded. There is no caching of the resolved `File` objects, so this future re-resolves on every rebuild.

6. **No form validation**: Unlike ContentScreen, there is no form validation on the facilities screen. The user can tap "CONTINUE" without selecting any attributes. This is by design (facilities are optional), but there is also no validation within the bottom sheet — a user can tap "ADD TO INFORMATION" with neither Yes/No selected, resulting in a `CreateEditLocationAttribute` with `selected: null`.

7. **HTML editor keyboard insets** (line 196-197): The `TcaFormField` HTML editor is wrapped in a `Container` with `margin: MediaQuery.of(context).viewInsets`, which adds margin equal to the keyboard height. Combined with the `isScrollControlled: true` bottom sheet and the `EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom)` on the outer container (line 145), this results in double-compensation for the keyboard, potentially pushing the content too high when the keyboard is open.