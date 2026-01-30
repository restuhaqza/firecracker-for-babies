# Chapter 8: Advanced Topics - Going Deeper 🚀

> "The expert in anything was once a beginner" - Helen Hayes

## 🎯 Overview

Now that you understand the basics, let's explore more advanced features and use cases.

---

## 🔥 Advanced Configuration

### Multiple Network Interfaces

Your microVM can have multiple network interfaces:

```bash
# First interface (eth0)
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/network-interfaces/eth0' \
    -H 'Content-Type: application/json' \
    -d '{
        "iface_id": "eth0",
        "guest_mac": "02:FC:00:00:00:01",
        "host_dev_name": "tap0"
    }'

# Second interface (eth1)
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/network-interfaces/eth1' \
    -H 'Content-Type: application/json' \
    -d '{
        "iface_id": "eth1",
        "guest_mac": "02:FC:00:00:00:02",
        "host_dev_name": "tap1"
    }'
```

**Use Cases:**
- Separate networks for different services
- Management network vs. data network
- Multi-homed applications

### Multiple Block Devices

Add additional disks to your microVM:

```bash
# Root filesystem
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/drives/rootfs' \
    -H 'Content-Type: application/json' \
    -d '{
        "drive_id": "rootfs",
        "path_on_host": "./rootfs.ext4",
        "is_root_device": true,
        "is_read_only": false
    }'

# Data disk
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/drives/data' \
    -H 'Content-Type: application/json' \
    -d '{
        "drive_id": "data",
        "path_on_host": "./data_disk.img",
        "is_root_device": false,
        "is_read_only": false
    }'
```

**Inside the guest:**
```bash
# The data disk will appear as /dev/vdb
lsblk
# Format and mount
sudo mkfs.ext4 /dev/vdb
sudo mount /dev/vdb /mnt/data
```

### vCPU Pinning

Pin vCPUs to specific host CPUs for performance:

```bash
# Configure machine with CPU affinity
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/machine-config' \
    -H 'Content-Type: application/json' \
    -d '{
        "vcpu_count": 2,
        "mem_size_mib": 512,
        "ht_enabled": false,
        "cpu_template": "v2"
    }'
```

**CPU Templates:**
- `v2`: Optimize for predictable performance (recommended)
- `v3`: Optimize for performance (may have slight variations)

### Memory Configuration

Fine-tune memory settings:

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/machine-config' \
    -H 'Content-Type: application/json' \
    -d '{
        "vcpu_count": 2,
        "mem_size_mib": 512,
        "ht_enabled": false
    }'
```

**Memory Size Options:**
- Minimum: ~128 MB
- Typical: 256 MB - 2 GB
- Maximum: Limited by host

---

## 🔧 Using the Jailer

The Jailer provides additional security layers.

### Why Use the Jailer?

Without Jailer:
```
Host
 └─ Firecracker Process
     └─ microVM
```

With Jailer:
```
Host
 └─ Jailer Process (root)
     ├─ chroot directory
     ├─ cgroup limits
     ├─ seccomp filters
     └─ Firecracker Process (non-root)
         └─ microVM
```

### Basic Jailer Usage

```bash
# Start microVM with jailer
jailer \
    --id=myvm \
    --exec-file=/usr/local/bin/firecracker \
    --node 0 \
    --uid 1000 \
    --gid 1000 \
    --chroot-base-dir=/srv/jailer
```

**Parameters:**
- `--id`: Unique identifier for this microVM
- `--exec-file`: Path to Firecracker binary
- --node: NUMA node (usually 0)
- `--uid`: User ID to run Firecracker as
- `--gid`: Group ID to run Firecracker as
- `--chroot-base-dir`: Base directory for chroot

### Jailer Directory Structure

```
/srv/jailer/
└── firecracker/
    └── myvm/
        ├── root/               # chroot directory
        │   ├── dev/            # /dev/kvm, etc.
        │   ├── run/            # API socket
        │   └── api.socket      # Unix socket
        └── firecracker         # Binary (hardlink)
```

### Jailer with API Calls

When using the Jailer, the API socket location changes:

```bash
# Without jailer
export FC_SOCKET=/tmp/firecracker.sock

# With jailer
export FC_SOCKET=/srv/jailer/firecracker/myvm/root/run/api.socket
```

---

## 🌐 Advanced Networking

### Bridge Networking

Connect multiple microVMs on the same network:

```bash
# Create bridge on host
sudo ip link add name br0 type bridge
sudo ip addr add 192.168.1.1/24 dev br0
sudo ip link set br0 up

# Connect tap0 to bridge
sudo ip link set tap0 master br0
sudo ip link set tap0 up
```

```
Host (192.168.1.1)
│
├─ br0 (bridge)
│   │
│   ├─ tap0 → microVM #1 (192.168.1.2)
│   ├─ tap1 → microVM #2 (192.168.1.3)
│   └─ tap2 → microVM #3 (192.168.1.4)
```

### Network Namespaces

Isolate microVM networks completely:

```bash
# Create network namespace
sudo ip netns add microvm-net

# Create veth pair
sudo ip link add veth0 type veth peer name veth1

# Move veth1 to namespace
sudo ip link set veth1 netns microvm-net

# Configure host side
sudo ip addr add 10.0.0.1/24 dev veth0
sudo ip link set veth0 up

# Configure namespace side
sudo ip netns exec microvm-net ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec microvm-net ip link set veth1 up

# Connect tap0 to namespace
sudo ip link set tap0 netns microvm-net
```

### Rate Limiting in Detail

Configure precise network rate limits:

```bash
curl --unix-socket $FC_SOCKET -i \
    -X PUT 'http://localhost/network-interfaces/eth0' \
    -H 'Content-Type: application/json' \
    -d '{
        "iface_id": "eth0",
        "guest_mac": "02:FC:00:00:00:01",
        "host_dev_name": "tap0",
        "rate_limiter": {
            "bandwidth": {
                "size": 100000000,
                "one_time_burst": 10000000,
                "refill_time": 1000
            }
        }
    }'
```

**Parameters:**
- `size`: Bytes per second (100 MB/s in this example)
- `one_time_burst`: Initial burst allowance
- `refill_time`: Refill interval in milliseconds

---

## 💾 Advanced Storage

### Creating Custom Root Filesystems

#### Method 1: Using Docker

```bash
# Export Docker container to filesystem
docker export $(docker create ubuntu:22.04) > ubuntu-rootfs.tar

# Convert to ext4
mkdir rootfs
sudo tar -xf ubuntu-rootfs.tar -C rootfs

# Create ext4 image
dd if=/dev/zero of=ubuntu-rootfs.ext4 bs=1M count=1024
mkfs.ext4 ubuntu-rootfs.ext4

# Mount and copy
sudo mount ubuntu-rootfs.ext4 /mnt
sudo cp -r rootfs/* /mnt/
sudo umount /mnt
```

#### Method 2: Using debootstrap

```bash
# Install debootstrap
sudo apt-get install debootstrap

# Create filesystem
dd if=/dev/zero of=ubuntu-rootfs.ext4 bs=1M count=2048
mkfs.ext4 ubuntu-rootfs.ext4

# Mount and bootstrap
sudo mount ubuntu-rootfs.ext4 /mnt
sudo debootstrap --arch=amd64 jammy /mnt http://archive.ubuntu.com/ubuntu/

# Clean up
sudo umount /mnt
```

### Optimizing Root Filesystems

Reduce filesystem size for faster startup:

```bash
# Strip unnecessary files
sudo rm -rf /mnt/var/cache/apt/archives/*.deb
sudo rm -rf /mnt/usr/share/man
sudo rm -rf /mnt/usr/share/doc

# Zero out free space (better compression)
sudo dd if=/dev/zero of=/mnt/zero.dat
sudo rm /mnt/zero.dat
```

### Snapshots and Clones

Use Copy-on-Write (CoW) for faster instance creation:

```bash
# Create base image
qemu-img create -f qcow2 base.qcow2 4G

# Create snapshot (instant!)
qemu-img create -f qcow2 -b base.qcow2 snapshot1.qcow2
qemu-img create -f qcow2 -b base.qcow2 snapshot2.qcow2

# Convert to raw for Firecracker
qemu-img convert -O raw snapshot1.qcow2 snapshot1.img
```

---

## 🔍 Debugging and Monitoring

### Enabling Logs

Firecracker supports detailed logging:

```bash
# Start with logging
firecracker \
    --api-sock /tmp/firecracker.sock \
    --log-path /tmp/firecracker.log \
    --level Debug \
    --metrics-path /tmp/firecracker-metrics.json
```

**Log Levels:**
- `Error`: Only errors
- `Warning`: Warnings and errors
- `Info`: Informational messages
- `Debug`: Detailed debugging info

### Reading Metrics

Firecracker exposes metrics in JSON format:

```bash
# Watch metrics in real-time
watch -n 1 'cat /tmp/firecracker-metrics.json | jq'
```

**Available Metrics:**
```json
{
  "utc_timestamp_ms": 1643723400000,
  "api_server": {
    "process_startup_time_us": 1234,
    "process_startup_time_cpu_us": 567
  },
  "vmm": {
    "memory_overhead": 5242880,
    "vcpus_count": 2
  },
  "balloon": {
    "activate_count": 0,
    "activate_fail_count": 0
  },
  "block": {
    "read_count": 100,
    "write_count": 50,
    "bytes_read": 1048576,
    "bytes_written": 524288
  },
  "net": {
    "rx_packets": 1000,
    "tx_packets": 500,
    "rx_bytes": 1048576,
    "tx_bytes": 524288,
    "rx_rate_limiter_throttled": 0,
    "tx_rate_limiter_throttled": 0
  }
}
```

### Attaching GDB

Debug Firecracker itself:

```bash
# Start Firecracker with GDB
gdb --args firecracker --api-sock /tmp/firecracker.sock

# Set breakpoints
(gdb) break main
(gdb) run

# When running, attach to process
(gdb) attach <pid>
```

---

## 🚀 Performance Optimization

### Kernel Configuration

Use optimized kernels for microVMs:

```bash
# Download optimized kernel
wget https://s3.amazonaws.com/spec.ccfc.min/img/quickstart_guide/xenial-generic-linux-4.14.gz

# Or build your own with minimal config:
# - Disable unnecessary drivers
# - Enable virtio drivers
# - Disable debug features
# - Optimize for size
```

### Memory Overcommit

Overcommit memory for higher density:

```bash
# Configure Firecracker to allow overcommit
# This allows you to allocate more virtual memory than physical RAM
# Only works if applications don't actually use all memory
```

**Trade-offs:**
- ✅ Higher density (more microVMs)
- ⚠️ Risk of OOM if all microVMs use full memory
- ⚠️ Requires careful monitoring

### CPU Overcommit

Run more vCPUs than physical CPUs:

```bash
# Host has 4 CPUs, run 8 microVMs with 2 vCPUs each
# Total: 16 vCPUs on 4 physical CPUs

# Works if microVMs are not CPU-intensive
# Great for I/O-bound workloads
```

### Fast Reboot

Reuse microVMs without full reboot:

```bash
# Instead of full stop/start, just restart the application inside
# This is what AWS Lambda does for "warm" functions
```

---

## 🎯 Production Considerations

### Resource Planning

Calculate how many microVMs you can run:

```
Host: 16 GB RAM, 8 CPUs

Each microVM: 512 MB RAM, 2 vCPUs

Theoretical maximum:
- Memory: 16 GB / 512 MB = 32 microVMs
- CPU: 8 / 2 = 4 microVMs (if CPU-bound)
- Realistic: ~20-25 microVMs (mixed workload)
```

### Monitoring Strategy

What to monitor:

1. **Host Level:**
   - CPU usage
   - Memory usage
   - Disk I/O
   - Network bandwidth

2. **Firecracker Level:**
   - Number of running microVMs
   - API response times
   - Error rates
   - Resource limits

3. **microVM Level:**
   - Application logs
   - Performance metrics
   - Resource usage per VM

### High Availability

Design for failure:

```bash
# Run Firecracker on multiple hosts
Host 1: microVMs 1-10
Host 2: microVMs 11-20
Host 3: microVMs 21-30

# If Host 2 fails:
# - Detect failure (heartbeat)
# - Restart microVMs 11-20 on Host 1 and 3
# - Update load balancer
```

---

## 🧪 Advanced Use Cases

### MicroVM Templates

Create reusable configurations:

```bash
#!/bin/bash
# Template script

MICROVM_ID=$1
FC_SOCKET=/tmp/firecracker-${MICROVM_ID}.sock

# Start Firecracker
firecracker --api-sock $FC_SOCKET &
sleep 1

# Configure from template
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/boot-source' \
    -d @templates/boot-source.json

curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/drives/rootfs' \
    -d @templates/drive-rootfs.json

curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/machine-config' \
    -d @templates/machine-config.json

# Start
curl --unix-socket $FC_SOCKET -X PUT 'http://localhost/actions' \
    -d '{"action_type": "InstanceStart"}'
```

### Batch Creation

Create multiple microVMs at once:

```bash
#!/bin/bash
for i in {1..10}; do
    ./create-microvm.sh $i &
done

wait
echo "Created 10 microVMs"
```

### API Gateway Integration

Build your own FaaS platform:

```python
from flask import Flask, request
import subprocess
import requests

app = Flask(__name__)

@app.route('/invoke/<function_id>')
def invoke(function_id):
    # Check for warm microVM
    if has_warm_microvm(function_id):
        microvm = get_warm_microvm(function_id)
    else:
        # Create new microVM
        microvm = create_microvm(function_id)
    
    # Send request to microVM
    response = send_to_microvm(microvm, request.data)
    
    return response
```

---

## 📊 Advanced Comparison

### Firecracker vs. Alternatives

| Feature | Firecracker | QEMU | Kata Containers |
|---------|-------------|-----|-----------------|
| Startup Time | < 125ms | 1-5s | ~1s |
| Memory Overhead | ~5 MB | ~100 MB | ~50 MB |
| Device Model | 5 devices | 20+ devices | Configurable |
| API | RESTful | QMP / CLI | CLI |
| Production Use | AWS Lambda | KVM | OpenShift |
| Best For | Serverless | General VMs | Secure Containers |

---

## ✅ Summary

### What You Learned:

1. ✅ Advanced configuration options
2. ✅ Using the Jailer for security
3. ✅ Advanced networking setups
4. ✅ Custom filesystem creation
5. ✅ Debugging and monitoring
6. ✅ Performance optimization
7. ✅ Production considerations
8. ✅ Advanced use cases

### Next Steps:

- [ ] Experiment with different configurations
- [ ] Build your own microVM template
- [ ] Set up monitoring
- [ ] Try production deployment
- [ ] Contribute to Firecracker!

---

## 🚀 What's Next?

Now that you're an advanced user, let's look at best practices:

[Continue to Chapter 9: Best Practices →](./09-best-practices.md)

---

*Ready for production? Read the [official Firecracker production guide](https://firecracker-microvm.github.io/) for more details*
