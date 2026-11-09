> [English](./en/WiFi-Bluetooth-Coexistence.md)

# Wi-Fi 与蓝牙共存：RTL8821CE 二合一卡的取舍与调优

> 返回 [主 README](../README.md)｜机型：HP 15s-dy0xxx / RTL8821CE / macOS Tahoe 26.7.2

RTL8821CE 是一颗 **Wi-Fi + Bluetooth 二合一** 的 combo 芯片：Wi-Fi 走 PCIe，蓝牙走 USB，但两者共享同一套 RF 前端、天线与共存仲裁逻辑。macOS 下 Wi-Fi 由 `AirPortRTW` 驱动，蓝牙由 `RealtekBluetoothFirmware` + `BlueToolFixup` + `RealtekBTPatcher` 驱动；它们是两套独立项目，默认不会像 Realtek 厂商驱动那样协调共存。本文记录本机的共存问题、修复组合与版本禁忌。

## 1. 硬件与驱动栈

| 功能 | 总线 | 硬件 ID | 驱动 / 组件 |
|:---|:---|:---|:---|
| Wi-Fi | PCIe | `10EC:C821` / subsystem `103C:831A` | `AirPortRTW.kext` 1.2.0 + `RealtekAirPortRTWFix` |
| 蓝牙 | USB | `0BDA:B00A` | `RealtekBluetoothFirmware.kext` 1.0.2（官方 release）+ `BlueToolFixup.kext` 2.7.3 + `RealtekBTPatcher.kext` |
| RF / 天线 | — | 2.4GHz / 5GHz 共用 | 硬件共存仲裁，驱动侧不可完全隔离 |

> 二合一卡的关键事实：**Wi-Fi 与蓝牙不能同时满负荷使用同一频段**。2.4GHz Wi-Fi 与蓝牙音频 / BLE 的冲突最明显；5GHz Wi-Fi 与蓝牙的频段间隔大，实际影响小得多。

## 2. 典型症状

| 症状 | 触发条件 | 说明 |
|:---|:---|:---|
| 蓝牙音频断续 / 爆音 / 断连 | Wi-Fi 大流量下载、Speedtest、局域网拷贝 | 2.4GHz Wi-Fi 占用 RF 时间片，蓝牙 A2DP 排不上队 |
| BLE HID 键盘延迟、掉键、偶发断连 | Wi-Fi 扫描或连接新 AP 时 | 主动扫描会短时抢占射频与天线 |
| Wi-Fi 速率骤降 | 蓝牙音频持续播放 / BLE 扫描 | 共存仲裁把部分时间片让给蓝牙 |
| 睡眠唤醒后 Wi-Fi 或蓝牙单侧失效 | 合盖唤醒、长时间睡眠 | 两套驱动的电源状态恢复顺序不一致，RF 未重新初始化 |
| 蓝牙能连但无数据 | 使用自编译 `RealtekBluetoothFirmware` 时 | 固件 RX 路径异常，表现为零广播、零扫描结果 |

> 如果蓝牙故障只在 Wi-Fi 繁忙时出现、空闲时恢复正常，优先按“共存”处理；如果空闲时也完全无广播，按蓝牙驱动 / 固件问题处理，见 [蓝牙 BLE HID 修复记录](./蓝牙-BLE-HID-修复记录.md)。

## 3. 根因

1. **共享 RF 与天线**：RTL8821CE 的 Wi-Fi 与 BT 共用前端。2.4GHz 频段上，Wi-Fi 的 OFDM 帧与 BT 的跳频时分复用，硬件共存仲裁只能在两者之间分配时间片，不能真正做到“同时满速”。
2. **两套 macOS 驱动缺少厂商级共存服务**：`AirPortRTW` 是社区维护的 Realtek Wi-Fi 驱动，`RealtekBluetoothFirmware` 是独立的蓝牙固件加载器；二者不共享 Realtek 原厂 Windows 驱动中的 coexistence 状态机。Wi-Fi 初始化、扫描、信道切换时，蓝牙侧不会收到完整的“即将占用射频”通知。
3. **扫描是强抢占**：macOS 的 Wi-Fi 扫描会遍历信道，扫描期间蓝牙音频和 BLE 连接最容易受影响；这就是“打开 Wi-Fi 设置页面就卡蓝牙”的原因。
4. **睡眠恢复顺序不一致**：唤醒时 Wi-Fi PCIe 设备与蓝牙 USB 设备的电源恢复、固件重载顺序不确定；任一侧先完成而另一侧未就绪，就会出现“一半能用、一半不能用”。

## 4. 修复组合

### 4.1 Wi-Fi：AirPortRTW 1.2.0 + RealtekAirPortRTWFix

- **`AirPortRTW.kext` 1.2.0**：上游 [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW)，提供 RTL8821CE 的 macOS 原生 `IO80211` Wi-Fi 接口。
- **`RealtekAirPortRTWFix`**：[Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix)，在保留 1.2.0 功能的基础上修复 RTL8821CE 的 Wi-Fi / BT 射频共存问题。实际使用时按该仓库的说明，把修复合并进 `AirPortRTW.kext` 构建或作为对应补丁加载，**不要同时保留两个来源不明的 `AirPortRTW` 副本**。
- 本机验证：**WPA3-Personal 可正常连接**，Wi-Fi 繁忙时蓝牙音频 / BLE 的断连频率明显下降。

### 4.2 蓝牙：官方 release 二进制是硬性要求

| 组件 | 版本 | 要求 |
|:---|:---|:---|
| `RealtekBluetoothFirmware.kext` | **1.0.2** | **必须使用官方 release 二进制**，不要用自行编译的版本 |
| `BlueToolFixup.kext` | 2.7.3 | 使用与 Tahoe 匹配的版本，保证 `bluetoothd` 接受第三方控制器 |
| `RealtekBTPatcher.kext` | 同作者仓库 release | BLE HID 事件注入补丁，配合 `-ibtpwrap` |

> **实测教训：自编译的 `RealtekBluetoothFirmware` 会导致蓝牙 RX 完全失效**——控制器能枚举、能上传固件，但扫描不到任何广播（零广播、零设备），BLE 键盘自然也无法配对。原因与自编译产物中的固件 / config 资源打包或下载后状态处理有关；不要在这上面节省时间，直接用官方 1.0.2 release 二进制。

### 4.3 绝不能同时加载的两个 Wi-Fi kext

**不要同时加载 `AirPort_RTW88.kext` 与 `AirPortRTW.kext`。**

- 两者都试图匹配 Realtek 的同一 PCI 设备（`10EC:C821`）；同时加载会出现重复匹配、抢占 `IOService`、随机哪个驱动先绑定，表现为 Wi-Fi 时好时坏、开机卡住、甚至 kernel panic。
- `AirPort_RTW88` 面向 RTW88 系列（如 RTL8822CE 等），其设备 ID 表与 RTL8821CE 的 `AirPortRTW` 路径不同；混用没有收益。
- 只保留 `AirPortRTW.kext` 1.2.0（含 `RealtekAirPortRTWFix` 修复）这一套。

## 5. 推荐加载顺序

`Kernel → Add` 中建议保持以下相对顺序（具体 `BundlePath` 以你的 `config.plist` 为准）：

```text
Lilu.kext
AirPortRTW.kext                 (1.2.0 + RealtekAirPortRTWFix 构建)
RealtekBluetoothFirmware.kext   (1.0.2 官方 release)
BlueToolFixup.kext              (2.7.3)
RealtekBTPatcher.kext
```

> 所有 Lilu 插件必须位于 `Lilu.kext` 之后；Wi-Fi 与蓝牙之间没有强依赖，但把固件加载器放在协议栈补丁之前便于排查。每次只改一个 kext，改完用 `ocvalidate` 校验并保留备用 EFI。

## 6. 验证方法

### 6.1 Wi-Fi

```bash
# 查看 Wi-Fi 接口与当前连接
system_profiler SPAirPortDataType

# 查看 PCI 设备是否正确绑定
ioreg -lw0 | grep -i "10ec,c821" -A 5
```

- 连接 WPA3-Personal AP，确认能获取 IP、能上网。
- 用 `iperf3` 或大文件拷贝制造持续流量，同时播放蓝牙音频 / 使用 BLE 键盘，观察是否断连。
- 在 Wi-Fi 菜单中手动“扫描”其他网络，观察蓝牙音频是否卡顿；这是最直接的共存压力测试。

### 6.2 蓝牙

```bash
# 蓝牙控制器与已配对设备
system_profiler SPBluetoothDataType

# RealtekBTPatcher 的注入计数
ioreg -lw0 | grep -oE '"IBTP[A-Za-z]+" = <[^>]*>'
```

- 确认 `IBTPWrapEnabled=Y`、`IBTPInjected>=1`（BLE HID 配对时）。
- BLE 键盘配对后静置 10 分钟，期间保持 Wi-Fi 有流量，观察是否掉线。
- 蓝牙音频播放时同时跑 Wi-Fi 测速，记录是否出现断续。

### 6.3 睡眠 / 唤醒

1. 合盖或手动睡眠 5 分钟；
2. 唤醒后分别检查 Wi-Fi 是否自动重连、蓝牙设备是否仍在；
3. 若单侧失效，先尝试关闭 / 打开对应开关，记录是否恢复；多次复现再考虑重置 NVRAM 或调整电源管理。

## 7. 日常使用建议

- **优先连接 5GHz Wi-Fi**：2.4GHz 与蓝牙的频段冲突最严重；5GHz 下 Wi-Fi 与蓝牙可以基本共存。
- **不要在用 BLE 键盘 / 蓝牙音频时频繁打开 Wi-Fi 扫描**；需要配网时先完成扫描，再回到正常使用。
- **保持天线连接良好**：二合一卡对天线阻抗敏感；换壳、拆机后确认主 / 辅天线都接好。
- **避免 USB 3.0 干扰**：2.4GHz 蓝牙与 USB 3.0 接口的宽带噪声可能互相干扰，尽量把 USB 3.0 设备远离天线区域。
- **睡眠唤醒后单侧失效**：目前没有彻底的软件修复；先 toggle 对应开关，仍无效则重启。该问题列为**待验证 / 已知限制**，不要期望通过换 kext 版本完全消除。

## 8. 已知限制

- 本机的 Wi-Fi / 蓝牙共存修复仅在 **macOS Tahoe 26.7.2 + RTL8821CE（`10EC:C821` / `0BDA:B00A`）** 上验证；其他 Realtek combo 卡（RTL8822CE、RTL8852 等）不保证适用。
- 共存是硬件层面的时分复用，**不可能让 2.4GHz Wi-Fi 与蓝牙同时满速**；本修复的目标是“可用、少断”，不是“零干扰”。
- 睡眠唤醒后的单侧失效尚未定位到单一根因，可能同时涉及 PCIe 电源管理、USB 电源管理与固件重载顺序；目前只能通过重启 / 重新开关恢复。
- `RealtekBluetoothFirmware` 与 `BlueToolFixup` 的版本组合敏感；升级任一组件后建议重新跑本文第 6 节的验证。

## 9. 参考资料

- [JoMei9019-real/AirPort-RTW](https://github.com/JoMei9019-real/AirPort-RTW) —— AirPortRTW 上游。
- [Emilelu/RealtekAirPortRTWFix](https://github.com/Emilelu/RealtekAirPortRTWFix) —— Wi-Fi / 蓝牙共存修复。
- [thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware) —— Realtek 蓝牙固件加载器（本机使用 1.0.2 官方 release）。
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) —— BlueToolFixup 上游。
- [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher) —— BLE HID 事件注入补丁。
- 本仓库 [蓝牙 BLE HID 修复记录](./蓝牙-BLE-HID-修复记录.md) —— 蓝牙侧的事件注入细节。
- 本仓库 [HDMI 修复记录](./HDMI-修复记录.md) —— 同机型的显示输出修复，便于区分“显示问题”与“无线问题”。

> 完整致谢与全部外部链接见 [主 README 的 Credits 章节](../README.md#credits--致谢)。
