---
id: 8e72069a-39fd-410d-a102-3c11851560b6
blueprint: captains_mate_app_43
title: 'View Member Screen'
use_synced_content: false
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1774535426
---
# ViewMemberScreen

**File:** `lib/src/screens/members/view_member_screen.dart`
**Route:** `/members/details/member`

---

## Purpose

Displays the details of a single CA member. On init the screen loads the member from the local Hive repository, then fetches the latest data from the API and merges it into the local copy. If the member has been deleted (no longer in Hive), a deleted-resource modal is shown and the user is redirected back. If the member data changes while viewing (detected via `ref.listen`), the screen refreshes and shows an updated-resource modal once. The screen also lists the member's boats (tappable to navigate to `ViewBoatScreen`), and conditionally shows an "Email the Member" button if the member is emailable. A sailing icon in the header allows navigating to the home map centred on the member's friend position (if the member is a friend and the Friends tab is active).

---

## UI Simplified Structure

```
Scaffold
 ├── AppBar (custom PreferredSize, transparent over primary background)
 │    ├── BackButton (white)
 │    └── Column (centred)
 │         ├── Text (member.fullName or "Member Details")
 │         └── Text ("Member", subtitle)
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
                         ├── _BuildMember
                         │    ├── Row
                         │    │    ├── Text ("MEMBER DETAILS")
                         │    │    └── Consumer → if member is a friend
                         │    │         and homeSelectedTab == friends:
                         │    │         GestureDetector (sailing icon)
                         │    │              └── onTap → navigate to HomeScreen
                         │    │                   with friendId, set map selection
                         │    └── TcaMemberCard(member)
                         │
                         ├── _BuildNearby (boats section)
                         │    ├── Text ("Boat(s)")
                         │    └── ...member.boats.map → TcaItemCard
                         │         ├── title: boat.fullName
                         │         └── onTap → pushReplacementNamed(ViewBoatScreen)
                         │
                         ├── if member.emailable:
                         │    TcaButton ("Email the Member")
                         │         └── onTap → pushNamed(SendMemberMessageScreen)
                         │
                         └── (commented-out "Alter Profile" button)
```

---

## Associated

### Providers

| Provider | File | Usage |
|---|---|---|
| `selectedMemberProvider` | `lib/src/data/providers/member_list_provider.dart` | `FutureProvider.autoDispose.family<Member?, String>` — loads a single member by ID from the local Hive `MemberRepository`. Used on init to get the cached member, and watched via `ref.listen` to detect changes during sync |
| `friendsProvider` | `lib/src/data/providers/friends.dart` | Watched inside `_BuildMember` to check if the current member matches a friend (by `memberId`). If so, a sailing icon is shown |
| `homeSelectedTab` | `lib/src/data/providers/home_selected_tab.dart` | Read inside `_BuildMember` to check if the current home tab is `friends`. The sailing icon only appears when both conditions are met (member is a friend AND friends tab is active) |
| `locationMapSelectedProvider` | `lib/src/data/providers/location.dart` | Written to when the sailing icon is tapped — sets the selected map marker to the friend's ID so the map centres on them |

### Models

| Model | File | Role |
|---|---|---|
| `Member` | `lib/src/features/members/models/member.dart` | Hive-stored member model. Fields used: `id`, `fullName`, `name`, `forenames`, `surname`, `dateElected`, `addresses`, `phones`, `emails`, `emailable`, `boats`. Has `isUpdated()` method for change detection and `copyWith` for merging API data |
| `Friend` | `lib/src/features/friends/models/friend.dart` | Used to match the member to a friend entry via `friend.memberId == member.id` |
| `Boat` | `lib/src/features/boats/models/boat.dart` | Used in the boats list — each `member.boats` entry is rendered via `TcaItemCard` |
| `SelectedMemberRoute` | `lib/src/helpers/route_argumnets/selected_member_route.dart` | Route args carrying `id` — extracted from `ModalRoute.of(context)!.settings.arguments` |
| `HomeScreenRoute` | `lib/src/helpers/route_argumnets/home_screen_route.dart` | Route args passed when navigating to HomeScreen with `friendId` for map centring |
| `SendMemberMessageScreenArguments` | `lib/src/screens/send_member_message_screen.dart` | Route args carrying `member` and `id` for the email screen |

### Widgets

| Widget | File | Role |
|---|---|---|
| `TcaMemberCard` | `lib/src/widgets/tca_member_card.dart` | Renders member details (name, address, phone, etc.) in a styled card |
| `TcaItemCard` | `lib/src/widgets/tca_item_card.dart` | Used for each boat row in the "Boat(s)" section |
| `TcaButton` | `lib/src/widgets/tca_button.dart` | Used for the "Email the Member" action |
| `TcaResourceDeletedModal` | `lib/src/widgets/modals/tca_resource_deleted_modal.dart` | Modal dialog shown when the member no longer exists in Hive. Non-dismissible; redirects to previous screen |
| `TcaResourceUpdatedModal` | `lib/src/widgets/modals/tca_resource_updated_modal.dart` | Modal dialog shown once when the member data has changed since the screen was opened |

---

## API / Backend Dependencies

| Call | Client | Endpoint | Description |
|---|---|---|---|
| `ApiClient.members.getMember(id)` | `MembersApiClient` | `GET /member/{id}` | Fetches the latest member data from the server. The response is merged into the locally-cached member via `copyWith`. Called on init and on every pull-to-refresh |

The API call is **not** required for the screen to function — if it fails (`.catchError`), the screen falls back to the Hive-cached data and sets `_isErrored = true` (though `_isErrored` is not used in the UI).

---

## Data Flow

### Initial Load

```
initState
  │
  └─ Post-frame callback → _initProviders()
       │
       ├─ setState(_isLoading = true)
       │
       └─ _loadMember()
            │
            ├─ Read route args → SelectedMemberRoute.id
            │
            ├─ selectedMemberProvider(id).future → storedMember (from Hive)
            │
            ├─ If storedMember == null:
            │    ├─ Show TcaResourceDeletedModal
            │    ├─ Navigator.pop()
            │    └─ return
            │
            ├─ ApiClient.members.getMember(id) → fresh Member
            │    └─ Merge into storedMember via copyWith
            │         (id, boats, name, forenames, surname, dateElected,
            │          addresses, phones, emails, emailable)
            │
            ├─ .catchError → setState(_isErrored = true)
            │
            └─ .whenComplete → setState:
                 ├─ _isLoading = false
                 ├─ _member = storedMember (merged)
                 └─ _originalMember = storedMember (first load only)
```

### Change Detection (During Sync)

```
build()
  │
  ├─ ref.listen(selectedMemberProvider(_member?.id ?? ''))
  │    └─ On change → _loadMember() (re-fetches from Hive + API)
  │
  ├─ Compare _originalMember vs _member via isUpdated()
  │    └─ If updated and modal not yet shown:
  │         └─ _showMemberUpdatedModal()
  │              ├─ _hasShownUpdateModal = true
  │              └─ Post-frame → showDialog(TcaResourceUpdatedModal)
  │
  └─ Render Scaffold with current _member data
```

### Friend Sailing Icon Navigation

```
User taps sailing icon (visible only if member is a friend + friends tab active)
  │
  ├─ Set locationMapSelectedProvider to friend.id
  │
  └─ Navigator.pushNamedAndRemoveUntil(
       HomeScreen.routeName,
       (route) => false,
       args: HomeScreenRoute(friendId: friend.id)
     )
```

### Boat Navigation

```
User taps a boat card
  │
  └─ Navigator.pushReplacementNamed(
       ViewBoatScreen.routeName,
       SelectedMemberRoute(id: boat.id!)
     )
```

### Email Navigation

```
User taps "Email the Member"
  │
  └─ Navigator.pushNamed(
       SendMemberMessageScreen.routeName,
       SendMemberMessageScreenArguments(member: member!, id: args.id)
     )
```

---

## Navigation Callers

| Source | File | Arguments | Context |
|---|---|---|---|
| MembersListScreen | `lib/src/screens/members/members_list_screen.dart` | `SelectedMemberRoute(id: member.id!)` | Tap on a member in the Members tab |
| ViewBoatScreen | `lib/src/screens/members/view_boat_screen.dart` | `SelectedMemberRoute(id: owner.id!)` | Tap on an owner in the boat's owners list (via `pushReplacementNamed`) |

---

## Known Caveats / Tech-Debt Notes

1. **`_isErrored` is set but never acted upon in the UI** (lines 37, 95): The flag is set to `true` when the API call fails and passed to `_BuildBody`, but `_BuildBody` never renders an error state. The screen silently falls back to cached data.

2. **Side effects in `build()`** (lines 119-122): The `isUpdated()` comparison and `_showMemberUpdatedModal()` call happen synchronously inside `build()`. While the modal is deferred via `addPostFrameCallback`, triggering side effects in `build()` is an anti-pattern that can cause unexpected behaviour during rebuilds.

3. **`ref.listen` with empty-string fallback** (line 112): When `_member` is null (during initial load), the listener watches `selectedMemberProvider('')` — triggering a Hive lookup for an empty-string ID, which returns null. This is wasteful but harmless.

4. **`pushReplacementNamed` for boat navigation** (line 427): Navigating to a boat replaces the member screen in the stack, so the user cannot press back to return. Combined with `ViewBoatScreen` using the same pattern for owner navigation, this creates a replacement chain where back-navigation returns to whatever was before the member screen (e.g. `MembersListScreen`).

5. **Route args re-read in `_BuildBody.build()`** (lines 218-219): `SelectedMemberRoute` is extracted from `ModalRoute.of(context)` inside the stateless `_BuildBody.build()` method. Since `_BuildBody` is a `StatelessWidget` nested within the main screen, this is read on every rebuild. It should be passed as a constructor parameter from the parent instead.

6. **`SelectedMemberRoute` used for boat navigation args** (line 428): When navigating to `ViewBoatScreen`, the screen passes `SelectedMemberRoute(id: boat.id!)`. However, `ViewBoatScreen` casts its arguments as `SelectedBoatRoute` (line 47 of that file). This is the same latent type-cast issue noted in the `ViewBoatScreen` documentation — it works only if the two route arg classes share a compatible structure.

7. **Commented-out "Alter Profile" button** (lines 276-291): A block of commented-out code for an "Alter Profile" button (linking to `Config.profileUrl`) remains in the file. This was presumably removed intentionally but the dead code adds noise.

8. **`ScrollController` created inside `build()`** (_BuildBody line 217): A new `ScrollController` is instantiated on every rebuild of `_BuildBody.build()`. This should be a field or managed via a `StatefulWidget` to avoid per-frame allocation.

9. **Container height set to full screen** (line 248): The content is given `height: MediaQuery.of(context).size.height`, forcing a fixed-height scrollable area. This adds excessive empty space below the member details but ensures the `RefreshIndicator` always works.

10. **No merge back to Hive**: The API-fetched member data is merged into local state via `copyWith` but is **not** persisted to the Hive `MemberRepository`. The cache only updates during the global sync process.

11. **Friend check uses `collection` package** (line 376): The `firstWhereOrNull` from the `collection` package is used to find a matching friend. This correctly returns null if no match exists, but the import of `collection/collection.dart` at line 25 pulls in the entire package for a single utility method.