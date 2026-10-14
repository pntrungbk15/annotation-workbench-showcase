# Model Gallery

These are the examples the application's Model Gallery shows: each output was produced by the model itself,
through the application's inference worker on a laptop CPU, after one warm-up run. Nothing is drawn by hand.
The latency is that single measured run (see [testing](testing.md#model-latency-and-memory) for medians).

## YOLOX-S

*YOLOX, anchor-free one-stage detector* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/frame0.jpg) | ![output](gallery/outputs/yolox-s-street.jpg) |

Closed-set detection of the 80 COCO classes on a busy crossing. 33 results: 21 person, 5 car, 3 handbag, 3 traffic light, 1 backpack.

- Input: [Diagonal crosswalk at Yonge and Dundas, Toronto (frame of the video)](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm) by Raysonho, CC0 1.0 (frame at 20 s, resized to 960 × 540).
- Run: version 0.1.1rc0, CPU, ONNX Runtime 1.31.0, 2026-10-14, 142 ms.

## YOLOX-Nano

*YOLOX, anchor-free one-stage detector* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/frame0.jpg) | ![output](gallery/outputs/yolox-nano-street.jpg) |

The smallest YOLOX on the same frame: faster, fewer and less certain boxes. 17 results: 12 person, 3 traffic light, 2 car.

- Input: [Diagonal crosswalk at Yonge and Dundas, Toronto (frame of the video)](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm) by Raysonho, CC0 1.0 (frame at 20 s, resized to 960 × 540).
- Run: version 0.1.1rc0, CPU, ONNX Runtime 1.31.0, 2026-10-14, 22 ms.

## YOLOX-Tiny

*YOLOX, anchor-free one-stage detector* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/frame0.jpg) | ![output](gallery/outputs/yolox-tiny-street.jpg) |

YOLOX-Tiny on the same frame: between Nano and S in speed and recall. 20 results: 9 person, 4 car, 4 traffic light, 2 handbag, 1 backpack.

- Input: [Diagonal crosswalk at Yonge and Dundas, Toronto (frame of the video)](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm) by Raysonho, CC0 1.0 (frame at 20 s, resized to 960 × 540).
- Run: version 0.1.1rc0, CPU, ONNX Runtime 1.31.0, 2026-10-14, 81 ms.

## Grounding DINO Tiny (find objects described in words)

*Grounding DINO, Swin-T image encoder + BERT text encoder, uint8 quantised* · weights licence: Apache-2.0 (onnx-community and IDEA-Research model cards)

| Input | Output |
|---|---|
| ![input](gallery/inputs/frame0.jpg) | ![output](gallery/outputs/grounding-street.jpg) |

Text prompt: 'person . traffic light . car'. Boxes are labelled with the phrase that matched. 6 results: 6 person.

- Input: [Diagonal crosswalk at Yonge and Dundas, Toronto (frame of the video)](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm) by Raysonho, CC0 1.0 (frame at 20 s, resized to 960 × 540).
- Run: version grounding-dino-tiny, onnx-community commit ff690b0, CPU, ONNX Runtime 1.31.0, 2026-10-14, 7123 ms; settings: phrases = ['person', 'traffic light', 'car'].

- Limitations: Several seconds per image on a CPU; small or distant objects are often missed.

## Depth Anything V2 Small (relative depth)

*DPT head on a ViT-S encoder* · weights licence: Apache-2.0 (Small only; Base/Large/Giant are CC-BY-NC-4.0)

| Input | Output |
|---|---|
| ![input](gallery/inputs/frame0.jpg) | ![output](gallery/outputs/depth-street.jpg) |

Relative depth of the same frame (warm is nearer). Relative depth: warm is nearer, cool farther.

- Input: [Diagonal crosswalk at Yonge and Dundas, Toronto (frame of the video)](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm) by Raysonho, CC0 1.0 (frame at 20 s, resized to 960 × 540).
- Run: version commit 4472b73, CPU, ONNX Runtime 1.31.0, 2026-10-14, 2877 ms.

- Limitations: Relative, not metric: values compare points within this image only.

## PP-OCR text detection + English recognition

*PP-OCRv4 DB text detector + PP-OCRv3 SVTR-LCNet CTC recogniser* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/frame0.jpg) | ![output](gallery/outputs/ocr-street.jpg) |

Scene text: detection and English recognition of the signs. 7 results: 7 text Text read: 07 / WFUTURE SHOP / Youcnsualem / ame / adidas.

- Input: [Diagonal crosswalk at Yonge and Dundas, Toronto (frame of the video)](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm) by Raysonho, CC0 1.0 (frame at 20 s, resized to 960 × 540).
- Run: version PP-OCRv4 det / PP-OCRv3 en rec, CPU, ONNX Runtime 1.31.0, 2026-10-14, 433 ms.

- Limitations: Recognition is English-only and struggles with stylised or very small lettering.

## YOLO11n-OBB (aerial, rotated boxes)

*YOLO11 nano with an oriented-box head (centre, size, angle)* · weights licence: AGPL-3.0 (Ultralytics; Enterprise licence sold separately)

| Input | Output |
|---|---|
| ![input](gallery/inputs/usgs_hro_la_2010-crop.jpg) | ![output](gallery/outputs/obb-aerial.jpg) |

Rotated boxes on aerial imagery, following the direction of each vehicle and pool. 36 results: 32 small vehicle, 3 swimming pool, 1 large vehicle.

- Input: [USGS high-resolution orthoimagery, Los Angeles, 2010](https://commons.wikimedia.org/wiki/File:USGS_HRO_16Dec2010_34d06m41.305sN_118d20m10.560sW.jpg) by U.S. Geological Survey, Public domain (U.S. Government work) (960 × 960 crop at full resolution).
- Run: version Ultralytics 8.4.54 export, opset 11, CPU, ONNX Runtime 1.31.0, 2026-10-14, 370 ms.

- Limitations: Trained on DOTA (academic use only); the nano model misses many vehicles; AGPL-3.0 weights.

## Mask R-CNN R-50-FPN (instance segmentation)

*Mask R-CNN, ResNet-50 FPN, per-box 28×28 masks* · weights licence: MIT (maskrcnn-benchmark model zoo; ONNX Model Zoo README carries SPDX MIT)

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000001000.jpg) | ![output](gallery/outputs/maskrcnn-team.jpg) |

Instance segmentation: one editable outline per person and object. 14 results: 13 person, 1 handbag.

- Input: [Tennis team](https://www.flickr.com/photos/impuls-f/4906536419/) by o0bsessed, CC BY 2.0 (resized; model output drawn over it).
- Run: version MaskRCNN-12 (opset 12), CPU, ONNX Runtime 1.31.0, 2026-10-14, 3114 ms.

- Limitations: About four seconds per image on a CPU.

## FCN ResNet-50 (semantic segmentation, int8)

*Fully convolutional network, ResNet-50 backbone, int8 quantised* · weights licence: torchvision pretrained weights (BSD-3-Clause project); ONNX Model Zoo model card SPDX MIT, HF card apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000007281.jpg) | ![output](gallery/outputs/semantic-riders.jpg) |

Semantic segmentation: a class for every pixel (Pascal VOC classes). A class per pixel.

- Input: [Riders on a beach](https://www.flickr.com/photos/kleche/5022207571/) by Phil_Heck, CC BY 2.0 (resized; model output drawn over it).
- Run: version fcn-resnet50-12-int8 (opset 12), CPU, ONNX Runtime 1.31.0, 2026-10-14, 1094 ms.

- Limitations: Objects only (20 classes); sand, sea and sky are background.

## RTMPose-m (person pose, 17 keypoints)

*RTMPose-m, SimCC head, top-down 256×192 (needs a person detector)* · weights licence: Apache-2.0 (released in the MMPose model zoo under the repository licence)

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000000885.jpg) | ![output](gallery/outputs/pose-tennis.jpg) |

Top-down pose: YOLOX-S finds the people, RTMPose places 17 joints on each. 7 results: 7 person.

- Input: [Tennis match](https://www.flickr.com/photos/rhythmstrip/9639200419/) by Edwin Martinez1, CC BY 2.0 (resized; model output drawn over it).
- Run: version rtmpose-m_simcc-body7_pt-body7_420e-256x192-e48f03d0_20230504, CPU, ONNX Runtime 1.31.0, 2026-10-14, 497 ms.

## SlimSAM-77 (interactive segmentation)

*Segment Anything (ViT-B) pruned to 77 % with SlimSAM; prompt encoder + mask decoder* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000020247.jpg) | ![output](gallery/outputs/sam-bear.jpg) |

One click on the nearer bear (green dot) gives its outline. 1 result: 1 polygon.

- Input: [Grizzly bears, Denali National Park](https://www.flickr.com/photos/denalinps/6473127891/) by DenaliNPS, CC BY 2.0 (resized; model output drawn over it).
- Run: version slimsam-77-uniform, CPU, ONNX Runtime 1.31.0, 2026-10-14, 5533 ms; settings: points = [[409, 300]], labels = [1].

- Limitations: Interactive: in the application each click refines the outline.

## ConvNeXt-Tiny (image classification)

*ConvNeXt-T, 224×224* · weights licence: Apache-2.0 (facebook/convnext-tiny-224 card; the Xenova repo has no licence tag of its own)

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000020247.jpg) | ![output](gallery/outputs/classify-bear.jpg) |

Image classification: the five most likely ImageNet classes. Top classes: brown bear, bruin, Ursus arctos (89%).

- Input: [Grizzly bears, Denali National Park](https://www.flickr.com/photos/denalinps/6473127891/) by DenaliNPS, CC BY 2.0 (resized; model output drawn over it).
- Run: version Xenova/convnext-tiny-224 commit 6359b46, CPU, ONNX Runtime 1.31.0, 2026-10-14, 103 ms.

## MobileNetV4-Conv-Small (fast classification)

*MobileNetV4 convolutional, small, 224×224* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000020247.jpg) | ![output](gallery/outputs/classify-bear-fast.jpg) |

The fast classifier on the same photo. Top classes: Arctic fox, white fox, Alopex lagopus (36%).

- Input: [Grizzly bears, Denali National Park](https://www.flickr.com/photos/denalinps/6473127891/) by DenaliNPS, CC BY 2.0 (resized; model output drawn over it).
- Run: version onnx-community/mobilenetv4s-webnn commit 4d0797f, CPU, ONNX Runtime 1.31.0, 2026-10-14, 16 ms.

## IS-Net general use (background removal)

*IS-Net (dichotomous image segmentation), 1024×1024* · weights licence: Not stated separately; DIS repo says 'code and evaluation metric use Apache License 2.0' and links the weights from the README

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000030504.jpg) | ![output](gallery/outputs/matting-skier.jpg) |

Background removal: the skier, poles and skis are kept. Foreground kept, background shown as a checkerboard.

- Input: [Ski touring](https://www.flickr.com/photos/luigimengato/5320611177/) by Luigi Mengato, CC BY 2.0 (resized; model output drawn over it).
- Run: version rembg release v0.0.0, CPU, ONNX Runtime 1.31.0, 2026-10-14, 3229 ms.

- Limitations: The publisher states no separate licence for these weights.

## MODNet (portrait matting)

*MODNet, MobileNetV2 backbone, trimap-free* · weights licence: Apache-2.0 (MODNet README: 'The code, models, and demos ... are released under the Apache License 2.0')

| Input | Output |
|---|---|
| ![input](gallery/inputs/000000021167.jpg) | ![output](gallery/outputs/matting-portrait.jpg) |

Portrait matting with soft edges. Foreground kept, background shown as a checkerboard.

- Input: [Couple at a reception](https://www.flickr.com/photos/chefmattrock/3397730036/) by ChefMattRock, CC BY 2.0 (resized; model output drawn over it).
- Run: version Xenova/modnet commit fa2fa54, CPU, ONNX Runtime 1.31.0, 2026-10-14, 743 ms.

- Limitations: Designed for portraits; people far from the camera may be dropped.

## Docling layout 'heron' (document layout)

*RT-DETRv2, 17 layout classes* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/document_page.jpg) | ![output](gallery/outputs/layout-report.jpg) |

Document layout: titles, paragraphs, lists, table, figure and captions, in reading order. 16 results: 3 section_header, 3 text, 3 list_item, 2 page_header, 2 caption, 1 table.

- Input: Sample report page written and rendered for this project by AI Annotation Workbench (scripts/make_document_sample.py), Original, part of this project.
- Run: version commit 40bde04, opset 18, CPU, ONNX Runtime 1.31.0, 2026-10-14, 1139 ms.

## PP-OCR text detection + English recognition

*PP-OCRv4 DB text detector + PP-OCRv3 SVTR-LCNet CTC recogniser* · weights licence: Apache-2.0

| Input | Output |
|---|---|
| ![input](gallery/inputs/document_page.jpg) | ![output](gallery/outputs/ocr-report.jpg) |

Text lines detected and read on the same page. 40 results: 40 text Text read: Quality Inspection Programme -Annotation Report / Revision 3 / Packaging Defect Dataset: Collection and / Review / 1 Data collection / Field teams photographed 1240 packaging lines across four plants. Each photographwas.

- Input: Sample report page written and rendered for this project by AI Annotation Workbench (scripts/make_document_sample.py), Original, part of this project.
- Run: version PP-OCRv4 det / PP-OCRv3 en rec, CPU, ONNX Runtime 1.31.0, 2026-10-14, 1315 ms.
