## Overview

This runbook covers fixing brightness controls on T2 MacBooks running Linux (specifically tested on Omarchy/Hyprland).
## 1. Getting Internal Display Brightness Working

### The Problem

T2 MacBooks often have brightness controls that change values but don't affect actual screen brightness. The Fn+brightness keys may only jump between 0→50→100 instead of smooth control.
### Diagnosis Steps

```bash
# Check what backlight devices exist
ls /sys/class/backlight/

# List all brightness-controllable devices
brightnessctl -l

# Check current status
brightnessctl
```

### The Fix

1. **Test each backlight device manually** to find the one that actually works:

  ```bash
   # Test each device (usually acpi_video0 or apple_backlight)
   echo 0 | sudo tee /sys/class/backlight/acpi_video0/brightness
   echo 50 | sudo tee /sys/class/backlight/acpi_video0/brightness
   echo 100 | sudo tee /sys/class/backlight/acpi_video0/brightness
   ```

2. **Once you find the working device** (in our case `acpi_video0`), force brightnessctl to use it:

 ```bash
   # Test targeted control
   brightnessctl -d acpi_video0 s 50%
   brightnessctl -d acpi_video0 s +10%
   brightnessctl -d acpi_video0 s 10%-
   ```

3. **Update your Hyprland config** to use the correct device:

  ```bash
   # Edit ~/.config/hypr/hyprland.conf
   bind = , XF86MonBrightnessUp, exec, brightnessctl -d acpi_video0 s +5%
   bind = , XF86MonBrightnessDown, exec, brightnessctl -d acpi_video0 s 5%-
   ```

4. **Reload Hyprland config**:

  ```bash
   hyprctl reload
   ```

### Alternative: Kernel Parameter Fix

If the above doesn't work, try forcing the system to use vendor-specific brightness drivers:

**For GRUB users:**

```bash
sudo nano /etc/default/grub
# Change: GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_backlight=vendor"
sudo update-grub && sudo reboot
```

**For Limine users:**

```bash
# Find your Limine config
sudo find /boot -name "limine.cfg"
# Add "acpi_backlight=vendor" to your kernel command line
sudo nano /path/to/limine.cfg
# Reboot
```

### Custom Brightness Script (Optional)

If you want proper notifications instead of chunky overlays:

```bash
# Create ~/.local/bin/brightness.sh
#!/bin/bash
DEVICE="acpi_video0"
case "$1" in
    "up") brightnessctl -d "$DEVICE" s +5% ;;
    "down") brightnessctl -d "$DEVICE" s 5%- ;;
    *) echo "Usage: $0 [up|down]"; exit 1 ;;
esac

CURRENT=$(brightnessctl -d "$DEVICE" get)
MAX=$(brightnessctl -d "$DEVICE" max)
PERCENT=$((CURRENT * 100 / MAX))
notify-send -t 1500 -h int:value:"$PERCENT" "Brightness" "${PERCENT}%"
```

```bash
chmod +x ~/.local/bin/brightness.sh

# Update hyprland.conf
bind = , XF86MonBrightnessUp, exec, ~/.local/bin/brightness.sh up
bind = , XF86MonBrightnessDown, exec, ~/.local/bin/brightness.sh down
```

---

## 2. External Monitor DDC Issues on T2 Macs

### What is DDC?

**DDC (Display Data Channel)** is a communication protocol that allows your computer to control external monitor settings (brightness, contrast, input source, etc.) over the same cable that carries the video signal. It uses I2C (Inter-Integrated Circuit) communication.

### Why DDC Fails on T2 Macs

T2 MacBooks have a security chip (the T2) that handles many hardware functions, including I2C communication. The T2's I2C implementation has known compatibility issues with standard DDC protocols:

- **Bus Access Issues**: The T2 restricts direct I2C bus access
- **Timing Problems**: DDC requires precise timing that the T2 often disrupts
- **Security Restrictions**: The T2 filters I2C communications for security
- **Driver Conflicts**: T2-specific drivers can conflict with standard I2C/DDC drivers

### Attempting DDC on T2 (Usually Fails)

```bash
# Install DDC utilities
sudo pacman -S ddcutil i2c-tools

# Load required modules
sudo modprobe i2c-dev
echo "i2c-dev" | sudo tee /etc/modules-load.d/i2c.conf

# Add user to i2c group (requires logout/login)
sudo usermod -a -G i2c $USER

# Test detection (usually shows "DDC communication failed")
sudo ddcutil detect

# Try forcing communication (rarely works)
sudo ddcutil --force-slave-address --bus=X getvcp 10
sudo ddcutil --sleep-multiplier=2.0 --bus=X getvcp 10
```

### Expected Result

```
Invalid display
   I2C bus:  /dev/i2c-X
   DRM_connector: card1-DP-1
   EDID synopsis: [Monitor details]
   DDC communication failed
```

This is **normal and expected** on T2 Macs. It's not a configuration error.

---

## 3. Gamma-Based Brightness Workaround

Since hardware DDC doesn't work on T2 Macs, use software gamma adjustment for external monitors.

### Method 1: Hyprsunset (Wayland/Hyprland)

```bash
# Install hyprsunset
sudo pacman -S hyprsunset

# Start hyprsunset daemon
hyprsunset &

# Test gamma-based brightness
hyprctl hyprsunset gamma 50   # Dimmer (50%)
hyprctl hyprsunset gamma 150  # Brighter (150%)
hyprctl hyprsunset identity   # Reset to normal (100%)
```

### Method 2: Custom External Monitor Script

```bash
# Create ~/.local/bin/external-brightness.sh
#!/bin/bash
case "$1" in
    "up")
        current=$(hyprctl hyprsunset -j | jq '.gamma // 100')
        new=$((current + 10))
        hyprctl hyprsunset gamma $new
        ;;
    "down")
        current=$(hyprctl hyprsunset -j | jq '.gamma // 100')
        new=$((current - 10))
        if [ $new -lt 10 ]; then new=10; fi
        hyprctl hyprsunset gamma $new
        ;;
    "reset")
        hyprctl hyprsunset identity
        ;;
    *)
        echo "Usage: $0 [up|down|reset]"
        exit 1
        ;;
esac
```

```bash
chmod +x ~/.local/bin/external-brightness.sh

# Add to hyprland.conf for external monitor control
bind = SUPER SHIFT, XF86MonBrightnessUp, exec, ~/.local/bin/external-brightness.sh up
bind = SUPER SHIFT, XF86MonBrightnessDown, exec, ~/.local/bin/external-brightness.sh down
```

### Limitations of Gamma Workaround

- **Not true brightness**: Adjusts color levels, not backlight intensity
- **Affects all monitors**: May dim internal display too (unless per-monitor gamma is configured)
- **Color accuracy**: Can affect color reproduction
- **Power consumption**: Doesn't reduce actual monitor power usage

### Alternative: Physical Controls

The most reliable solution is often using the external monitor's physical brightness controls:

```bash
# Add reminder hotkey to hyprland.conf
bind = SUPER, F1, exec, notify-send "External Brightness" "Use monitor's physical Menu → Brightness controls"
```

---

## Summary

**Internal Display**: Usually fixable by targeting the correct backlight device (`acpi_video0` in most cases)

**External Display**: DDC rarely works on T2 Macs due to T2 security chip limitations. Gamma workarounds or physical controls are the practical solutions.

**Key Takeaway**: T2 Macs work great for internal display brightness once configured, but external monitor DDC is a known limitation of the T2 architecture.