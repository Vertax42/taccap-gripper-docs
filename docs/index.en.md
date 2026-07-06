---
hide:
  - navigation
  - toc
---

<div class="tc-hero" markdown>

<span class="tc-eyebrow">XenseRobotics · TacCap-Gripper</span>

# Handheld tactile data collection, from unboxing to a dataset

<p class="tc-sub">TacCap-Gripper handheld tactile gripper × Pico4 Ultra tracker<br>capture synchronized vision · tactile · pose data with lerobot, straight to a training-ready <code>LeRobotDataset</code></p>

[Quickstart :material-arrow-right-bold:](quickstart.md){ .md-button .md-button--primary }
[Installation](02-environment.md){ .md-button }
[About the device](01-overview.md){ .md-button }

![TacCap-Gripper product](assets/product/xtac-umi-g1-hero.jpg){ .tc-hero-img }

</div>

## The whole flow in 5 minutes

```mermaid
flowchart LR
    A[Environment<br/>setup_env.sh] --> B[Host/Hardware<br/>serial perms · discovery]
    B --> C[Calibration<br/>encoder zero · tracker]
    C --> D[Data collection<br/>lerobot-record]
    D --> E[Dataset<br/>check · replay · push to Hub]
```

## Three steps

This is the **xense-taccap-lerobot data-collection quickstart**. Three parts: **set up the environment → record with the software → understand the data**.

<div class="grid cards" markdown>

-   :material-download-box-outline: __① Installation__

    ---

    Mamba env, submodules, one-shot `setup_env.sh`, host & device config.

    [:octicons-arrow-right-24: Environment Setup](02-environment.md)

-   :material-record-circle-outline: __② Software Usage__

    ---

    Calibration → `lerobot-record`. The core data-collection workflow.

    [:octicons-arrow-right-24: Data Collection](05-data-collection.md)

-   :material-database-outline: __③ Data__

    ---

    What a `LeRobotDataset` looks like, what's recorded per frame, checking & upload.

    [:octicons-arrow-right-24: Dataset & Examples](06-dataset.md)

-   :material-chip: __Device & Hardware__

    ---

    Get to know the handheld gripper (hardware is just one chapter).

    [:octicons-arrow-right-24: Overview](01-overview.md) · [Hardware](hardware.md)

</div>

## Related repositories

| Repo / package | Role |
|---|---|
| [`xense-taccap-lerobot`](https://github.com/Vertax42/xense-taccap-lerobot) | Data-collection repo (lerobot v5.1 fork, `taccap_gripper` robot class) |
| `xense.taccap` (`taccap-gripper` SDK) | Gripper device driver: IMU / encoder / wrist camera / protocol |
| `xensevr_pc_service_sdk` | Pico4 teleop / tracker PC service SDK |
| `xensesdk` | Visuotactile (OG) imaging & rectification (PyPI) |
