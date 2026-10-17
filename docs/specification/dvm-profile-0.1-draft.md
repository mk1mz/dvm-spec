# DVM Profile 0.1: Draft Specification

**Status: DRAFT, unofficial, not a standard, not endorsed by any standards body or vendor.**
Name "DVM" is provisional. Version 0.1 (2026-10-17).
Reference implementation of sections 4, 5, 6 and 8: `src/dvm/`, tests in `tests/`.
Sections marked **[UNVERIFIED]** depend on experiments (Phase 0) that have not been run.

Label key for every feature: **EXT-STD** existing standardized capability · **REUSE** existing capability being reused · **DVM** proposed DVM extension · **OPT** future optional · **OPEN** unresolved research question.

The key words MUST, MUST NOT, SHOULD, MAY are used as in RFC 2119 / RFC 8174.

---

## 1. Scope and terminology

DVM Profile 0.1 describes how a single conventional video file can carry a time-varying rectangular **viewport** over a fixed **encoded canvas**, and how a player turns that viewport into displayed pixels.

- **Encoded canvas**: the decoded picture from the video codec (width Bw × height Bh pixels). Unchanged by DVM.
- **Viewport**: a rectangle inside the canvas that the author wants shown at a given media time.
- **Output display / window**: the surface the player draws on. Its aspect ratio is independent of the viewport's.

A file with no baked-in black bars does **not** guarantee a screen without empty space: if the viewport's aspect ratio differs from the window's, the player must letterbox/pillarbox (`contain`), crop (`cover`), or show something behind the video (`mask`, only where a backdrop exists). DVM defines these behaviors; it cannot remove the mismatch.

DVM cannot create pixels. A viewport only reveals what was encoded. See `docs/prior-art.md` §5 for canvas sizing.

## 2. Normative references and their status

| Reference | Status | Label |
|---|---|---|
| ISO/IEC 14496-12 (ISOBMFF), 2020 ed. | Published; paid. Timed metadata tracks, track references. | EXT-STD |
| Apple, *Rectangular Mask Payload Metadata within the QuickTime Movie File Format*, v0.9 **Beta**, 2025-06-09 | Vendor draft; may change; no open licence or patent statement found. | REUSE |
| QuickTime File Format (QTFF) | Apple. Defines `mebx`/keys structures used by the payload carriage. | REUSE |

DVM does not redefine anything in these documents. Where this draft restates their content it is a summary, and the source text governs.

## 3. Carriage [UNVERIFIED]

REUSE (from Apple v0.9; verified in the source text):

1. The viewport timeline is a **timed metadata track** whose sample entry is `mebx` (boxed metadata).
2. Each sample carries one item with key `com.apple.quicktime.video.display-maskrect.mono` (keyspace `mdta`), whose value is a `RasterRectangleValue`: six big-endian unsigned 16-bit integers `reference_raster_width, reference_raster_height, rectangle_left, rectangle_width, rectangle_top, rectangle_height` (12 bytes).
3. The metadata track MUST be associated with the video track by a `rndr` track reference. Apple's documentation states playback ignores the metadata without it.
4. The metadata SHOULD be in a track holding no other timed metadata (Apple recommendation).
5. In ISOBMFF files the value's data type is not signaled; readers MUST know the key's type.

DVM [DVM]: `reference_raster_width/height` MUST equal the encoded canvas size of the video track at authoring time. Writers MUST NOT use `stereo-left/right` keys or `ExtendedRasterRectangleValue` in this profile (OPT).

**[UNVERIFIED]** The exact byte layout of the `mebx` sample entry, `keys` box and sample payload is not specified here. It MUST be taken from a reference file produced by Apple's tooling or from the QTFF/ISOBMFF text before any writer is implemented. A writer built from memory is non-conforming.

OPEN: whether metadata and video share an edit list (`elst`); how fragmented files split long samples (Apple notes HLS segmentation may split and duplicate a long sample at segment boundaries; DVM writers SHOULD do the same).

## 4. Timeline evaluation [DVM]

Time is the **presentation time** of the video frame being shown, in seconds, as an exact rational.

4.1 **Lookup.** A sample covers `[start, start+duration)`. The viewport for a frame is the value of the sample covering that frame's presentation time (REUSE: Apple's per-frame semantics).

4.2 **Hold.** A sample applies to every frame inside its interval. A change requires a new sample.

4.3 **Missing.** If no sample covers a frame, the full frame MUST be shown (REUSE: Apple). If the entire metadata track is absent or lacks `rndr`, the file is conventional video.

4.4 **Empty rectangle.** Width or height 0 means nothing is visible (REUSE: Apple).

4.5 **Malformed.** A covering sample with a zero reference dimension or wrong payload length MUST be treated as missing (full frame) [DVM]. Readers SHOULD surface a warning.

4.6 **Overlap/conflict.** Samples inside one track cannot overlap (container structure). If a track's samples are inconsistent, reject that track (full frame). If several mask tracks reference the same video track, the one with the lowest track ID is used [DVM, OPEN: Apple behavior unknown].

4.7 **Seeking, pause, rate.** Evaluation is a pure function of presentation time. After a seek, a reader evaluates at the first frame shown; no history is needed. Pausing shows the frame's viewport; playback rate changes only how fast presentation time advances. Variable frame rate needs nothing special: only each frame's presentation time matters.

4.8 **Presentation vs decode time.** The mapping uses presentation time (after composition offsets and edit lists). Metadata samples carry no decode-order semantics. [OPEN: edit-list interaction]

## 5. Arithmetic and rounding [DVM]

All geometry is integer; all intermediate time is exact rational.

5.1 **round-half-up(x)** = floor(x + 1/2).

5.2 **Scaling a payload to the decoded buffer** (Bw × Bh) from reference raster (Rw × Rh): scale the four *edges* independently, `edge' = round-half-up(edge × B / R)` computed as `(2·edge·B + R) div (2·R)` in at least 64-bit integers; then `width' = right' − left'`, `height' = bottom' − top'`; then clip to the buffer. A result with zero width/height is valid (empty).

Rationale: Apple's v0.9 pseudocode computes `(buffer / reference) * value`; if implemented literally in integers this truncates the ratio (e.g. 720/1080 → 0). DVM pins the arithmetic so independent readers agree. Scaling by edges guarantees adjacent rectangles never gap or overlap through rounding (tested in `tests/test_mask.py`).

5.3 **Transition interpolation (authoring side).** For `smooth`, with previous viewport A and target B, event start `s`, transition duration `d`, and time `t` with `s ≤ t < s+d`: `p = (t−s)/d` (exact rational); each edge (left, top, right, bottom) is `A_edge + (B_edge − A_edge)·p`, rounded once with 5.1. Only linear easing exists in 0.1. At `t ≥ s+d` the viewport is B exactly; at `t = s` it is A exactly.

5.4 **Sampling.** The compiler evaluates the timeline at each frame's presentation time and merges consecutive equal rectangles into one sample. Consequently a frame at exactly `t = s` is still the previous viewport.

## 6. Output presentation [DVM]

6.1 **Modes.** A player MUST implement `viewport-fit`; it MAY implement `mask`.

- `viewport-fit/contain` (default): scale the viewport uniformly so it fits entirely inside the window; centre it (offsets use floor). Window area not covered is player-defined (SHOULD be the player's normal video background).
- `viewport-fit/cover`: scale uniformly to fill the window; crop the viewport's excess equally on both sides (centred, floor). Never draws outside the viewport.
- `mask` (OPT): draw the full-canvas rectangle's pixels only inside the viewport and treat the rest as transparent, exposing whatever is behind the player (e.g. a spatial environment). Only meaningful where a backdrop exists. Matches the behavior Apple documents for visionOS 26.

6.2 **Aspect ratio preservation.** `contain` and `cover` MUST scale uniformly (no stretching).

6.3 **Window changes.** When the viewport changes size (a transition), the player SHOULD keep the window size fixed and let the placement change; resizing the OS window per frame is NOT RECOMMENDED. [OPEN: UX]

6.4 **Orientation and rotation.** Display rotation follows the container/track matrix as for conventional video. Viewport rotation is OPT and out of scope for 0.1.

6.5 **Accessibility.** Rapid geometry changes can cause discomfort; players SHOULD offer a user setting to ignore viewports and show the full frame. Captions and UI SHOULD be positioned relative to the window, not the canvas. [OPEN]

Implemented and tested: `contain`, `cover` (`src/dvm/present.py`). `mask` is not implemented.

## 7. Defaults and errors

| Situation | Behavior |
|---|---|
| No DVM track, or no `rndr` | Conventional playback, full frame |
| No sample for a frame | Full frame |
| Empty rectangle | Nothing visible |
| Rectangle beyond canvas | Clip to canvas |
| Zero reference raster / bad length | Treat sample as missing, warn |
| Authoring input invalid (§8) | Tool MUST reject with every problem listed; MUST NOT write output |
| Unknown authoring `version` | Reject |

## 8. Authoring format (application level, not a file format) [DVM]

Schema: `schemas/dvm-timeline-1.schema.json`, plus semantic checks in `src/dvm/timeline.py`. Rules beyond the schema: first event starts at 0; events are contiguous (no gaps, no overlaps); `x,y ≥ 0`, `width,height ≥ 1`, `x+width ≤ canvas width`, `y+height ≤ canvas height`; coordinates are integers; times are numbers, decimal strings, or rational strings (`"1/3"`); `smooth` is not allowed on the first event; `transition_duration` (default `min(0.5 s, duration)`) must be positive and ≤ the event's duration; `cut` forbids `transition_duration`; unknown keys are errors. After the last event the last viewport holds.

Note: the authoring `transition` describes how an event is *entered* from the previous one. In your 12 s example, a `smooth` event of 3 s with default `transition_duration` animates for 0.5 s and then holds.

## 9. Legacy behavior [OPEN, untested]

Expected (assumption): a player that does not know the metadata track ignores it and shows the entire canvas, including regions the author meant to hide. Mitigations: compose the canvas so the full frame is acceptable; or distribute a separate conventional rendition. Remuxers or editors may drop the track or the `rndr` reference; this must be tested per tool (`docs/compatibility.md`).

## 10. Versioning and forward compatibility

- The profile uses `MAJOR.MINOR`. Readers MUST ignore unknown optional features within the same MAJOR and MUST NOT guess semantics of an unknown MAJOR.
- The authoring `version` integer increments on incompatible authoring changes.
- Because the file payload is Apple's, DVM tracks the Apple version it was verified against (currently 0.9 Beta) and MUST be re-reviewed when Apple publishes a new version.

## 11. Conformance

Planned (Phase 4): synthetic grid/coordinate videos, expected-rectangle tables at selected timestamps, malformed-input vectors, remux-survival tests, VFR vectors. Currently available: unit tests of the evaluator, compiler, scaler and presenter; three example timelines. **No independent implementation has run these tests; no interoperability is claimed.**

## 12. Security considerations

Readers MUST bound allocation by file sizes, not by metadata values; MUST use ≥ 64-bit arithmetic for scaling; MUST NOT trust `reference_raster_*` to be non-zero (§4.5). Authoring tools MUST NOT overwrite the input file.

## 13. IPR and licensing

Apple's document carries "Copyright © 2025 Apple Inc." and no open licence or patent statement was found; no patent search has been done (one patent-literature lead on dynamic aspect ratio is listed in `docs/prior-art.md`, unanalyzed). Legal review is required before any claim of freedom to implement. DVM-original text: CC-BY-4.0 intended; code: Apache-2.0 (see LICENSE), both pending review.
