# Feature matrix

The comparison below uses [X-AnyLabeling](https://github.com/CVHub520/X-AnyLabeling)'s documented capabilities
(README, October 2026) as a reference for what an AI-assisted annotation tool offers. It records what AI Annotation
Workbench does today and how each claim was verified. X-AnyLabeling is GPL-3.0; nothing from it is used here.

Statuses: **Verified** (implemented and covered by automated tests, real models where a model is involved) ·
**Implemented** (works, not fully verified) · **Planned** · **Not supported** · **Not applicable**.

## Annotation

| Capability (reference) | AI Annotation Workbench | Status | Modules | Evidence |
|---|---|---|---|---|
| Rectangles | Draw, select, move, resize, change class, delete; stored in image pixels | Verified | `ui/canvas.py` (Pro `ShapeEditor`), `session.py` | `test_app_e2e` draws a box with mouse events through the view transform and checks its class and storage; FluentQt Pro's `ShapeEditor` tests cover drawing in image pixels, moving and resizing with the mouse, keyboard nudge, selection and handles at any zoom |
| Polygons | Draw, close, move, edit vertices, insert vertices | Verified | as above | `test_core`, `test_formats`, e2e smart polygon accepted as a polygon; Pro's `ShapeEditor` tests cover polygon drawing, vertex drag, insert and delete |
| Undo / redo | Every edit, accept-all and reject-all is one undoable step; persisted after redo | Verified | `session.py` | `test_app_e2e` (undo and redo of accept-all, then persistence) |
| Zoom, pan, fit, image navigation, class visibility | Wheel zoom, middle-drag pan, F fit, A/D previous/next, L labels, class filter | Implemented | `ui/annotate.py` | e2e opens images and maps points through the view transform; the shortcuts are wired as actions listed in the command palette, not individually tested |
| Keyboard shortcuts | V/R/P/S tools, 1–9 classes, N next proposal, Enter accept, Delete reject, Ctrl+Z/Y, Ctrl+R detect, Ctrl+K commands | Implemented | `ui/annotate.py`, `ui/window.py` | listed in the command palette |
| Rotated boxes | — | Planned | | needs a rotated-box primitive in the canvas and DOTA export |
| Points, lines, polylines, circles, cuboids | — | Not supported | | |
| Masks (raster) | Model masks are converted to polygons | Not supported as raster | `engine/ops.py` | |
| Image-level tags / classification | — | Planned | | |
| Keypoints / pose | — | Planned | | needs keypoint geometry with visibility and skeletons |
| Video annotation, tracking | — | Planned | | needs frame handling and track identities |
| Point clouds, text/multimodal (VQA, captioning) | — | Not supported | | |

## AI assistance

| Capability (reference) | AI Annotation Workbench | Status | Evidence |
|---|---|---|---|
| Object detection | YOLOX-Nano, YOLOX-Tiny, YOLOX-S (Megvii ONNX releases); confidence and NMS thresholds | Verified | `test_engine_real` runs Nano, Tiny and S on COCO photos and checks them against ground truth; e2e runs Nano in the app |
| Proposal review | Proposals are hatched and never exported until accepted; accept/reject one or all; next proposal; edit before accepting | Verified | `test_app_e2e` |
| Class mapping | Model label → project class, new class, or ignore; saved per model | Verified | `test_review`; e2e accepts proposals that create classes |
| Duplicate predictions | Predictions overlapping an accepted annotation of the same class are skipped (IoU threshold in Settings) | Verified | `test_review` |
| Batch inference | Selected or unreviewed images; progress; cancel; per-image failures counted, not fatal | Verified | `test_batch_inference_runs_and_can_be_cancelled` |
| Interactive segmentation (Segment Anything family) | SlimSAM-77: positive/negative clicks, box prompt, embedding cache per image, alternative outlines (Tab) | Verified | `test_engine_real` (bear, horse-and-rider), e2e smart polygon |
| OCR | PP-OCRv4 text detection + PP-OCRv3 English recognition; detection-only mode; text regions are a separate `text` kind with transcription | Verified | `test_engine_real` on rendered text |
| Instance segmentation models (YOLO-seg, D-FINE-seg) | Polygons are supported end to end; no instance segmentation model is integrated yet | Planned | |
| Rotated detection, pose, tracking, grounding (Grounding DINO, YOLO-World), depth, classification, VLMs | — | Planned (grounding, rotated, pose) · Not supported (others) | |
| Custom models | A YOLOX-format ONNX detector can be registered with its class names | Verified (registry) · Implemented (dialog) | `test_custom_models_are_added_and_invalid_ones_skipped` |
| Remote inference server | — | Not supported (local inference only) | |

## Models and runtime

| Capability | AI Annotation Workbench | Status | Evidence |
|---|---|---|---|
| Registry-driven model catalogue | `catalog/models.json`: task, architecture, runtime, adapter, sources, licences, files with sizes and SHA-256, requirements, providers, platforms, I/O, labels, defaults | Verified | `test_models` parses and validates every entry |
| Download with progress, cancel, resume | `.part` files, HTTP range resume (restart when the server ignores ranges), size and hash check before rename, HTTPS only | Verified | `test_models` (simulated transport): dropped connection, resume, server without ranges, corruption, cancel; real downloads of every catalogue model from the publishers checked against the pinned hashes (docs/testing.md) |
| Checksums | SHA-256 pinned per file; a missing checksum is shown as unavailable | Verified | `test_models` |
| Revalidate, remove, open folder, offline use | Installed models work without a network | Verified | `test_models`, e2e uses installed models only |
| Runtime compatibility | Worker probe: packages, versions, execution providers, CPU, memory; missing packages explained | Verified | `test_engine_real`; the application started with OpenCV absent and the probe named it (docs/testing.md) |
| Device reporting | Requested device, providers tried, provider that actually ran, fallback reason | Verified on CPU | e2e checks `CPUExecutionProvider` is recorded; GPU not available on the test machine |
| GPU (CUDA, DirectML) | Provider lists are implemented; fallback is detected | Implemented, not verified | no GPU on the development machine |
| TensorRT, OpenCV DNN, vLLM/SGLang | — | Not supported | |

## Data

| Capability (reference) | AI Annotation Workbench | Status | Evidence |
|---|---|---|---|
| Project format | SQLite `*.awproj`, versioned migrations, refuses newer and foreign files | Verified | `test_core` |
| Import images and folders | JPEG, PNG, BMP, TIFF, WebP, folders recursively; unreadable files kept with their error; re-import skips known files | Verified | `test_review` |
| YOLO import/export | Detection and segmentation label files, `data.yaml`, splits; boxes ↔ polygons conversions reported | Verified | `test_formats` round trips; e2e export and re-import |
| COCO import/export | Detection and polygon segmentation, category id mapping, splits, crowd flag | Verified | `test_formats` round trips; e2e export |
| VOC, DOTA, MOT, PPOCR, masks, ODVG, VLM formats | — | Planned (VOC, DOTA, PPOCR) · Not supported (others) | |
| Validation before export | Missing and unreadable images, annotations without class, degenerate and out-of-image geometry, near-duplicates, tiny boxes, proposals still pending, and what each format will transform or drop | Verified | `test_review`, `test_formats`, e2e missing-image test |
| Dataset statistics | Class distribution, coverage, confidence histogram, splits | Verified | e2e renders the evaluation page |

## Evaluation

| Capability | AI Annotation Workbench | Status | Evidence |
|---|---|---|---|
| Detection mAP@0.5, mAP@[.5:.95], precision, recall | Against reviewed images; per-class AP; configuration and environment stored | Verified | `test_metrics` matches pycocotools on real COCO ground truth |
| Latency, throughput, failures, worker memory | From every recorded inference | Verified | e2e |
| Self-labelling warning | Ground truth accepted from the evaluated model is counted and reported | Verified | e2e |
| Segmentation and OCR metrics | — | Planned | |
