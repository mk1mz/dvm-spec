# DVM — Feasibility, Prior Art, and Recommended MVP Architecture

Status: **Draft 0.1, pre-implementation.** Date: 2026-10-17.
Provisional name "Dynamic Viewport Media (DVM)". Not a registered or official standard.

Evidence labels used throughout:
- **[V]** verified in a primary source fetched during this review (link in Sources)
- **[R]** recollection from general knowledge, NOT verified here; must be checked before anything depends on it
- **[A]** assumption / hypothesis
- **[P]** proposed DVM design

---

## A. Feasibility assessment

**Verdict: technically feasible, but the core capability already exists in a published draft, so DVM should not start as a new format.**

1. Apple has published "Rectangular Mask Payload Metadata within the QuickTime Movie File Format", **version 0.9 (Beta), dated June 9, 2025** [V]. It defines a per-frame rectangle inside a constant-size decoded frame, carried as timed metadata, explicitly aimed at changing aspect ratio over time without baked-in bars [V].
2. Apple's WWDC25 material describes this as new in visionOS 26 ("per-frame dynamic masks") [V], and Apple's AVFoundation sample documents that on **visionOS 26+** the system reads the metadata, crops to the rectangle and renders the outside as transparent [V].
3. What Apple's document deliberately does **not** do: it says it does not prescribe the visual treatment a player uses for the area outside the rectangle [V]. It defines no interpolation, no mapping from rectangle to display window, no seek rules beyond "holds until changed", and no conformance suite [V: absent from the document].
4. The user's scenario (phone-style 9:16 → 1:1 → 16:9 on an ordinary screen) needs exactly the part Apple left open: **what a normal 2D display does with the rectangle** (scale the rectangle to fill the window, i.e. a dynamic crop/"viewport-fit", versus transparency, which only makes sense where there is something behind the video).
5. Physical limit (unchanged by any metadata): a viewport can only reveal pixels that were encoded. See section 5 for the numbers; this strongly shapes canvas choice.

**Recommendation:** define DVM as a **presentation profile layered on Apple's rectangle payload**, plus authoring, validation, deterministic evaluation rules and a conformance suite. Do not mint new box types or keys for the MVP.

---

## B. Prior-art comparison

| Item | What it is | Version / date | Relevance | Status label |
|---|---|---|---|---|
| Apple Rectangular Mask Payload Metadata | `RasterRectangleValue` (6 × uint16: reference raster W/H, left, width, top, height), carried in `mebx` timed metadata | 0.9 Beta, 2025-06-09 | Essentially the DVM viewport timeline | [V] |
| Apple AVFoundation sample "Adding a display mask rectangle metadata track" | Writer sample; needs `rndr` track association; static and per-frame examples | macOS 26 / Xcode 26 listed | Only known reference writer | [V] |
| ISO/IEC 14496-12 (ISOBMFF) | Timed metadata tracks; `pasp` and `clap` sample-entry boxes; fragmented MP4 | 2020 edition cited by Apple | Container foundation. `pasp`/`clap` are static per sample entry | [V] for pasp/clap override text via FFmpeg list quoting the spec; edition details need the paid text |
| `clap` / `pasp` | Static clean aperture / pixel aspect ratio | part of ISOBMFF | Cannot express a time-varying rectangle (a static rectangle only). Apple itself points to clean aperture for the constant case | [V] |
| MP4RA registry | Registered box/sample-entry codes | live registry | Where any new code would need registering | [V] (registry exists; contents not audited) |
| W3C Media Timed Events | Requirements Note (Interest Group Note), 2019-05-16 and 2020-06-25 versions; points to DataCue in WICG | 2020-06-25 | Web exposure of in-band timed metadata | [V] |
| `HTMLVideoElement.requestVideoFrameCallback()` | Per-frame callback with media time, for canvas work | MDN documented | Enables a JS polyfill player | [V] |
| DASH / CMAF events (`emsg`), DASH-IF event schemes | In-band and MPD events; `emsg` timing anchored per fragment/track | ISO/IEC 23009-1, 23000-19 | Streaming carriage alternative; ISO/IEC 23001-18 (event message tracks) appears in a validator suite | [V] existence; semantics need paid specs |
| Apple HLS guidance in the mask doc | Notes HLS segmentation may split long metadata samples at segment boundaries; metadata must be present per resolution tier | same doc | Streaming behavior is partly specified | [V] |
| MPEG OMAF "recommended viewport" timed metadata | Author-chosen time-varying viewport for omnidirectional video | OMAF | Conceptual precedent, spherical domain | [V] (via survey paper) |
| MPEG-2 / H.264 / HEVC pan-scan rectangle | In-bitstream pan-scan signaling | — | Possible older precedent for dynamic crop | [R] verify before citing |
| Patent literature on dynamic aspect ratio / "unboxing" (e.g. US 11490047) | Stores original aspect ratio as metadata, adapts at display | — | Possible patent exposure; **not analyzed**. Needs counsel | [V] exists; [A] relevance |
| FFmpeg MOV demuxer | Generic timed-metadata support was proposed in 2016 (Nokia patch series); `pasp` priority fix 2021 | — | Whether FFmpeg surfaces `mebx` rect samples is **unverified** | [V] history; support [A] |

**Not found / not verified in this review:** any independent (non-Apple) implementation of the mask payload; any support on iOS, macOS, Android, desktop players or browsers; any MPEG or W3C work item for it; any Apple patent or licence statement attached to the document. Absence of results is not proof of absence.

---

## C. Verified capabilities and unresolved questions

**Verified [V]**
- Payload layout, units (pixels relative to a stated reference raster), big-endian, origin top-left.
- Carriage: `mebx` timed metadata, ISOBMFF or QTFF; keys `com.apple.quicktime.video.display-maskrect.mono`, `...stereo-left`, `...stereo-right` (mdta keyspace).
- `rndr` track association is **required** for the playback system to use the metadata (Apple's sample statement).
- Missing mask for a frame → show the whole frame. Empty rectangle → nothing shown.
- Samples can span many frames; a change requires a new sample; the doc recommends a dedicated metadata track.
- Rectangles beyond the raster are clipped; reader scales between reference raster and actual buffer.
- ISOBMFF note: no data-type signaling like QTFF, so readers must know the key's type.
- Different resolution tiers can reuse the same rectangle via the reference raster.

**Specification ambiguity found [V text, my reading]:** the scaling pseudocode multiplies after dividing (`(buffer / reference) * value`), while its comment says to avoid integer division collapsing to zero. Implemented literally in integers it truncates. DVM must pin down exact integer arithmetic (see D).

**Unresolved**
1. Does any platform other than visionOS 26 honor the payload? (Test on iOS/macOS AVPlayer, Safari.)
2. Does FFmpeg/ffprobe expose `mebx` samples and the `rndr` reference, and does `-c copy` remuxing preserve them?
3. Do common editors/transcoders strip the track?
4. Does an unaware player ignore the track and play the full canvas without error? [A, likely, untested]
5. Licence/patent status of Apple's format and of implementing it outside Apple platforms.
6. Browser exposure of `mebx` samples (no verified path; DataCue/WebKit is partial per the W3C note).
7. Edit-list (`elst`) interaction: must metadata share the video track's edit list?
8. How Apple's player treats samples that change on non-frame boundaries or overlap across tracks.

---

## D. Three architecture alternatives

**Alt 1 — Profile over Apple's payload (recommended).**
Reuse the `mebx` track, key `...display-maskrect.mono`, `RasterRectangleValue`, `rndr` reference. DVM adds: (a) a *presentation* section (viewport-fit scaling and optional transparent mode), (b) deterministic integer arithmetic, (c) authoring convention: smooth transitions are **compiled to per-frame samples** at authoring time, so players need no interpolator and need no new syntax, (d) validation rules and test vectors, (e) a JSON authoring schema.
- Pros: interoperates with the only known implementation; no new codes to register; reader is small.
- Cons: per-frame samples during transitions (tiny: roughly a dozen bytes of payload plus sample-table overhead per frame; to be measured, not assumed); the intent "this was a smooth transition" is lost in the file; dependence on a vendor draft that can change.

**Alt 2 — New DVM keyframe track (own key or URI-labelled metadata).**
Keyframes with easing, compact, explicit semantics.
- Pros: clean semantics, editable intent.
- Cons: incompatible with Apple's playback; needs every player to implement interpolation; new namespace to maintain. Only justified if Alt 1 is shown to be inadequate. [P]

**Alt 3 — Sidecar / manifest (JSON, WebVTT metadata track, HLS/DASH events).**
- Pros: easiest for web via `requestVideoFrameCallback` and timed events.
- Cons: not a single file; breaks on download/remux; fails the stated "one file" goal. Useful as a **secondary web delivery path**, not the base.

(Also considered: codec SEI. Apple argues for separation from the encoded frame, for reasons including no rewrite of video and per-tier reuse [V]; SEI would also tie the feature to each codec.)

---

## E. Recommended MVP architecture

```
Authoring JSON  --validate-->  Timeline model  --compile (cut / smooth→per-frame)-->  RasterRectangleValue samples
                                                                                      |
Source MP4/MOV (unchanged, stream-copied)  -------------------------------------------+--> mux: video + audio + mebx(rndr→video)
```
Layers (as in your brief):
- **Canvas** = decoded frame, untouched codec, stream copy by default; encoder reports "re-encoded: no".
- **Viewport timeline** = Apple payload samples. Evaluation at a frame's presentation time: the sample whose [start, start+duration) contains it; none → full frame (Apple rule).
- **Output** = DVM profile modes: `viewport-fit` (rectangle scaled uniformly to fit/fill the window; the DVM default on ordinary displays) and `mask` (outside transparent; for spatial/overlay environments). [P]

Deterministic arithmetic [P]: with reference raster (Rw,Rh) and buffer (Bw,Bh), `out = floor((v * B * 2 + R) / (2 * R))` computed in 64-bit integers (round half up), left/width computed from edges (`right = round(...(left+width))`) so adjacent rectangles never leave gaps or overlap by rounding. Smooth interpolation (authoring side): linear in each edge coordinate, evaluated at frame PTS in rational time, rounded once.

Evidence-driven rule: **do not hand-write `mebx`/`keys` sample-entry bytes from memory.** Phase 0 obtains a reference file (Apple's sample on macOS 26, if available) and diffs against it, or reads the QuickTime/ISOBMFF text first.

Stack candidates (decide after Phase 0): own small ISOBMFF box reader/writer for metadata (FFmpeg support unverified) + FFmpeg libs or libmpv for decode + Qt6/OpenGL for rendering; evaluator kept as a pure library so tests need no GPU.

Legacy fallback (honest version) [A]: an unaware player probably ignores the metadata track and shows the **entire canvas**, including areas the author meant to hide. Mitigations: compose the canvas so the full frame is acceptable, or ship a separate conventional rendition. No guarantee for existing players.

---

## 5. Canvas math (why canvas choice matters)

Viewport pixels are bounded by the canvas. Examples (area vs. a 1920×1080 frame = 2.07 MP):

| Canvas | 9:16 viewport | 1:1 viewport | 16:9 viewport | Decoded pixels |
|---|---|---|---|---|
| 1080×1920 (portrait only) | 1080×1920 (full) | 1080×1080 | 1080×608 (≈32% of the pixels) | 2.07 MP (1.0×) |
| 1920×1920 (square canvas) | 1080×1920 | 1920×1920 | 1920×1080 | 3.69 MP (≈1.78×) |
| 3840×2160 | 1215×2160 | 2160×2160 | 3840×2160 | 8.29 MP |

So: portrait→landscape on a portrait canvas gives a soft landscape shot (it is a crop of what was encoded; nothing is recovered). A 1920×1920 canvas keeps all three presentations at ~1080-class resolution at 1.78× decode cost. Codec level / hardware decoder limits for such frame sizes must be checked ([R]: common H.264/HEVC level limits at 1080p-class may be exceeded; verify).

| Strategy | Quality | Storage/decode | Encoder effort | Compat | Difficulty |
|---|---|---|---|---|---|
| A. Oversized canvas + viewports | Good if canvas sized well | +pixels (e.g. 1.78×) | Low | Best (single track) | Low |
| B. Reframing-aware production | Best framing | None extra | Production effort | Complements A | Process, not tech |
| C. Multiple synchronized tracks | Native-res each | ~Sum of tracks, multiple decoders | Medium | Poor sync/seek in many players | High |
| D. Auxiliary image data | Case-specific | Extra | High | Unproven | High |

**MVP choice: A, combined with B as authoring guidance.** C/D deferred.

---

## F. Proposed specification outline (DVM Profile 0.1)

1. Scope, terminology (canvas, viewport, output), conformance language
2. Normative references (ISOBMFF, QTFF, Apple payload v0.9) and status of each
3. Carriage (track layout, `rndr` reference, key, sample timing, fragmented-file rules)
4. Timeline evaluation (PTS lookup, hold, gaps, overlap/conflict rule, edit lists, seeking, rate changes)
5. Arithmetic and rounding (section E)
6. Output modes (`viewport-fit`, `mask`), fit/fill, orientation, accessibility notes
7. Defaults and error handling (missing, malformed, out-of-bounds, zero size, unknown versions)
8. Authoring schema (application-level, **not** a file format) and compilation rules
9. Legacy behavior and remux/edit survival
10. Streaming notes (segment splitting, per-tier raster)
11. Conformance: test vectors, expected-rectangle tables
12. Registry/IPR/status of each dependency

Feature labels: Apple payload reuse = *existing capability reused*; viewport-fit mode, arithmetic, conflict rule, seek semantics = *proposed DVM extension*; rotation, non-rectangular edges (Apple's extended insets exist for stereo), keyframe track = *future optional*; browser delivery, edit-list behavior, platform support = *unresolved*.

---

## G. Phased plan with acceptance tests

| Phase | Work | Acceptance test |
|---|---|---|
| 0 Experiments | Obtain/produce reference mask file; `ffprobe`/own parser inspection; `-c copy` remux; test unaware players | Written results per unresolved item 1–4, 7 |
| 1 Core library | Schema + validator; evaluator; compiler (cut, smooth); arithmetic | Unit tests: expected rectangles at listed timestamps incl. rounding edge cases; malformed inputs rejected with readable messages |
| 2 Authoring tool | Stream-copy mux with `mebx` track; `inspect` command | Output opens in ffprobe/own parser; media streams bit-identical; reports "re-encoded: no"; deterministic output hash |
| 3 Reference player | Decode + viewport-fit render; debug overlay with expected vs actual rect | 9:16→1:1→9:16 changes at specified PTS ±1 frame; A/V sync measured; pause/seek/rate inside a transition correct |
| 4 Conformance | Synthetic grid/coordinate videos; vectors; remux survival; VFR | Second independent reader passes the same vectors |
| 5 Streaming/web | fMP4/HLS splitting; JS polyfill | Seek into mid-event yields correct rectangle |

Benchmarks (CPU/GPU, memory, metadata overhead, size) are produced by measurement only.

---

## H. Highest-risk assumptions

1. Apple's payload can be written correctly without Apple tooling (mebx/keys layout not yet verified by me).
2. Non-Apple players can be built around it with no hidden Apple-side behavior.
3. `rndr` and the track survive common remuxers and editors.
4. The spec is a **Beta** draft and may change; no stated open licence or patent position.
5. Unaware players tolerate the extra track.
6. Oversized canvases fit hardware decoder/level limits on target devices.
7. Viewport-fit across a hard cut looks acceptable to creators (a UX, not a technical, risk).
8. Browser delivery needs a polyfill; no verified native path.

## Standardization & licensing (initial view)
Prefer: engage Apple (feedback on presentation semantics, arithmetic ambiguity), MPEG Systems (ISOBMFF carriage/registration through MP4RA and the file-format group), W3C Media & Entertainment IG / WICG DataCue (web exposure), AOMedia/DASH-IF/CTA WAVE (streaming events). No endorsement exists. Licence recommendation for DVM's own code/spec text: Apache-2.0 (explicit patent grant) for code and CC-BY-4.0 for spec text, **pending legal review** and Apple's terms for the underlying format.

## Sources
- Apple, Rectangular Mask Payload Metadata (v0.9 Beta, 2025-06-09): https://developer.apple.com/av-foundation/Rectangular-Dynamic-Mask-Metadata.pdf
- Apple, Adding a display mask rectangle metadata track: https://developer.apple.com/documentation/avfoundation/adding-a-display-mask-rectangle-metadata-track-to-a-movie-file.md
- Apple WWDC25 session 304: https://developer.apple.com/videos/play/wwdc2025/304/
- Apple AVFoundation overview (spec list): https://developer.apple.com/av-foundation/
- Apple stereo/ISOBMFF extensions: https://developer.apple.com/av-foundation/Stereo-Video-ISOBMFF-Extensions.pdf
- FFmpeg-devel on `pasp`/`clap` text of ISO/IEC 14496-12: https://ffmpeg.org/pipermail/ffmpeg-devel/2021-April/278789.html
- FFmpeg timed-metadata patch series (2016): https://lists.ffmpeg.org/pipermail/ffmpeg-devel/2016-August/198120.html
- MP4RA registry: https://mp4ra.org/registered-types/codecs
- W3C Media Timed Events: https://www.w3.org/TR/media-timed-events/
- MDN requestVideoFrameCallback: https://developer.mozilla.org/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback
- DASH-IF event schemes: https://dashif.org/identifiers/event_schemes/
- OMAF overview: https://arxiv.org/pdf/2203.01183
- Patent lead (not analyzed): https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/11490047
