# Phone preview (M0 side-track)

A throwaway feasibility page that runs **in the phone's browser**: the user records or picks a
short video of **adults** (training session), and the page draws a skeleton plus a stable number
per person, and a rough movement number per person. Nothing is uploaded; inference runs on-device.

This is **not** the product pipeline (that is Python on the nursery laptop, chosen in M0) and must
never be used with video of children.

- Published as a private claude.ai artifact: https://claude.ai/artifact/Ki9gjCYBS18VLn4BSyTvBQ
- Library: TensorFlow.js 4.22.0 (Apache-2.0), loaded from jsDelivr.
- Model: MoveNet MultiPose Lightning v1 (Google, Apache-2.0), TF.js graph model, fixed 256x256 int32
  input, output `[1, 6, 56]` (17 keypoints x (y, x, score) + box (ymin, xmin, ymax, xmax, score)).
  Taken from the npm package `@vladmandic/human-models@3.0.4` (MIT), files
  `models/movenet-multipose.{json,bin}`. The artifact host does not serve `.bin`, so the weights
  are published as base64 text (`models/movenet-multipose.weights.txt`) and decoded in the page.
  - `movenet-multipose.json` sha256 `3c76e18669efc50148b28d0be97db491dc055cb2be292d248ec129031c5aa5de`
  - `movenet-multipose.bin`  sha256 `19828492d4a69c2e08915916c70f4d22c421c3b59152dfa5601834fc54f54118`
- Model files are not committed (see `.gitignore`). To rebuild them:
  `npm pack @vladmandic/human-models@3.0.4 && tar -xzf vladmandic-human-models-3.0.4.tgz package/models/movenet-multipose.json package/models/movenet-multipose.bin && base64 -w0 package/models/movenet-multipose.bin > movenet-multipose.weights.txt`
- Tracker: greedy matching on IoU + constant-velocity predicted centre; a track gets a number
  after 3 matched frames and is dropped after 1 s unseen. Movement = EMA-smoothed hip-centre
  speed in body-heights per second (box height), gaps > 0.5 s ignored.
- Known limits: max 6 people; live camera is not available inside artifacts for this account,
  so the page analyses a recorded clip at playback speed.
