---
id: 48440b38-4513-4968-9257-d3b9df4d0f43
blueprint: captains_mate_app_43
title: 'Members List Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535311
---
# MembersListScreen

**File:** `lib/src/screens/members/members_list_screen.dart`
**Route:** `/members`

---

## Purpose

A searchable, segmented directory of CA members and boats. The user switches between a **Members** tab and a **Boats** tab via a segmented control. A search bar filters results by name (case-insensitive substring match). All data is loaded from the local Hive repositories — no network calls are made. Tapping a member navigates to `ViewMemberScreen`; tapping a boat navigates to `ViewBoatScreen`.

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar
 │    └── Text ("MEMBERS" or "BOATS", changes with active tab)
 │
 └── body: Padding > Consumer > Column
      │
      ├── TcaSearchBar
      │    ├── hasFilters: false (no filter button)
      │    └── onSearch → setState(_searchValue)
      │
      ├── TcaSegmentedControl<MemberListScreenTabOption>
      │    ├── "MEMBERS" tab
      │    └── "BOATS" tab
      │    └── onValueChanged → clears search, toggles _isMemberType
      │
      └── Expanded
           └── TcaAsyncDataWrapper<List<dynamic>>
                └── provider: memberListProvider(SearchFilterOptions)
                     │
                     └── ListView.builder
                          ├── if Members tab: _BuildMemberItem
                          │    └── TcaListItem
                          │         ├── heading: member.fullName
                          │         ├── subHeading: boat names (comma-separated)
                          │         └── onTap → ViewMemberScreen
                          │
                          └── if Boats tab: _BuildBoatItem
                               └── TcaListItem
                                    ├── heading: boat.name
                                    ├── subHeading: owner names (comma-separated)
                                    └── onTap → ViewBoatScreen
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `memberListProvider` | `lib/src/data/providers/member_list_provider.dart` | `FutureProvider.autoDispose.family<List, SearchFilterOptions>` — loads all members or boats from Hive (`MemberRepository` or `BoatRepository`), filters by search string (case-insensitive `contains`), and sorts alphabetically by `fullName`. Re-fires whenever `SearchFilterOptions` changes (tab switch or search input) |

### Models

| Model | File | Role |
|---|---|---|
| `Member` | `lib/src/features/members/models/member.dart` | Hive-stored member model. `fullName` used as list heading; `boats` list used for sub-heading; `id` used for navigation |
| `Boat` | `lib/src/features/boats/models/boat.dart` | Hive-stored boat model. `name` used as list heading; `members` list used for sub-heading; `id` used for navigation |
| `SearchFilterOptions` | `lib/src/data/providers/member_list_provider.dart` | Simple options class with `isMemberType` (bool) and `searchValue` (String?). Used as the family key for `memberListProvider` |
| `MemberListScreenTabOption` | (defined in-file, line 16) | Local enum: `members`, `boats`. Used as the type parameter for `TcaSegmentedControl` |
| `SelectedMemberRoute` | `lib/src/helpers/route_argumnets/selected_member_route.dart` | Route args carrying `id` — used for both member and boat navigation |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaSearchBar` | `lib/src/widgets/tca_search_bar.dart` | Search input with debounced `onSearch` callback. `hasFilters: false` hides the filter button |
| `TcaSegmentedControl` | `lib/src/widgets/tca_segmented_control.dart` | iOS-style segmented control for switching between Members and Boats tabs |
| `TcaAsyncDataWrapper` | `lib/src/widgets/tca_async_data_wrapper.dart` | Handles loading/error/data states for `memberListProvider`. Typed as `List<dynamic>` since the provider returns either `List<Member>` or `List<Boat>` |
| `TcaListItem` | `lib/src/widgets/tca_list_item.dart` | Standard list row with heading, sub-heading, and tap handler |

---

## API / Backend Dependencies

None — all data is loaded from the local Hive `MemberRepository` and `BoatRepository`. Members and boats are synced from the API during the app's global sync process (on login / resume), not from this screen.

---

## Data Flow

### Search and Tab Switching

```
User types in search bar
  │
  └─ onSearch(value)
       ├─ if value != _searchValue: setState(_searchValue = value)
       └─ memberListProvider re-evaluated with new SearchFilterOptions
            └─ TcaAsyncDataWrapper rebuilds → filtered list

User taps "BOATS" tab
  │
  └─ onValueChanged(MemberListScreenTabOption.boats)
       ├─ setState:
       │    ├─ _searchValue = '' (clear search)
       │    ├─ _searchController.clear()
       │    └─ _isMemberType = false
       │
       └─ AppBar title changes to "BOATS"
       └─ memberListProvider re-evaluated (isMemberType: false)
            └─ Loads from BoatRepository instead of MemberRepository
```

### Navigation

```
User taps a member row
  │
  └─ pushNamed(ViewMemberScreen, SelectedMemberRoute(id: member.id))

User taps a boat row
  │
  └─ pushNamed(ViewBoatScreen, SelectedMemberRoute(id: boat.id))
```

---

## Navigation Callers

| Source | File | Context |
|---|---|---|
| TcaDrawer | `lib/src/widgets/tca_drawer.dart` | "Members" menu item in the app drawer |

---

## Known Caveats / Tech-Debt Notes

1. **`SelectedMemberRoute` reused for boats** (line 151): The boat item navigates to `ViewBoatScreen` using `SelectedMemberRoute(id: boat.id!)` rather than a dedicated `SelectedBoatRoute`. This works because both route argument classes carry just an `id` field, but it's semantically misleading. `ViewBoatScreen` itself actually expects `SelectedBoatRoute` — if the types were ever enforced at runtime with a strict cast, this would crash. *(Note: checking `view_boat_screen.dart` line 47 confirms it casts to `SelectedBoatRoute`, meaning this is a **latent bug** — the cast will fail at runtime if the types don't match. However, both classes likely have compatible structure.)*

2. **Provider returns `List<dynamic>`** (line 81, provider line 19): `memberListProvider` is typed as `FutureProvider<List, SearchFilterOptions>` (raw `List`), so `TcaAsyncDataWrapper` receives `List<dynamic>`. Items are cast to `Member` or `Boat` at the list item level (lines 92-93). This loses type safety — a type mismatch would only be caught at runtime.

3. **Search filtering uses `fullName` via dynamic dispatch** (provider line 30): The search filter accesses `element.fullName` on a dynamic `List`. Both `Member` and `Boat` have a `fullName` property, but this is not enforced by a shared interface or base class. If a model's property name changed, the error would only surface at runtime.

4. **`dispose()` calls `super.dispose()` before `_searchController.dispose()`** (lines 31-34): The conventional order is to dispose controllers first, then call `super.dispose()`. The current order is reversed, which can cause issues if the framework tries to access the controller after super disposal. In practice, this rarely causes problems but is technically incorrect.

5. **Search not debounced at widget level**: While `TcaSearchBar` may have internal debouncing, the `onSearch` callback triggers a `setState` that rebuilds the `Consumer` and re-fires the `FutureProvider` on every callback. If the search bar does not debounce, this could cause excessive Hive reads during rapid typing.

6. **No empty state message**: `TcaAsyncDataWrapper` handles the loading state, but if the search returns zero results, the `ListView.builder` simply renders zero items with no "No results found" message.

7. **Unused `Consumer` in `_buildTabOption`** (lines 105-114): The tab label widget is wrapped in a `Consumer` (with the `watch` parameter name from old Riverpod syntax), but it doesn't use `ref` to read any providers. The `Consumer` is unnecessary and adds a redundant rebuild scope.