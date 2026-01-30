# Chapter 4: Installation 📥

> "The journey of a thousand miles begins with a single step" - Lao Tzu

## 🎯 Overview

There are **three ways** to install Firecracker:

1. **Download Pre-built Binary** ⭐ **(Recommended for beginners)**
   - Fastest and easiest
   - No compilation needed
   - Ready to use in minutes

2. **Build from Source** 🔨
   - For developers who want to customize
   - Requires Rust toolchain
   - Takes longer

3. **Use Docker** 🐳
   - For testing and development
   - Isolated environment
   - Easy to clean up

**We recommend Method 1 for your first time!**

---

## 🚀 Method 1: Download Pre-built Binary (Recommended)

This is the **easiest way** to get started!

### Step 1: Choose Your Version

First, let's check what's available:

```bash
# Check the latest release
curl -s https://api.github.com/repos/firecracker-microvm/firecracker/releases/latest | grep "browser_download_url" | grep "linux"
```

Or visit: https://github.com/firecracker-microvm/firecracker/releases

### Step 2: Download Firecracker

We'll download the latest stable release:

```bash
# Set the version (you can change this)
VERSION=v1.14.1

# Download Firecracker binary
wget https://github.com/firecracker-microvm/firecracker/releases/download/${VERSION}/firecracker-${VERSION}-x86_64.tgz

# Extract the archive
tar xzvf firecracker-${VERSION}-x86_64.tgz

# The binary is now in the release-v1.14.1 directory
ls -lh release-${VERSION}-x86_64/
```

**Expected output:**
```
-rwxr-xr-x 1 user user 8.2M Jan 30 12:00 firecracker-${VERSION}-x86_64
-rwxr-xr-x 1 user user 2.1M Jan 30 12:00 jailer-${VERSION}-x86_64
```

### Step 3: Install Firecracker

```bash
# Copy to a location in your PATH
sudo cp release-${VERSION}-x86_64/firecracker-${VERSION}-x86_64 /usr/local/bin/firecracker
sudo cp release-${VERSION}-x86_64/jailer-${VERSION}-x86_64 /usr/local/bin/jailer

# Make them executable
sudo chmod +x /usr/local/bin/firecracker
sudo chmod +x /usr/local/bin/jailer

# Verify installation
firecracker --version
jailer --version
```

**Expected output:**
```
Firecracker 1.14.1
Jailer 1.14.1
```

### Step 4: Create Working Directory

```bash
# Create a directory for Firecracker files
mkdir -p ~/firecracker-demo
cd ~/firecracker-demo
```

### Step 5: Verification

Let's make sure everything works:

```bash
# Test Firecracker (just check if it runs)
firecracker --help
```

If you see the help message, you're all set! 🎉

---

## 🔨 Method 2: Build from Source

For developers who want to build Firecracker themselves.

### Step 1: Install Rust

```bash
# Install Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Source the Rust environment
source $HOME/.cargo/env

# Verify Rust installation
rustc --version
cargo --version
```

### Step 2: Install Build Dependencies

#### Ubuntu/Debian:
```bash
sudo apt-get update
sudo apt-get install -y \
    git \
    build-essential \
    python3 \
    pkg-config \
    libssl-dev
```

#### Fedora/RHEL/CentOS:
```bash
sudo dnf install -y \
    git \
    gcc \
    gcc-c++ \
    make \
    python3 \
    pkg-config \
    openssl-devel
```

### Step 3: Clone and Build

```bash
# Clone the repository
git clone https://github.com/firecracker-microvm/firecracker.git
cd firecracker

# Check out the latest release (optional, but recommended)
git checkout v1.14.1

# Build Firecracker
./tools/devtool build

# This will take a few minutes...
# The binaries will be in ./build/cargo_target/<target>/release/
```

### Step 4: Install

```bash
# Find the built binaries
ls -lh build/cargo_target/*/release/firecracker
ls -lh build/cargo_target/*/release/jailer

# Copy to /usr/local/bin
sudo cp build/cargo_target/*/release/firecracker /usr/local/bin/
sudo cp build/cargo_target/*/release/jailer /usr/local/bin/

# Make executable
sudo chmod +x /usr/local/bin/firecracker
sudo chmod +x /usr/local/bin/jailer

# Verify
firecracker --version
```

---

## 🐳 Method 3: Using Docker

Great for testing without installing anything permanently.

### Step 1: Install Docker

If you don't have Docker:

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -a -G docker $USER
newgrp docker
```

### Step 2: Pull Firecracker Image

```bash
# Pull the official Firecracker image
docker pull firecracker-microvm/firecracker:latest
```

### Step 3: Run Firecracker in Docker

```bash
# Run Firecracker
docker run -it --rm \
    --device /dev/kvm \
    --device /dev/vfio/vfio \
    --cap-add NET_ADMIN \
    --cap-add SYS_ADMIN \
    firecracker-microvm/firecracker:latest
```

**Note**: This method is more complex and requires additional setup. It's best for testing rather than production use.

---

## 📥 Additional Required Files

Firecracker needs a few more things to run:

### 1. Linux Kernel

You'll need a Linux kernel configured for microVMs:

```bash
cd ~/firecracker-demo

# Download a pre-built kernel (recommended for beginners)
wget https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/xenial-generic-linux-4.14.gz

# Or use your own (advanced)
```

### 2. Root Filesystem (Rootfs)

You'll need a root filesystem for the microVM:

```bash
cd ~/firecracker-demo

# Download an Ubuntu filesystem (recommended)
wget https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/xenial-rootfs.ext4

# Or create your own (advanced)
```

### 3. Verify Files

```bash
# List your files
ls -lh ~/firecracker-demo/
```

**Expected files:**
```
-rw-r--r-- 1 user user  11M xenial-generic-linux-4.14.gz
-rw-r--r-- 1 user user  44M xenial-rootfs.ext4
```

---

## 🧪 Test Your Installation

Let's verify everything is ready:

```bash
# Create a test script
cat > ~/firecracker-demo/check-install.sh << 'EOF'
#!/bin/bash

echo "=== Firecracker Installation Check ==="
echo ""

# Check Firecracker binary
echo "1. Firecracker Binary:"
if command -v firecracker &> /dev/null; then
    echo "   ✅ Firecracker is installed"
    firecracker --version
else
    echo "   ❌ Firecracker is NOT installed"
fi
echo ""

# Check Jailer binary
echo "2. Jailer Binary:"
if command -v jailer &> /dev/null; then
    echo "   ✅ Jailer is installed"
    jailer --version
else
    echo "   ⚠️  Jailer is NOT installed (optional but recommended)"
fi
echo ""

# Check kernel file
echo "3. Kernel File:"
if [ -f ~/firecracker-demo/xenial-generic-linux-4.14.gz ]; then
    echo "   ✅ Kernel file found"
else
    echo "   ❌ Kernel file NOT found"
fi
echo ""

# Check rootfs file
echo "4. Root Filesystem:"
if [ -f ~/firecracker-demo/xenial-rootfs.ext4 ]; then
    echo "   ✅ Root filesystem found"
else
    echo "   ❌ Root filesystem NOT found"
fi
echo ""

# Check KVM
echo "5. KVM Device:"
if [ -e /dev/kvm ]; then
    echo "   ✅ /dev/kvm exists"
else
    echo "   ❌ /dev/kvm does NOT exist"
fi
echo ""

echo "=== Check Complete! ==="
EOF

# Make it executable and run
chmod +x ~/firecracker-demo/check-install.sh
~/firecracker-demo/check-install.sh
```

---

## 🎯 Installation Summary

### What You Just Installed:

1. **Firecracker** - The main VMM binary
2. **Jailer** - Security tool (optional but recommended)
3. **Linux Kernel** - For the microVM
4. **Root Filesystem** - Ubuntu filesystem for the microVM

### Directory Structure:

```
~/firecracker-demo/
├── xenial-generic-linux-4.14.gz    # Kernel
├── xenial-rootfs.ext4              # Filesystem
└── check-install.sh                # Test script

/usr/local/bin/
├── firecracker                     # Firecracker binary
└── jailer                          # Jailer binary
```

---

## ✅ Success Criteria

You're ready for the next chapter if:

- [ ] `firecracker --version` works
- [ ] `jailer --version` works (optional)
- [ ] You have a kernel file
- [ ] You have a root filesystem
- [ ] `/dev/kvm` exists and is accessible

---

## 🆘 Troubleshooting

### Problem: "firecracker: command not found"
**Solution**: Make sure `/usr/local/bin` is in your PATH:
```bash
echo $PATH | grep usr/local/bin || export PATH=$PATH:/usr/local/bin
```

### Problem: "Permission denied"
**Solution**: Make sure binaries are executable:
```bash
sudo chmod +x /usr/local/bin/firecracker
sudo chmod +x /usr/local/bin/jailer
```

### Problem: "Download fails"
**Solution**: Try a different mirror or check your internet connection:
```bash
# Alternative download locations
# AWS (same files, different region)
# Or build from source
```

### Problem: Build fails
**Solution**: Make sure you have all dependencies:
```bash
# Check Rust version
rustc --version  # Should be 1.70 or later

# Install missing dependencies
sudo apt-get install build-essential pkg-config libssl-dev
```

---

## 🚀 Ready to Launch!

Congratulations! You've successfully installed Firecracker. 

**Next up**: We'll run your first microVM! 🎉

[Continue to Chapter 5: Quick Start →](./05-quick-start.md)

---

*Need help? Check the [official Firecracker docs](https://firecracker-microvm.github.io/getting-started/installation/)*
