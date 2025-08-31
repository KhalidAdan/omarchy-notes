## Issue Resolution: Xbox Controller Bluetooth Pairing

**Problem**: Xbox controller not pairing with Omarchy

**Solution Steps**:

### 1. Fix Bluetooth Service Issues

If `bluetoothctl` gives weird prompts or keyboard interference:

```bash
sudo systemctl restart bluetooth
```

### 2. Put Controller in Pairing Mode

- Hold Xbox button + pairing button (small button on top) until Xbox button flashes rapidly

### 3. Pair Xbox Controller

```bash
bluetoothctl
```

**Important Order** (this sequence worked):

1. `trust [MAC_ADDRESS]`
2. `connect [MAC_ADDRESS]`
3. `pair [MAC_ADDRESS]`

### 4. Find MAC Address

Use `scan on` in bluetoothctl to discover the controller's MAC address

## Notes

- Standard pairing order didn't work initially
- The trust → connect → pair sequence was key for this Starfield Xbox controller
- If controller still has issues, consider installing `xpadneo-dkms` from AUR for better driver support

## Troubleshooting

- If bluetoothctl shows keyboard prompts: restart bluetooth service first
- Service name is `bluetooth`, not `bluetoothctl`