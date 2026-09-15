# Guard Security Vision Suite

Guard is a Windows/Qt multi-camera security application backed by the Qt-independent C++17 **GuardVision** analytics library and a Node.js/TypeScript backend. The suite supports live camera streams, test image/video sources, motion detection, face identification, fire/smoke hazard detection, local event handling and backend synchronization.

## Repository layout

```text
.
├── Guard/          # Qt Widgets desktop application
├── GuardVision/    # C++17 analytics library
├── backend/        # Node.js/TypeScript API + WebSocket backend
├── model_tools/    # Fire/smoke and crop/weed model download/evaluation helpers
├── IndoorFireSmokeDetectionUpdate.md
├── FIRE_SMOKE_AND_CROP_WEED_RESEARCH.md
└── CHANNEL_DISCUSSION.md
```

## Key capabilities

- up to four camera tiles in one Guard process,
- Hikvision and generic RTSP sources,
- `--test` mode with image and movie frame sources,
- software motion analysis and motion-triggered recording,
- face detection/tracking/identification through the licensed Identify SDK,
- fire and smoke detection through the licensed HazardDetect SDK,
- backend room registration, face-registry refresh and event delivery,
- offline event queueing and persistent per-camera diagnostics.

## Architecture

```text
Cameras / test media
        |
        v
      Guard (Qt)
        |
        | decoded VideoFrame + per-camera config
        v
    GuardVision (C++17)
      |      |       |
   Motion   Face   Hazard
      |      |       |
      +------+-------+
             |
     Guard event adapter
             |
             +----> local UI / warning / recording
             |
             +----> backend REST/WebSocket API
```

Guard owns camera transport, decoding, UI, recording and backend communication. GuardVision owns analytics scheduling/state and SDK integration.

## Indoor fire/smoke update

This revision includes a focused indoor hazard update:

- corrected hazard-box mapping after inference-frame downscaling,
- per-class confidence thresholds,
- multi-frame alarm confirmation instead of one-frame activation,
- smoke-first processing and faster smoke activation,
- optional overlapping tile inference to recover small/early hazards,
- duplicate-region merging before state tracking.

Default Guard tuning:

```text
vendor/common candidate floor: 0.25
smoke threshold:               0.38
fire threshold:                0.55
smoke activation:              2 analyzed frames
fire activation:               3 analyzed frames
clear confirmation:            3 analyzed frames
small-hazard tiling:            enabled
```

See [`IndoorFireSmokeDetectionUpdate.md`](IndoorFireSmokeDetectionUpdate.md) for the investigation, tradeoffs and verification boundary.

## Important model compatibility note

The source package intentionally excludes proprietary SDK binaries and the production `fire_smoke.onnx`. GuardVision's current hazard implementation calls the licensed `HazardDetect` API. An arbitrary YOLO ONNX file is **not guaranteed to be compatible** with that SDK even if the file is named `fire_smoke.onnx`.

A public ONNX evaluation candidate and checksum-verified download scripts are provided under `model_tools/fire_smoke/`. Use the `GUARD_HAZARD_MODEL` environment override to A/B test a candidate on a licensed test machine before replacing any production model.

For a future vendor-independent implementation, the research note recommends a native C++ ONNX path using ONNX Runtime/OpenCV/TensorRT and a well-defined YOLO output parser.

## Prerequisites

### Guard

- Windows x64
- Qt 5.x matching the project/toolchain (existing build scripts use Qt 5.14.2 examples)
- MSVC toolchain compatible with the licensed SDK libraries
- OpenCV, HCNetSDK and FFmpeg under the expected shared SDK layout
- built GuardVision library

### GuardVision

- CMake 3.15+
- C++17 compiler
- Identify SDK for face features
- HazardDetect SDK for fire/smoke features

GuardVision can be built without proprietary SDKs by disabling face/hazard features.

### Backend

- Node.js `>=16.20.2 <17`
- PostgreSQL configured through the backend environment file

## SDK layout

The projects expect shared dependencies outside the source repository:

```text
Desktop/
├── Guard/
├── GuardVision/
│   └── bin/
│       ├── Debug/guardvision.lib
│       └── Release/guardvision.lib
└── sdk/
    ├── OpenCV/
    ├── Identify/
    ├── HazardDetect/
    ├── HCNetSDK/
    └── FFmpeg/
```

Do not commit licensed vendor SDK binaries, private models, credentials or production secrets unless you have explicit redistribution rights.

## Build GuardVision

### Motion-only / open build

```powershell
cmake -S GuardVision -B GuardVision\_build\motion `
  -G "Visual Studio 16 2019" -A x64 `
  -DGUARDVISION_ENABLE_FACE=OFF `
  -DGUARDVISION_ENABLE_HAZARD=OFF `
  -DGUARDVISION_BUILD_TESTS=ON

cmake --build GuardVision\_build\motion --config Release --parallel
ctest --test-dir GuardVision\_build\motion -C Release --output-on-failure
```

### Full licensed build

```powershell
cd GuardVision
.\scripts\build-release.ps1 `
  -IdentifyRoot D:\SDK\Identify `
  -HazardDetectRoot D:\SDK\HazardDetect `
  -WarningsAsErrors
```

## Build Guard

From the `Guard` directory:

```powershell
.\scripts\build-debug.ps1 -QtRoot C:\Qt\Qt5.14.2\5.14.2\msvc2017_64
```

or:

```powershell
.\scripts\build-release.ps1 -QtRoot C:\Qt\Qt5.14.2\5.14.2\msvc2017_64
```

Populate the shared SDK and GuardVision library locations before building.

## Run Guard

Normal camera mode:

```powershell
.\Guard.exe
```

Test mode:

```powershell
.\Guard.exe --test
```

Test mode allows each camera card to use supported images or movies instead of a physical camera.

### Override the hazard model for a controlled test

```powershell
$env:GUARD_HAZARD_MODEL = "C:\models\candidate.onnx"
.\Guard.exe --test
```

Only use this with a model confirmed compatible with the installed HazardDetect runtime.

## Backend

```bash
cd backend
npm ci
npm run check
npm test
npm run build
npm run dev
```

Database helpers include:

```bash
npm run db:generate
npm run db:migrate
npm run db:setup
npm run db:studio
```

Copy/configure the project's environment values locally; do not commit secrets.

## Verification

The indoor hazard patch was verified with:

- GuardVision no-proprietary-SDK configure/build,
- `guardvision_tests`,
- hazard-enabled compile/link against an API-compatible stub,
- Guard indoor-hazard tuning source checks,
- relevant normal-frame, settings-lock, hazard-reconnect and face-registry lifecycle checks.

Real licensed HazardDetect inference still needs to be benchmarked on the Windows deployment machine because the vendor SDK and production model are not included in the repository.

## Open-source model research

See [`FIRE_SMOKE_AND_CROP_WEED_RESEARCH.md`](FIRE_SMOKE_AND_CROP_WEED_RESEARCH.md) for:

- indoor fire/smoke datasets,
- smoke-focused YOLO references,
- C++ inference sources,
- a public ONNX evaluation candidate,
- crop/weed source and model recommendations,
- licensing cautions.

## Safety

Computer-vision fire/smoke detection is an auxiliary monitoring layer. It should not replace code-compliant certified smoke detectors, heat detectors or building fire-alarm systems.

## License

No new top-level license is asserted by this README. Preserve the existing project licenses and comply with the licenses of all third-party SDKs, datasets and model weights. In particular, some researched datasets/models have non-commercial or AGPL terms and should not be assumed suitable for closed-source commercial redistribution.
