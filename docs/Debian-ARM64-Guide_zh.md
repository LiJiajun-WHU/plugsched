# 在 ARM64 Debian 系统上使用 Plugsched 完整指南

本文档详细介绍如何在 ARM64 架构的 Debian 系统上使用 plugsched 进行内核调度器热升级，包括从下载到部署，再到实验测试的全过程。

## 目录
- [系统要求](#系统要求)
- [准备工作](#准备工作)
- [安装必要软件包](#安装必要软件包)
- [获取内核源代码](#获取内核源代码)
- [构建带调试符号的内核](#构建带调试符号的内核)
- [设置 Plugsched 容器环境](#设置-plugsched-容器环境)
- [提取和初始化调度器](#提取和初始化调度器)
- [修改调度器代码](#修改调度器代码)
- [构建调度器模块](#构建调度器模块)
- [安装和测试](#安装和测试)
- [卸载调度器模块](#卸载调度器模块)
- [故障排除](#故障排除)

## 系统要求

- **架构**: ARM64 (aarch64)
- **操作系统**: Debian 10/11/12 或基于 Debian 的发行版
- **内核版本**: 建议 4.19+ 或 5.x 系列
- **内存**: 建议至少 4GB RAM
- **磁盘空间**: 至少 20GB 可用空间（用于内核源码和构建产物）
- **权限**: root 或 sudo 权限

## 准备工作

### 1. 检查系统信息

```bash
# 检查架构
uname -m
# 应显示: aarch64

# 检查当前内核版本
uname -r
# 例如: 5.10.0-23-arm64

# 检查系统版本
cat /etc/os-release
```

### 2. 记录内核版本信息

```bash
export KERNEL_VERSION=$(uname -r)
echo "当前内核版本: $KERNEL_VERSION"
```

## 安装必要软件包

### 1. 更新软件包列表

```bash
sudo apt update
```

### 2. 安装基础开发工具

```bash
sudo apt install -y \
    build-essential \
    git \
    wget \
    curl \
    bc \
    bison \
    flex \
    libssl-dev \
    libelf-dev \
    rsync \
    kmod \
    cpio \
    libncurses-dev \
    pkg-config
```

### 3. 安装容器运行时

Debian 系统可以使用 Docker 或 Podman。这里推荐使用 Docker：

```bash
# 安装 Docker
sudo apt install -y docker.io

# 启动 Docker 服务
sudo systemctl start docker
sudo systemctl enable docker

# 将当前用户添加到 docker 组（可选，避免每次使用 sudo）
sudo usermod -aG docker $USER
# 注意：需要重新登录才能生效

# 如果选择使用 Podman（替代 Docker）
# sudo apt install -y podman
```

### 4. 安装 Python 开发依赖

```bash
sudo apt install -y \
    python3 \
    python3-pip \
    python3-dev \
    python3-yaml \
    libyaml-dev
```

### 5. 安装内核开发包和调试符号

```bash
# 安装当前运行内核的头文件
sudo apt install -y linux-headers-$(uname -r)

# 尝试安装调试符号包（如果可用）
# 注意：Debian 的调试符号包可能需要配置 dbg 源
sudo apt install -y linux-image-$(uname -r)-dbg 2>/dev/null || echo "调试符号包不可用，需要手动构建"
```

**重要提示**: Debian 系统通常不像 RHEL/Anolis 系统那样提供完整的 kernel-debuginfo 包。如果 `linux-image-*-dbg` 包不可用，您需要从源代码构建内核以生成 `vmlinux` 文件。

## 获取内核源代码

### 方法 1: 使用 apt-get source（推荐用于 Debian 官方内核）

```bash
# 创建工作目录
mkdir -p /tmp/work
cd /tmp/work

# 添加 deb-src 源（如果尚未启用）
sudo sed -i 's/^# deb-src/deb-src/' /etc/apt/sources.list
sudo apt update

# 下载内核源代码
apt-get source linux-image-$(uname -r)

# 源码将被提取到当前目录，例如: linux-5.10.xxx
```

### 方法 2: 从 Git 仓库克隆（用于特定内核版本）

```bash
cd /tmp/work

# 对于 Debian 官方内核
git clone --depth 1 --branch debian/$(uname -r | sed 's/-.*//') \
    https://salsa.debian.org/kernel-team/linux.git kernel

# 或者从 kernel.org 获取主线内核
# git clone --depth 1 --branch v5.10 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git kernel
```

### 方法 3: 下载内核源码包

```bash
cd /tmp/work

# 确定内核版本
KVER=$(uname -r | sed 's/-.*//') # 例如: 5.10.0

# 从 kernel.org 下载
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-${KVER}.tar.xz

# 解压
tar xf linux-${KVER}.tar.xz
mv linux-${KVER} kernel
```

## 构建带调试符号的内核

由于 Debian 系统可能没有提供 `vmlinux` 文件（带调试符号的内核），我们需要构建内核以生成它。

### 1. 准备内核配置

```bash
cd /tmp/work/kernel

# 复制当前运行内核的配置
cp /boot/config-$(uname -r) .config

# 或者使用默认配置
# make defconfig

# 启用必要的调试选项
scripts/config --enable CONFIG_DEBUG_INFO
scripts/config --enable CONFIG_DEBUG_INFO_DWARF4
scripts/config --disable CONFIG_DEBUG_INFO_REDUCED
scripts/config --enable CONFIG_KALLSYMS
scripts/config --enable CONFIG_KALLSYMS_ALL
```

### 2. 调整 Makefile 中的版本信息

确保构建的内核版本与当前运行的内核匹配：

```bash
# 获取完整的内核版本字符串
FULL_KVER=$(uname -r)

# 分离版本号和 EXTRAVERSION
BASE_VER=$(echo $FULL_KVER | cut -d'-' -f1)
EXTRA_VER=$(echo $FULL_KVER | cut -d'-' -f2-)

# 修改 Makefile
# 编辑 Makefile，设置 EXTRAVERSION
# 例如: EXTRAVERSION = -23-arm64
# 您可以使用 sed 或手动编辑
```

### 3. 构建内核

```bash
# 使用多核编译（根据您的 CPU 核心数调整 -j 参数）
make -j$(nproc)

# 构建完成后，vmlinux 文件将位于内核源码根目录
ls -lh vmlinux
```

**注意**: 完整的内核构建可能需要 1-2 小时，取决于您的硬件性能。

### 4. 设置调试符号路径

```bash
# 创建标准调试符号目录
sudo mkdir -p /usr/lib/debug/lib/modules/$(uname -r)/

# 复制 vmlinux 到标准位置
sudo cp vmlinux /usr/lib/debug/lib/modules/$(uname -r)/vmlinux

# 验证文件存在
ls -lh /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
```

### 5. 准备 Module.symvers 和 .config

```bash
# 创建内核开发文件目录
sudo mkdir -p /usr/src/kernels/$(uname -r)

# 复制必要文件
sudo cp Module.symvers /usr/src/kernels/$(uname -r)/
sudo cp .config /usr/src/kernels/$(uname -r)/
sudo cp Makefile /usr/src/kernels/$(uname -r)/

# 复制整个源码目录（可选，但推荐）
cd /tmp/work
sudo cp -r kernel /usr/src/kernels/$(uname -r)/source
```

## 设置 Plugsched 容器环境

### 1. 构建 Plugsched Docker 镜像（适用于 Debian/ARM64）

由于官方镜像基于 Anolis OS (RPM 系统)，我们需要创建一个适用于 Debian 的版本：

```bash
cd /tmp/work

# 克隆 plugsched 仓库
git clone https://github.com/LiJiajun-WHU/plugsched.git

cd plugsched

# 创建适用于 Debian 的 Dockerfile
cat > Dockerfile.debian << 'EOF'
# Dockerfile for Debian/ARM64
FROM debian:11

# 安装基础软件包
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    python3-dev \
    gcc \
    g++ \
    make \
    bison \
    flex \
    git \
    bc \
    libssl-dev \
    libelf-dev \
    libyaml-dev \
    rsync \
    rpm \
    perl \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
RUN pip3 install sh docopt pyyaml colorlog

# 复制 plugsched 文件
COPY . /usr/local/lib/plugsched/
RUN ln -s /usr/local/lib/plugsched/cli.py /usr/local/bin/plugsched-cli && \
    chmod +x /usr/local/bin/plugsched-cli

WORKDIR /root
EOF

# 构建镜像
docker build -f Dockerfile.debian -t plugsched-sdk:debian-arm64 .
```

### 2. 启动容器

```bash
# 返回工作目录
cd /tmp/work

# 启动容器，挂载必要的目录
docker run -itd \
    --name=plugsched \
    -v /tmp/work:/tmp/work \
    -v /usr/src/kernels:/usr/src/kernels \
    -v /usr/lib/debug/lib/modules:/usr/lib/debug/lib/modules \
    plugsched-sdk:debian-arm64

# 验证容器运行
docker ps | grep plugsched
```

### 3. 进入容器

```bash
docker exec -it plugsched bash
cd /tmp/work
```

## 提取和初始化调度器

在容器内执行以下操作：

### 1. 验证环境

```bash
# 检查 plugsched-cli 是否可用
plugsched-cli --help

# 检查必要文件是否存在
ls -la /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
ls -la /usr/src/kernels/$(uname -r)/Module.symvers
ls -la /usr/src/kernels/$(uname -r)/.config
```

### 2. 使用 dev_init 方式初始化（推荐用于 Debian）

由于 Debian 没有 kernel.srpm 包，我们使用 `dev_init` 方法：

```bash
cd /tmp/work

# 首先需要确保内核已构建（生成 vmlinux）
# 如果尚未构建，请先在主机上构建内核

# 使用 dev_init 初始化调度器
plugsched-cli dev_init /tmp/work/kernel ./scheduler
```

**注意**: `dev_init` 方法会自动在内核源码目录中查找 `vmlinux` 文件。

### 3. 或者使用 init 方式（如果 debuginfo 可用）

```bash
# 如果您已经有了 vmlinux 和相关文件在标准位置
plugsched-cli init $(uname -r) /tmp/work/kernel ./scheduler
```

### 4. 验证提取结果

```bash
# 检查生成的目录结构
ls -la ./scheduler/
ls -la ./scheduler/kernel/sched/mod/

# 应该看到调度器源文件，如:
# core.c, fair.c, rt.c, deadline.c, features.h 等
```

## 修改调度器代码

### 1. 示例修改：添加一个新的 sched_feature

为了测试 plugsched 是否工作，我们添加一个简单的功能：

```bash
cd /tmp/work/scheduler

# 编辑 features.h，添加新的 feature
cat >> kernel/sched/mod/features.h << 'EOF'

/* Plugsched test feature for Debian ARM64 */
SCHED_FEAT(DEBIAN_ARM64_TEST, false)
EOF

# 编辑 core.c，在 __schedule 函数中添加测试代码
# 找到 __schedule 函数，添加以下内容
```

在容器内使用编辑器（如 vi 或通过主机编辑）修改 `kernel/sched/mod/core.c`：

```c
// 在 __schedule 函数开始处添加：
if (sched_feat(DEBIAN_ARM64_TEST))
    printk_once("Plugsched: Debian ARM64 scheduler is working!\n");
```

### 2. 完整示例补丁

您可以创建一个补丁文件：

```bash
cat > /tmp/work/test-debian-arm64.patch << 'EOF'
diff --git a/scheduler/kernel/sched/mod/core.c b/scheduler/kernel/sched/mod/core.c
index xxxxx..yyyyy 100644
--- a/scheduler/kernel/sched/mod/core.c
+++ b/scheduler/kernel/sched/mod/core.c
@@ -3234,6 +3234,9 @@ static void __sched notrace __schedule(bool preempt)
 	struct rq *rq;
 	int cpu;
 
+	if (sched_feat(DEBIAN_ARM64_TEST))
+		printk_once("Plugsched: Debian ARM64 scheduler is working!\n");
+
 	cpu = smp_processor_id();
 	rq = cpu_rq(cpu);
 	prev = rq->curr;
diff --git a/scheduler/kernel/sched/mod/features.h b/scheduler/kernel/sched/mod/features.h
index xxxxx..yyyyy 100644
--- a/scheduler/kernel/sched/mod/features.h
+++ b/scheduler/kernel/sched/mod/features.h
@@ -1,4 +1,7 @@
 /* SPDX-License-Identifier: GPL-2.0 */
+/* Plugsched test feature for Debian ARM64 */
+SCHED_FEAT(DEBIAN_ARM64_TEST, false)
+
 /*
  * Only give sleepers 50% of their service deficit. This allows
  * them to run sooner, but does not allow tons of sleepers to
EOF

# 应用补丁
cd /tmp/work/scheduler
patch -p1 < /tmp/work/test-debian-arm64.patch
```

## 构建调度器模块

### 1. 在容器内构建

```bash
cd /tmp/work

# 构建调度器模块
plugsched-cli build /tmp/work/scheduler
```

### 2. 查看构建输出

构建成功后，会生成 RPM 包：

```bash
# 查找生成的 RPM 包
find /tmp/work/scheduler -name "*.rpm"

# 典型路径：
# /tmp/work/scheduler/working/rpmbuild/RPMS/aarch64/scheduler-xxx-*.aarch64.rpm
```

### 3. 复制 RPM 包到便于访问的位置

```bash
KVER=$(uname -r)
KVER_SHORT=${KVER%.*}

# 复制 RPM 包
cp /tmp/work/scheduler/working/rpmbuild/RPMS/aarch64/scheduler-*-${KVER_SHORT}.*.aarch64.rpm \
   /tmp/work/scheduler-debian-arm64.rpm

ls -lh /tmp/work/scheduler-debian-arm64.rpm
```

### 4. 退出容器

```bash
exit
```

## 安装和测试

回到主机系统，安装并测试调度器模块。

### 1. 安装 RPM 工具（如果尚未安装）

```bash
sudo apt install -y rpm alien
```

### 2. 转换 RPM 到 DEB（可选）

```bash
cd /tmp/work

# 将 RPM 转换为 DEB
sudo alien --to-deb scheduler-debian-arm64.rpm

# 这将生成 .deb 文件
ls -lh scheduler*.deb
```

### 3. 直接使用 RPM 安装（推荐）

Debian 系统也可以直接使用 rpm 命令：

```bash
cd /tmp/work

# 查看 RPM 内容
rpm -qlp scheduler-debian-arm64.rpm

# 安装调度器模块
sudo rpm -ivh scheduler-debian-arm64.rpm
```

### 4. 或者手动安装

```bash
# 提取 RPM 内容
cd /tmp/work
mkdir rpm-extract
cd rpm-extract
rpm2cpio ../scheduler-debian-arm64.rpm | cpio -idmv

# 手动复制文件
sudo cp -r lib/modules/$(uname -r)/* /lib/modules/$(uname -r)/
sudo depmod -a
```

### 5. 验证模块安装

```bash
# 检查模块是否已加载
lsmod | grep scheduler

# 应该看到类似输出：
# scheduler             503808  1

# 查看内核日志
dmesg | tail -n 20

# 应该看到类似信息：
# [ xxxx.xxxxxx] Hi, scheduler mod is installing!
# [ xxxx.xxxxxx] scheduler: total initialization time is ...
# [ xxxx.xxxxxx] scheduler module is loading
```

### 6. 检查新的 sched_feature

```bash
# 查看 sched_features
cat /sys/kernel/debug/sched_features

# 应该能看到 DEBIAN_ARM64_TEST 功能（默认关闭状态）
# NO_DEBIAN_ARM64_TEST ...
```

### 7. 测试新功能

```bash
# 启用新的 sched_feature
echo DEBIAN_ARM64_TEST > /sys/kernel/debug/sched_features

# 检查内核日志，应该能看到我们的测试消息
dmesg | grep "Debian ARM64"

# 预期输出：
# [ xxxx.xxxxxx] Plugsched: Debian ARM64 scheduler is working!
```

### 8. 性能测试（可选）

```bash
# 运行一些基准测试来验证调度器工作正常
# 例如使用 sysbench

sudo apt install -y sysbench

# CPU 测试
sysbench cpu --threads=4 --time=10 run

# 线程测试
sysbench threads --threads=64 --time=10 run
```

## 卸载调度器模块

### 1. 使用 RPM 卸载

```bash
# 查找已安装的调度器包
rpm -qa | grep scheduler

# 卸载
sudo rpm -e scheduler-xxx
```

### 2. 验证卸载

```bash
# 检查模块是否已卸载
lsmod | grep scheduler

# 查看卸载日志
dmesg | tail -n 10

# 应该看到：
# [ xxxx.xxxxxx] scheduler module is unloading
# [ xxxx.xxxxxx] Bye, scheduler mod has be removed!

# 检查 sched_feature 是否恢复
cat /sys/kernel/debug/sched_features

# DEBIAN_ARM64_TEST 应该不再存在
```

**重要**: 不要使用 `rmmod` 命令直接卸载调度器模块，必须使用 `rpm -e` 命令卸载。

## 故障排除

### 问题 1: 找不到 vmlinux 文件

**症状**:
```
vmlinux not found, please install kernel-debuginfo
```

**解决方案**:
- Debian 系统需要手动构建内核以生成 vmlinux
- 确保在构建时启用了 DEBUG_INFO 选项
- 将 vmlinux 复制到 `/usr/lib/debug/lib/modules/$(uname -r)/` 目录

### 问题 2: Module.symvers 不存在

**症状**:
```
Module.symvers not found
```

**解决方案**:
```bash
# 从已构建的内核复制
sudo cp /tmp/work/kernel/Module.symvers /usr/src/kernels/$(uname -r)/
```

### 问题 3: 容器无法访问主机文件

**症状**:
容器内看不到挂载的目录

**解决方案**:
```bash
# 确保使用了正确的挂载参数
# 检查 SELinux/AppArmor 设置（可能需要添加 :z 或 :Z 标志）
docker run -itd \
    --name=plugsched \
    -v /tmp/work:/tmp/work:z \
    -v /usr/src/kernels:/usr/src/kernels:z \
    -v /usr/lib/debug/lib/modules:/usr/lib/debug/lib/modules:z \
    plugsched-sdk:debian-arm64
```

### 问题 4: 构建失败 - 缺少 gcc-python-plugin

**症状**:
```
gcc-python-plugin not found
```

**解决方案**:
Debian 系统中 gcc-python-plugin 可能不可用或名称不同：

```bash
# 在 Dockerfile.debian 中添加
RUN apt-get install -y gcc-plugin-dev python3-dev

# 或者尝试从源代码安装 gcc-python-plugin
```

### 问题 5: 架构不匹配

**症状**:
```
Architecture mismatch: expected x86_64, got aarch64
```

**解决方案**:
- 确保使用 ARM64 架构的容器镜像
- 检查构建脚本中是否硬编码了 x86_64
- 修改 SPEC 文件以支持 aarch64

### 问题 6: 符号未定义错误

**症状**:
```
undefined symbol: xxxx
```

**解决方案**:
```bash
# 确保 vmlinux 包含所有必要的符号
nm /usr/lib/debug/lib/modules/$(uname -r)/vmlinux | grep xxxx

# 检查内核配置
grep CONFIG_KALLSYMS /boot/config-$(uname -r)
# 应该显示 CONFIG_KALLSYMS=y 和 CONFIG_KALLSYMS_ALL=y
```

### 问题 7: 无法加载模块 - 版本不匹配

**症状**:
```
module verification failed: signature and/or required key missing
```

**解决方案**:
```bash
# 如果内核启用了模块签名验证，需要禁用或签名模块
# 临时禁用（需要重启）：
sudo mokutil --disable-validation

# 或者在 GRUB 中添加内核参数：
# module.sig_enforce=0
```

### 问题 8: 内核恐慌或系统不稳定

**症状**:
系统崩溃或调度器行为异常

**解决方案**:
1. 通过串口控制台或紧急模式启动
2. 卸载调度器模块：`rpm -e scheduler-xxx`
3. 检查内核日志：`journalctl -k`
4. 验证您的修改没有破坏调度器逻辑
5. 在测试环境中验证修改

### 获取帮助

如果遇到其他问题：

1. 查看项目文档: https://github.com/LiJiajun-WHU/plugsched
2. 检查现有 Issues: https://github.com/LiJiajun-WHU/plugsched/issues
3. 提交新的 Issue，包含：
   - 系统信息（`uname -a`）
   - 内核版本
   - 完整的错误日志
   - 您所做的修改

## 参考资料

- [Plugsched GitHub 仓库](https://github.com/LiJiajun-WHU/plugsched)
- [README_zh.md](../README_zh.md) - Plugsched 中文文档
- [Support-various-Linux-distros.md](./Support-various-Linux-distros.md) - 支持不同 Linux 发行版
- [Working-without-rpm-or-srpm.md](./Working-without-rpm-or-srpm.md) - 无 RPM 包的工作流程
- [Debian Kernel Handbook](https://kernel-team.pages.debian.net/kernel-handbook/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)

## 总结

本指南介绍了在 ARM64 Debian 系统上使用 plugsched 的完整流程：

1. ✅ 安装必要的开发工具和依赖
2. ✅ 构建带调试符号的内核
3. ✅ 设置 plugsched 容器环境
4. ✅ 提取和修改调度器代码
5. ✅ 构建调度器模块
6. ✅ 安装和测试新调度器
7. ✅ 验证功能和性能
8. ✅ 安全卸载模块

Plugsched 为 Debian ARM64 系统提供了强大的调度器热升级能力，允许在不重启系统的情况下动态替换和测试新的调度策略。

**重要提示**: 
- 始终在测试环境中先验证您的修改
- 保持系统和内核的备份
- 遵循 [Limitations](../README_zh.md#limitations) 中的约束
- 不要在生产环境中直接使用未经测试的调度器

祝您使用愉快！
