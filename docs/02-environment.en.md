# 2. Environment Setup

Corresponds to the reference manual's "development environment setup". This chapter targets
**Ubuntu 22.04 / 24.04 LTS (amd64)** and gets `xense-taccap-lerobot` plus every hardware SDK
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

## Choosing an install path {#choose}

There are two paths. They produce the same collection environment — pick one and follow it
through.

| | **Mamba install from source** | **Docker delivery image** |
|---|---|---|
| What you get | The source repo; you build the environment | A packaged delivery directory and one script to run |
| Time | Longer — three hardware SDKs are compiled | Ten-odd minutes, mostly importing the image |
| NVIDIA GPU | Optional (collection works without one) | **Required**, driver ≥ 570.144 |
| Isolation | Installed into a Mamba env on the host | Lives in a container, host stays clean |
| Editing the code | Easy | Awkward |
| Steps | 2.1 – 2.5 below | [Docker delivery image](#docker) |

**Default to Mamba** — it works **with or without an NVIDIA GPU**, and it is the one to pick if
you will be editing the collection program. The commands in the rest of this manual are written
for this path.

Docker is there for a machine that already has the **NVIDIA driver**, when you would rather not
build an environment yourself.

!!! warning "Both paths need internet access"
    Installing pulls things down — conda fetches conda-forge and PyPI packages, clones the GitHub
    repo and its submodules, and downloads the XenseVR PC Service `.deb`; Docker installs Docker
    Engine and the NVIDIA Container Toolkit. **Neither path works on an offline machine.**

    On a restricted network, set the terminal's proxy up first (the Docker script accepts
    `XENSE_PROXY_URL` — see [One-shot install](#docker)).

Sections 2.1 – 2.5 below are the **Mamba install-from-source** path.

!!! info "On Docker? Skip 2.1 – 2.5 entirely"
    The image already has the Mamba environment, the collection program and all three hardware
    SDKs — nothing to clone, no environment to create, no `setup_env.sh` to run. Go straight to
    [Docker delivery image](#docker) below.

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
| [`third_party/taccap-gripper`](https://github.com/Vertax42/TacCap-Gripper) | `xense.taccap` (XTac-UMI G1 tactile gripper SDK) |
| [`third_party/XenseVR-PC-Service`](https://github.com/Vertax42/XenseVR-PC-Service) | `xensevr_pc_service_sdk` (Pico4 Ultra Enterprise tracker / headset camera) |

!!! note "xensesdk is not a submodule"
    `xensesdk` is the visuotactile sensor SDK. `setup_env.sh --install` installs it
    automatically — there is no submodule to pull.

!!! danger "Rebuild `xense.taccap` after updating the submodule"
    The `taccap-gripper` Python package ships a **compiled build artefact**. `git submodule
    update` only updates the files, it does not rebuild — with updated files against a stale
    build, `import xense.taccap` fails outright, e.g.:

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

- update the mamba/conda environment from `conda_environment.yaml`
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

!!! warning "Headset stereo and head pose need PC Service >= v0.2.0"
    Only v0.2.0 and later forward the headset camera's frames, and both the
    [stereo view and the head pose](05-data-collection.md#57) come down that path. Tracking is
    unaffected — if you do not use the headset camera, the version makes no difference.

## 2.5 Verify the install {#25}

All three packages importing means success:

```bash
python -c 'import xensevr_pc_service_sdk; print("xensevr_pc_service_sdk OK ->", xensevr_pc_service_sdk.__file__)'
python -c 'import xensesdk; print("xensesdk OK ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("xense.taccap OK ->", xense.taccap.__file__)'
```

One more if you use the [headset camera](05-data-collection.md#57) — it checks that what your
environment loads is the build that carries the camera API:

```bash
python -c 'import xensevr_pc_service_sdk as xrt; print("pico camera API:", hasattr(xrt, "has_pico_camera_frame"))'
```

`False` means an older version is being loaded; re-run `./setup_env.sh --install`.

Optional — confirm the video codec dependencies load (`torchcodec` is pinned by the PyTorch
compatibility matrix, PyAV is pinned to `15.1.0`; FFmpeg is not part of the conda solve):

```bash
python -c 'import torchcodec; print("torchcodec OK ->", torchcodec.__version__)'
python -c 'import av; print("PyAV OK ->", av.__version__)'
```

!!! tip "Need a system ffmpeg?"
    If you need a system ffmpeg with `libsvtav1`, install it separately (apt, or an official
    static build); the default encoding path on this 0.5.1 fork does not depend on it.

---

## Docker delivery image {#docker}

The image already contains the full `xense-taccap` environment, the CUDA user-space libraries, the
collection program and all three hardware SDKs (XenseSDK, TacCap-Gripper, the Pico4 bindings). The
container also starts the XenseVR PC Service on launch.

!!! warning "The container runs privileged"
    So that tactile sensors, the wrist camera, the gripper serial port and the Pico4 can be
    hot-plugged mid-session, the container runs privileged and shares the host network.
    **Only run it on a collection host you trust.**

### Host requirements

- Ubuntu 22.04 / 24.04, **amd64**
- **NVIDIA driver ≥ 570.144** — check with
  `nvidia-smi --query-gpu=driver_version --format=csv,noheader`

The installer **will not install or upgrade the GPU driver** for you (that depends on the card,
Secure Boot and a reboot); it stops and tells you if the driver is missing or too old. Docker
Engine, the Compose plugin and the NVIDIA Container Toolkit are installed automatically if absent.

### One-shot install

Copy the whole delivery directory to the collection host and run it as a **normal user**, not
root:

```bash
cd xense-taccap-lerobot-<version>-linux-amd64
./install_customer.sh
```

The script, in order: checks the system and GPU driver → installs Docker and the NVIDIA Container
Toolkit as needed → installs the gripper's serial udev rule (the ModemManager rule from
[3.2](03-host-hardware.md#32)) → verifies the image SHA256 and imports it → runs a PyTorch CUDA
smoke test.

!!! tip "Behind a proxy"
    ```bash
    XENSE_PROXY_URL=http://127.0.0.1:7897 ./install_customer.sh
    ```

### Host setup after installing

Give the current user Docker access right away:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
newgrp docker        # or log out and back in
docker images
```

!!! warning "`docker` group membership is close to root"
    Add only the users who need to collect — not every local account.

To show Rerun windows from inside the container, the host's current **desktop user** also has to
authorise it once:

```bash
xhost +si:localuser:root
# revoke when you are done
xhost -si:localuser:root
```

### Entering the container and verifying

```bash
docker compose run --rm xense-taccap
```

Inside the container, all four imports must pass and the GPU must be visible:

```bash
python -c 'import torch; print(torch.__version__, torch.cuda.is_available())'
python -c 'import xensesdk; print("xensesdk ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("taccap ->", xense.taccap.__file__)'
python -c 'import xensevr_pc_service_sdk; print("pico4 ->", xensevr_pc_service_sdk.__file__)'
```

Then confirm the devices are discovered:

```bash
lerobot-find-cameras
lerobot-info
```

You can also run a single command without an interactive shell:

```bash
docker compose run --rm xense-taccap lerobot-info
```

!!! note "Data survives the container"
    The dataset root inside the container is `/data/lerobot`. It lives in a Docker volume, as do
    the XenseSDK sensor-config cache and the Hugging Face and Torch caches, so `--rm` removing the
    temporary container does not touch them. Inspect with
    `docker compose run --rm xense-taccap bash -lc 'ls -la /data'`.

!!! tip "Working on data only, with no Pico4 attached"
    The container starts the XenseVR PC Service by default. Turn it off when you do not need the
    tracker:

    ```bash
    START_XENSEVR_SERVICE=0 docker compose run --rm xense-taccap
    ```

---

With the environment installed, **do the host and hardware configuration next** (serial
permissions, device discovery) — otherwise grippers get listed but cannot be opened.

Next → [3. Host & Device Setup](03-host-hardware.md)
