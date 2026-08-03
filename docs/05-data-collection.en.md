# 5. Data Collection

Corresponds to the reference manual's "SDK usage". This is the core chapter: collecting with
`lerobot-record` and writing out a `LeRobotDataset`.

## 5.1 How collection works

`taccap_gripper` **needs no teleoperator** (`RecordConfig.__post_init__` allows it to have
`teleop=None`). Recording is handled by the dedicated `self_driven_record_loop` in
`lerobot_record.py`, routed there through `SELF_DRIVEN_RECORD_ROBOTS`.

**Shifted-frame pairing**: each record pairs the observation from step *t-1* with the pose from
step *t* (the EEF TCP pose plus the normalised `gripper.pos`) as its action — so the action
**leads the observation by one step** and is a genuine "move to the next step" target rather than
a degenerate same-frame pose. One frame is dropped per episode (the first has no predecessor). The
reset phase between episodes is a passive wait: just reposition the setup, no teleoperation
involved.

!!! note "No --teleop.* on the command line"
    Because it is self-driven, recording commands carry **no `--teleop.*` arguments at all**.

## 5.2 Recording with a single gripper

Devices are **auto-discovered by the serial rules** — you never list gripper, tactile or camera
serials. A lone gripper is selected automatically; with both plugged in, pick one with
`--robot.side=left|right`.

```bash
lerobot-record \
    --robot.type=taccap_gripper \
    --robot.side=right \
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
    collect.

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
| `--display_compressed_images` | `true` | Display via JPEG in Rerun to save memory; set `false` for lossless |
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
| `--robot.tactile_fps` | `30` | Tactile recording frame rate |
| `--robot.tactile_output_types` | `["rectify"]` | Tactile stream **written to disk**, **exactly one** |
| `--robot.tactile_display_output_types` | `["difference"]` | Extra tactile streams that are **display-only**, never recorded |
| `--robot.tactile_diff_gain` | `1.0` | Gain of the `difference` image (display only) |
| `--robot.expected_tactiles_per_side` | — | Assert the tactile count per side |

Once the Pico4 Ultra Enterprise tracker is powered on, the 6-DoF pose is **recorded
automatically** — the tracker matches this unit's side from the second-to-last digit of its serial
(odd-left / even-right).

!!! tip "Tactile + gripper only"
    Add `--robot.enable_tracker=false` to turn off pose recording.

!!! tip "Non-conforming tracker serial / flaky PC-service enumeration"
    Pin the serial directly with `--robot.tracker_serial=<SN>` — it is used **verbatim**, with no
    enumeration and no validation (a typo surfaces as a device-not-found error at connect time).
    Leave it unset (the default) for auto-discovery.

## 5.3 Recording with both grippers

Use `--robot.type=bi_taccap_gripper` to record both grippers at once; every other parameter is as
above. Tactile sensors, wrist cameras and trackers each match left/right by the same rules.

```bash
lerobot-record \
    --robot.type=bi_taccap_gripper \
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

| Key | Source | Shape / type |
|---|---|---|
| `tcp.x`, `tcp.y`, `tcp.z` | Tracker pose → **EEF TCP** after the mount transform | float (metres) |
| `tcp.r1`..`tcp.r6` | The EE's 6-D rotation | float |
| `gripper.pos` | XTac-UMI G1 encoder, normalised | float ∈ [0, 1] |
| `imu.accel.{x,y,z}` (off by default) | XTac-UMI G1 IMU | float (m/s²) |
| `imu.gyro.{x,y,z}` (off by default) | XTac-UMI G1 IMU | float (rad/s) |
| `imu.mag.{x,y,z}` (off by default) | XTac-UMI G1 IMU | float (µT) |
| `tactile_left` / `tactile_right` | Rectified visuotactile image | uint8, about `(400, 700, 3)` |
| `wrist_cam` | Wrist camera | uint8 `(H, W, 3)` |

!!! note "6-D rotation convention"
    `r1..r3` is the rotation matrix's first column, `r4..r6` its second column.

!!! info "`tcp.*` records the gripper tip, not the tracker"
    The tracker is bolted to the gripper's handle, about 195 mm from the **two-finger midpoint**.
    Before anything is written to disk it is multiplied by a built-in rigid mount transform
    (measured off the CAD assembly, one per side), so `tcp.*` is the **EEF TCP pose**.

    That transform is **body-fixed** — it turns with the gripper and holds in any orientation, so
    it does not matter which way you start out. The TCP is the two-finger midpoint, which does not
    move when the jaw opens and closes symmetrically, so it is independent of `gripper.pos`. To
    see it for yourself, look at the EE and TRACKER frames and the line between them in Rerun's
    `/world` → [4.4 3D trajectory visualisation](04-calibration.md#44).

!!! tip "IMU is off by default"
    The 9 `imu.*` channels above are **off by default**; add `--robot.enable_imu=true` to record
    them (the same switch on bimanual rigs applies to both sides at once, with keys prefixed
    `left_` / `right_`).

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
- **Role** → `--robot.role=follower` binds the slave unit (default `leader`).

## 5.5 Recording options: streaming encoding and encoder warm-up {#55}

Video keys (tactile + wrist camera) are **encoded live during collection** rather than stored as
PNGs and encoded at the end of an episode, so `save_episode()` is near-instant. On by default
(`--dataset.streaming_encoding=true`):

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
    Opening a PyAV container plus codec context takes ~25 ms, and doing it lazily on the first
    frame blows that frame's `fps` budget badly. So each episode calls
    `prepare_episode_recording()` first, warming the encoder threads and opening codec contexts at
    the `(H, W, C)` each video key declares, blocking until all are ready — the first frame no
    longer pays initialisation cost.

## 5.6 Episodes and resets

- Record several episodes in one run with `--dataset.num_episodes=N`.
- Between episodes the reset is **passive**: reposition the setup, no teleoperation.
- lerobot's keyboard controls work while recording (re-record the current episode, end it early,
  and so on, per upstream `lerobot-record`).

!!! tip "Want to collect *good* data?"
    Running the command is only the first step. Do read
    [Best Practices](best-practices.md) — origin discipline, tactile contact, demonstration
    consistency and incremental verification are what actually decide the quality of what lands on
    disk.

Next → [Best Practices](best-practices.md) → [Dataset & Examples](06-dataset.md)
