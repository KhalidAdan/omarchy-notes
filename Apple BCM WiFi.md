### Wi-FI
Needed to run `sudo ip link set wlan0 up` to force the interface to come up

**Troubleshooting Steps:**

1. Verify T2 packages installed: `pacman -Q | grep -E "(linux-t2|apple-bcm-firmware|t2fanrd)"`
2. Check RF kill status: `rfkill list` (should show no blocks)
3. **Bring interface up manually**: `sudo ip link set wlan0 up`
4. Scan for networks: `iwctl station wlan0 scan && iwctl station wlan0 get-networks`

**📝 Note:** After fresh install, may need to manually bring up `wlan0` interface.

**Connecting**

To connect run `iwctl station wlan0 "ENTER_NETWORK_NAME_HERE"` and use your password to connect.