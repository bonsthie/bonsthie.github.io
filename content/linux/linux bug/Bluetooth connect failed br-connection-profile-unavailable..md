## 1. Check dmesg

```sh
sudo dmesg | grep -i -e bluetooth -e hci0

Bluetooth: Core ver 2.22
Bluetooth: HCI device and connection manager initialized
Bluetooth: hci0: HW/SW Version: 0x008a008a, Build Time: 20250625154126
Bluetooth: hci0: Device setup in 3255615 usecs
Bluetooth: hci0: HCI Enhanced Setup Synchronous Connection command is advertised, but not supported.
Bluetooth: hci0: AOSP extensions version v1.00
Bluetooth: hci0: AOSP quality report is supported
Bluetooth: MGMT ver 1.23
```

Confirmed that the Bluetooth device (hci0) was detected, initialized, and not blocked. No hardware or driver errors were present.

## 2. Update Firmware

Ran:

```
sudo pacman -Syu linux-firmware
```

This installed the required MediaTek Bluetooth firmware used by the IMC Networks module.

## 3. Re-check dmesg

Commands:

```
sudo dmesg | grep -i -e bluetooth -e hci0
```

Verify:

- hci0 initializes without error
- No firmware load failures
- No BR/EDR power errors  
    After firmware update, dmesg showed proper initialization of hci0 with no fallback or BR/EDR-related errors.