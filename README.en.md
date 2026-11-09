[简体中文](./README.md) | **English**

# Hackintosh-EFI-HP-15s-dy0xxx

> Model: HP 15s-dy0xxx (HP 15s series) | Intel Core i7-8565U | macOS Tahoe 26.7.2
>
> OpenCore 1.x (placeholder; actual version per the released EFI's `EFI/OC/OpenCore.efi`) | SMBIOS `MacBookPro15,2`

This repository documents the OpenCore EFI configuration, verification results, and two in-depth fixes for an HP 15s-dy0xxx running macOS Tahoe 26.7.2: **native HDMI output on the Intel UHD 620** and **BLE HID keyboard pairing on the Realtek RTL8821CE**. All conclusions are based on testing on this machine; anything not yet verified is explicitly marked "To be verified".

## Documentation

| Document | Contents |
|:---|:---|
| [docs/en/HDMI-Fix-Notes.md](./docs/en/HDMI-Fix-Notes.md) | The full HDMI fix, from VBT decoding to `con1 busId=1`, with log evidence and the 4K limitation analysis |
| [docs/en/Bluetooth-BLE-HID-Fix.md](./docs/en/Bluetooth-BLE-HID-Fix.md) | The `LE_READY_WAIT_FOR_LENGTH` root cause on the RTL8821CE and the `RealtekBTPatcher` event injection implementation |
| [docs/en/WiFi-Bluetooth-Coexistence.md](./docs/en/WiFi-Bluetooth-Coexistence.md) | RF coexistence, driver selection, and version pitfalls for the RTL8821CE combo card |
| [docs/en/Troubleshooting-and-Fixes.md](./docs/en/Troubleshooting-and-Fixes.md) | `disable-agdc` causing a 20-minute boot, the flags `0x0008` black screen, connector byte misalignment, Safari 120Hz, and more |
| [docs/en/BIOS-Settings.md](./docs/en/BIOS-Settings.md) | Recommended BIOS settings for this model: DVMT, CFG Lock, VT-d, Secure Boot, etc. |
| [docs/en/Complete-Configuration-and-Fixes.md](./docs/en/Complete-Configuration-and-Fixes.md) | **Complete checklist**: Bluetooth NVRAM (14 bytes), boot black screen / backlight fix, 9 SSDTs, all Booter/Misc quirks, dual boot, incident lessons, Lilu DEBUG logging notes |
| [CHANGELOG.en.md](./CHANGELOG.en.md) | Version history |
| [LICENSE](./LICENSE) | MIT License (Copyright © 2026 Emilelu) |

## Configuration Summary

| Item | Configuration |
|:---|:---|
| Model | HP 15s-dy0xxx (HP 15s series) |
| CPU | Intel Core i7-8565U (Whiskey Lake, 4 cores / 8 threads) |
| iGPU | Intel UHD Graphics 620, native `0x3EA0`, spoofed as `0x3EA5` in macOS |
| dGPU | NVIDIA GeForce MX250, disabled by `SSDT-Disable_GPU_RP05.aml` |
| Memory | Capacity to be confirmed (does not affect the conclusions in this document) |
| Storage | NVMe SSD 512GB (GPT) |
| Wireless / Bluetooth | Realtek RTL8821CE combo card, PCI `10EC:C821` / subsystem `103C:831A`, Bluetooth USB `0BDA:B00A` |
| Audio | Realtek ALC (exact codec and layout-id to be confirmed; AppleALC is used) |
| Trackpad | I2C HID (VoodooI2C + VoodooI2CHID) |
| Keyboard | PS/2 (VoodooPS2Controller) |
| Internal display | 1920×1080 @ 60Hz, eDP / native LVDS connector |
| OS | macOS Tahoe 26.7.2 (Darwin 25.6.0, build `25G332`) |
| Bootloader | OpenCore 1.x (placeholder; actual version per the released EFI's `EFI/OC/OpenCore.efi`) |
| SMBIOS | `MacBookPro15,2` |

### Disk Partitions (GPT)

| Partition | Size | Notes |
|:---|---:|:---|
| ESP | 313MB | EFI System Partition |
| Windows | 53.7GB | Windows system partition |
| Data | 53.7GB | Partition name `Data` in this machine's GPT |
| Exchange | 7.1GB | Partition name `Exchange` in this machine's GPT |
| APFS | 287GB | macOS container |
| UBUNTUIMG | 8GB | Ubuntu image partition |

> Sizes are approximate values as displayed by Disk Utility; refer to `diskutil list` on the actual machine.

## Feature Matrix

| Feature | Status | Notes |
|:---|:---:|:---|
| Internal eDP display | ✅ | 1920×1080 @ 60Hz, native LVDS connector |
| HDMI output | ✅ | **2560×1440 @ 120Hz**; `con1 busId = 1` is the key |
| HDMI audio | ⚠️ To be verified by listening | Successful log shows `fEnableAudio=1`, but no independent speaker listening test has been recorded |
| Wi-Fi | ✅ | WPA3-Personal works; AirPortRTW 1.2.0 + `RealtekAirPortRTWFix` coexistence patch |
| Bluetooth | ✅ | `RealtekBluetoothFirmware` 1.0.2 + `BlueToolFixup` 2.7.3 |
| BLE HID keyboard | ✅ | `RealtekBTPatcher.kext` + `-ibtpwrap`; pairing verified on this machine |
| Xbox controller (Classic BR/EDR) | ✅ | No additional patch required |
| Trackpad / keyboard | ✅ | VoodooI2C / VoodooPS2 |
| Discrete GPU MX250 | 🚫 Disabled | `SSDT-Disable_GPU_RP05.aml`; no usable driver in macOS |
| 4K output | ❌ | This machine has no LSPCON; 4K30 does not enter the mode table (an override EDID DTD does not create a mode), and 4K60 is further limited by native TMDS / 300MHz; see the [HDMI document](./docs/en/HDMI-Fix-Notes.md#8-why-4k-is-still-unavailable) |
| Sleep / wake | ⚠️ To be verified | Full sleep, wake, and lid-close testing has not been performed on this machine |

## Quick Start

### 1. BIOS Settings

First complete the BIOS adjustments in [docs/en/BIOS-Settings.md](./docs/en/BIOS-Settings.md). Minimum requirements:

- **Secure Boot: Disabled**
- **VT-d: Disabled** (if it cannot be disabled, enable `DisableIoMapper` in OpenCore)
- **CFG Lock: Disabled** (if the BIOS hides this option, use `AppleXcpmCfgLock` as a workaround)
- **DVMT Pre-Allocated: 64MB** (if the BIOS lacks this item, do not blindly modify unknown offsets)
- **Fast Boot: Disabled**, **UEFI Only**, **USB Boot: Enabled**

### 2. Creating the Installer

Create a macOS Tahoe installation USB drive following the [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/). Note:

- Use the OpenCore version that matches this repository's EFI; record the actual version number in `CHANGELOG.md`.
- Copy `EFI/BOOT/BOOTx64.efi` and the entire `EFI/OC/` directory to the ESP partition of the USB drive.
- After copying, run `ocvalidate` to validate `config.plist`. Do not skip this step.

### 3. First Installation

1. Press HP's boot menu key at power-on and select the `OpenCore` entry on the USB drive.
2. In the OpenCore picker, perform **Reset NVRAM** first, then reboot into the installer.
3. Keep `-v debug=0x100 keepsyms=1` for the first boot to make troubleshooting easier.
4. Do not sign in to iCloud / iMessage during installation; finish the installation and confirm that the hardware works first.

### 4. Post-Installation

1. Copy the USB drive's EFI in full to the ESP partition of the internal NVMe drive.
2. Use OpenCore Configurator or ProperTree to regenerate **machine-unique** SMBIOS information (`SystemSerialNumber`, `MLB`, `SystemUUID`, `ROM`).
3. After confirming that Wi-Fi, Bluetooth, the trackpad, the internal display, and HDMI all work, consider enabling FileVault.
4. Re-run `ocvalidate` after every change to `config.plist`, and keep a bootable backup EFI.

> ⚠️ **Do not use the four SMBIOS values from someone else's EFI.** This repository does not publish real serial numbers, MLB, UUID, or ROM values.

## Key Configuration

### DeviceProperties

Path: `PciRoot(0x0)/Pci(0x2,0x0)` (Intel UHD 620)

| Key | Type | Value | Notes |
|:---|:---|:---|:---|
| `AAPL,ig-platform-id` | data | `0900A53E` (Base64 `CQClPg==`) | `0x3EA50009`, the platform ID used by the UHD 620 |
| `device-id` | data | `A53E0000` (Base64 `pT4AAA==`) | Spoofs native `0x3EA0` as `0x3EA5` |
| `framebuffer-patch-enable` | data | `01000000` | Allows WhateverGreen to apply framebuffer patches |
| `framebuffer-con0-alldata` | data | 36 bytes, see the table below | Writes all three connectors con0 / con1 / con2 in one go |
| `AAPL01,override-no-connect` | data | Real 128-byte EDID | Corresponds to FB1 / HDMI; raw bytes are not published here because they contain the display serial number |
| `framebuffer-fbmem` / `framebuffer-stolenmem` | data | As in the current config (if present) | Recommended for removal in the 4K30 "to be verified" plan; may be kept for 1080p / 1440p, see the [HDMI document](./docs/en/HDMI-Fix-Notes.md#86-proposed-4k30-configuration-to-be-verified) |

The complete 36 bytes of `framebuffer-con0-alldata`:

```text
00 00 08 00 02 00 00 00 98 00 00 00
01 01 09 00 00 08 00 00 C7 01 00 00
02 04 0A 00 00 04 00 00 C7 01 00 00
```

Base64 (can be pasted directly into OpenCore `<data>`):

```text
AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA
```

Broken down by connector:

| Connector | index | busId | pipe | type | flags | Purpose |
|:---:|---:|---:|---:|:---:|:---:|:---|
| con0 | 0 | 0 | 8 | `0x0002` (LVDS) | `0x0098` | eDP internal display |
| con1 | 1 | **1** | 9 | `0x0800` (HDMI) | `0x01C7` | HDMI, DDI B |
| con2 | 2 | 4 | 10 | `0x0400` (DP) | `0x01C7` | DP @ DDI C |

> `con1 busId = 1` is the core of the HDMI fix on this machine. It is not as simple as "DDI B, therefore write 1": WhateverGreen's `busId` corresponds to the GMBUS / DDC pin, and this machine's VBT has `ddc_pin=0x01`, which points to `DDC_BUS_DDI_B` and maps to `GMBUS_PIN_1` on the CNP generation. See the [HDMI fix notes](./docs/en/HDMI-Fix-Notes.md#3-how-vbt-maps-to-whatevergreen) for the full derivation.

### boot-args

```text
debug=0x100 keepsyms=1 -v -amfipassbeta -igfxblr -vi2c-force-polling -btlfxa2dpcheck -ibtcompatdbg igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100 -igfxmpc -ibtpwrap
```

| Argument | Purpose |
|:---|:---|
| `debug=0x100 keepsyms=1 -v` | Verbose boot log, keep symbols, text-mode boot; keep during troubleshooting |
| `-amfipassbeta` | Allows AMFIPass to load on Tahoe (AMFIPass beta-system switch) |
| `-igfxblr` | Enables WhateverGreen's laptop backlight register fix |
| `-vi2c-force-polling` | Forces VoodooI2C polling to improve I2C trackpad stability |
| `-btlfxa2dpcheck` | BlueToolFixup's Bluetooth audio A2DP compatibility check, preventing the audio channel from dropping too early |
| `-ibtcompatdbg` | `RealtekBTPatcher` debug output; can be removed once stable |
| `igfxonln=1` | Forces the display online; required for HDMI to light up |
| `igfxonlnfbs=0x0100` | Applies force-online to FB1 only; `0x0100` is the FB1 bitmask |
| `igfxfcms=1` | Forces modeset completion; required for HDMI to light up |
| `igfxfcmsfbs=0x0100` | Applies forced modeset to FB1 only |
| `-igfxmpc` | Bare flag that raises WhateverGreen's maximum pixel clock limit; **do not write `-igfxmpc=600`** |
| `-ibtpwrap` | Enables BLE event injection in `RealtekBTPatcher`; without it, hooks are only installed and events counted, with no HCI buffer modification |

> `-igfxmpc` is a switch, not a key-value argument. WhateverGreen does not parse `=600`; internally it raises the limit to **675,000,000Hz** by default. But 675MHz does not mean 4K60 is usable — see [HDMI document section 8](./docs/en/HDMI-Fix-Notes.md#8-why-4k-is-still-unavailable).

### Core Kexts / ACPI

| Function | Component | Version / Source |
|:---|:---|:---|
| Patching engine | `Lilu.kext` | Version per the released EFI's `EFI/OC/Kexts` |
| iGPU / HDMI | `WhateverGreen.kext` | Version per the released EFI's `EFI/OC/Kexts` |
| SMC / battery | `VirtualSMC.kext`, `SMCBatteryManager.kext`, etc. | Per this machine's configuration; versions to be confirmed |
| Internal audio | `AppleALC.kext` | Version and layout-id to be confirmed |
| Trackpad | `VoodooI2C.kext`, `VoodooI2CHID.kext` | Versions to be confirmed |
| Keyboard | `VoodooPS2Controller.kext` | Version to be confirmed |
| Wi-Fi | `AirPortRTW.kext` | 1.2.0, built with the coexistence patch from [RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) |
| Bluetooth firmware | `RealtekBluetoothFirmware.kext` | 1.0.2 official release binary |
| Bluetooth compatibility | `BlueToolFixup.kext` | 2.7.3 |
| BLE HID fix | `RealtekBTPatcher.kext` | [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher) |
| Disable MX250 | `SSDT-Disable_GPU_RP05.aml` | Custom ACPI for this machine |

> For any kext version not listed in the table, always refer to `EFI/OC/Kexts/*/Contents/Info.plist` and `CHANGELOG.md`; do not guess version numbers in the documentation.

## HDMI Output: The Short Version

This machine outputs HDMI through **DDI B / native TMDS**, and the VBT's DDC pin is `0x01`, so WhateverGreen's `con1 busId` must be set to **1** (not the commonly seen 5). It must also be paired with `igfxonln=1`, `igfxonlnfbs=0x0100`, `igfxfcms=1`, and `igfxfcmsfbs=0x0100`; otherwise `getDisplayStatus=0` and modeset is never triggered. The verified resolution is **2560×1440 @ 120Hz**.

For the full VBT extraction, 5-byte header / `child_dev_size=38` parsing, triple evidence, failed attempts, and the 4K limitation, see [docs/en/HDMI-Fix-Notes.md](./docs/en/HDMI-Fix-Notes.md).

## Wireless and Bluetooth

### Wi-Fi

- Driver: `AirPortRTW.kext` 1.2.0 (upstream [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW))
- Coexistence patch: [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix), which preserves AirPortRTW 1.2.0 functionality and fixes Wi-Fi / BT RF coexistence on the RTL8821CE
- Verified: **WPA3-Personal connects successfully**

### Bluetooth

- Firmware: `RealtekBluetoothFirmware.kext` 1.0.2 (official release binary)
- Compatibility layer: `BlueToolFixup.kext` 2.7.3
- BLE HID fix: `RealtekBTPatcher.kext` + `-ibtpwrap`
- Verified: BLE HID keyboard pairing succeeds; Xbox controller (Classic BR/EDR) needs no patch

For the Bluetooth fix root cause, HCI event injection bytes, and safety switch, see [docs/en/Bluetooth-BLE-HID-Fix.md](./docs/en/Bluetooth-BLE-HID-Fix.md). For the shared RF / antenna coexistence issues between Wi-Fi and Bluetooth, the incompatibility of `AirPort_RTW88` with `AirPortRTW`, and the requirement to use the official `RealtekBluetoothFirmware` release, see [docs/en/WiFi-Bluetooth-Coexistence.md](./docs/en/WiFi-Bluetooth-Coexistence.md).

## Known Issues

| Issue | Status | Notes / Workaround |
|:---|:---:|:---|
| 4K output unavailable | Known limitation | This machine has no LSPCON; 4K30 is theoretically possible, but the mode table does not read the override EDID's DTD, and 4K60 is further limited by native TMDS / the 300MHz `fMaxPixelClock`. See the [HDMI document](./docs/en/HDMI-Fix-Notes.md#8-why-4k-is-still-unavailable) |
| Sleep / wake unverified | To be verified | Full lid-close, sleep, wake, and external-display hot-plug tests have not been performed; verify before relying on them |
| Internal audio layout-id not recorded | To be confirmed | AppleALC is integrated, but the codec model and layout-id still need to be filled in from this machine's `config.plist` |
| HDMI audio not listened to | To be verified | The log shows `fEnableAudio=1`, but no independent speaker test result has been recorded |
| macOS updates may break patches | Common risk | `RealtekBTPatcher`, `RealtekAirPortRTWFix`, and WhateverGreen binary matching may break with system updates; back up the EFI before updating |
| Do not use `-igfxmpc=600` | Configuration trap | WEG only recognizes the bare flag and ignores the value; raising the limit to 675MHz does not solve 4K60 either |
| Do not add `CNConnectorAlwaysConnected` to HDMI | Configuration trap | `flags 0x0008` makes HDMI behave like an internal display; `port-number=0` conflicts with the internal display and causes a black screen |

> For more configuration traps that directly cause black screens, boot hangs, or a 20-minute boot, plus the `Cmd+S` single-user recovery procedure, see [docs/en/Troubleshooting-and-Fixes.md](./docs/en/Troubleshooting-and-Fixes.md).

## Repository Structure (at release)

```text
hp-efi-repo/
├── EFI/
│   ├── BOOT/
│   │   └── BOOTx64.efi
│   └── OC/
│       ├── ACPI/
│       │   └── SSDT-Disable_GPU_RP05.aml
│       ├── Drivers/
│       ├── Kexts/
│       ├── Resources/
│       ├── Tools/
│       └── config.plist
├── docs/
│   ├── HDMI-修复记录.md
│   ├── 蓝牙-BLE-HID-修复记录.md
│   ├── WiFi-与蓝牙共存.md
│   ├── 疑难问题与修复记录.md
│   └── BIOS-设置.md
├── README.md
├── CHANGELOG.md
└── LICENSE
```

> The documentation in this repository contains no real SMBIOS information, Apple account data, or display serial numbers. Before publishing the EFI, check the `PlatformInfo` section of `config.plist` and the EDID once more.

## Credits

The HDMI connector mapping, Bluetooth HID event injection, and Wi-Fi coexistence fixes in this EFI build on the following public research and projects. The order is arbitrary; thanks to all authors and maintainers.

### HDMI / iGPU

- [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen) — the foundation for Intel framebuffer fixes and connector injection.
  - [`kern_fb.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_fb.hpp): the `ConnectorInfo` / `ConnectorFlags` structures, which explain the semantics of `busId`, `type`, `flags`, and `CNConnectorAlwaysConnected (0x0008)`.
  - [`kern_igfx_clock.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_clock.cpp): `MaxPixelClockOverride`, the 675MHz default of `-igfxmpc`, and the `IOFBTimingRange.maxPixelClock` rewrite logic.
  - [`kern_igfx_debug.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_debug.cpp): DEBUG trace points such as `getOnlineInfo`, `getDisplayStatus`, and `connectionProbe`.
- [acidanthera/Lilu](https://github.com/acidanthera/Lilu) — the loading and patching framework for all Lilu plugins (WhateverGreen, AppleALC, RealtekBTPatcher, etc.).
- [acidanthera/bugtracker #420](https://github.com/acidanthera/bugtracker/issues/420#issuecomment-511499674) — CFL HDMI connector discussion; the comment explicitly records "changing the `bus-id` ... (`1` works for me)".
- [acidanthera/bugtracker #1377](https://github.com/acidanthera/bugtracker/issues/1377) — even with `enable-max-pixel-clock-override` enabled, a 4K panel may remain stuck at 4K@48, showing that max pixel clock alone is insufficient for 4K60.
- [acidanthera/bugtracker #2374](https://github.com/acidanthera/bugtracker/issues/2374) — `AAPL00,override-no-connect` only supports 128-byte EDIDs; 256-byte / DisplayID extensions are not read.
- [b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41) — a byte-by-byte analysis of the Tahoe `AppleIntelFramebuffer::getOnlineInfo` native HDMI branch, with immediate-value evidence for `movq $0x11E1A300` (300,000,000Hz).
- [z1000biker/HP-Desktop-Pro-G2-Tahoe-UHD630-HDMI-Fix](https://github.com/z1000biker/HP-Desktop-Pro-G2-Tahoe-UHD630-HDMI-Fix) — an HP + UHD 630 + Tahoe HDMI fix case demonstrating that `AAPL02,override-no-connect` must target the actual FB index.
- [shammu2bd/HP-15s-du1xxx-Hackintosh](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh) — a Tahoe EFI for the same HP 15s series; its [`config.plist`](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh/blob/main/EFI/OC/config.plist) contains `framebuffer-con1-busid=1`.
- [dreAd1811/HACKINTOSH-EFI-HP-15s-du0xxx](https://github.com/dreAd1811/HACKINTOSH-EFI-HP-15s-du0xxx) — an EFI reference for the HP 15s-du0xxx, including the con1 HDMI and con2 dummy connector approach.
- [daliansky/XiaoMi-Pro-Hackintosh issue #117](https://github.com/daliansky/XiaoMi-Pro-Hackintosh/issues/117) — community discussion of the UHD 620 under 4K30 and 64MB DVMT.
- [torvalds/linux](https://github.com/torvalds/linux) — the authoritative reference for VBT and DDI/DDC mapping:
  - [`intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h): `bdb_general_definitions`, `child_device_config`, `DEVICE_TYPE_HDMI`, `DDC_BUS_DDI_B`.
  - [`intel_bios.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_bios.c): child count calculation via `(block_size - sizeof(*defs)) / child_dev_size` and BDB version checks.
  - [`intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c): the CNP-generation `cnp_encoder_to_ddc_pin()` mapping of `PORT_B -> GMBUS_PIN_1`.
  - [`intel_gmbus.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_gmbus.h): definitions such as `GMBUS_PIN_1`.
- [intel-gpu-tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools) — [`intel_vbt_decode`](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/master/tools/intel_vbt_decode.c), used to decode `i915_vbt.bin` into human-readable VBT structures.

### Bluetooth / BLE HID

- [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) — zxystd's original `IntelBTPatcher`, which provides the `IOUSBHostDevice::deviceRequest` / `IOUSBHostPipe` hook structure.
  - [issue #514](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/514) — the original LE PHY / Data Length readiness analysis.
  - [PR #515](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/pull/515) — reference implementation for the three `IOUSBHostIOSource::io` paths.
  - [issue #497](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/497) — related discussion of BLE keyboard pairing failures on Tahoe.
- [if-lensman/Hackintosh-MacPro7-1-EFI](https://github.com/if-lensman/Hackintosh-MacPro7-1-EFI) — the `newIOSourceIOCompletion` / `Rjj` / `Bundled` three-path hooks and deduplication approach for Tahoe `IOUSBHostIOSource::io`.
- [thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware) — the RTL8821C firmware loader; this machine uses the 1.0.2 official release binary.
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) — upstream of `BlueToolFixup`, which makes third-party Bluetooth controllers acceptable to `bluetoothd`.
- [soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix](https://github.com/soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix) — demonstrates that `status=4803 isPairing=0` also appears in successful pairing logs, ruling out a misleading lead.
- [flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw) — a similar fix for the MT7922; its [PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3) provides an `HCI_LE_Set_Data_Length (0x2022)` alternative.
- [watchthelight-labs/WTLLegionBtFix](https://github.com/watchthelight-labs/WTLLegionBtFix) — a reference implementation of the `bluetoothd` patch on macOS 26.

### Wi-Fi

- [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW) — upstream of AirPortRTW, Realtek's native `IO80211` Wi-Fi driver.
- [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) — a patched build that preserves AirPortRTW 1.2.0 functionality and restores Realtek BT coexistence.

### Safari / WebKit

- [WebKit/WebKit `DisplayLinkMac.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/mac/DisplayLinkMac.cpp) — macOS display link and refresh-rate synchronization implementation.
- [WebKit/WebKit `Page.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Page.cpp) — page rendering update frequency and `DisplayLink` scheduling.
- [WebKit/WebKit `AnimationFrameRate.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/platform/graphics/AnimationFrameRate.cpp) — animation frame rate and the 60fps limiting logic.
- [WebKit/WebKit `UnifiedWebPreferences.yaml`](https://github.com/WebKit/WebKit/blob/main/Source/WTF/Scripts/Preferences/UnifiedWebPreferences.yaml) — definitions of WebKit preferences such as `PreferPageRenderingUpdatesNear60FPSEnabled`.
- [WebKit bug 173434](https://bugs.webkit.org/show_bug.cgi?id=173434) — related discussion of high-refresh-rate page rendering being capped.
- [kevmoo/flutter-wasm-compare issue #40](https://github.com/kevmoo/flutter-wasm-compare/issues/40) — a case study observing high refresh rates versus web rendering frame rates.
- [wailsapp/wails issue #6056](https://github.com/wailsapp/wails/issues/6056) — a real-world example of the WebKit 60fps limit's impact.

### Related Repositories by the Same Author

- [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher) — implementation and full technical write-up of this machine's BLE HID fix.
- [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) — Wi-Fi / Bluetooth coexistence patch.
- [Emilelu/BlueToolFixup-Fix26](https://github.com/Emilelu/BlueToolFixup-Fix26) — macOS 26.7 compatibility fix for BlueToolFixup (repository archived, retained for reference).

### Other Foundational Projects

- [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg) — the OpenCore bootloader and `ocvalidate`.
- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/) — installation and configuration baseline.
- [VoodooI2C/VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) — I2C HID trackpad driver.
- [acidanthera/VoodooPS2](https://github.com/acidanthera/VoodooPS2) — PS/2 keyboard / trackpad driver.
- [acidanthera/AppleALC](https://github.com/acidanthera/AppleALC) — Realtek ALC audio codec driver.
- [acidanthera/VirtualSMC](https://github.com/acidanthera/VirtualSMC) — SMC emulation and battery / sensor plugins.

## Disclaimer

This project is provided **"AS IS"**, without any express or implied warranty, including but not limited to the warranties of merchantability, fitness for a particular purpose, and non-infringement.

- Modifying the BIOS, flashing an EFI, and installing a non-Apple operating system involve risks of **bricking, data loss, hardware damage, and voiding the warranty**; back up everything before proceeding and prepare a recoverable Windows / Linux boot drive.
- This repository does not provide Apple software, macOS installation images, or any content that circumvents copyright; obtain the system through Apple's official channels.
- Third-party kexts, EFI drivers, and patches are governed by their respective upstream licenses; read the LICENSE of the corresponding repository before use.
- The configuration in this repository is only intended for the described HP 15s-dy0xxx / i7-8565U / RTL8821CE configuration; verify other models, other BIOS versions, or other macOS versions yourself.
- Items marked "To be verified" in the documentation do not yet have complete test conclusions and should not be treated as production commitments.

---

*The joy of Hackintosh is in understanding every detail that does not work. May this record save you a few detours.*
