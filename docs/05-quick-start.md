# Chapter 5: Quick Start - Your First microVM! 🚀

> "The best way to learn is by doing" - Someone Awesome

## 🎯 What We'll Build

In this chapter, you'll:
1. ✅ Start your first Firecracker microVM
2. ✅ Connect to it via SSH
3. ✅ Run commands inside it
4. ✅ Understand what just happened

**Time required**: ~10 minutes  
**Difficulty**: Easy 🌟

---

## 📋 Pre-Flight Check

Make sure you have everything from the previous chapters:

```bash
# Quick check
cd ~/firecracker-demo
ls -lh xenial-generic-linux-4.14.gz xenial-rootfs.ext4
firecracker --version
```

If you see both files and the version info, you're good to go! 👍

---

## 🎬 Step 1: Start Firecracker

Firecracker works like this:
1. You start the Firecracker process
2. It listens for API requests
3. You tell it what to do via HTTP requests
4. It creates and manages microVMs

### Open Terminal 1 (The Firecracker Process)

We'll run Firecracker in the foreground so we can see what's happening:

```bash
cd ~/firecracker-demo

# Start Firecracker
firecracker --api-sock /tmp/firecracker.sock
```

**You should see:**
```
Starting Firecracker v1.7.0...
```

🎉 **Firecracker is now running!**

> **Note**: Keep this terminal open. Firecracker needs to keep running to manage the microVM.

---

## 🔌 Step 2: Configure the microVM

Now we need to tell Firecracker what our microVM should look like.

### Open Terminal 2 (The API Client)

We'll send HTTP requests to configure the microVM.

#### Set Up Environment Variables

This makes our commands easier to read:

```bash
# API socket location
export FC_SOCKET=/tmp/firecracker.sock

# Our files
export KERNEL=~/firecracker-demo/xenial-generic-linux-4.14.gz
export ROOTFS=~/firecracker-demo/xenial-rootfs.ext4
```

#### Configure Boot Source

Tell Firecracker which kernel to use:

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/boot-source' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d "{
        \"kernel_image_path\": \"$KERNEL\",
        \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
    }"
```

**Expected response:**
```json
HTTP/1.1 204 No Content
```

#### Configure Root Filesystem

Tell Firecracker which filesystem to use:

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/drives/rootfs' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d "{
        \"drive_id\": \"rootfs\",
        \"path_on_host\": \"$ROOTFS\",
        \"is_root_device\": true,
        \"is_read_only\": false
    }"
```

**Expected response:**
```json
HTTP/1.1 204 No Content
```

#### Configure Machine Resources

Set how many CPUs and how much memory:

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/machine-config' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '{
        "vcpu_count": 2,
        "mem_size_mib": 512,
        "ht_enabled": false
    }'
```

**Expected response:**
```json
HTTP/1.1 204 No Content
```

#### Configure Network (Optional)

Let's add a network interface so we can SSH in:

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/network-interfaces/eth0' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '{
        "iface_id": "eth0",
        "guest_mac": "02:FC:00:00:00:01",
        "host_dev_name": "tap0"
    }'
```

**Expected response:**
```json
HTTP/1.1 204 No Content
```

---

## 🎮 Step 3: Start the microVM

Now let's actually boot the microVM!

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/actions' \
    -H 'Accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '{
        "action_type": "InstanceStart"
    }'
```

**Expected response:**
```json
HTTP/1.1 204 No Content
```

🎉 **Your microVM is now booting!**

### Check Terminal 1

You should see boot messages scrolling by:

```
[    0.000000] Linux version 4.14.55
[    0.000000] Command line: console=ttyS0 reboot=k panic=1 pci=off
[    0.000000] KERNEL supported cpus:
...
[    2.123456] xenial-rootfs: Asking for cache data failed
[    2.234567]  VFS: Mounted root (ext4 filesystem).
...
Ubuntu 16.04.6 LTS ttyS0

localhost login:
```

---

## 🔐 Step 4: Connect to Your microVM

Let's SSH into your microVM!

### Set Up Network

First, configure the host network:

```bash
# In Terminal 2 (or a new terminal)
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
```

### SSH In

The default credentials are:
- **Username**: `ubuntu`
- **Password**: `ubuntu` (or no password)

```bash
# SSH into your microVM
ssh ubuntu@172.16.0.2
```

**Or use the console directly in Terminal 1:**

Just type `ubuntu` at the login prompt and press Enter.

---

## 🎉 Step 5: Explore Your microVM

Once you're logged in, try these commands:

```bash
# Who are you?
whoami

# What system is this?
uname -a

# How much memory?
free -h

# How many CPUs?
nproc

# Disk space?
df -h

# What processes are running?
ps aux

# Network interfaces?
ip addr

# Memory available?
cat /proc/meminfo | grep MemTotal
```

**Typical output:**
```
ubuntu
Linux localhost 4.14.55 #1 SMP Mon Jul 15 12:34:56 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
              total        used        free      shared  buff/cache   available
Mem:           488M         48M        368M        1.2M         71M        408M
Swap:            0B          0B          0B
2
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda        4.0G  720M  3.1G  19% /
...
MemTotal:         499796 kB
```

🎊 **Congratulations! You're inside a microVM!**

---

## 🧪 Try Something Cool

Let's run a simple web server:

```bash
# Inside the microVM
# Install Python
sudo apt-get update
sudo apt-get install -y python3

# Create a simple webpage
echo '<h1>Hello from Firecracker!</h1>' > /tmp/index.html

# Start a web server
cd /tmp
python3 -m http.server 8000
```

Now, from the host (Terminal 2):

```bash
# Access the web server
curl http://172.16.0.2:8000/index.html
```

You should see:
```html
<h1>Hello from Firecracker!</h1>
```

---

## 🛑 Step 6: Stop the microVM

When you're done, you can stop it:

**Option 1: From inside the microVM**
```bash
sudo poweroff
```

**Option 2: From the host (Terminal 2)**
```bash
curl --unix-socket $FC_SOCKET -X PATCH \
    'http://localhost/actions' \
    -H 'Content-Type: application/json' \
    -d '{
        "action_type": "InstanceStop"
    }'
```

**Option 3: Just kill Firecracker (Terminal 1)**
```bash
# Press Ctrl+C in Terminal 1
```

---

## 📊 What Just Happened? Let's Break It Down

Here's what we did, in plain English:

### 1. Started Firecracker
- Firecracker process began listening for API requests
- Like starting a server

### 2. Configured the microVM
- **Boot Source**: Told it which kernel to use (Linux 4.14)
- **Root Filesystem**: Told it which disk image to use (Ubuntu)
- **Machine Resources**: Gave it 2 CPUs and 512MB RAM
- **Network**: Added a network interface

### 3. Started the microVM
- Firecracker used KVM to create a virtual machine
- Loaded the kernel
- Booted the Ubuntu filesystem

### 4. Connected via SSH
- The microVM got its own IP address
- We connected like any other Linux machine

### 5. Ran commands
- It's a real Linux system!
- Just very small and fast

---

## 🎯 Key Takeaways

### What You Learned:
1. ✅ Firecracker uses a **RESTful API** for control
2. ✅ microVMs boot in **milliseconds** (not minutes!)
3. ✅ Each microVM is a **complete Linux system**
4. ✅ Very **lightweight** (only 512MB RAM, 2 vCPUs)
5. ✅ Uses **real virtualization** (KVM) for security

### Comparison:
| Aspect | Regular VM | Your microVM |
|--------|-----------|--------------|
| Boot Time | 1-5 minutes | ~1 second |
| RAM Used | 1-2 GB | 512 MB |
| Setup | Complex | Simple API |
| Security | High | High |
| Use Case | Heavy apps | Functions, containers |

---

## 🚀 What's Next?

Now that you've run your first microVM:

1. **Try changing the configuration**:
   - More CPUs (vcpu_count)
   - More memory (mem_size_mib)
   - Different kernel or filesystem

2. **Run multiple microVMs**:
   - Start another Firecracker process
   - Use a different socket name
   - See how many you can run!

3. **Learn the architecture**:
   - How does Firecracker work?
   - What is KVM?
   - What makes it so fast?

[Continue to Chapter 6: Architecture →](./06-architecture.md)

---

## 💡 Tips and Tricks

### Save Your Configuration

Instead of typing all those curl commands, save them to a script:

```bash
cat > ~/firecracker-demo/start-microvm.sh << 'EOF'
#!/bin/bash
FC_SOCKET=/tmp/firecracker.sock
KERNEL=~/firecracker-demo/xenial-generic-linux-4.14.gz
ROOTFS=~/firecracker-demo/xenial-rootfs.ext4

# Configure boot source
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/boot-source' \
    -H 'Content-Type: application/json' \
    -d "{\"kernel_image_path\": \"$KERNEL\", \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"}"

# Configure rootfs
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/drives/rootfs' \
    -H 'Content-Type: application/json' \
    -d "{\"drive_id\": \"rootfs\", \"path_on_host\": \"$ROOTFS\", \"is_root_device\": true, \"is_read_only\": false}"

# Configure machine
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/machine-config' \
    -H 'Content-Type: application/json' \
    -d '{"vcpu_count": 2, "mem_size_mib": 512, "ht_enabled": false}'

# Start the microVM
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/actions' \
    -H 'Content-Type: application/json' \
    -d '{"action_type": "InstanceStart"}'
EOF

chmod +x ~/firecracker-demo/start-microvm.sh
```

Now you can just run:
```bash
~/firecracker-demo/start-microvm.sh
```

---

## 🆘 Common Issues

### "Connection refused" on curl
**Solution**: Make sure Firecracker is still running in Terminal 1

### "Permission denied" on SSH
**Solution**: Try with password `ubuntu` or check your network setup

### microVM doesn't get IP
**Solution**: Make sure you configured the tap0 interface:
```bash
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
```

### Can't ping the microVM
**Solution**: Check firewall rules and make sure tap0 is up

---

**You did it!** You've successfully run your first Firecracker microVM! 🎊🎉

*Ready to learn how it all works? [Continue to Chapter 6 →](./06-architecture.md)*
