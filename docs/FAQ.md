# Frequently Asked Questions (FAQ)

Common questions and answers about MikroTik 3X-UI setup.

---

## 🎯 General Questions

### Q: Why run 3X-UI on MikroTik instead of a VPS?

**A:** Several advantages:

✅ **Cost savings** - No monthly VPS bills  
✅ **Lower latency** - VPN server at home/office  
✅ **Full control** - Your hardware, your rules  
✅ **Privacy** - No third-party VPS provider  
✅ **Learning** - Great way to learn containers and networking  

**Best for:**
- Personal/family VPN (5-20 users)
- Small office VPN
- Learning and experimentation

**Not ideal for:**
- Commercial VPN service (100+ users)
- High-bandwidth requirements (>500 Mbps)
- Locations with unstable internet

---

### Q: What MikroTik hardware do I need?

**A:** Minimum requirements:

**For testing/personal use (1-5 users):**
- MikroTik CHR (Cloud Hosted Router)
- 512MB RAM
- 1 vCPU
- 300MB disk space

**For family/small team (5-20 users):**
- Physical router: RB4011, RB5009, or CCR series
- 1GB+ RAM
- 2+ CPU cores
- 1GB+ disk/USB storage

**For heavy use (20-50 users):**
- CCR2004 or better
- 2GB+ RAM
- 4+ CPU cores
- Good cooling

**Architecture:** Must be **x86_64** (AMD64). ARM devices like hAP series won't work with this guide.

---

### Q: Can I use this on hAP lite, hAP ac2, or other ARM MikroTik?

**A:** Unfortunately, **NO**. 

This guide uses 3X-UI Docker image compiled for **x86_64** architecture. MikroTik hAP series uses **ARM** processors.

**Workaround options:**
1. Use MikroTik as router + separate x86 device for VPN
2. Run 3X-UI on Raspberry Pi 4 + connect to MikroTik network
3. Use RouterOS built-in VPN (SSTP, L2TP, OpenVPN)

---

### Q: Will this work with CHR free license?

**A:** Yes! CHR free license includes:
- ✅ Container support
- ✅ 1 Mbps upload limit (doesn't affect download!)
- ✅ All routing features

**Performance:**
- Download: Unlimited
- Upload: 1 Mbps (enough for browsing, streaming)
- For higher upload, buy CHR license

---

## 🐳 Container Questions

### Q: Why use skopeo instead of docker save?

**A:** Critical for ARM Macs (M1/M2):

**Problem:**
```bash
docker save ghcr.io/mhsanaei/3x-ui > image.tar
# On M1 Mac, this creates ARM64 image!
# MikroTik will show: "Exec format error"
```

**Solution:**
```bash
skopeo copy --override-arch amd64 ...
# Forces x86_64 architecture
```

Even on Intel Macs/Linux, `skopeo` is safer as it explicitly specifies architecture.

---

### Q: Can I run multiple containers?

**A:** Yes! Just use different networks:

```routeros
# Container 1: 3X-UI
/interface veth add name=3x-ui address=10.10.4.4/24 gateway=10.10.4.1

# Container 2: Something else  
/interface veth add name=container2 address=10.10.5.4/24 gateway=10.10.5.1
```

Each needs its own bridge, NAT rules, and ports.

---

### Q: How do I update 3X-UI?

**A:** Two methods:

**Method 1: In-place update (via panel)**
1. Login to 3X-UI panel
2. Click "Update" button
3. Wait for completion
4. Restart Xray

**Method 2: Replace container (recommended for major versions)**
1. Pull new image with skopeo
2. Stop and remove old container
3. Create new container with **same root-dir**
4. Start new container
5. Settings preserved!

```routeros
/container stop [find interface~"3x-ui"]
/container remove [find interface~"3x-ui"]
/container add file=3x-ui-new.tar interface=3x-ui root-dir=disk1/3xui ...
/container start [find interface~="3x-ui"]
```

---

## 🌐 Network Questions

### Q: My ISP blocks port 443, what do I do?

**A:** Use alternative ports:

**Test which ports work:**
```bash
./scripts/test-ports.sh YOUR_PUBLIC_IP
```

**Good alternatives:**
- 1443 (looks like HTTPS)
- 2053, 2083 (Cloudflare CDN ports)
- 8443 (alternative HTTPS)
- 3000-3999 (less likely blocked)

**Avoid:**
- 22 (SSH - often blocked)
- 80, 443 (HTTP/HTTPS - blocked on residential)
- 585, 1194, 1723 (known VPN ports)

---

### Q: Can I use dynamic DNS with this setup?

**A:** Yes! Many services work:

**Free options:**
- DuckDNS (duckdns.org)
- No-IP (noip.com)
- Dynu (dynu.com)

**MikroTik built-in:**
```routeros
/ip cloud set ddns-enabled=yes

# Your hostname: XXXXXXX.sn.mynetname.net
/ip cloud print
```

Use the hostname instead of IP in client configs!

---

### Q: Does this work with Starlink/CGNAT?

**A:** Depends:

**If you have public IP:**
✅ Yes, works perfectly

**If behind CGNAT (Carrier-Grade NAT):**
❌ Incoming connections blocked
✅ But you can still use it as VPN **client** (connect TO other servers)

**Check if you're behind CGNAT:**
```bash
curl https://api.ipify.org
# Compare with your router's WAN IP

# If different = CGNAT
# If same = Public IP
```

**CGNAT workarounds:**
- Ask ISP for public IP (sometimes costs extra)
- Use ZeroTier/Tailscale for NAT traversal
- Rent small VPS as VPN server, MikroTik as client

---

## 🔒 Security Questions

### Q: Is this secure?

**A:** Yes, IF configured properly:

✅ **VLESS Reality** - State-of-the-art encryption  
✅ **No logs** by default  
✅ **Self-hosted** - You control everything  

**Security checklist:**
- ✅ Changed default admin password
- ✅ Using strong passwords for VPN users
- ✅ Firewall rules properly configured
- ✅ Panel only accessible via HTTPS
- ✅ Regular updates
- ✅ Not sharing VPN configs publicly

**What's NOT secure:**
- ❌ Using admin/admin password
- ❌ Sharing one config with many people
- ❌ No firewall rules
- ❌ Outdated software

---

### Q: Can my ISP see I'm running a VPN server?

**A:** It depends:

**What ISP CAN see:**
- You're running something on port X
- Data amounts (encrypted, but visible)
- Connection patterns

**What ISP CANNOT see (with proper setup):**
- What data you're proxying
- Where your users are connecting to
- Content of VPN traffic

**With VLESS Reality:**
- Traffic looks like HTTPS to google.com
- Nearly impossible to detect without blocking Google itself

**Best practices:**
- Use common ports (443, 8443)
- Use Reality protocol for maximum camouflage
- Don't run suspicious services alongside

---

### Q: Will this get me in trouble?

**A:** Depends on your country's laws:

**Legal in most countries:**
- Personal/family VPN use
- Accessing geo-blocked content
- Privacy protection

**May be restricted:**
- Some countries restrict/ban VPNs
- Using for illegal activities (everywhere)
- Violating terms of service

**Check:**
- Local laws about VPN operation
- Your ISP's acceptable use policy
- Consider legal implications for your location

**We are not lawyers** - this is not legal advice!

---

## 🚀 Performance Questions

### Q: What speeds can I expect?

**A:** Depends on hardware:

**CHR with 1 vCPU:**
- 50-100 Mbps per connection
- 3-5 concurrent users max

**CHR with 2 vCPU:**
- 100-200 Mbps per connection
- 10-15 concurrent users

**Physical router (RB4011):**
- 200-500 Mbps
- 20-30 concurrent users

**CCR2004:**
- 500-1000 Mbps
- 50+ concurrent users

**Factors:**
- Protocol (Reality > VMess > Trojan)
- Encryption overhead
- Network latency
- Client device performance

---

### Q: Why is Reality faster than VMess?

**A:** Several reasons:

1. **XTLS** - More efficient encryption
2. **No double encryption** - Reality doesn't wrap in TLS
3. **Better flow control** - xtls-rprx-vision optimizes packet flow
4. **Modern design** - Built for performance

**Speed comparison (typical):**
- VLESS Reality with XTLS: ~95% of bare connection
- VMess: ~85% of bare connection
- Trojan: ~90% of bare connection

---

## 🐛 Troubleshooting Questions

### Q: Container won't start, what do I check?

**A:** Step-by-step diagnosis:

1. **Check logs:**
   ```routeros
   /log print where topics~"container"
   ```

2. **Common errors:**
   - "Exec format error" → Wrong architecture (use skopeo)
   - "No space" → Not enough disk (check `/disk print`)
   - "Network error" → Interface/bridge misconfigured

3. **Verify basics:**
   ```routeros
   /container print detail
   /interface print where name~"3x-ui"
   /ip address print where interface~"3x-ui"
   ```

4. **Try manual start:**
   ```routeros
   /container start [find interface~"3x-ui"]
   ```

See [Troubleshooting Guide](docs/07-troubleshooting.md) for complete solutions.

---

### Q: Panel works locally but not externally, why?

**A:** Firewall rule order issue (most common!):

```routeros
# Check order
/ip firewall filter print where chain=input

# If DROP rule comes BEFORE your accept rule:
# Move accept rule to top
/ip firewall filter move [find comment="3x-ui"] destination=0
```

**Test:**
```routeros
# Temporarily disable drops
/ip firewall filter disable [find chain=input and action=drop]

# Test external access
# If works → rule order issue
# If doesn't → NAT or port issue
```

---

### Q: VPN connects but no internet, what's wrong?

**A:** Routing/NAT issue:

1. **Check container has internet:**
   ```bash
   /container shell [find interface~"3x-ui"]
   ping -c 3 8.8.8.8
   ```

2. **If ping fails, check NAT:**
   ```routeros
   /ip firewall nat print where src-address~"10.10.4"
   ```

3. **Add if missing:**
   ```routeros
   /ip firewall nat add chain=srcnat action=masquerade \
     src-address=10.10.4.0/24 out-interface=ether1
   ```

See [Troubleshooting Guide](docs/07-troubleshooting.md#issue-vpn-connects-but-no-internet) for full diagnosis.

---

## 💻 Client Questions

### Q: Which VPN client should I use?

**A:** By platform:

**Windows:**
- V2rayN (recommended) - https://github.com/2dust/v2rayN
- Nekoray - https://github.com/MatsuriDayo/nekoray

**macOS:**
- V2Box
- Qv2ray

**Android:**
- V2rayNG (recommended) - https://github.com/2dust/v2rayNG
- SagerNet
- Matsuri

**iOS:**
- Shadowrocket (paid, $2.99)
- Stash (paid)

**Linux:**
- Nekoray
- Qv2ray
- v2ray command line

---

### Q: Can I use this on my phone?

**A:** Yes! Works on all platforms:

**Setup steps:**
1. Install client app (see above)
2. Copy subscription URL from 3X-UI panel
3. Import subscription in app
4. Connect!

**Or QR code:**
1. In 3X-UI panel, click QR code icon
2. Scan with client app
3. Connect!

---

## 📊 Usage Questions

### Q: How many users can I support?

**A:** Depends on hardware and usage:

**Light usage (browsing, messaging):**
- CHR 1 vCPU: 5-10 users
- CHR 2 vCPU: 15-20 users
- RB4011: 25-30 users
- CCR2004: 50+ users

**Heavy usage (video streaming):**
- Divide numbers above by 2-3

**Monitor in panel:**
- Real-time connections
- Bandwidth per user
- System resources

---

### Q: Can I set data limits per user?

**A:** Yes! In 3X-UI panel:

1. Go to **Inbounds**
2. Click on client
3. Set limits:
   - Total traffic (GB)
   - Expiry date
   - Max connections per user
   - Download/upload limits

4. User is automatically blocked when limit reached

---

### Q: How do I see who's connected?

**A:** Multiple ways:

**In 3X-UI Panel:**
- Dashboard shows active connections
- Statistics page shows per-user traffic
- Real-time bandwidth graphs

**MikroTik:**
```routeros
# Show container connections
/ip firewall connection print where dst-address~"10.10.4.4"

# Enable NAT logging
/ip firewall nat set [find comment~"3x-ui"] log=yes
/log print follow where topics~"firewall"
```

---

## 🔄 Maintenance Questions

### Q: Do I need to maintain this?

**A:** Minimal maintenance required:

**Monthly:**
- ✅ Check for 3X-UI updates
- ✅ Review user statistics
- ✅ Check system logs

**Quarterly:**
- ✅ Rotate encryption keys (security best practice)
- ✅ Review and remove inactive users
- ✅ Backup configuration

**As needed:**
- ✅ Update when security patches released
- ✅ Adjust firewall if attack detected

**MikroTik auto-starts container** - no manual intervention needed after setup!

---

### Q: How do I backup my configuration?

**A:** Multiple layers:

**1. MikroTik config:**
```routeros
/export file=mikrotik-backup
```

**2. Container data:**
```bash
# From your PC
scp -r admin@MIKROTIK_IP:/disk1/3xui ./backup/
```

**3. 3X-UI panel:**
- Panel → Settings → Export/Import
- Download JSON backup

**Restore:**
```routeros
# Upload backup and restore
/container stop [find interface~="3x-ui"]
# Restore root-dir files
/container start [find interface~="3x-ui"]
```

---

## 🎓 Learning Questions

### Q: I'm new to MikroTik, is this too advanced?

**A:** This guide is beginner-friendly!

**Prerequisites:**
- Basic command line knowledge
- Understanding of IP addresses
- Willingness to learn

**We provide:**
- ✅ Step-by-step instructions
- ✅ Complete command examples
- ✅ Detailed troubleshooting
- ✅ Quick setup scripts

**Start here:**
1. Follow [Preparation](docs/01-preparation.md)
2. Use [Quick Setup Script](scripts/quick-setup.rsc)
3. Refer to [Troubleshooting](docs/07-troubleshooting.md) if stuck

---

### Q: Where can I learn more?

**A:** Great resources:

**MikroTik:**
- [Official Wiki](https://wiki.mikrotik.com/)
- [MikroTik Forum](https://forum.mikrotik.com/)
- [YouTube: The Network Berg](https://www.youtube.com/c/TheNetworkBerg)

**3X-UI / Xray:**
- [3X-UI GitHub](https://github.com/MHSanaei/3x-ui)
- [Xray Documentation](https://xtls.github.io/)
- [V2Ray Guide](https://guide.v2fly.org/)

**This Project:**
- [Documentation](docs/)
- [Examples](examples/)
- [GitHub Issues](https://github.com/Pashgen/mikrotik-3xui-vpn/issues)

---

## 💬 Community

### Q: Where can I ask questions?

**A:** Several options:

1. **GitHub Issues** - https://github.com/Pashgen/mikrotik-3xui-vpn/issues
   - Bug reports
   - Feature requests
   - Technical questions

2. **GitHub Discussions** - https://github.com/Pashgen/mikrotik-3xui-vpn/discussions
   - General questions
   - Setup help
   - Share configurations

3. **MikroTik Forum** - https://forum.mikrotik.com/
   - MikroTik-specific questions

---

### Q: How can I contribute?

**A:** Contributions welcome!

**Ways to contribute:**
- 📝 Report bugs
- 💡 Suggest features
- 📖 Improve documentation
- 🔧 Submit fixes
- 🌍 Translate guides
- ⭐ Star the repo!

See [Contributing Guide](CONTRIBUTING.md) for details.

---

**Can't find your question?** [Open an issue](https://github.com/Pashgen/mikrotik-3xui-vpn/issues/new) or [start a discussion](https://github.com/Pashgen/mikrotik-3xui-vpn/discussions/new)!

---

[⬅ Back to README](README.md)
