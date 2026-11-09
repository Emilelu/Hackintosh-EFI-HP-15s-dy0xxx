> [简体中文](../WiFi-与蓝牙共存.md)

# Wi-Fi and Bluetooth Coexistence: Trade-offs and Tuning for the RTL8821CE Combo Card

> Back to [main README](../../README.en.md) | Model: HP 15s-dy0xxx / RTL8821CE / macOS Tahoe 26.7.2

The RTL8821CE is a **Wi-Fi + Bluetooth combo** chip: Wi-Fi runs over PCIe and Bluetooth over USB, but both share the same RF front end, antennas, and coexistence arbitration logic. Under macOS, Wi-Fi is driven by `AirPortRTW`, while Bluetooth is driven by `RealtekBluetoothFirmware` + `BlueToolFixup` + `RealtekBTPatcher`; these are two independent projects and do not coordinate coexistence by default the way Realtek's vendor driver does. This document records the coexistence issues, fix combination, and version pitfalls on this machine.

## 1. Hardware and Driver Stack

| Function | Bus | Hardware ID | Driver / Component |
|:---|:---|:---|:---|
| Wi-Fi | PCIe | `10EC:C821` / subsystem `103C:831A` | `AirPortRTW.kext` 1.2.0 + `RealtekAirPortRTWFix` |
| Bluetooth | USB | `0BDA:B00A` | `RealtekBluetoothFirmware.kext` 1.0.2 (official release) + `BlueToolFixup.kext` 2.7.3 + `RealtekBTPatcher.kext` |
| RF / antenna | — | 2.4GHz / 5GHz shared | Hardware coexistence arbitration; cannot be fully isolated on the driver side |

> Key fact about combo cards: **Wi-Fi and Bluetooth cannot both use the same band at full load at the same time**. The conflict between 2.4GHz Wi-Fi and Bluetooth audio / BLE is the most obvious; 5GHz Wi-Fi is much farther away in frequency from Bluetooth, so the practical impact is much smaller.

## 2. Typical Symptoms

| Symptom | Trigger condition | Explanation |
|:---|:---|:---|
| Bluetooth audio stutters / crackles / disconnects | Large Wi-Fi downloads, Speedtest, LAN file copies | 2.4GHz Wi-Fi consumes RF time slices, leaving Bluetooth A2DP unable to get scheduled |
| BLE HID keyboard lag, dropped keys, occasional disconnects | Wi-Fi scanning or connecting to a new AP | Active scanning briefly preempts the radio and antenna |
| Wi-Fi throughput drops sharply | Continuous Bluetooth audio playback / BLE scanning | Coexistence arbitration yields some time slices to Bluetooth |
| Wi-Fi or Bluetooth stops working on one side after sleep/wake | Wake from lid close, long sleep | The two drivers' power-state restore order is inconsistent and the RF is not reinitialized |
| Bluetooth connects but carries no data | When using a self-compiled `RealtekBluetoothFirmware` | Abnormal firmware RX path, manifesting as zero advertising and zero scan results |

> If Bluetooth problems appear only when Wi-Fi is busy and recover when idle, treat it as a "coexistence" issue first; if there is no advertising at all even when idle, treat it as a Bluetooth driver / firmware issue, see the [Bluetooth BLE HID fix notes](./Bluetooth-BLE-HID-Fix.md).

## 3. Root Causes

1. **Shared RF and antennas**: The RTL8821CE's Wi-Fi and BT share the same front end. On the 2.4GHz band, Wi-Fi OFDM frames and BT frequency hopping are time-division multiplexed; hardware coexistence arbitration can only allocate time slices between them and cannot truly achieve "full speed on both at once".
2. **The two macOS drivers lack vendor-level coexistence services**: `AirPortRTW` is a community-maintained Realtek Wi-Fi driver and `RealtekBluetoothFirmware` is a separate Bluetooth firmware loader; they do not share the coexistence state machine found in Realtek's original Windows driver. When Wi-Fi initializes, scans, or changes channels, the Bluetooth side does not receive a complete "radio about to be occupied" notification.
3. **Scanning is aggressive preemption**: macOS Wi-Fi scans traverse channels, and Bluetooth audio and BLE connections are most affected during scanning; this is why "opening the Wi-Fi settings page makes Bluetooth stutter".
4. **Inconsistent resume order**: On wake, the power restore and firmware reload order of the Wi-Fi PCIe device and the Bluetooth USB device is nondeterministic; if one side finishes while the other is not ready, the result is "one half works, the other does not".

## 4. Fix Combination

### 4.1 Wi-Fi: AirPortRTW 1.2.0 + RealtekAirPortRTWFix

- **`AirPortRTW.kext` 1.2.0**: upstream [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW), providing a native macOS `IO80211` Wi-Fi interface for the RTL8821CE.
- **`RealtekAirPortRTWFix`**: [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix), which fixes RTL8821CE Wi-Fi / BT RF coexistence while preserving 1.2.0 functionality. In practice, follow that repository's instructions to merge the fix into the `AirPortRTW.kext` build or load it as the corresponding patch; **do not keep two copies of `AirPortRTW` from unknown sources at the same time**.
- Verified on this machine: **WPA3-Personal connects normally**, and the frequency of Bluetooth audio / BLE disconnects drops noticeably when Wi-Fi is busy.

### 4.2 Bluetooth: the Official Release Binary Is Mandatory

| Component | Version | Requirement |
|:---|:---|:---|
| `RealtekBluetoothFirmware.kext` | **1.0.2** | **Must use the official release binary**; do not use a self-compiled build |
| `BlueToolFixup.kext` | 2.7.3 | Use a version that matches Tahoe so that `bluetoothd` accepts third-party controllers |
| `RealtekBTPatcher.kext` | Release from the same author's repository | BLE HID event injection patch, used with `-ibtpwrap` |

> **Measured lesson: a self-compiled `RealtekBluetoothFirmware` completely breaks Bluetooth RX** — the controller enumerates and can upload firmware, but scans find no advertising at all (zero advertising, zero devices), so a BLE keyboard naturally cannot pair either. The cause is related to firmware / config resource packaging or post-download state handling in the self-compiled artifact; do not waste time on it — use the official 1.0.2 release binary directly.

### 4.3 The Two Wi-Fi Kexts That Must Never Be Loaded Together

**Do not load `AirPort_RTW88.kext` and `AirPortRTW.kext` at the same time.**

- Both try to match the same Realtek PCI device (`10EC:C821`); loading both causes duplicate matching, `IOService` contention, and a random winner for binding, manifesting as intermittent Wi-Fi, boot hangs, or even kernel panics.
- `AirPort_RTW88` targets the RTW88 series (such as the RTL8822CE), and its device ID table differs from the RTL8821CE `AirPortRTW` path; mixing them yields no benefit.
- Keep only `AirPortRTW.kext` 1.2.0 (with the `RealtekAirPortRTWFix` fix).

## 5. Recommended Load Order

In `Kernel → Add`, keep the following relative order (the exact `BundlePath` depends on your `config.plist`):

```text
Lilu.kext
AirPortRTW.kext                 (1.2.0 + RealtekAirPortRTWFix build)
RealtekBluetoothFirmware.kext   (1.0.2 official release)
BlueToolFixup.kext              (2.7.3)
RealtekBTPatcher.kext
```

> All Lilu plugins must come after `Lilu.kext`; there is no hard dependency between Wi-Fi and Bluetooth, but placing the firmware loader before the stack patches makes troubleshooting easier. Change only one kext at a time, validate with `ocvalidate`, and keep a backup EFI.

## 6. Verification Methods

### 6.1 Wi-Fi

```bash
# Show the Wi-Fi interface and current connection
system_profiler SPAirPortDataType

# Check whether the PCI device is bound correctly
ioreg -lw0 | grep -i "10ec,c821" -A 5
```

- Connect to a WPA3-Personal AP and confirm that it obtains an IP address and can reach the Internet.
- Use `iperf3` or large file copies to generate sustained traffic while playing Bluetooth audio / using the BLE keyboard, and observe whether disconnects occur.
- Manually "scan" for other networks from the Wi-Fi menu and observe whether Bluetooth audio stutters; this is the most direct coexistence stress test.

### 6.2 Bluetooth

```bash
# Bluetooth controller and paired devices
system_profiler SPBluetoothDataType

# RealtekBTPatcher injection counters
ioreg -lw0 | grep -oE '"IBTP[A-Za-z]+" = <[^>]*>'
```

- Confirm `IBTPWrapEnabled=Y` and `IBTPInjected>=1` (during BLE HID pairing).
- After pairing the BLE keyboard, leave it idle for 10 minutes with Wi-Fi traffic active and observe whether it drops.
- While playing Bluetooth audio, run a Wi-Fi speed test at the same time and record whether stuttering occurs.

### 6.3 Sleep / Wake

1. Close the lid or trigger sleep manually for 5 minutes;
2. After waking, check separately whether Wi-Fi reconnects automatically and whether the Bluetooth device is still present;
3. If one side has failed, first try toggling the corresponding switch off/on and record whether it recovers; only consider resetting NVRAM or adjusting power management after repeated reproductions.

## 7. Daily Usage Recommendations

- **Prefer 5GHz Wi-Fi**: the band conflict between 2.4GHz and Bluetooth is the most severe; at 5GHz, Wi-Fi and Bluetooth can largely coexist.
- **Do not frequently open Wi-Fi scanning while using a BLE keyboard / Bluetooth audio**; when you need to configure a network, finish scanning first and then return to normal use.
- **Keep the antennas well connected**: combo cards are sensitive to antenna impedance; after replacing the case or disassembling the machine, confirm that both the main and auxiliary antennas are connected.
- **Avoid USB 3.0 interference**: 2.4GHz Bluetooth and the broadband noise from USB 3.0 ports can interfere with each other; keep USB 3.0 devices as far away from the antenna area as possible.
- **One side fails after sleep/wake**: there is currently no complete software fix; toggle the corresponding switch first, and reboot if that does not help. This issue is listed as **to be verified / known limitation**; do not expect to eliminate it completely by changing kext versions.

## 8. Known Limitations

- The Wi-Fi / Bluetooth coexistence fix on this machine has only been verified on **macOS Tahoe 26.7.2 + RTL8821CE (`10EC:C821` / `0BDA:B00A`)**; other Realtek combo cards (RTL8822CE, RTL8852, etc.) are not guaranteed to work.
- Coexistence is hardware-level time-division multiplexing; it is **impossible to make 2.4GHz Wi-Fi and Bluetooth both run at full speed simultaneously**. The goal of this fix is "usable, with fewer disconnects", not "zero interference".
- The one-sided failure after sleep/wake has not been traced to a single root cause; it may involve PCIe power management, USB power management, and firmware reload order simultaneously. Currently it can only be recovered by rebooting or toggling the switch.
- The version combination of `RealtekBluetoothFirmware` and `BlueToolFixup` is sensitive; after upgrading either component, re-run the verification in section 6 of this document.

## 9. References

- [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW) — upstream of AirPortRTW.
- [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) — Wi-Fi / Bluetooth coexistence fix.
- [thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware) — Realtek Bluetooth firmware loader (this machine uses the 1.0.2 official release).
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) — upstream of BlueToolFixup.
- [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher) — BLE HID event injection patch.
- This repository's [Bluetooth BLE HID fix notes](./Bluetooth-BLE-HID-Fix.md) — details of the Bluetooth-side event injection.
- This repository's [HDMI fix notes](./HDMI-Fix-Notes.md) — display output fix for the same model, useful for distinguishing "display issues" from "wireless issues".

> See the [Credits section of the main README](../../README.en.md#credits) for the full acknowledgements and all external links.
