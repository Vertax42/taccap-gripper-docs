# 2. Environment Setup

Corresponds to the reference manual's "development environment setup". This chapter targets
Ubuntu 22.04 LTS / Ubuntu 24.04 LTS and gets `xense-taccap-lerobot` plus every hardware SDK
installed and verified.

!!! info "Verified environment for the XTac-UMI G1"
    XTac-UMI G1 hardware bring-up and collection were verified inside the `mamba` environment
    `xense-taccap`. The test host currently reads as:

    - OS: Ubuntu 22.04.5 LTS / Ubuntu 24.04.4 LTS
    - Linux kernel: **6.8, 6.14 and 7.0 series all verified** — the kernel version is not a
      constraint
    - Architecture: `x86_64`
    - Python: `3.12.13` (the main repository requires **≥ 3.12**)
    - Repo commit and per-package versions: see [Versions & Support](versions.md) (single source
      of truth, so nothing drifts between copies)

    Ubuntu 22.04 LTS is also covered by this chapter. Other distributions or architectures need
    verifying separately against their actual driver, UVC, serial-permission and `.deb` support.

    An NVIDIA GPU on the collection host is recommended: it lets `--dataset.vcodec=auto` pick the
    GPU H.264 hardware encoder, easing CPU load when several video streams are encoded live.
    **With an NVIDIA GPU the driver must be ≥ 570.144** (check with
    `nvidia-smi --query-gpu=driver_version --format=csv,noheader`).

!!! info "Overview"
    Four steps: install Mamba → clone the repo (with submodules) → create the environment →
    `setup_env.sh --install` → verify.

## 2.1 Prerequisite: install Mamba / Miniforge

Mamba is strongly recommended: dependency solving is **~10× faster** than stock conda, and faster
on every channel.

```bash
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

## 2.2 Clone the repo and its submodules

Hardware SDKs live in `third_party/` submodules, so the clone **must** be recursive:

```bash
git clone \
  --recurse-submodules \
  https://github.com/Vertax42/xense-taccap-lerobot.git
cd xense-taccap-lerobot
```

Already cloned without submodules:

```bash
git submodule update --init --recursive --progress
```

Submodules and the packages they install:

| Submodule | Package installed |
|---|---|
| `third_party/taccap-gripper` | `xense.taccap` (XTac-UMI G1 tactile gripper SDK) |
| `third_party/XenseVR-PC-Service` | `xensevr_pc_service_sdk` (Pico4 Ultra Enterprise tracker / headset camera) |

!!! note "The Insight head-camera path is gone"
    The head camera is now the Pico4 Ultra Enterprise headset's own stereo camera, sharing the
    trackers' connection — see [5.7 Headset camera](05-data-collection.md#57). The `pyinsight`
    and `XenseVR-RobotVision-PC` submodules have been removed from the main repository.

!!! note "xensesdk is not a submodule"
    `xensesdk` is the visuotactile sensor SDK. `setup_env.sh --install` installs it
    automatically — there is no submodule to pull.

!!! danger "Rebuild `xense.taccap` after updating the submodule"
    The `taccap-gripper` Python package ships a **compiled native extension**
    (`_taccap_native*.so`). `git submodule update` swaps the source but does not rebuild it — with
    new sources against a stale `.so`, `import xense.taccap` fails outright, e.g.:

    ```text
    AttributeError: module 'xense.taccap._taccap_native' has no attribute 'GripperAutoCalConfig'
    ```

    After pulling any submodule update touching `cpp/` or `python/bindings/`, rebuild:

    ```bash
    cd ~/xense-taccap-lerobot
    LIBRARY_PATH="${CONDA_PREFIX}/lib" \
      uv pip install -e third_party/taccap-gripper --no-deps --no-build-isolation
    ```

    Or just run `bash setup_env.sh --install` (which handles the other SDKs too). **No sudo
    needed.** Verify afterwards:

    ```bash
    python -c "import xense.taccap as t; print(t.__version__)"
    ```

## 2.3 Create and activate the environment

```bash
./setup_env.sh --mamba
mamba activate xense-taccap
```

!!! tip "Environment name"
    `--mamba` creates `xense-taccap` by default; append a name after `--mamba` to use your own.

## 2.4 One-shot install

```bash
./setup_env.sh --install
```

This step will:

- update the conda/mamba environment from `conda_environment.yaml`
- install the main package from `pyproject.toml`
- install the `xensesdk` visuotactile sensor SDK
- install the **XenseVR PC Service daemon** (a ~100 MB `.deb`, into `/opt/apps/roboticsservice`)
- build the SDKs under `third_party`: `xensevr_pc_service_sdk` (Pico4 Ultra Enterprise) and
  `xense.taccap` (gripper)

!!! note "Where the XenseVR PC Service .deb comes from"
    `./setup_env.sh --install` downloads the `.deb` for your architecture straight from the
    [v0.2.0 release](https://github.com/Vertax42/XenseVR-PC-Service/releases/tag/v0.2.0)
    (override the URL with `$XENSEVR_DEB_URL`) and runs `sudo dpkg -i`. It is skipped when the
    same version is already installed.

!!! warning "v0.2.0 is what makes the headset camera work"
    v0.2.0 **forwards message type `0x30`** (video frame with timestamp) to SDK clients rather
    than dropping it — that is the path the [headset camera](05-data-collection.md#57) frames
    take. Tracking behaviour is identical to v0.1.0 (`0x30` was unused there, and an older
    headset APK works unchanged against the new service), so **if you do not use the head camera,
    this upgrade changes nothing for you**.

    **arm64 hosts stay on v0.1.0.** v0.2.0 ships an amd64 asset only; `setup_env.sh` detects
    arm64, pins to the newest release that has an arm64 build and says so — at the cost of no
    head camera there. Build it yourself on an ARM64 host with the repository's
    `RoboticsService/qt-gcc_aarch64.sh` if you need it.

## 2.5 Verify the install {#25}

All three packages importing means success:

```bash
python -c 'import xensevr_pc_service_sdk; print("xensevr_pc_service_sdk OK ->", xensevr_pc_service_sdk.__file__)'
python -c 'import xensesdk; print("xensesdk OK ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("xense.taccap OK ->", xense.taccap.__file__)'
```

One more if you use the [headset camera](05-data-collection.md#57) — it checks that the pybind
layer in your environment is the build that carries the camera API:

```bash
python -c 'import xensevr_pc_service_sdk as xrt; print("pico camera API:", hasattr(xrt, "has_pico_camera_frame"))'
```

`False` means an older pybind is being loaded; re-run `./setup_env.sh --install`.

Optional — confirm the video codec dependencies load (`torchcodec` is pinned by the PyTorch
compatibility matrix, PyAV is pinned to `15.1.0`; FFmpeg is not part of the conda solve):

```bash
python -c 'import torchcodec; print("torchcodec OK ->", torchcodec.__version__)'
python -c 'import av; print("PyAV OK ->", av.__version__)'
```

!!! tip "Need a system ffmpeg?"
    If you need a system ffmpeg with `libsvtav1`, install it separately (apt, or an upstream
    static build); the default encoding path on this 0.5.1 fork does not depend on it.

---

With the environment installed, **do the host and hardware configuration next** (serial
permissions, device discovery) — otherwise grippers get listed but cannot be opened.

Next → [3. Host & Device Setup](03-host-hardware.md)
