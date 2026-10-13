# Models

Models are not part of the application. The Model Manager (Models workspace) downloads them on request from their
publishers into the models folder (Settings → Models folder; by default inside the user's application-data folder),
verifies them, and from then on works offline.

## Catalogue

| Model | Task | Runtime | Download | Weights licence | Code licence | Source |
|---|---|---|---|---|---|---|
| YOLOX-Nano | Object detection, 80 COCO classes, 416 px | ONNX Runtime | 3.7 MB | Apache-2.0 | Apache-2.0 | [Megvii YOLOX 0.1.1rc0](https://github.com/Megvii-BaseDetection/YOLOX/releases/tag/0.1.1rc0) |
| YOLOX-Tiny | Object detection, 80 COCO classes, 416 px | ONNX Runtime | 20.2 MB | Apache-2.0 | Apache-2.0 | as above |
| YOLOX-S | Object detection, 80 COCO classes, 640 px | ONNX Runtime | 35.9 MB | Apache-2.0 | Apache-2.0 | as above |
| PP-OCR (PP-OCRv4 detection + PP-OCRv3 English recognition) | Text detection and recognition | ONNX Runtime | 13.7 MB | Apache-2.0 | Apache-2.0 | [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) models, ONNX conversion by [RapidOCR](https://huggingface.co/SWHL/RapidOCR) (pinned commit) |
| SlimSAM-77 (int8) | Interactive segmentation (click, box) | ONNX Runtime | 13.8 MB | Apache-2.0 | Apache-2.0 | [SlimSAM](https://github.com/czg1225/SlimSAM), from Segment Anything; ONNX export [Xenova/slimsam-77-uniform](https://huggingface.co/Xenova/slimsam-77-uniform) (pinned commit) |

The detectors were trained on COCO 2017 (annotations CC BY 4.0, images under their Flickr licences). Every model
above permits commercial use under Apache-2.0; the Models workspace shows each model's licence, its terms and the
link to the licence text before download.

Models were chosen for permissive licences that suit both GPL and commercial distribution. Popular alternatives
were left out on purpose: Ultralytics YOLOv5/v8/11 weights and code are AGPL-3.0 (or an Ultralytics commercial
licence), which would bind every user of a hosted or distributed build to AGPL terms.

## Downloads

- Only HTTPS URLs are accepted, and only from the catalogue (or a file the user picks for a local model). Nothing
  is executed and no installer is run.
- A file downloads to `<name>.part`; it is renamed only after its size and SHA-256 match the catalogue. Hugging Face
  URLs are pinned to a commit, so the content behind them cannot change.
- An interrupted download resumes with an HTTP range request; a server that ignores ranges restarts the file.
- Cancel keeps the partial file for a later resume. A corrupted file (wrong size or hash) is deleted and reported.
- **Verify files** re-hashes an installed model. Each file's details say where its SHA-256 comes from (recorded for
  this catalogue, or published by the host). A file without a published checksum is shown as checked by size only,
  never as verified by hash.
- **Remove** asks for confirmation, deletes the model's folder and keeps every annotation made with it.

Verified on 2026-10-13: all five catalogue models downloaded from the URLs above on a home connection and matched
their pinned sizes and hashes.

## Runtime and devices

Inference runs in a separate worker process with ONNX Runtime (`pip install .[inference]`). The application
starts without it and explains what is missing.

| Setting | Providers tried, in order |
|---|---|
| CPU | CPU |
| CUDA (NVIDIA) | CUDA, CPU |
| DirectML (Windows GPUs) | DirectML, CPU |
| Automatic | CUDA, DirectML, Core ML, CPU |

The worker reports the requested device, the providers it tried, the provider that actually ran, and why it fell
back (provider not in this ONNX Runtime build, or its libraries failed to load). GPU providers need the matching
ONNX Runtime package (`onnxruntime-gpu` with CUDA and cuDNN, or `onnxruntime-directml`); the application does not
install them. Verified so far: CPU on Linux (WSL2) and Windows. GPU execution has not been verified, because the
development machine has no supported GPU.

Settings → Inference Python can point the worker at another interpreter, for example one that has a GPU build of
ONNX Runtime.

## Local detectors

**Add a local detector** registers a YOLOX-format ONNX export (input `1×3×S×S` BGR 0–255, raw head output) with
its class names in output order. The file is copied into the models folder. Its licence is the user's
responsibility; the Model Manager shows its licence as "set by the user".

## Adding a model

- **Same family**, for example another YOLOX size: add an entry to `annotation_workbench/catalog/models.json` with
  its files, sizes, SHA-256, licence, labels and default parameters. The catalogue test checks that every entry is
  complete and safe.
- **New family**: write an adapter in `annotation_workbench/engine/adapters/` that implements `load`, `predict`
  and `close` and returns predictions in the common schema (see [architecture](architecture.md)). Register it in
  `engine/adapters/__init__.py` and add a real-model test to the suite.
