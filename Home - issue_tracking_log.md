# T2 MacBook Omarchy Linux Issue Tracking Log

## Purpose

This document serves as a comprehensive issue tracking log for getting a T2 MacBook fully functional with Omarchy Linux. It documents the journey from a fresh installation to a fully working system, including all encountered issues, their resolution status, and references to detailed runbooks.

This log is maintained for future reference and to help other developers understand the common challenges and solutions when installing Linux on T2 MacBooks. Each issue links to detailed technical runbooks for implementation.

---

## Hardware Platform
- **Model**: T2 MacBook Pro (2018-2020)  
- **Operating System**: Omarchy Linux (Arch-based)
- **Desktop Environment**: Hyprland (Wayland compositor)
- **Installation Method**: T2 Linux ISO with T2-specific kernel and drivers

---

## Issue Status Legend
- ✅ **Resolved**: Issue completely fixed with stable solution
- ⚠️ **Partial**: Issue mostly resolved with minor limitations  
- ❌ **Unresolved**: Issue remains problematic
- 🔄 **Transient**: Issue occurs intermittently
- 📋 **Documented**: Issue documented but user chose not to fix

---

## Core System Issues

### ✅ Holding Backspace Key Behavior
**Issue**: Holding backspace doesn't continually delete characters in applications like Obsidian  
**Resolution**: Fixed after system restart following T2 driver installations  
**Root Cause**: Incomplete keyboard driver initialization  
**Action Required**: None - resolves automatically after proper T2 setup

### ✅ WiFi Connectivity  
**Issue**: WiFi interface (wlan0) doesn't come up automatically after fresh installation  
**Resolution**: Manual interface activation required: `sudo ip link set wlan0 up`  
**Root Cause**: T2-specific Broadcom WiFi drivers require manual interface initialization  
**Documentation**: [[wifi_runbook]]
**Permanent Fix**: Systemd service created for automatic interface activation on boot

### ✅ Bluetooth Basic Functionality
**Issue**: Bluetooth service not working initially  
**Resolution**: Fixed after system restart  
**Root Cause**: Service initialization timing with T2 drivers  
**Action Required**: None after initial setup [[bluetooth_testing]]

### ✅ Bluetooth Audio Quality  
**Issue**: Audio crackling and stuttering when initially connecting Bluetooth devices  
**Resolution**: Enable RTKit daemon for proper PipeWire priority management  
**Root Cause**: PipeWire audio processes start at normal priority before RTKit elevates them  
**Documentation**: [[bluetooth_audio_runbook]]
**Technical Details**: PipeWire-pulse runs at normal priority causing buffer underruns until RTKit grants real-time priority

### ✅ Display Brightness Control
**Issue**: Brightness function keys work but don't affect actual screen brightness  
**Resolution**: Target correct backlight device (`acpi_video0`) in brightness controls  
**Root Cause**: Multiple backlight devices available, default selection incorrect  
**Documentation**: 
**Implementation**: Modified Hyprland keybindings to use `brightnessctl -d acpi_video0`

### ✅ Keyboard and Trackpad Hardware
**Issue**: Keyboard and trackpad non-functional after fresh installation  
**Resolution**: Install T2-specific drivers (linux-t2, apple-bce, t2fanrd) and restart system  
**Root Cause**: T2 security chip requires specialized drivers for input device communication  
**Documentation**: [[keyboard_trackpad_runbook]]
**Implementation**: apple-bce module provides Bridge Control Engine interface for T2 input devices
### ✅ Keyboard Backlight Control
**Issue**: Keyboard backlight function keys non-functional  
**Resolution**: Configure brightnessctl for `apple::kbd_backlight` device  
**Root Cause**: T2-specific keyboard backlight device path  
**Documentation**: [[display_brightness_runbook]]
**Implementation**: Custom key bindings in Hyprland configuration

### ✅ Clamshell Mode (Laptop Lid Behavior)
**Issue**: No clamshell mode functionality - closing lid with external monitor doesn't work properly  
**Resolution**: Install and configure hyprdock with acpid for hardware lid event detection  
**Root Cause**: No built-in clamshell mode support in Hyprland  
**Documentation**: [[clamshell_mode_runbook]]
**Technical Implementation**: 
  - acpid for lid event detection
  - hyprdock daemon for monitor management  
  - systemd-logind lid switch handling disabled

### ✅ Xbox Controller Bluetooth Pairing
**Issue**: Xbox controller fails to pair using standard Bluetooth pairing procedures  
**Resolution**: Use specific pairing sequence: trust → connect → pair  
**Root Cause**: T2 Bluetooth stack requires non-standard pairing order  
**Documentation**: [[xbox_controller_runbook]]
**Critical Detail**: Standard pair → connect sequence fails; trust-first sequence succeeds

### ✅ External Monitor Sleep/Wake Reconnection

**Issue**: External monitors fail to reconnect or display properly after system wakes from sleep/suspend  
**Resolution**: systemd-sleep script automatically runs DPMS reset after resume: `hyprctl dispatch dpms on`  
**Root Cause**: Hyprland display power management state becomes desynchronized after suspend/resume cycle  
**Documentation**: [[external_monitor_sleep_runbook]]
**Permanent Fix**: `/usr/lib/systemd/system-sleep/99-monitor-wake.sh` script handles automatic reconnection  
**Fallback Method**: Manual VT switching (Ctrl+Alt+F3 → Ctrl+Alt+F1) forces display reinitialization

### ✅ Node.js Development Environment Setup

**Issue**: Need to develop Node.js and React applications on Omarchy  
**Resolution**: Install Node.js, npm, and configure VS Code with essential extensions  
**Root Cause**: Fresh system lacks development tooling  
**Guide**: [[nodejs_runbook]]
**Action Required**: Run installation commands and configure VS Code settings

---

## Hardware-Specific Limitations

### 📋 Sony WH-1000XM5 Hardware Volume Controls
**Issue**: Volume controls on Sony headphones don't work with PipeWire  
**Resolution**: User chose not to fix - documented limitation  
**Root Cause**: Sony's volume control protocol conflicts with PipeWire volume management  
**Workaround**: Use software volume controls instead of hardware buttons  
**Impact**: Low - software volume control works normally

### 🔄 External Monitor Connection After Sleep
**Issue**: When laptop goes to sleep, external monitor sometimes doesn't reconnect  
**Status**: Transient issue - occurs intermittently  
**Temporary Workaround**: Manual monitor detection via `hyprctl dispatch dpms off/on`  
**Root Cause**: Power management timing issues with external display detection  
**Frequency**: Occasional - not consistent enough for dedicated debugging effort

---

## System Integration Notes

### T2-Specific Package Dependencies
**Critical Packages**:
- `linux-t2`: T2-specific kernel with hardware support
- `apple-bcm-firmware`: Broadcom WiFi/Bluetooth firmware  
- `t2fanrd`: T2 fan control daemon
- `apple-bce`: T2 audio and other hardware support

### Service Dependencies
**Required Services**:
- `acpid`: Hardware event detection (lid, power button)
- `rtkit-daemon`: Real-time priority management for audio
- `bluetooth`: T2 Bluetooth functionality
- `NetworkManager` or `iwd`: Network management with T2 WiFi

### Configuration Integration Points
**Key Configuration Files**:
- `/etc/systemd/logind.conf`: Lid switch behavior  
- `~/.config/hypr/hyprland.conf`: Window manager and hardware key bindings
- `~/.config/hyprdock/config.toml`: Clamshell mode behavior
- `~/.config/pipewire/`: Audio system optimization

---

## Performance Impact Assessment

### Positive Impacts
- **Audio Quality**: RTKit configuration eliminated Bluetooth audio stuttering  
- **Display Management**: Clamshell mode provides seamless laptop/desktop transitions
- **Hardware Integration**: All T2-specific hardware now functional
- **Power Management**: Proper brightness control improves battery life

### Remaining Limitations  
- **External Monitor DDC**: Hardware brightness control for external monitors not supported on T2
- **Sleep Reliability**: Occasional external monitor reconnection issues
- **Hardware Volume**: Some Bluetooth devices have limited hardware control integration

---

## Development Experience Notes

**Quote from User**: *"I really feel 12 again tinkering with Linux!"*

This captures the essential experience - while T2 MacBooks require additional configuration compared to standard PC hardware, the process is methodical and the solutions are stable once implemented. Each issue has a clear technical cause and reproducible solution.

### Lessons Learned
1. **T2 Driver Dependencies**: Many issues resolve after proper T2-specific package installation
2. **Service Timing**: Several issues are caused by service initialization timing rather than fundamental incompatibility
3. **Hardware Abstraction**: T2 chip creates unique device paths and behaviors requiring specific configuration
4. **Community Resources**: T2 Linux community has developed robust solutions for all major compatibility issues

### Recommendations for Future Installers
1. **Follow T2 Linux Installation Guide**: Use official T2 ISO and installation procedures
2. **Install T2 Packages First**: Complete T2-specific package installation before troubleshooting individual issues  
3. **Systematic Approach**: Address issues in order of system impact (network, audio, display, peripherals)
4. **Documentation**: Keep detailed notes - T2 hardware quirks can be device-specific

---

## References and Resources

### Primary Documentation Sources
- [T2 Linux Wiki](https://wiki.t2linux.org/) - Comprehensive T2 hardware support documentation
- [Omarchy Manual](https://manuals.omamix.org/2/the-omarchy-manual) - Distribution-specific configuration
- [Hyprland Wiki](https://wiki.hypr.land/) - Window manager configuration and troubleshooting
- [Arch Linux Wiki](https://wiki.archlinux.org/) - General Linux configuration reference

### Community Resources
- [T2 Linux GitHub](https://github.com/t2linux) - T2-specific kernel and driver development
- [T2 Linux Subreddit](https://reddit.com/r/t2linux) - Community support and troubleshooting
- [Hyprland Discord](https://discord.gg/hQ9XvMUjjr) - Real-time support for Hyprland issues

### Technical Runbooks Created
Each issue resolution has been documented in detailed technical runbooks:

1. **WiFi Connectivity**: Complete setup for T2 Broadcom WiFi including permanent fixes
2. **Bluetooth Audio**: RTKit configuration and PipeWire optimization for quality audio
3. **Xbox Controller**: Specific pairing sequence for T2 Bluetooth compatibility  
4. **Clamshell Mode**: Full hyprdock installation and configuration for macOS-like behavior
5. **Brightness Control**: Internal display and keyboard backlight configuration
6. **Issue Tracking**: This comprehensive log for future reference

---

## Future Maintenance

### Monitoring Points
- **Kernel Updates**: T2-specific kernel updates may require configuration review
- **PipeWire Updates**: Audio configuration may need adjustment with major PipeWire releases  
- **Hyprland Updates**: Monitor configuration and key bindings may require updates
- **Service Changes**: systemd service configurations should be reviewed after major updates

### Backup Recommendations
**Critical Configuration Files to Backup**:
```bash
# System configurations
/etc/systemd/logind.conf
/etc/udev/rules.d/90-wlan0-up.rules

# User configurations  
~/.config/hypr/hyprland.conf
~/.config/hyprdock/config.toml
~/.config/pipewire/
~/.config/waybar/config
```

### Update Procedures
1. **Before Major Updates**: Backup all T2-specific configurations
2. **After Kernel Updates**: Verify T2 drivers are still loaded (`lsmod | grep -E "bce|t2"`)
3. **After Audio Updates**: Test Bluetooth audio quality and RTKit functionality
4. **After Display Updates**: Verify clamshell mode and brightness controls

---

## Contributing Back to Community

### Documentation Contributions
This issue tracking log and associated runbooks represent solutions to common T2 MacBook Linux installation challenges. Key contributions include:

- **Systematic Problem Documentation**: Each issue documented with root cause analysis
- **Reproducible Solutions**: Step-by-step procedures that can be followed by other users
- **Integration Points**: Clear understanding of how different system components interact
- **Testing Procedures**: Verification steps to ensure solutions work correctly

### Knowledge Sharing Opportunities
1. **T2 Linux Wiki Contributions**: Add specific Omarchy/Hyprland procedures to community wiki
2. **GitHub Issue Responses**: Reference detailed runbooks when helping other users  
3. **Community Forum Posts**: Share systematic approach to T2 troubleshooting
4. **Video Tutorials**: Create visual guides for complex procedures like clamshell setup

### Code Contributions
Potential areas for upstream contributions:
- **hyprdock**: T2-specific configuration templates or documentation
- **Omarchy**: Integration of T2 hardware setup into distribution tools
- **T2 Linux**: Documentation improvements based on real-world installation experience

---

## Final System State

### Fully Functional Components
- ✅ **WiFi**: Automatic connection with proper interface management
- ✅ **Bluetooth**: High-quality audio without stuttering  
- ✅ **Display**: Internal brightness control and clamshell mode
- ✅ **Input**: Keyboard backlight control and Xbox controller support
- ✅ **Power**: Proper suspend/resume behavior
- ✅ **Audio**: Professional-quality Bluetooth audio with RTKit optimization

### Known Limitations  
- **External Monitor Brightness**: No DDC support due to T2 hardware limitations
- **Some Hardware Volume Controls**: Device-specific compatibility varies
- **Occasional Sleep Issues**: Minor transient issues with external monitor detection

### Overall Assessment
**Result**: T2 MacBook Pro running Omarchy Linux provides an excellent desktop Linux experience that matches or exceeds macOS functionality in most areas. The initial setup investment pays off with a stable, high-performance system.

**Performance**: Native Linux performance on Apple hardware with proper power management and thermal control through T2-specific drivers.

**Compatibility**: All essential hardware functions working with documented solutions for edge cases.

**Stability**: System stable for daily development work with no critical unresolved issues.

---

## Appendix: Quick Reference Commands

### System Health Checks
```bash
# Verify T2 drivers loaded
lsmod | grep -E "(bce|t2|apple)"

# Check critical services
systemctl status acpid bluetooth rtkit-daemon

# Monitor hardware events  
journalctl -f | grep -E "(lid|bluetooth|wifi)"

# Test hardware functionality
brightnessctl -d acpi_video0 get        # Display brightness
brightnessctl -d apple::kbd_backlight get  # Keyboard backlight  
hyprctl monitors                         # Display configuration
iwctl station wlan0 get-networks        # WiFi functionality
```

### Emergency Recovery
```bash
# WiFi recovery
sudo ip link set wlan0 up
sudo systemctl restart NetworkManager

# Audio recovery  
systemctl --user restart pipewire pipewire-pulse
sudo systemctl restart rtkit-daemon

# Display recovery
hyprctl dispatch dpms off
hyprctl dispatch dpms on

# Bluetooth recovery
sudo systemctl restart bluetooth
bluetoothctl power off && bluetoothctl power on
```

### Configuration Verification
```bash
# Check lid behavior
grep HandleLid /etc/systemd/logind.conf

# Verify hyprdock running
pgrep hyprdock && echo "Clamshell mode active"

# Check audio priority  
ps -elf | grep pipewire | grep -E "(nice|rt)"

# Monitor setup verification
cat ~/.config/hypr/hyprland.conf | grep "exec-once.*hyprdock"
```

---

**Document Version**: 1.0  
**Last Updated**: Post-installation system configuration complete  
**Status**: All major issues resolved, system fully functional  
**Next Review**: After first major system update