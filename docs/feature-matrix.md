# Feature and capability matrix

This page records what AI Annotation Workbench does, how each claim was verified, and what is still missing. The
breadth of [X-AnyLabeling](https://github.com/CVHub520/X-AnyLabeling) (GPL-3.0, README of October 2026) is used as
a reference for what an AI-assisted annotation tool offers; nothing from it is used here.

## Status vocabulary

Every row uses the highest level it has actually reached; levels are never merged.

| Status | Meaning |
|---|---|
| **Planned** | Not implemented. |
| **Adapter implemented** | Code exists; no automated test exercises it with a real model yet. |
| **Unit-tested** | Deterministic tests with synthetic data (geometry, formats, metrics, protocol). |
| **Real inference verified** | The real model ran through the inference worker on real photos in the test suite, checked against ground truth where it exists (COCO), structurally otherwise. |
| **GUI workflow verified** | An end-to-end test drives the real window and the real worker process through the workflow. |
| **GPU verified** | Ran on a GPU execution provider. **No model has reached this level**: the development machine has no supported GPU (see [GPU](#gpu-and-hardware)). |
| **Unsupported / blocked** | Deliberately not offered, or blocked by something outside the project (named in the row). |

## Model and task capability matrix

All models run with ONNX Runtime in the worker process, are downloaded on request (never bundled) and are verified
against the SHA-256 in the catalogue. "Device support" is what the code requests; only the CPU execution provider has
been exercised. Latency and memory: [testing](testing.md#model-latency-and-memory).

| Task | Model | Input → output | Source, version | Download, checksum source | Weights licence | Status | Known limitations |
|---|---|---|---|---|---|---|---|
| Object detection | YOLOX-Nano / Tiny / S | image → boxes (80 COCO classes) | Megvii YOLOX 0.1.1rc0 | 3.7 / 20 / 36 MB, pinned | Apache-2.0 | GUI workflow verified | Closed set of 80 classes |
| Rotated detection | YOLO11n-OBB | aerial image → rotated boxes (15 DOTA classes), tiled at 640 px | Ultralytics 8.4.54 export (AXERA-TECH mirror, pinned commit) | 10.8 MB, Hugging Face | **AGPL-3.0**; DOTA data academic use only | GUI workflow verified | Nadir aerial imagery only; the nano model misses many small vehicles; licence restrictions shown before download |
| Instance segmentation | Mask R-CNN R-50-FPN | image → polygons (80 COCO classes) | ONNX Model Zoo MaskRCNN-12 | 178 MB, Hugging Face | MIT | Real inference verified (median mask IoU ≥ 0.7 vs COCO) | ~4–5 s per image on CPU; instances in several pieces keep the largest piece (count reported in `parts`); holes are not represented |
| Semantic segmentation | FCN ResNet-50 (int8) | image → class map layer (Pascal VOC, 21) | ONNX Model Zoo fcn-resnet50-12-int8 | 35.5 MB, Hugging Face | BSD-3 / MIT | GUI workflow verified | Object classes only (no sky, road…); no permissive ADE20K ONNX model exists |
| Interactive segmentation | SlimSAM-77 int8 | image + clicks / box → polygon | SlimSAM, Xenova ONNX (pinned) | 13.8 MB, Hugging Face | Apache-2.0 | GUI workflow verified | First click per image runs the encoder (~3–7 s on CPU) |
| Text-prompted detection | Grounding DINO Tiny (uint8) | image + phrases → boxes labelled with phrases | IDEA Research, onnx-community (pinned) | 195 MB, Hugging Face (+ vocabulary, pinned) | Apache-2.0 | GUI workflow verified | ~5–10 s and up to 3 GB per image on CPU; small objects often missed |
| Pose estimation | RTMPose-m + YOLOX detector | image → 17 COCO keypoints per person | OpenMMLab MMDeploy SDK zip (ONNX extracted on install) | 51 MB zip, pinned | Apache-2.0 (training sets include research-only data) | GUI workflow verified | Top-down: people the detector misses get no pose |
| Classification | ConvNeXt-Tiny; MobileNetV4-Conv-Small | image → ranked tags (ImageNet-1k) | Meta / timm, Hugging Face exports (pinned) | 114.5 / 15 MB, Hugging Face | Apache-2.0 | GUI workflow verified (ConvNeXt); real inference verified (MobileNetV4) | ImageNet classes only |
| OCR | PP-OCRv4 det + PP-OCRv3 en rec | image → text regions (quads or rotated boxes) + text, reading order | PaddleOCR / RapidOCR ONNX | 13.7 MB, Hugging Face | Apache-2.0 | GUI workflow verified | English recognition; stylised scene text is often misread |
| Document layout | Docling "heron" (RT-DETRv2) | page → 17 region classes with reading order | docling-project ONNX (pinned) | 171 MB, Hugging Face | Apache-2.0 | GUI workflow verified | Reading order is top-to-bottom, left-to-right (no column detection) |
| Depth | Depth Anything V2 Small | image → relative inverse depth layer (16-bit) | onnx-community (pinned) | 99 MB, Hugging Face | Apache-2.0 (Small only) | GUI workflow verified | Relative, not metric; normalised per image |
| Matting / background removal | IS-Net general use; MODNet | image → alpha layer | rembg release; Xenova MODNet (pinned) | 179 / 26 MB, pinned / Hugging Face | IS-Net: no separate weight licence stated; MODNet Apache-2.0 | GUI workflow verified (MODNet); real inference verified (both: alpha IoU ≥ 0.75 vs the COCO person outline) | IS-Net ~3–6 s on CPU; MODNet is for portraits |
| Tracking | ByteTrack-style tracker (this project) on any detector | video frames → boxes with track ids | clean-room implementation of the published algorithm | — | GPL-3.0-or-later (this project) | GUI workflow verified | No re-identification features: long occlusions start new identities |
| Counting | detection- or mask-derived counts | annotations → per-class counts | this project | — | — | GUI workflow verified | One count per annotation; not a density estimate |
| Counting by exemplar | GeCo / GeCo2 / CountGD / DAVE / LOCA / CounTR | — | — | — | — | **Blocked**: none of them publishes ONNX weights (checked 2026-10-14); a self-made export is planned | — |
| Video object segmentation (SAM 2 and similar) | — | — | — | — | — | Planned | |
| 3D point clouds | — | — | — | — | — | Unsupported | needs its own geometry, viewer and interaction model |
| Vision-language (captioning, VQA) | — | — | — | — | — | Unsupported | |

## Annotation

| Capability | What it does | Status | Evidence |
|---|---|---|---|
| Boxes and polygons | Draw, select, move, resize, edit/insert/delete vertices, change class | GUI workflow verified | `test_app_e2e` (mouse drawing through the view transform); Pro `ShapeEditor` tests |
| Rotated boxes | Draw (O), resize in the box's own frame, rotate with the handle (Shift: 15° steps) or `[` `]`, edit centre/size/angle in Properties; stored as an exact rectangle | GUI workflow verified | `test_rotated_box_is_drawn_rotated_saved_and_undone`; Pro shape-editor tests for drawing, frame-aligned resize, rotation handle and keys |
| Keypoints | Draw a 17-point COCO person (K) with a guided template (Space skips an occluded joint, Enter finishes early), drag joints, set visibility (visible / occluded / not labelled) per joint | GUI workflow verified | `test_keypoints_follow_the_template_and_visibility_is_editable` |
| Image tags (classification) | Tag the whole image with a class, accept or remove proposed tags; undoable on the image's undo stack | GUI workflow verified | `test_image_tag_is_added_undone_and_counted` |
| Raster layers | Semantic class maps, relative depth and alpha mattes shown under the regions with opacity, legend, accept/remove; "To polygons" outlines a layer's regions as editable proposals | GUI workflow verified | `test_semantic_layer_is_shown_and_outlined_by_the_worker` |
| Undo / redo | Every edit, accept-all, reject-all and tag change is one step; persisted | GUI workflow verified | `test_app_e2e`, rotated-box test |
| Track ids | Any annotation can carry a track id (Properties); tracking fills them; trails of the last 30 frames on the canvas | GUI workflow verified | `test_video_frames_are_added_and_tracked_with_identities` |
| Masks as raster annotations | Instance masks are stored as polygons (largest piece), semantic maps as layers | Implemented as described | multi-piece instances and holes are not represented exactly |

## Video

| Capability | What it does | Status | Evidence |
|---|---|---|---|
| Add a video | Extract every n-th frame (up to a maximum) in the worker, with progress and cancel; frames keep their index and time | GUI workflow verified | `test_frames_are_extracted_…`, video e2e |
| Navigate | Frames in order in the image list; timeline (Pro `VideoTimeline`) with markers for annotated frames; click to jump; A/D step | GUI workflow verified | video e2e |
| Track through a video | Detector on every frame from the open one, boxes linked by the tracker into tracks (proposals to review) | GUI workflow verified | video e2e; `test_detector_and_tracker_link_people_across_real_video_frames` (CC0 street video) |
| Export tracks | MOT Challenge `gt.txt` + `seqinfo.ini` per video | Unit-tested | `test_mot_writes_tracked_boxes_only` |

## Model Gallery and Model Manager

| Capability | Status | Evidence |
|---|---|---|
| Gallery of every catalogue model with its real output on a redistributable photo (input, output, model version, device, ONNX Runtime version, date, latency, licence of the photo); filters by task, installed state and verified example; compare two models on the same photo (Pro `ImageCompare`); "Try it in Annotate" adds the photo to the project and runs the model | GUI workflow verified | `test_gallery_…` (both), 17 examples built by `scripts/build_gallery.py` |
| Download with progress, resume, cancel; size and SHA-256 verification; files extracted from a downloaded ZIP with their own hash; verify, remove, offline use | Unit-tested and real (all 16 models installed through the store from the cache with hash checks) | `test_models.py`, `test_files_inside_a_downloaded_archive_are_extracted_and_verified` |
| Helper models (pose runs a detector first) | GUI workflow verified | `test_each_task_family_runs_from_the_annotate_page[rtmpose…]` |
| Memory budget in the worker (least recently used models released above 40 % of RAM) | Unit-tested | `test_worker_releases_old_models_over_its_memory_budget_…` |
| Device report (requested, tried, actual provider, fallback reason) | Real inference verified on CPU | e2e |

## Data formats

| Format | Read | Write | Notes |
|---|---|---|---|
| YOLO detect / segment | ✓ | ✓ | rotated boxes become axis-aligned boxes in detect (reported as "rotation lost") |
| YOLO OBB | ✓ (`task: obb`) | ✓ | exact for rotated boxes; boxes at 0°; polygons as minimum-area rotated boxes (reported) |
| YOLO pose | ✓ (`kpt_shape`) | ✓ | visibility kept; `flip_idx` written for COCO-17 |
| COCO instances + keypoints | ✓ | ✓ | rotated boxes as polygons plus `attributes.rotation` (read back as rotated); keypoints with COCO categories and 1-based skeleton |
| DOTA | ✓ | ✓ | four corners, class, difficulty |
| MOT Challenge | (tracks file reader) | ✓ | tracked boxes on video frames only |
| PaddleOCR `Label.txt` | ✓ | ✓ | `###` for regions without transcription |
| Semantic masks (PNG label maps) | — | ✓ | from accepted semantic layers, else rasterised regions |
| Depth / alpha layers (PNG) | — | ✓ | 16-bit depth with a JSON note on its meaning; alpha plus RGBA cut-out |
| Classification CSV | — | ✓ | one row per tag, untagged images listed |
| VOC XML, LabelMe JSON, ODVG | — | — | Planned |

Every export is preceded by a validation report specific to the chosen format; every conversion or omission is
counted in the export report (`test_formats_tasks.py`).

## Evaluation

| Metric | Status | Evidence |
|---|---|---|
| Box AP (mAP@0.5, mAP@[.5:.95]), precision, recall | Unit-tested against pycocotools | `test_metrics.py` |
| Mask AP (pixel IoU of polygons) | Unit-tested, agrees with pycocotools `segm` within 0.02–0.03 AP | `test_mask_ap_agrees_with_pycocotools_segm` |
| Keypoint AP (OKS) | Unit-tested, equal to pycocotools `keypoints` | `test_keypoint_ap_agrees_with_pycocotools_oks` |
| Rotated-box AP (exact polygon IoU) | Unit-tested | `test_rotated_ap_counts_an_unrotated_guess_as_a_miss` |
| Counting MAE / RMSE / bias | Unit-tested; reported with every detector evaluation | `test_counting_error` |
| OCR detection P/R, CER, end-to-end CER, exact match | Unit-tested | `test_ocr_accuracy_…` |
| Top-1 / top-5 accuracy, semantic mIoU and pixel accuracy | Unit-tested | `test_classification_semantic_and_tracking_metrics` |
| Tracking MOTA, IDF1, identity switches | Unit-tested (synthetic tracks) | as above; no annotated tracking ground truth in the test data, so no tracker accuracy is claimed |
| Evaluation runs in the application per task | GUI workflow verified for box AP | `test_app_e2e`; the other tasks share the runner but have no end-to-end test with ground truth yet |

## GPU and hardware

GPU execution providers (CUDA, DirectML, Core ML) are requested in order and the provider that actually ran is
reported, including the reason for a fallback to CPU. None of this has run on a GPU: the development machine (Intel
Core i5-1235U, Iris Xe) has no CUDA device and DirectML was not tested. To verify on a GPU machine:

run the project's benchmark with `onnxruntime-gpu` and the CUDA device setting; 
The `device` column of the output must say `CUDAExecutionProvider`; a `CPUExecutionProvider` there means the run
fell back and is not a GPU result.
