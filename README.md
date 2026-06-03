# 背景移除技能

基于 [rembg](https://github.com/danielgatis/rembg) 的 Claude Code 技能，用于 AI 驱动的图片和视频背景移除。

## 技能概述

`removebackground` 技能让 Claude Code 能够使用先进的 AI 分割模型去除图片和视频的背景。支持单张图片处理和批量处理，支持通过 FFmpeg 管道处理视频，并提供多种模型以适应不同的质量和速度需求。

## 触发条件

用户提到以下任意内容时技能会被激活：

- "remove background" / "background removal" / "remove bg"
- "transparent background" / "alpha channel"
- "cut out subject" / "cutout image"
- 去除背景 / 去背景 / 透明背景 / 抠图 / 视频去背景
- 上传图片或视频并要求分离主体

## 功能

- **单张图片** → 输出带透明背景的 RGBA PNG 或 WebP
- **批量处理** → 处理整个文件夹的图片（使用共享会话以提高性能）
- **视频处理** → FFmpeg 管道提取帧、逐帧去背景、重新合成为透明 WebM（VP9）
- **6 种 AI 模型** — 从轻量 4 MB（`u2netp`）到高质量 370 MB（`birefnet-general`）
- **Alpha 抠图** — 改善头发、毛发和复杂边缘的抠图质量
- **遮罩提取** — 输出灰度遮罩，用于调试或合成
- **内存处理** — 输出字节流，适用于 HTTP 或类文件对象

## 环境要求

- Python >= 3.11, < 3.14
- FFmpeg（视频处理必须）
- pip 包：`rembg`、`Pillow`

## 安装

```bash
# CPU（最常用）
pip install "rembg[cpu,cli]" --break-system-packages

# NVIDIA GPU
pip install "rembg[gpu,cli]" --break-system-packages

# AMD ROCm
pip install onnxruntime-rocm
pip install "rembg[rocm,cli]" --break-system-packages
```

## 模型

| 模型 | 大小 | 适用场景 |
|---|---|---|
| `u2net`（默认） | ~170 MB | 通用场景 |
| `u2netp` | ~4 MB | 快速 / 轻量 |
| `u2net_human_seg` | ~170 MB | 人物 / 人像 |
| `isnet-general-use` | ~176 MB | 高质量通用 |
| `birefnet-general` | ~370 MB | 最佳质量（较慢） |
| `sam` | ~375 MB | 交互式分割 |

模型首次使用时自动下载至 `~/.u2net/`。

## 输出规范

| 任务 | 扩展名 | 示例 |
|---|---|---|
| 单张图片 | `_no_bg.png` | `photo_no_bg.png` |
| 批量图片 | `_no_bg.png` | 按原始文件名 |
| 透明视频 | `_no_bg.webm` | VP9 编码 |
| 高质量视频 | `_no_bg.mov` | ProRes 4444 |
| 仅遮罩 | `_mask.png` | 灰度图 |

> 注意：JPEG 不支持 Alpha 通道。透明图片请使用 PNG 或 WebP，透明视频请使用 WebM（VP9）或 MOV（ProRes 4444）。

## 文件结构

```
removebackground/
├── SKILL.md               # 技能定义文件（安装至 .claude/skills/）
├── speaking.mp4           # 示例输入视频
└── speaking_no_bg.webm    # 示例输出（已移除背景）
```

## 许可证

MIT — 与 [rembg](https://github.com/johnson7788/remove_background_skills) 项目保持一致。
