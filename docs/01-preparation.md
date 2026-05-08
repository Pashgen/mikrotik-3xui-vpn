# Docker Image Preparation

This guide covers the critical first step: preparing the 3X-UI Docker image with the correct architecture for MikroTik x86_64 hardware.

---

## ⚠️ Architecture Warning

**CRITICAL**: MikroTik CHR runs on **x86_64 (AMD64)** architecture. If you prepare the Docker image incorrectly, you'll get:

```
Error: Exec format error
```

This happens when:
- Using `docker save` on M1/M2 Mac (creates ARM64 image)
- Pulling without architecture specification
- Using Docker Desktop without platform flag

---

## 📋 Prerequisites

### Required Tools

**Option 1: Skopeo (Recommended)**
- ✅ Guarantees correct architecture
- ✅ Works on all platforms
- ✅ No Docker daemon needed
- ✅ Explicit architecture control

**Option 2: Docker with Platform Flag**
- ⚠️ Requires Docker daemon
- ⚠️ Easy to forget `--platform`
- ✅ More familiar to Docker users

---

## 🛠️ Method 1: Using Skopeo (Recommended)

### Install Skopeo

**macOS:**
```bash
brew install skopeo
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install skopeo
```

**Linux (Fedora/RHEL):**
```bash
sudo dnf install skopeo
```

**Windows (WSL2):**
```bash
# In WSL2 Ubuntu
sudo apt update
sudo apt install skopeo
```

### Pull and Convert Image

```bash
# Navigate to your working directory
cd ~/Desktop

# Pull 3X-UI image with explicit AMD64 architecture
skopeo copy --override-arch amd64 --override-os linux \
  docker://ghcr.io/mhsanaei/3x-ui:latest \
  docker-archive:3x-ui-amd64.tar:ghcr.io/mhsanaei/3x-ui:latest
```

**What this does:**
- `--override-arch amd64` - Forces AMD64/x86_64 architecture
- `--override-os linux` - Ensures Linux OS
- `docker://` - Source is Docker registry
- `docker-archive:` - Destination is TAR file
- Image name preserved for MikroTik compatibility

### Verify Image

```bash
# Check file size (should be ~100-150MB)
ls -lh 3x-ui-amd64.tar

# Extract and verify architecture (optional)
mkdir temp-verify
tar -xf 3x-ui-amd64.tar -C temp-verify
cat temp-verify/manifest.json | grep -i architecture
# Should show: "architecture": "amd64"

# Cleanup
rm -rf temp-verify
```

---

## 🐳 Method 2: Using Docker

**⚠️ Warning**: This method requires careful attention to the `--platform` flag!

### Pull with Correct Platform

```bash
# Pull with explicit platform
docker pull --platform linux/amd64 ghcr.io/mhsanaei/3x-ui:latest

# Save to tar
docker save -o 3x-ui-amd64.tar ghcr.io/mhsanaei/3x-ui:latest
```

### On M1/M2 Mac - Extra Verification

```bash
# Build a temporary container to verify architecture
docker run --rm --platform linux/amd64 ghcr.io/mhsanaei/3x-ui:latest uname -m
# Should output: x86_64

# If it shows "aarch64" or "arm64" - START OVER with skopeo!
```

---

## 📤 Upload to MikroTik

### Option 1: SCP (Recommended)

```bash
# Upload via SSH
scp 3x-ui-amd64.tar admin@YOUR_MIKROTIK_IP:/

# Example:
scp 3x-ui-amd64.tar admin@192.168.88.1:/
```

### Option 2: WebFig

1. Open WebFig: `http://YOUR_MIKROTIK_IP`
2. Go to **Files**
3. Click **Upload**
4. Select `3x-ui-amd64.tar`
5. Wait for upload to complete

### Option 3: WinBox

1. Open WinBox
2. Connect to your MikroTik
3. Menu: **Files**
4. Drag and drop `3x-ui-amd64.tar`

### Option 4: FTP

```bash
# Enable FTP on MikroTik first
# On MikroTik:
/ip service enable ftp

# Upload from your machine
ftp YOUR_MIKROTIK_IP
# Username: admin
# Password: your_password
put 3x-ui-amd64.tar
bye

# Disable FTP after upload (security)
/ip service disable ftp
```

---

## ✅ Verification on MikroTik

After upload, verify the file:

```routeros
# Check file exists and size
/file print where name~"3x-ui"

# Example output:
# NAME                TYPE   SIZE        
# 3x-ui-amd64.tar     .tar   ~120MB
```

---

## 🔄 Updating 3X-UI

To update to a newer version:

```bash
# 1. Pull new version with skopeo
skopeo copy --override-arch amd64 --override-os linux \
  docker://ghcr.io/mhsanaei/3x-ui:latest \
  docker-archive:3x-ui-amd64-new.tar:ghcr.io/mhsanaei/3x-ui:latest

# 2. Upload to MikroTik
scp 3x-ui-amd64-new.tar admin@YOUR_MIKROTIK_IP:/

# 3. On MikroTik, stop container
/container stop [find interface~"3x-ui"]

# 4. Remove old container (keeps data if root-dir unchanged)
/container remove [find interface~"3x-ui"]

# 5. Add new container with same root-dir
/container add file=3x-ui-amd64-new.tar \
  interface=3x-ui \
  root-dir=disk1/3xui \
  dns=8.8.8.8 \
  start-on-boot=yes

# 6. Start container
/container start [find interface~"3x-ui"]
```

**⚠️ Important**: Using the same `root-dir` preserves all settings, users, and configurations!

---

## 🐛 Common Issues

### Issue: "Exec format error"

**Cause**: Wrong architecture (ARM64 instead of AMD64)

**Solution**: 
1. Delete the tar file
2. Start over with `skopeo` method
3. Verify with `uname -m` inside container (should be `x86_64`)

### Issue: "No space left on device"

**Cause**: Not enough disk space on MikroTik

**Solution**:
```routeros
# Check disk space
/disk print

# If using USB/SD card, ensure it's mounted
/disk mount [find type="disk"]

# Clean old container images
/file remove [find name~"old-image.tar"]
```

### Issue: Upload fails or times out

**Cause**: Large file, slow connection, or firewall blocking

**Solution**:
```routeros
# Increase timeout for SCP/FTP
# Or split upload into chunks (advanced)

# Use USB stick instead:
# 1. Copy tar to USB
# 2. Insert USB into MikroTik
# 3. Wait for mount
# 4. Copy: /file copy [find name="3x-ui.tar" where type="disk"] disk1/
```

### Issue: File corrupted after upload

**Cause**: Transfer interrupted or file system issue

**Solution**:
```bash
# Verify checksum before upload
sha256sum 3x-ui-amd64.tar > checksum.txt

# After upload, verify on MikroTik
/file print detail where name="3x-ui-amd64.tar"

# Re-upload if sizes don't match
```

---

## 📊 Image Size Considerations

**Expected sizes:**
- 3X-UI Docker image: ~100-150MB compressed
- After extraction in container: ~250-300MB
- With data and logs: ~350-500MB

**Minimum MikroTik storage:**
- CHR: 128MB disk (tight, but works)
- Physical router with USB: 1GB+ recommended
- CCR series with internal storage: No issues

---

## 🔐 Security Notes

### Image Verification

Always verify you're pulling from official sources:

```bash
# Official 3X-UI repository
ghcr.io/mhsanaei/3x-ui:latest

# Verify on GitHub:
# https://github.com/MHSanaei/3x-ui/pkgs/container/3x-ui
```

### Clean Up Local Files

After successful deployment:

```bash
# Remove local tar file (contains no sensitive data, but good practice)
rm ~/Desktop/3x-ui-amd64.tar

# Or keep for backups/other routers
mkdir ~/mikrotik-images
mv 3x-ui-amd64.tar ~/mikrotik-images/
```

---

## 📚 Next Steps

Once your image is prepared and uploaded:

1. [Container Setup →](02-container-setup.md) - Create and configure the container
2. [Network Configuration →](03-network-config.md) - Set up networking

---

## 🤔 FAQ

**Q: Can I use Docker Hub instead of GHCR?**  
A: Yes, but 3X-UI official images are on GHCR. Docker Hub mirrors may be outdated.

**Q: Do I need Docker installed on MikroTik?**  
A: No! MikroTik has its own container runtime. You only need Docker locally for image preparation.

**Q: Can I use Podman instead of Docker?**  
A: Yes! Podman can save images to tar format compatible with MikroTik.

**Q: How often should I update?**  
A: Check [3X-UI releases](https://github.com/MHSanaei/3x-ui/releases) monthly. Update when security fixes are released.

---

[⬅ Back to README](../README.md) | [Next: Container Setup →](02-container-setup.md)
