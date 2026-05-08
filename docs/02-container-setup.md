# Container Setup

This guide covers creating and configuring the 3X-UI container on MikroTik RouterOS.

---

## 📋 Prerequisites

Before starting:
- ✅ Docker image uploaded to MikroTik (see [Preparation](01-preparation.md))
- ✅ SSH access to your MikroTik
- ✅ Basic knowledge of RouterOS CLI
- ✅ At least 300MB free disk space

Check disk space:
```routeros
/disk print
```

---

## 🌐 Network Setup

### Step 1: Create Virtual Ethernet Interface

```routeros
/interface veth add \
  name=3x-ui \
  address=10.10.4.4/24 \
  gateway=10.10.4.1
```

**Parameters explained:**
- `name=3x-ui` - Interface name (can be anything)
- `address=10.10.4.4/24` - Container IP address
- `gateway=10.10.4.1` - Gateway IP (will be on bridge)

**Why this network:**
- `10.10.4.0/24` - Private network for container
- Avoids conflicts with common home networks (192.168.x.x)
- Can be changed if conflicts with your setup

### Step 2: Create Bridge

```routeros
/interface bridge add name=bridge-3x-ui
```

### Step 3: Add veth to Bridge

```routeros
/interface bridge port add \
  bridge=bridge-3x-ui \
  interface=3x-ui
```

### Step 4: Assign IP to Bridge

```routeros
/ip address add \
  address=10.10.4.1/24 \
  interface=bridge-3x-ui
```

### Verify Network Setup

```routeros
# Check interfaces
/interface print where name~"3x-ui"

# Check bridge
/interface bridge print where name~"3x-ui"

# Check IP address
/ip address print where interface="bridge-3x-ui"
```

Expected output:
```
Flags: D - DYNAMIC; X - DISABLED, I - INACTIVE; H - SLAVE; R - RUNNING
NAME         TYPE   MTU   L2MTU  MAX-L2MTU  MAC-ADDRESS
3x-ui        veth   1500        65535      XX:XX:XX:XX:XX:XX
bridge-3x-ui bridge 1500  1500   65535      XX:XX:XX:XX:XX:XX

ADDRESS            NETWORK       INTERFACE
10.10.4.1/24       10.10.4.0     bridge-3x-ui
```

---

## 📦 Container Creation

### Step 1: Add Container

```routeros
/container add \
  file=3x-ui-amd64.tar \
  interface=3x-ui \
  root-dir=disk1/3xui \
  dns=8.8.8.8,1.1.1.1 \
  hostname=3xui-vpn \
  start-on-boot=yes \
  logging=yes \
  comment="3X-UI VPN Panel"
```

**Parameters explained:**
- `file=` - Docker image file name
- `interface=` - Virtual ethernet interface to attach
- `root-dir=` - Persistent storage directory
- `dns=` - DNS servers for container
- `hostname=` - Container hostname
- `start-on-boot=yes` - Auto-start after reboot
- `logging=yes` - Enable container logs
- `comment=` - Optional description

**Important**: `root-dir` path depends on your storage:
- CHR with single disk: `disk1/3xui`
- Router with USB: `usb1/3xui`
- Router with SD card: `sd1/3xui`

Check available disks:
```routeros
/disk print
```

### Step 2: Start Container

```routeros
/container start [find interface~"3x-ui"]
```

### Step 3: Wait for Startup

Container needs ~30 seconds to fully start. Monitor progress:

```routeros
# Watch container status
/container print detail

# Watch logs
/log print follow where topics~"container"
```

Expected log output:
```
container container extracted container 3x-ui-amd64.tar
container started container
```

### Step 4: Verify Container is Running

```routeros
/container print
```

Expected output:
```
# NAME         TAG  OS     ARCH   INTERFACE  ROOT-DIR        STATUS
0 3x-ui-amd64      linux  amd64  3x-ui      disk1/3xui      running
```

**Status values:**
- `running` - ✅ Container is working
- `extracting` - ⏳ Still extracting image
- `stopped` - ❌ Container stopped (check logs)
- `error` - ❌ Error occurred (check logs)

---

## 🔧 Container Configuration

### Access Container Shell

```routeros
/container shell [find interface~"3x-ui"]
```

You're now inside the container's shell!

### Verify Container Basics

```bash
# Check architecture (must be x86_64)
uname -m
# Output: x86_64

# Check 3X-UI process
ps aux | grep x-ui
# Should see x-ui process running

# Check internet connectivity
ping -c 3 8.8.8.8
ping -c 3 google.com
```

### Check 3X-UI Status

```bash
# Check if x-ui is listening
netstat -tulpn | grep x-ui

# Expected output:
# tcp    0.0.0.0:2053   LISTEN   1/x-ui
# tcp    :::2096        LISTEN   1/x-ui
```

**Default ports:**
- `2053` - Web panel (HTTP)
- `2096` - Subscription service

### View 3X-UI Logs

```bash
# View logs
tail -f /var/log/x-ui/3xui.log

# Press Ctrl+C to stop
```

### Exit Container Shell

```bash
exit
```

---

## 🌍 Enable Internet Access for Container

Container needs internet to:
- Update Xray core
- Download geo databases
- Proxy VPN traffic

### Configure NAT

```routeros
/ip firewall nat add \
  chain=srcnat \
  action=masquerade \
  src-address=10.10.4.0/24 \
  out-interface=ether1 \
  comment="Container NAT"
```

**Replace `ether1` with your WAN interface name!**

Find your WAN interface:
```routeros
/interface print where type="ether"
```

### Allow Container Forwarding

```routeros
/ip firewall filter add \
  chain=forward \
  action=accept \
  in-interface=bridge-3x-ui \
  place-before=0 \
  comment="Container forward IN"

/ip firewall filter add \
  chain=forward \
  action=accept \
  out-interface=bridge-3x-ui \
  connection-state=established,related \
  place-before=0 \
  comment="Container forward OUT"
```

**Why `place-before=0`?**
- Ensures these rules come BEFORE any drop rules
- Critical for proper forwarding

### Test Internet from Container

```routeros
/container shell [find interface~"3x-ui"]
```

Inside container:
```bash
# Test DNS resolution
nslookup google.com

# Test connectivity
ping -c 3 8.8.8.8
ping -c 3 google.com

# Test HTTPS
wget -qO- https://api.ipify.org

exit
```

If ping works, internet is configured correctly! ✅

---

## 🔍 Container Management Commands

### View Container Status

```routeros
# List all containers
/container print

# Detailed info
/container print detail

# Show specific container
/container print where interface~"3x-ui"
```

### Start/Stop Container

```routeros
# Stop
/container stop [find interface~"3x-ui"]

# Start
/container start [find interface~"3x-ui"]

# Restart
/container stop [find interface~"3x-ui"]
:delay 3
/container start [find interface~"3x-ui"]
```

### Remove Container (Preserves Data)

```routeros
# Stop first
/container stop [find interface~"3x-ui"]

# Remove container (data in root-dir is preserved!)
/container remove [find interface~"3x-ui"]

# Data remains in disk1/3xui for re-creation
```

### Full Cleanup (Removes Everything)

```routeros
# Stop and remove container
/container stop [find interface~"3x-ui"]
/container remove [find interface~"3x-ui"]

# Remove network
/ip address remove [find interface="bridge-3x-ui"]
/interface bridge port remove [find interface="3x-ui"]
/interface bridge remove bridge-3x-ui
/interface veth remove 3x-ui

# Remove data directory
/file remove [find name="disk1/3xui" type="directory"]

# Remove Docker image
/file remove 3x-ui-amd64.tar
```

---

## 📊 Container Resource Usage

### Monitor Resources

```routeros
# System resources
/system resource print

# Container-specific (if supported)
/container print stats
```

### Expected Resource Usage

**Idle (no VPN connections):**
- CPU: <5%
- RAM: ~100-150MB

**Under load (10 active VPN connections):**
- CPU: 10-30%
- RAM: ~200-300MB

**MikroTik CHR Minimum Requirements:**
- 512MB RAM (1GB+ recommended)
- Dual-core CPU recommended
- 300MB disk space

---

## 🐛 Troubleshooting

### Container Won't Start

```routeros
# Check logs
/log print where topics~"container"

# Common causes:
# 1. Wrong architecture - see "Exec format error"
# 2. No disk space - run /disk print
# 3. Interface conflict - check /interface print
```

### "Exec format error"

**Cause**: Wrong architecture (ARM64 instead of AMD64)

**Solution**: Re-create image with `skopeo` (see [Preparation](01-preparation.md))

### Container Keeps Restarting

```routeros
# Check why it's restarting
/log print where message~"container"

# Access shell to debug
/container shell [find interface~"3x-ui"]
ps aux
dmesg | tail
```

### No Internet Inside Container

```bash
# Inside container shell:

# Test DNS
nslookup google.com
# If fails: DNS issue

# Test IP connectivity
ping 8.8.8.8
# If fails: routing issue

# Check routes
ip route
# Should see: default via 10.10.4.1
```

**Fix routing:**
```routeros
# Recreate veth with correct gateway
/interface veth remove 3x-ui
/interface veth add name=3x-ui address=10.10.4.4/24 gateway=10.10.4.1
```

### Container Uses Too Much RAM

If your MikroTik has limited RAM:

```routeros
# Check current RAM usage
/system resource print

# Reduce Xray log level (inside container):
/container shell [find interface~"3x-ui"]
# Edit config to reduce logging
```

---

## 🔐 Security Hardening

### Change Default Password (Critical!)

Access container and change 3X-UI password:

```routeros
/container shell [find interface~"3x-ui"]
```

```bash
# Change admin password
/app/x-ui setting -username admin -password YOUR_SECURE_PASSWORD

exit
```

### Restrict Panel Access

By default, panel is accessible from container network only. We'll configure external access in [Panel Access guide](05-panel-access.md) with proper security.

### Enable Logging

Already enabled with `logging=yes`, but verify:

```routeros
/log print where topics~"container"
```

---

## 📚 Next Steps

Container is now running! Next:

1. [Network Configuration →](03-network-config.md) - Configure firewall and NAT
2. [SSL Certificates →](04-ssl-certificates.md) - Set up HTTPS
3. [Panel Access →](05-panel-access.md) - Access the web panel

---

## 💡 Pro Tips

### Auto-restart on Failure

Container already set with `start-on-boot=yes`, which also auto-restarts on crashes.

### Backup Container Data

```routeros
# Backup container root directory
/file copy [find name="disk1/3xui" type="directory"] disk1/3xui-backup

# Or export via SCP (from your PC)
scp -r admin@MIKROTIK_IP:/disk1/3xui ./3xui-backup
```

### Multiple Containers

You can run multiple containers:

```routeros
# Container 1: 3X-UI on 10.10.4.4
# Container 2: Something else on 10.10.5.4

/interface veth add name=container2 address=10.10.5.4/24 gateway=10.10.5.1
# ... repeat bridge setup with different IPs
```

---

[⬅ Back: Preparation](01-preparation.md) | [Next: Network Configuration →](03-network-config.md)
