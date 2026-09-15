# Open-Source Fire/Smoke and Crop/Weed Vision Research

Research date: 2026-09-15

## Fire and smoke

### Recommended stack for Guard

There is no single public model I would label universally "best" for an indoor commercial surveillance product without testing it against the actual camera domain. For this project, the strongest combination is:

- **Indoor domain data:** `PengBo0/Home-fire-dataset`
- **Smoke-oriented training/reference project:** `wzjgo339/fire-smoke-detection`
- **C++ runtime:** `taifyang/yolo-inference`, or the official OpenCV DNN YOLO sample for a smaller dependency surface
- **Ready ONNX evaluation candidate:** `DarshanM0di/fireandsmoke`

### 1. Home-Fire Dataset — best domain match

Source: https://github.com/PengBo0/Home-fire-dataset

Why it is relevant:

- 6,500 images
- indoor household/surveillance scenarios
- early ignition stages
- explicit emphasis on small-scale flames and early smoke
- YOLO annotations

License: CC BY-NC 4.0. This is a non-commercial license, so commercial product training requires separate licensing/permission or different training data.

### 2. D-Fire YOLOv8m fire/smoke project — strongest smoke metrics among the reviewed open sources

Source: https://github.com/wzjgo339/fire-smoke-detection

Repository-reported test metrics:

| Metric | Smoke | Fire |
|---|---:|---:|
| mAP@50 | 0.779 | 0.642 |
| Precision | 0.832 | 0.720 |
| Recall | 0.815 | 0.683 |

The source supports ONNX/TensorRT export and uses the D-Fire dataset. The referenced trained `best.pt` is shown as a local training output path rather than a normal public release download, so plan to retrain or request the author’s weights.

### 3. Ready ONNX candidate — DarshanM0di/fireandsmoke

Model card: https://huggingface.co/DarshanM0di/fireandsmoke

Model file: https://huggingface.co/DarshanM0di/fireandsmoke/blob/main/best%20%284%29.onnx

Direct resolve URL used by the included download scripts:

https://huggingface.co/DarshanM0di/fireandsmoke/resolve/main/best%20%284%29.onnx?download=true

Published properties:

- MIT license
- ONNX
- 640 x 640 input
- 9.81 MB
- self-reported AP50 74.3%
- self-reported precision 73.6%
- self-reported recall 68.3%

Published SHA-256:

`5043e0efd1d474bcc901716e97946b8f8a787e4bd29451891f8dccfbe93322b6`

This is an **evaluation candidate**, not a validated drop-in replacement for Guard's proprietary HazardDetect model contract.

### 4. C++ inference source — taifyang/yolo-inference

Source: https://github.com/taifyang/yolo-inference

This is the best practical open C++ runtime fit found for Guard because it supports multiple YOLO generations and ONNX Runtime, OpenCV, OpenVINO, TensorRT and LibTorch backends. Its documented test matrix includes OpenCV 4.10.0, which is close to Guard's existing dependency generation.

### 5. Minimal/official C++ alternative — OpenCV DNN YOLO sample

Source: https://github.com/opencv/opencv/blob/4.x/samples/dnn/yolo_detector.cpp

If dependency minimization matters more than convenience, OpenCV's official `cv::dnn` sample is a strong base for a native C++ ONNX detector. It keeps inference in the OpenCV ecosystem already used by the project.

## Crop and weed identification

### Recommended ready model — WeedBlaster Vision YOLOv8s

Model: https://huggingface.co/NvMayMay/weedblaster-vision-yolov8s

Weight file: https://huggingface.co/NvMayMay/weedblaster-vision-yolov8s/blob/main/best.pt

Direct resolve URL used by the included download scripts:

https://huggingface.co/NvMayMay/weedblaster-vision-yolov8s/resolve/main/best.pt?download=true

Model card characteristics:

- YOLOv8s
- 1280 x 1280 input
- 8 crop classes plus one Weed superclass
- classes: Maize, Sugar Beet, Soy, Sunflower, Potato, Pea, Bean, Pumpkin, Weed
- self-reported overall precision: 0.960
- self-reported weed precision: 0.850
- weight size: 22.6 MB
- AGPL-3.0 license

Published SHA-256:

`2a10f51e1d78db493b6c75662986748bebc01562fbce70f7f2b49f0176b4ecb6`

Validate licensing before embedding this model in a closed-source/commercial product because AGPL has significant redistribution/source-availability obligations.

### Dataset/source

Underlying CropAndWeed dataset:

https://github.com/cropandweed/cropandweed-dataset

The model card describes 8,034 annotated images and about 112k plant instances across 74 species in the source dataset, later mapped to the 9-class crop-vs-weed task.

### Deployment source — OpenWeedLocator ONNX

Source: https://github.com/cropcrusaders/OpenWeedLocator-onnx

OpenWeedLocator is a practical open weed-detection/deployment project aimed at in-crop and fallow scenarios and is useful as a reference for field hardware and ONNX deployment workflows.

## Recommendation summary

For **Guard fire/smoke**, keep the new temporal and small-object code changes regardless of model choice. Then collect a camera-specific validation set and compare at least two candidate models. The most valuable future model improvement is likely to come from hard-negative indoor footage and early-smoke examples, not from architecture changes alone.

For **crop/weed**, WeedBlaster is the most immediately useful ready model found for the requested crop-vs-weed task, but its AGPL license needs review before commercial integration.
