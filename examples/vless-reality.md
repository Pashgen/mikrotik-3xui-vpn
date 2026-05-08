# VLESS Reality Configuration Guide

Complete guide for setting up VLESS with Reality protocol - the most advanced anti-censorship VPN configuration.

---

## 🎯 What is VLESS Reality?

**VLESS Reality** is the latest evolution in anti-censorship technology:

- 🔒 **No TLS certificates needed** - Reality mimics real HTTPS traffic
- 🎭 **Perfect camouflage** - Looks exactly like browsing google.com or cloudflare.com
- 🚀 **Better performance** - Lower latency than traditional TLS
- 🛡️ **DPI-resistant** - Bypasses even sophisticated deep packet inspection

**How it works:**  
Reality makes your VPN traffic look like a legitimate HTTPS connection to a popular website (like Google). Even with deep packet inspection, censors cannot distinguish it from real traffic.

---

## 📋 Prerequisites

Before configuring Reality:
- ✅ 3X-UI panel running and accessible
- ✅ Container has internet access
- ✅ VPN port accessible externally (test with `nc -zv YOUR_IP PORT`)
- ✅ Client that supports Reality (V2rayN, V2rayNG, SagerNet, Nekoray)

---

## 🔧 Server Configuration (3X-UI Panel)

### Step 1: Create Reality Inbound

1. Login to 3X-UI panel: `https://YOUR_IP:8443/`

2. Go to **Inbounds** → **Add Inbound**

3. **Basic Settings:**
   - **Remark**: `VLESS Reality - Finland`
   - **Protocol**: `VLESS`
   - **Listen IP**: `0.0.0.0` (or leave empty)
   - **Port**: `1443` (or any available port)

4. **Network Settings:**
   - **Transport/Network**: `TCP`
   - **Security**: `Reality`

5. **Reality Configuration:**

   Click **Generate** buttons to create:
   - **Public Key** / **Private Key** - Automatically generated pair
   - **Short IDs** - Click multiple times to generate several IDs

   **Server Name (SNI)**: Choose a real, popular website:
   ```
   Good choices:
   - google.com
   - www.google.com
   - cloudflare.com
   - www.microsoft.com
   - www.apple.com
   - yahoo.com
   ```

   **⚠️ Important SNI rules:**
   - Must be a **real** website
   - Must support **HTTPS** (TLS 1.3)
   - Must be **highly available** (no downtime)
   - Popular sites work best (harder to block)

   **Fingerprint**: `chrome` (most common)
   
   Other options: `firefox`, `safari`, `edge`, `ios`, `android`

6. **Flow Settings:**
   - **Flow**: `xtls-rprx-vision`
   - Enables XTLS for better performance

7. **Sniffing** (Enable these):
   - ✅ HTTP
   - ✅ TLS
   - ✅ QUIC
   - ✅ FAKEDNS

8. **Add Client:**
   - **Email/Name**: `user1` (or any identifier)
   - **UUID**: Auto-generated (don't change)
   - **Flow**: `xtls-rprx-vision`
   - **Enable**: ✅

9. Click **Save**

### Step 2: Configure MikroTik Firewall/NAT

```routeros
# Allow VPN port (example: 1443)
/ip firewall filter add chain=input action=accept \
  protocol=tcp dst-port=1443 place-before=0 \
  comment="VLESS Reality"

# NAT forwarding to container
/ip firewall nat add chain=dstnat action=dst-nat \
  to-addresses=10.10.4.4 to-ports=1443 \
  protocol=tcp dst-port=1443 \
  in-interface=ether1 \
  comment="VLESS Reality"
```

**Replace:**
- `1443` with your chosen port
- `ether1` with your WAN interface

### Step 3: Test Port Accessibility

From external machine:
```bash
nc -zv YOUR_PUBLIC_IP 1443
# Should show: Connection succeeded
```

---

## 📱 Client Configuration

### Export from Panel

In 3X-UI panel:
1. Go to **Inbounds**
2. Find your Reality inbound
3. Click **Export** icon (QR code or link icon)
4. Copy the `vless://` URL

**URL format:**
```
vless://UUID@IP:PORT?type=tcp&security=reality&pbk=PUBLIC_KEY&fp=chrome&sni=google.com&sid=SHORT_ID&flow=xtls-rprx-vision#Name
```

### Windows: V2rayN

1. **Download V2rayN**: https://github.com/2dust/v2rayN/releases
2. Extract and run `v2rayN.exe`
3. **Servers** → **Import from clipboard**
4. Paste the `vless://` URL
5. Right-click server → **Set as active server**
6. **System Proxy** → **Global mode** or **PAC mode**
7. Test: https://whoer.net

### Android: V2rayNG

1. **Download V2rayNG**: https://github.com/2dust/v2rayNG/releases
2. Install APK
3. Tap **+** → **Import from clipboard**
4. Paste the `vless://` URL
5. Tap connection to connect
6. Test: https://whoer.net

### iOS: Shadowrocket

1. **Buy Shadowrocket** from App Store (paid app)
2. Tap **+** → **Type** → **VLESS**
3. Manual configuration:
   - **Address**: YOUR_PUBLIC_IP
   - **Port**: 1443
   - **UUID**: From panel
   - **TLS**: Reality
   - **Public Key**: From panel
   - **Short ID**: From panel
   - **SNI**: google.com
   - **Fingerprint**: chrome
   - **Flow**: xtls-rprx-vision
4. Save and connect

### Linux: Nekoray

1. **Download Nekoray**: https://github.com/MatsuriDayo/nekoray/releases
2. Add server → Import URL or manual config
3. Start connection

---

## 🔍 Manual Configuration Parameters

If importing URL doesn't work, configure manually:

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Protocol** | VLESS | Must be VLESS |
| **Address** | YOUR_PUBLIC_IP | Your MikroTik public IP |
| **Port** | 1443 | Your configured port |
| **UUID** | From panel | User ID from inbound |
| **Flow** | xtls-rprx-vision | Enable XTLS |
| **Encryption** | none | VLESS uses none |
| **Network** | tcp | Transport protocol |
| **Security** | reality | Critical! |
| **SNI** | google.com | Must match server |
| **Fingerprint** | chrome | Must match server |
| **Public Key** | From panel | Copy exactly |
| **Short ID** | From panel | Any from list |
| **Spider X** | / | Default, or leave empty |

---

## ✅ Verification Checklist

### Server-side (MikroTik)

```routeros
# 1. Check Xray is listening
/container shell [find interface~"3x-ui"]
netstat -tulpn | grep 1443
# Should show: tcp ... :::1443 ... LISTEN ... xray

# 2. Check logs for connections
tail -f /var/log/x-ui/3xui.log
# Connect with client - should see "accepted connection"

# 3. Exit container
exit

# 4. Check NAT logs
/ip firewall nat set [find dst-port=1443] log=yes
/log print follow where topics~"firewall"
# Connect with client - should see packets
```

### Client-side

1. **Connect** to VPN
2. **Check IP**: Visit https://whoer.net
   - IP should show Finland (or your server location)
   - DNS should be clean
3. **Speed test**: https://fast.com or https://speedtest.net
4. **Leak test**: https://ipleak.net
   - No DNS leaks
   - No WebRTC leaks

---

## 🐛 Troubleshooting Reality

### Issue: Connection Fails Immediately

**Check parameter mismatch:**

```bash
# Common mistakes:
❌ SNI doesn't match (server: google.com, client: www.google.com)
❌ Public key copied wrong (case-sensitive!)
❌ Short ID missing or wrong
❌ Fingerprint mismatch (server: chrome, client: firefox)
❌ Flow setting different (server has flow, client doesn't)
```

**Solution:** Export URL from panel and import into client (don't type manually!)

### Issue: Connects Then Immediately Disconnects

**Cause:** Reality handshake succeeds, but traffic routing fails.

**Check:**
```routeros
# Container has internet?
/container shell [find interface~"3x-ui"]
ping -c 3 8.8.8.8
```

If ping fails, see [Troubleshooting Guide](../docs/07-troubleshooting.md#issue-no-internet-inside-container).

### Issue: Slow Speeds with Reality

**Causes:**
1. **Hardware limitations** - CHR with 1 vCPU struggles with many connections
2. **Flow not enabled** - Make sure `xtls-rprx-vision` is set on both sides
3. **MTU issues** - Try reducing MTU in client

**Solutions:**
```bash
# Enable flow on both server and client
# Server: In inbound settings
# Client: In connection settings

# Check Xray is using flow
/container shell [find interface~"3x-ui"]
cat /app/bin/config.json | grep -i flow
```

### Issue: Works on WiFi But Not Mobile Data

**Cause:** Mobile carrier blocking VPN ports.

**Solutions:**
1. **Change port** to 443 or 8443 (if available)
2. **Use WebSocket transport** instead of TCP (more overhead, but bypasses port blocking)
3. **Enable CDN** (advanced, requires domain)

---

## 🎨 Advanced Reality Configurations

### Multiple Short IDs

Generate multiple Short IDs in panel and clients can use any of them:

```
Short IDs: ["abc123", "def456", "789ghi"]
```

Benefits:
- Load distribution
- Harder to fingerprint
- Fallback options

### Custom Spider X

Default `/` works fine, but you can customize:

```
Spider X: /generate_204
Spider X: /api/v1/status
```

Must be valid paths on the target SNI domain.

### Fallback Configuration

If Reality is blocked, configure fallback inbound:

1. **Primary**: Reality on port 1443
2. **Fallback**: VMess with WebSocket on port 8443

Client tries Reality first, falls back to VMess if blocked.

---

## 📊 Performance Optimization

### For Low-End Hardware (CHR with 1 vCPU)

```bash
# Inside container, reduce Xray logging
/container shell [find interface~"3x-ui"]

# Edit config
vi /app/bin/config.json

# Change log level:
"log": {
  "loglevel": "warning"  # Instead of "debug"
}

# Restart Xray
killall xray-linux-amd64
```

### For High Traffic (50+ Users)

1. **Increase hardware**:
   - 2+ vCPUs
   - 2GB+ RAM

2. **Optimize Xray**:
   ```json
   "policy": {
     "levels": {
       "0": {
         "connIdle": 300,
         "uplinkOnly": 0
       }
     }
   }
   ```

3. **Load balancing** (advanced):
   - Multiple MikroTik routers
   - Round-robin DNS
   - Or single router with multiple containers

---

## 🔐 Security Best Practices

### 1. Regular Key Rotation

Every 3-6 months:
- Generate new Public/Private key pair
- Update clients with new config
- Disable old keys

### 2. Per-User UUIDs

Don't share one UUID among multiple users:
- Each user gets unique UUID
- Easier to track usage
- Revoke individual users if needed

### 3. Monitor Traffic

In 3X-UI panel:
- Check **Statistics** regularly
- Set traffic limits per user
- Alert on abnormal usage

### 4. Firewall Hardening

```routeros
# Rate limit connections to VPN port
/ip firewall filter add chain=input protocol=tcp dst-port=1443 \
  connection-limit=50,32 action=drop \
  comment="Rate limit VPN"

# Place before accept rule!
```

---

## 📚 Further Reading

- [Xray Reality Documentation](https://xtls.github.io/)
- [Reality Protocol Specification](https://github.com/XTLS/REALITY)
- [3X-UI GitHub](https://github.com/MHSanaei/3x-ui)

---

## 💡 Reality vs Other Protocols

| Feature | Reality | VMess | Trojan |
|---------|---------|-------|--------|
| **Censorship resistance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Setup complexity** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Client support** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Certificate required** | ❌ | ❌ | ✅ |

**Recommendation:** Use Reality for maximum censorship resistance!

---

[⬅ Back to README](../README.md) | [Troubleshooting →](../docs/07-troubleshooting.md)
