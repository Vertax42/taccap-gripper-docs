# 4. Calibration & Smoke Test

Extends the reference manual's "test the network connection". Calibrate the gripper first, then
work through the checks that confirm the whole chain is alive.

## 4.1 Gripper calibration (zero + travel span) {#41}

### 4.1.1 Why it is required

`gripper.pos` in the dataset is a **normalised opening**: `0.0` fully closed, `1.0` fully open.
Those endpoints are not computed — they are two numbers written into MCU flash by calibration:

| Endpoint | Source | Command |
| --- | --- | --- |
| `0.0` closed | encoder zero | `Cmd::SetEncoderZero` |
| `1.0` open | that gripper's travel span (encoder max) | `Cmd::EncoderMaxCal` (firmware >= V2.1) |

**What happens without the travel span.** The software falls back to dividing by the config
constant `gripper_open_rad` (default `1.7`) — one number standing in for every gripper ever
built. Real travel varies per unit: a measured one came out at **1.1486 rad (65.8°)**, so under
the fallback a fully open jaw reads `1.1486 / 1.7 = 0.676` and **never reaches 1.0** — with a
ceiling that differs from unit to unit. A policy's notion of "fully open" then does not match the
physical motion.

!!! danger "Calibrating one side of a bimanual rig is worse than calibrating neither"
    With neither calibrated the two scales at least agree. Calibrating one leaves
    `left_gripper.pos` and `right_gripper.pos` on **different scales** — the same physical grip
    reads differently on each side, and nothing in the data shows it. **Do both or neither.**

### 4.1.2 How to calibrate

List the firmware SNs of the connected grippers:

```bash
python -c "from xense.taccap import scan_grippers, Side; \
  [print(f'{\"L\" if g.side==Side.Left else \"R\"} fw={g.firmware_sn} mcu={g.mcu_serial}') for g in scan_grippers()]"
```

Run this for **every** leader gripper (substitute the SN from above):

```bash
python third_party/taccap-gripper/python/examples/calibrate.py TCGU01A28Z0024m
```

One command, two steps, following the prompts:

1. **Hold the gripper fully closed** → Enter. Latched as the encoder zero, then re-read to verify
   the residual (tolerance ±0.01 rad).
2. **Open to the mechanical limit** → Enter. That angle is sampled and, once confirmed, written to
   MCU flash as the encoder max.

Output looks like:

```text
Step 1 — hold the gripper FULLY CLOSED
  post-zero reading: +0.0058 rad
Step 2 — open the gripper to its mechanical limit
  full-open reading: 1.1486 rad (65.8°)
Write 1.1486 rad (65.8°) to MCU flash? [y/N] y
✓ stored: max_rad = 1.1486 rad (65.8°)
```

!!! warning "Get the pose right before pressing Enter"
    The firmware latches the raw count at the instant it receives the command. The gripper must
    **already** be in the target pose (fully closed for step 1, against the mechanical limit for
    step 2) when you press Enter — moving it afterwards wastes the calibration.

!!! tip "Closed is always 0"
    The zero lives in firmware; there is **no `gripper_closed_rad` config**. `position_rad` keeps
    reporting raw radians — normalisation adds a `position` field rather than changing what was
    already there.

### 4.1.3 Confirm it took effect

**One: the connect log.** Each side prints a line when the collection program connects:

```text
[left]  Jaw normalised by the firmware's encoder-max calibration    ← in effect
[left]  Firmware encoder-max calibration unavailable (...)          ← not calibrated, using fallback
```

**Two: the curve in Rerun.** Run with `--display_data=true` and find `gripper.pos` in the scalar
panel:

| Action | Expected |
| --- | --- |
| Fully open | reaches **1.0** |
| Fully closed | drops to **0.0** |

Topping out below 1.0 (0.68, say) means it was never calibrated — matching the second log line.

### 4.1.4 Scope

- **Leaders only.** `Cmd::EncoderMaxCal` is leader-only; a follower NACKs `InvalidCmd` (it has no
  MT6816 encoder). Data collected with followers uses the `gripper_open_rad` fallback on both
  sides, which is unchanged from the older behaviour.
- **Firmware >= V2.1 (leader 1.2.0) required.** Older firmware does not have the command:
  `calibrate.py` **exits without changing anything**, and during collection the software warns and
  falls back automatically — the session continues, but the calibration has no effect. Since 0.1.7
  the SDK ships the released firmware images, so you can flash it yourself:
  → [Firmware OTA upgrade](versions.md#ota). **Update the SDK before the firmware** — the other
  order runs into an old bug where a failed update reported success.
- **Calibration is one-off.** The values live in MCU flash: they survive power cycles and moving
  to another host. Only redo it after removing or refitting the encoder, changing the mechanical
  limit, or erasing the firmware.

!!! note "The firmware's power-on auto-calibration does not replace this"
    V1.9 added `GripperAutoCalConfig` (close-to-stall ⇒ zero, open-to-stall ⇒ max_open at
    power-up), but that interface is on the **follower**, not the leader. Collection uses leaders,
    so manual calibration is still required.

## 4.2 Pico4 Ultra Enterprise tracker self-check

```bash
python -m lerobot.robots.taccap_gripper.calibrate_tracker
# Pin a specific tracker SN (format in 3.3, e.g. PC2310MLL3200496G):
python -m lerobot.robots.taccap_gripper.calibrate_tracker <tracker SN>
# Apply that side's built-in tracker->TCP mount transform:
python -m lerobot.robots.taccap_gripper.calibrate_tracker --side right
```

Prints `raw` (the tracker's own pose) and `ee` (the TCP after the rigid mount transform) at 10 Hz.
Wave the gripper: `raw xyz` should move smoothly and the SN should match what you expect.

!!! note "The mount transform is built in — you do not measure it"
    The tracker is bolted to the gripper, so what it reports is the **tracker's** pose, not the TCP
    we want to record. The rigid offset between them is built into
    `ee_transform.tracker_to_tcp` (measured off the CAD assembly), **with each side measured
    separately** — the two are close to mirror images but not exactly (0.03° apart in rotation,
    1.27 mm in translation), so the left value is not the right one mirrored.

    `--side` picks which built-in value to apply; **without `--side` the transform is identity**
    and `ee` simply follows `raw`. Override only if you need to (a re-machined mount, say) via
    `--robot.tracker_to_ee_pos` / `--robot.tracker_to_ee_quat`; the two are independent, so you can
    pin just the translation and keep the built-in rotation.

!!! tip "Pivot check: verify the mount transform with no extra hardware"
    Rest the gripper's **two-finger midpoint** on a fixed point and, holding the handle, sweep
    through as many orientations as it allows. **`ee xyz` should stay put while `raw xyz` swings
    widely** — that is the whole test, and whatever drift you see is the transform's error.
    **Test both sides**; a left value mirrored the wrong way shows up as `ee` moving about twice
    as far as it should.

If the quaternion flips hemisphere (sign jumps), the reader applies a continuity fix — **please
file a bug if you still see jumps**.

## 4.3 End-to-end smoke test

Use the supported `lerobot-teleoperate` entry point to verify the gripper, tactile and wrist camera
streams. This command only reads and previews devices — **it does not record a dataset**. To check
the gripper and cameras on their own, the commands below disable the Pico4 tracker and exit after
10 seconds.

**Single gripper (right side shown):**

```bash
lerobot-teleoperate \
    --robot.type=taccap_gripper \
    --robot.side=right \
    --robot.enable_tracker=false \
    --fps=30 \
    --teleop_time_s=10 \
    --debug_timing=true
```

**Bimanual:**

```bash
lerobot-teleoperate \
    --robot.type=bi_taccap_gripper \
    --robot.enable_tracker=false \
    --fps=30 \
    --teleop_time_s=10 \
    --debug_timing=true
```

Steady sample-timing output and camera counts, with no discovery, connection or read errors, means
the basic data flow is healthy.

## 4.4 3D trajectory visualisation (Rerun) {#44}

Run `lerobot-teleoperate` with `--display_data=true` to preview every observation plus the 3D
trajectory live.

**Single gripper (right side shown):**

```bash
lerobot-teleoperate \
    --robot.type=taccap_gripper \
    --robot.side=right \
    --fps=30 \
    --display_data=true \
    --show_trajectory=true
```

**Bimanual:**

```bash
lerobot-teleoperate \
    --robot.type=bi_taccap_gripper \
    --fps=30 \
    --display_data=true \
    --show_trajectory=true
```

The Rerun viewer gains a `/world` 3D view: the gripper is drawn as a labelled ellipsoid plus an
axis triad at its live **EEF TCP pose** (`tcp.*`), trailing a breadcrumb of where it has been.

- Our poses are already in the gravity-aligned world frame, and the scene declares `FLU`
  (X forward, Y left, Z up) — more complete than `RIGHT_HAND_Z_UP`, which only names the up axis,
  so the viewer knows which way is *forward* and aims its initial camera down +X. The world axes
  are labelled `+X forward` / `+Y left` / `+Z up`, so the orientation stays readable after you
  orbit.
- When the robot also publishes the tracker's own pose, Rerun draws **a smaller, dimmer tracker
  frame next to the EE frame** (display only, never recorded), joined by a dashed line labelled
  with its **length in mm**. That length is **the fastest way to tell whether the mount transform
  is right**: it should hold constant throughout a session. A jumping value, or one well off the
  mechanical dimension, means the transform or the assembly is wrong.
- `--show_trajectory` is on by default; set it to `false` to drop that view. It is skipped
  automatically when `--robot.enable_tracker=false`, since there is no pose to draw.

!!! note "How this differs from the standalone SDK example"
    The markers and breadcrumbs look much like the SDK's `python/examples/rerun_dual_with_tracker.py`.
    The LeRobot flow uses the gravity-aligned `FLU` world frame; the standalone SDK example shows
    the raw Pico4 `LEFT_HAND_Y_UP` frame.

Once calibration and the self-checks pass, you are ready to collect.

Next → [5. Data collection](05-data-collection.md)
