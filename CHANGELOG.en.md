[简体中文](./CHANGELOG.md) | **English**

# Changelog

This project follows [Semantic Versioning](https://semver.org/); dates use local time (Asia/Shanghai).

## [Unreleased]

### Planned

- Complete the measured records for the BIOS version, codec model, and AppleALC layout-id
- Finish full testing of sleep / wake / lid close / HDMI hot-plug
- Add HDMI audio listening test results
- Verify the actual version numbers of OpenCore and all kexts against the final EFI binaries

## [1.0.0] - 2026-11-10

### Added

- Initial release: OpenCore EFI documentation set for the HP 15s-dy0xxx (i7-8565U / UHD 620 / RTL8821CE)
- `README.md`: hardware / system summary, feature matrix, quick start, DeviceProperties, boot-args, known issues, and Credits
- [`docs/en/HDMI-Fix-Notes.md`](./docs/en/HDMI-Fix-Notes.md): VBT extraction and parsing, triple evidence for `con1 busId=1`, the 36-byte connector table, log evidence, failed attempts, and the 4K limitation analysis
- [`docs/en/Bluetooth-BLE-HID-Fix.md`](./docs/en/Bluetooth-BLE-HID-Fix.md): `LE_READY_WAIT_FOR_LENGTH` root cause, the three `IOUSBHostIOSource::io` hooks in `RealtekBTPatcher`, HCI event injection, and verification results
- [`docs/en/WiFi-Bluetooth-Coexistence.md`](./docs/en/WiFi-Bluetooth-Coexistence.md): RF coexistence for the RTL8821CE combo card, driver selection, the incompatibility of `AirPort_RTW88` / `AirPortRTW`, and the requirement to use the official `RealtekBluetoothFirmware` release
- [`docs/en/Troubleshooting-and-Fixes.md`](./docs/en/Troubleshooting-and-Fixes.md): `disable-agdc` 20-minute boot, `CNConnectorAlwaysConnected` black screen, connector byte misalignment, `framebuffer-patch-enable` side effects, Safari 120Hz, and more
- [`docs/en/BIOS-Settings.md`](./docs/en/BIOS-Settings.md): recommended settings for Secure Boot, VT-d, CFG Lock, DVMT, boot entries, and hidden options
- [`docs/en/Complete-Configuration-and-Fixes.md`](./docs/en/Complete-Configuration-and-Fixes.md): Bluetooth NVRAM, backlight / boot black screen, SSDT list, Booter/Misc quirks, dual boot, and incident lessons
- MIT `LICENSE` (Copyright © 2026 Emilelu)

### Verified

- macOS Tahoe 26.7.2 (Darwin 25.6.0, build `25G332`) boots
- Intel UHD 620 internal display 1920×1080 @ 60Hz
- HDMI output 2560×1440 @ 120Hz (`con1 busId=1` + force-online / force-modeset group)
- Wi-Fi WPA3-Personal (AirPortRTW 1.2.0 + RealtekAirPortRTWFix)
- Bluetooth firmware loading and BLE HID keyboard pairing (RealtekBluetoothFirmware 1.0.2 + BlueToolFixup 2.7.3 + RealtekBTPatcher)
- Xbox controller (Classic BR/EDR) requires no patch
- VoodooI2C trackpad / VoodooPS2 keyboard
- MX250 disabled by `SSDT-Disable_GPU_RP05.aml`

### Known Limitations

- 4K output not implemented: this machine has no LSPCON, the Tahoe native HDMI path has a 300MHz `fMaxPixelClock` limit, and 4K30 is also restricted by the mode table
- Sleep / wake has not been fully verified
- HDMI audio only has the `fEnableAudio=1` log entry; no independent listening test has been recorded
- Internal audio codec model and layout-id to be confirmed
- The OpenCore version is determined by the released EFI's `EFI/OC/OpenCore.efi`; verify and record it before release

### Notes

- This release covers documentation repository content only; EFI binaries, SMBIOS credentials, and raw bytes of the real EDID are not published with the documentation
- Version and license information for third-party kexts is governed by their respective upstream repositories; see the Credits section of `README.md` for details
