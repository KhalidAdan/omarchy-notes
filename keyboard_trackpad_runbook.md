# T2 MacBook Keyboard and Trackpad Setup Runbook for Omarchy Linux

## Overview

This runbook covers setting up keyboard and trackpad functionality on T2 MacBooks running Omarchy Linux. Most functionality works automatically after installing T2-specific drivers and restarting the system.

## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- Root/sudo access
- Internet connection for package installation

## Problem Description

Fresh T2 MacBook installations may have non-functional keyboard and trackpad until proper T2 drivers are installed.

## Solution Steps

### Install T2-Specific Drivers

```bash
# Install T2-specific kernel and drivers
sudo pacman -S linux-t2 apple-bce t2fanrd

# Install T2 firmware
sudo pacman -S apple-bcm-firmware

# Verify installation
pacman -Q | grep -E "(linux-t2|apple-bce|t2fanrd)"
```

### Load Apple BCE Module

```bash
# Load the Bridge Control Engine module for T2 devices
sudo modprobe apple-bce

# Verify module loaded
lsmod | grep apple_bce

# Make it load on boot
echo 'apple-bce' | sudo tee /etc/modules-load.d/apple-bce.conf
```

### Restart System

```bash
# Restart to ensure all T2 drivers initialize properly
sudo reboot
```

## Verification

After restart, verify input devices work:

```bash
# Check keyboard device detection
cat /proc/bus/input/devices | grep -A 5 -B 5 -i keyboard

# Check trackpad device detection  
cat /proc/bus/input/devices | grep -A 5 -B 5 -i trackpad

# Test keyboard functionality
# - Type normally - should work
# - Hold any key - should repeat after brief delay

# Test trackpad functionality
# - Move finger on trackpad - cursor should move
# - Tap trackpad - should click
# - Two-finger scroll - should scroll vertically/horizontally
```

## Troubleshooting

### Issue: Input Devices Not Working After Driver Install

**Solution:**
```bash
# Check if apple-bce module loaded
lsmod | grep apple_bce

# If not loaded, load manually
sudo modprobe apple_bce

# Check dmesg for hardware detection
dmesg | grep -i -E "(keyboard|trackpad|apple)"

# Restart if drivers were just installed
sudo reboot
```

### Issue: Trackpad Not Detected

**Solution:**
```bash
# Verify trackpad module is loaded (should happen automatically)
lsmod | grep bcm5974

# Check for trackpad in input devices
ls /dev/input/by-id/ | grep -i trackpad
```

## Notes

- **Restart Required**: Input functionality typically works after reboot following T2 driver installation
- **Automatic Detection**: Both keyboard and trackpad should work without additional configuration
- **Key Repeat**: Fixed automatically after restart with proper drivers
- **Trackpad Gestures**: Basic multi-touch works out of the box after driver installation