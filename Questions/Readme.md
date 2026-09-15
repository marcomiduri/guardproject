I completed the indoor fire/smoke investigation, code changes, testing, open-source research, documentation, and packaging.

The false alarms were not caused only by fire_smoke.onnx. Guard was configured with a `0.35` global confidence threshold and one-frame alarm activation, so a single weak prediction could immediately trigger an alarm. I changed the design to use separate fire/smoke thresholds and temporal confirmation. Smoke now has higher priority: the default smoke threshold is `0.38` with 2-frame confirmation, while fire uses 0.55 with 3-frame confirmation.

I also found a real small-object bug in GuardVision. When a large camera frame was reduced for inference, the returned fire/smoke box was being converted using the resized dimensions instead of the original source ROI. That could make small smoke/flame boxes artificially tiny and cause them to be discarded. This is fixed. I also added optional overlapping 2×2 tile inference for small/early hazards and duplicate-region merging.

GuardVision tests pass, including the new smoke-priority tests. The dedicated Guard indoor-hazard test passes, as do the normal-frame-pipeline, camera-settings connection-lock, hazard reconnect, and face-registry lifecycle checks.

One important limitation remains: your uploaded project does not contain the proprietary HazardDetect SDK or the original fire_smoke.onnx. GuardVision calls HazardDetect_DetectBgr, so I cannot safely replace that model with a generic YOLO ONNX and claim it is compatible—the proprietary DLL may expect a particular model/output format. I therefore did not silently overwrite the production model. The package instead includes a checksum-verified model download/evaluation kit and supports controlled testing through GUARD_HAZARD_MODEL.

For your indoor use case, the best domain-specific dataset I found is Home-Fire Dataset: 6,500 indoor images with emphasis on early smoke and small flames. Note that its CC BY-NC 4.0 license restricts commercial use. Among the open projects I reviewed, the D-Fire YOLOv8m project has particularly promising smoke results: its reported test smoke precision is 0.832, recall 0.815, and mAP@50 0.779.

For native C++, my preferred open inference source for this project is taifyang/yolo-inference because it supports modern YOLO models with ONNX Runtime, OpenCV, OpenVINO, TensorRT and LibTorch, including OpenCV 4.10-era setups. If you want fewer third-party abstractions, OpenCV's official dnn/yolo_detector.cpp is the cleaner long-term base.

For crop/weed identification, the most immediately useful ready model I found is WeedBlaster YOLOv8s. It recognizes eight crops—Maize, Sugar Beet, Soy, Sunflower, Potato, Pea, Bean and Pumpkin—plus a Weed superclass, with self-reported overall precision of 0.960 and weed precision of 0.850. It is AGPL-3.0, so review that license carefully for a commercial closed-source product.

Downloads:

Recommended complete package: GuardSuite_indoor_fire_smoke_update.zip — Guard + GuardVision + current fixed backend + GitHub README + research + discussion transcript + model tools.
Guard_indoor_fire_smoke_fixed.zip
GuardVision_indoor_fire_smoke_fixed.zip
backend_current_fixed.zip — the local .env is deliberately excluded; .env.example remains.
AI_model_download_tools.zip — checksum-verified PowerShell/Linux downloaders for the fire/smoke candidate and crop/weed model, plus crop/weed ONNX exporter.
README.md — GitHub-ready root README.
CHANNEL_DISCUSSION.md — visible discussion transcript from this channel.
IndoorFireSmokeDetectionUpdate.md — technical investigation and fix details.
FIRE_SMOKE_AND_CROP_WEED_RESEARCH.md — model/source comparison, URLs, licensing and recommendations.
SHA-256 checksums
— 9.81 MB, 640×640, published SHA-256 5043e0efd1d474bcc901716e97946b8f8a787e4bd29451891f8dccfbe93322b6. The model card reports AP50 74.3%, precision 73.6%, and recall 68.3%; these are self-reported and should be validated on your room-camera footage.
— best.pt, 22.6 MB, with published SHA-256 2a10f51e1d78db493b6c75662986748bebc01562fbce70f7f2b49f0176b4ecb6.

I could not embed the Hugging Face ONNX bytes directly in the ZIP because this execution environment cannot retrieve Hugging Face's Xet-hosted binary payload. The download kit handles that on your Windows/Linux machine and verifies the exact published checksum before use.

For a production Guard deployment, I would ultimately train a dedicated indoor model using licensed indoor fire/smoke data plus hard negatives recorded from your actual rooms—steam, humidifiers, sunlight, reflections, TVs/monitors, lamps, curtains and motion blur. That is likely to reduce false alarms more than simply swapping between generic public fire models.
