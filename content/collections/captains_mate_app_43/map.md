---
id: c2d9b21e-f726-4333-8d6b-f8cf76ac8cf8
blueprint: captains_mate_app_43
title: Map
use_synced_content: false
parent: a1b7cec3-99c2-4a5d-9c8b-9790b8204982
updated_by: 071c7123-3915-4c72-b79c-13c21fc2598f
updated_at: 1781167915
---
# Map System

## Overview

The map is built on [flutter_map](https://pub.dev/packages/flutter_map), an OpenStreetMap-compatible mapping library for Flutter. Tile caching is handled by [flutter_map_tile_caching (FMTC)](https://pub.dev/packages/flutter_map_tile_caching) using an ObjectBox backend. Users can also save rectangular map sections for offline use.

---

## Packages

| Package | Purpose |
|---|---|
| `flutter_map` | Core map widget and layer system |
| `flutter_map_tile_caching` | Tile caching, offline downloads, and store management |
| `flutter_map_marker_cluster` | Clusters markers at lower zoom levels |
| `latlong2` | `LatLng` type used throughout the map and tile calculation |

---

## Tile Layers and External Services

Three tile layers are stacked. The map style (map vs satellite) controls which base layer is active.

### Base map (map style)
- **URL**: `https://theca.org.uk/osmprox/tiles/{z}/{x}/{y}.png`
- A reverse proxy hosted by TCA that serves OpenStreetMap tiles.
- The app identifies itself via a `User-Agent` header set from the package version at runtime (`ca_mobile_app, version: X.Y.Z, build: N`). In debug mode `'DEBUG MODE'` is used; in profile mode `'PROFILE MODE'`; in release mode the string starts empty and is populated asynchronously once `PackageInfo.fromPlatform()` resolves.
- **Cache store**: `cache-base`

### Satellite (satellite style)
- **URL**: `https://www.google.com/maps/vt?lyrs=s&x={x}&y={y}&z={z}`
- Google Maps satellite imagery.
- **Cache store**: `cache-satellite`

### Sea marks (both map and satellite styles, not shown in the map style switcher preview)
- **URL**: `https://t1.openseamap.org/seamark/{z}/{x}/{y}.png`
- An overlay from [OpenSeaMap](https://www.openseamap.org) that adds navigational marks, buoys, and hazards on top of the base layer.
- Only serves tiles where sea mark data exists — tiles for land areas or sparse coastal regions will return a non-200 response. This is expected behaviour.
- **Cache store**: `cache-sea`

### Layer rendering order

```
Base map (or satellite)
    └── Sea marks overlay          [both styles; hidden in switcher preview]
        └── Offline section layers [map style only; disabled in satellite and switcher]
            └── Offline polygons   [map style only; disabled in satellite and switcher]
                └── Markers / clusters
```

---

## Tile Caching (FMTC)

### Initialisation

FMTC uses an **ObjectBox** backend, initialised once at app startup in `AppInitialisationService.initialize()`:

```dart
await FMTCObjectBoxBackend().initialise();
```

After that, `MaptileHelper.init()` ensures the three named stores exist (`cache-base`, `cache-satellite`, `cache-sea`), then recreates and re-downloads any offline section stores that are empty (to handle migrations from older app versions).

### Browse strategies

Each `TileLayer` uses `FMTCTileProvider` with a `BrowseStoreStrategy` that controls how tiles are loaded at browse time:

| Strategy | Behaviour |
|---|---|
| `readUpdateCreate` | Check cache → if miss, fetch from network and store result |
| `read` | Check cache only → never hit the network |

The three global stores (`cache-base`, `cache-satellite`, `cache-sea`) use `readUpdateCreate`, so they build up automatically as the user browses. Offline section stores use `read` — they only ever serve pre-downloaded tiles within their bounds.

### Error behaviour

When a tile is not in the cache and the tile server returns a non-200 HTTP response, FMTC throws `FMTCBrowsingError(negativeFetchResponse)`. This is most commonly triggered by the OpenSeaMap layer, which returns 404 for tiles with no sea mark data. Adding an `errorImage` to the tile layer causes a fallback image to be rendered instead of propagating the error.

---

## Offline Map Sections

Users can save a named rectangular region of the map for offline use. Each saved section is a `SavedMapSection` persisted in Hive (box: `saved_map_sections`).

### Data model

```
SavedMapSection
├── id          (UUID)
├── name        (user-defined)
├── topLeft     (LatLng)
├── bottomRight (LatLng)
├── minZoom     (int)
├── maxZoom     (int, capped at Constants.maxDownloadZoom = 13)
├── createdAt   (DateTime)
├── downloading (bool)
├── baseDownloadProgress (0.0–1.0)
└── seaDownloadProgress  (0.0–1.0)
```

Each section gets two dedicated FMTC stores named `{uuid}-cache-base` and `{uuid}-cache-sea`.

### Download flow

`MaptileHelper.downloadSection()` runs two concurrent foreground download streams (base + sea), each with 20 parallel threads. Progress is written back to Hive after every event so the UI can reflect it in real time.

```
section.toBaseDownloadable()  →  FMTCStore("{id}-cache-base").download.startForeground(...)
section.toSeaDownloadable()   →  FMTCStore("{id}-cache-sea").download.startForeground(...)
```

Both streams are awaited together with `Future.wait`. When complete, `downloading` is set to false and the provider is invalidated to trigger a UI refresh.

### Rendering offline sections

Downloaded sections are rendered as additional `read`-only tile layers constrained to their `LatLngBounds` (via `tileBounds`). This means:

- Inside the bounds: tiles are served from the offline store.
- Outside the bounds: the layer is skipped entirely.

A semi-transparent black polygon (`~7.5% opacity`) is drawn over each downloaded region so users can see where offline coverage exists.

Sections that are still downloading are excluded from rendering.

### Deletion

`MaptileHelper.deleteSection()` cancels any in-progress download streams, then deletes both FMTC stores.

`MaptileHelper.deleteOldMapCaches()` exists and filters sections created before `Config.mapTileCacheSyncDate`, but it is **never called** — not from `init()` or anywhere else. Old sections are not automatically purged; `deleteOldMapCaches()` is dead code.

---

## Map Styles

`TcaMapStyle` has two values: `map` and `satellite`. Style is passed as a prop to `TcaMap`. The satellite style disables the **offline section layers** but sea marks continue to render in satellite mode — the sea marks layer is only suppressed in the switcher preview (`isSwitcher: true`).

A style switcher widget (`isSwitcher: true`) renders a non-interactive preview of the map with interaction flags set to `0` and no sea layer.

---

## Interaction and Zoom

| Setting | Value |
|---|---|
| Default zoom | 9 |
| Min zoom | 2 |
| Max zoom | 18 |
| Default location zoom (on marker tap) | 13 |
| Max offline download zoom | 13 |

Enabled gestures: drag, fling, pinch-move, pinch-zoom, double-tap-to-zoom. Rotation is not enabled. The switcher and any `isDisabled` instance suppress all gestures.

---

## Marker Clustering

Markers are split by type and passed to `MarkerClusterLayerWidget` instances. There are normally three groups (`TcaLocationMarker`, `TcaHlrMarker`, `TcaFriendMarker`), but when the HLR filter is active (`hasHlr == true`), `TcaLocationMarker` is split across two cluster layers: locations with `primaryHlrs` use `isHlr: true` (no clustering) and locations without cluster normally — giving four cluster widget instances in total.

HLR markers use a `maxClusterRadius` of `0` (no clustering), while location and friend markers cluster within a 40 px radius.

Popups are managed by a shared `PopupController` and show a card with a "View" button that navigates to the detail screen.

---

## Known Server Issues

### TCA proxy returning HTTP 500 at zoom 14 and above

The TCA basemap proxy (`https://theca.org.uk/osmprox/tiles/{z}/{x}/{y}.png`) returns HTTP 500 errors for tiles at zoom level 14 and 15, confirmed by requesting tile URLs directly in a browser. This is a server-side infrastructure problem — the proxy receives the request but fails to serve the tile. Zoom levels 13 and below appear unaffected.

This is the primary cause of missing base map tiles when users zoom in. It needs to be investigated on the server side — likely a proxy configuration issue or a max zoom limit that is misconfigured to error rather than return a clean 404.

**Affected zoom levels**: 14+ (confirmed 14 and 15, likely higher)
**Region confirmed**: South Wales / Bristol Channel (51.5°N, 4°W), may be wider

### Google satellite tiles returning 404 for ocean and coastal areas

The Google satellite URL (`https://www.google.com/maps/vt?lyrs=s&x={x}&y={y}&z={z}`) returns 404 for tiles in areas where Google has no satellite imagery — primarily open ocean and some coastal regions. This is expected behaviour from Google, not an app bug. The URL generally works for most areas.

The satellite layer is also always active in the background via the map style switcher widget, which shows a satellite preview even when the user is in standard map mode. This means Google tile requests are made after every pan (with a 1-second debounce), regardless of whether the user has ever opened satellite mode.

---

## Tech Debt / Known Issues

### Bug: `_loadProviderState` never updates instance variables

`_TcaMapState._loadProviderState()` ([tca_map.dart:105](lib/src/widgets/tca_map.dart#L105)) declares local variables with the same names as the instance fields it intends to populate:

```dart
var _filterTypes = (await ref.read(filterTypeOptionProvider.future));
var _savedMapSections = (await ref.read(savedMapSectionProvider.future));

setState(() {
  _filterTypes = _filterTypes;       // local assigned to itself
  _savedMapSections = _savedMapSections; // local assigned to itself
});
```

The locals shadow the instance fields inside the entire function scope. The `setState` closure captures the locals, so the instance fields are never written. In practice this means `_filterTypes` is always an empty list and location type labels (e.g. "Marina, Anchorage") will never appear in map popups. `_savedMapSections` is partially rescued by the `ref.listen` in the build method, but that listener is also set up inside a `Consumer` builder which can register duplicate listeners on rebuilds.

### No `errorImage` on the satellite layer

The base map and sea marks layers both have `errorImage` fallbacks. The **satellite layer** does not — if a Google tile returns a non-200 response (common over open ocean), `FMTCBrowsingError(negativeFetchResponse)` propagates unhandled. The satellite layer should have an `errorImage` set to a transparent or blank asset.

### Unofficial Google Maps satellite URL

The satellite tile URL (`https://www.google.com/maps/vt?lyrs=s&...`) is an undocumented internal Google Maps endpoint. It is not covered by any public API agreement, can change without notice, and its use may violate Google's Terms of Service. If it breaks it will silently fail — there is no fallback source configured.

### Offline zoom cap vs map max zoom mismatch

`Constants.maxDownloadZoom = 13` means offline sections never include tiles above zoom 13. The map itself allows zooming to 18. When a user is within an offline section's bounds and zooms past 13, the `read`-only tile layers stop serving tiles but the network layers (using `readUpdateCreate`) take over — so offline behaviour silently degrades to online. There is no UI indication of this.

### No cache size limit or TTL on global stores

The three global stores (`cache-base`, `cache-satellite`, `cache-sea`) use `readUpdateCreate` with no maximum size, eviction policy, or tile expiry configured. They will grow unbounded as the user browses. On devices with limited storage this could eventually cause issues. FMTC supports `maxLength` and `cachedValidDuration` on the store or provider level.

### Single OpenSeaMap subdomain

The sea marks URL is hardcoded to `t1.openseamap.org`. OpenSeaMap operates subdomains `t1`–`t8` for load distribution. Using only `t1` concentrates all requests on one server and may contribute to rate limiting when panning rapidly.

### `User-Agent` is empty on first tile requests

`_userAgent` starts as an empty string in release builds and is only populated after `PackageInfo.fromPlatform()` resolves asynchronously in `initState`. Any tile fetches that happen before that future completes (likely on first map render) will be sent without a proper `User-Agent`, which may affect server-side analytics or rate limiting on the TCA proxy.

### `Consumer` wrapping inside a `ConsumerStatefulWidget`

`_TcaMapState.build()` wraps its entire return value in a `Consumer` widget, even though `_TcaMapState` already extends `ConsumerState` and has direct access to `ref`. The inner `Consumer` is redundant and causes an unnecessary extra widget in the tree.

### Untyped fields

`TcaMap.isDisabled` has no type annotation (`final isDisabled;`), and `locationTypes` in `_buildLocationPopup` is declared with `var` and never given a type. Both should be explicitly typed.

### `hybridTemplate` is dead code

**File:** [lib/src/helpers/maptile.dart](lib/src/helpers/maptile.dart)

`MaptileHelper.hybridTemplate` (`https://www.google.com/maps/vt?lyrs=y&x={x}&y={y}&z={z}`) is defined as a static constant but is never referenced in any tile layer. It is not included in the `templates` list either. It should be removed or used if a hybrid (satellite + labels) map style is ever added.

### Hive field gaps in `SavedMapSection`

Fields 6, 7, and 10 in the Hive type adapter were removed but their slot numbers are retired with comments only. This is correct practice for Hive, but there is no documentation of what those fields contained, which makes it harder to reason about future migrations.