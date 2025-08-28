*Complete setup guide for macOS-style clamshell mode on Omarchy/Hyprland*. Out of the box on my t2 macbook pro I did not have clamshell mode, this is how I got it to work.
## Overview
This runbook configures true macOS-style clamshell behavior:
- **Lid closed + external monitor** = continues on external display only
- **Lid closed + no external monitor** = laptop suspends
- **Lid opened** = dual display mode (if external connected)

## Prerequisites
- Omarchy installed and running
- External monitor, keyboard, and mouse for testing
- Root/sudo access

## Step 1: Install Dependencies

### Install Rust (if not already installed)
```bash
# Check if Rust is installed
cargo --version

# If not installed, use Omarchy menu:
# Super + Alt + Space > Install > Development > Rust

# OR install manually:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

### Install acpid (Required for lid event detection)
```bash
# Install acpid
sudo pacman -S acpid

# Enable and start acpid service
sudo systemctl enable acpid
sudo systemctl start acpid

# Verify acpid is running
systemctl status acpid
```

### Install hyprdock
```bash
# Install GTK4 dependencies first
sudo pacman -S gtk4

# Install hyprdock via cargo (bypasses AUR issues)
cargo install hyprdock
```

## Step 2: Configure System Lid Behavior

### Edit systemd logind configuration
```bash
sudo nvim /etc/systemd/logind.conf
```

Find and uncomment/modify these lines:
```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

### Restart logind service
```bash
sudo systemctl restart systemd-logind
```

## Step 3: Configure hyprdock

### Create hyprdock configuration directory
```bash
mkdir -p ~/.config/hyprdock
```

### Create hyprdock configuration file
```bash
nvim ~/.config/hyprdock/config.toml
```

Add this configuration:
```toml
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
```

## Step 4: Configure Hyprland Autostart

### Edit Hyprland configuration
```bash
nvim ~/.config/hypr/hyprland.conf
```

### Add hyprdock to autostart
Add this line to your Hyprland configuration:
```bash
exec-once = hyprdock -s
```

*Note: The `-s` flag runs hyprdock in server/daemon mode*

## Step 5: Apply Configuration

### Restart Hyprland
```bash
# Use Super + Escape, then select "Relaunch"
# OR log out and back in
```

## Step 6: Test Clamshell Functionality

### Test Sequence
1. **Connect external monitor and peripherals**
2. **Verify both displays are active**
3. **Close laptop lid** → should continue on external monitor only
4. **Open laptop lid** → should re-enable both displays
5. **Disconnect external monitor, close lid** → should suspend laptop
6. **Open lid** → should wake and show internal display

### Troubleshooting Commands
```bash
# Check if hyprdock is running
pgrep hyprdock

# Check acpid status
systemctl status acpid

# View current monitors
hyprctl monitors

# Restart hyprdock manually if needed
pkill hyprdock
hyprdock -s &

# Check lid switch events (test by opening/closing lid)
journalctl -f | grep -i lid
```

## Verification Checklist
- [ ] acpid service is running
- [ ] logind lid switches are set to ignore
- [ ] hyprdock config file exists and is configured
- [ ] hyprdock autostart is added to Hyprland config
- [ ] Hyprland has been restarted
- [ ] Clamshell behavior works as expected

## Notes
- **Server Mode**: The `-s` flag is crucial - it runs hyprdock as a daemon that monitors lid events
- **Dependencies**: acpid is required for hardware lid event detection
- **Monitor Names**: Adjust `monitor_name` in config if your internal display isn't `eDP-1`
- **External Monitor**: hyprdock auto-detects external monitors
- **Suspend Behavior**: Without external monitor, closing lid will suspend via systemctl

## Common Issues
- **No lid detection**: Ensure acpid is running and enabled
- **Hyprdock not starting**: Check that cargo binary path is in your PATH
- **External monitor not detected**: Verify monitor is properly connected and recognized by `hyprctl monitors`
- **Suspend not working**: Check that systemctl suspend works manually

## File Locations
- hyprdock config: `~/.config/hyprdock/config.toml`
- Hyprland config: `~/.config/hypr/hyprland.conf`
- System lid config: `/etc/systemd/logind.conf`
- hyprdock binary: `~/.cargo/bin/hyprdock`


## Sources

- [Hyprdock GitHub Repository](https://github.com/Xetibo/hyprdock)
- [Hyprdock Crates.io Package](https://crates.io/crates/hyprdock)
- [Omarchy Official Website](https://omarchy.org/)
- [Omarchy Manual](https://manuals.omamix.org/2/the-omarchy-manual)
- [Hyprland Wiki](https://wiki.hypr.land/)
- [Arch Linux acpid Wiki](https://wiki.archlinux.org/title/Acpid)
- [systemd-logind Documentation](https://www.freedesktop.org/software/systemd/man/logind.conf.html)
- [Rust Installation Guide](https://www.rust-lang.org/tools/install)
---
*This runbook provides a complete, reproducible setup for macOS-style clamshell mode on Linux using hyprdock and Hyprland.*