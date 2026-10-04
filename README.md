# ComfyUI-Workflow-Video-Quality-Control

An autonomous, multi-stage Quality Assurance (QA) and subtitle verification suite for video post-production in ComfyUI.

Designed for VFX compositors, video editors, and motion design studios, this repository documents the progressive evolution of automated video auditing. It bridges computer vision (OCR), speech recognition (ASR), large multimodal language models, and external host software (Adobe After Effects) to eliminate human error in subtitle timing, burned-in on-screen typography, and spoken audio compliance.

---

## The 3-Tier Workflow Evolution

This repository contains three tested production versions reflecting different architectural approaches:

```text
ComfyUI-Workflow-Video-Quality-Control/
├── Quality Control - V01.json    # Phase 1: Local Vision-Language Ingestion
├── Quality Control - V02.json    # Phase 2: Frame Slicing & Resolution Optimization
├── Quality Control - V03.json    # Phase 3: Autonomous Host DCC / After Effects MCP Bridge
└── README.md
```

### 1. `Quality Control - V01.json` (Direct Multimodal Audit)
- **Architecture:** Local Ingestion & Remote LLM Reasoning.
- **Workflow:**
  - Ingests raw video footage and splits visual/audio channels via `LoadVideoAdvanced`.
  - Runs local **Florence-2 Large (`florence-2-large-ft`)** with OCR task conditioning on frame sequences.
  - Concurrently transcribes audio using **WhisperX** with automated language alignment.
  - Combines OCR frames, speech transcript, and video metadata via `StringFormat`, sending the combined payload to **Google Gemini (`gemini-3.6-flash`)** to flag line-by-line subtitle mismatches, substitutions, or dropped words.
- **Best For:** Self-contained ComfyUI workflows running on single workstations.

### 2. `Quality Control - V02.json` (Optimized Frame Slicing & Resampling)
- **Architecture:** Enhanced Node Schema & Memory Hygiene.
- **Workflow:**
  - Upgrades the ingestion stage with fine-grained parameterization: explicit `frame_start`, `frame_count`, `crop_padding`, and target FPS resampling (`fps_target: 20`).
  - Standardizes aspect ratio downsampling (`target_width: 480`) to drastically reduce GPU memory footprint during dense Florence-2 OCR passes.
  - Improves string formatting stability and metadata serialization for large multi-shot ad assets.
- **Best For:** Long-form social ads and high-resolution (4K/ProRes) source files that require memory-efficient sampling.

### 3. `Quality Control - V03.json` (Autonomous Host Orchestration via MCP)
- **Architecture:** Full Agentic Integration with Adobe After Effects & Claude Code.
- **Workflow:**
  - Dispatches decoupled background processes using `UniversalScriptRunner` to launch the **After Effects MCP Server** and **Adobe After Effects**.
  - Utilizes `MainBrainNode` with **Win32 thread-attached IPC** to focus the active terminal hosting **Claude Code**, injecting structured OCR and transcript audit directives.
  - Directs Claude Code to interactively sample composition frames via ExtendScript (`$.grabCurrentFrameSnapshot`), snap the playhead (`$.jumpToFrame`), and place frame-accurate review markers directly on the target comp layer (`QA_Error_Markers`).
  - Employs `AEMarkerReaderNode` to read the exported marker log (`ae_qa_report.txt`) directly back into ComfyUI for instant review inside the canvas.
- **Best For:** Professional studio environments where QA notes must live directly on the editor's After Effects timeline.

---

## Workflow Comparison Matrix

| Feature | Version 01 | Version 02 | Version 03 |
| :--- | :--- | :--- | :--- |
| **Execution Model** | ComfyUI Internal | ComfyUI Internal | ComfyUI + Host DCC (After Effects) |
| **OCR Engine** | Florence-2 Large | Florence-2 Large | Host ExtendScript Frame Grabber |
| **Transcript Source** | WhisperX | WhisperX | Claude Code Audio Ingestion |
| **Reasoning Model** | Gemini 3.6 Flash | Gemini 3.6 Flash | Claude Code (Agentic CLI) |
| **Output Location** | ComfyUI `ShowText` | ComfyUI `ShowText` | AE Comp Timeline Markers + ComfyUI |
| **Memory Footprint** | Moderate (Full frames) | Low (Sliced & resized) | Zero VRAM overhead (CLI IPC) |

---

## Prerequisites & Required Nodes

Depending on which workflow version you run, ensure the relevant custom nodes are installed:

### For V01 & V02:
- **ComfyUI-LoadVideoAdvanced:** Video ingestion and frame pre-processing.
- **ComfyUI-Florence2:** Florence-2 model loader and OCR runner (`Florence2ModelLoader`, `Florence2Run`).
- **ComfyUI-WhisperX:** Audio transcription and word-level alignment (`Apply WhisperX`).
- **ComfyUI-Gemini:** Google Gemini API multimodal integration (`GeminiAPI`).
- **ComfyUI-Custom-Scripts:** Preview and text display nodes (`ShowText|pysssss`).

### For V03:
- **[ComfyUI-UniversalScriptRunner](https://github.com/harsh-shrivas/ComfyUI-UniversalScriptRunner):** Non-blocking external process launcher.
- **[ComfyUI-MainBrainNode](https://github.com/harsh-shrivas/ComfyUI-MainBrainNode):** Win32 thread-attaching terminal bridge to Claude Code.
- **[ComfyUI-AE-Marker-Reader](https://github.com/harsh-shrivas/ComfyUI-AE-Marker-Reader):** Local ExtendScript timeline marker extractor.
- **Adobe After Effects (2024+)** with the After Effects MCP Server configured.

---

## Setup & Configuration

### Running V01 or V02:
1. Load `Quality Control - V01.json` or `Quality Control - V02.json` into ComfyUI.
2. In `LoadVideoAdvanced`, enter the absolute path to your video file (`video_path`).
3. In `GeminiAPI`, paste your Google Gemini API key into `api_key`.
4. Click **Queue Prompt**. Review discrepancies in the `ShowText` preview window.

### Running V03:
1. Load `Quality Control - V03.json` into ComfyUI.
2. In `UniversalScriptRunner` (Node 1), set the path to your Claude Code launcher script.
3. In `UniversalScriptRunner` (Node 2), set the path to your `afterfx.exe` executable.
4. In `UniversalAssetReader`, specify the path to your video under review.
5. In `AEMarkerReaderNode`, set `marker_layer_name` to your target review layer (defaults to `QA_Error_Markers`).
6. Click **Queue Prompt**. ComfyUI will launch the tools, inject audit directives to Claude Code, place markers on the After Effects timeline, and pull the formatted report back into ComfyUI.

---

## License

MIT License. Free to use, modify, and integrate into internal VFX, commercial video, and animation pipelines.
