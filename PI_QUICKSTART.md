# Roomba-GPT – Raspberry Pi 4 Setup Guide (Hardened & Ready)

This guide is divided into two sections:

- **PART A**: System Hardening & Networking (firewall, static IP, VPN)
- **PART B**: Roomba-GPT Project Setup (repo, submodules, Python, brainstem)

---

## 🛡️ PART A: Secure Pi System Setup (Offline/Dev Mode)

### 🔐 1. Initial Lockdown

Disable unused default services:
```bash
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now cups
sudo systemctl disable --now bluetooth
sudo systemctl disable --now triggerhappy
sudo systemctl disable --now wpa_supplicant
```

Replace default user:
```bash
sudo adduser <name>
sudo usermod -aG sudo <name>
sudo deluser pi --remove-home
```

---

### 🧱 2. Configure Firewall (LAN Only)

```
sudo apt install ufw -y

# Allow local-only connections
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/16
sudo ufw enable
```

Optional: block all outgoing traffic except LAN
```
sudo ufw default deny outgoing
sudo ufw allow out to 192.168.10.1
```

---

### 🌐 3. Static IP for Direct Ethernet (eth0 ⇄ eth0)

Edit `/etc/dhcpcd.conf`:
```
interface eth0
static ip_address=192.168.10.2/24
static routers=192.168.10.1
static domain_name_servers=192.168.10.1
```

Then:
```
sudo systemctl restart dhcpcd
```

---

### 🔁 4. VPN Routing Options

#### ✅ Option A: Route Pi traffic through Dev PC’s VPN

On Dev PC:
```
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Forward Pi subnet to VPN interface (e.g., tun0)
sudo iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o tun0 -j MASQUERADE
```

On Pi:
```
sudo ip route add default via 192.168.10.1
```

#### 🔒 Option B: VPN client on the Pi (WireGuard)

```
sudo apt install wireguard resolvconf -y
sudo nano /etc/wireguard/wg0.conf
```

Enable it:
```
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

---

### ✅ 5. Test Connectivity

```
ping 192.168.10.1         # Dev PC
ping 8.8.8.8              # (if routed)
curl ifconfig.me          # (to check VPN)
```

## 🤖 PART B: Roomba-GPT Project Setup

### 📦 1. Clone the Repo and Submodules

```
sudo apt update
sudo apt install git -y

git clone --recurse-submodules git@github.com:TabulaRasaProject/roomba-gpt.git
cd roomba-gpt

# Optional: update GPT-OSS submodule
git submodule update --remote mind/gpt-oss
```

---

### 🐍 2. Python Environment

```
sudo apt install python3 python3-venv python3-pip -y

cd mind
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

### 🧠 3. Upload Firmware (Arduino Mega)

Upload `roomba_oi_bridge.ino` to Arduino Mega.

Wiring summary:
- Roomba TX → Mega RX1 (pin 19)
- Roomba RX → Mega TX1 (pin 18)
- Common GND
- Optional: BRC → Mega D7 (for programmatic wake)

---

### 🔧 4. Roomba Brainstem Control

From the Pi:
```
sudo usermod -aG dialout $USER
newgrp dialout

cd ~/roomba-gpt/body/brainstem
python3 brainstem.py beep
```

Test motion:
```
python3 brainstem.py drive 200 200 1000
```

---

### ⚙️ 5. Troubleshooting

- **“Device busy”**: Close Arduino IDE or Serial Monitor
- **No motion?**
  - Tap Roomba’s `CLEAN` button or send `Q` command
  - Confirm correct baud (115200 default)
  - Check TX/RX crossover and grounded properly

---

### 🔁 6. Submodule Dev Notes

To sync submodules:
```
git submodule update --remote --recursive
git add mind/gpt-oss
git commit -m "update gpt-oss"
git push
```

To reset GPT-OSS if needed:
```
cd mind/gpt-oss
git checkout main
git pull origin main
```

---

Built with ❤️ by Tabula Rasa Project – [github.com/TabulaRasaProject](https://github.com/TabulaRasaProject)
