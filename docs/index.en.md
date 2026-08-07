---
hide:
  - navigation
  - toc
---

<div class="tc-hero" markdown>

<span class="tc-eyebrow">XenseRobotics · XTac-UMI G1</span>

# Handheld tactile data collection, from unboxing to a dataset

<p class="tc-sub">XTac-UMI G1 handheld tactile gripper × Pico4 Ultra Enterprise Edition headset and tracker<br>capture synchronized vision · tactile · first-person stereo · hand and head pose with lerobot, straight to a training-ready <code>LeRobotDataset</code></p>

[Quickstart :material-arrow-right-bold:](quickstart.md){ .md-button .md-button--primary }
[Installation](02-environment.md){ .md-button }
[About the device](01-overview.md){ .md-button }

![XTac-UMI G1 product](assets/product/xtac-umi-g1-hero.jpg){ .tc-hero-img }

</div>

!!! info "English coverage"
    The Home and Overview pages are available in English. Other navigation entries currently fall back to the Chinese source pages; command examples remain directly usable.

## The whole flow in 5 minutes

```mermaid
flowchart LR
    A[Environment<br/>setup_env.sh] --> B[Host/Hardware<br/>serial perms · discovery]
    B --> C[Calibration<br/>encoder zero · tracker]
    C --> P[Live preview<br/>lerobot-teleoperate]
    P --> D[Data collection<br/>lerobot-record]
    D --> E[Dataset<br/>check · replay · push to Hub]
```

## Three steps

This is the **xense-taccap-lerobot data-collection quickstart**. Three parts: **get ready → record → understand the data**.

<div class="grid cards" markdown>

-   :material-check-decagram-outline: __① Getting Ready (prerequisites)__

    ---

    Know your hardware → connect & power it on → set up the software environment and host/device config.

    [:octicons-arrow-right-24: Hardware](hardware.md) · [Environment Setup](02-environment.md)

-   :material-record-circle-outline: __② Software Usage__

    ---

    Calibration → preview the streams with `lerobot-teleoperate` → record with `lerobot-record`. The core data-collection workflow.

    [:octicons-arrow-right-24: Calibration](04-calibration.md) · [Data Collection](05-data-collection.md)

-   :material-database-outline: __③ Data__

    ---

    What a `LeRobotDataset` looks like, what's recorded per frame, checking & upload.

    [:octicons-arrow-right-24: Dataset & Examples](06-dataset.md)

</div>

## Related repositories

| Repo / package | Role |
|---|---|
| [`xense-taccap-lerobot`](https://github.com/Vertax42/xense-taccap-lerobot) | Data-collection repo (lerobot 0.5.1 customized branch, providing the `taccap_gripper` robot type) |
| [`xense.taccap`](https://github.com/Vertax42/TacCap-Gripper) | Gripper SDK (repo `TacCap-Gripper`, submodule `third_party/taccap-gripper`): IMU, encoder, keys, protocol, and follower-only motor control |
| [`xensevr_pc_service_sdk`](https://github.com/Vertax42/XenseVR-PC-Service) | Pico4 Ultra tracker PC service (submodule `third_party/XenseVR-PC-Service`); from v0.2.0 it also carries the [headset camera](05-data-collection.md#57) frames |
| [`xensesdk`](https://github.com/XenseRobotics/xensesdk) | Visuotactile sensor SDK, provided by the install script ([docs](https://xensedoc.readthedocs.io/en/latest/)) |
