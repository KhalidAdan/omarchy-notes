# T2 MacBook WiFi Setup Runbook for Omarchy Linux

## Overview

This runbook covers resolving WiFi connectivity issues on T2 MacBooks (2018-2020) running Omarchy/Arch Linux. The primary issue is that the WiFi interface (`wlan0`) may not automatically come up after installation, requiring manual intervention.

## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- Root/sudo access
- Physical ethernet connection or USB WiFi adapter for initial setup (if needed)

## Problem Description

After a fresh Omarchy installation on T2 MacBooks, the WiFi interface often remains down despite having the correct drivers and firmware installed. This manifests as:

- `iwctl` commands failing with "No device found"
- WiFi interface not appearing in `ip link` output
- Interface shows as DOWN in system logs

## Solution Steps

### Step 1: Verify T2 Package Installation

First, confirm that all required T2-specific packages are installed:

```bash
# Check for T2-specific packages
pacman -Q | grep -E "(linux-t2|apple-bcm-firmware|t2fanrd)"
```

**Expected output should include:**
- `linux-t2` - T2-specific Linux kernel
- `apple-bcm-firmware` - Broadcom WiFi/Bluetooth firmware for Apple devices
- `t2fanrd` - T2 fan control daemon

If any packages are missing, install them:

```bash
sudo pacman -S linux-t2 apple-bcm-firmware t2fanrd
```

### Step 2: Check RF Kill Status

Verify that WiFi isn't blocked by RF kill switches:

```bash
# List all RF devices and their block status
rfkill list
```

**Expected output:**
```
0: hci0: Bluetooth
    Soft blocked: no
    Hard blocked: no
1: phy0: Wireless LAN
    Soft blocked: no
    Hard blocked: no
```

If WiFi shows as blocked, unblock it:

```bash
# Unblock WiFi if soft blocked
sudo rfkill unblock wifi

# Unblock all if hard blocked (rare)
sudo rfkill unblock all
```

### Step 3: Manually Bring Up WiFi Interface

This is the key step that resolves most WiFi issues on T2 MacBooks:

```bash
# Force the wlan0 interface to come up
sudo ip link set wlan0 up
```

**Verification:**
```bash
# Check interface status - should show UP and RUNNING
ip link show wlan0
```

Expected output:
```
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
```

### Step 4: Scan and Connect to Networks

Once the interface is up, test WiFi functionality:

```bash
# Scan for available networks
iwctl station wlan0 scan

# List discovered networks
iwctl station wlan0 get-networks

# Connect to your network
iwctl station wlan0 connect "YOUR_NETWORK_NAME"
```

When prompted, enter your WiFi password.

### Step 5: Verify Connection

```bash
# Check connection status
iwctl station wlan0 show

# Test connectivity
ping -c 4 8.8.8.8
```

## Making the Fix Permanent

### Method 1: Systemd Service (Recommended)

Create a systemd service to automatically bring up the interface on boot:

```bash
# Create the service file
sudo tee /etc/systemd/system/wlan0-up.service << 'EOF'
[Unit]
Description=Bring up wlan0 interface
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set wlan0 up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Enable the service
sudo systemctl enable wlan0-up.service

# Test the service
sudo systemctl start wlan0-up.service
sudo systemctl status wlan0-up.service
```

### Method 2: NetworkManager Hook

If using NetworkManager, create a dispatcher script:

```bash
# Create dispatcher script
sudo tee /etc/NetworkManager/dispatcher.d/10-wlan0-up << 'EOF'
#!/bin/bash

if [ "$1" = "wlan0" ] && [ "$2" = "pre-up" ]; then
    /sbin/ip link set wlan0 up
fi
EOF

# Make executable
sudo chmod +x /etc/NetworkManager/dispatcher.d/10-wlan0-up
```

### Method 3: Udev Rule

Create a udev rule to automatically bring up the interface when detected:

```bash
# Create udev rule
sudo tee /etc/udev/rules.d/90-wlan0-up.rules << 'EOF'
# Bring up wlan0 interface automatically on T2 MacBooks
SUBSYSTEM=="net", ATTR{address}=="*", NAME=="wlan0", RUN+="/sbin/ip link set %k up"
EOF

# Reload udev rules
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=net
```

## Troubleshooting

### Common Issues and Solutions

#### Issue: "No device found" error from iwctl
**Cause:** Interface is down or driver not loaded
**Solution:**
```bash
# Check if interface exists but is down
ip link show
sudo ip link set wlan0 up

# Check if driver is loaded
lsmod | grep brcm
dmesg | grep -i wifi
```

#### Issue: Interface comes up but can't scan
**Cause:** Firmware loading delay
**Solution:**
```bash
# Wait a moment after bringing interface up
sudo ip link set wlan0 up
sleep 5
iwctl station wlan0 scan
```

#### Issue: Connection drops frequently
**Cause:** Power management interfering
**Solution:**
```bash
# Disable WiFi power management
sudo iw dev wlan0 set power_save off

# Make permanent by adding to startup script
echo 'iw dev wlan0 set power_save off' | sudo tee -a /etc/rc.local
```

#### Issue: Interface doesn't come up on every boot
**Cause:** Timing issues during boot
**Solution:** Use the systemd service method above, or add a delay:

```bash
# Modify the systemd service to add delay
sudo systemctl edit wlan0-up.service
```

Add this content:
```ini
[Service]
ExecStartPre=/bin/sleep 10
```

### Diagnostic Commands

```bash
# Check hardware detection
lspci | grep -i network
lsusb | grep -i wireless

# Check driver status
lsmod | grep brcm
dmesg | grep -E "(brcm|wifi|wlan)"

# Check firmware loading
journalctl | grep -i firmware

# Check NetworkManager status
systemctl status NetworkManager
nmcli device status

# Check interface configuration
ip addr show wlan0
iwconfig wlan0
```

## Advanced Configuration

### Custom WiFi Profile

For automatic connection to specific networks:

```bash
# Create iwctl configuration directory
mkdir -p /var/lib/iwd

# Create network profile (replace with your network details)
sudo tee /var/lib/iwd/YourNetworkName.psk << 'EOF'
[Security]
PreSharedKey=your_wifi_password_hash

[Settings]
AutoConnect=true
EOF

# Set proper permissions
sudo chmod 600 /var/lib/iwd/*.psk
```

### Monitor Mode Support

To enable monitor mode for WiFi analysis:

```bash
# Check if monitor mode is supported
iw list | grep -A 8 "Supported interface modes"

# Enable monitor mode (will disconnect from networks)
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up

# Return to managed mode
sudo ip link set wlan0 down
sudo iw dev wlan0 set type managed
sudo ip link set wlan0 up
```

## Performance Optimization

### Antenna Configuration

Some T2 MacBooks have multiple antennas. You can check and optimize:

```bash
# Check antenna configuration
iw dev wlan0 get antenna

# List available antennas
iw phy phy0 info | grep -A 10 "Available Antennas"

# Set specific antenna (if supported)
# sudo iw dev wlan0 set antenna <tx_antenna> <rx_antenna>
```

### Regulatory Domain

Ensure proper regulatory domain for your region:

```bash
# Check current regulatory domain
iw reg get

# Set regulatory domain (example: US)
sudo iw reg set US

# Make permanent
echo 'WIRELESS_REGDOM="US"' | sudo tee -a /etc/conf.d/wireless-regdom
```

## References

- [T2 Linux Wiki - WiFi](https://wiki.t2linux.org/guides/wifi/)
- [Arch Linux Wireless Configuration](https://wiki.archlinux.org/title/Network_configuration/Wireless)
- [iwd Documentation](https://wiki.archlinux.org/title/Iwd)
- [Omarchy Network Configuration](https://manuals.omamix.org/2/the-omarchy-manual)

## Notes

- **Fresh Install Behavior**: This manual interface activation is typically only needed once after a fresh installation
- **Hardware Compatibility**: This applies specifically to T2 MacBook models with Broadcom WiFi chips
- **Alternative WiFi Managers**: This guide assumes `iwd` (default on Omarchy). For `wpa_supplicant` users, the interface activation step remains the same
- **Security**: Always verify network certificates when connecting to enterprise networks