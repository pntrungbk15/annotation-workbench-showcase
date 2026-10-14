# Architecture

AI Annotation Workbench is two processes: a FluentQt desktop application that owns the project and every
interaction, and an inference worker that owns the models. They talk over the worker's standard input and output,
one JSON object per line.

```
┌──────────────────────────── application process (PyQt5, FluentQt Community + Pro) ────────────────────────────┐
│ ui/            window (TopNavigationWindow) · workspace · annotate (ShapeEditor canvas, results, video bar)     │
│                dataset (DataGrid) · gallery (real model outputs) · models · evaluation · settings               │
│ session.py     the open image's annotations and tags, undo/redo, debounced writes                              │
│ project/       *.awproj SQLite file (fluentqt.storage, migrations 1–2), import, validation, sample             │
│ formats/       YOLO (detect/segment/OBB/pose), COCO, DOTA, MOT, PaddleOCR, PNG masks/layers, CSV; report       │
│ inference/     InferenceClient (QProcess), batch and tracking runner, applying replies, review, class mapping  │
│ models/        catalogue (registry, helpers), ModelStore (download, resume, verify, archives, remove)          │
│ evaluation/    box/rotated/mask/OKS AP, counting, OCR, classification, mIoU, tracking metrics; runs per task   │
│ core/          annotation kinds, geometry (rotated boxes, rasterising), skeletons, assignment, settings       │
└───────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────┘
                                                │ JSON lines: probe · predict · frames · vectorize · cancel · unload · quit
┌───────────────────────────────── inference worker (no Qt) ─────────────────────────────────────────────────────┐
│ engine/worker.py     request loop (reader thread for cancel); errors become replies; memory budget for models  │
│ engine/runtime.py    ONNX Runtime sessions, requested vs. actual execution provider, image decoding, probe      │
│ engine/adapters/     yolox · yolo-obb · maskrcnn · semantic · sam · grounding-dino · rtmpose · classifier ·     │
│                      ppocr · rtdetr-layout · depth · matting                                                    │
│ engine/tracking.py   ByteTrack-style tracker (Kalman filter, two-stage IoU association, Hungarian matching)     │
│ engine/video.py      frame extraction · engine/text.py BERT WordPiece · engine/ops.py letterbox, NMS, masks    │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## Why a separate worker process

- **The window never blocks.** Model loading and inference take from tens of milliseconds to seconds on a CPU;
  they run in another process, and the UI receives results as signals from a `QProcess`.
- **Optional dependencies stay optional.** The application process never imports ONNX Runtime or OpenCV. Without
  them, projects, manual annotation, import, validation and export work; the Models workspace explains what is
  missing (the worker's `probe` lists missing packages).
- **Native crashes are contained.** A provider that crashes (a broken GPU driver, for example) ends the worker, not
  the application; the client reports it and starts a new worker with the next request.
- **Windows DLL order.** PyQt5 ships an older MSVC runtime than ONNX Runtime needs; keeping them in different
  processes avoids the conflict. A packaged build starts the worker as `AnnotationWorkbench.exe --worker`.

Requests carry an id. The application drops replies to requests it no longer wants (the user moved to another image,
a batch was cancelled), so a stale result is never displayed.

## Model adapters

Every model family implements `engine.base.Adapter`:

| Method | Responsibility |
|---|---|
| `load(device)` | Create ONNX Runtime sessions and return a `DeviceReport`: requested device, providers tried, providers actually active, fallback reason. |
| `predict(image, params)` | Preprocess, run, postprocess; return predictions in the common schema and per-stage timings. |
| `close()` | Release sessions and caches. |

The common prediction schema is small, one dictionary per result:

| Kind | Fields |
|---|---|
| `box` | `box: [x, y, w, h]` |
| `polygon`, `text` | `points: [[x, y], …]`, `text` for OCR |
| `rotated` | `rotated: [cx, cy, w, h, angle]`, angle in degrees clockwise on screen from the x axis to the width edge |
| `keypoints` | `points`, `visibility` (COCO 0/1/2), `keypointScores`, `skeleton`, `box` of the person |
| `tag` | `label`, `score`, `rank` (image classification) |

plus `label` and `score`, and optional `trackId`, `readingOrder`, `prompt`. Raster results (semantic class maps,
relative depth, alpha mattes) are written by the adapter as lossless PNG files into a folder the application names,
and described in the reply's `layers`; the application stores them in the project and deletes the files. Everything
after the worker (review, persistence, export) depends only on this schema.

Some adapters need another model first: a request can carry `helpers` (RTMPose gets a YOLOX detector this way, and
the worker keeps both loaded). A request with `track` sends box predictions through the tracker of a session, which
keeps identities across the frames it receives in order; the application drives the frames one at a time, so a
long video can be cancelled after any frame.

Adapters are registered by name in `engine/adapters/__init__.py` and imported lazily. The catalogue
(`catalog/models.json`) declares for every model: id, task, architecture, runtime, adapter, version, source and
documentation URLs, weights licence and code licence, files with URL, size and SHA-256 (and where the hash comes
from), Python requirements, execution providers, platforms, input and output description, labels and default
parameters. Adding a model of an existing family is a catalogue entry; adding a family is one adapter module.

The worker keeps up to four recently used models loaded and releases the least recently used one when it holds more,
or when its memory passes a budget (40 % of RAM by default): Grounding DINO alone needs about 2–3 GB on a CPU. Models
of the request in progress (a pose model and its detector) are never released to make room. The SAM adapter also
caches image embeddings for the last three images, so further clicks on the same image only run the small decoder.

Replies are JSON, so NumPy values from an adapter are converted when the reply is encoded, and a reply that still
cannot be encoded becomes an error reply: the worker never ends because of one result. Long requests (extracting
video frames) send `progress` events and stop on `cancel`, which a reader thread receives while the request runs;
everything else is answered in order.

## Execution providers

The device setting (CPU, CUDA, DirectML, automatic) becomes an ordered provider list. ONNX Runtime silently drops a
provider whose libraries fail to load, so the worker compares the providers it asked for with
`session.get_providers()` and reports both, plus the reason for a fallback. The annotate panel, evaluation records
and inference history show the provider that actually ran.

## Project file

A project is one SQLite file (`*.awproj`) opened through `fluentqt.storage`:

- `PRAGMA application_id` identifies the file type; `PRAGMA user_version` with numbered migrations versions the
  schema. A newer application migrates older projects; an older application refuses a newer project instead of
  damaging it.
- Tables: `meta`, `classes`, `images` (paths relative to the project folder when possible, size, reviewed flag, error,
  split, and for video frames the video, frame index and time), `annotations` (kind, class, points, source
  human/model, state proposed/accepted, score, model id, text, attributes such as visibility or track id), `layers`
  (raster results: kind, model, state, PNG data, description), `videos` (source, frame rate, frame count, size, step,
  frames folder), `inference_runs` (per-stage timings and provider of every inference), `evaluations` (metrics,
  configuration and environment of every evaluation run).
- Migration 2 added `layers`, `videos` and the frame columns; a version-1 project is upgraded in place (after a backup
  copy) and keeps all its annotations.
- Each logical operation is one transaction. The session writes edits shortly after they happen and at once when
  the image changes or the project closes, so a crash loses at most the last edit.
- Image files are referenced, not copied. A missing or unreadable image is marked with an error; its annotations
  are kept and the validation report lists it.

## Annotation kinds

`box`, `polygon`, `text` (polygon with a transcription), `rotated` (four corners, always an exact rectangle),
`keypoints` (one point per template keypoint with COCO visibilities; COCO-17 is the built-in template) and `tag`
(image-level class, no geometry). Tags are not shapes, so the session keeps them beside the image's shape model and
records their changes as commands on the same undo stack. Raster layers are not annotations: they are results that a
person accepts (to export them) or outlines into polygons.

## Annotation states

An annotation is `proposed` (from a model, not yet reviewed, hatched on the canvas, never exported) or `accepted`.
Its `source` (`human` or `model`), `modelId` and `score` stay with it after acceptance, so evaluation can report how
much of the ground truth came from the model being evaluated. Polygons drawn with click-to-segment are `human`
annotations with `assistedBy` in their attributes.

## Proposal review

`inference/review.py` turns predictions into proposals:

- **Class mapping.** A model's label maps to an existing project class, to "create a class with this name when the
  first proposal is accepted", or to "ignore". Defaults match names case-insensitively; choices are saved per model.
- **Duplicates.** A prediction that overlaps an accepted annotation of the same class by at least the overlap set in
  Settings (IoU 0.7 by default) is not proposed again; the panel says how many were skipped. Re-running a model on a
  reviewed image therefore only proposes what is new.
- **Acceptance.** Accept, reject, accept all and reject all are single undoable steps.

## FluentQt components in use

| Need | Component | Package |
|---|---|---|
| Main window with workspaces | `TopNavigationWindow`, `CommandPalette` (Ctrl+K), `NotificationCenter` | Pro |
| Annotation canvas: boxes, rotated boxes, polygons, keypoint sets with skeletons, selection, move/resize/rotate, vertex editing, zoom, pan | `ShapeEditor`, `ShapeModel`, `Shape` (rotated rectangles and point sets were added to Pro for this application; the canvas adds prompt points, suggestion outlines, raster layers and track trails) | Pro |
| Model Gallery: input/output and model/model comparison | `ImageCompare`, `FilterBar`, `CardWidget`, `FlowLayout` | Pro, Community |
| Video frames | `VideoTimeline`, `TimelineMarker` | Pro |
| Annotation, class and model properties | `PropertyInspector`, `Property` | Pro |
| Dataset table with filter, sort, multi-selection; evaluation history | `DataGrid` | Pro |
| Dataset statistics, confidence histogram, latency | `ChartView`, `Series`, `histogramSeries`, `PieChart`, `MetricStrip` | Pro |
| Model list and details | `CardListView`, `CardListItem`, `FilterBar`, `ModelInfoCard`, `ThresholdSlider` (renamable label added to Pro) | Pro |
| Progress of long jobs; workflow steps | `ProgressDialog`, `StepIndicator` | Pro |
| Project storage | `fluentqt.storage.Database`, `Migration` | Community |
| Buttons, command bars, settings cards, dialogs, info bars, empty states, theme | Community widgets | Community |

FluentQt Pro's node editor is not used: a fixed preprocessing → model → postprocessing chain per adapter covers the
supported models, and a graph editor would add configuration without adding a capability.
