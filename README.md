# AI Annotation Workbench

**A desktop workbench for labelling images and videos for computer vision: boxes, rotated boxes, polygons, keypoints
and tags; 16 local models in 11 task families propose annotations, outlines, depth maps, mattes and tracks; you
review and correct them, evaluate models and export to the format your training code expects.**

![Portfolio](https://img.shields.io/badge/portfolio-showcase_only-1F3A5F)
![Source](https://img.shields.io/badge/source-private-6B6B6B)
![Qt](https://img.shields.io/badge/Qt-PyQt5_5.15-41CD52)
![Inference](https://img.shields.io/badge/inference-ONNX_Runtime-005CED)
![Runs on](https://img.shields.io/badge/runs_on-laptop_CPU-6D28D9)

> **About this repository.** It presents the application through screenshots, its design, its model gallery and
> its measured results. It contains no source code: the implementation is private at this stage, and it is built on
> FluentQt Community and FluentQt Pro, which are not publicly available either. The photos shown are credited in
> [Credits](#credits).

![Model Gallery: Grounding DINO compared with YOLOX-S on the same street photo](docs/images/09-gallery-compare-light.png)

## What it is

Labelling is where most computer-vision projects spend their time. Models can help, but only if a person stays in
control of what goes into the dataset. In this workbench every model output is a **proposal**: hatched on the canvas,
attributed to the model and its score, and never exported until someone accepts it. Corrections are ordinary edits
with undo.

| Workspace | |
|---|---|
| **Annotate** | Boxes, **rotated boxes** (rotate with a handle or `[` `]`), polygons, **keypoints** (a guided 17-point person template with visibility per joint) and **image tags**. Run a model on the image: detection, rotated detection, instance and semantic segmentation, **text-prompted detection**, pose, classification, OCR, document layout, depth, background removal; **smart polygon** with a click. **Results**: per-class counts and what they are based on, image tags, raster layers with opacity and legend. **Video**: frames on a timeline, tracking through the video, the trail of every track. |
| **Dataset** | Data grid with thumbnails, status, splits, counts, classes and problems; import images, folders, videos, YOLO, COCO, DOTA and PaddleOCR; validation specific to the export format. |
| **Gallery** | Every model with **its own output on a redistributable photo**, the facts of the run and the photo's licence; two models compared on the same photo; *Try it* runs the model on that photo in your project. |
| **Models** | Catalogue with task, licence notes, sources, sizes and SHA-256; downloads with resume and verification (archives extracted and checked); verify, remove, offline use; the inference runtime as it really is. |
| **Evaluation** | The metric of each task: box, rotated-box, mask and keypoint (OKS) AP, counting error, OCR character error rate, top-k accuracy, mIoU; configuration and environment stored per run. |

## Screenshots

All 16 are the real application with real model output on a laptop CPU.

| | |
|---|---|
| ![Workspace](docs/images/01-workspace-light.png) **Workspace**: current project, its progress and recent projects | ![Detection](docs/images/02-annotate-detection-light.png) **Detection**: 33 YOLOX-S proposals waiting for review, with their provenance |
| ![Dataset](docs/images/03-dataset-light.png) **Dataset**: thumbnails, status, splits, counts and classes | ![Models](docs/images/04-models-light.png) **Models**: catalogue, install state, task, licence and runtime |
| ![Evaluation](docs/images/05-evaluation-light.png) **Evaluation**: YOLOX-S on the 12 reviewed sample images, per-class AP and the run history | ![Settings](docs/images/06-settings-light.png) **Settings**: theme, device, inference runtime and folders |
| ![Instance segmentation](docs/images/07-annotate-segmentation-light.png) **Instance segmentation**: Mask R-CNN outlines as editable polygons | ![OCR](docs/images/08-annotate-ocr-light.png) **OCR**: text regions with their recognised text |
| ![Gallery](docs/images/09-gallery-compare-light.png) **Model Gallery**: Grounding DINO compared with YOLOX-S on the same photo | ![Rotated boxes](docs/images/10-annotate-rotated-light.png) **Rotated boxes**: YOLO11n-OBB on aerial imagery, angle editable |
| ![Pose](docs/images/11-annotate-pose-light.png) **Pose**: RTMPose keypoints with visibility per joint | ![Depth](docs/images/12-annotate-depth-light.png) **Depth**: Depth Anything V2 relative depth layer with its legend |
| ![Matting](docs/images/13-annotate-matting-light.png) **Matting**: MODNet alpha matte, removed background as a checkerboard | ![Counting](docs/images/14-annotate-counting-light.png) **Counting**: per-class counts and what they are based on |
| ![Layout](docs/images/15-annotate-layout-light.png) **Document layout**: Docling regions with reading order | ![Tracking](docs/images/16-annotate-tracking-light.png) **Tracking**: track ids, trails and the frame timeline on a video |

More model outputs, each with its input, run facts and photo licence: **[Model Gallery](docs/gallery.md)**.

## Architecture

```mermaid
flowchart LR
    subgraph App["Application process: PyQt5, FluentQt Community + Pro"]
        UI[Workspace · Annotate · Dataset · Gallery · Models · Evaluation · Settings]
        S[Session<br/>shapes, tags, undo/redo]
        R[Proposal review<br/>class mapping, duplicates]
        F[Formats<br/>YOLO · COCO · DOTA · MOT · PaddleOCR · masks]
        M[Model Manager<br/>catalogue, downloads, archives, hashes]
        E[Evaluation<br/>AP, OKS, counting, OCR, mIoU, MOTA/IDF1]
    end
    subgraph Worker["Inference worker process: no Qt"]
        A[12 adapter families<br/>boxes · rotated · polygons · keypoints · tags · layers]
        T[Tracker<br/>Kalman + two-stage association]
        RT[ONNX Runtime<br/>requested vs actual provider]
    end
    P[(Project file<br/>SQLite, migrations)]
    UI <--> S --> P
    UI <-- JSON lines over stdin/stdout --> Worker
    A --> RT
    A --> T
    M --> W[(Models folder)] --> Worker
```

* **Two processes.** The window never waits for a model, a native crash in a provider ends the worker and not the
  application, and the application works without any AI package installed. Long jobs (video frames) report progress
  and can be cancelled; replies the user has moved past are dropped.
* **One schema, many tasks.** Adapters return boxes, rotated boxes, polygons, keypoints or tags in a common schema, or
  raster layers as lossless files; review, storage, export and evaluation depend only on that. A pose model runs a
  detector first through a helper mechanism; the worker keeps models within a memory budget.
* **Honest reporting.** The worker compares the execution providers it asked for with those that actually run, and
  every screen and record shows the device that ran. The feature matrix separates "implemented", "real inference
  verified", "GUI workflow verified" and "GPU verified" (nothing is GPU-verified yet).
* **Project file with migrations.** One SQLite file per project; older projects upgrade in place; images and video
  frames are referenced, not copied.

More: [architecture](docs/architecture.md) · [feature and capability matrix](docs/feature-matrix.md) ·
[models and licences](docs/models.md) · [formats](docs/formats.md) · [testing](docs/testing.md) ·
[model gallery](docs/gallery.md)

## Models

| Task | Models | Weights licence |
|---|---|---|
| Object detection | YOLOX-Nano, -Tiny, -S | Apache-2.0 |
| Rotated detection (aerial) | YOLO11n-OBB | AGPL-3.0; DOTA data academic use only |
| Instance segmentation | Mask R-CNN R-50-FPN | MIT |
| Semantic segmentation | FCN ResNet-50 (int8) | BSD-3 / MIT |
| Interactive segmentation | SlimSAM-77 | Apache-2.0 |
| Text-prompted detection | Grounding DINO Tiny | Apache-2.0 |
| Pose | RTMPose-m with a YOLOX person detector | Apache-2.0 |
| Classification | ConvNeXt-Tiny, MobileNetV4-Conv-Small | Apache-2.0 |
| OCR | PP-OCRv4 detection + PP-OCRv3 English recognition | Apache-2.0 |
| Document layout | Docling "heron" (RT-DETRv2) | Apache-2.0 |
| Depth | Depth Anything V2 Small | Apache-2.0 |
| Background removal | IS-Net general use, MODNet | not stated by the publisher / Apache-2.0 |

All models are downloaded from their publishers on request and verified against pinned SHA-256 hashes; none are
bundled. Tracking uses the project's own ByteTrack-style tracker on any detector.

## Measured

On an Intel Core i5-1235U laptop (CPU only, ONNX Runtime 1.31), median per image after a warm-up: YOLOX-Nano 22 ms,
YOLOX-S 142 ms, RTMPose with YOLOX-S 0.48 s for 7 people, Mask R-CNN 3.5 s, Grounding DINO 5.4 s (2.9 GB). Full table
with load times and memory: [testing](docs/testing.md#model-latency-and-memory).

- **125 automated tests pass** with real models: every model family through the inference worker on real photos
  (checked against COCO ground truth where it exists) and every task family end to end in the real window.
- **Metrics checked against pycocotools**: box AP, mask AP (within 0.02–0.03) and keypoint AP (equal).
- **Evaluation in the application**, YOLOX-S on the sample's 12 reviewed images: mAP@0.5 0.612, mAP@[.5:.95] 0.491.
  Twelve images make this a pipeline check, not a measurement of YOLOX-S's accuracy.

## Status

Version 0.2, alpha. Verified with real models on CPU, end to end in the application: detection, rotated detection,
semantic segmentation, text-prompted detection, pose, classification, OCR, document layout, depth, matting,
click-to-segment, video frames with tracking, the Model Gallery, exports and evaluation; instance segmentation is
verified through the worker against COCO ground truth.

Not done yet:
- GPU execution: implemented, never run on a GPU (the development machine has none)
- exemplar-based counting: no counting model publishes ONNX weights
- video object segmentation
- VOC and LabelMe formats

## Technology

Python 3.12 · PyQt5 · FluentQt Community and Pro (top navigation window, shape editor with rotated rectangles and
keypoint sets, property inspector, data grid, charts, image compare, video timeline) · ONNX Runtime · OpenCV ·
NumPy · SQLite · Nuitka (Windows standalone build).

## Credits

- **COCO 2017 validation photos** (CC BY 2.0, adapted: resized, model output drawn over them): tennis team by
  [o0bsessed](https://www.flickr.com/photos/impuls-f/4906536419/), riders by
  [Phil_Heck](https://www.flickr.com/photos/kleche/5022207571/), grizzly bears by
  [DenaliNPS](https://www.flickr.com/photos/denalinps/6473127891/), tennis match by
  [Edwin Martinez1](https://www.flickr.com/photos/rhythmstrip/9639200419/), ski touring by
  [Luigi Mengato](https://www.flickr.com/photos/luigimengato/5320611177/), couple by
  [ChefMattRock](https://www.flickr.com/photos/chefmattrock/3397730036/), ribbon cutting by
  [ubrayj02](https://www.flickr.com/photos/ubrayj02/5457068434/). The sample project's other photos, visible in the
  dataset and annotation screenshots, are COCO 2017 photos under CC BY, CC BY-SA, "no known copyright" or US
  government terms; COCO annotations are CC BY 4.0 (COCO Consortium).
- **Street video**: [Diagonal crosswalk at Yonge and Dundas](https://commons.wikimedia.org/wiki/File:DiagonalCrosswalkYongeDundas.webm)
  by Raysonho, CC0 1.0.
- **Aerial photo**: [USGS high-resolution orthoimagery](https://commons.wikimedia.org/wiki/File:USGS_HRO_16Dec2010_34d06m41.305sN_118d20m10.560sW.jpg),
  public domain.
- **Models**: see the [model table](docs/models.md) for publishers and licences.
- The feature set was compared with [X-AnyLabeling](https://github.com/CVHub520/X-AnyLabeling) as a reference for
  what annotation tools offer; no code, assets or text from it are used.

Screenshots and documentation © Phạm Ngọc Trung.
