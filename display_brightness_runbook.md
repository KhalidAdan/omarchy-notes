# T2 MacBook Display Brightness Control Runbook for Omarchy Linux

## Overview

This runbook fixes display brightness function keys on T2 MacBooks running Omarchy Linux. The issue is that Fn+brightness keys change values but don't affect actual screen brightness, or only provide coarse control (0%→50%→100%) instead of smooth adjustment.

## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- Hyprland window manager running
- Root/sudo access

## Problem Description

T2 MacBooks experience brightness control issues where:
- **Function keys change values but no visual effect** on screen brightness
- **Coarse brightness jumps** instead of smooth 5-10% increments
- **Multiple brightness devices** available but only one actually controls hardware

## Solution Steps

### Step 1: Find the Working Brightness Device

```bash
# Check what backlight devices exist
ls /sys/class/backlight/

# Test each device to find the one that actually works
echo 0 | sudo tee /sys/class/backlight/acpi_video0/brightness
echo 50 | sudo tee /sys/class/backlight/acpi_video0/brightness
echo 100 | sudo tee /sys/class/backlight/acpi_video0/brightness
```

**Watch your screen** - the device that actually changes brightness is your target (usually `acpi_video0`).

### Step 2: Install brightnessctl

```bash
sudo pacman -S brightnessctl
```

### Step 3: Test Targeted Control

```bash
# Test brightness control with the correct device
brightnessctl -d acpi_video0 s 50%
brightnessctl -d acpi_video0 s +10%
brightnessctl -d acpi_video0 s 10%-
```

### Step 4: Configure Hyprland Function Keys

```bash
# Edit Hyprland configuration
nano ~/.config/hypr/hyprland.conf

# Add these brightness key bindings
bind = , XF86MonBrightnessUp, exec, brightnessctl -d acpi_video0 s +5%
bind = , XF86MonBrightnessDown, exec, brightnessctl -d acpi_video0 s 5%-

# Reload configuration
hyprctl reload
```

## Alternative: Kernel Parameter Fix

If device targeting doesn't work, force vendor brightness drivers:

```bash
# For Limine bootloader (Omarchy default)
sudo nano /boot/limine/limine.cfg

# Add "acpi_backlight=vendor" to kernel command line
# Example: KERNEL_CMDLINE=root=UUID=your-uuid rw acpi_backlight=vendor

# Reboot
sudo reboot
```

## Troubleshooting

### Function Keys Not Working
```bash
# Install wev to detect key codes
sudo pacman -S wev

# Run wev and press Fn+F1/F2
wev

# Use detected codes in config:
bind = , code:XXX, exec, brightnessctl -d acpi_video0 s 5%-
bind = , code:YYY, exec, brightnessctl -d acpi_video0 s +5%
```

### No Brightness Devices Found
```bash
# Load required modules
sudo modprobe video backlight acpi_video

# Check dmesg for errors
dmesg | grep -i backlight
```

## Verification

```bash
# Test function keys work
# Press Fn + brightness keys - screen should change smoothly

# Check current brightness
brightnessctl -d acpi_video0

# Manual test
brightnessctl -d acpi_video0 s 25%  # Should dim screen
brightnessctl -d acpi_video0 s 75%  # Should brighten screen
```

## Notes

- **External monitors**: Hardware brightness control not supported on T2 - use physical monitor controls
- **Device targeting**: Always specify `-d acpi_video0` for reliable control
- **Step size**: 5% increments provide good balance of smoothness and responsiveness