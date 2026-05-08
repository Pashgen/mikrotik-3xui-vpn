# Troubleshooting Guide

Complete troubleshooting guide based on real-world deployment experience. This covers EVERY issue we encountered and how to fix it.

---

## 🔍 Quick Diagnosis

### Container Issues

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| "Exec format error" | Wrong architecture (ARM64) | Rebuild with skopeo |
| Container won't start | No disk space | Check `/disk print` |
| Container restarts constantly | Application crash | Check `/container shell` logs |
| High RAM usage | Too many connections | Reduce connections or upgrade RAM |

### Network Issues

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| Can't ping from container | No internet routing | Check NAT rules |
| Panel accessible locally, not externally | Firewall blocking | Check rule order |
| VPN connects but no traffic | No Xray logs | Check inbound config |
| Port timeout from outside | ISP blocking port | Use different port |

### VPN Issues

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| Client connects, no internet | Routing/masquerade issue | Check container internet |
| No Xray logs on connection | Wrong client config | Verify protocol match |
| Reality handshake fails | Mismatched parameters | Check pbk, sid, sni |

---

## 🐛 Container Problems

### Issue: "Exec format error"

**Full error in logs:**
```
container: process_start: exec format error
```

**Cause:**  
Docker image is ARM64 architecture, but MikroTik is x86_64.

**Solution:**
1. Delete current image:
   ```routeros
   /file remove 3x-ui-amd64.tar
   ```

2. Re-create image with skopeo:
   ```bash
   skopeo copy --override-arch amd64 --override-os linux \
     docker://ghcr.io/mhsanaei/3x-ui:latest \
     docker-archive:3x-ui-amd64.tar:ghcr.io/mhsanaei/3x-ui:latest
   ```

3. Upload and recreate container

**Verification:**
```routeros
/container shell [find interface~"3x-ui"]
uname -m
# Must output: x86_64
```

---

### Issue: Container Won't Start - No Disk Space

**Symptoms:**
- Container status stuck at "extracting"
- Error: "no space left on device"

**Diagnosis:**
```routeros
/disk print
```

**Solution 1: Free up space**
```routeros
# Remove old files
/file print
/file remove [find name~"old-file"]

# Remove old containers
/container print
/container remove [find tag~"old"]
```

**Solution 2: Use external storage**
```routeros
# Check for USB/SD card
/disk print

# Mount if not mounted
/disk mount [find type="disk"]

# Use external storage for container
/container add ... root-dir=usb1/3xui ...
```

**Solution 3: Increase CHR disk (CHR only)**
If running CHR on a hypervisor, increase the virtual disk size.

---

### Issue: Container Keeps Restarting

**Symptoms:**
- Container starts then stops repeatedly
- Status changes between "running" and "stopped"

**Diagnosis:**
```routeros
# Watch logs in real-time
/log print follow where topics~"container"

# Access shell to see what's failing
/container shell [find interface~"3x-ui"]
ps aux
dmesg | tail -20
```

**Common causes:**

**1. Application crashing:**
```bash
# Inside container
tail -100 /var/log/x-ui/3xui.log | grep -i error
```

**2. Missing dependencies:**
```bash
# Check if x-ui binary exists
ls -la /app/x-ui
```

**3. Permission issues:**
```bash
# Check ownership
ls -la /etc/x-ui/
```

**Solution:**  
Usually requires container rebuild or checking application logs for specific errors.

---

### Issue: Can't Access Container Shell

**Symptoms:**
```routeros
/container shell [find interface~"3x-ui"]
# Returns immediately or shows error
```

**Solution:**
```routeros
# Check container is actually running
/container print detail

# If stopped, start it
/container start [find interface~"3x-ui"]

# Wait 30 seconds, then try again
:delay 30
/container shell [find interface~"3x-ui"]
```

---

## 🌐 Network Problems

### Issue: No Internet Inside Container

**Symptom:**  
From container shell, can't ping 8.8.8.8 or google.com

**Diagnosis:**
```bash
# Inside container
ping -c 3 8.8.8.8        # Test IP connectivity
ping -c 3 google.com     # Test DNS
ip route                 # Check routing table
cat /etc/resolv.conf     # Check DNS config
```

**Solution 1: Fix NAT**
```routeros
# Check NAT exists
/ip firewall nat print where src-address~"10.10.4"

# If missing, add it
/ip firewall nat add chain=srcnat action=masquerade \
  src-address=10.10.4.0/24 out-interface=ether1 \
  comment="Container NAT"

# Replace ether1 with your WAN interface!
```

**Solution 2: Fix forwarding**
```routeros
# Allow container traffic
/ip firewall filter add chain=forward action=accept \
  in-interface=bridge-3x-ui place-before=0
```

**Solution 3: Fix gateway**
```routeros
# Recreate veth with correct gateway
/interface veth remove 3x-ui
/interface veth add name=3x-ui address=10.10.4.4/24 gateway=10.10.4.1
```

**Verification:**
```bash
# From container
ping -c 3 8.8.8.8     # Should work
wget -qO- https://api.ipify.org  # Shows public IP
```

---

### Issue: Panel Not Accessible Externally

**Symptom:**  
`curl https://ROUTER_IP:8443` times out from internet, but works from MikroTik

**Diagnosis:**
```routeros
# Test from MikroTik to container
/tool fetch url="https://10.10.4.4:8443/" mode=https check-certificate=no

# Test from your PC
curl -v https://YOUR_PUBLIC_IP:8443
```

**If MikroTik → container works, but external doesn't:**

**Solution: Fix firewall rule order**

The most common issue! Firewall rules are processed in order. If a DROP rule comes before your ACCEPT rule, traffic is blocked.

```routeros
# Check rule order
/ip firewall filter print where chain=input

# Look for your rule and any DROP rules
# If DROP is BEFORE your accept rule, reorder:

# Move accept rule to position 0 (top)
/ip firewall filter move [find comment="3x-ui HTTPS"] destination=0

# Or place before first DROP rule (example: rule 50)
/ip firewall filter move [find comment="3x-ui HTTPS"] destination=50
```

**Rule order example:**
```
# ✅ CORRECT:
0  accept  tcp  dstport=8443  (3x-ui)
1  accept  established,related
50 drop    all  (default drop)

# ❌ WRONG:
0  accept  established,related  
50 drop    all  (default drop)
60 accept  tcp  dstport=8443  (3x-ui) ← Never reached!
```

**Quick test - temporarily disable DROP rules:**
```routeros
# Disable all input drops temporarily
/ip firewall filter disable [find chain=input and action=drop]

# Test connection
curl https://YOUR_PUBLIC_IP:8443

# Re-enable after test
/ip firewall filter enable [find chain=input and action=drop]
```

If it works with drops disabled, the issue is rule order!

---

### Issue: Port Blocked by ISP

**Symptom:**  
`nc -zv YOUR_IP PORT` shows "Operation timed out" even though MikroTik logs show packets arriving

**Diagnosis:**
```routeros
# Enable NAT logging
/ip firewall nat set [find dst-port=YOUR_PORT] log=yes

# Watch logs while testing
/log print follow where topics~"firewall"

# Try connection from external machine
# Do you see packets in logs?
```

**If packets appear in logs but connection times out:**  
Your ISP is blocking the port!

**Solution: Use different ports**

Commonly blocked ports:
- ❌ 22 (SSH)
- ❌ 80, 443 (if residential connection)
- ❌ 585, 1194, 1723 (VPN ports)
- ❌ 8388 (Shadowsocks default)

Safe ports (usually not blocked):
- ✅ 1443 (looks like HTTPS)
- ✅ 2053, 2083 (Cloudflare ports)
- ✅ 8443 (alternative HTTPS)
- ✅ 3000-3999 (development ports)

**Port testing script:**
```bash
# Test multiple ports from external machine
for port in 1443 2053 8443 3000; do
  echo "Testing port $port..."
  nc -zv -w2 YOUR_PUBLIC_IP $port
done
```

---

### Issue: "Connection refused" Instead of Timeout

**Symptom:**  
`nc -zv YOUR_IP PORT` shows "Connection refused" immediately

**This is actually GOOD news!** It means:
- ✅ Packets reach your router
- ✅ ISP is not blocking
- ❌ But something on MikroTik is responding "no"

**Common causes:**

**1. Port already in use by MikroTik service:**
```routeros
/ip service print

# Example: SSH using port 22
# Solution: Change SSH port or use different port for VPN
/ip service set ssh port=2222
```

**2. NAT rule conflict:**
```routeros
# Check if multiple NAT rules exist for same port
/ip firewall nat print where dst-port=YOUR_PORT

# Remove duplicates or conflicts
```

**3. Firewall explicitly rejecting:**
```routeros
# Look for "reject" rules
/ip firewall filter print where action=reject

# Temporarily disable to test
/ip firewall filter disable [find action=reject]
```

---

## 🔌 VPN Connection Problems

### Issue: VPN Connects But No Internet

**Symptom:**  
- V2rayN/client shows "connected"
- No traffic in 3X-UI panel statistics  
- No internet access through VPN

**Diagnosis step-by-step:**

**1. Check Xray is receiving connections:**
```routeros
/container shell [find interface~"3x-ui"]
tail -f /var/log/x-ui/3xui.log
# Try connecting - do you see log entries?
```

**2. If NO logs appear:**

Client is not reaching Xray! Check:

```routeros
# From MikroTik - check packets arriving
/ip firewall nat set [find dst-port=YOUR_VPN_PORT] log=yes
/log print follow where topics~"firewall"
```

If you see:
```
dstnat: in:ether1 out:bridge-3x-ui ... ->10.10.4.4:PORT
```

Packets ARE reaching container, but Xray isn't responding.

**Solution A: Verify inbound configuration**

In 3X-UI panel:
- Protocol matches client (VLESS vs VMess)
- Port matches
- Security settings match (TLS/Reality/none)
- Listen address is `0.0.0.0` or empty

**Solution B: Restart Xray**
```bash
# Inside container
killall xray-linux-amd64
# x-ui will auto-restart it
sleep 3
ps aux | grep xray
```

**3. If logs DO appear:**

Xray is receiving traffic! Check container internet:

```bash
# Inside container
ping -c 3 8.8.8.8
ping -c 3 google.com
```

If ping fails, see [No Internet Inside Container](#issue-no-internet-inside-container) above.

---

### Issue: Reality Handshake Fails

**Symptom:**  
Client connects, immediately disconnects. Logs show handshake errors.

**Cause:**  
Reality parameters mismatch between server and client.

**Critical Reality parameters (MUST MATCH EXACTLY):**

| Parameter | Server (3X-UI) | Client | Notes |
|-----------|----------------|--------|-------|
| Public Key | Generated on server | Copy from panel | Case-sensitive |
| Short ID | Generated on server | Copy from panel | Hex string |
| SNI | Server name (e.g. google.com) | Must match exactly | Domain must be real |
| Fingerprint | chrome/firefox/safari | Must match | Client TLS fingerprint |

**Solution:**

1. In 3X-UI panel, click "Export" on inbound
2. Copy the full `vless://` URL
3. Import directly into client (don't type manually!)

**Correct VLESS Reality URL format:**
```
vless://UUID@IP:PORT?type=tcp&security=reality&pbk=PUBLIC_KEY&fp=chrome&sni=google.com&sid=SHORT_ID&flow=xtls-rprx-vision#NAME
```

**Verification checklist:**
- ✅ `security=reality` present
- ✅ `pbk=` (public key) present and correct
- ✅ `sid=` (short ID) present
- ✅ `sni=` matches a real domain (google.com, cloudflare.com)
- ✅ `fp=` matches client's TLS fingerprint capability
- ✅ `flow=xtls-rprx-vision` if using flow

---

### Issue: Only WebSocket Connections in Logs

**Symptom:**  
Logs show:
```
WebSocket client connected: xxx-xxx-xxx
WebSocket client disconnected: xxx-xxx-xxx
```

But NO Xray inbound logs like:
```
accepted connection from X.X.X.X:PORT
```

**Cause:**  
You're connecting to the PANEL, not the VPN inbound!

**Solution:**

Check client configuration:
- ❌ Port 8443 or 2096 = Panel/subscription port
- ✅ Your configured VPN port (e.g. 1443) = Correct!

**Verify in client:**
- Server address: YOUR_PUBLIC_IP
- Port: **VPN inbound port** (NOT panel port!)
- Protocol: matches inbound (VLESS/VMess/Trojan)

---

### Issue: Xray Not Listening on Port

**Symptom:**  
```bash
netstat -tulpn | grep YOUR_PORT
# Returns nothing
```

**Cause:**  
Inbound not created or Xray crashed.

**Solution 1: Check inbound exists**

In 3X-UI panel:
- Go to Inbounds
- Check status is ✅ (enabled)
- Check port number is correct

**Solution 2: Restart Xray**

In panel: **Operations** → **Restart Xray Core**

Or manually:
```bash
# Inside container
killall xray-linux-amd64
# Wait for auto-restart
sleep 5
netstat -tulpn | grep LISTEN
```

**Solution 3: Check Xray config**
```bash
# Inside container
cat /app/bin/config.json | grep -A 20 '"inbounds"'

# Look for your port in the config
```

If port is not in config, Xray doesn't know about it. Recreate inbound in panel.

---

## 🔐 SSL/Certificate Problems

### Issue: Browser Shows Certificate Error

**Symptom:**  
"Your connection is not private" / "NET::ERR_CERT_AUTHORITY_INVALID"

**This is NORMAL for self-signed certificates!**

**Solution:**  
Click "Advanced" → "Proceed to site (unsafe)"

Or install certificate on your device (see [SSL Certificates guide](04-ssl-certificates.md)).

---

### Issue: Let's Encrypt Certificate Fails

**Symptom:**  
Certificate generation fails in 3X-UI panel.

**Common causes:**

**1. Port 80 not accessible:**
```bash
# Test from external machine
nc -zv YOUR_PUBLIC_IP 80
```

If times out, either:
- ISP blocks port 80 (common on residential)
- MikroTik firewall blocking
- No NAT rule for port 80

**Solution:** Use self-signed certificate instead, or use DNS challenge (advanced).

**2. Domain not pointing to your IP:**
```bash
# Check DNS
dig YOUR_DOMAIN.COM +short
# Should show YOUR_PUBLIC_IP
```

---

## 📊 Performance Issues

### Issue: High CPU Usage

**Diagnosis:**
```routeros
/system resource print
```

**Causes and solutions:**

**1. Too many connections:**
- Check active users in 3X-UI panel
- Set connection limits per user

**2. Xray logging level too high:**
```bash
# Inside container
# Edit /app/bin/config.json
# Change "loglevel": "debug" to "warning"
```

**3. Hardware limitations:**
- CHR with 1 vCPU will struggle with 50+ concurrent connections
- Upgrade to more CPU cores if possible

### Issue: High RAM Usage

**Diagnosis:**
```routeros
/system resource print
```

**Solutions:**

**1. Reduce Xray logging:**
```bash
# Lower log retention
rm /var/log/x-ui/*.log.old
```

**2. Limit concurrent connections:**
In 3X-UI panel, set per-user limits.

**3. Upgrade MikroTik RAM** (if physical router).

---

## 🗂️ Data/Configuration Issues

### Issue: Settings Lost After Restart

**Cause:**  
Container root directory not persisted.

**Solution:**

Ensure you specified `root-dir` when creating container:
```routeros
/container print detail
# Check root-dir= parameter
```

If missing, recreate container:
```routeros
/container stop [find interface~"3x-ui"]
/container remove [find interface~="3x-ui"]

/container add file=3x-ui-amd64.tar \
  interface=3x-ui \
  root-dir=disk1/3xui \
  ...
```

---

### Issue: Can't Change Admin Password

**Symptom:**  
Password change in panel doesn't work, or login fails after change.

**Solution:**

Use container command-line tool:
```bash
# Inside container
/app/x-ui setting -username admin -password YOUR_NEW_PASSWORD

# Verify
/app/x-ui setting -show | grep username
```

Don't try to manually edit SQLite database - use the built-in command!

---

## 🛠️ Diagnostic Commands Reference

### Container Health Check

```routeros
# Container status
/container print detail

# Container logs
/log print where topics~"container"

# Shell access
/container shell [find interface~"3x-ui"]
```

### Network Diagnosis

```routeros
# Check interfaces
/interface print where name~"3x-ui"

# Check IP addresses
/ip address print where interface~"3x-ui"

# Check NAT rules
/ip firewall nat print

# Check firewall rules
/ip firewall filter print where chain=input

# Enable logging on specific rule
/ip firewall filter set [find comment="3x-ui"] log=yes
/log print follow where topics~"firewall"
```

### Inside Container

```bash
# Process list
ps aux | grep -E 'x-ui|xray'

# Network ports
netstat -tulpn

# Internet test
ping -c 3 8.8.8.8
wget -qO- https://api.ipify.org

# Logs
tail -100 /var/log/x-ui/3xui.log
```

### From External Machine

```bash
# Port test
nc -zv YOUR_IP PORT

# Connection test with timeout
nc -zv -w5 YOUR_IP PORT

# TLS test (if using TLS)
openssl s_client -connect YOUR_IP:PORT

# Full protocol test
curl -v https://YOUR_IP:PORT
```

---

## 📞 Getting Help

If you're still stuck:

1. **Gather diagnostics:**
   ```routeros
   /export file=config-backup
   /log print where topics~"container"
   ```

2. **Check existing issues:**  
   Search [GitHub issues](https://github.com/Pashgen/mikrotik-3xui-vpn/issues)

3. **Open new issue with:**
   - MikroTik model and RouterOS version
   - Exact error messages
   - Relevant log excerpts
   - What you've already tried

4. **Community help:**
   - [MikroTik Forum](https://forum.mikrotik.com/)
   - [3X-UI GitHub](https://github.com/MHSanaei/3x-ui/issues)

---

[⬅ Back to README](../README.md) | [Documentation Index](../README.md#documentation)
