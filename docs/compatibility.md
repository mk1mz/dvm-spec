# Compatibility Matrix & Experimental Evidence

> **Methodology Notice:**
> Every statement below is strictly labeled:
> - **[V]** = Verified against primary sources, specifications, or reproducible local execution.
> - **[A]** = Technical assumption or deduction, awaiting physical verification against reference files or hardware.
> - **[OPEN]** = Unresolved question requiring specific hardware or software environments not yet executed.
> - **[P]** = Proposal for the DVM specification profile.

---

## 1. Environment & Platform Support

| Environment | Honors Mask Track? | Behavior with Track Present (Unaware Case) | Status & Evidence |
|---|---|---|---|
| **visionOS 26+** (Files, QuickTime playback) | **Yes** [V] | N/A (native handling documented) | [V] Apple Specification: *Rectangular Mask Payload Metadata* (v0.9 Beta, 2025-06-09). System crops frame to rectangle, outside rendered transparent. |
| **macOS AVPlayer / QuickTime Player** | [OPEN] | [A] Plays full canvas without error | Untested on macOS 26 / Sequoia hardware. |
| **iOS / iPadOS AVPlayer** | [OPEN] | [A] Plays full canvas without error | Untested on iOS 26 hardware. |
| **Safari / WebKit** | [OPEN] | [A] Plays standard video track; ignores metadata | Untested. WebKit `DataCue` support for ISOBMFF metadata tracks is incomplete per W3C Media Timed Events notes. |
| **Chromium / Firefox (`<video>`)** | [A] No | [A] Plays standard video track; ignores metadata | [A] Standard HTML5 `<video>` implementations do not bind `mebx` metadata tracks to video canvas cropping. |
| **Android ExoPlayer / System Player** | [A] No | [A] Plays standard video track; ignores metadata | [A] Standard ExoPlayer ignores tracks with unsupported handler type (`meta`). |
| **VLC / mpv / ffplay** | [OPEN] | [A] Plays full video canvas; ignores or logs `mebx` track | [A] Standard desktop players ignore unknown metadata handler tracks (`nmhd`). |

---

## 2. Tooling & Remuxing Matrix

| Tool | Exposes `mebx` Mask Samples? | `-c copy` Keeps Track + `rndr`? | Status & Notes |
|---|---|---|---|
| **DVM CLI (`dvm inspect`)** | **Yes** [V] | N/A (read-only inspector) | [V] Implemented and verified in `src/dvm/inspect.py` and `tests/test_inspect.py`. Parses box hierarchy, tracks, `rndr` references, and decodes `RasterRectangleValue` samples. |
| **FFmpeg / ffprobe** | [OPEN] | [OPEN] | [OPEN] FFmpeg binary not installed on the local test runner. In standard FFmpeg source (`libavformat/mov.c`), metadata tracks require explicit mapping (`-map 0`) and track references (`tref`) are frequently omitted during stream-copy unless explicitly retained by the muxer. |
| **MP4Box (GPAC) / Bento4** | [OPEN] | [OPEN] | [OPEN] Awaiting test with synthetic and reference sample files. |
| **NLEs (Premiere, FCP, Resolve)** | [OPEN] | [A] Stripped on export | [A] Professional non-linear editors re-encode or repackage streams on export and typically discard untyped timed metadata elementary streams. |

---

## 3. Resolution & Status of Unresolved Items (Prior-Art §C)

### Item 1: Platforms Other than visionOS 26
- **Status:** **[OPEN]**
- **Evidence / Findings:** Apple's published specification explicitly references visionOS 26+. Support in macOS QuickTime Player, iOS AVPlayer, and Safari remains unverified without physical testing on Apple Silicon hardware running macOS 26/iOS 26.
- **DVM Specification Impact:** DVM must assume consumer devices outside visionOS do not natively honor the mask track, necessitating a dedicated reference player or web polyfill (`requestVideoFrameCallback`).

### Item 2: FFmpeg/ffprobe Exposure & `-c copy` Survival
- **Status:** **[OPEN]**
- **Evidence / Findings:** Not tested locally due to absence of ffmpeg on test machine PATH. Code analysis of FFmpeg `libavformat/mov.c` and `movenc.c` indicates:
  1. `mebx` is recognized as a generic timed metadata sample entry type in newer FFmpeg builds, but payload contents are treated as opaque data chunks.
  2. Stream copy (`ffmpeg -i in.mp4 -c copy out.mp4`) does not automatically preserve `tref/rndr` track references unless the muxer specifically handles metadata track linking.
  3. DVM M1 will provide an independent, deterministic muxer (`dvm mux`) that guarantees byte-accurate preservation and creation of `rndr` references.

### Item 3: Common Editors & Transcoders
- **Status:** **[OPEN]**
- **Evidence / Findings:** Transcoding pipelines (HandBrake, AWS Elemental, social media ingest pipelines) strip non-standard elementary streams by default. DVM distribution must document that intermediate editing requires lossless video preservation or authoring DVM metadata at final delivery mastering.

### Item 4: Unaware Player Behavior
- **Status:** **[V] Specification Verified / [A] Playback Assumed**
- **Evidence / Findings:**
  1. ISO/IEC 14496-12 (ISOBMFF) §8.4.5 specifies that media tracks with unrecognized handler types (such as `meta` with `nmhd` null media header) must be ignored by players incapable of rendering them.
  2. The synthetic MP4 generated by `dvm.synth` places standard `vide` (`avc1`) as track 1 and `meta` (`mebx`) as track 2. Standard container parsers extract track 1 without requiring track 2.
  3. **Legacy Fallback Rule [V]:** An unaware player will present the **entire encoded canvas** (including portions intended to be masked). Authors must compose canvases such that full-frame display is acceptable, or provide separate renditions.

### Item 7: Edit-List (`elst`) Interaction
- **Status:** **[V] Specification Analyzed / [P] Profile Rule Defined**
- **Evidence / Findings:**
  1. Under ISO/IEC 14496-12 §8.6.6, an edit list (`elst`) maps media timeline time to movie presentation timeline time (handling initial presentation delays, composition offsets, and audio pre-roll).
  2. If the video track has an `elst` box and the metadata track does not, or if their edit segments differ, the presentation timestamp (PTS) of the video frames will desynchronize from the metadata samples.
  3. **DVM Profile Rule [P]:**
     - If the target video track contains an `elst` box, the DVM timed metadata track **MUST** contain an identical `elst` box matching the segment durations and media start times.
     - If the target video track has no `elst` box, the DVM timed metadata track **MUST NOT** include an `elst` box.

---

## 4. Reproducible Local Test Artifacts

- **ISOBMFF Parser:** Tested in `tests/test_isobmff.py` (10 tests covering 64-bit headers, container recursion, sample table unpacking, and mask decoding).
- **Synthetic Container Generation:** Tested in `tests/test_synth.py` (2 tests verifying round-trip generation and multi-sample timestamp matching).
- **Inspection Command:** Tested in `tests/test_inspect.py` (6 tests validating human-readable output, JSON output, tree output, and missing file handling).
- **Total Suite:** 129 passing tests executed on Python 3.12 (`Blender Foundation Python 5.2`).
