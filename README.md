# MikroTik CHR + 3X-UI VPN Container

<div align="center">

![MikroTik](https://img.shields.io/badge/MikroTik-RouterOS_7.x-293239?style=for-the-badge&logo=mikrotik)
![Docker](https://img.shields.io/badge/Docker-Container-2496ED?style=for-the-badge&logo=docker)
![3X-UI](https://img.shields.io/badge/3X--UI-VPN_Panel-00C853?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)

**Complete guide for running 3X-UI VPN panel inside MikroTik RouterOS container**

[Quick Start](#quick-start) • [Documentation](#documentation) • [Troubleshooting](#troubleshooting) • [Features](#features)

</div>

---

## 📋 Overview

This guide provides a **complete, battle-tested solution** for deploying the 3X-UI VPN management panel inside a MikroTik RouterOS 7.x container. Perfect for running VLESS, VMess, Trojan, and Shadowsocks VPN servers directly on your MikroTik router.

### Why This Solution?

- ✅ **All-in-one**: VPN server + router in one device
- ✅ **No external VPS needed**: Runs on your MikroTik hardware
- ✅ **Full Xray support**: VLESS Reality, VMess, Trojan, Shadowsocks
- ✅ **Web management**: Beautiful 3X-UI panel for easy configuration
- ✅ **Production ready**: SSL certificates, firewall rules, NAT configured
- ✅ **Tested on**: MikroTik CHR (Cloud Hosted Router) x86_64

---

## 🎯 Features

- 🚀 **3X-UI Panel** with HTTPS access
- 🔐 **SSL/TLS** support (self-signed or Let's Encrypt)
- 🌐 **Multiple VPN protocols**: VLESS, VMess, Trojan, Shadowsocks
- 🔒 **VLESS Reality** support for advanced censorship circumvention
- 📊 **Traffic monitoring** and user management
- 🛡️ **Firewall rules** and proper NAT configuration
- 📱 **Subscription URLs** for easy client setup
- 🔄 **Automatic Xray updates** via 3X-UI

---

## 📦 Requirements

### Hardware
- **MikroTik RouterOS 7.x** (tested on CHR 7.19)
- **x86_64 architecture** (CHR, RB4011, CCR series)
- **Minimum 512MB RAM** (1GB+ recommended)
- **Minimum 128MB disk space** for container

### Software
- RouterOS 7.x with **Container** package enabled
- Docker on your local machine (for image preparation)
- Basic knowledge of MikroTik CLI

### Network
- **Public IP address** or port forwarding configured
- **Open ports** on your ISP (common VPN ports may be blocked)

---

## 🚀 Quick Start

### 1. Prepare Docker Image

On your **local machine** (Mac/Linux/Windows with Docker):

```bash
# Install skopeo (for proper architecture conversion)
# Mac:
brew install skopeo

# Linux:
sudo apt install skopeo

# Pull 3X-UI image for AMD64 architecture
skopeo copy --override-arch amd64 --override-os linux \
  docker://ghcr.io/mhsanaei/3x-ui:latest \
  docker-archive:$HOME/3x-ui-amd64.tar:ghcr.io/mhsanaei/3x-ui:latest
```

**⚠️ Critical**: Must use `skopeo` with `--override-arch amd64` to avoid ARM64 images on M1/M2 Macs!

### 2. Upload to MikroTik

```bash
# Upload tar file to MikroTik
scp 3x-ui-amd64.tar admin@YOUR_MIKROTIK_IP:/
```

Or use MikroTik WebFig/WinBox to upload to Files.

### 3. Quick Setup Script

Run this on your MikroTik via SSH:

```routeros
# Create network bridge and veth interface
/interface veth add name=3x-ui address=10.10.4.4/24 gateway=10.10.4.1
/interface bridge add name=bridge-3x-ui
/interface bridge port add bridge=bridge-3x-ui interface=3x-ui
/ip address add address=10.10.4.1/24 interface=bridge-3x-ui

# Create container
/container add file=3x-ui-amd64.tar \
  interface=3x-ui \
  root-dir=disk1/3xui \
  dns=8.8.8.8,1.1.1.1 \
  start-on-boot=yes \
  logging=yes

# Configure NAT for container internet access
/ip firewall nat add chain=srcnat action=masquerade \
  src-address=10.10.4.0/24 out-interface=ether1 \
  comment="Container NAT"

# Allow container forwarding
/ip firewall filter add chain=forward action=accept \
  in-interface=bridge-3x-ui place-before=0 \
  comment="Container forward"

# Start container
/container start [find interface~"3x-ui"]
```

### 4. Configure Panel Access

Wait 30 seconds for container to start, then:

```routeros
# Allow panel access (port 8443)
/ip firewall filter add chain=input action=accept \
  protocol=tcp dst-port=8443 place-before=0 \
  comment="3x-ui HTTPS panel"

# NAT panel port
/ip firewall nat add chain=dstnat action=dst-nat \
  to-addresses=10.10.4.4 to-ports=8443 \
  protocol=tcp dst-port=8443 \
  in-interface=ether1 \
  comment="3x-ui panel"
```

### 5. Access Panel

Open in browser:
```
https://YOUR_PUBLIC_IP:8443/
```

**Default credentials:**
- Username: `admin`
- Password: `admin`

**⚠️ Change password immediately after first login!**

---

## 📚 Documentation

Detailed step-by-step guides:

1. [**Preparation**](docs/01-preparation.md) - Docker image preparation and architecture considerations
2. [**Container Setup**](docs/02-container-setup.md) - Installing and configuring the container
3. [**Network Configuration**](docs/03-network-config.md) - Network, NAT, and firewall rules
4. [**SSL Certificates**](docs/04-ssl-certificates.md) - Self-signed and Let's Encrypt certificates
5. [**Panel Access**](docs/05-panel-access.md) - Accessing and securing the web panel
6. [**VPN Inbound Setup**](docs/06-inbound-setup.md) - Configuring VLESS, VMess, Trojan
7. [**Troubleshooting**](docs/07-troubleshooting.md) - Common issues and solutions

### 📖 Additional Guides

- [**VLESS Reality Setup**](examples/vless-reality.md) - Advanced anti-censorship configuration
- [**Port Selection Guide**](docs/port-selection.md) - Choosing ports that won't be blocked
- [**Client Configuration**](docs/client-setup.md) - V2rayN, Shadowrocket, Clash setup
- [**Performance Tuning**](docs/performance.md) - Optimizing for your hardware

---

## 🔧 Configuration Examples

### VLESS with Reality (Recommended)

```json
{
  "protocol": "vless",
  "port": 1443,
  "security": "reality",
  "flow": "xtls-rprx-vision",
  "sni": "google.com",
  "network": "tcp"
}
```

See [VLESS Reality Guide](examples/vless-reality.md) for full configuration.

### Simple VLESS (Testing)

```json
{
  "protocol": "vless",
  "port": 1443,
  "security": "none",
  "network": "tcp"
}
```

### VMess

```json
{
  "protocol": "vmess",
  "port": 1443,
  "security": "none",
  "network": "tcp",
  "alterId": 0
}
```

---

## 🛠️ Troubleshooting

### Container won't start

```routeros
# Check container status
/container print detail

# View logs
/log print where topics~"container"

# Check disk space
/disk print
```

### Panel not accessible

```routeros
# Test from MikroTik
/tool fetch url="https://10.10.4.4:8443/" mode=https check-certificate=no

# Check firewall rules order
/ip firewall filter print where chain=input
```

### VPN not connecting

```bash
# Test port accessibility (from client machine)
nc -zv YOUR_PUBLIC_IP 1443

# Check MikroTik NAT logs
/ip firewall nat set [find dst-port=1443] log=yes
/log print follow where topics~"firewall"
```

See [Troubleshooting Guide](docs/07-troubleshooting.md) for more solutions.

---

## ⚠️ Important Notes

### Architecture Issues
- ⛔ **Never** use `docker save` on M1/M2 Macs without `--platform linux/amd64`
- ✅ **Always** use `skopeo` with `--override-arch amd64` for proper architecture
- 🔍 Verify with: `/container shell [find interface~"3x-ui"]` then `uname -m` (should show `x86_64`)

### Port Blocking
- 🚫 Many ISPs block common VPN ports: 585, 443, 1194, 8388
- ✅ Use uncommon ports: 1443, 2053, 8443, 3000
- 🔍 Test port before setup: `nc -zv YOUR_IP PORT`

### Firewall Rules
- ⚠️ Rule **order matters** - accept rules must come BEFORE drop rules
- ✅ Use `place-before=0` or specific position numbers
- 🔍 Check order: `/ip firewall filter print where chain=input`

### SSL Certificates
- 🔒 Let's Encrypt requires port 80 open (often blocked by ISPs)
- ✅ Self-signed certificates work fine for most use cases
- 📱 Mobile clients may require adding certificate to trust store

---

## 🎓 Learning Resources

### MikroTik Container Documentation
- [Official Container Guide](https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container)
- [RouterOS Container Examples](https://wiki.mikrotik.com/wiki/Manual:Container)

### 3X-UI Documentation
- [Official 3X-UI GitHub](https://github.com/MHSanaei/3x-ui)
- [3X-UI Wiki](https://github.com/MHSanaei/3x-ui/wiki)

### Xray Protocol Documentation
- [Xray Documentation](https://xtls.github.io/)
- [VLESS Protocol](https://xtls.github.io/config/inbounds/vless.html)
- [Reality Protocol](https://github.com/XTLS/REALITY)

---

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request. For major changes:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

---

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- [MikroTik](https://mikrotik.com) - For RouterOS and container support
- [3X-UI](https://github.com/MHSanaei/3x-ui) - Excellent Xray management panel
- [Xray-core](https://github.com/XTLS/Xray-core) - Powerful proxy framework
- Community testers and contributors

---

## 💬 Support

- 📖 Check [Documentation](docs/) first
- 🐛 [Open an Issue](https://github.com/Pashgen/mikrotik-3xui-vpn/issues) for bugs
- 💡 [Discussions](https://github.com/Pashgen/mikrotik-3xui-vpn/discussions) for questions
- ⭐ Star this repo if it helped you!

---

<div align="center">

**Made with ❤️ for the MikroTik community**

[⬆ Back to Top](#mikrotik-chr--3x-ui-vpn-container)

</div>
