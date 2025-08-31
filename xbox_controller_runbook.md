# T2 MacBook Xbox Controller Bluetooth Pairing Runbook for Omarchy Linux

## Overview

This runbook addresses Xbox controller Bluetooth pairing issues on T2 MacBooks running Omarchy Linux. The primary challenge is that standard pairing procedures often fail, requiring a specific sequence of trust, connect, and pair commands to establish a stable connection.

## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- Xbox Wireless Controller (any generation)
- Working Bluetooth connectivity
- Root/sudo access

## Problem Description

Standard Xbox controller pairing methods frequently fail on T2 MacBooks with symptoms including:

- **Controller not discoverable** during standard pairing attempts
- **Connection timeouts** when using typical pair→connect sequence
- **Bluetoothctl interface issues** showing keyboard interference or unusual prompts
- **Successful scanning** but failed pairing/connection

## Solution Steps

### Step 1: Fix Bluetooth Service Issues

If `bluetoothctl` exhibits strange behavior, restart the Bluetooth service:

```bash
# Restart Bluetooth service to clear any state issues
sudo systemctl restart bluetooth

# Verify service is running properly
systemctl status bluetooth

# Check for any error messages
journalctl -u bluetooth --since "5 minutes ago"
```

**Signs you need this step:**
- `bluetoothctl` shows keyboard prompts instead of Bluetooth prompts
- Interface appears frozen or unresponsive
- Connection attempts immediately fail

### Step 2: Prepare Controller for Pairing

Put your Xbox controller into pairing mode:

```bash
# Physical button sequence on controller:
# 1. Hold Xbox button (center logo) until it lights up
# 2. Hold pairing button (small button on top edge near USB port) 
# 3. Keep both pressed until Xbox button flashes rapidly
# 4. Release both buttons - Xbox button should continue flashing rapidly
```

**Visual confirmation:**
- Xbox button flashes rapidly (approximately 2-3 times per second)
- Controller remains in pairing mode for about 20 seconds
- If button stops flashing, repeat the sequence

### Step 3: Discover Controller MAC Address

Use `bluetoothctl` to scan and identify your controller:

```bash
# Launch bluetoothctl
bluetoothctl

# Enable scanning
scan on

# Wait 10-15 seconds and look for Xbox controller
# Example output:
# [NEW] Device XX:XX:XX:XX:XX:XX Xbox Wireless Controller
# or
# [NEW] Device XX:XX:XX:XX:XX:XX Wireless Controller
```

**Important:** Note the complete MAC address (XX:XX:XX:XX:XX:XX) - you'll need this for the next steps.

### Step 4: Critical Pairing Sequence

This is the key step that differs from standard Bluetooth pairing. Execute commands in this exact order:

```bash
# Still in bluetoothctl, execute in this SPECIFIC ORDER:

# 1. TRUST the device first (non-standard but required)
trust XX:XX:XX:XX:XX:XX

# 2. CONNECT to the device (before pairing)
connect XX:XX:XX:XX:XX:XX

# 3. PAIR with the device (last step)
pair XX:XX:XX:XX:XX:XX
```

**Expected output:**
```
[bluetooth]# trust XX:XX:XX:XX:XX:XX
[CHG] Device XX:XX:XX:XX:XX:XX Trusted: yes
Changing XX:XX:XX:XX:XX:XX trust succeeded

[bluetooth]# connect XX:XX:XX:XX:XX:XX
Attempting to connect to XX:XX:XX:XX:XX:XX
[CHG] Device XX:XX:XX:XX:XX:XX Connected: yes
Connection successful

[bluetooth]# pair XX:XX:XX:XX:XX:XX
Attempting to pair with XX:XX:XX:XX:XX:XX
[CHG] Device XX:XX:XX:XX:XX:XX Paired: yes
Pairing successful
```

### Step 5: Verify Connection

```bash
# Check device info
info XX:XX:XX:XX:XX:XX

# Expected to see:
# Connected: yes
# Trusted: yes
# Paired: yes

# Exit bluetoothctl
exit

# Test controller input
jstest /dev/input/js0
# or
evtest /dev/input/event*
```

### Step 6: Configure Auto-Connection (Optional)

To make the controller automatically connect when turned on:

```bash
# Re-enter bluetoothctl
bluetoothctl

# Ensure auto-connection is enabled
trust XX:XX:XX:XX:XX:XX  # (if not already done)

# Some controllers benefit from this additional setting
experimental on
```

## Advanced Driver Installation (Optional)

For enhanced compatibility and features, install the `xpadneo` driver:

```bash
# Install xpadneo from AUR for better driver support
paru -S xpadneo-dkms
# or
yay -S xpadneo-dkms

# If using manjaro or other non-arch based systems:
sudo pacman -S linux-headers dkms
git clone https://github.com/atar-axis/xpadneo.git
cd xpadneo
sudo ./install.sh

# Verify installation
dkms status
lsmod | grep hid_xpadneo
```

**Benefits of xpadneo:**
- Better analog stick precision
- Proper trigger support
- Improved wireless stability
- Enhanced haptic feedback
- Better battery reporting

## Troubleshooting

### Issue: Controller Not Discoverable

**Symptoms:**
- No device appears during scanning
- Controller flashing but not found

**Solutions:**

```bash
# Reset Bluetooth adapter
sudo rfkill block bluetooth
sleep 2
sudo rfkill unblock bluetooth
sleep 5

# Restart Bluetooth service
sudo systemctl restart bluetooth

# Clear Bluetooth cache
sudo rm -rf /var/lib/bluetooth/*/cache

# Try different scanning approach
bluetoothctl
agent on
default-agent
discoverable on
pairable on
scan on
```

### Issue: Pairing Succeeds but Controller Doesn't Work

**Symptoms:**
- Shows as connected in bluetoothctl
- No input detected in games or jstest

**Solutions:**

```bash
# Check if controller is recognized as input device
ls /dev/input/js*
ls /dev/input/event*

# Check kernel messages
dmesg | grep -i xbox
dmesg | grep -i gamepad

# Verify input subsystem recognizes controller
cat /proc/bus/input/devices | grep -A 5 -B 5 Xbox

# If not recognized, try forcing driver load
sudo modprobe xpad
sudo modprobe joydev
```

### Issue: Connection Drops Frequently

**Symptoms:**
- Controller connects then disconnects
- Intermittent input loss

**Solutions:**

```bash
# Disable USB autosuspend for Bluetooth adapter
# Find Bluetooth USB device
lsusb | grep -i bluetooth

# Create udev rule to disable autosuspend
sudo tee /etc/udev/rules.d/50-bluetooth-autosuspend.rules << 'EOF'
# Disable autosuspend for Bluetooth devices
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="05ac", ATTR{idProduct}=="8290", TEST=="power/control", ATTR{power/control}="on"
EOF

# Reload udev rules
sudo udevadm control --reload-rules

# Increase Bluetooth supervision timeout
sudo tee -a /etc/bluetooth/main.conf << 'EOF'

[General]
# Increase connection supervision timeout
SupervisionTimeout=5000
EOF

# Restart Bluetooth
sudo systemctl restart bluetooth
```

### Issue: Multiple Controllers Interfering

**Symptoms:**
- First controller works, second doesn't
- Controllers randomly switching player numbers

**Solutions:**

```bash
# Check connected controllers
bluetoothctl devices

# List all input devices
for dev in /dev/input/js*; do
    echo "=== $dev ==="
    udevadm info --query=all --name=$dev | grep ID_
done

# For multiple controllers, ensure each has unique trust/pairing
# Repeat pairing process for each controller individually
```

### Issue: Audio Through Controller Not Working

**Symptoms:**
- Controller connects and works for input
- No audio through controller headphone jack

**Solutions:**

```bash
# Check if audio profile is available
pactl list cards | grep -A 10 -B 5 xbox

# Install additional audio support
sudo pacman -S pipewire-alsa pipewire-pulse

# For some Xbox controllers with audio:
# Create audio profile (this is rare and controller-specific)
# Most Xbox controllers don't support audio over Bluetooth
```

## Controller-Specific Notes

### Xbox Wireless Controller (2020+)
- **Best compatibility** with the pairing sequence above
- **USB-C charging** - can pair while charging
- **Share button** - may require xpadneo for full functionality

### Xbox One Controller (2016-2019)
- **Good compatibility** - follows standard pairing sequence
- **Micro-USB charging** - can pair while charging
- **Requires** the trust→connect→pair sequence for reliable connection

### Xbox One Controller (Original 2013-2015)
- **No Bluetooth support** - requires USB connection or Xbox Wireless Adapter
- **USB only** - will work immediately when plugged in
- **Wired reliability** - most stable option for older controllers

### Xbox Elite Controllers
- **Same pairing process** as standard controllers
- **Additional buttons** - may require xpadneo for full mapping
- **Paddle support** - works out of box with xpadneo driver

## Performance Optimization

### Reduce Input Lag

```bash
# Optimize Bluetooth latency
sudo tee /etc/bluetooth/input.conf << 'EOF'
[General]
# Optimize for gaming
IdleTimeout=30
# Reduce connection interval for lower latency
UserspaceHID=true
EOF

# Restart Bluetooth
sudo systemctl restart bluetooth
```

### Battery Life Optimization

```bash
# Enable power saving when idle (requires xpadneo)
echo 1 | sudo tee /sys/module/hid_xpadneo/parameters/motor_rumble_on_mode_change

# Check battery status (if supported)
cat /sys/class/power_supply/xbox_controller_battery_*/capacity
```

### Multiple Controller Setup

```bash
# For local multiplayer gaming, pair multiple controllers:
# 1. Pair first controller following the sequence above
# 2. Turn off first controller
# 3. Pair second controller with same sequence
# 4. Repeat for up to 4 controllers
# 5. Turn on all controllers - they should connect automatically

# Verify multiple controllers
ls /dev/input/js*
# Should show js0, js1, js2, js3 for four controllers

# Test multiple inputs
for i in /dev/input/js*; do
    echo "Testing $i"
    jstest --select "$i" &
done
```

## Gaming Integration

### Steam Controller Support

```bash
# Ensure Steam recognizes Xbox controllers
# Steam should automatically detect properly paired controllers
# Enable Steam Input for additional customization

# For native Steam support, disable xpadneo temporarily if needed:
sudo modprobe -r hid_xpadneo
# Use built-in kernel driver
```

### RetroArch Configuration

```bash
# Install RetroArch
sudo pacman -S retroarch

# Configure Xbox controller in RetroArch:
# 1. Launch RetroArch
# 2. Go to Settings → Input → Port 1 Controls
# 3. Set Device Index to your controller
# 4. Bind controls manually if auto-detection fails
```

### Native Game Support

```bash
# Most native Linux games automatically recognize Xbox controllers
# For games that don't, create a symlink:
sudo ln -sf /dev/input/js0 /dev/input/xbox_controller

# Some games look for specific event device:
ls -la /dev/input/by-id/ | grep -i xbox
```

## Verification Checklist

Use this checklist to verify successful controller setup:

- [ ] **Bluetooth service restarted** without errors
- [ ] **Controller enters pairing mode** (rapid flashing Xbox button)
- [ ] **Controller appears in scan** results with correct name
- [ ] **Trust command succeeds** - shows "Changing trust succeeded"
- [ ] **Connect command succeeds** - shows "Connection successful"  
- [ ] **Pair command succeeds** - shows "Pairing successful"
- [ ] **Controller info shows** Connected: yes, Trusted: yes, Paired: yes
- [ ] **Input device created** - /dev/input/js0 exists
- [ ] **jstest shows input** - buttons and sticks respond
- [ ] **Controller auto-connects** when turned on after pairing

## Diagnostic Commands

```bash
# Check Bluetooth status and devices
bluetoothctl show
bluetoothctl devices
bluetoothctl info XX:XX:XX:XX:XX:XX

# Check input devices
ls -la /dev/input/
cat /proc/bus/input/devices

# Monitor Bluetooth events
journalctl -f | grep -i bluetooth

# Test controller input
jstest /dev/input/js0
evtest  # Select Xbox controller from list

# Check driver status
lsmod | grep -E "(xpad|hid_xpadneo|joydev)"
dmesg | grep -i xbox
```

## Recovery Procedures

### Complete Controller Reset

If pairing becomes completely broken:

```bash
# Remove controller from Bluetooth completely
bluetoothctl
remove XX:XX:XX:XX:XX:XX
exit

# Clear Bluetooth cache
sudo systemctl stop bluetooth
sudo rm -rf /var/lib/bluetooth/*/cache
sudo systemctl start bluetooth

# Reset controller hardware:
# Hold Xbox + Menu + View buttons for 6 seconds
# Controller will turn off - turn back on and re-pair
```

### Bluetooth Stack Reset

For persistent Bluetooth issues:

```bash
# Complete Bluetooth reset
sudo systemctl stop bluetooth
sudo rfkill block bluetooth
sleep 5
sudo rfkill unblock bluetooth
sudo systemctl start bluetooth

# If still having issues, reload Bluetooth modules
sudo modprobe -r btusb
sudo modprobe -r bluetooth
sudo modprobe bluetooth
sudo modprobe btusb
```

## References

- [Xbox Controller Support on Linux](https://wiki.archlinux.org/title/Gamepad#Xbox_controllers)
- [xpadneo GitHub Repository](https://github.com/atar-axis/xpadneo)
- [Bluetooth Gaming Controllers Guide](https://wiki.archlinux.org/title/Bluetooth#Troubleshooting)
- [T2 Linux Hardware Compatibility](https://wiki.t2linux.org/guides/bluetooth/)
- [Linux Input Subsystem Documentation](https://www.kernel.org/doc/Documentation/input/)

## Notes

- **Pairing Sequence Critical**: The trust→connect→pair order is essential for T2 MacBooks
- **Service Restart**: Bluetooth service restart often resolves interface issues
- **Driver Enhancement**: xpadneo provides significant improvements but isn't required for basic functionality
- **Multiple Controllers**: Each controller must be paired individually using the same sequence
- **Battery Life**: Bluetooth Xbox controllers have excellent battery life (20+ hours typical)
- **Compatibility**: This method works with all Bluetooth-capable Xbox controllers