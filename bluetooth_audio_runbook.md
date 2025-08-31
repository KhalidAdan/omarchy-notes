# T2 MacBook Bluetooth Audio Optimization Runbook for Omarchy Linux

## Overview

This runbook addresses initial audio stuttering and crackling issues when using Bluetooth audio devices on T2 MacBooks running Omarchy Linux. The primary cause is PipeWire audio processes starting at normal priority before being elevated by RTKit, causing brief audio interruptions during the transition period.

## Prerequisites

- T2 MacBook (2018-2020) with Omarchy installed
- Bluetooth audio device (headphones, speakers, etc.)
- Working Bluetooth connectivity
- Root/sudo access

## Problem Description

When connecting Bluetooth audio devices to T2 MacBooks running Omarchy, users experience:

- **Initial audio crunchiness/crackling** for 10-30 seconds after connection
- **Stuttering playback** when audio first starts
- **Normal audio quality** after the initial period
- **No "client too slow" errors** in logs (distinguishing it from other audio issues)

## Root Cause Analysis

The issue occurs because:

1. **PipeWire starts at normal priority** (nice level 0)
2. **Audio processing can't keep up** initially, causing buffer underruns
3. **RTKit eventually elevates PipeWire** to high priority (nice level -11)
4. **Audio smooths out** once proper priority is established

### Diagnostic Verification

You can verify this behavior by examining process priorities and RTKit logs:

```bash
# Check PipeWire process priorities
ps -elf | grep pipewire

# Expected output showing priority differences:
# pipewire main process: priority -11 (high priority)
# pipewire-pulse: priority 0 (normal priority - this causes issues)
```

```bash
# Check RTKit priority elevation logs
journalctl | grep -E "(rtkit|nice-level)" | tail -10

# Look for messages like:
# "Successfully made thread XXXX of process YYYY owned by '1000' high priority at nice level -11"
```

## Solution: Enable RTKit Priority Management

### Step 1: Verify RTKit Installation

```bash
# Check if RTKit is installed
pacman -Q rtkit

# If not installed:
sudo pacman -S rtkit
```

### Step 2: Enable RTKit Service

```bash
# Enable RTKit daemon to start on boot
sudo systemctl enable rtkit-daemon.service

# Start RTKit daemon immediately
sudo systemctl start rtkit-daemon.service

# Verify RTKit is running
systemctl status rtkit-daemon.service
```

**Expected output:**
```
● rtkit-daemon.service - RealtimeKit Scheduling Policy Service
     Loaded: loaded (/usr/lib/systemd/system/rtkit-daemon.service; enabled; preset: disabled)
     Active: active (running) since [timestamp]
```

### Step 3: Configure PipeWire for RTKit Integration

Ensure PipeWire is configured to request real-time priorities:

```bash
# Check PipeWire configuration
cat ~/.config/pipewire/pipewire.conf 2>/dev/null || echo "Using system defaults"

# View system PipeWire RTKit settings
grep -A 10 -B 5 "rt.prio\|nice.level" /usr/share/pipewire/pipewire.conf
```

### Step 4: Configure PipeWire-Pulse for Priority

The key component causing issues is `pipewire-pulse`. Configure it for proper priority:

```bash
# Create user PipeWire directory if it doesn't exist
mkdir -p ~/.config/pipewire

# Create or edit pipewire-pulse configuration
cat > ~/.config/pipewire/pipewire-pulse.conf << 'EOF'
# PipeWire-Pulse configuration for T2 MacBooks
context.properties = {
    default.clock.rate      = 48000
    default.clock.quantum   = 1024
    default.clock.min-quantum = 32
    default.clock.max-quantum = 2048
    
    # Request high priority from RTKit
    rt.prio = 88
    nice.level = -11
}

context.modules = [
    {   name = libpipewire-module-rt
        args = {
            rt.prio      = 20
            rt.time.soft = 200000
            rt.time.hard = 200000
        }
        flags = [ ifexists nofail ]
    }
    {   name = libpipewire-module-protocol-native }
    {   name = libpipewire-module-client-node }
    {   name = libpipewire-module-adapter }
    {   name = libpipewire-module-metadata }
    {   name = libpipewire-module-protocol-pulse
        args = {
            # Increase buffer sizes for Bluetooth stability
            pulse.min.req = 1024/48000
            pulse.default.req = 8192/48000
            pulse.max.req = 16384/48000
            pulse.min.quantum = 1024/48000
            pulse.max.quantum = 8192/48000
        }
    }
]
EOF
```

### Step 5: Restart Audio Services

```bash
# Restart PipeWire services
systemctl --user restart pipewire pipewire-pulse

# Verify services are running with correct priorities
ps -elf | grep pipewire

# Check RTKit has granted priorities
journalctl --user -u pipewire | grep -i rtkit
```

### Step 6: Test Bluetooth Audio

```bash
# Connect to your Bluetooth device
bluetoothctl connect XX:XX:XX:XX:XX:XX

# Test audio playback immediately after connection
paplay /usr/share/sounds/alsa/Front_Left.wav

# Monitor for audio issues during initial connection
journalctl --user -f -u pipewire-pulse
```

## Advanced Configuration

### Optimize Bluetooth Audio Codec Settings

For high-quality audio devices like Sony WH-1000XM5:

```bash
# Check available codecs for your device
pactl list cards | grep -A 20 "bluez_card"

# Force specific codec (example: AAC for better quality)
# Create or edit bluetooth configuration
sudo mkdir -p /etc/bluetooth
sudo tee -a /etc/bluetooth/main.conf << 'EOF'

[Policy]
# Enable all codecs
AutoEnable=false

[A2DP]
# Prefer high-quality codecs
Disable=false
EOF

# Restart Bluetooth service
sudo systemctl restart bluetooth
```

### Custom Audio Buffer Configuration

For persistent audio issues, create a custom PipeWire configuration:

```bash
# Create advanced PipeWire configuration
cat > ~/.config/pipewire/pipewire.conf << 'EOF'
context.properties = {
    # Optimize for Bluetooth audio
    default.clock.rate = 48000
    default.clock.quantum = 1024
    default.clock.min-quantum = 32
    default.clock.max-quantum = 8192
    
    # Memory and scheduling optimization
    mem.warn-mlock = false
    mem.allow-mlock = true
    
    # Higher priority settings
    rt.prio = 95
    nice.level = -19
}

context.spa-libs = {
    audio.convert.* = audioconvert/libspa-audioconvert
    support.*       = support/libspa-support
}

context.modules = [
    {   name = libpipewire-module-rtkit
        args = {
            rt.prio      = 20
            rt.time.soft = 2000000
            rt.time.hard = 2000000
        }
        flags = [ ifexists nofail ]
    }
    {   name = libpipewire-module-protocol-native }
    {   name = libpipewire-module-profiler }
    {   name = libpipewire-module-metadata }
    {   name = libpipewire-module-spa-device-factory }
    {   name = libpipewire-module-spa-node-factory }
    {   name = libpipewire-module-client-node }
    {   name = libpipewire-module-client-device }
    {   name = libpipewire-module-portal }
    {   name = libpipewire-module-access
        args = {}
    }
    {   name = libpipewire-module-adapter }
    {   name = libpipewire-module-link-factory }
    {   name = libpipewire-module-session-manager }
]
EOF
```

## Troubleshooting

### Issue: RTKit Not Granting Priorities

**Symptoms:**
- PipeWire processes remain at normal priority
- Audio continues to crackle beyond initial period

**Diagnosis:**
```bash
# Check RTKit limits
cat /proc/sys/kernel/sched_rt_runtime_us
cat /proc/sys/kernel/sched_rt_period_us

# Check user limits
ulimit -r
```

**Solution:**
```bash
# Add user to audio group
sudo usermod -a -G audio $USER

# Configure RTKit limits
sudo tee /etc/security/limits.d/95-audio.conf << 'EOF'
@audio - rtprio 95
@audio - memlock unlimited
@audio - nice -19
EOF

# Log out and back in for group changes to take effect
```

### Issue: Bluetooth Device Not Using Optimal Codec

**Diagnosis:**
```bash
# Check current codec in use
pactl list sinks | grep -A 20 bluez | grep codec
```

**Solution:**
```bash
# Install additional Bluetooth codecs
sudo pacman -S pipewire-media-session libldac

# Restart Bluetooth and PipeWire
sudo systemctl restart bluetooth
systemctl --user restart pipewire-media-session
```

### Issue: Persistent Audio Dropouts

**Diagnosis:**
```bash
# Monitor for xruns (buffer underruns)
journalctl --user -f | grep -E "(xrun|underrun|client too slow)"

# Check system load during audio playback
top -p $(pgrep pipewire)
```

**Solution:**
```bash
# Increase audio buffer sizes
wireplumber --version && {
    mkdir -p ~/.config/wireplumber/main.lua.d
    cat > ~/.config/wireplumber/main.lua.d/51-increase-buffers.lua << 'EOF'
-- Increase buffer sizes for Bluetooth stability
table.insert(alsa_monitor.rules, {
  matches = {
    {
      { "api.alsa.card.name", "matches", "bluez_*" },
    },
  },
  apply_properties = {
    ["api.alsa.period-size"]   = 2048,
    ["api.alsa.headroom"]      = 8192,
  },
})
EOF
}

# Restart audio services
systemctl --user restart pipewire wireplumber
```

## Performance Verification

### Monitoring Tools

```bash
# Monitor real-time audio performance
pw-top

# Check PipeWire graph and latency
pw-dump | jq '.[] | select(.info.props."media.class" == "Audio/Sink")'

# Monitor RTKit activity
journalctl -f | grep rtkit-daemon
```

### Benchmark Audio Latency

```bash
# Test audio latency (requires jack tools)
sudo pacman -S jack-example-tools

# Measure round-trip latency
jack_delay -I system:capture_1 -O system:playback_1
```

## Device-Specific Notes

### Sony WH-1000XM5
- **Codec**: Automatically selects AAC for optimal quality
- **Volume Control Collision**: Hardware volume controls may conflict with PipeWire
- **Connection**: Stable connection with proper RTKit configuration
- **Latency**: ~40ms typical latency with optimized settings

### AirPods/AirPods Pro
- **Codec**: Uses SBC by default, AAC if properly configured
- **Auto-switching**: May require manual codec selection
- **Battery reporting**: Limited battery status information

### Generic A2DP Devices
- **Codec**: Falls back to SBC, upgrade to aptX if supported
- **Compatibility**: Should work with basic configuration
- **Quality**: Varies significantly by device manufacturer

## References

- [PipeWire Documentation](https://docs.pipewire.org/)
- [RTKit Documentation](https://github.com/heftig/rtkit)
- [Arch Linux Audio Guide](https://wiki.archlinux.org/title/PipeWire)
- [Bluetooth Audio Troubleshooting](https://wiki.archlinux.org/title/Bluetooth_headset)
- [T2 Linux Audio Support](https://wiki.t2linux.org/guides/audio/)

## Notes

- **Boot Persistence**: RTKit must be enabled as a service to prevent audio issues on every boot
- **Hardware Compatibility**: This specifically addresses T2 MacBook Bluetooth audio issues
- **Quality vs. Latency**: Higher buffer sizes improve stability but may increase latency
- **Power Management**: Some Bluetooth devices may still experience issues with aggressive power management