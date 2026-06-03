---
name: rembg
description: >
  Use this skill whenever the user wants to remove the background from an image or video, make
  backgrounds transparent, extract foregrounds, or produce images/videos with alpha channels.
  Triggers include: "remove background", "transparent background", "cut out subject",
  "去除背景", "去背景", "透明背景", "抠图", "视频去背景", "background removal", "alpha channel",
  "remove bg", "cutout image", any mention of rembg. Also triggers when user uploads an image/video
  and asks to isolate the subject, or produce PNG/WebP with transparent background.
  Use for both single images and batch folder processing, and for video frame pipelines via FFmpeg.
source: https://github.com/danielgatis/rembg
license: MIT
---

# rembg — Background Removal Skill

## Overview

`rembg` removes backgrounds from images (and videos via FFmpeg) using AI segmentation models.
It supports CPU and GPU (CUDA / ROCm), runs as a CLI or Python library, and outputs transparent PNG / WebP.

Python requirement: **>=3.11, <3.14**

---

## Step 0 — Check / Install rembg

Always run the install check first. Use `--break-system-packages` with pip.

```python
import subprocess, sys

def ensure_rembg():
    try:
        import rembg  # noqa: F401
    except ImportError:
        subprocess.run(
            [sys.executable, "-m", "pip", "install", "rembg[cpu,cli]",
             "--break-system-packages", "-q"],
            check=True
        )

ensure_rembg()
```

> **GPU users (NVIDIA CUDA):** replace `rembg[cpu,cli]` with `rembg[gpu,cli]`
> **AMD ROCm:** install `onnxruntime-rocm` first, then `rembg[rocm,cli]`

---

## Single Image — Remove Background

```python
from rembg import remove
from PIL import Image
import os

input_path  = "/mnt/user-data/uploads/photo.jpg"   # <- uploaded file
output_path = "/mnt/user-data/outputs/photo_no_bg.png"

img    = Image.open(input_path)
result = remove(img)                  # returns RGBA PIL Image
result.save(output_path)

print(f"Saved: {output_path}  size={os.path.getsize(output_path)//1024} KB")
```

**Key points:**
- Output is always **RGBA PNG** (alpha = transparency).
- Do NOT save as `.jpg`; JPEG has no alpha channel and will crash or lose transparency.
- Acceptable output formats: `.png`, `.webp`.

---

## Batch Folder — Process All Images

```python
from pathlib import Path
from rembg import remove, new_session
from PIL import Image

input_dir  = Path("/mnt/user-data/uploads/images/")
output_dir = Path("/mnt/user-data/outputs/no_bg/")
output_dir.mkdir(parents=True, exist_ok=True)

session = new_session()          # reuse session for performance

for img_file in sorted(input_dir.glob("*")):
    if img_file.suffix.lower() not in {".jpg", ".jpeg", ".png", ".webp", ".bmp"}:
        continue
    out_file = output_dir / (img_file.stem + "_no_bg.png")
    img    = Image.open(img_file)
    result = remove(img, session=session)
    result.save(out_file)
    print(f"✓ {img_file.name} -> {out_file.name}")

print("Done.")
```

---

## Choosing a Model

Pass `model_name` to `new_session()` or `remove()`:

| Model | Best for | Size |
|---|---|---|
| `u2net` (default) | General use | ~170 MB |
| `u2netp` | Fast / lightweight | ~4 MB |
| `u2net_human_seg` | People / portraits | ~170 MB |
| `isnet-general-use` | High quality general | ~176 MB |
| `birefnet-general` | Best quality (slow) | ~370 MB |
| `sam` | SAM interactive segmentation | ~375 MB |

```python
from rembg import remove, new_session

session = new_session("u2net_human_seg")   # portrait / people
result  = remove(img, session=session)
```

Models auto-download to `~/.u2net/` on first use.

---

## Video Background Removal (via FFmpeg pipe)

rembg processes video frames from FFmpeg's stdout stream.

```bash
# Step 1 — get video dimensions
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height \
  -of csv=p=0 input.mp4
# -> e.g. 1280,720

# Step 2 — pipe frames through rembg, output per-frame PNGs
ffmpeg -i input.mp4 -an -f rawvideo -pix_fmt rgb24 pipe:1 \
  | rembg b 1280 720 -o /tmp/frames/frame-%04d.png

# Step 3 — reassemble transparent frames into WebM (supports alpha)
ffmpeg -framerate 30 -i /tmp/frames/frame-%04d.png \
  -c:v libvpx-vp9 -pix_fmt yuva420p \
  -b:v 2M output_transparent.webm
```

Python helper for the full pipeline:

```python
import subprocess, shutil
from pathlib import Path

def remove_video_background(input_video: str, output_video: str, fps: int = 30):
    """Remove background from a video file, output transparent WebM."""
    # Detect dimensions
    probe = subprocess.run(
        ["ffprobe", "-v", "error", "-select_streams", "v:0",
         "-show_entries", "stream=width,height",
         "-of", "csv=p=0", input_video],
        capture_output=True, text=True, check=True
    )
    w, h = probe.stdout.strip().split(",")

    frames_dir = Path("/tmp/rembg_frames")
    frames_dir.mkdir(exist_ok=True)

    try:
        # Pipe video frames through rembg
        ffmpeg_proc = subprocess.Popen(
            ["ffmpeg", "-i", input_video, "-an",
             "-f", "rawvideo", "-pix_fmt", "rgb24", "pipe:1"],
            stdout=subprocess.PIPE, stderr=subprocess.DEVNULL
        )
        rembg_proc = subprocess.Popen(
            ["rembg", "b", w, h, "-o", str(frames_dir / "frame-%04d.png")],
            stdin=ffmpeg_proc.stdout, stderr=subprocess.DEVNULL
        )
        ffmpeg_proc.stdout.close()
        rembg_proc.wait()
        ffmpeg_proc.wait()

        # Reassemble as transparent WebM
        subprocess.run(
            ["ffmpeg", "-y", "-framerate", str(fps),
             "-i", str(frames_dir / "frame-%04d.png"),
             "-c:v", "libvpx-vp9", "-pix_fmt", "yuva420p",
             "-b:v", "2M", output_video],
            check=True, stderr=subprocess.DEVNULL
        )
        print(f"Saved: {output_video}")
    finally:
        shutil.rmtree(frames_dir, ignore_errors=True)

# Usage
remove_video_background(
    input_video  = "/mnt/user-data/uploads/clip.mp4",
    output_video = "/mnt/user-data/outputs/clip_no_bg.webm"
)
```

> **Note:** Regular MP4 does NOT support transparency. Use `.webm` (VP9) or `.mov` (ProRes 4444).
> WebM with VP9 is the most widely supported transparent video format in browsers and modern editors.

---

## Alpha Matting (Better Edges)

Alpha matting improves edge quality (hair, fur, complex outlines) at the cost of speed:

```python
from rembg import remove
from PIL import Image

img    = Image.open("input.png")
result = remove(
    img,
    alpha_matting                      = True,
    alpha_matting_foreground_threshold = 240,
    alpha_matting_background_threshold = 10,
    alpha_matting_erode_size           = 10,
)
result.save("output.png")
```

---

## Return Only the Mask

Useful for debugging or compositing:

```python
from rembg import remove
from PIL import Image

img  = Image.open("input.png")
mask = remove(img, only_mask=True)   # returns grayscale mask
mask.save("mask.png")
```

---

## Output to Bytes (HTTP / in-memory)

```python
from rembg import remove

with open("input.png", "rb") as f:
    result_bytes = remove(f.read())   # returns PNG bytes
# result_bytes can be sent over HTTP or written to any file-like object
```

---

## CLI Quick Reference

```bash
# Single image
rembg i input.png output.png

# Specify model
rembg i -m u2net_human_seg portrait.jpg portrait_out.png

# Alpha matting
rembg i -a face.jpg face_out.png

# Mask only
rembg i -om input.png mask.png

# Batch folder
rembg p /path/to/input/ /path/to/output/

# Watch folder (auto-process new files)
rembg p -w /path/to/input/ /path/to/output/

# HTTP server
rembg s --host 0.0.0.0 --port 7000

# Video pipe (width height must match source)
ffmpeg -i video.mp4 -an -f rawvideo -pix_fmt rgb24 pipe:1 | rembg b 1280 720 -o frames/frame-%04d.png
```

---

## Common Mistakes to Avoid

| Wrong | Right |
|---|---|
| Save output as `.jpg` | Always save as `.png` or `.webp` |
| Forget `--break-system-packages` when pip installing | Add `--break-system-packages` |
| Process video directly with rembg | Use FFmpeg pipe (`rembg b`) |
| Save transparent video as `.mp4` | Use `.webm` (VP9) or `.mov` (ProRes 4444) |
| Create new session per image in a loop | Create one `new_session()`, reuse it |
| Hard-code width/height for video | Use `ffprobe` to detect dimensions dynamically |

---

## Output File Conventions

| Task | Extension | Notes |
|---|---|---|
| Single image | `_no_bg.png` | e.g. `photo_no_bg.png` |
| Batch images | `_no_bg.png` per file | keep original filename stem |
| Transparent video | `_no_bg.webm` | VP9 codec |
| High-quality video (editor) | `_no_bg.mov` | ProRes 4444, large file |
| Mask only | `_mask.png` | grayscale |

Always write final outputs to `/mnt/user-data/outputs/` and call `present_files` so the user can download them.
