# 5. Data Collection

Corresponds to the reference manual's "SDK usage". This is the core chapter: collecting with
`lerobot-record` and writing out a `LeRobotDataset`.

## 5.1 How collection works

`taccap_gripper` **needs no teleoperator** — recording is self-driven.

The data uses **shifted-frame pairing**: the action **leads the observation by one step** — the
observation from step *t-1* is paired with the pose from step *t* (the EEF TCP pose plus the
normalised `gripper.pos`, and the **headset pose** too when the [headset camera](#57) is on) as
its action. Each episode is therefore one frame shorter (the first
has no predecessor). The reset phase between episodes is a passive wait: just reposition the
setup, no teleoperation involved.

!!! note "No --teleop.* on the command line"
    Because it is self-driven, recording commands carry **no `--teleop.*` arguments at all**.

## Before recording: preview once with `lerobot-teleoperate` {#preview}

`lerobot-teleoperate` **only reads and previews the devices — it writes nothing**, so running it
costs you nothing. Recording does cost something: a ruined episode has to be redone. And most of
what goes wrong (a camera stream that never came up, a tracker with no pose, `gripper.pos` not
reaching 1.0, the two arms swapped) is obvious at a glance in the preview.

Unlike [4.3 End-to-end smoke test](04-calibration.md#43), **everything stays enabled here** —
the point is to confirm the whole chain is present.

```bash
lerobot-teleoperate \
    --robot.type=bi_taccap_gripper \
    --robot.enable_head_camera=false \
    --fps=30 \
    --display_data=true \
    --show_trajectory=true
```

**Single gripper**: use `--robot.type=taccap_gripper` plus `--robot.side=left|right`;
everything else is the same.

Check each of these in Rerun:

| What | Expected |
|---|---|
| Both tactile streams | Both showing; the texture changes clearly when pressed |
| Wrist camera | Showing, with no cable or clutter in the view |
| `gripper.pos` | **1.0** wide open, **0.0** closed — short of 1.0, see [4.1.3](04-calibration.md#413) |
| The EE marker and trail in `/world` | Moves smoothly with the gripper; no jumps, no freezing — **keep the tracker in the headset's view**, since anything blocking it loses tracking |
| Bimanual: the two trails | Independent, and each on the correct hand — not swapped |

`Ctrl+C` once it all looks right, then record below. If you use the headset camera, set
`--robot.enable_head_camera` to `true` here as well so you preview that too (see [§5.7](#57)).

## 5.2 Recording with both grippers

Devices are **auto-discovered by the serial rules** — you never list gripper, tactile or camera
serials. Tactile sensors, wrist cameras and trackers each match left/right by the same rules.

```bash
lerobot-record \
    --robot.type=bi_taccap_gripper \
    --robot.enable_tracker=true \
    --robot.enable_head_camera=false \
    --display_data=true \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=1 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --dataset.episode_time_s=120 \
    --dataset.reset_time_s=60 \
    --dataset.single_task='Pick up the object'
```

### Parameter reference {#params}

`lerobot-record` parameters fall into three groups: **dataset** (`--dataset.*`), **recording
control** (top level), and **device** (`--robot.*`). For the complete definitions see lerobot's
official [recording guide](https://huggingface.co/docs/lerobot/v0.5.1/en/il_robots#record-a-dataset)
(matching the lerobot 0.5.1 baseline this project customises).

#### Dataset parameters `--dataset.*`

| Parameter | Default | Meaning |
|---|---|---|
| `repo_id` | **required** | Dataset identifier, `{HF username}/{dataset name}`, e.g. `Xense/pick_demo` |
| `single_task` | **required** | Short, accurate task description, written to `meta/tasks` (e.g. `'Pick up the object'`) |
| `root` | `$HF_LEROBOT_HOME/repo_id` | Local storage directory; defaults to the cache path |
| `fps` | `30` | Sampling (recording) frame-rate cap |
| `episode_time_s` | `120` | Recording length per episode (seconds) |
| `reset_time_s` | `60` | Reset time between episodes (seconds), a passive wait while you reset the scene |
| `num_episodes` | `50` | Number of episodes to record |
| `video` | `true` | Encode frames as video (mp4) |
| `push_to_hub` | `false` | Local-only by default; set `true` explicitly to upload |
| `private` | `false` | Upload as a private Hub repo |
| `tags` | none | Tags for the Hub dataset |
| `streaming_encoding` | `true` | Live streaming encode (see [§5.5](#55)) |
| `vcodec` | `auto` | Video encoder (`h264`/`hevc`/`libsvtav1`/`auto`/a hardware encoder) |
| `encoder_threads` | auto | Threads per encoder instance |
| `encoder_queue_maxsize` | `30` | Buffered frames per camera (~1 s @ 30 fps); back-pressure drops the oldest when encoding falls behind |
| `video_encoding_batch_size` | `1` | Episodes accumulated before batch encoding (1 = encode immediately) |

!!! note "Spell out the parameters that matter"
    Always set `fps=30`, `episode_time_s=120`, `reset_time_s=60` and `push_to_hub=false`
    explicitly in your commands, so a different checkout's defaults cannot change what you
    collect. The examples above spell out `--robot.enable_head_camera=false` for the same reason:
    `false` is the default, and writing it keeps the switch visible. Set it to `true` to record
    the headset's stereo view and pose as well (see [§5.7](#57)); that needs **PC Service >=
    v0.2.0 on an amd64 host**, so leave it `false` on arm64.

!!! note "`fps` vs. sensor frame rate"
    `fps` is the **recording sample rate**, not a sensor ceiling. The visuotactile sensors
    themselves run at 120 Hz ([hardware specs](hardware.md#specs)); recording at a lower `fps` is
    a **usage choice** and does not change the sensor's specification.

#### Recording control (top-level parameters)

| Parameter | Default | Meaning |
|---|---|---|
| `--robot.type` | **required** | `taccap_gripper` (single) / `bi_taccap_gripper` (bimanual) |
| `--fps` | `30` | **Main loop** rate (device reads and preview). Separate from `--dataset.fps` (the recording sample rate) — two parameters, usually set the same |
| `--display_data` | `false` | Show camera streams and the 3D view in Rerun |
| `--show_trajectory` | `true` | Overlay the 3D pose + trajectory in Rerun (needs `display_data` and a `tcp.*`) |
| `--display_compressed_images` | `false` | Whether to JPEG-compress images before showing them in Rerun. **Off by default** — the encoding happens on the record loop and eats a large share of the frame budget; it only pays off when the Rerun viewer is on another machine (`--display_ip`) |
| `--display_image_every_n` | `1` | Refresh the camera tiles only every N frames (scalars always stay at full rate). **A last resort**, for a loop that still overruns — it is the only option here that changes what the operator sees |
| `--play_sounds` | `true` | Spoken announcements of recording events |
| `--resume` | `false` | **Continue recording** into an existing dataset |
| `--teleop.*` | — | Teleoperator; **not needed** — the XTac-UMI G1 is self-driven |

#### Device parameters `--robot.*` (XTac-UMI G1 specific)

| Parameter | Default | Meaning |
|---|---|---|
| `--robot.side` | auto | `left`/`right`, required when both grippers are connected |
| `--robot.role` | `leader` | `follower` binds the slave gripper |
| `--robot.enable_tracker` | `true` | Off records tactile + gripper only (no pose) |
| `--robot.tracker_serial` | unset | Pin the tracker SN, bypassing automatic side matching |
| `--robot.enable_wrist_camera` | `true` | Turn the wrist camera off |
| `--robot.wrist_camera_width/_height/_fps` | — | Wrist camera resolution / frame rate |
| `--robot.enable_head_camera` | `false` | Record the Pico4 Ultra Enterprise **headset camera** — see [§5.7](#57) |
| `--robot.head_camera_eyes` | `both` | `both` records each eye as its own key; `left` / `right` records one |
| `--robot.head_camera_width/_height` | `1024` / `768` | **Per-eye** size; only `1024x768` or `1280x960` are accepted |
| `--robot.head_camera_fps` | `30` | Head camera recording frame rate |
| `--robot.head_camera_pair_max_skew_ms` | `20.0` | Max timestamp gap still counted as one stereo capture when the eyes' sequence numbers differ |
| `--robot.tactile_fps` | `30` | Tactile recording frame rate |
| `--robot.tactile_output_types` | `["rectify"]` | Tactile stream **written to disk**, **exactly one** |
| `--robot.tactile_display_output_types` | `["difference"]` | Extra tactile streams that are **display-only**, never recorded |
| `--robot.tactile_diff_gain` | `1.0` | Gain of the `difference` image (display only) |
| `--robot.expected_tactiles_per_side` | — | Assert the tactile count per side |

Once the Pico4 Ultra Enterprise tracker is powered on, the 6-DoF pose is **recorded
automatically** — the tracker matches this unit's side from the digit before its serial's trailing `G`
(odd-left / even-right).

!!! tip "Tactile + gripper only"
    Add `--robot.enable_tracker=false` to turn off pose recording.

!!! tip "Non-conforming tracker serial / flaky PC-service enumeration"
    Pin the serial directly with `--robot.tracker_serial=<SN>` — it is used **verbatim**, with no
    enumeration and no validation (a typo surfaces as a device-not-found error at connect time).
    Leave it unset (the default) for auto-discovery.

## 5.3 Recording with a single gripper

To record just one, use `--robot.type=taccap_gripper`; every other parameter is as above. A lone
gripper is selected automatically; with both plugged in, pick one with `--robot.side=left|right`.

```bash
lerobot-record \
    --robot.type=taccap_gripper \
    --robot.side=right \
    --robot.enable_tracker=true \
    --robot.enable_head_camera=false \
    --display_data=true \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=1 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --dataset.episode_time_s=120 \
    --dataset.reset_time_s=60 \
    --dataset.single_task='Pick up the object'
```

## 5.4 What each frame records {#54}

| Key | Source | Enabled by | Shape / type |
|---|---|---|---|
| `tcp.x`, `tcp.y`, `tcp.z` | Tracker → EEF TCP position | `--robot.enable_tracker` (default `true`) | float (m) |
| `tcp.r1`..`tcp.r6` | Same, as a 6-D rotation | as above | float |
| `gripper.pos` | Jaw encoder | `--robot.enable_gripper` (default `true`) | float ∈ [0, 1] |
| `tactile_left` / `tactile_right` | The two visuotactile sensors | **always recorded**, no switch | uint8, about `(400, 700, 3)` |
| `wrist_cam` | Wrist camera | `--robot.enable_wrist_camera` (default `true`) | uint8 `(H, W, 3)` |
| `left_head` / `right_head` | Headset stereo, **one key per eye** | `--robot.enable_head_camera` (default `false`) | uint8, `(768, 1024, 3)` by default |
| `head_camera.x/y/z` | Headset position (same frame as `tcp.*`), **also an action** | as above | float (m) |
| `head_camera.r1..r6` | Headset orientation as a 6-D rotation, **also an action** | as above | float |
| `imu.accel.{x,y,z}` | Gripper IMU acceleration | `--robot.enable_imu` (default `false`, **reserved, not recorded**) | float (m/s²) |
| `imu.gyro.{x,y,z}` | Gripper IMU angular rate | as above | float (rad/s) |
| `imu.mag.{x,y,z}` | Gripper IMU magnetometer | as above | float (µT) |

!!! note "6-D rotation convention"
    What is stored is the **first two columns** of the rotation matrix **R** — **columns, not
    rows**:

    ```text
    R = ⎡ r1  r4  · ⎤     column 1 (r1,r2,r3) = the body X axis, in world coordinates
        ⎢ r2  r5  · ⎥     column 2 (r4,r5,r6) = the body Y axis, in world coordinates
        ⎣ r3  r6  · ⎦     column 3 = the cross product of the first two
    ```

    **R maps body → world**: it takes gripper-body coordinates into the world frame, so the two
    columns read directly as the gripper's **X** and **Y** axes as unit vectors in world
    coordinates. The third column (the Z axis) is the cross product of the first two, which is why
    six numbers reconstruct the full rotation and the third column is not stored.

    The world frame is gravity-aligned, **X forward / Y left / Z up** (see
    [startup and frame alignment](03-host-hardware.md#pico-frame)). `tcp.*` and `head_camera.*`
    follow the same convention.

!!! info "`tcp.*` records the gripper tip, not the tracker"
    The tracker is bolted to the gripper's handle, about 195 mm from the **two-finger midpoint**.
    Before anything is written to disk it is multiplied by a built-in rigid mount transform
    (measured off the CAD assembly, one per side), so `tcp.*` is the **EEF TCP pose**.

    That transform is **body-fixed** — it turns with the gripper and holds in any orientation, so
    it does not matter which way you start out. The TCP is the two-finger midpoint, which does not
    move when the jaw opens and closes symmetrically, so it is independent of `gripper.pos`. To
    see it for yourself, lay the gripper flat and check that the EE marker in Rerun's `/world`
    sits at the two-finger midpoint → [4.4 3D trajectory visualisation](04-calibration.md#44).

!!! tip "The IMU is a reserved capability — the standard flow does not record it"
    The gripper has an IMU and the collection program supports recording it, but the **standard
    flow leaves it off**. The `imu.*` rows above document the format; a dataset collected by
    following this manual **will not contain those 9 columns**.

    If you do need inertial data, add `--robot.enable_imu=true` (the same switch on bimanual rigs
    applies to both sides at once, with keys prefixed `left_` / `right_`).

    Turning them on raises the `observation.state` dimension accordingly: 10 → 19 for a single
    gripper, 20 → 38 for bimanual. Leave it off if you do not need inertial data and save 9
    columns.

**Tuning the observation keys**:

- **Tactile** → `tactile_left` / `tactile_right`; the rectified image is landscape `(400,700,3)`
  (width and height are derived automatically — **do not hard-code them**). Tune with
  `--robot.tactile_fps` / `--robot.tactile_output_types`; assert the per-side count with
  `--robot.expected_tactiles_per_side`.

!!! danger "What lands on disk is `rectify`, not the image you see in Rerun"
    The two tactile streams are **deliberately different**:

    - **On disk** = `--robot.tactile_output_types`, default `rectify` — the raw image with **no
      baseline subtraction**, keeping everything the sensor saw. **Exactly one type** (each sensor
      maps to one video key in the dataset); more than one is an error pointing you at the display
      stream.
    - **Display** = `--robot.tactile_display_output_types`, default `difference` — an enhanced
      difference image against the **baseline captured at sensor init**. Contact is far easier to
      read there, which is why that is what Rerun shows the operator; the key is shaped
      `tactile_left_difference` and is **not** in `observation_features`, so it never reaches
      disk.

    The difference image is **destructive**: the baseline is grabbed when the sensor initialises,
    so **any force pressing on the gel at connect time is subtracted out of the whole session**.
    That is why it is display-only — do not switch `--robot.tactile_output_types` to `difference`
    just because it looks clearer.

    `--robot.tactile_diff_gain` (default `1.0`) only affects the display stream's gain. The
    factory value of 1.5 is noisy on this gel and clips; it scales signal and noise alike, so it
    **does not change the signal-to-noise ratio** — it only leaves headroom.
- **Wrist camera** → `wrist_cam`; skip it with `--robot.enable_wrist_camera=false`, tune with
  `--robot.wrist_camera_width/_height/_fps`.
- **Headset camera** → `left_head` / `right_head` plus `head_camera.*`; **off by default**, turn
  it on with `--robot.enable_head_camera=true` — see [§5.7](#57).
- **Role** → `--robot.role=follower` binds the slave unit (default `leader`).

## 5.5 Recording options: streaming encoding and encoder warm-up {#55}

Video keys (tactile + wrist camera) are **encoded live during collection** rather than stored as
PNGs and encoded at the end of an episode, so **there is almost no wait when an episode
ends**. On by default (`--dataset.streaming_encoding=true`):

```bash
lerobot-record \
    --robot.type=taccap_gripper --robot.side=right \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=20 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --dataset.reset_time_s=60 \
    --dataset.episode_time_s=120 \
    --dataset.single_task='Pick up the object' \
    --dataset.streaming_encoding=true \
    --dataset.encoder_threads=2 \
    --dataset.vcodec=auto
```

- One `_CameraEncoderThread` per camera, fed raw frames through a bounded queue
  (`--dataset.encoder_queue_maxsize`, roughly one second of frames). When an encoder falls behind
  it **drops the oldest frame and warns** rather than blocking the collection loop.
- `--dataset.vcodec=auto` prefers hardware encoding where available. An NVIDIA GPU on the
  collection host is recommended so the GPU H.264 encoder can take the CPU load off encoding
  several live video streams. Recording still works without one, but high resolutions or many
  cameras make it likelier that encoding falls behind.

!!! note "Encoder warm-up"
    Encoder initialisation takes ~25 ms, and leaving it until the first frame blows that frame's
    `fps` budget badly. So each episode **warms the encoders up first** and waits until all are
    ready before recording starts — the first frame no longer pays initialisation cost.

## 5.6 Episodes and resets

- Record several episodes in one run with `--dataset.num_episodes=N`.
- Between episodes the reset is **passive**: reposition the setup, no teleoperation.
- lerobot's keyboard controls work while recording (re-record the current episode, end it early,
  and so on, following the usual `lerobot-record` conventions).

!!! tip "Want to collect *good* data?"
    Running the command is only the first step. Do read
    [Best Practices](best-practices.md) — origin discipline, tactile contact, demonstration
    consistency and incremental verification are what actually decide the quality of what lands on
    disk.

## 5.7 Optional: the headset camera (first-person view) {#57}

**Off by default.** Turning it on records the Pico4 Ultra Enterprise headset's **own stereo
camera** plus the headset's pose — the operator's first-person view and where they were looking.
Both `taccap_gripper` (single) and `bi_taccap_gripper` support it, with the same flags.

```bash
lerobot-record \
    --robot.type=bi_taccap_gripper \
    --robot.enable_tracker=true \
    --robot.enable_head_camera=true \
    --display_data=true \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.single_task='Pick up the object' \
    --dataset.fps=30 \
    --dataset.push_to_hub=false
```

It produces three groups of keys:

| Key | Meaning |
|---|---|
| `left_head` / `right_head` | Headset camera, **one video key per eye**, `(768, 1024, 3)` each by default |
| `head_camera.x/y/z` | Headset position (m); **also in the action** |
| `head_camera.r1..r6` | Headset orientation, 6-D rotation (same convention as `tcp.*`); **also in the action** |

!!! warning "`left_` / `right_` here means the **eyes**, not the arms"
    On a bimanual rig `{side}_wrist` and `{side}_tcp.*` are per-**arm**, but there is only one
    headset: `left_head` / `right_head` are the headset's **left and right eye**. By the same
    token, enabling the head camera on two single-gripper processes gives you **the same stream
    from the same headset twice**, not two independent views.

### Prerequisites

1. **XenseVR PC Service ≥ v0.2.0** (amd64). v0.1.0 drops the `0x30` messages that carry camera
   frames; arm64 hosts are currently pinned to v0.1.0, i.e. **no head camera there**. See
   [2.4 One-shot install](02-environment.md).
2. **The headset app must be streaming.** The camera and the trackers **share one SDK
   connection**, so the headset has to be connected to the PC Service (see
   [3.5 Start the XenseVR PC Service](03-host-hardware.md#35)). Conversely, turning the camera off
   does not drop the trackers' connection, and vice versa.

### Resolution and recording a single eye

`--robot.head_camera_width/_height` accept **only `1024x768` (default) and `1280x960`**. Anything
else is an error rather than a silent downgrade, and so is a first frame whose size disagrees with
the config — rescaling would quietly change the recorded field of view. Both modes are 4:3,
matching the sensor (PICO's camera-access API caps a frame at 2328x1748, which is also 4:3, so a
16:9 request would be a crop or a stretch rather than more field of view).

`--robot.head_camera_eyes=left` (or `right`) records one eye: half the JPEG decoding, half the
encoder load, and one head video key in the dataset instead of two.

!!! danger "Changing the size or the eye selection changes the data"
    Either change alters the recorded frame, so **episodes either side of it are not comparable**.
    Decide before you start recording, and note it in [Data management](data-management.md).

### Pairing the two eyes

The eyes arrive as **two independent messages**, and once they are separate video keys a
mismatched pair leaves **no trace in the data**. So each frame the two eyes' newest frames are
compared: identical sequence numbers are a definitive match; otherwise their timestamps must agree
within `--robot.head_camera_pair_max_skew_ms` (default 20 ms, against a ~33 ms frame period at
30 fps). Exceeding it does not stop recording — it raises a **rate-limited warning naming the
measured skew**, so the condition is visible rather than silently recorded.

!!! note "Why the frames are collected on a background thread"
    Reading the SDK straight from the record loop caught one eye updated and the other not on
    **7% of frames** (599 samples; the left eye was consistently one or two ahead). A background
    thread now polls at 120 Hz and holds a pair back until both halves are in — measured 7% to 0%,
    with no loss of frame rate.

### Pose and visualisation

`head_camera.*` is the **headset pose**, remapped into the **same gravity-aligned world frame as
`tcp.*`** (the same Pico→world transform the tracker uses). Head and hands are therefore directly
comparable and can be drawn in one 3D scene: with `--display_data=true`, the `/world` view gains
an amber `HEAD` marker (see [4.4](04-calibration.md#44)).

!!! note "The headset pose is both an observation and an action"
    `head_camera.*` is written to both `observation` and `action`, taking part in the
    [shifted-frame pairing](#51) exactly as `tcp.*` does — **where the operator looked while
    demonstrating is part of the demonstration**, something a policy is meant to reproduce rather
    than mere context. It carries the same position + 6-D rotation layout as `tcp.*`, so it can be
    commanded the same way.

    On a bimanual rig it is **unprefixed and appears once** — one headset serves both arms, unlike
    `{side}_tcp.*`, which that rig reports twice.

    Image data stays out of the action; it lives in the observation only.

!!! note "Dimension change"
    Enabling it adds 9 dimensions (the headset pose) to `observation.state`: 10 → 19 for a single
    gripper, 20 → 29 for bimanual. `--robot.enable_imu=true` adds on top of that as usual.

Next → [Best Practices](best-practices.md) → [Dataset & Examples](06-dataset.md)
