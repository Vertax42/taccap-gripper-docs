# 安装与构建

本页针对**单独使用 SDK**(不经数采主仓库)的场景。若你只是数采,按
[环境部署](02-environment.md)走 `setup_env.sh` 即可,SDK 会作为子模块一并装好。

!!! info "命令执行目录"
    本页命令均需在 `taccap-gripper` 源码根目录运行。随数采主仓库使用时先执行 `cd third_party/taccap-gripper`;独立使用时先克隆 SDK 仓库并进入该目录。

## 前置条件

| 项 | 要求 |
|---|---|
| 操作系统 | Linux(Ubuntu 22.04 / 24.04 已验证);相机路径为 V4L2/UVC,不支持 macOS / Windows |
| 工具链 | gcc/g++ ≥ 13、CMake ≥ 3.20、Ninja、pkg-config |
| Python(绑定) | CPython ≥ 3.10;仓库 `environment.yml` 固定 3.12 作为推荐开发环境 |
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

仅构建 C++ 共享库、示例和测试时:

```bash
cmake -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DTACCAP_BUILD_PYTHON=OFF \
    -DTACCAP_BUILD_EXAMPLES=ON \
    -DTACCAP_BUILD_TESTS=ON
cmake --build build -j
```

### 集成到其他 CMake / ROS2 工程

当前 SDK **没有安装头文件或导出 CMake package config**,因此不能直接使用 `find_package(taccap-gripper)`。推荐把 SDK 作为源码子目录引入:

```cmake
add_subdirectory(path/to/taccap-gripper taccap-gripper-build)
target_link_libraries(my_target PRIVATE taccap_core)
```

`taccap_core` 会传递公共头文件目录以及 OpenCV、spdlog 依赖。仅复制 `libtaccap_core.so` 不足以完成 C++ 集成,还需要匹配的公共头文件和依赖,不推荐这样使用。

### 构建选项

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
python -c "from xense.taccap import scan_grippers
for g in scan_grippers():
    print(f'side={g.side.name} role={g.role.name} ch343={g.mcu_serial} fw_sn={g.firmware_sn!r}')"
```

每个已连接夹爪应输出一行;连接单夹爪时只有一行是正常现象,不要求左右设备同时存在。新序列号规范下
`firmware_sn` 应非空且 `role` 应为 `Leader` / `Follower`;旧序列号、未烧录 SN、冷启动读取失败或旧固件可能显示
空 SN / `Unknown`,不能简单判定为“固件 < V1.6”。排障见 [3.1 串口权限](03-host-hardware.md#31) /
[3.2 ModemManager](03-host-hardware.md#32)。
