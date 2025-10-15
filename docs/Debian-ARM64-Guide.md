# Complete Guide: Using Plugsched on ARM64 Debian Systems

This document provides a comprehensive guide on how to use plugsched for live updating the Linux kernel scheduler on ARM64 Debian systems, covering the entire process from download to deployment and testing.

## Table of Contents
- [System Requirements](#system-requirements)
- [Prerequisites](#prerequisites)
- [Installing Required Packages](#installing-required-packages)
- [Obtaining Kernel Source Code](#obtaining-kernel-source-code)
- [Building Kernel with Debug Symbols](#building-kernel-with-debug-symbols)
- [Setting Up Plugsched Container Environment](#setting-up-plugsched-container-environment)
- [Extracting and Initializing Scheduler](#extracting-and-initializing-scheduler)
- [Modifying Scheduler Code](#modifying-scheduler-code)
- [Building Scheduler Module](#building-scheduler-module)
- [Installation and Testing](#installation-and-testing)
- [Uninstalling Scheduler Module](#uninstalling-scheduler-module)
- [Troubleshooting](#troubleshooting)

## System Requirements

- **Architecture**: ARM64 (aarch64)
- **Operating System**: Debian 10/11/12 or Debian-based distributions
- **Kernel Version**: Recommended 4.19+ or 5.x series
- **Memory**: At least 4GB RAM recommended
- **Disk Space**: At least 20GB available space (for kernel source and build artifacts)
- **Permissions**: root or sudo access

## Prerequisites

### 1. Check System Information

```bash
# Check architecture
uname -m
# Should display: aarch64

# Check current kernel version
uname -r
# Example: 5.10.0-23-arm64

# Check system version
cat /etc/os-release
```

### 2. Record Kernel Version Information

```bash
export KERNEL_VERSION=$(uname -r)
echo "Current kernel version: $KERNEL_VERSION"
```

## Installing Required Packages

### 1. Update Package Lists

```bash
sudo apt update
```

### 2. Install Basic Development Tools

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

### 3. Install Container Runtime

Debian systems can use Docker or Podman. We recommend Docker:

```bash
# Install Docker
sudo apt install -y docker.io

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add current user to docker group (optional, to avoid using sudo)
sudo usermod -aG docker $USER
# Note: You need to log out and back in for this to take effect

# Alternative: If you prefer Podman
# sudo apt install -y podman
```

### 4. Install Python Development Dependencies

```bash
sudo apt install -y \
    python3 \
    python3-pip \
    python3-dev \
    python3-yaml \
    libyaml-dev
```

### 5. Install Kernel Development Packages and Debug Symbols

```bash
# Install kernel headers for the running kernel
sudo apt install -y linux-headers-$(uname -r)

# Try to install debug symbol package (if available)
# Note: Debian debug symbol packages may require configuring dbg sources
sudo apt install -y linux-image-$(uname -r)-dbg 2>/dev/null || echo "Debug symbols package not available, manual build required"
```

**Important Note**: Debian systems typically don't provide complete kernel-debuginfo packages like RHEL/Anolis systems. If the `linux-image-*-dbg` package is not available, you'll need to build the kernel from source to generate the `vmlinux` file.

## Obtaining Kernel Source Code

### Method 1: Using apt-get source (Recommended for Debian Official Kernels)

```bash
# Create working directory
mkdir -p /tmp/work
cd /tmp/work

# Add deb-src sources (if not already enabled)
sudo sed -i 's/^# deb-src/deb-src/' /etc/apt/sources.list
sudo apt update

# Download kernel source code
apt-get source linux-image-$(uname -r)

# Source will be extracted to current directory, e.g.: linux-5.10.xxx
```

### Method 2: Clone from Git Repository (For Specific Kernel Versions)

```bash
cd /tmp/work

# For Debian official kernel
git clone --depth 1 --branch debian/$(uname -r | sed 's/-.*//') \
    https://salsa.debian.org/kernel-team/linux.git kernel

# Or get mainline kernel from kernel.org
# git clone --depth 1 --branch v5.10 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git kernel
```

### Method 3: Download Kernel Tarball

```bash
cd /tmp/work

# Determine kernel version
KVER=$(uname -r | sed 's/-.*//') # Example: 5.10.0

# Download from kernel.org
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-${KVER}.tar.xz

# Extract
tar xf linux-${KVER}.tar.xz
mv linux-${KVER} kernel
```

## Building Kernel with Debug Symbols

Since Debian systems may not provide the `vmlinux` file (kernel with debug symbols), we need to build the kernel to generate it.

### 1. Prepare Kernel Configuration

```bash
cd /tmp/work/kernel

# Copy current running kernel's configuration
cp /boot/config-$(uname -r) .config

# Or use default configuration
# make defconfig

# Enable necessary debug options
scripts/config --enable CONFIG_DEBUG_INFO
scripts/config --enable CONFIG_DEBUG_INFO_DWARF4
scripts/config --disable CONFIG_DEBUG_INFO_REDUCED
scripts/config --enable CONFIG_KALLSYMS
scripts/config --enable CONFIG_KALLSYMS_ALL
```

### 2. Adjust Version Information in Makefile

Ensure the built kernel version matches the currently running kernel:

```bash
# Get full kernel version string
FULL_KVER=$(uname -r)

# Separate version number and EXTRAVERSION
BASE_VER=$(echo $FULL_KVER | cut -d'-' -f1)
EXTRA_VER=$(echo $FULL_KVER | cut -d'-' -f2-)

# Modify Makefile
# Edit Makefile to set EXTRAVERSION
# Example: EXTRAVERSION = -23-arm64
# You can use sed or edit manually
```

### 3. Build Kernel

```bash
# Use multi-core compilation (adjust -j parameter based on your CPU cores)
make -j$(nproc)

# After build completes, vmlinux file will be in kernel source root directory
ls -lh vmlinux
```

**Note**: Full kernel build may take 1-2 hours depending on your hardware performance.

### 4. Set Up Debug Symbol Paths

```bash
# Create standard debug symbol directory
sudo mkdir -p /usr/lib/debug/lib/modules/$(uname -r)/

# Copy vmlinux to standard location
sudo cp vmlinux /usr/lib/debug/lib/modules/$(uname -r)/vmlinux

# Verify file exists
ls -lh /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
```

### 5. Prepare Module.symvers and .config

```bash
# Create kernel development files directory
sudo mkdir -p /usr/src/kernels/$(uname -r)

# Copy necessary files
sudo cp Module.symvers /usr/src/kernels/$(uname -r)/
sudo cp .config /usr/src/kernels/$(uname -r)/
sudo cp Makefile /usr/src/kernels/$(uname -r)/

# Copy entire source directory (optional but recommended)
cd /tmp/work
sudo cp -r kernel /usr/src/kernels/$(uname -r)/source
```

## Setting Up Plugsched Container Environment

### 1. Build Plugsched Docker Image (For Debian/ARM64)

Since the official image is based on Anolis OS (RPM system), we need to create a Debian-compatible version:

```bash
cd /tmp/work

# Clone plugsched repository
git clone https://github.com/LiJiajun-WHU/plugsched.git

cd plugsched

# Create Dockerfile for Debian
cat > Dockerfile.debian << 'EOF'
# Dockerfile for Debian/ARM64
FROM debian:11

# Install basic packages
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

# Install Python dependencies
RUN pip3 install sh docopt pyyaml colorlog

# Copy plugsched files
COPY . /usr/local/lib/plugsched/
RUN ln -s /usr/local/lib/plugsched/cli.py /usr/local/bin/plugsched-cli && \
    chmod +x /usr/local/bin/plugsched-cli

WORKDIR /root
EOF

# Build image
docker build -f Dockerfile.debian -t plugsched-sdk:debian-arm64 .
```

### 2. Start Container

```bash
# Return to working directory
cd /tmp/work

# Start container with necessary volume mounts
docker run -itd \
    --name=plugsched \
    -v /tmp/work:/tmp/work \
    -v /usr/src/kernels:/usr/src/kernels \
    -v /usr/lib/debug/lib/modules:/usr/lib/debug/lib/modules \
    plugsched-sdk:debian-arm64

# Verify container is running
docker ps | grep plugsched
```

### 3. Enter Container

```bash
docker exec -it plugsched bash
cd /tmp/work
```

## Extracting and Initializing Scheduler

Execute the following operations inside the container:

### 1. Verify Environment

```bash
# Check if plugsched-cli is available
plugsched-cli --help

# Check if necessary files exist
ls -la /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
ls -la /usr/src/kernels/$(uname -r)/Module.symvers
ls -la /usr/src/kernels/$(uname -r)/.config
```

### 2. Initialize Using dev_init Method (Recommended for Debian)

Since Debian doesn't have kernel.srpm packages, we use the `dev_init` method:

```bash
cd /tmp/work

# Ensure kernel is already built (vmlinux generated)
# If not yet built, first build kernel on host

# Initialize scheduler using dev_init
plugsched-cli dev_init /tmp/work/kernel ./scheduler
```

**Note**: The `dev_init` method will automatically look for the `vmlinux` file in the kernel source directory.

### 3. Or Use init Method (If debuginfo is Available)

```bash
# If you already have vmlinux and related files in standard locations
plugsched-cli init $(uname -r) /tmp/work/kernel ./scheduler
```

### 4. Verify Extraction Results

```bash
# Check generated directory structure
ls -la ./scheduler/
ls -la ./scheduler/kernel/sched/mod/

# You should see scheduler source files such as:
# core.c, fair.c, rt.c, deadline.c, features.h, etc.
```

## Modifying Scheduler Code

### 1. Example Modification: Adding a New sched_feature

To test if plugsched works, let's add a simple feature:

```bash
cd /tmp/work/scheduler

# Edit features.h to add new feature
cat >> kernel/sched/mod/features.h << 'EOF'

/* Plugsched test feature for Debian ARM64 */
SCHED_FEAT(DEBIAN_ARM64_TEST, false)
EOF

# Edit core.c to add test code in __schedule function
# Find __schedule function and add the following
```

Edit `kernel/sched/mod/core.c` using an editor inside the container (or via host):

```c
// Add at the beginning of __schedule function:
if (sched_feat(DEBIAN_ARM64_TEST))
    printk_once("Plugsched: Debian ARM64 scheduler is working!\n");
```

### 2. Complete Example Patch

You can create a patch file:

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

# Apply patch
cd /tmp/work/scheduler
patch -p1 < /tmp/work/test-debian-arm64.patch
```

## Building Scheduler Module

### 1. Build Inside Container

```bash
cd /tmp/work

# Build scheduler module
plugsched-cli build /tmp/work/scheduler
```

### 2. View Build Output

After successful build, an RPM package will be generated:

```bash
# Find generated RPM package
find /tmp/work/scheduler -name "*.rpm"

# Typical path:
# /tmp/work/scheduler/working/rpmbuild/RPMS/aarch64/scheduler-xxx-*.aarch64.rpm
```

### 3. Copy RPM Package to Accessible Location

```bash
KVER=$(uname -r)
KVER_SHORT=${KVER%.*}

# Copy RPM package
cp /tmp/work/scheduler/working/rpmbuild/RPMS/aarch64/scheduler-*-${KVER_SHORT}.*.aarch64.rpm \
   /tmp/work/scheduler-debian-arm64.rpm

ls -lh /tmp/work/scheduler-debian-arm64.rpm
```

### 4. Exit Container

```bash
exit
```

## Installation and Testing

Back on the host system, install and test the scheduler module.

### 1. Install RPM Tools (If Not Already Installed)

```bash
sudo apt install -y rpm alien
```

### 2. Convert RPM to DEB (Optional)

```bash
cd /tmp/work

# Convert RPM to DEB
sudo alien --to-deb scheduler-debian-arm64.rpm

# This will generate a .deb file
ls -lh scheduler*.deb
```

### 3. Direct Installation Using RPM (Recommended)

Debian systems can also use the rpm command directly:

```bash
cd /tmp/work

# View RPM contents
rpm -qlp scheduler-debian-arm64.rpm

# Install scheduler module
sudo rpm -ivh scheduler-debian-arm64.rpm
```

### 4. Or Manual Installation

```bash
# Extract RPM contents
cd /tmp/work
mkdir rpm-extract
cd rpm-extract
rpm2cpio ../scheduler-debian-arm64.rpm | cpio -idmv

# Manually copy files
sudo cp -r lib/modules/$(uname -r)/* /lib/modules/$(uname -r)/
sudo depmod -a
```

### 5. Verify Module Installation

```bash
# Check if module is loaded
lsmod | grep scheduler

# Should see output like:
# scheduler             503808  1

# View kernel log
dmesg | tail -n 20

# Should see messages like:
# [ xxxx.xxxxxx] Hi, scheduler mod is installing!
# [ xxxx.xxxxxx] scheduler: total initialization time is ...
# [ xxxx.xxxxxx] scheduler module is loading
```

### 6. Check New sched_feature

```bash
# View sched_features
cat /sys/kernel/debug/sched_features

# You should see the DEBIAN_ARM64_TEST feature (disabled by default)
# NO_DEBIAN_ARM64_TEST ...
```

### 7. Test New Feature

```bash
# Enable new sched_feature
echo DEBIAN_ARM64_TEST > /sys/kernel/debug/sched_features

# Check kernel log for our test message
dmesg | grep "Debian ARM64"

# Expected output:
# [ xxxx.xxxxxx] Plugsched: Debian ARM64 scheduler is working!
```

### 8. Performance Testing (Optional)

```bash
# Run some benchmarks to verify scheduler works correctly
# For example, using sysbench

sudo apt install -y sysbench

# CPU test
sysbench cpu --threads=4 --time=10 run

# Thread test
sysbench threads --threads=64 --time=10 run
```

## Uninstalling Scheduler Module

### 1. Uninstall Using RPM

```bash
# Find installed scheduler package
rpm -qa | grep scheduler

# Uninstall
sudo rpm -e scheduler-xxx
```

### 2. Verify Uninstallation

```bash
# Check if module is unloaded
lsmod | grep scheduler

# View uninstall log
dmesg | tail -n 10

# Should see:
# [ xxxx.xxxxxx] scheduler module is unloading
# [ xxxx.xxxxxx] Bye, scheduler mod has be removed!

# Check if sched_feature is restored
cat /sys/kernel/debug/sched_features

# DEBIAN_ARM64_TEST should no longer exist
```

**Important**: Do not use the `rmmod` command to directly unload the scheduler module. You must use the `rpm -e` command to uninstall.

## Troubleshooting

### Issue 1: vmlinux File Not Found

**Symptoms**:
```
vmlinux not found, please install kernel-debuginfo
```

**Solution**:
- Debian systems require manually building the kernel to generate vmlinux
- Ensure DEBUG_INFO options are enabled during build
- Copy vmlinux to `/usr/lib/debug/lib/modules/$(uname -r)/` directory

### Issue 2: Module.symvers Does Not Exist

**Symptoms**:
```
Module.symvers not found
```

**Solution**:
```bash
# Copy from built kernel
sudo cp /tmp/work/kernel/Module.symvers /usr/src/kernels/$(uname -r)/
```

### Issue 3: Container Cannot Access Host Files

**Symptoms**:
Cannot see mounted directories inside container

**Solution**:
```bash
# Ensure correct mount parameters are used
# Check SELinux/AppArmor settings (may need to add :z or :Z flags)
docker run -itd \
    --name=plugsched \
    -v /tmp/work:/tmp/work:z \
    -v /usr/src/kernels:/usr/src/kernels:z \
    -v /usr/lib/debug/lib/modules:/usr/lib/debug/lib/modules:z \
    plugsched-sdk:debian-arm64
```

### Issue 4: Build Failure - Missing gcc-python-plugin

**Symptoms**:
```
gcc-python-plugin not found
```

**Solution**:
gcc-python-plugin may not be available or named differently in Debian:

```bash
# Add to Dockerfile.debian
RUN apt-get install -y gcc-plugin-dev python3-dev

# Or try installing gcc-python-plugin from source
```

### Issue 5: Architecture Mismatch

**Symptoms**:
```
Architecture mismatch: expected x86_64, got aarch64
```

**Solution**:
- Ensure you're using ARM64 architecture container image
- Check if build scripts have hardcoded x86_64
- Modify SPEC file to support aarch64

### Issue 6: Undefined Symbol Errors

**Symptoms**:
```
undefined symbol: xxxx
```

**Solution**:
```bash
# Ensure vmlinux contains all necessary symbols
nm /usr/lib/debug/lib/modules/$(uname -r)/vmlinux | grep xxxx

# Check kernel configuration
grep CONFIG_KALLSYMS /boot/config-$(uname -r)
# Should show CONFIG_KALLSYMS=y and CONFIG_KALLSYMS_ALL=y
```

### Issue 7: Cannot Load Module - Version Mismatch

**Symptoms**:
```
module verification failed: signature and/or required key missing
```

**Solution**:
```bash
# If kernel has module signature verification enabled, need to disable or sign module
# Temporarily disable (requires reboot):
sudo mokutil --disable-validation

# Or add kernel parameter in GRUB:
# module.sig_enforce=0
```

### Issue 8: Kernel Panic or System Instability

**Symptoms**:
System crashes or scheduler behaves abnormally

**Solution**:
1. Boot via serial console or emergency mode
2. Uninstall scheduler module: `rpm -e scheduler-xxx`
3. Check kernel logs: `journalctl -k`
4. Verify your modifications didn't break scheduler logic
5. Test modifications in test environment

### Getting Help

If you encounter other issues:

1. Check project documentation: https://github.com/LiJiajun-WHU/plugsched
2. Review existing Issues: https://github.com/LiJiajun-WHU/plugsched/issues
3. Submit a new Issue including:
   - System information (`uname -a`)
   - Kernel version
   - Complete error logs
   - Your modifications

## References

- [Plugsched GitHub Repository](https://github.com/LiJiajun-WHU/plugsched)
- [README.md](../README.md) - Plugsched English Documentation
- [Support-various-Linux-distros.md](./Support-various-Linux-distros.md) - Supporting Different Linux Distributions
- [Working-without-rpm-or-srpm.md](./Working-without-rpm-or-srpm.md) - Workflows Without RPM Packages
- [Debian Kernel Handbook](https://kernel-team.pages.debian.net/kernel-handbook/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)

## Summary

This guide covered the complete process of using plugsched on ARM64 Debian systems:

1. ✅ Install necessary development tools and dependencies
2. ✅ Build kernel with debug symbols
3. ✅ Set up plugsched container environment
4. ✅ Extract and modify scheduler code
5. ✅ Build scheduler module
6. ✅ Install and test new scheduler
7. ✅ Verify functionality and performance
8. ✅ Safely uninstall module

Plugsched provides powerful scheduler live update capabilities for Debian ARM64 systems, allowing dynamic replacement and testing of new scheduling policies without system reboot.

**Important Notes**: 
- Always verify your modifications in test environment first
- Keep system and kernel backups
- Follow constraints in [Limitations](../README.md#limitations)
- Do not use untested schedulers in production environment

Happy coding!
