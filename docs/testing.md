# Testing

The test suite is part of the private source; it runs with Qt's offscreen platform and real models.


Real-model tests use model files from `ANNOTATION_WORKBENCH_TEST_MODELS` (default `~/.cache/annotate-test-models`)
and data from `ANNOTATION_WORKBENCH_TEST_DATA` (default `~/.cache/annotate-test-data`: a COCO val2017 subset built
by `scripts/make_coco_subset.py`, a 12-second excerpt of a CC0 street video, a USGS public-domain orthophoto). Models
are installed into a temporary models folder through the real model store, so their hashes are checked and archives
are extracted exactly as after a download. Tests are skipped, not faked, when files are missing. No test substitutes
a mock prediction for a model.

## Verification levels

| Level | What it means here | Files |
|---|---|---|
| 1. Unit | Geometry (rotated boxes, exact rotated IoU, minimum-area boxes, polygon rasterising), annotation kinds, project storage and migration 2, review and proposals, formats, metrics, tracker (Hungarian assignment checked against brute force), download and archive logic, worker memory budget | `test_core`, `test_geometry_tasks`, `test_formats`, `test_formats_tasks`, `test_metrics`, `test_metrics_tasks`, `test_models`, `test_review` |
| 2. Model contract | The tokenizer matches the reference library's ids; every reply survives the JSON encoding the application receives | `test_engine_models` |
| 3. Real inference | Every model family through the worker on real photos; checked against COCO ground truth where it exists | `test_engine_real`, `test_engine_models` |
| 4. GUI end to end | The real window and the real worker process: every task family from the Run panel, rotated-box and keypoint editing, tags, layers, video frames and tracking, the Model Gallery, evaluation, import/export | `test_app_e2e`, `test_app_tasks_e2e` |
| 5. Hardware and packaging | CPU only (no GPU available); Windows standalone build with a packaged smoke test | `deploy/build.py`, see below |

## What the real-inference tests check

| Model | Check |
|---|---|
| YOLOX-Nano / Tiny / S | Boxes inside the image, scores above the threshold, most large COCO objects found |
| Mask R-CNN | ≥ 75 % of the people found; median mask IoU with the COCO outlines ≥ 0.7 |
| FCN ResNet-50 | A label-map layer of the image's size; the horses' COCO outlines are ≥ 60 % *horse* |
| RTMPose-m + YOLOX-S | ONNX file extracted from the downloaded ZIP; one pose per COCO person; all visible keypoints inside the person's box |
| Grounding DINO | The dog found with score ≥ 0.5 and IoU ≥ 0.7 with its COCO box; labels are the prompt's phrases |
| Depth Anything V2 | Relative, larger-is-nearer metadata; the ground in front is nearer than the sky |
| IS-Net, MODNet | Alpha ≥ 128 overlaps the COCO person outline with IoU ≥ 0.75 |
| ConvNeXt, MobileNetV4 | The bear photo is tagged *brown bear* first with ≥ 0.5 |
| Docling layout | On the project's sample page: table, picture, two captions, three list items, footer, in reading order |
| YOLO11n-OBB | Tiled inference finds over ten times more vehicles than the whole image at once; boxes take several angles |
| PP-OCR | Detection and recognition on rendered text; detection only |
| SlimSAM | A click outlines the object (IoU with the box > 0.7); a click on a small part still gives the whole object |
| Tracker on a real video | YOLOX-S over 30 frames of a CC0 street video: no track id twice in a frame, most tracks last 10+ frames (structure only: there is no annotated tracking ground truth, so no tracking accuracy is claimed) |

The tracking and mask/keypoint metrics are checked separately: mask AP agrees with pycocotools `segm` within
0.02–0.03 AP (the two rasterise polygons differently), keypoint AP equals pycocotools `keypoints`.

## Results

Run on 2026-10-14: Intel Core i5-1235U (12 logical CPUs, Iris Xe, no CUDA GPU), WSL2 Ubuntu, Python 3.12, PyQt5
5.15, ONNX Runtime 1.31 (CPU execution provider), OpenCV 5.0.

- **Application: 125 passed, 0 skipped** in 111 s (all model files and test data present).
- **FluentQt Pro: 402 passed** (includes the 24 `ShapeEditor` tests for rotated rectangles and keypoint sets). Two
  timing-sensitive Pro tests failed while a Nuitka build was compiling on the same machine and passed when it was idle.
- `ruff check --select F,E9`: clean for the application and Pro.

### Model latency and memory

`scripts/benchmark_models.py` on an idle machine: each model in a fresh worker process on its gallery photo, one
warm-up request, then 5 timed requests (reading, preprocessing, inference, postprocessing). Memory is the worker's
resident set with only that model (and its helper) loaded. CPU execution provider throughout.

| Model | Image | Median | Fastest | Load | Worker memory |
|---|---|---|---|---|---|
| YOLOX-Nano | 960 × 540 | 22 ms | 20 ms | 101 ms | 107 MB |
| YOLOX-Tiny | 960 × 540 | 45 ms | 43 ms | 134 ms | 136 MB |
| YOLOX-S | 960 × 540 | 142 ms | 142 ms | 229 ms | 201 MB |
| YOLO11n-OBB (tiled, 4 tiles) | 960 × 960 | 360 ms | 321 ms | 303 ms | 178 MB |
| Mask R-CNN R-50-FPN | 640 × 480 | 3,527 ms | 3,347 ms | 1,860 ms | 1,098 MB |
| FCN ResNet-50 int8 | 640 × 361 | 1,047 ms | 1,014 ms | 304 ms | 314 MB |
| SlimSAM-77 (encoder and decoder each request) | 640 × 457 | 2,472 ms | 2,456 ms | 578 ms | 3,556 MB |
| Grounding DINO Tiny uint8 (3 phrases) | 960 × 540 | 5,370 ms | 5,277 ms | 1,766 ms | 2,908 MB |
| RTMPose-m + YOLOX-S (7 people) | 640 × 427 | 483 ms | 448 ms | 630 ms | 354 MB |
| ConvNeXt-Tiny | 640 × 457 | 83 ms | 79 ms | 608 ms | 288 MB |
| MobileNetV4-Conv-Small | 640 × 457 | 16 ms | 16 ms | 214 ms | 112 MB |
| PP-OCR det + rec | 960 × 540 | 227 ms | 225 ms | 330 ms | 291 MB |
| Docling layout "heron" | 679 × 960 | 1,025 ms | 902 ms | 1,083 ms | 520 MB |
| Depth Anything V2 Small | 960 × 540 | 2,530 ms | 2,323 ms | 655 ms | 834 MB |
| IS-Net general use | 480 × 640 | 2,285 ms | 1,999 ms | 951 ms | 1,536 MB |
| MODNet | 426 × 640 | 539 ms | 515 ms | 267 ms | 430 MB |

In the application, SlimSAM runs its encoder once per image and later clicks take about 0.1 s (the benchmark gives
no cache key, so every request includes the encoder). These are single-image latencies on one laptop CPU, for
comparing models and planning work, not accuracy figures. Tracking added about 10 ms per frame on top of the
detector (median over 60 frames of the street video).

### Evaluation in the application

YOLOX-S on the sample project's 12 reviewed COCO images (67 objects, default thresholds): **mAP@0.5 0.612,
mAP@[.5:.95] 0.491, precision 0.725, recall 0.746, count MAE 0.09**, median latency 139 ms (screenshot
`05-evaluation`). Twelve images make this a check that the pipeline works end to end, not a measurement of
YOLOX-S's accuracy.

### Packaged build

See the Windows build section at the end of this page.

## Clean-room check

The application and FluentQt Pro were compared with a shallow clone of X-AnyLabeling (commit a7d2765, 2026-10-13,
554 Python files) with the workspace's provenance checker (24-token windows of normalised code, plus comment and
docstring lines) and byte-for-byte for images and JSON files. Result: no shared comments, no identical assets, and
five 24-token matches, all generic Python idioms in two maintainer scripts (`ROOT = Path(__file__).resolve().parents[N];
sys.path.insert(0, str(ROOT))` in `build_gallery.py`, and a run of `parser.add_argument(…)` calls in
`capture_screenshots.py`). The model adapters, tracker, metrics and UI share no code with it.

## Manual checks

- Every new catalogue file was downloaded from its catalogue URL and checked against its size and hash
  (2026-10-14); Hugging Face hashes match the host's LFS hashes, the others were recorded on download.
- All 16 models were installed through the model store from those files (hash checks, ZIP extraction) and run on the
  gallery photos (`scripts/build_gallery.py`); the 17 gallery outputs were inspected.
- The 16 documentation screenshots were captured from the application with real model output and inspected one by
  one; scenes that showed ground truth instead of model output, faint layers or stale messages were fixed and
  recaptured.
- The licences of the gallery photos were checked on Flickr on 2026-10-14; two photos whose page no longer showed a
  CC licence were replaced.
