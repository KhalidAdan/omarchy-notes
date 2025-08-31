This guide covers setting up keyboard backlight controls on T2 MacBooks (2018-2020) running Omarchy/Arch Linux with Hyprland.
## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- T2 Linux drivers properly installed (apple-bce module loaded)
- Hyprland window manager running

## Verification

First, verify your keyboard backlight is detected by the system:

```bash
# Check if keyboard backlight device exists
ls /sys/class/leds/ | grep kbd_backlight
```

Expected output: `apple::kbd_backlight`

If you don't see this, your T2 drivers may not be properly installed. Refer to the [T2 Linux installation guide](https://wiki.t2linux.org/).

## Method 1: Manual Control (Testing)

Test basic functionality with direct sysfs commands:

```bash
# Check current brightness and maximum values
cat /sys/class/leds/apple::kbd_backlight/brightness
cat /sys/class/leds/apple::kbd_backlight/max_brightness

# Turn backlight on to maximum
sudo sh -c 'cat /sys/class/leds/apple::kbd_backlight/max_brightness > /sys/class/leds/apple::kbd_backlight/brightness'

# Turn backlight off
sudo sh -c 'echo 0 > /sys/class/leds/apple::kbd_backlight/brightness'

# Set custom brightness (example: 128 if max is 255)
sudo sh -c 'echo 128 > /sys/class/leds/apple::kbd_backlight/brightness'
```

## Method 2: Using brightnessctl (Recommended)

### Installation

```bash
sudo pacman -S brightnessctl
```

### Usage

```bash
# List all available backlight devices
brightnessctl --list

# Control keyboard backlight specifically
brightnessctl -d apple::kbd_backlight set 100%    # Full brightness
brightnessctl -d apple::kbd_backlight set 50%     # Half brightness
brightnessctl -d apple::kbd_backlight set +10%    # Increase by 10%
brightnessctl -d apple::kbd_backlight set 10%-    # Decrease by 10%
brightnessctl -d apple::kbd_backlight set 0       # Turn off

# Check current status
brightnessctl -d apple::kbd_backlight get
```

## Method 3: Keyboard Shortcuts in Hyprland

### Finding the Right Key Bindings

MacBook Fn+F5/F6 keys can map to different key symbols. Try these methods in order:

#### Option A: XF86 Key Symbols (Try First)

Edit `~/.config/hypr/hyprland.conf`:

```bash
# Add these bindings to your hyprland.conf
bind = , XF86KbdBrightnessDown, exec, brightnessctl -d apple::kbd_backlight set 10%-
bind = , XF86KbdBrightnessUp, exec, brightnessctl -d apple::kbd_backlight set +10%
```

#### Option B: Direct F-Keys with Modifiers

If XF86 symbols don't work:

```bash
# Try Fn modifier
bind = FN, F5, exec, brightnessctl -d apple::kbd_backlight set 10%-
bind = FN, F6, exec, brightnessctl -d apple::kbd_backlight set +10%

# Or try without modifier (some MacBooks send F5/F6 directly)
bind = , F5, exec, brightnessctl -d apple::kbd_backlight set 10%-
bind = , F6, exec, brightnessctl -d apple::kbd_backlight set +10%
```

#### Option C: Key Code Detection (Most Reliable)

If the above methods don't work, use `wev` to detect exact key codes:

```bash
# Install wev if not available
sudo pacman -S wev

# Run wev and press your Fn+F5 and Fn+F6 keys
wev
```

Look for output like:

```
[14:    wl_keyboard] key: serial: 1234, time: 5678, key: 61 (F5), state: 1 (pressed)
```

Note the key codes, then use them in your config:

```bash
# Replace XXX and YYY with actual codes from wev
bind = , code:61, exec, brightnessctl -d apple::kbd_backlight set 10%-
bind = , code:62, exec, brightnessctl -d apple::kbd_backlight set +10%
```

### Apply Configuration

After adding bindings to `~/.config/hypr/hyprland.conf`:

```bash
# Reload Hyprland configuration
hyprctl reload
```

### Test the Shortcuts

Press your configured key combination to test. You should see the keyboard backlight change immediately.

## Method 4: Custom Control Script

For advanced control, create a dedicated script:

```bash
# Create script directory
mkdir -p ~/bin

# Create the script (see full script below)
nano ~/bin/kbd-backlight.sh
```

**Script content:**

```bash
#!/bin/bash
# MacBook T2 Keyboard Backlight Control Script
# Usage: kbd-backlight.sh {on|off|up|down|set <value>|percent <0-100>|status}

KBD_PATH="/sys/class/leds/apple::kbd_backlight"

# Check if backlight exists
if [ ! -d "$KBD_PATH" ]; then
    echo "Error: Keyboard backlight not found at $KBD_PATH"
    exit 1
fi

get_current() { cat "$KBD_PATH/brightness"; }
get_max() { cat "$KBD_PATH/max_brightness"; }

set_brightness() {
    local value=$1
    local max=$(get_max)
    
    # Clamp value between 0 and max
    [ "$value" -lt 0 ] && value=0
    [ "$value" -gt "$max" ] && value=$max
    
    sudo sh -c "echo $value > $KBD_PATH/brightness"
    echo "Keyboard backlight: $value/$max ($(( value * 100 / max ))%)"
}

case "$1" in
    "on"|"max") set_brightness $(get_max) ;;
    "off") set_brightness 0 ;;
    "up")
        current=$(get_current)
        max=$(get_max)
        step=$((max / 10))
        [ "$step" -lt 1 ] && step=1
        set_brightness $((current + step))
        ;;
    "down")
        current=$(get_current)
        max=$(get_max)
        step=$((max / 10))
        [ "$step" -lt 1 ] && step=1
        set_brightness $((current - step))
        ;;
    "set")
        [ -z "$2" ] && { echo "Usage: $0 set <0-$(get_max)>"; exit 1; }
        set_brightness "$2"
        ;;
    "percent")
        [ -z "$2" ] && { echo "Usage: $0 percent <0-100>"; exit 1; }
        value=$(($(get_max) * $2 / 100))
        set_brightness "$value"
        ;;
    "status"|"")
        current=$(get_current)
        max=$(get_max)
        echo "Keyboard backlight: $current/$max ($(( current * 100 / max ))%)"
        ;;
    *)
        echo "Usage: $0 {on|off|up|down|max|set <value>|percent <0-100>|status}"
        ;;
esac
```

Make it executable and use in Hyprland:

```bash
chmod +x ~/bin/kbd-backlight.sh

# Add to hyprland.conf
bind = , XF86KbdBrightnessDown, exec, ~/bin/kbd-backlight.sh down
bind = , XF86KbdBrightnessUp, exec, ~/bin/kbd-backlight.sh up
```

## Method 5: Automatic Brightness Management

### Using hypridle (Omarchy's idle daemon)

Configure automatic keyboard backlight dimming when idle:

```bash
# Edit hypridle configuration
nano ~/.config/hypr/hypridle.conf
```

Add this listener:

```bash
# Turn off keyboard backlight after 2.5 minutes of inactivity
listener {
    timeout = 150
    on-timeout = brightnessctl -d apple::kbd_backlight set 0
    on-resume = brightnessctl -rd apple::kbd_backlight
}
```

### Using systemd timer (Alternative)

Create a systemd service for automatic brightness adjustment:

```bash
# Create user systemd directory
mkdir -p ~/.config/systemd/user

# Create service file
cat > ~/.config/systemd/user/kbd-backlight-idle.service << 'EOF'
[Unit]
Description=Keyboard Backlight Idle Management
After=graphical-session.target

[Service]
Type=simple
ExecStart=/bin/bash -c 'while true; do sleep 300; brightnessctl -d apple::kbd_backlight set 0; done'
Restart=always

[Install]
WantedBy=default.target
EOF

# Enable and start
systemctl --user enable --now kbd-backlight-idle.service
```

## Troubleshooting

### Common Issues

1. **No keyboard backlight device found**
    
    - Verify T2 drivers are installed: `lsmod | grep bce`
    - Check dmesg for apple-bce messages: `dmesg | grep -i bce`
2. **Permission denied when writing to brightness**
    
    - Add user to video group: `sudo usermod -a -G video $USER`
    - Log out and back in for group changes to take effect
3. **Keyboard shortcuts not working**
    
    - Use `wev` to detect actual key codes
    - Check Hyprland config syntax: `hyprctl reload` should not show errors
4. **Backlight turns off immediately**
    
    - Check if another service is controlling brightness
    - Disable conflicting services: `systemctl --user disable xyz.service`

### Verification Commands

```bash
# Check system status
systemctl --user status hypridle
ls -la /sys/class/leds/apple::kbd_backlight/
cat /sys/class/leds/apple::kbd_backlight/brightness
brightnessctl --list | grep kbd

# Test manual control
brightnessctl -d apple::kbd_backlight set 50%
echo $?  # Should return 0 on success
```

## Advanced Configuration

### Custom Brightness Steps

Modify the step size in your scripts or bindings:

```bash
# Large steps (20%)
bind = , XF86KbdBrightnessUp, exec, brightnessctl -d apple::kbd_backlight set +20%

# Small steps (5%)  
bind = SHIFT, XF86KbdBrightnessUp, exec, brightnessctl -d apple::kbd_backlight set +5%
```

### Integration with Status Bars

For Waybar integration:

```json
{
    "custom/kbd-backlight": {
        "exec": "brightnessctl -d apple::kbd_backlight get",
        "interval": 5,
        "format": "󰌌 {}%",
        "on-click": "brightnessctl -d apple::kbd_backlight set +10%",
        "on-click-right": "brightnessctl -d apple::kbd_backlight set 10%-"
    }
}
```

## References

- [T2 Linux Wiki](https://wiki.t2linux.org/)
- [Hyprland Configuration](https://wiki.hypr.land/Configuring/Binds/)
- [brightnessctl Documentation](https://github.com/Hummer12007/brightnessctl)
- [Arch Linux Backlight Guide](https://wiki.archlinux.org/title/Backlight)