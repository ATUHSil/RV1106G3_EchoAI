<!-- created: 2026-07-28 -->
<!-- updated: 2026-07-28 -->

# wsl + docker

> 本项目的开发环境基于 WSL2 Arch Linux + Docker，以下是我个人成功的搭建经历。

# Docker + WSL2 环境搭建指南

> Echo-Mate 项目开发环境配置，基于 WSL2 Arch Linux + Docker。

- 注意如果第一次用wsl2推荐用ubuntu而不是arch 

## 环境概述

- **宿主机**: Windows 10/11 + WSL2
- **开发环境**: WSL2 Arch Linux + Docker
- **容器镜像**: luckfoxtech/luckfox_pico:1.0（RV1106 交叉编译工具链）

## 前提条件

- Windows 10 版本 2004+ 或 Windows 11
- WSL2 已启用并安装了 Arch Linux(或其他linux系统)
- 能以管理员权限运行 PowerShell
- 基本的命令行操作能力

---

## 安装步骤

### 1. 安装 Docker Engine

在 WSL2 Arch Linux 中执行：

```bash
# 安装 Docker 相关组件
sudo pacman -S docker docker-compose docker-buildx

# 设置开机自启并启动服务
sudo systemctl enable docker
sudo systemctl start docker

# 将当前用户加入 docker 组，避免每次都要 sudo
sudo usermod -aG docker $USER
```

> 加入 docker 组后需要重新登录才会生效。可以先执行 `newgrp docker` 立即生效。

### 2. 配置 rshared 挂载

Docker 在 WSL2 中运行需要根文件系统以 shared 方式挂载。编辑 `/etc/wsl.conf`：

```ini
[user]
default = GreedArch

[boot]
systemd = true
command = "mount --make-rshared / && usermod -s /usr/bin/zsh root"

[automount]
options = "metadata,umask=022"
```

配置完成后，在 WSL2 中手动执行一次：

```bash
sudo mount --make-rshared /
```

> 这一步很关键。如果跳过，Docker 容器启动时会报 mount 相关错误。

### 3. 安装 X11 工具

用于 GUI 应用转发（容器内图形界面显示到 Windows 桌面）：

```bash
sudo pacman -S xorg-xeyes
```

`xeyes` 是个简单的 X11 测试程序，眼睛会跟随鼠标移动。用来验证 GUI 转发是否正常工作。

### 4. 安装 USB/IP 工具

用于将 Windows 上的 USB 设备（比如开发板）透传到 WSL2：

```bash
sudo pacman -S usbip usbutils
```

### 5. Windows 端安装 usbipd-win

在 Windows PowerShell（管理员模式）中执行：

```powershell
winget install usbipd
```

安装完成后重启 PowerShell。

---

## Docker 配置

### 1. 配置 docker-compose.yml

在 Echo-Mate 项目根目录创建 `.env` 文件：

```
SDK_PATH=../SDK/rv1106-sdk
```

修改 `docker-compose.yml` 第 13 行，使用环境变量引用 SDK 路径：

```yaml
- ${SDK_PATH}:/sdk
```

### 2. 拉取 Docker 镜像

```bash
docker pull luckfoxtech/luckfox_pico:1.0
```

镜像约 2-3GB，包含完整的 RV1106 交叉编译工具链。首次下载需要一些时间。

### 3. 镜像更新

基础镜像可能缺少一些常用工具。如果需要在容器中使用 `curl`，需要在 Dockerfile 中添加：

```dockerfile
# 在 RUN apt-get install 行中添加
curl \
```

然后重新构建镜像：

```bash
docker-compose build
```

---

## 验证步骤

### 验证 Docker 安装

```bash
docker --version
docker-compose --version
docker buildx version

# 运行测试容器
docker run --rm hello-world
```

看到 "Hello from Docker!" 就说明安装成功。

### 验证 GUI 转发

```bash
# 先测试 WSL2 本地的 X11
xeyes

# 再测试容器内的 X11 转发
docker run --rm -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix luckfoxtech/luckfox_pico:1.0 xeyes
```

如果弹出眼睛窗口，GUI 转发正常。

---

## 编译测试结果

以下是在本环境中的实际测试记录（2026-06-22）。

### SDK buildroot sysroot 编译

在交叉编译 Demo 之前，必须先在 Docker 容器内编译 SDK 的 buildroot sysroot，生成交叉编译所需的库文件。

**原因**: Luckfox SDK 的 buildroot `.config` 已启用 `opus`、`portaudio`、`jsoncpp` 等包，但 sysroot 中缺少编译产物。直接交叉编译 Demo 会报链接错误 `cannot find -lopus` / `-lportaudio` / `-ljsoncpp`。

**操作命令**:
```bash
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c "
  cd /project/SDK/rv1106-sdk/sysdrv/source/buildroot/buildroot-2023.02.6 &&
  make
"
```

**验证**: 编译完成后，确认 sysroot 中存在库文件：
```bash
ls /project/SDK/rv1106-sdk/sysdrv/source/buildroot/buildroot-2023.02.6/output/host/arm-buildroot-linux-uclibcgnueabihf/sysroot/usr/lib/ | grep -E 'opus|portaudio|jsoncpp'
```

**耗时**: 首次编译约 20-40 分钟（取决于机器性能），后续增量编译会快很多。

---

### SDK 编译测试

**板级配置**: Echo Mate（预设在 SDK 中）

**测试结果**:
- 编译成功启动，3 分钟快速测试通过
- 完整编译预计需要 30+ 分钟（首次编译较慢，后续增量编译会快很多）

**操作命令**:
```bash
# 启动编译（后台运行，避免超时）
nohup docker-compose run rv1106-dev bash -c "cd /project/SDK/rv1106-sdk && ./build.sh" > build.log 2>&1 &

# 监控编译进度
tail -f build.log
```

### Demo 编译测试

#### x86 模拟器编译

**测试结果**:
- cmake 配置成功
- 编译成功
- 链接成功

**已知限制**:
- x86 模拟器模式下 YOLO 页面自动排除（需要 ARM 专用库）
- Weather 页面需要 `json-c` 库，在 x86 模拟器下可选

**操作命令**:
```bash
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c \
  "cd /project/Demo/DeskBot_demo && rm -rf build && mkdir build && cd build && cmake .. && make"
```

#### ARM 交叉编译（成功）

**测试结果**:
- ✅ cmake 配置成功（`-DTARGET_ARM=ON`）
- ✅ 编译成功（包括 AIChatCore、yoloCameraCore）
- ✅ 链接成功，生成 ARM ELF 可执行文件 `bin/main`
- ✅ 资源文件正确复制到 `bin/`（model、lib、配置等）

**修复内容**:
1. `toolchain.cmake` — 加 `set(CMAKE_FIND_LIBRARY_SUFFIXES .a)` 强制静态链接
2. `gui_app/CMakeLists.txt` — 手动 `find_library` json-c，显式链接 openssl/zlib/atomic
3. 手动编译 ARM 静态库 `libz.a`、`libcurl.a`，清理 x86_64 污染库，修复 json-c CMake 配置
4. 一键脚本：`build_arm.sh`（位于 `Echo-Mate/Demo/DeskBot_demo/`）

**操作命令**:
```bash
# 一键编译（自动处理静态库 + 清理 + 交叉编译）
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c \
  "bash /project/Demo/DeskBot_demo/build_arm.sh"
```


**验证 ARM 构建**:
```bash
# 检查文件类型
file /project/Demo/DeskBot_demo/bin/main
# 应输出: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-uClibc.so.0

# 确认没有动态链接 libjson-c / libcurl（已静态打包）
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c \
  "arm-rockchip830-linux-uclibcgnueabihf-readelf -d /project/Demo/DeskBot_demo/bin/main | grep NEEDED"
```

---

## 使用 Docker 编译项目

### 编译 SDK

```bash
cd Echo-Mate

# 选择目标板型
docker-compose run rv1106-dev bash -c "cd /project/SDK/rv1106-sdk && ./build.sh lunch"

# 开始编译
docker-compose run rv1106-dev bash -c "cd /project/SDK/rv1106-sdk && ./build.sh"
```

### 编译 Demo

> **前置条件**: ARM 交叉编译前，必须先完成 [SDK buildroot sysroot 编译](#sdk-buildroot-sysroot-编译)。

```bash

# x86 模拟器编译（用于 GUI 测试）
docker-compose run rv1106-dev bash -c "cd /project/Demo/DeskBot_demo && rm -rf build && mkdir build && cd build && cmake .. && make"

# ARM 交叉编译（一键脚本，自动处理静态库 + 清理 + 编译）
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c \
  "bash /project/Demo/DeskBot_demo/build_arm.sh"
```

编译产物会通过 Docker volume 映射回宿主机文件系统。

---

## 常见问题

### ARM 交叉编译链接失败：cannot find -lopus / -lportaudio / -ljsoncpp

Luckfox SDK 的 buildroot `.config` 虽然启用了这些包，但 sysroot 中缺少编译产物。需要先编译 buildroot sysroot：

```bash
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c "
  cd /project/SDK/rv1106-sdk/sysdrv/source/buildroot/buildroot-2023.02.6 &&
  make
"
```

编译完成后重新尝试 Demo 交叉编译。

### ARM 交叉编译推荐使用 build_arm.sh 一键脚本

ARM 交叉编译涉及多个手动步骤（编译静态库、清理污染、修复 CMake 配置），现在已封装为 `Echo-Mate/Demo/DeskBot_demo/build_arm.sh`：

```bash
docker-compose -f Echo-Mate/docker-compose.yml run rv1106-dev bash -c \
  "bash /project/Demo/DeskBot_demo/build_arm.sh"
```

脚本会自动处理：libz/libcurl/libjson-c 静态库编译、x86_64 污染库清理、json-c CMake 配置修复、交叉编译、可选部署。

> 注意：buildroot sysroot 的静态库在 Docker 容器重建后会丢失，`build_arm.sh` 会在每次编译时自动重新生成。

### 编译器警告：parameter passing changed in GCC 7.1

这是 boost::asio 与 GCC 8.3 的兼容性提示（`note`），不是错误。可以忽略。

### Docker 容器启动失败

多半是 rshared 挂载没配置好。检查一下：

```bash
mount | grep "on / type"
# 如果输出中没有 "shared"，执行：
sudo mount --make-rshared /
```

### GUI 应用不显示

检查 `conf/dev_conf.h` 中的 `LV_USE_SIMULATOR` 配置：

```c
#define LV_USE_SIMULATOR 1   // Docker/x86 环境用 SDL2
#define LV_USE_SIMULATOR 0   // 开发板硬件 framebuffer
```

- x86/Docker 环境必须设为 `1`，使用 SDL2 后端
- 开发板环境设为 `0`，使用 framebuffer 设备

如果已设为 `1` 但仍不显示，检查 DISPLAY 环境变量：
```bash
echo $DISPLAY
# 应该输出类似 :0 或 :1
```

如果为空，说明 WSL2 的 X11 转发没配置好。需要在 Windows 端运行 X Server（比如 VcXsrv 或 WSLg）。

### USB 设备不显示

重新绑定设备：

```powershell
usbipd detach --busid=<BUSID>
usbipd attach --wsl --busid=<BUSID>
```

确认 WSL2 内核支持 USB/IP：

```bash
lsmod | grep vhci
```

### 权限问题

如果执行 docker 命令提示权限不足：

```bash
# 确认用户在 docker 组中
groups $USER

# 如果没有 docker 组，重新添加
sudo usermod -aG docker $USER
# 然后重新登录
```

### curl 缺失问题

基础镜像 `luckfoxtech/luckfox_pico:1.0` 默认不包含 `curl`。如果在容器中执行脚本时遇到 `curl: command not found`，需要：

1. 在 Dockerfile 中添加 `curl`：
   ```dockerfile
   RUN apt-get update && apt-get install -y \
       curl \
       # ... 其他包
   ```

2. 重新构建镜像：
   ```bash
   docker-compose build
   ```

### x86 模拟器 GUI 运行

在 x86 环境下运行 Demo GUI 需要启用 SDL2 模拟器模式：

**修复内容**:
1. `gui_app/CMakeLists.txt` - x86 模式下自动排除 YOLO 页面
2. `gui_app/ui.c` - x86 模式下不注册 YOLO 页面
3. `conf/dev_conf.h` - 设置 `LV_USE_SIMULATOR 1`，使用 SDL2 后端

**运行命令**:
```bash
# 编译（x86 模拟器模式）
docker-compose run rv1106-dev bash -c "cd /project/Demo/DeskBot_demo && rm -rf build && mkdir build && cd build && cmake .. && make"

# 运行 GUI
docker-compose run --rm rv1106-dev bash -c "cd /project/Demo/DeskBot_demo/bin && ./main"
```

**注意**: 
- x86 模式下 YOLO 页面不可用（需要 ARM 专用库）
- 如需完整功能，使用 `-DTARGET_ARM=ON` 交叉编译到 ARM 开发板

### AI Chat Server 运行

Server 端需要 Python 3.10 环境，在docker运行：

```bash
cd /project/Demo/AIChat_demo/Server

# 创建 venv
python -m venv venv && source venv/bin/activate

# 安装依赖（PyTorch 需单独安装 CPU 版本）
pip install torchaudio==2.3.0 --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple --timeout 120 --retries 10

# 运行（access_token 需与 system_para.conf 一致）
python ./main.py --access_token="123456"
```

**注意**:
- `funasr` 依赖 `torchaudio`，需安装 CPU 版本避免 `libcudart.so.13` 错误
- 网络不稳定时使用清华镜像源
- 防火墙需放通 8000 端口（Client 连接用）

---

## 重要修复：API Key 401 问题（2026-07-27）

### 现象

板子唤醒并识别语音后，LLM 调用返回 401：

```
websocket closed due to Handshake status 401 Unauthorized
{"code":"InvalidApiKey","message":"Invalid API-key provided."}
```

**官方通知**: https://www.aliyun.com/notice/detail?notice-id=118406

阿里云百炼在 2026-06-26 后更换了 API Key 格式，新 key 长度从约 32 字节增长到 84 字节。

### 根因

C 结构体 `aliyun_api_key` 缓冲区只有 36 字节，写入配置时新 key 被静默截断。客户端通过 hello 消息把截断后的 key 发给服务器，服务器无条件用这个无效 key 覆盖了自己配置中的有效 key，导致后续所有 LLM 请求返回 401。


---

## 参考链接

- [Docker 官方文档](https://docs.docker.com/engine/install/archlinux/)
- [usbipd-win 项目](https://github.com/dorssel/usbipd-win)
- [WSL2 官方文档](https://learn.microsoft.com/en-us/windows/wsl/)
- [Luckfox Pico SDK](https://github.com/LuckfoxTECH/luckfox-pico)


