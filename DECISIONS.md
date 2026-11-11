# Decisions log

Each significant choice, the alternatives considered, and why. Newest last.

## 2026-11-11 — Project lives in `nursery-observer/` inside the website repo (temporary)
- **Chosen:** keep files in a subfolder of `mk-running-lab-website` on branch `claude/sweet-sagan-tuw93n`.
- **Alternatives:** separate private repo `nursery-observer` (preferred, approved by Mohamed).
- **Why:** this cloud session can only reach the website repo and cannot create repositories.
  Move everything (with history) to the private repo as soon as it exists.

## 2026-11-11 — Phone preview before the laptop is available
- **Chosen:** a browser page (private claude.ai artifact) that analyses a recorded clip on the phone,
  with MoveNet MultiPose Lightning (Apache-2.0) on TensorFlow.js (Apache-2.0) and a simple
  IoU tracker. Adults only (training session), no children.
- **Alternatives:** wait for the laptop; upload adult video to the cloud session and run Python there;
  MediaPipe Pose Landmarker in the browser (needs WebAssembly files served from the artifact,
  riskier inside the artifact sandbox).
- **Why:** Mohamed can test today with only a phone; video never leaves the phone (G5).
  **Cost:** max 6 people, lighter model than the final one, no live camera inside artifacts.
  This is a feasibility preview only; the product's pose/tracking stack is still chosen in M0.
