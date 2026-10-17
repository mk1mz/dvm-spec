# Dynamic Viewport Media (DVM) — Specification & Profile

> **Draft Specification and Standards Proposal.**
> DVM is an open profile over Apple's *Rectangular Mask Payload Metadata* (v0.9 Beta, 2025-06-09).

This repository contains the core specification, JSON authoring schemas, architectural decision records (ADRs), prior art analysis, and reference timeline examples.

---

## The Idea: What is DVM?

Videos today are trapped in fixed aspect ratios. A vertical 9:16 video on a 16:9 monitor has giant black bars on the sides (pillarbox); a 16:9 movie on a phone has black bars on the top and bottom (letterbox).

**Dynamic Viewport Media (DVM)** solves this:
- **One video file, one continuous timeline.**
- **No baked-in black bars:** the video is encoded once on a master canvas (e.g. 1920×1920 square or 3840×2160).
- **Time-varying viewport:** a lightweight timed metadata track describes which rectangle $[x, y, w, h]$ to display at each moment (e.g. 9:16 portrait &rarr; 1:1 square &rarr; 16:9 landscape &rarr; 9:16).
- **Player-side dynamic framing:** compatible players smoothly crop and scale the video to fit the user's screen in real time (`contain` or `cover`).

---

## Contents of this Repository

- **[`specification/dvm-profile-0.1-draft.md`](specification/dvm-profile-0.1-draft.md):** Full draft specification of the DVM profile.
- **[`schemas/dvm-timeline-1.schema.json`](schemas/dvm-timeline-1.schema.json):** Strict JSON Schema for authoring DVM timelines.
- **[`adr/0001-profile-over-apple-payload.md`](adr/0001-profile-over-apple-payload.md):** Architectural decision to build as a profile over Apple's `RasterRectangleValue` (`mebx` / `rndr`).
- **[`docs/prior-art.md`](docs/prior-art.md):** Comprehensive analysis of prior art (MPEG OMAF, QuickTime, pan-scan, visionOS).
- **[`docs/compatibility.md`](docs/compatibility.md):** Platform compatibility matrix and experimental evidence.
- **[`docs/standardization-roadmap.md`](docs/standardization-roadmap.md):** Standards engagement roadmap (MPEG Systems / MP4RA / W3C).
- **[`examples/`](examples/):**
  - `portrait-square-landscape.json`: 9:16 &rarr; 1:1 &rarr; 16:9 &rarr; 9:16 scenario.
  - `era-cuts.json`: Historical film ratios (4:3 &rarr; 1.85:1 &rarr; 2.39:1 scope).
  - `slow-push-in.json`: Continuous slow camera push-in zoom.

---

## Related Repositories

- **DVM Core & Muxer (`dvm-core`):** Python parser, stream-copy MP4 muxer, and CLI tool.
- **DVM Studio (`dvm-studio`):** Web-based visual video editor and reference player.

---

## License

Specification text is intended to be CC-BY-4.0. Code schemas and examples are under Apache-2.0. Apple's specification is separately copyrighted by Apple Inc.
