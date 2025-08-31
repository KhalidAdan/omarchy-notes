# T2 MacBook Clamshell Mode Setup Runbook for Omarchy Linux

## Overview

This runbook provides complete setup for macOS-style clamshell behavior on T2 MacBooks running Omarchy Linux with Hyprland. Out of the box, T2 MacBooks don't have proper clamshell mode functionality. This guide configures true macOS-like behavior where closing the lid with an external monitor connected continues operation on the external display only.

## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- Hyprland window manager running
- External monitor, keyboard, and mouse for testing
- Root/sudo access
- Internet connection for package installation

## Target Behavior

This configuration achieves:

- **Lid closed + external monitor connected** = continues on external display only
- **Lid closed + no external monitor** = laptop suspends
- **Lid opened with external monitor** = dual display mode
- **Seamless transitions** between clamshell and dual display modes

## Step 1: Install Dependencies

### Install Rust Development Environment

```bash
# Check if Rust is already installed
cargo --version

# If Rust is not installed, use Omarchy's package manager:
# Super + Alt + Space > Install > Development > Rust

# Or install manually:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Verify installation
cargo --version
rustc --version
```

### Install ACPI Daemon (Critical Component)

```bash
# Install acpid for hardware lid event detection
sudo pacman -S acpid

# Enable acpid service to start on boot
sudo systemctl enable acpid

# Start acpid service immediately
sudo systemctl start acpid

# Verify acpid is running properly
systemctl status acpid
```

**Expected output:**
```
● acpid.service - ACPI event daemon
     Loaded: loaded (/usr/lib/systemd/system/acpid.service; enabled; preset: disabled)
     Active: active (running) since [timestamp]
```

### Install GTK4 Dependencies

```bash
# Install GTK4 required for hyprdock
sudo pacman -S gtk4

# Verify GTK4 installation
pkg-config --modversion gtk4
```

### Install hyprdock via Cargo

```bash
# Install hyprdock (bypasses potential AUR packaging issues)
cargo install hyprdock

# Verify installation
~/.cargo/bin/hyprdock --version
```

## Step 2: Configure System Lid Behavior

### Disable System Lid Handling

```bash
# Edit systemd logind configuration
sudo cp /etc/systemd/logind.conf /etc/systemd/logind.conf.backup

sudo tee /etc/systemd/logind.conf << 'EOF'
#  This file is part of systemd.
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
#HandlePowerKey=poweroff
#HandleSuspendKey=suspend
#HandleHibernateKey=hibernate
#PowerKeyIgnoreInhibited=no
#SuspendKeyIgnoreInhibited=no
#HibernateKeyIgnoreInhibited=no
#LidSwitchIgnoreInhibited=yes
#IdleAction=ignore
#IdleActionSec=30min
EOF
```

### Restart Login Manager

```bash
# Restart systemd-logind to apply changes
sudo systemctl restart systemd-logind

# Verify the service restarted without errors
systemctl status systemd-logind
```

## Step 3: Configure hyprdock

### Create Configuration Directory

```bash
# Create hyprdock configuration directory
mkdir -p ~/.config/hyprdock
```

### Create hyprdock Configuration

```bash
# Create comprehensive hyprdock configuration
cat > ~/.config/hyprdock/config.toml << 'EOF'
# hyprdock configuration for T2 MacBook clamshell mode
monitor_name = "eDP-1"
default_external_mode = "extend"
css_string = ""

[init_command]
base = ""
args = []

[open_bar_command]
base = ""
args = []

[close_bar_command]
base = ""
args = []

[reload_bar_command]
base = ""
args = []

[suspend_command]
base = "systemctl"
args = ["suspend"]

[lock_command]
base = "hyprlock"
args = []

[utility_command]
base = "playerctl"
args = ["--all-players", "-a", "pause"]

[get_monitors_command]
base = "hyprctl"
args = ["monitors"]

[enable_internal_monitor_command]
base = "hyprctl"
args = ["keyword", "monitor", "eDP-1,preferred,auto,1"]

[disable_internal_monitor_command]
base = "hyprctl"
args = ["keyword", "monitor", "eDP-1,disable"]

[enable_external_monitor_command]
base = "hyprctl"
args = ["keyword", "monitor", ",preferred,auto,1"]

[disable_external_monitor_command]
base = "hyprctl"
args = ["keyword", "monitor", ",disable"]

[extend_command]
base = "hyprctl"
args = ["keyword", "monitor", ",preferred,1920x0,1"]

[mirror_command]
base = "hyprctl"
args = ["keyword", "monitor", ",preferred,0x0,1"]

[wallpaper_command]
base = "hyprctl"
args = ["dispatch", "hyprpaper"]
EOF
```

### Verify Configuration File

```bash
# Check configuration file was created correctly
cat ~/.config/hyprdock/config.toml

# Ensure hyprdock can read the configuration
~/.cargo/bin/hyprdock --help
```

## Step 4: Configure Hyprland Integration

### Add hyprdock to Hyprland Autostart

```bash
# Create backup of existing Hyprland config
cp ~/.config/hypr/hyprland.conf ~/.config/hypr/hyprland.conf.backup

# Add hyprdock autostart to Hyprland configuration
echo "" >> ~/.config/hypr/hyprland.conf
echo "# hyprdock clamshell mode daemon" >> ~/.config/hypr/hyprland.conf
echo "exec-once = ~/.cargo/bin/hyprdock -s" >> ~/.config/hypr/hyprland.conf
```

### Verify PATH Configuration

```bash
# Ensure cargo binaries are in PATH
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.zshrc

# Source the updated PATH
source ~/.bashrc  # or source ~/.zshrc if using zsh

# Verify hyprdock is accessible
which hyprdock
hyprdock --version
```

## Step 5: Apply and Test Configuration

### Restart Hyprland

```bash
# Method 1: Use Hyprland restart shortcut
# Super + Escape, then select "Relaunch"

# Method 2: Manual restart
hyprctl dispatch exit
# Log back in to restart Hyprland with new configuration
```

### Verify hyprdock is Running

```bash
# Check if hyprdock process is running
pgrep -f hyprdock
ps aux | grep hyprdock

# Check hyprdock logs for any errors
journalctl --user -f | grep hyprdock &
```

## Step 6: Test Clamshell Functionality

### Comprehensive Test Sequence

1. **Setup Test Environment:**
   ```bash
   # Connect external monitor and peripherals
   # Verify both displays are detected
   hyprctl monitors
   ```

2. **Test Dual Display Mode:**
   ```bash
   # With lid open, both displays should be active
   # Move windows between displays to verify functionality
   ```

3. **Test Clamshell Mode:**
   ```bash
   # Close laptop lid with external monitor connected
   # System should continue running on external monitor only
   # Internal display should turn off
   ```

4. **Test Resume to Dual Display:**
   ```bash
   # Open laptop lid
   # Both displays should become active again
   # Windows should be accessible on both screens
   ```

5. **Test Suspend Behavior:**
   ```bash
   # Disconnect external monitor
   # Close laptop lid
   # System should suspend
   ```

### Verification Commands

```bash
# Check current monitor status
hyprctl monitors

# Monitor lid events in real-time
journalctl -f | grep -i lid

# Check acpid events
sudo journalctl -u acpid -f

# Test manual hyprdock commands
hyprdock --help
```

## Troubleshooting

### Issue: hyprdock Not Starting

**Symptoms:**
- No hyprdock process running
- Clamshell mode not working

**Diagnosis:**
```bash
# Check if hyprdock binary exists
ls -la ~/.cargo/bin/hyprdock

# Try running hyprdock manually
~/.cargo/bin/hyprdock -s

# Check for error messages
journalctl --user | grep hyprdock
```

**Solutions:**
```bash
# Reinstall hyprdock if binary is missing
cargo install hyprdock --force

# Check PATH configuration
echo $PATH | grep -o "$HOME/.cargo/bin"

# Add PATH if missing
export PATH="$HOME/.cargo/bin:$PATH"
```

### Issue: Lid Events Not Detected

**Symptoms:**
- Lid close/open doesn't trigger any actions
- No lid events in logs

**Diagnosis:**
```bash
# Check acpid status
systemctl status acpid

# Monitor ACPI events manually
sudo journalctl -u acpid -f
# Open and close lid to see if events appear
```

**Solutions:**
```bash
# Restart acpid service
sudo systemctl restart acpid

# Check ACPI lid device exists
ls /proc/acpi/button/lid/

# Test lid switch manually
cat /proc/acpi/button/lid/LID0/state
```

### Issue: External Monitor Not Detected

**Symptoms:**
- Clamshell mode doesn't activate with external monitor
- Monitor shows as disconnected

**Diagnosis:**
```bash
# Check connected monitors
hyprctl monitors
xrandr  # if available

# Check display connection
cat /sys/class/drm/card*/status
```

**Solutions:**
```bash
# Verify cable connection and power
# Try different cable or port if available

# Force monitor detection
hyprctl dispatch dpms off
sleep 2
hyprctl dispatch dpms on

# Check monitor configuration in Hyprland
grep -A 5 -B 5 "monitor" ~/.config/hypr/hyprland.conf
```

### Issue: System Suspends with External Monitor Connected

**Symptoms:**
- System suspends when lid closes even with external monitor
- Clamshell mode not activating

**Diagnosis:**
```bash
# Check logind configuration
cat /etc/systemd/logind.conf | grep -E "HandleLid"

# Verify hyprdock configuration
cat ~/.config/hyprdock/config.toml | grep monitor_name
```

**Solutions:**
```bash
# Ensure logind ignores lid switches
sudo systemctl restart systemd-logind

# Check monitor name in hyprdock config
hyprctl monitors | grep Monitor
# Update config.toml if monitor name differs from "eDP-1"
```

### Issue: Dual Display Mode Not Working

**Symptoms:**
- Only external monitor active when lid opens
- Internal display stays disabled

**Diagnosis:**
```bash
# Check current monitor configuration
hyprctl monitors

# Test manual monitor enable
hyprctl keyword monitor "eDP-1,preferred,auto,1"
```

**Solutions:**
```bash
# Verify hyprdock enable_internal_monitor_command
cat ~/.config/hyprdock/config.toml | grep -A 2 enable_internal

# Test commands manually
hyprctl keyword monitor "eDP-1,preferred,auto,1"
hyprctl keyword monitor ",preferred,1920x0,1"
```

## Advanced Configuration

### Custom Monitor Layout

For specific monitor arrangements:

```bash
# Edit hyprdock config for custom layout
nano ~/.config/hyprdock/config.toml

# Example: External monitor to the left of internal
# Modify extend_command:
[extend_command]
base = "hyprctl"
args = ["keyword", "monitor", ",preferred,-1920x0,1"]
```

### Integration with Status Bars

For Waybar integration:

```bash
# Add clamshell status indicator to Waybar
cat >> ~/.config/waybar/config << 'EOF'
"custom/clamshell": {
    "exec": "if pgrep -x hyprdock > /dev/null; then echo '🖥️'; else echo '💻'; fi",
    "interval": 5,
    "tooltip-text": "Clamshell mode status"
}
EOF
```

### Power Management Integration

```bash
# Create power profile for clamshell mode
sudo tee /etc/udev/rules.d/99-clamshell-power.rules << 'EOF'
# Optimize power settings for clamshell mode
SUBSYSTEM=="power_supply", ATTR{type}=="Mains", ATTR{online}=="1", RUN+="/usr/bin/cpupower frequency-set -g performance"
SUBSYSTEM=="power_supply", ATTR{type}=="Mains", ATTR{online}=="0", RUN+="/usr/bin/cpupower frequency-set -g powersave"
EOF

sudo udevadm control --reload-rules
```

## Performance Optimization

### Reduce Mode Switching Latency

```bash
# Optimize hyprdock responsiveness
cat > ~/.config/hyprdock/config.toml.optimized << 'EOF'
monitor_name = "eDP-1"
default_external_mode = "extend"
css_string = ""

# Faster suspend command
[suspend_command]
base = "systemctl"
args = ["suspend", "--no-block"]

# Faster monitor commands
[enable_internal_monitor_command]
base = "hyprctl"
args = ["--batch", "keyword monitor eDP-1,preferred,auto,1"]

[disable_internal_monitor_command]
base = "hyprctl" 
args = ["--batch", "keyword monitor eDP-1,disable"]
EOF
```

### Background Process Optimization

```bash
# Ensure hyprdock runs with appropriate priority
echo 'exec-once = nice -n -5 ~/.cargo/bin/hyprdock -s' >> ~/.config/hypr/hyprland.conf
```

## File Locations Reference

- **hyprdock configuration**: `~/.config/hyprdock/config.toml`
- **Hyprland configuration**: `~/.config/hypr/hyprland.conf`
- **System lid configuration**: `/etc/systemd/logind.conf`
- **hyprdock binary**: `~/.cargo/bin/hyprdock`
- **ACPI configuration**: `/etc/acpi/`

## Verification Checklist

- [ ] **Rust installed** and `cargo --version` works
- [ ] **acpid service** enabled and running
- [ ] **GTK4 installed** successfully
- [ ] **hyprdock binary** exists at `~/.cargo/bin/hyprdock`
- [ ] **System lid switches** set to ignore in `/etc/systemd/logind.conf`
- [ ] **hyprdock configuration** file created with correct monitor name
- [ ] **Hyprland autostart** includes hyprdock with `-s` flag
- [ ] **PATH includes** `~/.cargo/bin` directory
- [ ] **External monitor** detected by `hyprctl monitors`
- [ ] **Lid close with external monitor** continues on external display only
- [ ] **Lid open with external monitor** enables dual display mode
- [ ] **Lid close without external monitor** suspends system

## References

- [hyprdock GitHub Repository](https://github.com/Xetibo/hyprdock)
- [hyprdock Crates.io Package](https://crates.io/crates/hyprdock)
- [Hyprland Wiki - Monitor Configuration](https://wiki.hypr.land/Configuring/Monitors/)
- [Arch Linux acpid Documentation](https://wiki.archlinux.org/title/Acpid)
- [systemd-logind Manual](https://www.freedesktop.org/software/systemd/man/logind.conf.html)
- [T2 Linux Hardware Support](https://wiki.t2linux.org/)

## Notes

- **Server Mode Critical**: The `-s` flag runs hyprdock as a daemon that monitors hardware events
- **ACPI Dependency**: acpid is essential for detecting physical lid open/close events
- **Monitor Name Variability**: Some systems may use `eDP-1`, others `eDP1` - verify with `hyprctl monitors`
- **External Monitor Detection**: hyprdock automatically detects most external monitors
- **Power Behavior**: Without external monitor, lid closure triggers system suspend
- **Compatibility**: This configuration works with all T2 MacBook models (2018-2020)