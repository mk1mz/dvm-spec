# Dynamic Viewport Media (DVM) — Specification & Standard

[![Status: Draft Specification](https://img.shields.io/badge/Status-Draft_Specification-blue.svg)](spec/SPEC.md)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-green.svg)](LICENSE)
[![Format: ISOBMFF / QTFF](https://img.shields.io/badge/Standard-ISOBMFF_%2F_Apple_mebx-purple.svg)](https://developer.apple.com/)

> **Dynamic Viewport Media (DVM)** is an open standard profile over Apple's *Rectangular Mask Payload Metadata* (`mebx` / `rndr`) that liberates video from fixed aspect ratios.
> One master video file can dynamically morph its camera window across **9:16 portrait**, **1:1 square**, and **16:9 widescreen** in real time — **with zero baked-in black bars and zero video re-encoding**.

---

## 🎯 The Problem: Why DVM Exists

Every video today is locked into a rigid aspect ratio:
- Vertical video (**9:16**) on a widescreen monitor gets giant black bars on the sides (pillarbox).
- Widescreen movies (**16:9**) on a vertical smartphone get bars top and bottom (letterbox).
- Creators are forced to manually export, store, and manage **3 separate video files** for TikTok, Instagram, and YouTube.

```
TRADITIONAL VIDEO (Baked Black Bars)
┌───────────────────────────────────────┐
│█████████                     █████████│  <-- Permanent black bars
│█████████   [Action Target]   █████████│      burned into pixel data.
│█████████                     █████████│      Wastes bandwidth & resolution.
└───────────────────────────────────────┘

DVM DYNAMIC VIEWPORT (Adaptive Framing)
┌───────────────────────────────────────┐
│ Master Canvas (Full Resolution)       │
│         ┌───────────┐                 │  <-- Player dynamically scales
│         │  Active   │                 │      and centers the viewport
│         │ Viewport  │                 │      to fit any user screen.
│         └───────────┘                 │      Zero black bars. Zero waste.
└───────────────────────────────────────┘
```

---

## ⚡ The Solution: Dynamic Viewport Media

With **DVM**:
1. **One Master Video Canvas**: The video is encoded once at full resolution (e.g., 4K or 1920×1920).
2. **Synchronized Timed Metadata**: A lightweight metadata track (`mebx`) contains real-time camera crop instructions (`[x, y, w, h]` in 16.16 fixed-point normalized coordinates).
3. **Player-Side Framing**: Compatible players smoothly scale and pan the active camera window in real time (`contain` or `cover`) to perfectly fill the user's screen.
4. **100% Lossless Stream-Copy**: Adding or tweaking viewport paths takes **0.1 seconds** because media packets are untouched — **no video re-encoding required**.

---

## 🏛️ The DVM Ecosystem

DVM is architected as an open, modular ecosystem split across three dedicated repositories:

| Repository | Role | Description |
|---|---|---|
| 📑 [**`dvm-spec`**](https://github.com/mk1mz/dvm-spec) | **The Standard** | Core specification, authoring JSON schemas, ADRs, and conformance vectors. |
| ⚙️ [**`dvm-core`**](https://github.com/mk1mz/dvm-core) | **Python Engine** | Zero-dependency ISOBMFF parser, lossless stream-copy muxer, and CLI tool. |
| 🎨 [**`dvm-studio`**](https://github.com/mk1mz/dvm-studio) | **Web App & Player** | Zero-build visual video editor with interactive crop box and reference player. |

---

## 🔬 Container Architecture

DVM standardizes the track architecture inside ISOBMFF / QuickTime container files:

```
[ftyp] major_brand: 'mp42' / 'qt  '
[moov]
  ├── [mvhd] Movie Header
  ├── [trak] Track ID 1: Video Elementary Stream (H.264 / HEVC / AV1)
  │     └── [tkhd] Track dimensions
  │     └── [mdia] -> [hdlr] 'vide'
  └── [trak] Track ID 2: Timed Metadata Stream
        ├── [tkhd]
        ├── [tref]
        │     └── [rndr] References Track ID 1  <-- Links metadata to video track
        └── [mdia]
              ├── [hdlr] 'meta'
              └── [minf] -> [stbl]
                    └── [stsd] -> [mebx]
                          └── Payload: 'RasterRectangleValue'
                                (x, y, width, height in 16.16 fixed-point)
```

---

## 📂 Repository Contents

- **[`spec/SPEC.md`](spec/SPEC.md)**: Full formal specification of the DVM profile.
- **[`schemas/timeline.schema.json`](schemas/timeline.schema.json)**: Strict JSON Schema for validating authoring timelines.
- **[`adr/`](adr/)**: Architectural Decision Records:
  - `0001-profile-over-apple-payload.md`: Rationale for targeting Apple's `RasterRectangleValue` standard.
  - `0002-two-pass-stco-resolution.md`: Eliminating chunk offset drift during stream-copy.
  - `0003-run-length-compression.md`: Compressing stationary viewport holds.
- **[`docs/compatibility.md`](docs/compatibility.md)**: Compatibility matrix across macOS AVPlayer, iOS, Chrome, VLC, and Windows Media Player.
- **[`docs/prior-art.md`](docs/prior-art.md)**: Comparative analysis against MPEG OMAF, QuickTime Pan-Scan, and visionOS Spatial Video.
- **[`examples/`](examples/)**: Reference authoring timelines:
  - `portrait-square-landscape.json`: 9:16 &rarr; 1:1 &rarr; 16:9 &rarr; 9:16 sequence.
  - `era-cuts.json`: Aspect ratio timeline (4:3 &rarr; 1.85:1 &rarr; 2.39:1 scope).
  - `slow-push-in.json`: Continuous slow camera push-in zoom.

---

## 🤝 Contributing

Contributions to the specification, schemas, and player implementations are welcome! Please open an issue or pull request.

## 📄 License

The specification text and documentation are licensed under **Creative Commons Attribution 4.0 International (CC-BY-4.0)**. Code schemas and examples are licensed under the **Apache License, Version 2.0**.
