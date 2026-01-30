# Chapter 3: Prerequisites 📋

> "Proper preparation prevents poor performance" - Wise Saying

## 🎯 Before We Start

Before installing Firecracker, let's make sure your computer is ready! Don't worry - we'll check everything together.

---

## 💻 Hardware Requirements

### 1. **CPU with Virtualization Support** 🖥️

Your computer's CPU needs to support hardware virtualization (Intel VT-x or AMD-V).

#### How to Check on Linux:
```bash
# Check if CPU supports virtualization
grep -E 'vmx|svm' /proc/cpuinfo
```

**What you want to see:**
- `vmx` = Intel VT-x (Intel processors)
- `svm` = AMD-V (AMD processors)

**If you see these flags**: ✅ Great! Your CPU supports virtualization!

**If you see nothing**: ❌ Your CPU doesn't support virtualization, or it's disabled in BIOS

#### Enable Virtualization in BIOS (if needed):
1. Restart your computer
2. Enter BIOS/UEFI setup (usually F2, F10, or Delete key)
3. Find CPU settings or Virtualization settings
4. Enable Intel VT-x or AMD-V
5. Save and restart

#### Supported Processors:
- ✅ Intel 64-bit CPUs with VT-x
- ✅ AMD 64-bit CPUs with AMD-V
- ✅ ARM 64-bit CPUs with virtualization

### 2. **Memory Requirements** 🧠

- **Minimum**: 4 GB RAM
- **Recommended**: 8 GB RAM or more
- **Why?**: Each microVM uses 5MB, but you need memory for:
  - Host operating system
  - Linux kernel
  - Root filesystem
  - Multiple microVMs

### 3. **Disk Space** 💾

- **Minimum**: 2 GB free space
- **Recommended**: 10 GB or more
- **Why?**: You'll need space for:
  - Firecracker binary (~10 MB)
  - Linux kernel images (~5-10 MB each)
  - Root filesystems (~50-200 MB each)
  - Logs and configuration files

---

## 🐧 Software Requirements

### Operating System

**Firecracker runs on Linux** - specifically Linux kernel 4.14 or higher.

#### Supported Distributions:
- ✅ Ubuntu 18.04 LTS or newer
- ✅ Debian 10 or newer
- ✅ Amazon Linux 2
- ✅ Fedora 28 or newer
- ✅ RHEL/CentOS 7 or newer
- ✅ Arch Linux

#### NOT Supported:
- ❌ Windows (you can use WSL2, but it's experimental)
- ❌ macOS (you can use a Linux VM)
- ❌ BSD variants

**Tip**: If you're on Windows/Mac, consider:
- Using a cloud provider (AWS, GCP, DigitalOcean)
- Running Linux in a virtual machine
- Using WSL2 on Windows (experimental)

### KVM Module

KVM (Kernel-based Virtual Machine) must be available and loaded.

#### Check if KVM is available:
```bash
# Check if KVM modules are loaded
lsmod | grep kvm
```

**Expected output:**
```
kvm_intel             245760  0
kvm                   655360  1 kvm_intel
```

#### Load KVM if needed:
```bash
# Load KVM modules
sudo modprobe kvm
sudo modprobe kvm_intel    # For Intel CPUs
# OR
sudo modprobe kvm_amd      # For AMD CPUs

# Make it load on boot
echo 'kvm' | sudo tee -a /etc/modules-load.d/kvm.conf
```

#### Check /dev/kvm device:
```bash
ls -l /dev/kvm
```

**Expected output:**
```
crw-rw----+ 1 root kvm 10, 232 Jan 30 12:00 /dev/kvm
```

**If /dev/kvm doesn't exist**:
```bash
# Create the device
sudo modprobe kvm
sudo mknod /dev/kvm c 10 232
sudo chmod 666 /dev/kvm
```

### Required System Packages

#### Ubuntu/Debian:
```bash
sudo apt-get update
sudo apt-get install -y \
    wget \
    git \
    curl \
    build-essential \
    python3 \
    python3-pip \
    jq
```

#### Fedora/RHEL/CentOS:
```bash
sudo dnf install -y \
    wget \
    git \
    curl \
    gcc \
    gcc-c++ \
    make \
    python3 \
    python3-pip \
    jq
```

### Permissions

You need access to `/dev/kvm` and the ability to run `sudo`.

#### Add user to kvm group (optional but recommended):
```bash
# Add current user to kvm group
sudo usermod -a -G kvm $USER

# Log out and log back in for changes to take effect
# Or use: newgrp kvm
```

#### Verify your group membership:
```bash
groups
```

You should see `kvm` in the list.

---

## 🧪 Quick Pre-Flight Checklist

Run this quick check to see if you're ready:

```bash
#!/bin/bash
echo "=== Firecracker Pre-flight Check ==="
echo ""

# Check CPU virtualization
echo "1. CPU Virtualization Support:"
if grep -E 'vmx|svm' /proc/cpuinfo > /dev/null; then
    echo "   ✅ CPU supports virtualization"
else
    echo "   ❌ CPU does NOT support virtualization (or it's disabled)"
fi
echo ""

# Check KVM
echo "2. KVM Module:"
if lsmod | grep kvm > /dev/null; then
    echo "   ✅ KVM module is loaded"
else
    echo "   ❌ KVM module is NOT loaded"
fi
echo ""

# Check /dev/kvm
echo "3. /dev/kvm Device:"
if [ -e /dev/kvm ]; then
    echo "   ✅ /dev/kvm exists"
    if [ -r /dev/kvm ] && [ -w /dev/kvm ]; then
        echo "   ✅ You have read/write access to /dev/kvm"
    else
        echo "   ⚠️  Limited access to /dev/kvm (may need sudo)"
    fi
else
    echo "   ❌ /dev/kvm does NOT exist"
fi
echo ""

# Check Linux kernel version
echo "4. Linux Kernel Version:"
kernel_version=$(uname -r | cut -d. -f1,2)
required_version="4.14"
if [ "$(printf '%s\n' "$required_version" "$kernel_version" | sort -V | head -n1)" = "$required_version" ]; then
    echo "   ✅ Kernel version $(uname -r) (>= 4.14)"
else
    echo "   ❌ Kernel version $(uname -r) is too old (need >= 4.14)"
fi
echo ""

# Check available memory
echo "5. Available Memory:"
total_mem=$(free -m | awk '/^Mem:/{print $2}')
if [ "$total_mem" -ge 4096 ]; then
    echo "   ✅ $total_mem MB RAM (>= 4 GB)"
else
    echo "   ⚠️  Only $total_mem MB RAM (4 GB+ recommended)"
fi
echo ""

# Check disk space
echo "6. Available Disk Space:"
disk_space=$(df -m . | awk 'NR==2 {print $4}')
if [ "$disk_space" -ge 2048 ]; then
    echo "   ✅ $disk_space MB available (>= 2 GB)"
else
    echo "   ❌ Only $disk_space MB available (need at least 2 GB)"
fi
echo ""

echo "=== Check Complete! ==="
```

Save this as `check-prerequisites.sh` and run:
```bash
chmod +x check-prerequisites.sh
./check-prerequisites.sh
```

---

## 📚 Knowledge Prerequisites

### Must Know:
- ✅ Basic command line skills (cd, ls, mkdir, etc.)
- ✅ How to use a text editor (nano, vim, or VS Code)
- ✅ Basic Linux concepts (files, permissions, processes)

### Nice to Know (but not required):
- 📖 What virtualization is
- 📖 Basic understanding of containers
- 📖 Linux kernel basics

### Don't Need:
- ❌ Rust programming (we'll use pre-built binaries)
- ❌ Advanced Linux knowledge
- ❌ Virtualization expertise

---

## 🌐 Alternative: Cloud Environment

If your computer doesn't meet the requirements, or if you just want to try it quickly:

### AWS Lightsail/EC2:
```bash
# Create a t3.micro or t3.nano instance
# OS: Ubuntu 20.04 LTS or 22.04 LTS
# Then SSH in and follow the installation guide
```

### Google Cloud Platform:
```bash
# Create a e2-micro instance
# OS: Ubuntu 20.04 LTS or 22.04 LTS
# Then SSH in and follow the installation guide
```

### DigitalOcean:
```bash
# Create a Basic Droplet
# Plan: $6/month (1GB RAM)
# OS: Ubuntu 22.04 LTS
```

**Advantages of cloud:**
- ✅ Everything pre-configured
- ✅ Fast and easy
- ✅ No local resource usage
- ✅ Can delete when done

---

## ✅ Ready Checklist

Before moving to installation, make sure you have:

- [ ] CPU with virtualization support enabled
- [ ] Linux OS (kernel 4.14+)
- [ ] At least 4 GB RAM (8 GB recommended)
- [ ] At least 2 GB free disk space
- [ ] KVM module loaded
- [ ] /dev/kvm device exists
- [ ] Basic command line skills
- [ ] sudo access (for installation)

---

## 🤔 Having Issues?

### Common Problems:

**Problem**: "No /dev/kvm device"
- **Solution**: Load KVM module: `sudo modprobe kvm`

**Problem**: "Permission denied on /dev/kvm"
- **Solution**: Add user to kvm group or use sudo

**Problem**: "CPU doesn't support virtualization"
- **Solution**: Enable in BIOS or use a cloud provider

**Problem**: "Kernel too old"
- **Solution**: Update your Linux kernel or use a newer distro

---

## 🚀 All Set?

Once you've completed the checklist, you're ready to install Firecracker!

[Continue to Chapter 4: Installation →](./04-installation.md)

---

*Need help? Join the [Firecracker Slack community](https://firecracker-microvm.github.io/) for support!*
