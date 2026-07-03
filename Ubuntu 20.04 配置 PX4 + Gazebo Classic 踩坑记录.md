# Ubuntu 20.04 配置 PX4 + Gazebo Classic 踩坑记录

## 1. 环境选择

本次环境为：

```bash
Ubuntu 20.04
Gazebo Classic 11
PX4-Autopilot
```

Ubuntu 20.04 不适合安装新版 Gazebo Harmonic，因此不要使用：

```bash
sudo apt install gz-harmonic
make px4_sitl gz_x500
```

在 Ubuntu 20.04 上应使用 Gazebo Classic 11：

```bash
sudo apt update
sudo apt install -y gazebo11 libgazebo11-dev
```

检查 Gazebo Classic 是否安装成功：

```bash
gazebo --version
```

如果显示类似下面内容，说明 Gazebo Classic 安装成功：

```bash
Gazebo multi-robot simulator, version 11.x.x
```

## 2. 为什么需要回退 PX4 版本

新版 PX4，例如 `main` 分支或 `v1.18.0-alpha`，更偏向新版 Gazebo / GZ 仿真生态，常用命令类似：

```bash
make px4_sitl gz_x500
```

但 Ubuntu 20.04 无法正常安装 `gz-harmonic`，因此这条路线不适合 Ubuntu 20.04。

如果在新版 PX4 中执行：

```bash
make px4_sitl gazebo
```

可能会出现：

```bash
ninja: error: unknown target 'gazebo', did you mean 'geo'?
```

这说明当前 PX4 版本已经不再提供或不适配旧的 Gazebo Classic `gazebo` target。

因此，在 Ubuntu 20.04 上使用 Gazebo Classic，推荐将 PX4 回退到较稳定的旧版本，例如：

```bash
PX4 v1.14.3
```

推荐组合：

```text
Ubuntu 20.04 + PX4 v1.14.3 + Gazebo Classic 11
```

## 3. 回退 PX4 到 v1.14.3

进入 PX4 源码目录：

```bash
cd ~/PX4-Autopilot
```

查看当前版本：

```bash
git describe --tags --always
```

如果显示类似：

```bash
v1.18.0-alpha1-xxx
```

说明当前是较新的开发版本，不适合继续在 Ubuntu 20.04 上折腾 Gazebo Classic。

获取所有 tag：

```bash
git fetch --tags
```

切换到 PX4 v1.14.3：

```bash
git checkout v1.14.3
```

切换后重新确认版本：

```bash
git describe --tags --always
```

应显示：

```bash
v1.14.3
```

## 4. 重新同步 PX4 子模块

切换 PX4 版本后，必须重新同步子模块，否则容易出现源码缺失、版本不一致等问题。

执行：

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

如果网络不稳定，子模块可能拉取失败。可以先测试网络：

```bash
curl -I https://github.com
curl -I https://gitlab.com
```

如果 WSL 无法访问 GitHub 或 GitLab，需要先确保 Windows 代理、VPN 或 TUN 模式已经作用到 WSL。

## 5. 子模块缺失问题

如果编译时报错：

```bash
Cannot find source file:
  heatshrink/heatshrink_decoder.c

Cannot find source file:
  devices/src/crc.cpp
```

说明 PX4 子模块没有完整拉取。

可以单独强制修复这两个子模块：

```bash
git submodule deinit -f src/lib/heatshrink/heatshrink
git submodule deinit -f src/drivers/gps/devices

rm -rf src/lib/heatshrink/heatshrink
rm -rf src/drivers/gps/devices

git submodule update --init --recursive --force --progress src/lib/heatshrink/heatshrink
git submodule update --init --recursive --force --progress src/drivers/gps/devices
```

检查关键文件是否存在：

```bash
ls src/lib/heatshrink/heatshrink/heatshrink_decoder.c
ls src/drivers/gps/devices/src/crc.cpp
```

如果能正常显示文件路径，说明子模块修复成功。

## 6. 安装 Gazebo Classic 及依赖

安装 Gazebo Classic 11：

```bash
sudo apt update
sudo apt install -y gazebo11 libgazebo11-dev
```

如果 `sitl_gazebo-classic` 配置失败，可以补充常见依赖：

```bash
sudo apt install -y \
  build-essential cmake ninja-build pkg-config \
  libprotobuf-dev protobuf-compiler libprotoc-dev \
  libeigen3-dev libxml2-utils \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
```

完整依赖安装命令可以写成：

```bash
sudo apt update

sudo apt install -y \
  build-essential cmake ninja-build pkg-config \
  gazebo11 libgazebo11-dev \
  libprotobuf-dev protobuf-compiler libprotoc-dev \
  libeigen3-dev libxml2-utils \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
```

## 7. 清理旧构建目录

切换 PX4 版本或修复依赖后，建议清理旧构建目录：

```bash
cd ~/PX4-Autopilot
rm -rf build
```

或者只清理 SITL 构建目录：

```bash
rm -rf build/px4_sitl_default
```

如果之前 Gazebo Classic 子工程配置失败，也可以单独清理：

```bash
rm -rf build/px4_sitl_default/build_gazebo-classic
rm -rf build/px4_sitl_default/external/Stamp/sitl_gazebo-classic
```

## 8. 编译并启动 Gazebo Classic 仿真

在 PX4 v1.14.3 下执行：

```bash
make px4_sitl gazebo
```

如果成功，会同时启动：

```text
PX4 SITL
Gazebo Classic
```

终端中会进入 PX4 shell，通常能看到：

```bash
pxh>
```

此时说明 PX4 SITL 已经运行成功。

## 9. 常见问题总结

### 9.1 Ubuntu 20.04 找不到 gz-harmonic

错误：

```bash
E: Unable to locate package gz-harmonic
```

原因：

```text
Ubuntu 20.04 不适合安装 Gazebo Harmonic。
```

解决：

```text
使用 Gazebo Classic 11，不使用 gz-harmonic。
```

### 9.2 make px4_sitl gazebo 报 unknown target

错误：

```bash
ninja: error: unknown target 'gazebo', did you mean 'geo'?
```

原因：

```text
当前 PX4 版本过新，例如 v1.18.0-alpha，已经不适合 Ubuntu 20.04 + Gazebo Classic 路线。
```

解决：

```bash
cd ~/PX4-Autopilot

git fetch --tags
git checkout v1.14.3
git submodule sync --recursive
git submodule update --init --recursive

rm -rf build
make px4_sitl gazebo
```

### 9.3 子模块文件缺失

错误：

```bash
Cannot find source file:
  heatshrink/heatshrink_decoder.c

Cannot find source file:
  devices/src/crc.cpp
```

原因：

```text
PX4 子模块没有完整拉取，或者切换版本后子模块没有重新同步。
```

解决：

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

必要时单独强制重拉缺失子模块。

### 9.4 gitlab.com 无法访问

错误：

```bash
Could not resolve host: gitlab.com
```

原因：

```text
WSL 没有正确使用 Windows 的代理或 VPN，导致 GitLab 子模块无法拉取。
```

解决：

```bash
curl -I https://github.com
curl -I https://gitlab.com
```

确认 WSL 可以访问 GitHub 和 GitLab 后，再重新执行：

```bash
git submodule update --init --recursive
```

### 9.5 sitl_gazebo-classic configure 失败

如果出现：

```bash
FAILED: external/Stamp/sitl_gazebo-classic/sitl_gazebo-classic-configure
```

可以查看错误日志：

```bash
cat build/px4_sitl_default/build_gazebo-classic/CMakeFiles/CMakeError.log
```

也可以单独重新配置 Gazebo Classic 子工程，查看更直接的错误：

```bash
rm -rf build/px4_sitl_default/build_gazebo-classic
mkdir -p build/px4_sitl_default/build_gazebo-classic

cd build/px4_sitl_default/build_gazebo-classic

cmake \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DSEND_ODOMETRY_DATA=ON \
  -DGENERATE_ROS_MODELS=ON \
  -GNinja \
  ~/PX4-Autopilot/Tools/simulation/gazebo-classic/sitl_gazebo-classic
```

如果提示缺少依赖，补充安装：

```bash
sudo apt install -y \
  build-essential cmake ninja-build pkg-config \
  gazebo11 libgazebo11-dev \
  libprotobuf-dev protobuf-compiler libprotoc-dev \
  libeigen3-dev libxml2-utils \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
```

## 10. 最终推荐流程

如果从一个较新的 PX4 版本回退到 Ubuntu 20.04 可用的 Gazebo Classic 环境，可以直接按下面流程执行：

```bash
cd ~/PX4-Autopilot

# 1. 回退 PX4 版本
git fetch --tags
git checkout v1.14.3

# 2. 重新同步子模块
git submodule sync --recursive
git submodule update --init --recursive

# 3. 安装 Gazebo Classic 及依赖
sudo apt update

sudo apt install -y \
  build-essential cmake ninja-build pkg-config \
  gazebo11 libgazebo11-dev \
  libprotobuf-dev protobuf-compiler libprotoc-dev \
  libeigen3-dev libxml2-utils \
  libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev

# 4. 清理旧构建
rm -rf build

# 5. 编译并启动仿真
make px4_sitl gazebo
```

## 11. 版本路线总结

```text
Ubuntu 20.04 + PX4 v1.14.3 + Gazebo Classic 11：
推荐，适合旧版 PX4 仿真环境。

Ubuntu 20.04 + PX4 新版 + Gazebo Harmonic：
不推荐，Ubuntu 20.04 不适合安装 gz-harmonic。

Ubuntu 22.04 / 24.04 + PX4 新版 + Gazebo Harmonic：
推荐，适合新版 PX4 仿真环境。
```