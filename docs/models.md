# Models

Models are not part of the application. The Model Manager (Models workspace) downloads them on request from their
publishers into the models folder (Settings → Models folder; by default inside the user's application-data folder),
verifies them, and from then on works offline.

## Catalogue

16 models in 11 task families. Sizes are download sizes; licences are those of the weights (code licences and notes
are in the Models workspace and in `catalog/models.json`).

| Model | Task | Download | Weights licence | Source (pinned) |
|---|---|---|---|---|
| YOLOX-Nano / Tiny / S | Object detection, 80 COCO classes | 3.7 / 20.2 / 35.9 MB | Apache-2.0 | [Megvii YOLOX 0.1.1rc0](https://github.com/Megvii-BaseDetection/YOLOX/releases/tag/0.1.1rc0) |
| YOLO11n-OBB | Rotated detection, 15 DOTA aerial classes | 10.8 MB | **AGPL-3.0**, DOTA data academic use only | Ultralytics export mirrored at [AXERA-TECH/yolo11-obb](https://huggingface.co/AXERA-TECH/yolo11-obb) |
| Mask R-CNN R-50-FPN | Instance segmentation, 80 COCO classes | 178 MB | MIT | [ONNX Model Zoo MaskRCNN-12](https://huggingface.co/onnxmodelzoo/MaskRCNN-12) |
| FCN ResNet-50 int8 | Semantic segmentation, Pascal VOC | 35.5 MB | BSD-3 / MIT | [ONNX Model Zoo fcn-resnet50-12-int8](https://huggingface.co/onnxmodelzoo/fcn-resnet50-12-int8) |
| SlimSAM-77 int8 | Interactive segmentation (clicks, box) | 13.8 MB | Apache-2.0 | [SlimSAM](https://github.com/czg1225/SlimSAM), [Xenova/slimsam-77-uniform](https://huggingface.co/Xenova/slimsam-77-uniform) |
| Grounding DINO Tiny uint8 | Text-prompted (open-vocabulary) detection | 195 MB + vocabulary | Apache-2.0 | [IDEA Research](https://github.com/IDEA-Research/GroundingDINO), [onnx-community](https://huggingface.co/onnx-community/grounding-dino-tiny-ONNX) |
| RTMPose-m (Body7) | Person pose, 17 COCO keypoints (runs a YOLOX detector first) | 50.8 MB zip | Apache-2.0 (training data includes research-only sets) | [OpenMMLab MMPose](https://github.com/open-mmlab/mmpose/tree/main/projects/rtmpose) ONNX SDK |
| ConvNeXt-Tiny | Image classification, ImageNet-1k | 114.5 MB | Apache-2.0 | [facebook/convnext-tiny-224](https://huggingface.co/facebook/convnext-tiny-224), Xenova export |
| MobileNetV4-Conv-Small | Fast image classification, ImageNet-1k | 15.1 MB | Apache-2.0 | [timm](https://huggingface.co/timm/mobilenetv4_conv_small.e2400_r224_in1k), onnx-community export |
| PP-OCRv4 det + PP-OCRv3 en rec | Text detection and recognition | 13.7 MB | Apache-2.0 | [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR), RapidOCR ONNX |
| Docling layout "heron" | Document layout, 17 region classes | 171 MB | Apache-2.0 | [docling-project](https://huggingface.co/docling-project/docling-layout-heron-onnx) |
| Depth Anything V2 Small | Monocular relative depth | 99 MB | Apache-2.0 (Small only) | [Depth Anything V2](https://github.com/DepthAnything/Depth-Anything-V2), onnx-community export |
| IS-Net general use | Background removal | 179 MB | not stated separately by the publisher | [DIS](https://github.com/xuebinqin/DIS), ONNX by [rembg](https://github.com/danielgatis/rembg) |
| MODNet | Portrait matting | 25.9 MB | Apache-2.0 | [MODNet](https://github.com/ZHKKKe/MODNet), Xenova export |

Each model was downloaded from the URL in the catalogue, checked against its hash and run on real photos before
it was added (2026-10-14). Where the host publishes a SHA-256 (Hugging Face LFS) the catalogue says so; otherwise
the hash was recorded when the file was verified for the catalogue, and the Models workspace says that too.

### Licences

The application is GPL-3.0-or-later and downloads models only when the user asks, showing each model's licence and
notes first. The catalogue prefers permissive weights; three entries need care, and say so in the application:

- **YOLO11n-OBB** is AGPL-3.0 (Ultralytics). It is the only oriented detector with downloadable ONNX weights that was
  found: RTMDet-R, PP-YOLOE-R and Oriented R-CNN publish PyTorch or Paddle weights only. Every DOTA-trained model is
  also bound by DOTA's academic-use-only terms. Drawing and exporting rotated boxes does not need this model.
- **IS-Net general use**: the DIS repository is Apache-2.0 and links these weights without a separate licence; the
  DIS5K dataset has its own terms. MODNet (Apache-2.0) is the clearly licensed alternative for portraits.
- **RTMPose and the ImageNet classifiers** were trained partly on datasets with research-oriented terms (AI
  Challenger, MPII, Halpe, PoseTrack; ImageNet).

Models not added on purpose: Depth Anything V2 Base/Large and SegFormer (non-commercial licences), MaskFormer
(CC-BY-NC), MobileViT segmentation (Apple research licence), BiRefNet-lite (about 60 s and 6.6 GB per image on a
laptop CPU), and exemplar counters (GeCo, CountGD, DAVE, LOCA, CounTR: no ONNX weights published).

### Helpers

A model can name helper models it runs first (`"helpers"` in its catalogue entry). RTMPose needs a person
detector: the first installed of YOLOX-S, YOLOX-Tiny and YOLOX-Nano runs on the image, and each person box is
cropped for the pose network. The Run panel names the helper it will use, or which one to install.

### Model Gallery

The Gallery workspace shows each model's real output on a photo the project may redistribute: COCO 2017 val photos
under CC BY 2.0 (attributed to their Flickr pages), a CC0 street video by Raysonho on Wikimedia Commons, a USGS
public-domain orthophoto, and a report page written for this project. `scripts/build_gallery.py` produces every
output with the inference worker and records the model version, device, ONNX Runtime version, date and latency
(after a warm-up run) in `annotation_workbench/gallery/gallery.json`; `scripts/gallery_plan.json` lists the inputs,
their sources and the settings. A model without an example says so; nothing is illustrated by hand.

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

Verified on 2026-10-13 (the first five models) and 2026-10-14 (the other eleven): every file downloaded from its
catalogue URL and matched its size and hash. Files inside an archive (RTMPose's ONNX file in OpenMMLab's ZIP) are
extracted after the archive is verified, and checked against their own size and hash; only the named member is read.

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

The Model Manager can register a YOLOX-format ONNX detector the user trained, with its class names; its licence is the user's responsibility.
