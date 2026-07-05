# 安装与构建

本页针对**单独使用 SDK**(不经数采主仓库)的场景。若你只是数采,按
[环境部署](02-environment.md)走 `setup_env.sh` 即可,SDK 会作为子模块一并装好。

## 前置条件

| 项 | 要求 |
|---|---|
| 操作系统 | Linux(Ubuntu 22.04+);采集路径 V4L2 + UVC XU,不支持 macOS / Windows |
| 工具链 | gcc/g++ ≥ 13、CMake ≥ 3.20、Ninja、pkg-config |
| Python(绑定) | CPython 3.12 |
| 推荐 | `mamba` / `conda`——`environment.yml` 固定整套工具链与 C++ 依赖 |

## 创建开发环境

```bash
mamba env create -f environment.yml
mamba activate taccap
```

这会装好 gcc-14、C++ 依赖、Python 3.12、pybind11、scikit-build-core、numpy、pyserial、
opencv-python==4.12.0.88、rerun-sdk。验证:

```bash
which cmake     # → .../envs/taccap/bin/cmake
gcc --version   # → 14.x
```

## 设备权限(一次性)

```bash
sudo usermod -aG dialout,video "$USER"
# 注销重登(或 newgrp dialout && newgrp video)后生效
```

## Python 安装(多数用户推荐)

`pyproject.toml` 用 scikit-build-core 驱动 CMake,一条 `pip` 同时构建 C++ 核与 pybind11 扩展:

```bash
# 开发/可编辑安装
pip install -e . --no-build-isolation
# 或常规安装
pip install .
```

验证:

```bash
python -c "import xense.taccap as t; print(t.hello()); print(t.__version__)"
```

## C++ 独立构建(无 Python)

集成进 ROS2 包或其他 CMake 工程时:

```bash
cmake -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DTACCAP_BUILD_PYTHON=OFF \
    -DTACCAP_BUILD_EXAMPLES=ON \
    -DTACCAP_BUILD_TESTS=ON
cmake --build build -j
```

| CMake 选项 | 默认 | 作用 |
|---|---|---|
| `TACCAP_BUILD_PYTHON` | `ON` | 构建 `_taccap_native` pybind11 模块 |
| `TACCAP_BUILD_EXAMPLES` | `OFF` | 构建 `leader_demo` 冒烟程序 |
| `TACCAP_BUILD_TESTS` | `OFF` | 构建 `cpp/tests/` 下 gtest 套件 |

C++ 测试:

```bash
ctest --test-dir build --output-on-failure
```

## 硬件自检

夹爪插好后:

```bash
python -c "from xense.taccap import scan_grippers, Side
for g in scan_grippers():
    s='L' if g.side==Side.Left else 'R'
    print(f'  [{s}] ch343={g.mcu_serial} fw_sn={g.firmware_sn!r}')"
```

健康输出:`[L]` 与 `[R]` 各一,`firmware_sn` 非空。为空说明 SN 未烧录或固件 < V1.6。
排障见 [3.1 串口权限](03-host-hardware.md#31) / [3.2 ModemManager](03-host-hardware.md#32)。
