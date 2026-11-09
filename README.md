**简体中文** | [English](./README.en.md)

# Hackintosh-EFI-HP-15s-dy0xxx

> 机型：HP 15s-dy0xxx（HP 15s 系列）｜Intel Core i7-8565U｜macOS Tahoe 26.7.2
>
> OpenCore 1.x（占位；实际版本以发布 EFI 的 `EFI/OC/OpenCore.efi` 为准）｜SMBIOS `MacBookPro15,2`

本仓库记录一台 HP 15s-dy0xxx 在 macOS Tahoe 26.7.2 下的 OpenCore EFI 配置、验证结果与两项深度修复：**Intel UHD 620 原生 HDMI 输出**与 **Realtek RTL8821CE 的 BLE HID 键盘配对**。所有结论均来自本机实测；尚未验证的内容会明确标注「待验证」。

## 文档导航

| 文档 | 内容 |
|:---|:---|
| [docs/HDMI-修复记录.md](./docs/HDMI-修复记录.md) | 从 VBT 解析到 `con1 busId=1` 的完整 HDMI 修复过程、日志证据与 4K 限制分析 |
| [docs/蓝牙-BLE-HID-修复记录.md](./docs/蓝牙-BLE-HID-修复记录.md) | RTL8821CE 的 `LE_READY_WAIT_FOR_LENGTH` 根因与 `RealtekBTPatcher` 事件注入实现 |
| [docs/WiFi-与蓝牙共存.md](./docs/WiFi-与蓝牙共存.md) | RTL8821CE 二合一卡的 RF 共存、驱动选型与版本禁忌 |
| [docs/疑难问题与修复记录.md](./docs/疑难问题与修复记录.md) | `disable-agdc` 20 分钟开机、flags `0x0008` 黑屏、connector 字节错位、Safari 120Hz 等 |
| [docs/BIOS-设置.md](./docs/BIOS-设置.md) | 本机型建议的 BIOS 设置：DVMT、CFG Lock、VT-d、Secure Boot 等 |
| [docs/完整配置清单与已修复项.md](./docs/完整配置清单与已修复项.md) | **全量清单**：蓝牙 NVRAM（14 字节）、开机黑屏/背光修复、9 个 SSDT、Booter/Misc 全部 Quirks、双系统、事故教训、Lilu DEBUG 日志说明 |
| [CHANGELOG.md](./CHANGELOG.md) | 版本记录 |
| [LICENSE](./LICENSE) | MIT 许可证（Copyright © 2026 Emilelu） |

## 配置摘要

| 项目 | 配置 |
|:---|:---|
| 机型 | HP 15s-dy0xxx（HP 15s 系列） |
| CPU | Intel Core i7-8565U（Whiskey Lake，4 核 8 线程） |
| 核显 | Intel UHD Graphics 620，原生 `0x3EA0`，macOS 中 spoof 为 `0x3EA5` |
| 独显 | NVIDIA GeForce MX250，由 `SSDT-Disable_GPU_RP05.aml` 禁用 |
| 内存 | 容量待核对（不影响本文档结论） |
| 存储 | NVMe SSD 512GB（GPT） |
| 无线 / 蓝牙 | Realtek RTL8821CE 组合卡，PCI `10EC:C821` / subsystem `103C:831A`，蓝牙 USB `0BDA:B00A` |
| 声卡 | Realtek ALC（具体 codec 与 layout-id 待核对，使用 AppleALC） |
| 触控板 | I2C HID（VoodooI2C + VoodooI2CHID） |
| 键盘 | PS/2（VoodooPS2Controller） |
| 内屏 | 1920×1080 @ 60Hz，eDP / 原生 LVDS 连接器 |
| 系统 | macOS Tahoe 26.7.2（Darwin 25.6.0，build `25G332`） |
| 引导 | OpenCore 1.x（占位；实际版本以发布 EFI 的 `EFI/OC/OpenCore.efi` 为准） |
| SMBIOS | `MacBookPro15,2` |

### 磁盘分区（GPT）

| 分区 | 大小 | 备注 |
|:---|---:|:---|
| ESP | 313MB | EFI 系统分区 |
| Windows | 53.7GB | Windows 系统分区 |
| Data | 53.7GB | 本机 GPT 中的分区名 `Data` |
| Exchange | 7.1GB | 本机 GPT 中的分区名 `Exchange` |
| APFS | 287GB | macOS 容器 |
| UBUNTUIMG | 8GB | Ubuntu 镜像分区 |

> 容量为磁盘工具显示的近似值，实际以本机 `diskutil list` 为准。

## 功能矩阵

| 功能 | 状态 | 说明 |
|:---|:---:|:---|
| 内屏 eDP | ✅ | 1920×1080 @ 60Hz，原生 LVDS 连接器 |
| HDMI 外接 | ✅ | **2560×1440 @ 120Hz**；`con1 busId = 1` 是关键 |
| HDMI 音频 | ⚠️ 待实听验证 | 成功日志出现 `fEnableAudio=1`，但未做独立扬声器实听记录 |
| Wi-Fi | ✅ | WPA3-Personal 可用；AirPortRTW 1.2.0 + `RealtekAirPortRTWFix` 共存补丁 |
| 蓝牙 | ✅ | `RealtekBluetoothFirmware` 1.0.2 + `BlueToolFixup` 2.7.3 |
| BLE HID 键盘 | ✅ | `RealtekBTPatcher.kext` + `-ibtpwrap`，本机实测配对成功 |
| Xbox 手柄（Classic BR/EDR） | ✅ | 无需额外补丁 |
| 触控板 / 键盘 | ✅ | VoodooI2C / VoodooPS2 |
| 独立显卡 MX250 | 🚫 已禁用 | `SSDT-Disable_GPU_RP05.aml`，macOS 无可用驱动 |
| 4K 输出 | ❌ | 本机无 LSPCON；4K30 未进入模式表（override EDID 的 DTD 不会自动生成模式），4K60 另受原生 TMDS / 300MHz 限制，详见 [HDMI 文档](./docs/HDMI-修复记录.md#8-为什么-4k-仍然不可用) |
| 睡眠 / 唤醒 | ⚠️ 待验证 | 本机尚未做完整睡眠、唤醒、合盖测试 |

## 快速开始

### 1. BIOS 设置

先按 [docs/BIOS-设置.md](./docs/BIOS-设置.md) 完成 BIOS 调整。最低要求：

- **Secure Boot：Disabled**
- **VT-d：Disabled**（若无法关闭，则在 OpenCore 中启用 `DisableIoMapper`）
- **CFG Lock：Disabled**（若 BIOS 隐藏该选项，使用 `AppleXcpmCfgLock` 规避）
- **DVMT Pre-Allocated：64MB**（若 BIOS 无此项，先不要盲改未知偏移）
- **Fast Boot：Disabled**，**UEFI Only**，**USB Boot：Enabled**

### 2. 制作安装盘

按 [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/) 制作 macOS Tahoe 安装 U 盘。注意：

- 使用与本仓库 EFI 匹配的 OpenCore 版本；实际版本号请记录到 `CHANGELOG.md`。
- 将 `EFI/BOOT/BOOTx64.efi` 与 `EFI/OC/` 完整复制到 U 盘的 ESP 分区。
- 复制完成后运行 `ocvalidate` 校验 `config.plist`，不要跳过。

### 3. 首次安装

1. 开机按 HP 的启动菜单键，选择 U 盘的 `OpenCore` 启动项。
2. 在 OpenCore 选择器中先执行 **Reset NVRAM**，再重启进入安装器。
3. 首次启动保留 `-v debug=0x100 keepsyms=1`，便于定位问题。
4. 安装过程中不要登录 iCloud / iMessage；先完成安装并确认硬件工作正常。

### 4. 安装后

1. 将 U 盘 EFI 完整复制到内置 NVMe 的 ESP 分区。
2. 使用 OpenCore Configurator 或 ProperTree 重新生成**本机唯一**的 SMBIOS 信息（`SystemSerialNumber`、`MLB`、`SystemUUID`、`ROM`）。
3. 确认 Wi-Fi、蓝牙、触控板、内屏、HDMI 均正常后，再考虑开启 FileVault。
4. 每次修改 `config.plist` 后都重新运行 `ocvalidate`，并保留一个可启动的备用 EFI。

> ⚠️ **不要直接使用他人 EFI 中的 SMBIOS 四件套**。本仓库不发布真实序列号、MLB、UUID 或 ROM。

## 关键配置

### DeviceProperties

路径：`PciRoot(0x0)/Pci(0x2,0x0)`（Intel UHD 620）

| Key | 类型 | 值 | 说明 |
|:---|:---|:---|:---|
| `AAPL,ig-platform-id` | data | `0900A53E`（Base64 `CQClPg==`） | `0x3EA50009`，UHD 620 使用的平台 ID |
| `device-id` | data | `A53E0000`（Base64 `pT4AAA==`） | 将原生 `0x3EA0` spoof 为 `0x3EA5` |
| `framebuffer-patch-enable` | data | `01000000` | 允许 WhateverGreen 应用 framebuffer patch |
| `framebuffer-con0-alldata` | data | 36 字节，见下表 | 一次写入 con0 / con1 / con2 三个连接器 |
| `AAPL01,override-no-connect` | data | 真实 128 字节 EDID | 对应 FB1 / HDMI；原始字节因含显示器序列号不在此公开 |
| `framebuffer-fbmem` / `framebuffer-stolenmem` | data | 以当前 config 为准（若存在） | 4K30 待验证方案建议删除；1080p / 1440p 使用时可保留，详见 [HDMI 文档](./docs/HDMI-修复记录.md#86-待验证的-4k30-方案) |

`framebuffer-con0-alldata` 的完整 36 字节：

```text
00 00 08 00 02 00 00 00 98 00 00 00
01 01 09 00 00 08 00 00 C7 01 00 00
02 04 0A 00 00 04 00 00 C7 01 00 00
```

Base64（可直接填入 OpenCore `<data>`）：

```text
AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA
```

按连接器拆解：

| 连接器 | index | busId | pipe | type | flags | 用途 |
|:---:|---:|---:|---:|:---:|:---:|:---|
| con0 | 0 | 0 | 8 | `0x0002`（LVDS） | `0x0098` | eDP 内屏 |
| con1 | 1 | **1** | 9 | `0x0800`（HDMI） | `0x01C7` | HDMI，DDI B |
| con2 | 2 | 4 | 10 | `0x0400`（DP） | `0x01C7` | DP @ DDI C |

> `con1 busId = 1` 是本机 HDMI 修复的核心。它不是「DDI B 所以写 1」这么简单：WhateverGreen 的 `busId` 对应 GMBUS / DDC pin，本机 VBT 的 `ddc_pin=0x01` 指向 `DDC_BUS_DDI_B`，在 CNP 世代映射到 `GMBUS_PIN_1`。完整推导见 [HDMI 修复记录](./docs/HDMI-修复记录.md#3-vbt-如何映射到-whatevergreen)。

### boot-args

```text
debug=0x100 keepsyms=1 -v -amfipassbeta -igfxblr -vi2c-force-polling -btlfxa2dpcheck -ibtcompatdbg igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100 -igfxmpc -ibtpwrap
```

| 参数 | 作用 |
|:---|:---|
| `debug=0x100 keepsyms=1 -v` | 详细启动日志、保留符号、文字模式启动；排障期保留 |
| `-amfipassbeta` | 允许 AMFIPass 在 Tahoe 上加载（AMFIPass 的 beta 系统开关） |
| `-igfxblr` | 启用 WhateverGreen 的笔记本背光寄存器修复 |
| `-vi2c-force-polling` | VoodooI2C 强制轮询，提升 I2C 触控板稳定性 |
| `-btlfxa2dpcheck` | BlueToolFixup 的蓝牙音频 A2DP 兼容检查，避免音频通道过早断开 |
| `-ibtcompatdbg` | `RealtekBTPatcher` 调试输出；稳定后可移除 |
| `igfxonln=1` | 强制显示在线（force online），HDMI 点亮必需 |
| `igfxonlnfbs=0x0100` | 仅对 FB1 应用 force-online；`0x0100` 是 FB1 的位掩码 |
| `igfxfcms=1` | 强制完成 modeset，HDMI 点亮必需 |
| `igfxfcmsfbs=0x0100` | 仅对 FB1 应用强制 modeset |
| `-igfxmpc` | 裸标志，提高 WhateverGreen 的最大像素时钟限制；**不要写 `-igfxmpc=600`** |
| `-ibtpwrap` | 启用 `RealtekBTPatcher` 的 BLE 事件注入；不加时只安装 hook、只计数，不修改 HCI buffer |

> `-igfxmpc` 是开关，不是键值参数。WhateverGreen 不解析 `=600`；其内部默认将上限设为 **675,000,000Hz**。但 675MHz 并不等于 4K60 可用，原因见 [HDMI 文档第 8 节](./docs/HDMI-修复记录.md#8-为什么-4k-仍然不可用)。

### 核心 Kext / ACPI

| 功能 | 组件 | 版本 / 来源 |
|:---|:---|:---|
| 补丁引擎 | `Lilu.kext` | 版本以发布 EFI 的 `EFI/OC/Kexts` 为准 |
| 核显 / HDMI | `WhateverGreen.kext` | 版本以发布 EFI 的 `EFI/OC/Kexts` 为准 |
| SMC / 电池 | `VirtualSMC.kext`、`SMCBatteryManager.kext` 等 | 按本机配置，版本待核对 |
| 内置音频 | `AppleALC.kext` | 版本与 layout-id 待核对 |
| 触控板 | `VoodooI2C.kext`、`VoodooI2CHID.kext` | 版本待核对 |
| 键盘 | `VoodooPS2Controller.kext` | 版本待核对 |
| Wi-Fi | `AirPortRTW.kext` | 1.2.0，使用 [RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) 的共存补丁构建 |
| 蓝牙固件 | `RealtekBluetoothFirmware.kext` | 1.0.2 官方 release 二进制 |
| 蓝牙兼容 | `BlueToolFixup.kext` | 2.7.3 |
| BLE HID 修复 | `RealtekBTPatcher.kext` | [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher) |
| 禁用 MX250 | `SSDT-Disable_GPU_RP05.aml` | 本机 ACPI 定制 |

> 表中未给出的 kext 版本一律以 `EFI/OC/Kexts/*/Contents/Info.plist` 与 `CHANGELOG.md` 为准，不在文档中猜测版本号。

## HDMI 外接：一句话结论

本机 HDMI 由 **DDI B / 原生 TMDS** 输出，VBT 的 DDC pin 为 `0x01`，因此 WhateverGreen 的 `con1 busId` 必须写成 **1**（不是常见的 5）。同时必须配合 `igfxonln=1`、`igfxonlnfbs=0x0100`、`igfxfcms=1`、`igfxfcmsfbs=0x0100`，否则 `getDisplayStatus=0`，不会触发 modeset。实测分辨率 **2560×1440 @ 120Hz**。

完整的 VBT 提取、5 字节头 / `child_dev_size=38` 解析、三重证据、失败尝试与 4K 限制，见 [docs/HDMI-修复记录.md](./docs/HDMI-修复记录.md)。

## 无线与蓝牙

### Wi-Fi

- 驱动：`AirPortRTW.kext` 1.2.0（上游 [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW)）
- 共存补丁：[Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix)，保留 AirPortRTW 1.2.0 功能并修复 RTL8821CE 的 Wi-Fi / BT RF 共存
- 实测：**WPA3-Personal 可连接**

### 蓝牙

- 固件：`RealtekBluetoothFirmware.kext` 1.0.2（官方 release 二进制）
- 兼容层：`BlueToolFixup.kext` 2.7.3
- BLE HID 修复：`RealtekBTPatcher.kext` + `-ibtpwrap`
- 实测：BLE HID 键盘配对成功；Xbox 手柄（Classic BR/EDR）无需补丁

蓝牙修复的根因、HCI 事件注入字节与安全开关，见 [docs/蓝牙-BLE-HID-修复记录.md](./docs/蓝牙-BLE-HID-修复记录.md)。Wi-Fi 与蓝牙共用 RF / 天线的共存问题、`AirPort_RTW88` 与 `AirPortRTW` 不可同时加载、`RealtekBluetoothFirmware` 必须使用官方 release 等注意事项，见 [docs/WiFi-与蓝牙共存.md](./docs/WiFi-与蓝牙共存.md)。

## 已知问题

| 问题 | 状态 | 说明 / 绕过 |
|:---|:---:|:---|
| 4K 输出不可用 | 已知限制 | 本机无 LSPCON，且 Tahoe 原生 HDMI 分支存在 300MHz `fMaxPixelClock` 限制；4K30 理论可行但模式表不读取 override EDID 的 DTD。详见 [HDMI 文档](./docs/HDMI-修复记录.md#8-为什么-4k-仍然不可用) |
| HiDPI 缩放受限 | 已知限制 | 内屏 1920×1080 若开启 HiDPI，逻辑分辨率仅为 960×540，实际可用性有限；外接 4K 的 HiDPI 又依赖 4K 输出先可用。本项目未把 HiDPI 作为目标配置 |
| 睡眠 / 唤醒未验证 | 待验证 | 尚未进行完整合盖、睡眠、唤醒、外接显示器热插拔测试；请自行验证后再依赖 |
| 内置音频 layout-id 未记录 | 待核对 | AppleALC 已集成，但 codec 型号与 layout-id 需要从本机 `config.plist` 补齐 |
| HDMI 音频未实听 | 待验证 | 日志有 `fEnableAudio=1`，但未记录独立扬声器测试结果 |
| macOS 更新可能破坏补丁 | 常见风险 | `RealtekBTPatcher`、`RealtekAirPortRTWFix` 与 WhateverGreen 的二进制匹配可能随系统更新失效；更新前备份 EFI |
| 不要使用 `-igfxmpc=600` | 配置陷阱 | WEG 只识别裸标志，数值被忽略；提升到 675MHz 也不解决 4K60 |
| 不要给 HDMI 加 `CNConnectorAlwaysConnected` | 配置陷阱 | `flags 0x0008` 会让 HDMI 也按内屏处理，port-number=0 与内屏冲突，导致黑屏 |

> 更多会直接导致黑屏、卡启动或 20 分钟开机的配置陷阱，以及 `Cmd+S` 单用户恢复流程，见 [docs/疑难问题与修复记录.md](./docs/疑难问题与修复记录.md)。

## 仓库结构（发布时）

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

> 本仓库的文档部分不包含任何真实 SMBIOS 信息、Apple 账号数据或显示器序列号。发布 EFI 前请再次检查 `config.plist` 的 `PlatformInfo` 与 EDID。

## Credits / 致谢

本 EFI 的 HDMI 连接器映射、蓝牙 HID 事件注入与 Wi-Fi 共存修补，建立在以下公开研究与项目之上。排名不分先后，感谢所有作者与维护者。

### HDMI / 核显

- [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen) —— Intel framebuffer 修复与连接器注入的基础。
  - [`kern_fb.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_fb.hpp)：`ConnectorInfo` / `ConnectorFlags` 结构，解释 `busId`、`type`、`flags` 与 `CNConnectorAlwaysConnected (0x0008)` 的语义。
  - [`kern_igfx_clock.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_clock.cpp)：`MaxPixelClockOverride`，`-igfxmpc` 的 675MHz 默认值与 `IOFBTimingRange.maxPixelClock` 改写逻辑。
  - [`kern_igfx_debug.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_debug.cpp)：`getOnlineInfo`、`getDisplayStatus`、`connectionProbe` 等 DEBUG 追踪点。
- [acidanthera/Lilu](https://github.com/acidanthera/Lilu) —— 所有 Lilu 插件（WhateverGreen、AppleALC、RealtekBTPatcher 等）的加载与 patch 框架。
- [acidanthera/bugtracker #420](https://github.com/acidanthera/bugtracker/issues/420#issuecomment-511499674) —— CFL HDMI 连接器讨论；评论中明确记录「changing the `bus-id` ... (`1` works for me)」。
- [acidanthera/bugtracker #1377](https://github.com/acidanthera/bugtracker/issues/1377) —— 即使启用 `enable-max-pixel-clock-override`，4K 面板仍可能停留在 4K@48，说明单靠 max pixel clock 不足以解决 4K60。
- [acidanthera/bugtracker #2374](https://github.com/acidanthera/bugtracker/issues/2374) —— `AAPL00,override-no-connect` 只支持 128 字节 EDID，256 字节 / DisplayID 扩展不会被读取。
- [b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41) —— 对 Tahoe `AppleIntelFramebuffer::getOnlineInfo` native HDMI 分支的逐字节分析，给出 `movq $0x11E1A300`（300,000,000Hz）立即数证据。
- [z1000biker/HP-Desktop-Pro-G2-Tahoe-UHD630-HDMI-Fix](https://github.com/z1000biker/HP-Desktop-Pro-G2-Tahoe-UHD630-HDMI-Fix) —— HP + UHD 630 + Tahoe 的 HDMI 修复案例，示范 `AAPL02,override-no-connect` 必须对准实际 FB index。
- [shammu2bd/HP-15s-du1xxx-Hackintosh](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh) —— HP 15s 同系列 Tahoe EFI；其 [`config.plist`](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh/blob/main/EFI/OC/config.plist) 中可见 `framebuffer-con1-busid=1`。
- [dreAd1811/HACKINTOSH-EFI-HP-15s-du0xxx](https://github.com/dreAd1811/HACKINTOSH-EFI-HP-15s-du0xxx) —— HP 15s-du0xxx 的 EFI 参考，包含 con1 HDMI 与 con2 dummy 连接器思路。
- [daliansky/XiaoMi-Pro-Hackintosh issue #117](https://github.com/daliansky/XiaoMi-Pro-Hackintosh/issues/117) —— UHD 620 在 4K30 与 64MB DVMT 条件下的社区讨论。
- [torvalds/linux](https://github.com/torvalds/linux) —— VBT 与 DDI/DDC 映射的权威参考：
  - [`intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h)：`bdb_general_definitions`、`child_device_config`、`DEVICE_TYPE_HDMI`、`DDC_BUS_DDI_B`。
  - [`intel_bios.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_bios.c)：`(block_size - sizeof(*defs)) / child_dev_size` 的 child 数计算与 BDB 版本校验。
  - [`intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c)：CNP 世代 `cnp_encoder_to_ddc_pin()` 的 `PORT_B -> GMBUS_PIN_1` 映射。
  - [`intel_gmbus.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_gmbus.h)：`GMBUS_PIN_1` 等 pin 定义。
- [intel-gpu-tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools) —— [`intel_vbt_decode`](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/master/tools/intel_vbt_decode.c)，用于把 `i915_vbt.bin` 解码为人类可读的 VBT 结构。

### 蓝牙 / BLE HID

- [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) —— zxystd 原版 `IntelBTPatcher`，提供 `IOUSBHostDevice::deviceRequest` / `IOUSBHostPipe` 的 hook 结构。
  - [issue #514](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/514) —— 原始 LE PHY / Data Length readiness 分析。
  - [PR #515](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/pull/515) —— 三条 `IOUSBHostIOSource::io` 路径的参考实现。
  - [issue #497](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/497) —— Tahoe 上 BLE 键盘配对失败的相关讨论。
- [if-lensman/Hackintosh-MacPro7-1-EFI](https://github.com/if-lensman/Hackintosh-MacPro7-1-EFI) —— Tahoe `IOUSBHostIOSource::io` 的 `newIOSourceIOCompletion` / `Rjj` / `Bundled` 三路 hook 与去重思路。
- [thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware) —— RTL8821C 固件加载器，本机使用 1.0.2 官方 release 二进制。
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) —— `BlueToolFixup` 上游，使第三方蓝牙控制器可被 `bluetoothd` 接受。
- [soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix](https://github.com/soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix) —— 证明 `status=4803 isPairing=0` 在成功配对日志中同样会出现，排除误导性线索。
- [flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw) —— MT7922 的同类修复；其 [PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3) 提供 `HCI_LE_Set_Data_Length (0x2022)` 替代方案。
- [watchthelight-labs/WTLLegionBtFix](https://github.com/watchthelight-labs/WTLLegionBtFix) —— macOS 26 上 `bluetoothd` patch 的参考实现。

### Wi-Fi

- [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW) —— AirPortRTW 上游，Realtek 原生 `IO80211` Wi-Fi 驱动。
- [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) —— 保留 AirPortRTW 1.2.0 功能并恢复 Realtek BT 共存的补丁构建。

### Safari / WebKit

- [WebKit/WebKit `DisplayLinkMac.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/mac/DisplayLinkMac.cpp) —— macOS 显示链接与刷新率同步实现。
- [WebKit/WebKit `Page.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Page.cpp) —— 页面渲染更新频率与 `DisplayLink` 调度。
- [WebKit/WebKit `AnimationFrameRate.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/platform/graphics/AnimationFrameRate.cpp) —— 动画帧率与 60fps 限制相关逻辑。
- [WebKit/WebKit `UnifiedWebPreferences.yaml`](https://github.com/WebKit/WebKit/blob/main/Source/WTF/Scripts/Preferences/UnifiedWebPreferences.yaml) —— `PreferPageRenderingUpdatesNear60FPSEnabled` 等 WebKit 偏好定义。
- [WebKit bug 173434](https://bugs.webkit.org/show_bug.cgi?id=173434) —— 高刷新率页面渲染被限制的相关讨论。
- [kevmoo/flutter-wasm-compare issue #40](https://github.com/kevmoo/flutter-wasm-compare/issues/40) —— 高刷新率与 Web 渲染帧率观测案例。
- [wailsapp/wails issue #6056](https://github.com/wailsapp/wails/issues/6056) —— WebKit 60fps 限制的实际影响案例。

### 同作者相关仓库

- [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher) —— 本机 BLE HID 修复的实现与完整技术说明。
- [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) —— Wi-Fi / 蓝牙共存补丁。
- [Emilelu/BlueToolFixup-Fix26](https://github.com/Emilelu/BlueToolFixup-Fix26) —— BlueToolFixup 的 macOS 26.7 兼容修复（仓库已归档，保留作为参考）。

### 其他基础项目

- [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg) —— OpenCore 引导器与 `ocvalidate`。
- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/) —— 安装与配置基线。
- [VoodooI2C/VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) —— I2C HID 触控板驱动。
- [acidanthera/VoodooPS2](https://github.com/acidanthera/VoodooPS2) —— PS/2 键盘 / 触控板驱动。
- [acidanthera/AppleALC](https://github.com/acidanthera/AppleALC) —— Realtek ALC 音频编解码驱动。
- [acidanthera/VirtualSMC](https://github.com/acidanthera/VirtualSMC) —— SMC 模拟与电池 / 传感器插件。

## 免责声明

本项目按 **“现状”（AS IS）** 提供，不附带任何明示或暗示的担保，包括但不限于适销性、特定用途适用性与不侵权担保。

- 修改 BIOS、刷写 EFI、安装非 Apple 操作系统存在**变砖、数据丢失、硬件损坏、保修失效**的风险；请在操作前完整备份，并准备可恢复的 Windows / Linux 启动盘。
- 本仓库不提供 Apple 软件、macOS 安装镜像或任何规避版权的内容；请通过 Apple 官方渠道获取系统。
- 第三方 kext、EFI 驱动与补丁遵循各自的上游许可证；使用前请阅读对应仓库的 LICENSE。
- 本仓库中的配置仅对所述 HP 15s-dy0xxx / i7-8565U / RTL8821CE 配置负责；其他机型、其他 BIOS 版本或其他 macOS 版本请自行验证。
- 文档中的「待验证」项尚未获得完整测试结论，不应作为生产环境承诺。

---

*黑苹果的乐趣在于把每一个不工作的细节弄明白。愿这份记录能帮你少走一点弯路。*
