# Indoor Fire/Smoke Detection Update

Date: 2026-09-15

## Scope

This update targets Guard installations that operate **inside rooms/buildings** and improves four practical failure modes:

1. false hazard alarms from weak single-frame detections,
2. missed small/early smoke or flame regions,
3. smoke not receiving enough priority for early warning,
4. incorrect hazard-box scaling after inference-frame downscaling.

The uploaded source package does **not** contain the proprietary HazardDetect SDK, its header/library/DLL, or the original `fire_smoke.onnx`. GuardVision therefore cannot prove that an arbitrary third-party ONNX graph is binary-compatible with the vendor `HazardDetect_DetectBgr` runtime. This update deliberately avoids replacing the production model blindly.

## Root causes found in the code

### 1. Single-frame alarm activation

Guard previously configured hazard activation with `activationFrameCount = 1` and a global confidence threshold of `0.35`. A single weak false positive could therefore become an active alarm immediately.

### 2. Small-object box mapping bug

GuardVision downsizes large camera frames before passing BGR data to the vendor hazard detector. The returned normalized boxes were then converted back to pixel coordinates using the **downscaled inference dimensions** instead of the original ROI/source dimensions. The resulting boxes could be too small, especially for 1080p/4K camera input, and could fail the minimum-region filter.

### 3. One threshold for both fire and smoke

Fire and smoke have different visual characteristics. Early smoke is often diffuse and lower-confidence than a clear flame. A single threshold forces an undesirable tradeoff between smoke recall and fire/false-positive precision.

### 4. Full-frame-only inference reduces small-event visibility

A small smoke plume can occupy only a few pixels after a large camera frame is resized for inference. The previous pipeline did not have a recovery pass for this case.

## Code changes

### GuardVision public hazard configuration

`HazardConfig` now supports:

```cpp
double fireMinimumConfidence = -1.0;
double smokeMinimumConfidence = -1.0;
int fireActivationFrameCount = 0;
int smokeActivationFrameCount = 0;
bool smallHazardTilingEnabled = false;
```

Negative per-class confidence values preserve the legacy common threshold. A per-class activation value of `0` preserves the legacy common activation count, so existing callers remain source-compatible.

### Guard indoor defaults

The Guard defaults are now:

| Setting | Value | Purpose |
|---|---:|---|
| Analysis interval | 120 ms | keeps CPU cost controlled |
| Vendor/common candidate floor | 0.25 | preserves faint early-smoke candidates |
| Smoke post-filter threshold | 0.38 | higher smoke recall |
| Fire post-filter threshold | 0.55 | reduces flame-like false positives |
| Smoke activation | 2 analyzed frames | earlier smoke alert |
| Fire activation | 3 analyzed frames | stronger temporal confirmation |
| Clear confirmation | 3 analyzed frames | prevents alarm chatter |
| Minimum region | 4 x 4 source pixels | permits small hazards after correct scaling |
| Small-hazard tiling | enabled | improves small smoke/fire visibility |

These are starting defaults, not universal calibrated values. The best thresholds should be tuned against the actual room cameras and lighting.

### Correct source-coordinate mapping

`HazardProcessor` now maps normalized detector boxes through the original source ROI rather than the resized inference buffer. Minimum-region checks therefore operate in source-frame pixels as intended.

### Smoke-first event handling

Smoke detections are processed before fire detections, and smoke can use a lower activation count. If both hazards become active on the same analyzed frame, the smoke event is emitted first.

### Overlapping small-hazard tiles

When the full-frame pass misses a hazard and `smallHazardTilingEnabled` is true, GuardVision can run overlapping quadrant passes. This makes an early/small plume occupy a larger percentage of the detector input. Duplicate boxes are merged using IoU before state tracking.

Tradeoff: tile recovery can add up to four extra detector calls on miss frames. Measure CPU/GPU usage on the deployment PC. Disable the flag if latency is unacceptable.

## Model recommendation

### Best domain dataset for this application

**PengBo0/Home-fire-dataset** is the best domain match found for an indoor-only Guard deployment. It contains 6,500 indoor images and explicitly emphasizes early smoke and small flames. The dataset is CC BY-NC 4.0, so it is not suitable for commercial training use unless the licensing situation is separately resolved.

### Best smoke-oriented open training source found

**wzjgo339/fire-smoke-detection** trains YOLOv8m on D-Fire. Its repository reports test smoke precision 0.832, smoke recall 0.815 and smoke mAP@50 0.779, with a large negative-image fraction. This is a useful training/reference source, but the repository does not expose its referenced `best.pt` as a normal downloadable release artifact.

### Ready ONNX evaluation candidate

**DarshanM0di/fireandsmoke** provides a 9.81 MB MIT-licensed ONNX file, input 640x640. Its model card reports AP50 74.3%, precision 73.6%, and recall 68.3%. Treat these as self-reported metrics and benchmark it on your room-camera clips before production use.

SHA-256 of the published ONNX file:

```text
5043e0efd1d474bcc901716e97946b8f8a787e4bd29451891f8dccfbe93322b6
```

**Important:** this ONNX is not automatically compatible with the proprietary HazardDetect SDK. Do not overwrite the production `fire_smoke.onnx` without first verifying that `HazardDetect_Init` accepts the graph and that class/output mapping is correct. The included download scripts obtain the candidate and verify its checksum.

## Recommended production training strategy

For Guard's indoor use, a purpose-trained model is preferable to a generic fire model:

1. Start from a modern lightweight YOLO detector.
2. Train on licensed indoor fire/smoke images.
3. Add substantial **hard-negative room footage** from the exact deployment cameras: steam, humidifiers, sunlight, lamps, TV/monitor content, reflections, white curtains, glossy surfaces, dust and motion blur.
4. Oversample early/low-opacity smoke and small flame examples.
5. Validate by camera/site, not by random frame split, so near-duplicate video frames do not inflate accuracy.
6. Optimize the alarm policy separately from detector confidence: temporal confirmation, per-class thresholds, and clearing hysteresis should remain in GuardVision.

## Verification performed in this environment

- GuardVision motion/no-proprietary-SDK build: **PASS**.
- `guardvision_tests`: **PASS**.
- Hazard-enabled compile/link check against an API-compatible local stub: **PASS**.
- Dedicated Guard indoor-hazard source test: **PASS**.
- Relevant existing Guard checks for normal frame submission, camera-settings connection locking, hazard reconnect incidents and face-registry lifecycle: **PASS**.
- Some unrelated Guard source-text regression checks already fail in the prior baseline and still fail unchanged; they are not caused by this hazard patch.

Real proprietary HazardDetect inference could not be executed because the SDK/runtime/model are intentionally absent from the uploaded source package.

## Safety boundary

Camera vision can provide earlier situational awareness, but it should not be the only life-safety mechanism. Use Guard alongside appropriately certified smoke/heat detection and building fire-alarm systems.
