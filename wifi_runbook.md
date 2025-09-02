# T2 MacBook Wi-Fi Setup

## Problem
After fresh Omarchy installation, Wi-Fi interface doesn't come up automatically. You'll see "No device found" errors when trying to use `iwctl`.

## Quick Fix
```bash
sudo ip link set wlan0 up
```

**That's it.** This is typically a one-time fix after installation.

## Troubleshooting Steps

If the quick fix doesn't work, debug in this order:

1. **Verify T2 packages are installed:**
   ```bash
   pacman -Q | grep -E "(linux-t2|apple-bcm-firmware|t2fanrd)"
   ```

2. **Check RF kill status:**
   ```bash
   rfkill list
   ```
   Should show `Soft blocked: no` for Wireless LAN. If blocked:
   ```bash
   sudo rfkill unblock wifi
   ```

3. **Bring interface up manually:**
   ```bash
   sudo ip link set wlan0 up
   ```

4. **Scan for networks:**
   ```bash
   iwctl station wlan0 scan && iwctl station wlan0 get-networks
   ```

## Connecting to Wi-Fi

```bash
iwctl station wlan0 connect "YOUR_NETWORK_NAME"
```
Enter your password when prompted.

## Verification

```bash
# Test connectivity
ping -c 3 8.8.8.8
```

## Common Issues

- **"No device found"**: Interface is down → run `sudo ip link set wlan0 up`
- **Can't scan networks**: Wait 5 seconds after bringing interface up, then try scanning
- **Missing T2 packages**: Install via `sudo pacman -S linux-t2 apple-bcm-firmware t2fanrd`

## Notes

- **One-time fix**: Usually only needed once after fresh install
- **Persistence**: Interface stays up after reboot (no systemd service needed)
- **Hardware**: Applies to T2 MacBooks (2018-2020) with Broadcom Wi-Fi chips
- **Alternative**: If issues persist, try `sudo systemctl restart NetworkManager`