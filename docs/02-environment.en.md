# 2. Environment Setup

Corresponds to the reference manual's "development environment setup". This chapter targets
Ubuntu 22.04 LTS / Ubuntu 24.04 LTS and gets `xense-taccap-lerobot` plus every hardware SDK
installed and verified.

!!! info "Verified environment for the XTac-UMI G1"
    XTac-UMI G1 hardware bring-up and collection were verified inside the `mamba` environment
    `xense-taccap`. The test host currently reads as:

    - OS: Ubuntu 24.04.4 LTS (Noble Numbat)
    - Linux kernel: `7.0.0-28-generic` (that host's `uname -r` output — **not** a minimum
      kernel requirement)
    - Architecture: `x86_64`
    - Python: `3.12.13`
    - Repo commit and per-package versions: see [Versions & Support](versions.md) (single source
      of truth, so nothing drifts between copies)

    Ubuntu 22.04 LTS is also covered by this chapter. Other distributions or architectures need
    verifying separately against their actual driver, UVC, serial-permission and `.deb` support.

    An NVIDIA GPU on the collection host is recommended: it lets `--dataset.vcodec=auto` pick the
    GPU H.264 hardware encoder, easing CPU load when several video streams are encoded live.

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
| `third_party/XenseVR-PC-Service` | `xensevr_pc_service_sdk` (Pico4 Ultra Enterprise teleop / tracker) |
| `third_party/XenseVR-RobotVision-PC` | ZED-M → Pico4 Ultra Enterprise stereo passthrough (built separately) |
| `third_party/pyinsight` | `pyinsight` (Insight head-camera interface) |

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
- install `pyinsight` (Insight head-camera interface)

!!! note "A pyinsight failure does not abort the install"
    If that step fails it only prints `[WARN] pyinsight installation skipped or failed`; nothing
    else is affected. If the submodule was never pulled, run
    `git submodule update --init third_party/pyinsight` and re-run. Check device/HID readiness
    separately with `pyinsight-check-env --hidraw`.

!!! note "Where the XenseVR PC Service .deb comes from"
    `./setup_env.sh --install` downloads the `.deb` for your architecture straight from the
    [v0.1.0 release](https://github.com/Vertax42/XenseVR-PC-Service/releases/tag/v0.1.0)
    (override the URL with `$XENSEVR_DEB_URL`) and runs `sudo dpkg -i`. It is skipped when the
    same version is already installed.

## 2.5 Verify the install {#25}

All three packages importing means success:

```bash
python -c 'import xensevr_pc_service_sdk; print("xensevr_pc_service_sdk OK ->", xensevr_pc_service_sdk.__file__)'
python -c 'import xensesdk; print("xensesdk OK ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("xense.taccap OK ->", xense.taccap.__file__)'
```

One more if you use the Insight head camera:

```bash
python -c 'import importlib.metadata as M; from pyinsight import find_library; print("pyinsight v" + M.version("pyinsight"), "->", find_library())'
```

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
