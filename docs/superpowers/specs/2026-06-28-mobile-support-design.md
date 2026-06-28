# Mobile Support Design — Stunda for Android + iOS

**Date:** 2026-06-28
**Status:** Approved
**Author:** Kodsama \<kodsama@protonmail.com\>

## Overview

Stunda is a Flutter desktop photo toolkit (GPS tagging, map explore, duplicate
finding, RAW pruning, and a "Shrink library" wizard) currently shipping on
macOS, Windows, and Linux. The codebase is a Dart pub workspace:
`packages/engine` (pure-Dart, Flutter-free, running scan/hash/detect work in a
worker isolate), `packages/cli`, `packages/mcp`, and `app/` (the Flutter app,
package `stunda`, bundle id `ai.kodsama.stunda`).

This document specifies full Android + iOS support with feature parity across
all five features — Explore on map, Find duplicates, Shrink library,
GPS-tag-from-tracks, and Prune RAW — including bundling ONNX Runtime and the ML
models on mobile so Smart-similarity and People/animal detection work there too.

The approved architecture is **Approach A — Materialize proxies at the edge**:
the app reads the device photo library on the main isolate, materializes
downscaled temp-file proxies (with original metadata embedded), and feeds those
plain files into the existing, unchanged engine pipeline.

## Goals

- Full Android + iOS support with parity across all five features.
- Smart-similarity and People/animal detection working on mobile via bundled
  ONNX Runtime + the two `.onnx` models.
- Keep the engine pure-Dart and Flutter-free; reuse the existing pipeline
  unchanged for scanning, hashing, quality scoring, and detection.
- Preserve the ≥99% coverage gate for all new pure-Dart code.

## Non-goals

- No redesign of the desktop scan/keeper/detection pipeline.
- No new features beyond mobile parity for the existing five.
- No cloud/sync, no web platform.
- On-device runtime verification is out of scope for CI (see Risks).

## Problem framing

Mobile breaks three assumptions the desktop build relies on, plus two
platform-integration gaps:

1. **No free filesystem traversal.** The scanner walks folders with
   `Directory.list()`. That is forbidden on mobile: iOS exposes photos only
   through `PHPhotoLibrary`, and Android 10+ through `MediaStore` — both as
   `content://` URIs, not file paths.
2. **exiftool can't run.** The Perl exiftool subprocess cannot execute on
   iOS/Android. This turns out to be **irrelevant** on mobile: the photo
   library already exposes GPS and date metadata directly, so no subprocess is
   needed to read it.
3. **ONNX Runtime native libs aren't bundled** for mobile, so detection and
   Smart-similarity have no inference backend.
4. **Trash is desktop-only** (platform-specific trash paths / PowerShell).

The engine is already well-abstracted behind ports — `ProcessRunner`,
`ExifBackend`, and `Trash` — and JPEG/PNG EXIF read/write are already pure-Dart.
Mobile support is therefore mostly a matter of supplying mobile implementations
behind those seams plus one new port for the photo library.

## Architecture — Approach A: Materialize proxies at the edge

**The key constraint:** photo-library plugins and platform channels only work on
the **main isolate**. The engine runs scanning, hashing, and detection in a
**worker isolate**. Plugins/channels are unavailable there. Therefore all
photo-library access happens app-side on the main isolate, and the engine
isolate only ever touches plain temp files — exactly as it does today.

**Data flow (mobile):**

1. Main isolate enumerates the device library and builds the
   `FolderScanResult`-equivalent purely from asset metadata (no folder walk).
2. For pixel-needing features, the main isolate exports a downscaled proxy JPEG
   per asset to a temp dir, embedding the original `DateTimeOriginal` + GPS into
   the proxy EXIF.
3. The engine isolate scans/hashes/quality-scores/detects those temp proxies
   exactly as on desktop — it never knows it isn't looking at a real folder.
4. The main isolate maps engine results (keyed by proxy path) back to asset ids
   and substitutes original dimensions/size for keeper selection and display.
5. Deletions route to `PhotoLibrary.delete`; GPS writes route to
   `PhotoLibrary.writeGps`.

On desktop nothing changes: the existing filesystem scanner and ports remain the
default.

## The `PhotoLibrary` port + `LibraryAsset`

New, pure-Dart, in
`packages/engine/lib/src/data/ports/photo_library.dart`. Abstract interface:

- `enumerate() -> List<LibraryAsset>`
- `exportProxy(id, maxEdge) -> tempPath`
- `thumbnail(id, edge) -> bytes`
- `fullBytes(id) -> bytes`
- `writeGps(id, lat, lng)`
- `delete(List<id>)`

`LibraryAsset` is a new pure-Dart value type: `id`, `filename`,
`type` (jpg/heic/raw/png/…), `width`, `height`, `byteSize`, `createdAt`, and
optional `latitude`/`longitude`.

A pure-Dart `PhotoLibraryTrash implements Trash` delegates deletion to a
`PhotoLibrary`, so the existing trash seam works on mobile with no engine
pipeline changes.

## Mobile scan flow

App-side implementation lives in
`app/lib/src/engine/device_photo_library.dart`, implementing `PhotoLibrary` with
the `photo_manager` pub plugin for enumerate / metadata / thumbnail / bytes /
delete. Because `photo_manager` cannot write GPS, a small custom platform method
channel `stunda/photo` handles GPS write-back:

- **iOS:** `PHAssetChangeRequest.location` (Swift).
- **Android:** androidx `ExifInterface` on the MediaStore content-URI file
  descriptor (Kotlin).

Proxy details: max-edge ~1024, cached by `assetId` + modification time. The
original `DateTimeOriginal` and GPS are embedded into the proxy via the engine's
pure-Dart JPEG backend so the unchanged engine pipeline reads correct metadata.
Result mapping uses a `proxyPath ↔ assetId` map; original `width`/`height`/
`byteSize` come from the `LibraryAsset` (not the downscaled proxy) so keeper
selection and on-screen sizes are correct.

## Per-feature wiring

- **Explore on map:** reads GPS from asset metadata directly — no proxies
  needed; thumbnails loaded on demand.
- **Find duplicates** and **Shrink library:** proxies + the engine pipeline;
  deletion via native `PhotoLibrary.delete`.
- **GPS-tag-from-tracks:** the engine resolves coordinates from imported GPX /
  Google tracks against each photo's capture time; the main isolate then writes
  via `PhotoLibrary.writeGps` — no file write.
- **Prune RAW:** pair RAW + JPEG assets by filename basename, trash unpaired
  RAW via native delete.

## ONNX on mobile

Bundle the ONNX Runtime native libraries:

- **Android:** the `onnxruntime-android` AAR (gradle dependency), which packages
  `libonnxruntime.so` per ABI, loadable via
  `DynamicLibrary.open('libonnxruntime.so')`.
- **iOS:** the `onnxruntime-c` CocoaPod dynamic framework, loadable via
  `DynamicLibrary.open('onnxruntime.framework/onnxruntime')`.

The two `.onnx` model files ship as Flutter assets. On mobile, assets aren't
real files, so at first run the app copies them from `rootBundle` to an
app-support dir and points the engine's bundle dir there. Extend
`ortLibraryFileName()` (engine) and the app's `onnxPlatformSubdir` / bundle
resolver for `android`/`ios`.

Detection runs via FFI in the worker isolate — `DynamicLibrary` works in any
isolate. On mobile, the metadata-first people tier (exiftool subject tags) is
unavailable, so detection goes straight to the ONNX tier.

## UI/UX & permissions

- `desktop_drop` drag-and-drop is guarded out on mobile.
- The mobile entry replaces "pick a folder" with a "grant photo access / scan
  library" flow (permission request via `photo_manager`).
- Responsive, touch-friendly layouts.
- Permissions declared:
  - **iOS Info.plist:** `NSPhotoLibraryUsageDescription`,
    `NSPhotoLibraryAddUsageDescription`.
  - **Android manifest:** `READ_MEDIA_IMAGES`, `ACCESS_MEDIA_LOCATION`.

## Testing & CI

- All new pure-Dart code — the `PhotoLibrary` port, `LibraryAsset`,
  `PhotoLibraryTrash`, and the scan-orchestration / result-mapping helpers — is
  unit-tested to keep the ≥99% coverage gate.
- `DevicePhotoLibrary` is kept thin behind seams so the untestable plugin/
  channel surface stays small.
- CI gains an Android job (`flutter build apk`) and an iOS job
  (`flutter build ios --no-codesign`) alongside the existing desktop matrix.
- Device-runtime behavior — real enumeration and real GPS writes — requires
  on-device testing outside CI.

## Phasing / rollout

1. **P1** — platform scaffolding.
2. **P2** — engine port + `LibraryAsset` + `PhotoLibraryTrash` + tests.
3. **P3** — app `DevicePhotoLibrary` + proxy export + scan orchestration +
   asset mapping.
4. **P4** — native GPS-write channels + permissions.
5. **P5** — ONNX mobile bundling.
6. **P6** — mobile UI/UX.
7. **P7** — build verification + CI.

## Risks & limitations

- **iOS GPS write creates an edited asset version.**
  `PHAssetChangeRequest.location` produces an adjustment/edit on the asset rather
  than mutating the original in place.
- **On-device runtime can't be verified in CI.** CI confirms the apps build;
  real enumeration, GPS write-back, and ONNX inference must be checked on
  physical devices.
- **App-size increase from ONNX.** Bundling ONNX Runtime native libs plus the
  `.onnx` models grows the installed app size on both platforms.
- **RAW in mobile libraries is uncommon.** Prune RAW remains supported for
  parity, but few mobile libraries contain RAW assets, so its real-world impact
  on mobile is limited.
