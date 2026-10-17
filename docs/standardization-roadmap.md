# Standardization and adoption roadmap

Principle: **collaborate, don't fork.** The mask payload already exists in an Apple draft; DVM's value is the presentation profile, exact arithmetic, authoring tooling and conformance tests. No endorsement, certification or patent clearance exists for any of this.

## Stages

1. **Evidence (now → Phase 0/1):** verify carriage with a reference file; run remux/legacy tests; publish results in `docs/compatibility.md`.
2. **Independent reader:** a second implementation by another developer, written from the spec only, passing the same vectors. No interoperability claim before this.
3. **Upstream feedback:** send Apple (a) the integer-arithmetic ambiguity in the scaling pseudocode, (b) the missing presentation/seek semantics, (c) requests for an open licence/patent statement.
4. **Registration/clarification:** discuss with the ISOBMFF maintainers whether the payload and `rndr` usage should be described in an ISOBMFF amendment or registered via MP4RA. Do not register anything unilaterally.
5. **Web:** propose a JS reference polyfill (using `requestVideoFrameCallback`) and discuss in-band exposure with the W3C Media & Entertainment community and WICG DataCue.
6. **Streaming:** specify segment-boundary splitting and per-tier raster handling with DASH-IF / CTA WAVE / AOMedia style bodies as appropriate.

## Organizations to approach (candidates only; none contacted)

| Organization / group | Why |
|---|---|
| Apple (AVFoundation / QuickTime spec owners) | Owner of the payload draft |
| ISO/IEC MPEG Systems (ISOBMFF maintainers); MP4RA | Container carriage and registration |
| W3C Media & Entertainment Interest Group; WICG (DataCue) | Web exposure of timed metadata |
| DASH-IF, CTA WAVE, AOMedia | Streaming events/profiles, interoperability points |
| FFmpeg, GStreamer, mpv communities | Open-source reader support |

## Governance (draft)

- **Spec changes:** open an issue using the "spec change" label; a PR must update spec text, tests and the conformance vectors together; maintainers record decisions in `docs/adr/`.
- **Versioning/deprecation:** see spec §10. A feature is deprecated for one MINOR release before removal within a MAJOR.
- **Test vectors:** each vector has a source file, expected rectangle table, generation script, and a note of which implementations pass.
- **Interop bugs:** use `.github/ISSUE_TEMPLATE/interop-bug.md`.

## Licensing recommendation (pending counsel)

Apache-2.0 for code (explicit patent grant, permissive for ecosystem adoption); CC-BY-4.0 for DVM-original spec text. Open items: Apple's terms for the underlying payload, and the unanalyzed patent-literature lead in `prior-art.md`.
