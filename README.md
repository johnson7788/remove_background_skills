# Remove Background Skills

Claude Code skill for AI-powered image and video background removal, powered by [rembg](https://github.com/danielgatis/rembg).

## Skill Overview

The `removebackground` skill equips Claude Code with the ability to remove backgrounds from images and videos using state-of-the-art AI segmentation models. It handles both single images and batch processing, supports video pipelines via FFmpeg, and offers model selection for different quality/speed trade-offs.

## Triggers

The skill activates when the user mentions any of the following:

- "remove background" / "background removal" / "remove bg"
- "transparent background" / "alpha channel"
- "cut out subject" / "cutout image"
- 去除背景 / 去背景 / 透明背景 / 抠图 / 视频去背景
- Uploads an image/video and asks to isolate the subject

## Features

- **Single image** → RGBA PNG or WebP with transparent background
- **Batch processing** → process entire folders of images (shared session for performance)
- **Video** → FFmpeg pipeline extracting frames, removing background per-frame, reassembling as transparent WebM (VP9)
- **6 AI models** — from lightweight 4 MB (`u2netp`) to high-quality 370 MB (`birefnet-general`)
- **Alpha matting** — improved edge quality for hair, fur, and complex outlines
- **Mask extraction** — grayscale mask output for debugging or compositing
- **In-memory processing** — output to bytes for HTTP/file-like objects

## Requirements

- Python >= 3.11, < 3.14
- FFmpeg (for video)
- pip packages: `rembg`, `Pillow`

## Installation

```bash
# CPU (most common)
pip install "rembg[cpu,cli]" --break-system-packages

# NVIDIA GPU
pip install "rembg[gpu,cli]" --break-system-packages

# AMD ROCm
pip install onnxruntime-rocm
pip install "rembg[rocm,cli]" --break-system-packages
```

## Models

| Model | Size | Best For |
|---|---|---|
| `u2net` (default) | ~170 MB | General use |
| `u2netp` | ~4 MB | Fast / lightweight |
| `u2net_human_seg` | ~170 MB | People / portraits |
| `isnet-general-use` | ~176 MB | High quality general |
| `birefnet-general` | ~370 MB | Best quality (slow) |
| `sam` | ~375 MB | Interactive segmentation |

Models auto-download to `~/.u2net/` on first use.

## Output Conventions

| Task | Extension | Example |
|---|---|---|
| Single image | `_no_bg.png` | `photo_no_bg.png` |
| Batch images | `_no_bg.png` | per original filename |
| Transparent video | `_no_bg.webm` | VP9 codec |
| High-quality video | `_no_bg.mov` | ProRes 4444 |
| Mask only | `_mask.png` | grayscale |

> Note: JPEG does not support alpha channels. Always use PNG or WebP for transparent images, and WebM (VP9) or MOV (ProRes 4444) for transparent video.

## Files

```
removebackground/
├── SKILL.md              # Skill definition (installed to .claude/skills/)
├── speaking.mp4           # Example input video
└── speaking_no_bg.webm    # Example output (background removed)
```

## License

MIT — matches the [rembg](https://github.com/danielgatis/rembg) project license.
