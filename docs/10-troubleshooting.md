# Chapter 10: Troubleshooting - Solving Common Problems 🔧

> "Don't panic! Almost every problem has a solution" - Hitchhiker's Guide

## 🎯 Overview

Even experts run into issues. This chapter helps you diagnose and fix common Firecracker problems.

---

## 🚨 Quick Diagnostic Steps

When something goes wrong, follow these steps:

### Step 1: Check the Logs

```bash
# If you enabled logging
cat /var/log/firecracker.log

# Or check journalctl if running as service
journalctl -u firecracker -n 50
```

### Step 2: Check API Connectivity

```bash
# Test if API socket is responding
curl --unix-socket /tmp/firecracker.sock http://localhost/info

# Expected: JSON with microVM info
# Error: "Failed to connect" = API not running
```

### Step 3: Check Resources

```bash
# Check if Firecracker process is running
ps aux | grep firecracker

# Check /dev/kvm exists
ls -l /dev/kvm

# Check available memory
free -h

# Check disk space
df -h
```

### Step 4: Check MicroVM Status

```bash
# Get microVM info
curl --unix-socket /tmp/firecracker.sock http://localhost/info

# Expected JSON:
{
  "id": "microvm-id",
  "state": "Running",
  "vcpu_count": 2,
  "mem_size_mib": 512
}
```

---

## 🔧 Common Issues and Solutions

### Issue 1: "Failed to connect to Firecracker API"

**Symptoms:**
```bash
curl: (7) Failed to connect to localhost port 80: Connection refused
```

**Causes & Solutions:**

#### Cause 1: Firecracker not running
```bash
# Check if process exists
ps aux | grep firecracker

# Solution: Start Firecracker
firecracker --api-sock /tmp/firecracker.sock
```

#### Cause 2: Wrong socket path
```bash
# Check actual socket location
ls -la /tmp/firecracker*.sock
ls -la /srv/jailer/firecracker/*/root/run/api.socket

# Solution: Use correct path
export FC_SOCKET=/tmp/firecracker.sock  # or actual path
curl --unix-socket $FC_SOCKET http://localhost/info
```

#### Cause 3: Socket permissions
```bash
# Check permissions
ls -l /tmp/firecracker.sock

# If wrong permissions:
sudo chmod 666 /tmp/firecracker.sock

# Or add user to appropriate group
```

---

### Issue 2: microVM Won't Start

**Symptoms:**
```bash
# API returns 400 Bad Request
# Or nothing happens
```

**Causes & Solutions:**

#### Cause 1: Missing kernel or rootfs
```bash
# Check files exist
ls -lh ~/firecracker-demo/xenial-generic-linux-4.14.gz
ls -lh ~/firecracker-demo/xenial-rootfs.ext4

# Solution: Download missing files
wget https://s3.amazonaws.com/.../xenial-generic-linux-4.14.gz
```

#### Cause 2: Invalid configuration
```bash
# Check configuration
curl --unix-socket $FC_SOCKET http://localhost/machine-config
curl --unix-socket $FC_SOCKET http://localhost/boot-source
curl --unix-socket $FC_SOCKET http://localhost/drives/rootfs

# Solution: Fix configuration errors
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/boot-source' \
    -H 'Content-Type: application/json' \
    -d '{"kernel_image_path": "/path/to/kernel"}'
```

#### Cause 3: /dev/kvm not accessible
```bash
# Check permissions
ls -l /dev/kvm

# Solution: Add user to kvm group
sudo usermod -a -G kvm $USER
newgrp kvm
```

#### Cause 4: Insufficient resources
```bash
# Check available memory
free -h

# Solution: Reduce microVM memory
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/machine-config' \
    -H 'Content-Type: application/json' \
    -d '{"vcpu_count": 1, "mem_size_mib": 256}'
```

---

### Issue 3: microVM Starts But No Network

**Symptoms:**
- Can't SSH into microVM
- Can't ping external addresses
- Network interface is down

**Causes & Solutions:**

#### Cause 1: Network interface not configured
```bash
# Check if interface exists in microVM
# From microVM console:
ip addr

# Should show eth0
# If not, configure it:
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/network-interfaces/eth0' \
    -H 'Content-Type: application/json' \
    -d '{
        "iface_id": "eth0",
        "guest_mac": "02:FC:00:00:00:01",
        "host_dev_name": "tap0"
    }'
```

#### Cause 2: Host tap device not configured
```bash
# Check tap device
ip addr show tap0

# If doesn't exist:
sudo ip tuntap add dev tap0 mode tap
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
```

#### Cause 3: No IP address in guest
```bash
# From microVM console:
sudo ip addr add 172.16.0.2/24 dev eth0
sudo ip link set eth0 up

# Or configure DHCP (if your setup supports it)
```

#### Cause 4: Firewall blocking
```bash
# Check firewall rules
sudo iptables -L -n

# Add rule to allow traffic
sudo iptables -I FORWARD -i tap0 -j ACCEPT
sudo iptables -I FORWARD -o tap0 -j ACCEPT
```

---

### Issue 4: High Memory Usage

**Symptoms:**
- Host running out of memory
- microVMs using more than expected

**Solutions:**

#### Check actual usage:
```bash
# Check Firecracker process memory
ps aux | grep firecracker
# Look at VSZ (virtual memory) and RSS (resident set)

# Check metrics
cat /tmp/firecracker-metrics.json | jq '.vmm.memory_overhead'
```

#### Solutions:
```bash
# 1. Reduce microVM memory
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/machine-config' \
    -d '{"mem_size_mib": 256}'

# 2. Stop idle microVMs
curl --unix-socket $FC_SOCKET -X PATCH 'http://localhost/actions' \
    -d '{"action_type": "InstanceStop"}'

# 3. Use memory ballooning (advanced)
# Not covered in this guide
```

---

### Issue 5: Poor Performance

**Symptoms:**
- microVMs are slow
- High CPU usage
- Slow boot times

**Solutions:**

#### Check CPU template:
```bash
# Use optimized template
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/machine-config' \
    -d '{"cpu_template": "v2"}'
```

#### Reduce vCPU count:
```bash
# If CPU-bound, reduce vCPUs
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/machine-config' \
    -d '{"vcpu_count": 1}'
```

#### Enable rate limiting:
```bash
# Prevent one microVM from hogging resources
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/network-interfaces/eth0' \
    -d '{
        "iface_id": "eth0",
        "rate_limiter": {
            "bandwidth": {"size": 100000000, "refill_time": 1000}
        }
    }'
```

#### Check host resources:
```bash
# Check if host is overloaded
htop
iostat -x 1
vmstat 1

# Solution: Add more hosts or reduce load
```

---

### Issue 6: "Cannot allocate memory"

**Symptoms:**
```
Error: Cannot allocate memory
```

**Solutions:**

```bash
# 1. Check available memory
free -h

# 2. Reduce swap usage
sudo swapoff -a
sudo swapon -a

# 3. Kill idle processes
sudo killall firecracker

# 4. Reduce microVM memory
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/machine-config' \
    -d '{"mem_size_mib": 128}'

# 5. Check for memory leaks
# Monitor over time:
watch -n 1 'ps aux | grep firecracker'
```

---

### Issue 7: Permission Denied Errors

**Symptoms:**
```
Error: Permission denied
```

**Solutions:**

#### Check file permissions:
```bash
# Check kernel and rootfs files
ls -l xenial-generic-linux-4.14.gz
ls -l xenial-rootfs.ext4

# Make readable:
chmod 644 xenial-generic-linux-4.14.gz
chmod 644 xenial-rootfs.ext4
```

#### Check /dev/kvm:
```bash
ls -l /dev/kvm
# Should be crw-rw----+ 1 root kvm

# Add user to kvm group:
sudo usermod -a -G kvm $USER
newgrp kvm
```

#### Check socket permissions:
```bash
ls -l /tmp/firecracker.sock

# Fix permissions:
sudo chmod 666 /tmp/firecracker.sock
```

---

### Issue 8: Build Failures

**Symptoms:**
```
Error: Build failed
```

**Solutions:**

```bash
# 1. Check Rust version
rustc --version
# Should be 1.70 or later

# Update Rust:
rustup update

# 2. Clean and rebuild
cd firecracker
cargo clean
./tools/devtool build

# 3. Check dependencies
sudo apt-get install build-essential pkg-config libssl-dev

# 4. Check disk space
df -h
# Need at least 2 GB free for building
```

---

## 🩺 Diagnostic Tools

### firecracker-check

Create a diagnostic script:

```bash
#!/bin/bash
echo "=== Firecracker Diagnostic ==="
echo ""

echo "1. Firecracker Process:"
if pgrep -x firecracker > /dev/null; then
    echo "   ✅ Running"
    pgrep -x firecracker | head -5
else
    echo "   ❌ Not running"
fi
echo ""

echo "2. API Socket:"
if [ -S /tmp/firecracker.sock ]; then
    echo "   ✅ Exists at /tmp/firecracker.sock"
    ls -l /tmp/firecracker.sock
elif [ -S /srv/jailer/firecracker/*/root/run/api.socket ]; then
    echo "   ✅ Exists in jailer path"
    ls -l /srv/jailer/firecracker/*/root/run/api.socket
else
    echo "   ❌ Not found"
fi
echo ""

echo "3. KVM Device:"
if [ -e /dev/kvm ]; then
    echo "   ✅ /dev/kvm exists"
    ls -l /dev/kvm
    if [ -r /dev/kvm ] && [ -w /dev/kvm ]; then
        echo "   ✅ Readable and writable"
    else
        echo "   ⚠️  Permission issues"
    fi
else
    echo "   ❌ /dev/kvm not found"
fi
echo ""

echo "4. Required Files:"
for file in \
    ~/firecracker-demo/xenial-generic-linux-4.14.gz \
    ~/firecracker-demo/xenial-rootfs.ext4
do
    if [ -f "$file" ]; then
        echo "   ✅ $file"
    else
        echo "   ❌ $file (not found)"
    fi
done
echo ""

echo "5. System Resources:"
echo "   Memory: $(free -h | grep Mem | awk '{print $3 "/" $2}')"
echo "   Disk: $(df -h . | awk 'NR==2 {print $3 "/" $2 " (" $5 " used)"}')"
echo ""

echo "=== Diagnostic Complete ==="
```

Save as `firecracker-check.sh` and run:
```bash
chmod +x firecracker-check.sh
./firecracker-check.sh
```

### Get Full Diagnostics

```bash
# Collect all diagnostic info
{
  echo "=== System Info ==="
  uname -a
  cat /etc/os-release
  echo ""

  echo "=== Resources ==="
  free -h
  df -h
  echo ""

  echo "=== Firecracker Process ==="
  ps aux | grep firecracker
  echo ""

  echo "=== KVM Module ==="
  lsmod | grep kvm
  echo ""

  echo "=== Recent Logs ==="
  tail -n 50 /var/log/firecracker.log
  echo ""

  echo "=== Metrics ==="
  cat /tmp/firecracker-metrics.json | jq .
} > firecracker-diagnostics.txt
```

---

## 🆘 Getting Help

### Community Resources

1. **Firecracker Slack**
   - Join: https://firecracker-microvm.github.io/
   - Channel: #firecracker-users
   - Active community, quick responses

2. **GitHub Issues**
   - https://github.com/firecracker-microvm/firecracker/issues
   - Search existing issues first
   - Create new issue with details

3. **Stack Overflow**
   - Tag: `firecracker`
   - Check for similar questions

4. **Official Documentation**
   - https://firecracker-microvm.github.io/
   - Comprehensive reference

### How to Ask for Help

When asking for help, include:

1. **What you were trying to do**
   ```bash
   I was trying to start a microVM with 2 vCPUs and 512 MB RAM
   ```

2. **What you expected to happen**
   ```bash
   Expected: microVM starts in < 1 second
   ```

3. **What actually happened**
   ```bash
   Actual: Got "Failed to connect" error
   ```

4. **Steps to reproduce**
   ```bash
   1. Started Firecracker
   2. Sent API request to configure boot source
   3. Sent API request to start microVM
   ```

5. **Error messages**
   ```bash
   Full error output:
   curl: (7) Failed to connect to localhost port 80
   ```

6. **Your environment**
   ```bash
   OS: Ubuntu 22.04
   Kernel: 5.15.0
   Firecracker: v1.7.0
   ```

### Example Good Question:

```markdown
## Problem: Can't start microVM on Ubuntu 22.04

### What I'm trying to do:
Start a Firecracker microVM following the Quick Start guide

### Expected behavior:
microVM boots and I can SSH in

### Actual behavior:
Getting "Failed to connect to Firecracker API" error

### Commands I ran:
```bash
firecracker --api-sock /tmp/firecracker.sock &
curl --unix-socket /tmp/firecracker.sock http://localhost/info
```

### Error message:
```
curl: (7) Failed to connect to localhost port 80: Connection refused
```

### Environment:
- OS: Ubuntu 22.04 LTS
- Kernel: 5.15.0-72-generic
- Firecracker: v1.7.0
- /dev/kvm exists and is readable

### What I've tried:
- Checked that Firecracker is running (it is)
- Verified socket path (correct)
- Checked permissions (seem okay)

Still getting the error. Any ideas?
```

---

## 📊 Troubleshooting Checklist

Use this checklist when troubleshooting:

### Pre-flight Checks
- [ ] Firecracker process running
- [ ] API socket exists
- [ ] /dev/kvm accessible
- [ ] Kernel file exists
- [ ] Root filesystem exists
- [ ] Sufficient memory available
- [ ] Sufficient disk space

### Configuration Checks
- [ ] Boot source configured
- [ ] Root filesystem configured
- [ ] Machine configuration set
- [ ] Network interface added (if needed)
- [ ] Rate limiters configured (if needed)

### Runtime Checks
- [ ] API requests succeed
- [ ] microVM starts
- [ ] Console shows boot messages
- [ ] Network accessible (if configured)
- [ ] Can connect via SSH/console

### Performance Checks
- [ ] Memory usage reasonable
- [ ] CPU usage normal
- [ ] Boot time < 1 second
- [ ] Network performance good
- [ ] No excessive errors in logs

---

## ✅ Summary

### Key Troubleshooting Steps:

1. 📋 **Check logs first** - Always start here
2. 🔍 **Run diagnostics** - Use the diagnostic script
3. 🧪 **Test components** - Check each part individually
4. 📊 **Monitor resources** - Ensure sufficient memory/CPU
5. 🆘 **Ask for help** - Community is friendly

### Remember:

- Most issues are configuration errors
- Check permissions on files and devices
- Verify paths are correct
- Ensure sufficient resources
- Don't panic - there's usually a simple fix!

---

## 🎉 Congratulations!

You've completed the Firecracker for Babies guide! 🎊

You now know:
- ✅ What Firecracker is and why it exists
- ✅ How to install and run it
- ✅ How it works under the hood
- ✅ Advanced features and configurations
- ✅ Best practices for production use
- ✅ How to troubleshoot common issues

### What's Next?

1. **Build something cool!**
   - Create your own serverless platform
   - Build a FaaS system
   - Experiment with microVMs

2. **Contribute to Firecracker!**
   - Join the community
   - Fix bugs
   - Add features
   - Write documentation

3. **Keep learning!**
   - Read the official docs
   - Study the source code
   - Follow the blog
   - Attend conferences

---

## 📚 Additional Resources

- [Official Firecracker Documentation](https://firecracker-microvm.github.io/)
- [Firecracker GitHub](https://github.com/firecracker-microvm/firecracker)
- [Firecracker Blog](https://firecracker-microvm.github.io/blog/)
- [AWS Lambda Blog](https://aws.amazon.com/blogs/compute/category/lambda/)
- [rust-vmm Community](https://github.com/rust-vmm)

---

**Happy hacking with Firecracker!** 🚀🔥

*Found this guide helpful? Star it on GitHub and share it with your friends!*
