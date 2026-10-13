# Dataset formats

Exports contain accepted annotations only; proposals that were never reviewed are not exported. Every export starts
with a validation report that lists what is wrong, and what the chosen format will transform or omit. Problems
marked as errors are skipped individually; nothing is exported silently into a form that cannot hold it. Exports
need an empty target folder, so an existing dataset is never overwritten.

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

Import reads `data.yaml` (`names` as a list, block list or mapping), finds images and labels per split, accepts
detection and segmentation lines in the same file, clips geometry to the image, and reports malformed lines, unknown
class indexes, polygons without area and images without labels.

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

Import keeps the largest part of a multi-part polygon (reported), skips RLE masks (reported) and maps sparse
category ids to project classes.

## Checks before export

| Code | Meaning |
|---|---|
| `missing-file`, `image-error` | The image is missing or unreadable; it is skipped, its annotations stay in the project |
| `no-class` | An annotation has no class; it is skipped |
| `outside`, `partly-outside` | Geometry outside the image is skipped; geometry crossing the border is clipped |
| `degenerate`, `tiny` | Polygons without area are skipped; boxes under 2 px are flagged |
| `duplicate` | Two annotations of the same class almost coincide (IoU > 0.95) |
| `pending-proposals` | Proposals not reviewed yet; they are not exported |
| `transform` | What the format changes: polygons to boxes, boxes to polygons, transcriptions dropped |

Duplicate image file names from different folders are renamed in the export instead of overwriting each other.
