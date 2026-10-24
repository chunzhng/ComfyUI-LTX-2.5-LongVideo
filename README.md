# ComfyUI LTX 2.5 Long Video

A fork of [LTX 2.5 Director (CS)](https://github.com/CGlide/WhatDreamsCost-CSGlide) that adds **keyframe-pair rendering** to the existing chunk writer. Drop a sequence of keyframe images on the timeline and render it as one long video: each adjacent pair becomes an FLF segment whose last frame is pinned to the next keyframe, and the writer joins the segments with a one-frame overlap.

This is a **drop-in replacement** for `LTX-2.5-Director-main`. The node IDs are unchanged (`LTXDirectorCS25`, `LTXChunkWriterCS25`, ...), so existing workflows keep working.

## Features

- **Render Pairs** — adjacent timeline images render as `A→B`, `B→C`, ... Each pair is a separate sampling window; the first image is the start anchor, the second is pinned as `isEndFrame`. No generated tail, so the seam starts from the exact next keyframe.
- **Render All (handoff)** — the upstream continuous long-video mode: fixed or snapped chunk windows, handoff frame strips carried into the next chunk, optional motion sequence. Use this when you have one continuous shot instead of keyframes.
- **Chunk writer / assembler** — saves frames per chunk, assembles the final mp4 with configurable seam modes, overlap, level matching, and audio muxing.
- **Timeline** — image, text/prompt-relay, audio and motion segments; prompt zones and packed timelines from upstream.

## Install

1. Clone or copy this folder into `ComfyUI/custom_nodes/`.
2. **Remove or disable the original `LTX-2.5-Director-main`** if it is installed. This fork registers the same node IDs, and ComfyUI cannot load both.
3. Restart ComfyUI.
4. `av` (PyAV), `numpy` and `Pillow` are required for mp4/audio output. ComfyUI already ships `numpy` and `Pillow`; if the writer reports that video is skipped, install `av` into ComfyUI's Python.

## Render Pairs

1. Load `example_workflows/LTX_LongVideo_Pairs.json` (or build the same graph with your own model loaders).
2. Replace the model, input images and audio with your own.
3. Put one image segment per keyframe on the main track, in order. The example uses:
   - `A` at frame `0`
   - `B` at frame `960`
   - `C` at frame `1920`
4. Set `run_name` and `runs_folder` on `LTX Chunk Writer CS (2.5)`. The writer's default output is `ComfyUI/output/<runs_folder>/<run_name>/`.
5. Open the timeline's **Render** menu and click **Render Pairs**.
6. The final file is `<runs_folder>/<run_name>/<run_name>_final.mp4`.

What the button does per pair:

- window `[A.start, B.start + 1)`, so B lands on the last frame;
- start image is an anchor, end image is `isEndFrame`;
- writer values are forced for the run: `assemble_overlap=1`, `handoff_frames=0`, `seam_mode=early_cut`, `seam_blend_frames=3`, `total_chunks=<pair count>`;
- the original timeline, window and writer values are restored after the run (including on failure).

At 24 fps the tested pair spacing is **961 frames = 40.04 s** per pair. LTX lengths should be `8n+1`; 961 is `8×120+1`. Other spacings work but the model may pad them.

## Continuous long video (Render All)

Use **Render All** when the timeline is one continuous shot and you want handoff instead of keyframes. Tested recipe on a 24 GB Turing card:

- 5 chunks × 961 frames (40.04 s each)
- writer `assemble_overlap=24`, `handoff_frames=24`
- writer node property `motion_frames=9` (9 sharp tail frames as the motion guide)
- `seam_mode=early_scurve`, `seam_blend_frames=8`
- result: 4709 frames / 196.2 s

The planner can snap cuts to prompt-zone or image boundaries (`Zones`, `Zones+Img`, `Fixed`). Keep the total length and target chunk length in the Auto Chunk Render row.

## Tips and limits

- Pair mode treats **every non-handoff image segment** on the main track as a keyframe. Do not put decorative images there; use ghost/handoff segments for other purposes.
- The first image starts the output; the last image ends it. Audio segments stay at their absolute timeline positions and are sliced by the render window.
- The example requests `544×960`; the VAE rounds the output to `512×960`. Raise `custom_width`/`custom_height` for higher resolution, at the cost of VRAM and time.
- Pair mode restores the timeline in a `finally` block. Render All keeps the failed window in place so you can resume a failed chunk by hand.
- A run is sequential: one prompt at a time, waiting for each chunk before setting up the next. Do not start a second render from another tab while one is running.
- On tight VRAM the first VAE decode after sampling can sit at 100% GPU for several minutes while ComfyUI streams weights. The example keeps an `easy cleanGpuUsed` node (ComfyUI-Easy-Use) bypassed for compatibility; enable it before `VAEDecode` if the decode thrashes, or launch ComfyUI with `--reserve-vram`.
- The example workflow contains that `easy cleanGpuUsed` node. Install ComfyUI-Easy-Use, or delete the node if your setup does not have it.

## Credits and license

- Upstream: [LTX 2.5 Director (CS)](https://github.com/CGlide/WhatDreamsCost-CSGlide), built on [WhatDreamsCost's LTX Director](https://github.com/WhatDreamsCost/WhatDreamsCost-ComfyUI). The timeline editor, chunk writer, assembler and handoff renderer come from those projects.
- The `Yusu-WhatDreamsCost-ComfyUI` variant follows the same timeline model and informed the workflow compatibility work.
- Render Pairs (planner + UI button) and this packaging are additions in this fork.
- License: **GPL-3.0**, same as upstream. See `LICENSE`. If you redistribute a modified version, keep the license and credit the upstream authors.

## 中文快速开始

1. 把本目录放进 `ComfyUI/custom_nodes/`，删除或停用原来的 `LTX-2.5-Director-main`（节点 ID 相同，不能共存），重启 ComfyUI。
2. 打开 `example_workflows/LTX_LongVideo_Pairs.json`，换成你自己的模型、输入图片和音频。
3. 时间线上按顺序放关键帧图片：A 在 0 帧、B 在 960 帧、C 在 1920 帧（24 fps 下每对 961 帧 = 40.04 秒）。
4. 在 `LTX Chunk Writer CS (2.5)` 上设置 `run_name` 和 `runs_folder`。
5. 打开时间线的 **Render** 菜单，点 **Render Pairs**。成片在 `ComfyUI/output/<runs_folder>/<run_name>/<run_name>_final.mp4`。
6. 连续长镜头用原来的 **Render All**（handoff 模式），参数见上面英文的测试配方。

注意事项：相邻图片会自动配对；主轨上的其他图片也会被当成关键帧。关键帧间距建议 `8n+1`（961 已验证）。`544×960` 实际输出 `512×960`，需要更高分辨率就调 `custom_width`/`custom_height`。
