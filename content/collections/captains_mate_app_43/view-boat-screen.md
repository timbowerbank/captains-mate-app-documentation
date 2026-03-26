---
id: 6f8afcde-baf2-425b-852a-4c57ecc1e140
blueprint: captains_mate_app_43
title: 'View Boat Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535374
---
# ViewBoatScreen

**File:** `lib/src/screens/members/view_boat_screen.dart`
**Route:** `/boats/details/boat`

---

## Purpose

Displays the details of a single boat. On init the screen loads the boat from the local Hive repository, then fetches the latest data from the API and merges it into the local copy. If the boat has been deleted (no longer in Hive), a deleted-resource modal is shown and the user is redirected back. If the boat data changes while viewing (detected via `ref.listen`), the screen refreshes and shows an updated-resource modal once. The screen also lists the boat's owners, each tappable to navigate to `ViewMemberScreen`.

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar (custom PreferredSize, transparent over primary background)
 │    ├── BackButton (white)
 │    └── Column (centred)
 │         ├── Text (boat.fullName or "Boat Details")
 │         └── Text ("Boat", subtitle)
 │
 ├── extendBodyBehindAppBar: true
 │
 └── body: _BuildBody
      └── NestedScrollView
           ├── headerSliverBuilder:
           │    └── SliverOverlapAbsorber > SliverSafeArea > SliverAppBar
           │         ├── toolbarHeight: 0
           │         └── bottom: _buildTabs()
           │              └── PreferredSize (48px)
           │                   └── Container (white, rounded top corners)
           │                        └── Tab ("OVERVIEW")
           │
           └── body: RefreshIndicator
                └── if isLoading: CircularProgressIndicator
                    else: SingleChildScrollView > Column
                         │
                         ├── _BuildBoat
                         │    ├── Text ("BOAT DETAILS")
                         │    ├── TcaBoatCard(boat)
                         │    └── _BuildMembers
                         │         ├── Text ("Owner(s)")
                         │         └── ...boat.fullOwners.map → GestureDetector
                         │              └── TcaMemberCard(owner)
                         │                   └── onTap → ViewMemberScreen
                         │
                         └── (empty space fills remaining height)
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `selectedBoatProvider` | `lib/src/data/providers/member_list_provider.dart` | `FutureProvider.autoDispose.family<Boat?, String>` — loads a single boat by ID from the local Hive `BoatRepository`. Used on init to get the cached boat, and watched via `ref.listen` to detect changes during sync |

### Models

| Model | File | Role |
|---|---|---|
| `Boat` | `lib/src/features/boats/models/boat.dart` | Hive-stored boat model. Fields used: `id`, `fullName`, `name`, `members`, `length`, `type`, `callsign`, `mmsi`, `model`, `additional`, `owners`, `fullOwners`. Has `isUpdated()` method for change detection and `copyWith` for merging API data |
| `Member` | `lib/src/features/members/models/member.dart` | Used for the owners list — each `fullOwners` entry is a `Member` rendered via `TcaMemberCard` |
| `SelectedBoatRoute` | `lib/src/helpers/route_argumnets/selected_boat_route.dart` | Route args carrying `id` — extracted from `ModalRoute.of(context)!.settings.arguments` |
| `SelectedMemberRoute` | `lib/src/helpers/route_argumnets/selected_member_route.dart` | Route args passed when navigating to `ViewMemberScreen` from an owner card |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaBoatCard` | `lib/src/widgets/tca_boat_card.dart` | Renders boat details (name, type, length, callsign, MMSI, model, etc.) in a styled card |
| `TcaMemberCard` | `lib/src/widgets/tca_member_card.dart` | Renders owner/member details in a styled card. Used for each entry in the "Owner(s)" section |
| `TcaResourceDeletedModal` | `lib/src/widgets/modals/tca_resource_deleted_modal.dart` | Modal dialog shown when the boat no longer exists in Hive. Non-dismissible; redirects to previous screen on "Go Back" |
| `TcaResourceUpdatedModal` | `lib/src/widgets/modals/tca_resource_updated_modal.dart` | Modal dialog shown once when the boat data has changed since the screen was opened |

---

## API / Backend Dependencies

| Call | Client | Endpoint | Description |
|---|---|---|---|
| `ApiClient.boat.getBoat(id)` | `BoatApiClient` | `GET /boat/{id}` | Fetches the latest boat data from the server. The response is merged into the locally-cached boat via `copyWith`. Called on init and on every refresh (pull-to-refresh) |

The API call is **not** required — if it fails (`.catchError`), the screen falls back to the Hive-cached data and sets `_isErrored = true` (though `_isErrored` is not currently used in the UI beyond being passed to `_BuildBody`).

---

## Data Flow

### Initial Load

```
initState
  │
  └─ Post-frame callback → initProviders()
       │
       ├─ setState(_isLoading = true)
       │
       └─ _loadBoat()
            │
            ├─ Read route args → SelectedBoatRoute.id
            │
            ├─ selectedBoatProvider(id).future → storedBoat (from Hive)
            │
            ├─ If storedBoat == null:
            │    ├─ Show TcaResourceDeletedModal
            │    ├─ Navigator.pop()
            │    └─ return
            │
            ├─ ApiClient.boat.getBoat(id) → fresh Boat
            │    └─ Merge into storedBoat via copyWith (preserves local fields)
            │
            ├─ .catchError → setState(_isErrored = true)
            │
            └─ .whenComplete → setState:
                 ├─ _boat = storedBoat (merged)
                 ├─ _isLoading = false
                 └─ _originalBoat = storedBoat (first load only, for change detection)
```

### Change Detection (During Sync)

```
build()
  │
  ├─ ref.listen(selectedBoatProvider(_boat?.id ?? ''))
  │    └─ On change → _loadBoat() (re-fetches from Hive + API)
  │
  ├─ Compare _originalBoat vs _boat via isUpdated()
  │    └─ If updated and modal not yet shown:
  │         └─ _showBoatUpdatedModal()
  │              ├─ Set _hasShownUpdateModal = true
  │              └─ Post-frame → showDialog(TcaResourceUpdatedModal)
  │
  └─ Render Scaffold with current _boat data
```

### Pull-to-Refresh

```
User pulls down on the list
  │
  └─ RefreshIndicator.onRefresh → initProviders()
       └─ _loadBoat() (same as initial load)
```

### Owner Navigation

```
User taps an owner card
  │
  └─ Navigator.pushReplacementNamed(
       ViewMemberScreen.routeName,
       SelectedMemberRoute(id: owner.id!)
     )
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| MembersListScreen | `lib/src/screens/members/members_list_screen.dart` | `SelectedMemberRoute(id: boat.id!)` | Tap on a boat in the Boats tab *(note: uses `SelectedMemberRoute` — see caveats)* |
| ViewMemberScreen | `lib/src/screens/members/view_member_screen.dart` | `SelectedMemberRoute(id: boat.id!)` | Tap on a boat in the member's boats list (via `pushReplacementNamed`) |

---

## Known Caveats / Tech-Debt Notes

1. **Route args cast to `SelectedBoatRoute` but callers pass `SelectedMemberRoute`** (line 47): The screen casts route arguments as `SelectedBoatRoute`, but `MembersListScreen` passes `SelectedMemberRoute(id: boat.id!)` and `ViewMemberScreen` also passes `SelectedMemberRoute(id: boat.id!)`. If these two classes have different runtime types, this cast will throw a `TypeError`. This works only if both classes happen to share the same structure and Dart doesn't enforce strict nominal typing at the cast point (which it does — this is likely a **latent runtime crash** unless the classes are aliased or one extends the other).

2. **`_isErrored` is set but never acted upon in the UI** (lines 30, 87): The `_isErrored` flag is set to `true` when the API call fails and is passed to `_BuildBody`, but `_BuildBody` never reads or renders anything based on it. There is no error state UI — the screen silently falls back to cached data.

3. **`_originalBoat` change detection runs in `build()`** (lines 112-115): The comparison `_originalBoat!.isUpdated(_boat!)` and the call to `_showBoatUpdatedModal()` happen synchronously inside `build()`. While the modal itself is deferred via `addPostFrameCallback`, triggering side effects in `build()` is an anti-pattern that can lead to unexpected behaviour during frame rebuilds.

4. **`pushReplacementNamed` for owner navigation** (line 361): When tapping an owner, the screen uses `pushReplacementNamed` to navigate to `ViewMemberScreen`. This means the boat screen is removed from the navigation stack — the user cannot press back to return to the boat. The same pattern is used in `ViewMemberScreen` for navigating to boats, creating a replacement chain rather than a stack.

5. **`ref.listen` in `build()` with empty-string fallback** (line 106): When `_boat` is null (during initial load), the listener watches `selectedBoatProvider('')` — a provider parameterised with an empty string. This triggers a Hive lookup for an ID of `''`, which is wasteful (returns null). Once `_boat` is loaded, subsequent rebuilds watch the correct ID.

6. **`ScrollController` created inside `build()`** (line 210): A new `ScrollController` is instantiated on every rebuild of `_BuildBody.build()`. This should be a field on the widget or managed via a `StatefulWidget` to avoid recreating it on each frame.

7. **Container height set to full screen** (line 248): The `SingleChildScrollView` child is given `height: MediaQuery.of(context).size.height`, which forces the content area to be exactly the screen height. This means the content is always scrollable (the `RefreshIndicator` works), but it also adds excessive empty space below the boat details.

8. **No merge back to Hive**: The API-fetched boat data is merged into `storedBoat` via `copyWith` and stored in local state, but is **not** written back to the Hive repository. The next time the screen opens, it will re-fetch from the API. This is intentional (avoids writing partial data), but means the Hive cache only updates during the global sync.