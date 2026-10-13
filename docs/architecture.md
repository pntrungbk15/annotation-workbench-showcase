# Architecture

AI Annotation Workbench is two processes: a FluentQt desktop application that owns the project and every
interaction, and an inference worker that owns the models. They talk over the worker's standard input and output,
one JSON object per line.

```
┌──────────────────────────── application process (PyQt5, FluentQt Community + Pro) ────────────────────────────┐
│ ui/            window (TopNavigationWindow) · workspace · annotate (ShapeEditor canvas) · dataset (DataGrid)    │
│                models (Model Manager) · evaluation (charts) · settings                                          │
│ session.py     the open image's annotations, undo/redo, debounced writes                                       │
│ project/       *.awproj SQLite file (fluentqt.storage, numbered migrations), import, validation, sample        │
│ formats/       YOLO and COCO readers and writers, transformation report                                        │
│ inference/     InferenceClient (QProcess), batch runner, proposal review and class mapping                     │
│ models/        catalogue (registry), ModelStore (download, resume, verify, remove), download thread           │
│ evaluation/    COCO-style detection metrics, latency statistics, evaluation runs                               │
│ core/          annotation model, geometry, settings                                                            │
└───────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────┘
                                                │ JSON lines: probe · predict · unload · quit
┌───────────────────────────────── inference worker (no Qt) ─────────────────────────────────────────────────────┐
│ engine/worker.py     request loop; every failure becomes an error reply, the worker keeps serving              │
│ engine/runtime.py    ONNX Runtime sessions, requested vs. actual execution provider, image decoding, probe      │
│ engine/adapters/     yolox (detection) · ppocr (text detection + recognition) · sam (click-to-segment)          │
│ engine/ops.py        letterbox, NMS, mask → polygon                                                             │
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

The common prediction schema is small: `{"kind": "box", "label", "score", "box": [x, y, w, h]}` or
`{"kind": "polygon", "label", "score", "points": [[x, y], ...]}`, with an optional `text` for OCR. Everything after
the worker (review, persistence, export) depends only on this schema.

Adapters are registered by name in `engine/adapters/__init__.py` and imported lazily. The catalogue
(`catalog/models.json`) declares for every model: id, task, architecture, runtime, adapter, version, source and
documentation URLs, weights licence and code licence, files with URL, size and SHA-256 (and where the hash comes
from), Python requirements, execution providers, platforms, input and output description, labels and default
parameters. Adding a model of an existing family is a catalogue entry; adding a family is one adapter module.

The worker keeps the three most recently used models loaded and releases the oldest. The SAM adapter also caches
image embeddings for the last three images, so further clicks on the same image only run the small decoder.

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
  split), `annotations` (kind, class, points, source human/model, state proposed/accepted, score, model id, text,
  attributes), `inference_runs` (per-stage timings and provider of every inference), `evaluations` (metrics,
  configuration and environment of every evaluation run).
- Each logical operation is one transaction. The session writes edits shortly after they happen and at once when
  the image changes or the project closes, so a crash loses at most the last edit.
- Image files are referenced, not copied. A missing or unreadable image is marked with an error; its annotations
  are kept and the validation report lists it.

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
| Annotation canvas: boxes, polygons, selection, move/resize, vertex editing, zoom, pan | `ShapeEditor`, `ShapeModel`, `Shape` (extended with prompt points and suggestion outlines for click-to-segment) | Pro |
| Annotation, class and model properties | `PropertyInspector`, `Property` | Pro |
| Dataset table with filter, sort, multi-selection; evaluation history | `DataGrid` | Pro |
| Dataset statistics, confidence histogram, latency | `ChartView`, `Series`, `histogramSeries`, `PieChart`, `MetricStrip` | Pro |
| Model list and details | `CardListView`, `CardListItem`, `FilterBar`, `ModelInfoCard`, `ThresholdSlider` | Pro |
| Progress of long jobs; workflow steps | `ProgressDialog`, `StepIndicator` | Pro |
| Project storage | `fluentqt.storage.Database`, `Migration` | Community |
| Buttons, command bars, settings cards, dialogs, info bars, empty states, theme | Community widgets | Community |

FluentQt Pro's node editor is not used: a fixed preprocessing → model → postprocessing chain per adapter covers the
supported models, and a graph editor would add configuration without adding a capability.
