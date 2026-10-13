# Testing

The suite lives in the maintainers' workspace (`devkit/tests/annotation-workbench/`), not in this repository. It runs
with the offscreen Qt platform:

```bash
QT_QPA_PLATFORM=offscreen python -m pytest devkit/tests/annotation-workbench
```

Real-model tests use model files from `ANNOTATION_WORKBENCH_TEST_MODELS` (default `~/.cache/annotate-test-models`)
and a COCO val2017 subset from `ANNOTATION_WORKBENCH_TEST_DATA` (built by `scripts/make_coco_subset.py`); they are
skipped, not faked, when either is missing. No test substitutes a mock prediction for a model. The download tests
use a simulated HTTP transport, so dropped connections, servers without range support and corrupted bodies can be
reproduced.

## What is tested

| File | Covers |
|---|---|
| `test_core.py` | Geometry (YOLO conversions, IoU, clipping, polygon area and simplification); annotation validation; image status; project create, save, reopen; refusal of foreign and newer project files; class rename, conflict, removal with reassignment; model proposals replaced rather than accumulated; bounded inference history |
| `test_review.py` | Label → class mapping (by name, saved choices, stale choices); proposals with provenance; class creation on acceptance; duplicate predictions skipped; folder import with unreadable files kept and re-imports skipped; the validation report's codes |
| `test_formats.py` | YOLO detection export (normalised boxes, polygons converted and reported); YOLO segmentation round trip; YOLO import errors and clipping; `data.yaml` name forms; COCO round trip; sparse COCO category ids and problem reports; missing files and unclassified annotations skipped; OCR transcriptions in COCO attributes; duplicate file names renamed |
| `test_metrics.py` | AP for perfect predictions; ranking and duplicate false positives; crowd regions; classes without ground truth; latency and histograms; **agreement with pycocotools on real COCO ground truth** |
| `test_models.py` | The catalogue is valid and complete; unsafe entries rejected (unknown adapters, path traversal in file names, non-HTTPS URLs, malformed hashes, missing licence); custom models; downloads verify size and SHA-256, resume after a dropped connection, restart when ranges are ignored, delete corrupted files, keep partial files on cancel; install, verify, remove |
| `test_engine_real.py` | **Real models through the worker:** YOLOX-Nano, -Tiny and -S on COCO photos, checked against ground truth (boxes inside the image, scores above the threshold, most large objects found, provider and timings reported); PP-OCR detection and recognition on rendered text, and detection only; SlimSAM outlines a clicked object (IoU with the ground-truth box > 0.7); a click on a small part still outlines the whole object, with the parts as alternatives; embedding reuse; errors for bad images and missing files without the worker dying; device fallback reported; the JSON-lines process protocol |
| `test_app_e2e.py` | **The real application:** open the sample project, detect on an image with the real worker, accept all, undo and redo, draw a box with mouse events, mark reviewed, outline an object with SlimSAM and keep it, Tab through alternative outlines, close and reopen, evaluate against reviewed images, export COCO and YOLO and read them back; missing image reported with its annotations kept; batch inference and cancel; the dataset grid |

## Results

Run on 2026-10-13: Intel Core i5-1235U (12 logical CPUs, no GPU), WSL2 Ubuntu, Python 3.12, PyQt5 5.15,
ONNX Runtime 1.31 (CPU execution provider), OpenCV 5.0.

**58 passed, 0 skipped, in 38 s** (all real-model tests ran: model cache and COCO subset present).

Latency through the worker, median over 20 COCO val2017 photos after the model is loaded (reading, preprocessing,
inference, postprocessing):

| Model | Median | Notes |
|---|---|---|
| YOLOX-Nano (416 px) | 20 ms | 17 ms median in the application's evaluation run below |
| YOLOX-Tiny (416 px) | 56 ms | |
| YOLOX-S (640 px) | 219 ms | |
| SlimSAM-77 int8, first click on an image | 3.0 s | runs the image encoder; the embedding is cached |
| SlimSAM-77 int8, further clicks | 0.10 s | decoder only |
| PP-OCR detection + recognition | 0.31 s | one COCO photo (a bear, no text: detection cost only) |

Evaluation inside the application on the sample project (YOLOX-Nano, default thresholds, the 12 reviewed images
with COCO ground truth, 67 objects): **mAP@0.5 0.429, mAP@[.5:.95] 0.312, precision 0.809, recall 0.567**. Twelve
images make this a check that the evaluation pipeline works end to end, not a measurement of YOLOX-Nano's
accuracy; the detection metrics themselves are checked against pycocotools in `test_metrics.py`.

**Packaged build (Windows 11, Nuitka 4.2.2, Python 3.11, ONNX Runtime 1.30):** `deploy/build.py` produced a
423 MB standalone folder without model weights. Every workspace opened and closed by itself (exit 0), and the packaged
worker loaded YOLOX-Nano and found the person and the skis in COCO image 785 (153 ms on the CPU execution provider).
The crash log stayed empty. The first two builds failed this test: one excluded a Pro module the canvas needs, and
the other started the worker through a `python.exe` that does not exist in a Nuitka build. Both are fixed.

## Clean-room check

The project was compared token by token with a shallow clone of X-AnyLabeling (554 Python files), using the
workspace's provenance checker (24-token windows of normalised code, plus comment and docstring lines). It found
no shared code and no shared comments.

## Manual checks

- The application started with ONNX Runtime and OpenCV blocked in its process and the worker running on an
  interpreter without OpenCV: the window opens and the worker's runtime check names `opencv-python-headless` as
  missing. Project, import, validation and export code does not import either package (they are imported only in
  the worker).
- Downloads of all five catalogue models from their publishers, checked against the pinned sizes and SHA-256
  (2026-10-13).
