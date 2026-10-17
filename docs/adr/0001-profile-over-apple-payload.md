# ADR 0001: DVM is a profile over Apple's rectangular mask payload

- Status: **Accepted (provisional)**, 2026-10-17
- Revisit if: Phase 0 shows the payload cannot be written/read outside Apple tooling, Apple changes the payload, or licensing blocks implementation.

## Context

The goal is one video file whose visible rectangle changes over time without baked-in bars. Research (`docs/prior-art.md`) found that Apple published a per-frame rectangular mask payload (v0.9 Beta, 2025-06-09), carried in `mebx` timed metadata with a required `rndr` track reference, and that visionOS 26 renders it. It deliberately leaves out: output mapping on ordinary displays, interpolation, exact rounding, seeking semantics, conformance tests.

## Options

1. **Profile over Apple's payload** (chosen).
2. New DVM keyframe track with own key and easing.
3. Sidecar/manifest (JSON, WebVTT, HLS/DASH events).
(Codec SEI was considered and dropped: ties the feature to each codec and forces re-writing video to change geometry.)

## Decision

Reuse the payload, key, track structure and `rndr` reference unchanged. DVM adds only: output modes (`viewport-fit` contain/cover; `mask` optional), exact integer arithmetic, authoring-time compilation of smooth transitions into per-frame samples, conflict/malformed rules, a JSON authoring schema, and a conformance suite. No new box types, keys or codes.

## Consequences

+ Interoperates with the only known implementation; nothing to register.
+ Players need no interpolation logic.
+ Small, testable surface.
− Transition intent is not recoverable from the file (only the sampled result).
− Per-frame samples during transitions (≈12 payload bytes each plus sample-table overhead; to be measured).
− Depends on a Beta vendor document with unknown licence/patent terms.
− Unaware players probably show the full canvas.

## Guardrails

- Do not write `mebx`/`keys` bytes from memory; derive from a reference file or the spec text.
- Do not claim any platform support beyond what is verified (visionOS 26 only, per Apple docs).
- Contribute findings (arithmetic ambiguity, presentation semantics) upstream rather than forking the format.
