> [English](./CHANGELOG.en.md)

# Changelog

本项目遵循 [Semantic Versioning](https://semver.org/)；日期使用本地日期（Asia/Shanghai）。

## [Unreleased]

### Planned

- 补齐 BIOS 版本、codec 型号与 AppleALC layout-id 的实测记录
- 完成睡眠 / 唤醒 / 合盖 / HDMI 热插拔的完整测试
- 补充 HDMI 音频实听结果
- 根据最终 EFI 二进制核对 OpenCore 与所有 kext 的实际版本号

## [1.0.0] - 2026-11-10

### Added

- 初始版本：HP 15s-dy0xxx（i7-8565U / UHD 620 / RTL8821CE）的 OpenCore EFI 文档集
- `README.md`：硬件 / 系统摘要、功能矩阵、快速开始、DeviceProperties、boot-args、已知问题与 Credits
- [`docs/HDMI-修复记录.md`](./docs/HDMI-修复记录.md)：VBT 提取与解析、`con1 busId=1` 的三重证据、36 字节连接器表、日志证据、失败尝试与 4K 限制分析
- [`docs/蓝牙-BLE-HID-修复记录.md`](./docs/蓝牙-BLE-HID-修复记录.md)：`LE_READY_WAIT_FOR_LENGTH` 根因、`RealtekBTPatcher` 三路 `IOUSBHostIOSource::io` hook、HCI 事件注入与验证结果
- [`docs/WiFi-与蓝牙共存.md`](./docs/WiFi-与蓝牙共存.md)：RTL8821CE 二合一卡的 RF 共存、驱动选型、`AirPort_RTW88` / `AirPortRTW` 不可同时加载、`RealtekBluetoothFirmware` 必须使用官方 release
- [`docs/疑难问题与修复记录.md`](./docs/疑难问题与修复记录.md)：`disable-agdc` 20 分钟开机、`CNConnectorAlwaysConnected` 黑屏、connector 字节错位、`framebuffer-patch-enable` 副作用、Safari 120Hz 等
- [`docs/BIOS-设置.md`](./docs/BIOS-设置.md)：Secure Boot、VT-d、CFG Lock、DVMT、启动项与隐藏选项处理建议
- [`docs/完整配置清单与已修复项.md`](./docs/完整配置清单与已修复项.md)：蓝牙 NVRAM、背光/开机黑屏、SSDT 清单、Booter/Misc Quirks、双系统与事故教训
- MIT `LICENSE`（Copyright © 2026 Emilelu）

### Verified

- macOS Tahoe 26.7.2（Darwin 25.6.0，build `25G332`）可启动
- Intel UHD 620 内屏 1920×1080 @ 60Hz
- HDMI 外接 2560×1440 @ 120Hz（`con1 busId=1` + force-online / force-modeset 组）
- Wi-Fi WPA3-Personal（AirPortRTW 1.2.0 + RealtekAirPortRTWFix）
- 蓝牙固件加载与 BLE HID 键盘配对（RealtekBluetoothFirmware 1.0.2 + BlueToolFixup 2.7.3 + RealtekBTPatcher）
- Xbox 手柄（Classic BR/EDR）无需补丁
- VoodooI2C 触控板 / VoodooPS2 键盘
- MX250 由 `SSDT-Disable_GPU_RP05.aml` 禁用

### Known Limitations

- 4K 输出未实现：本机无 LSPCON，Tahoe 原生 HDMI 分支存在 300MHz `fMaxPixelClock` 限制，4K30 亦受模式表限制
- 睡眠 / 唤醒尚未完整验证
- HDMI 音频仅有 `fEnableAudio=1` 日志，未做独立实听记录
- 内置音频 codec 型号与 layout-id 待核对
- OpenCore 版本以发布 EFI 的 `EFI/OC/OpenCore.efi` 为准，发布前请核对并记录

### Notes

- 本版本为文档仓库内容；EFI 二进制、SMBIOS 凭据与真实 EDID 原始字节不随文档公开
- 第三方 kext 的版本与许可证以各自上游仓库为准，详见 `README.md` 的 Credits 章节
