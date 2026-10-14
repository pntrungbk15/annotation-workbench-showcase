# Dataset formats

Exports contain accepted annotations only; proposals that were never reviewed are not exported. Every export starts
with a validation report that lists what is wrong, and what the chosen format will transform or omit. Problems
marked as errors are skipped individually; nothing is exported silently into a form that cannot hold it. Exports
need an empty target folder, so an existing dataset is never overwritten.

## Geometry conventions

- Coordinates are image pixels, x to the right and y downwards, from the top-left corner of the top-left pixel, in
  the image's stored orientation (EXIF rotation is ignored everywhere, so annotations and model inputs agree).
- **Rotated boxes** are `(cx, cy, w, h, angle)`: centre, width along the box's own x axis, height, and the angle in
  degrees from the image x axis to the width edge, **clockwise on screen**, normalised to (−180, 180]. Their corners
  are p0, p1, p2, p3 with p0→p1 the width edge (p0 is the top-left corner at 0°). The project stores the four
  corners and rebuilds an exact rectangle from them on every edit, so conversions never accumulate skew.
- **Keypoints** follow a template (COCO-17 for people) with one visibility per point: 0 not labelled, 1 labelled but
  occluded, 2 labelled and visible. A point with visibility 0 has no meaningful coordinates and is exported as
  `0, 0, 0`.
- **Raster layers** have the image's size: semantic maps are 8-bit class indexes with their label list, depth is
  16-bit relative inverse depth rescaled per image (larger is nearer, not metric), alpha is 8-bit opacity.

## YOLO (Ultralytics layout)

```
data.yaml                 path, train / val / test image folders, names
images/<split>/name.jpg   <split> is train, val or test; no split folders when no split is assigned
labels/<split>/name.txt   one line per object
```

| | Detection export | Segmentation export |
|---|---|---|
| Line | `class cx cy w h` | `class x1 y1 x2 y2 …` |
| Coordinates | normalised to 0–1 by image width and height | same |
| Boxes | written as boxes | written as four-corner polygons (reported) |
| Polygons | written as their bounding boxes (reported) | written as polygons |
| Text transcriptions | dropped (reported) | dropped (reported) |
| Class index | position of the class in the project's class list | same |

Two more tasks use the same layout:

| | OBB export | Pose export |
|---|---|---|
| Line | `class x1 y1 x2 y2 x3 y3 x4 y4` (corners in order) | `class cx cy w h px1 py1 v1 …` |
| `data.yaml` | `task: obb` | `task: pose`, `kpt_shape: [17, 3]`, `flip_idx` for COCO-17 |
| Rotated boxes | exact | skipped (reported) |
| Boxes | rotated boxes at 0° (reported) | skipped (reported) |
| Polygons, text regions | minimum-area rotated boxes (reported) | skipped (reported) |
| Keypoints | skipped (reported) | with visibility; unlabelled points as `0 0 0` |

In detection exports rotated boxes become their axis-aligned bounding boxes and the report says the rotation was
lost; in segmentation exports they become four-corner polygons. Image tags are never written to YOLO files.

Import reads `data.yaml` (`names` as a list, block list or mapping), finds images and labels per split, accepts
detection and segmentation lines in the same file, reads OBB lines when `task: obb` is set (fitting the smallest
rotated box around corners from other tools) and pose lines when `kpt_shape` is set, clips geometry to the image,
and reports malformed lines, unknown class indexes, polygons without area and images without labels.

## COCO (object detection and instance segmentation)

```
annotations/instances_<split>.json   instances_default.json when no split is assigned
images/<split>/name.jpg              when images are copied
```

| | |
|---|---|
| Boxes | `bbox: [x, y, width, height]` in pixels from the top-left corner; empty `segmentation` |
| Polygons | `segmentation: [[x1, y1, x2, y2, …]]` with their `bbox` and `area` |
| Category ids | class positions starting at 1 in the export; any ids (COCO's own skip numbers) are mapped on import |
| Crowd | `iscrowd` is kept from imports |
| Text transcriptions | stored in `attributes.text` (an extension; reported) |

| Rotated boxes | four-corner `segmentation` plus `attributes.rotation = [cx, cy, w, h, angle]` (read back as rotated boxes) |
| Keypoints | `keypoints` (x, y, v triples) and `num_keypoints`; categories with keypoints list the 17 COCO names and the 1-based `skeleton` |
| Image tags | not written (COCO instances has no image labels; use the classification CSV) |

Import keeps the largest part of a multi-part polygon (reported), skips RLE masks (reported), reads keypoints, and
maps sparse category ids to project classes.

## DOTA (rotated boxes)

`labelTxt/name.txt` with `x1 y1 x2 y2 x3 y3 x4 y4 class difficult` per object (pixels), images in `images/`. Spaces
in class names are written as hyphens. Boxes are written at 0° and polygons as minimum-area rotated boxes (both
reported); keypoints and tags are skipped. Import fits the smallest rotated box around each quadrilateral and ignores
the `imagesource:` / `gsd:` header lines.

## MOT Challenge (tracks)

One folder per video with `gt/gt.txt` (`frame, id, x, y, w, h, 1, class, 1`, frames numbered from 1 in extraction
order) and `seqinfo.ini` (frame rate of the extracted frames, size, and `frameStep`, so frame *n* is video frame
(n − 1) × step); `classes.txt` maps the class column to names. Only accepted boxes with a track id on video frames are
written; everything else is counted in the report.

## PaddleOCR labels

`Label.txt` with `images/name.jpg<TAB>[{"transcription", "points", "difficult"}]` per image. Text regions and rotated
boxes that carry text are written with their four corners (text polygons with more vertices as minimum-area rotated
boxes, reported); a region without transcription is written as `###`, PaddleOCR's marker for illegible text.

## Semantic masks, raster layers and classification

- **Semantic masks**: `masks/name.png` (8-bit class index per pixel, 0 = background) and `classes.json` with index,
  name and colour. A mask comes from the image's accepted semantic layer (model labels matched to project classes by
  name; other labels get their own indexes after the project classes, reported), otherwise from its regions
  rasterised in drawing order (boxes as rectangles, reported).
- **Depth and alpha layers**: accepted layers as PNG, depth as 16-bit with a JSON note on what the values mean, alpha
  as 8-bit plus an RGBA cut-out of the image.
- **Classification CSV**: `classification.csv` with `file, label, score, source`, one row per accepted tag; images
  without a tag are listed with an empty label.

## Checks before export

| Code | Meaning |
|---|---|
| `missing-file`, `image-error` | The image is missing or unreadable; it is skipped, its annotations stay in the project |
| `no-class` | An annotation has no class; it is skipped |
| `outside`, `partly-outside` | Geometry outside the image is skipped; geometry crossing the border is clipped |
| `degenerate`, `tiny` | Polygons without area are skipped; boxes under 2 px are flagged |
| `duplicate` | Two annotations of the same class almost coincide (IoU > 0.95) |
| `pending-proposals` | Proposals not reviewed yet; they are not exported |
| `transform` | What the format changes: polygons to boxes, boxes to polygons, rotation lost, polygons to rotated boxes, transcriptions dropped |
| `skipped` | Kinds the chosen format cannot hold (keypoints in DOTA, tags in COCO, untracked boxes in MOT…) |

Duplicate image file names from different folders are renamed in the export instead of overwriting each other.
