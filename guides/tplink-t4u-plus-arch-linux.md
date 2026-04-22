# TP-Link Archer T4U Plus (AC1300) on Arch Linux
### Full Setup Guide: AP Mode, Ethernet Sharing & Security Hardening

> Applies to: Arch Linux, Manjaro, EndeavourOS, Garuda, and any Arch-based distro  
> Adapter: TP-Link Archer T4U Plus (USB, AC1300, RTL8822BU chipset)  
> Tested on: KDE Plasma + NetworkManager + kernel 6.19.x

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Identify Your Interfaces](#step-1--identify-your-interfaces)
4. [Step 2 — Fix the Driver](#step-2--fix-the-driver)
5. [Step 3 — Create the Hotspot](#step-3--create-the-hotspot)
6. [Step 4 — Ethernet Sharing (NAT)](#step-4--ethernet-sharing-nat)
7. [Step 5 — Security Hardening](#step-5--security-hardening)
8. [Step 6 — Verify Persistence](#step-6--verify-persistence)
9. [Troubleshooting](#troubleshooting)

---

## The Problem

The T4U Plus uses the **RTL8822BU** chipset. Linux kernels 5.x and above include an in-kernel driver (`rtw88_8822bu`) for this chipset — but this driver has **broken AP (Access Point) mode for USB devices**. It will load fine, the interface will appear, but any attempt to create a hotspot will fail with:

```
Error: Connection activation failed: 802.1X supplicant took too long to authenticate
```

The fix is to replace it with the **morrownr out-of-tree driver** (`88x2bu`), which is a vendor-derived port with full AP mode support.

---

## Prerequisites

```bash
sudo pacman -S dkms linux-headers base-devel bc git
```

> Make sure your **running kernel version** matches the installed headers:
> ```bash
> uname -r                    # running kernel
> ls /usr/lib/modules/        # installed kernels
> ls /usr/src/ | grep linux   # installed headers
> ```
> If they don't match, reboot into the newer kernel first, or install matching headers from the pacman cache:
> ```bash
> sudo pacman -U /var/cache/pacman/pkg/linux-headers-$(uname -r | sed 's/-/./g' | sed 's/arch/arch/')-x86_64.pkg.tar.zst
> ```

---

## Step 1 — Identify Your Interfaces

```bash
ip link show
iw dev
```

You'll see multiple wireless interfaces. Identify which one is the T4U Plus:

```bash
iw dev
```

Look for the interface on `phy#X` with a **high txpower (15–20 dBm)** — that's your USB adapter. The internal laptop WiFi will show low txpower (~3 dBm).

Example output:
```
phy#4
    Interface wlp0s20f0u3       ← T4U Plus (USB, high txpower)
        type managed
        txpower 20.00 dBm

phy#0
    Interface wlp3s0            ← Internal laptop WiFi (low txpower)
        type managed
        txpower 3.00 dBm
```

Also note your ethernet interface (e.g. `enp4s0`) — this is what you'll be sharing internet from.

---

## Step 2 — Fix the Driver

### 2a — Remove the broken in-kernel driver

```bash
sudo modprobe -r rtw88_8822bu rtw88_8822b rtw88_usb
```

### 2b — Blacklist it permanently

```bash
sudo tee /etc/modprobe.d/rtw88_blacklist.conf << EOF
blacklist rtw88_8822bu
blacklist rtw88_8822b
blacklist rtw88_usb
EOF
```

### 2c — Install the out-of-tree driver

```bash
git clone https://github.com/morrownr/88x2bu-20210702.git
cd 88x2bu-20210702
sudo ./install-driver.sh
```

The script will:
- Compile the driver against your current kernel
- Register it with DKMS (auto-recompiles on kernel updates)
- Prompt you about configuration — you can answer **no** to all optional prompts
- Prompt to reboot — say **yes**

### 2d — Verify after reboot

```bash
lsmod | grep 88x2bu        # should show: 88x2bu  4694016  0
iw dev                     # your USB interface should still be present
```

---

## Step 3 — Create the Hotspot

Replace `YOUR_SSID` and `YOUR_PASSWORD` with your desired values. Use your actual interface name from Step 1 (e.g. `wlp0s20f0u3`).

```bash
sudo nmcli con add \
  type wifi \
  ifname YOUR_INTERFACE \
  con-name MyHotspot \
  autoconnect no \
  ssid "YOUR_SSID" \
  802-11-wireless.mode ap \
  802-11-wireless.band a \
  802-11-wireless.channel 44 \
  802-11-wireless-security.key-mgmt wpa-psk \
  802-11-wireless-security.psk "YOUR_PASSWORD" \
  ipv4.method shared \
  ipv6.method ignore
```

**Key parameters explained:**
| Parameter | Value | Reason |
|---|---|---|
| `band a` | 5GHz | Higher speed, less congestion |
| `channel 44` | 5220 MHz | Stable non-DFS 5GHz channel |
| `ipv4.method shared` | — | NM handles NAT + DHCP automatically |

> **India-specific note:** Allowed 5GHz non-DFS channels are 36, 40, 44, 48. Use one of these to avoid regulatory issues.

### Bring it up

```bash
sudo nmcli con up MyHotspot ifname YOUR_INTERFACE
```

### Verify it's broadcasting

```bash
iw dev YOUR_INTERFACE info
```

Expected output:
```
Interface wlp0s20f0u3
    ssid YOUR_SSID
    type AP
    channel 44 (5220 MHz), width: 20 MHz
    txpower 18.00 dBm
```

---

## Step 4 — Ethernet Sharing (NAT)

`ipv4.method shared` in the profile above already handles everything:

- Starts a **dnsmasq DHCP server** on the hotspot interface
- Enables **IP forwarding** in the kernel
- Adds a **NAT masquerade rule** via nftables

Verify NAT is active:
```bash
sudo nft list ruleset | grep masquerade
```

Expected:
```
ip saddr 10.42.0.0/24 ip daddr != 10.42.0.0/24 masquerade
```

Traffic flow:
```
Internet
   │
[Router] ── Ethernet ── [enp4s0 on your machine]
                                │
                        NM ipv4.method shared
                        (NAT + DHCP server)
                                │
                        [YOUR_INTERFACE - T4U Plus AP]
                                │
                        5GHz WiFi clients (10.42.0.0/24)
```

---

## Step 5 — Security Hardening

### 5a — Assign a static IP to your trusted device

First connect your device to the hotspot, then find its MAC address:

```bash
sudo iw dev YOUR_INTERFACE station dump | grep Station
```

Create a static DHCP lease so it always gets the same IP:

```bash
sudo mkdir -p /etc/NetworkManager/dnsmasq-shared.d

sudo tee /etc/NetworkManager/dnsmasq-shared.d/static.conf << EOF
dhcp-host=YOUR_DEVICE_MAC,10.42.0.10
EOF
```

### 5b — Limit DHCP to 1 device

```bash
sudo tee /etc/NetworkManager/dnsmasq-shared.d/limit.conf << EOF
dhcp-lease-max=1
EOF
```

### 5c — nftables firewall — block all non-trusted traffic

First restart the hotspot to apply the static lease, then add the drop rule:

```bash
sudo nmcli con down MyHotspot && sudo nmcli con up MyHotspot ifname YOUR_INTERFACE
```

Check the nftables table NM created:
```bash
sudo nft list tables
# Should show: table ip nm-shared-YOUR_INTERFACE
```

Insert the whitelist rule:
```bash
sudo nft insert rule ip nm-shared-YOUR_INTERFACE filter_forward \
  ip saddr 10.42.0.0/24 ip saddr != 10.42.0.10 \
  iifname "YOUR_INTERFACE" drop
```

Verify:
```bash
sudo nft list table ip nm-shared-YOUR_INTERFACE | grep drop
```

### 5d — Make the nftables rule persistent (dispatcher script)

NM recreates its nftables table on every hotspot start, so the rule needs to be re-injected each time:

```bash
sudo tee /etc/NetworkManager/dispatcher.d/98-nft-whitelist.sh << 'EOF'
#!/bin/bash
IFACE=$1
EVENT=$2

if [ "$IFACE" = "YOUR_INTERFACE" ] && [ "$EVENT" = "up" ]; then
  sleep 2
  nft insert rule ip nm-shared-YOUR_INTERFACE filter_forward \
    ip saddr 10.42.0.0/24 ip saddr != 10.42.0.10 \
    iifname "YOUR_INTERFACE" drop
fi
EOF

sudo chmod +x /etc/NetworkManager/dispatcher.d/98-nft-whitelist.sh
```

### 5e — MAC address whitelist (kick unknown devices)

```bash
sudo tee /etc/NetworkManager/dispatcher.d/99-mac-whitelist.sh << 'EOF'
#!/bin/bash
IFACE=$1
EVENT=$2

if [ "$IFACE" = "YOUR_INTERFACE" ] && [ "$EVENT" = "up" ]; then
  ALLOWED="YOUR_DEVICE_MAC"
  while true; do
    iw dev YOUR_INTERFACE station dump | grep Station | awk '{print $2}' | while read mac; do
      if [ "$mac" != "$ALLOWED" ]; then
        iw dev YOUR_INTERFACE station del "$mac"
      fi
    done
    sleep 5
  done &
fi
EOF

sudo chmod +x /etc/NetworkManager/dispatcher.d/99-mac-whitelist.sh
```

### 5f — Encrypted DNS via dnscrypt-proxy

Install and enable:
```bash
sudo pacman -S dnscrypt-proxy
sudo systemctl enable --now dnscrypt-proxy
```

Verify it's listening:
```bash
sudo ss -tulpn | grep dnscrypt
# Should show port 53 on 127.0.0.1
```

Point dnsmasq at it:
```bash
sudo tee /etc/NetworkManager/dnsmasq-shared.d/dnscrypt.conf << EOF
no-resolv
server=127.0.0.1#53
EOF
```

### 5g — Enable autoconnect

```bash
sudo nmcli con modify MyHotspot autoconnect yes
```

Restart to apply all changes:
```bash
sudo nmcli con down MyHotspot && sudo nmcli con up MyHotspot ifname YOUR_INTERFACE
```

---

## Step 6 — Verify Persistence

After a full reboot:

```bash
# Hotspot state
sudo nmcli con show MyHotspot | grep GENERAL.STATE

# Interface in AP mode
iw dev YOUR_INTERFACE info

# NAT active
sudo nft list ruleset | grep masquerade

# Drop rule active
sudo nft list table ip nm-shared-YOUR_INTERFACE | grep drop

# dnscrypt running
sudo systemctl status dnscrypt-proxy --no-pager | head -5
```

All should report active/running with no manual intervention needed.

---

## Troubleshooting

### Hotspot fails with "supplicant timeout"
The in-kernel `rtw88_8822bu` is still loaded. Check:
```bash
lsmod | grep rtw88
```
If it appears, the blacklist didn't take effect. Re-check `/etc/modprobe.d/rtw88_blacklist.conf` and run `sudo mkinitcpio -P`.

### Interface not found after reboot
The USB adapter may have gotten a different interface name. Check `iw dev` and update all scripts and NM profile accordingly:
```bash
sudo nmcli con modify MyHotspot 802-11-wireless.cloned-mac-address ""
```

### Driver fails to compile ("kernel headers not found")
Your running kernel and installed headers don't match. Check:
```bash
uname -r
ls /usr/lib/modules/
```
Install the matching headers from pacman cache or reboot into the correct kernel.

### iphone / iOS won't connect
- Avoid invisible/zero-width Unicode SSIDs — iOS strips them
- Use hidden SSID + normal name instead: `802-11-wireless.hidden yes`
- To connect to a hidden network on iOS: **Settings → Wi-Fi → Other...** → enter SSID manually
- Force WPA2 only if connection hangs:
  ```bash
  sudo nmcli con modify MyHotspot \
    802-11-wireless-security.proto rsn \
    802-11-wireless-security.pairwise ccmp \
    802-11-wireless-security.group ccmp
  ```
- If still failing, try a different 5GHz channel (36, 40, 44, 48)

### DNS not going through dnscrypt
```bash
nslookup google.com 127.0.0.1    # should resolve successfully
sudo ss -tulpn | grep dnscrypt   # confirm listening on :53
```
If dnscrypt is on port 5353 instead of 53, update `/etc/NetworkManager/dnsmasq-shared.d/dnscrypt.conf` accordingly.

### DKMS fails on kernel update
```bash
sudo dkms status              # check 88x2bu build status
sudo dkms install 88x2bu/... -k $(uname -r)   # rebuild manually
```

---

## Security Summary

| Layer | Method | Persistent |
|---|---|---|
| Encryption | WPA2-PSK (AES/CCMP) | ✅ NM profile |
| MAC filtering | Dispatcher script kicks unknown MACs | ✅ |
| DHCP limiting | dnsmasq `dhcp-lease-max=1` | ✅ |
| Static IP lease | dnsmasq `dhcp-host` | ✅ |
| Traffic firewall | nftables drop rule via dispatcher | ✅ |
| DNS privacy | dnscrypt-proxy (DoH/DNSCrypt) | ✅ systemd service |
| Frequency | 5GHz only (band a) | ✅ NM profile |

---

## Known Limitations

- **WPA3 not supported** in AP mode with the `88x2bu` driver
- **802.11w (Management Frame Protection)** not available
- **Hidden SSID** may be unreliable depending on driver version — normal SSID is more compatible
- The **MAC whitelist script** uses a polling loop (every 5s) — not event-driven

---

## References

- [morrownr/88x2bu-20210702](https://github.com/morrownr/88x2bu-20210702) — out-of-tree driver
- [NetworkManager dispatcher scripts](https://wiki.archlinux.org/title/NetworkManager#Network_services_with_NetworkManager_dispatcher)
- [dnscrypt-proxy — Arch Wiki](https://wiki.archlinux.org/title/Dnscrypt-proxy)
- [nftables — Arch Wiki](https://wiki.archlinux.org/title/Nftables)
